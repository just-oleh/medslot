# MedSlot: Архітектура

## 1. Класифікація архітектури

| Вимір | Вибір | Обґрунтування |
|---|---|---|
| Стиль бекенду | **Модульний моноліт** | Один сервіс розгортається разом, але код поділений на модулі з чіткими межами. Для одного розробника й однієї клініки простіше за мікросервіси, а за потреби модуль можна виділити окремо |
| Клієнт | **SPA** (single page application) | Динамічний вибір слотів без перезавантаження сторінки |
| Взаємодія | **REST + JSON** над HTTPS | Просто, добре документується через OpenAPI |
| Асинхронна частина | **Фоновий воркер + черга** | Сповіщення й обробка листа очікування не повинні гальмувати запис |
| Дані | **PostgreSQL** (основне сховище) + **Redis** (тимчасові дані) | Реляційна модель для записів, швидкі атомарні ключі для утримання слотів і лічильників |
| Розгортання | Docker-контейнери (деталі в окремому документі Deployment model) | Однаковий запуск у розробці й на сервері |

### Чому не мікросервіси
Мікросервіси дають незалежне масштабування й розгортання, але додають мережеві збої, розподілені транзакції та складну інфраструктуру. Для MedSlot основне навантаження зосереджене в одному модулі (Booking), тому виграш мінімальний. Межі модулів (нижче) залишають можливість розділення пізніше.

## 2. Діаграма компонентів

```mermaid
flowchart LR
    subgraph Client
        SPA[Web SPA]
    end
    subgraph Backend["Backend (модульний моноліт)"]
        API[REST API + валідація]
        AUTH[Auth / Identity]
        BOOK[Booking]
        SCHED[Schedules]
        ADMIN[Admin]
        OUTBOX[Outbox Worker]
        NOTIFY[Notification Worker]
    end
    DB[(PostgreSQL)]
    CACHE[(Redis)]
    QUEUE[[Черга повідомлень]]
    MAIL[Email / SMS provider]

    SPA -->|HTTPS + JWT| API
    API --> AUTH
    API --> BOOK
    API --> SCHED
    API --> ADMIN
    AUTH --> DB
    BOOK --> DB
    BOOK --> CACHE
    SCHED --> DB
    ADMIN --> DB
    API --> CACHE
    OUTBOX --> DB
    OUTBOX --> QUEUE
    QUEUE --> NOTIFY
    NOTIFY --> MAIL
```

## 3. Модулі та відповідальності

| Модуль | Відповідає за | Залежить від |
|---|---|---|
| **REST API** | Маршрутизація, перевірка токена, валідація, єдиний формат помилок, rate limiting | Auth, Redis |
| **Auth / Identity** | Реєстрація, вхід, ролі, токени, блокування акаунтів | PostgreSQL |
| **Booking** | Слоти, утримання, запис, скасування, перенесення, лист очікування | PostgreSQL, Redis |
| **Schedules** | Розклади лікарів, закриття слотів | PostgreSQL |
| **Admin** | Лікарі, клініка, блокування, статистика | PostgreSQL |
| **Outbox Worker** | Передача подій із таблиці `outbox` у чергу | PostgreSQL, черга |
| **Notification Worker** | Надсилання email, повтори, обробка збоїв | Черга, провайдер |

Модулі не звертаються до таблиць один одного напряму, лише через інтерфейси модулів.

## 4. Ключові архітектурні рішення

| Рішення | Альтернатива | Причина вибору |
|---|---|---|
| Redis для утримання слотів | Блокування рядків у БД | Швидше, TTL автоматично знімає утримання; БД усе одно має унікальний індекс як остання гарантія |
| Transactional Outbox для подій | Пряма відправка в чергу з API | Подія не губиться, якщо сервіс падає між зміною даних і публікацією |
| Короткоживучі JWT (15 хв) + refresh-токени | Серверні сесії | Клієнт без стану на кожному запиті, а refresh можна відкликати |
| Єдиний формат помилок | Різні формати по модулях | Клієнт обробляє всі помилки однаково, див. [error-handling.md](error-handling.md) |

## 5. Діаграми послідовності

### 5.1 Вхід у систему

```mermaid
sequenceDiagram
    actor U as Користувач
    participant SPA as Web SPA
    participant API as REST API
    participant AUTH as Auth
    participant DB as PostgreSQL

    U->>SPA: вводить email і пароль
    SPA->>API: POST /auth/login
    API->>AUTH: перевірити облікові дані
    AUTH->>DB: SELECT користувач за email
    DB-->>AUTH: запис користувача
    AUTH->>AUTH: порівняти хеш пароля
    alt дані вірні і акаунт активний
        AUTH-->>API: успіх
        API-->>SPA: 200 accessToken і refreshToken
        SPA-->>U: перехід до особистого кабінету
    else неправильні дані
        AUTH-->>API: невдача
        API-->>SPA: 401 INVALID_CREDENTIALS
        SPA-->>U: повідомлення про помилку
    end
```

### 5.2 Запис до лікаря (успішний сценарій)

```mermaid
sequenceDiagram
    actor P as Пацієнт
    participant SPA as Web SPA
    participant API as REST API
    participant BOOK as Booking
    participant R as Redis
    participant DB as PostgreSQL
    participant N as Notification Worker

    P->>SPA: обирає лікаря і дату
    SPA->>API: GET /doctors/ID/slots
    API->>BOOK: вільні слоти
    BOOK->>DB: SELECT слоти зі статусом FREE
    DB-->>BOOK: список слотів
    BOOK-->>SPA: список слотів
    P->>SPA: обирає слот
    SPA->>API: POST /slots/ID/hold
    API->>BOOK: утримати слот
    BOOK->>R: SET hold NX EX 300
    R-->>BOOK: OK
    BOOK-->>SPA: 200 утримання на 5 хвилин
    P->>SPA: підтверджує запис
    SPA->>API: POST /appointments
    API->>BOOK: створити запис
    BOOK->>DB: INSERT appointment і outbox
    DB-->>BOOK: OK
    BOOK->>R: DEL hold
    BOOK-->>SPA: 201 запис підтверджено
    DB-->>N: подія AppointmentCreated через outbox і чергу
    N-->>P: email з підтвердженням
```

### 5.3 Скасування запису

```mermaid
sequenceDiagram
    actor P as Пацієнт
    participant SPA as Web SPA
    participant API as REST API
    participant BOOK as Booking
    participant DB as PostgreSQL

    P->>SPA: натискає Скасувати
    SPA->>API: DELETE /appointments/ID
    API->>BOOK: скасувати запис
    BOOK->>DB: SELECT запис
    DB-->>BOOK: запис і час прийому
    alt запис належить пацієнту і до прийому понад 2 години
        BOOK->>DB: UPDATE status CANCELLED, INSERT outbox SlotReleased
        DB-->>BOOK: OK
        BOOK-->>SPA: 204
        SPA-->>P: запис скасовано
    else чужий запис
        BOOK-->>SPA: 403 NOT_APPOINTMENT_OWNER
    else менше 2 годин до прийому
        BOOK-->>SPA: 422 CANCELLATION_WINDOW_PASSED
    end
```

Додаткові діаграми послідовності, пов'язані з паралельною роботою (гонка за слот, Saga перенесення, Outbox, ретраї), описано в [concurrency.md](concurrency.md).
