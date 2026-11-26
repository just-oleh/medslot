# MedSlot: Error Handling Design

Специфікація API: [`api/openapi.yaml`](api/openapi.yaml) (OpenAPI 3.0, 13 операцій). Її можна відкрити в [Swagger Editor](https://editor.swagger.io/), вставивши вміст файлу.

## 1. Єдиний формат помилки

Усі помилки повертаються з `Content-Type: application/json` у такій структурі:

```json
{
  "status": 409,
  "code": "SLOT_ALREADY_HELD",
  "title": "Slot is not available",
  "detail": "Slot is currently held by another patient",
  "traceId": "7f3c1b2e-9a44-4d0a-8b5f-0c2d1e6a9b10",
  "errors": [{ "field": "password", "message": "Must be 12 to 128 characters" }]
}
```

| Поле | Призначення |
|---|---|
| `status` | HTTP-код, дублюється в тілі |
| `code` | Стабільний машинозчитуваний код, на який орієнтується клієнт (текст `detail` може змінюватись) |
| `title`, `detail` | Короткий і докладний опис для логів |
| `traceId` | Ідентифікатор запиту, щоб знайти подію в журналі |
| `errors` | Список помилок окремих полів (лише 400 і 422) |

Правила: деталі внутрішньої реалізації (стек викликів, SQL, шляхи) у відповідь не потрапляють; для автентифікації повідомлення однакове і не розкриває, чи існує email.

## 2. Коди статусів

| Код | Коли використовується | Повторювати запит? |
|---|---|---|
| **400** | Некоректний JSON, неправильний тип чи формат параметра | Ні, виправити запит |
| **401** | Немає токена, токен прострочений або відкликаний, неправильний пароль | Оновити токен або увійти знову |
| **403** | Недостатньо прав, чужий ресурс, акаунт заблоковано | Ні |
| **404** | Ресурс не існує | Ні |
| **409** | Конфлікт стану: слот зайнятий, дубль, ресурс уже змінений | Залежить від `code`, зазвичай обрати інший слот |
| **422** | Формат правильний, але порушено валідацію чи бізнес-правило | Ні, виправити дані |
| **429** | Перевищено ліміт запитів | Так, після `Retry-After` |
| **5xx** | Внутрішня помилка або недоступність залежності | Так, з експоненційною паузою |

Різниця між 400 і 422: **400** означає, що сервер не зміг розібрати запит, а **422** означає, що запит зрозумілий, але дані не відповідають правилам.

## 3. Приклади для кожної операції

Базова адреса: `https://api.medslot.example/v1`. У прикладах заголовок `Authorization: Bearer <token>` для скорочення пропущено.

### 3.1 POST /auth/register: 422 (слабкий пароль)

Запит:
```http
POST /auth/register
{ "email": "olena@example.com", "password": "12345", "fullName": "Олена Коваль" }
```
Відповідь `422`:
```json
{
  "status": 422, "code": "VALIDATION_FAILED", "title": "Validation failed",
  "detail": "One or more fields are invalid", "traceId": "a1b2c3d4",
  "errors": [{ "field": "password", "message": "Must be 12 to 128 characters" }]
}
```
**Реакція клієнта:** показати повідомлення під полем `password`, не очищати інші поля, нічого не повторювати автоматично.

Той самий endpoint, **409** `EMAIL_ALREADY_EXISTS`: якщо email уже зареєстрований, клієнт пропонує перейти на вхід або відновлення пароля.

### 3.2 POST /auth/login: 401, 403 і 429

Запит:
```http
POST /auth/login
{ "email": "olena@example.com", "password": "wrong-password-123" }
```
Відповідь `401`:
```json
{ "status": 401, "code": "INVALID_CREDENTIALS", "title": "Authentication failed",
  "detail": "Email or password is incorrect", "traceId": "b2c3d4e5" }
```
**Реакція клієнта:** показати загальне повідомлення «Невірний email або пароль», без підказки, що саме неправильно.

Інші відповіді цього endpoint:
- **403** `ACCOUNT_BLOCKED`: показати «Акаунт заблоковано, зверніться до адміністратора клініки».
- **429** `TOO_MANY_LOGIN_ATTEMPTS` із заголовком `Retry-After: 60`: заблокувати кнопку входу й показати таймер.

### 3.3 POST /auth/refresh: 401 (токен відкликано)

Запит:
```http
POST /auth/refresh
{ "refreshToken": "revoked-or-expired-token" }
```
Відповідь `401`:
```json
{ "status": 401, "code": "REFRESH_TOKEN_INVALID", "title": "Token invalid",
  "detail": "Refresh token is expired or revoked", "traceId": "c3d4e5f6" }
```
**Реакція клієнта:** очистити збережені токени й перенаправити на сторінку входу. Повторювати refresh не потрібно.

### 3.4 GET /doctors: 400 (некоректний параметр)

Запит:
```http
GET /doctors?specialty=therapist&size=500
```
Відповідь `400`:
```json
{ "status": 400, "code": "INVALID_QUERY_PARAMETER", "title": "Bad request",
  "detail": "Parameter size must be between 1 and 50", "traceId": "d4e5f6a7",
  "errors": [{ "field": "size", "message": "Must be between 1 and 50" }] }
```
**Реакція клієнта:** це помилка програми, а не користувача. Записати в лог, використати значення за замовчуванням (`size=20`) і повторити запит один раз.

### 3.5 GET /doctors/{doctorId}/slots: 404 і 400

Запит:
```http
GET /doctors/00000000-0000-0000-0000-000000000000/slots?date=2026-12-10
```
Відповідь `404`:
```json
{ "status": 404, "code": "DOCTOR_NOT_FOUND", "title": "Not found",
  "detail": "Doctor does not exist", "traceId": "e5f6a7b8" }
```
**Реакція клієнта:** показати «Лікаря не знайдено» і повернути до списку лікарів, оновивши його.

Якщо `date=10-12-2026`, повертається **400** `INVALID_DATE_FORMAT`: клієнт виправляє формат на `YYYY-MM-DD`.

### 3.6 POST /slots/{slotId}/hold: 409 (слот утримує інший пацієнт)

Запит:
```http
POST /slots/5c1d2e3f-0000-4000-8000-000000000001/hold
```
Відповідь `409`:
```json
{ "status": 409, "code": "SLOT_ALREADY_HELD", "title": "Slot is not available",
  "detail": "Slot is currently held by another patient", "traceId": "f6a7b8c9" }
```
**Реакція клієнта:** показати «Цей час щойно обрав інший пацієнт», оновити список слотів і запропонувати лист очікування. Те саме повертає **404** `SLOT_NOT_FOUND`, якщо слот видалено.

### 3.7 POST /appointments: 409 і 422

Запит:
```http
POST /appointments
Idempotency-Key: 4f0e7b9c-1111-4222-8333-444455556666
{ "slotId": "5c1d2e3f-0000-4000-8000-000000000001", "holdId": "9a8b7c6d-0000-4000-8000-000000000002" }
```
Відповідь `409` (слот уже підтверджений для іншого пацієнта):
```json
{ "status": 409, "code": "SLOT_ALREADY_BOOKED", "title": "Slot is not available",
  "detail": "Slot has been booked by another patient", "traceId": "a7b8c9d0" }
```
**Реакція клієнта:** повернути до вибору слотів, показати «На жаль, цей час уже зайнятий».

Відповідь `422` `HOLD_EXPIRED`, якщо минуло 5 хвилин утримання:
```json
{ "status": 422, "code": "HOLD_EXPIRED", "title": "Hold expired",
  "detail": "The 5 minute hold on this slot has expired", "traceId": "b8c9d0e1" }
```
**Реакція клієнта:** запропонувати «Обрати час знову» і виконати `POST /slots/{id}/hold` ще раз.

Повтор запиту з тим самим `Idempotency-Key` і тим самим тілом повертає **той самий результат** (201), а не створює другий запис. Це безпечно робити після обриву мережі.

### 3.8 GET /appointments: 401 (прострочений access-токен)

Запит:
```http
GET /appointments?status=CONFIRMED
```
Відповідь `401`:
```json
{ "status": 401, "code": "ACCESS_TOKEN_EXPIRED", "title": "Unauthorized",
  "detail": "Access token has expired", "traceId": "c9d0e1f2" }
```
**Реакція клієнта:** автоматично викликати `/auth/refresh`, зберегти нову пару токенів і **один раз** повторити початковий запит. Якщо refresh теж повертає 401, перейти на сторінку входу.

### 3.9 DELETE /appointments/{id}: 403 і 422

Запит:
```http
DELETE /appointments/1111aaaa-0000-4000-8000-00000000aaaa
```
Відповідь `403` (запис іншого пацієнта):
```json
{ "status": 403, "code": "NOT_APPOINTMENT_OWNER", "title": "Forbidden",
  "detail": "You can only manage your own appointments", "traceId": "d0e1f2a3" }
```
**Реакція клієнта:** показати «Немає доступу до цього запису» і повернути до власного списку.

Відповідь `422` `CANCELLATION_WINDOW_PASSED`, якщо до прийому менше 2 годин:
```json
{ "status": 422, "code": "CANCELLATION_WINDOW_PASSED", "title": "Cannot cancel",
  "detail": "Appointments can be cancelled at least 2 hours before the start", "traceId": "e1f2a3b4" }
```
**Реакція клієнта:** сховати кнопку скасування, показати телефон клініки.

### 3.10 PATCH /appointments/{id}: 409 (цільовий слот зайнятий)

Запит:
```http
PATCH /appointments/1111aaaa-0000-4000-8000-00000000aaaa
{ "newSlotId": "5c1d2e3f-0000-4000-8000-000000000003" }
```
Відповідь `409`:
```json
{ "status": 409, "code": "TARGET_SLOT_UNAVAILABLE", "title": "Slot is not available",
  "detail": "The requested slot is already taken. Your current appointment is unchanged",
  "traceId": "f2a3b4c5" }
```
**Реакція клієнта:** повідомити, що поточний запис **збережено**, і запропонувати обрати інший слот. Це гарантує сага перенесення (див. [concurrency.md](concurrency.md), потік 6).

### 3.11 POST /slots/{slotId}/waitlist: 409 і 422

Запит:
```http
POST /slots/5c1d2e3f-0000-4000-8000-000000000001/waitlist
```
Відповідь `409`:
```json
{ "status": 409, "code": "ALREADY_IN_WAITLIST", "title": "Conflict",
  "detail": "You are already in the waitlist for this slot", "traceId": "a3b4c5d6" }
```
**Реакція клієнта:** показати «Ви вже в черзі», кнопку не вимикати помилкою, а змінити на «Вийти з черги».

Відповідь `422` `SLOT_IS_FREE`: слот уже вільний, у чергу ставати не потрібно, клієнт переходить до звичайного запису.

### 3.12 POST /admin/doctors: 403, 400, 422

Запит від пацієнта (не адміністратора):
```http
POST /admin/doctors
{ "email": "doc@clinic.com", "fullName": "Андрій Мельник", "specialty": "therapist", "clinicId": "..." }
```
Відповідь `403`:
```json
{ "status": 403, "code": "INSUFFICIENT_ROLE", "title": "Forbidden",
  "detail": "This operation requires role CLINIC_ADMIN", "traceId": "b4c5d6e7" }
```
**Реакція клієнта:** не показувати цей розділ у меню для ролі пацієнта. Якщо відповідь усе ж прийшла, показати «Недостатньо прав».

Якщо тіло запиту не JSON: **400** `MALFORMED_JSON`. Якщо `specialty` порожнє: **422** `VALIDATION_FAILED` із `errors[]`. Якщо лікар із таким email уже є: **409**.

### 3.13 POST /admin/patients/{id}/block: 404 і 429

Запит:
```http
POST /admin/patients/00000000-0000-0000-0000-000000000000/block
```
Відповідь `404`:
```json
{ "status": 404, "code": "PATIENT_NOT_FOUND", "title": "Not found",
  "detail": "Patient does not exist", "traceId": "c5d6e7f8" }
```
**Реакція клієнта:** показати «Пацієнта не знайдено» і оновити список.

Відповідь `429` (більше 60 запитів на хвилину):
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```
```json
{ "status": 429, "code": "RATE_LIMIT_EXCEEDED", "title": "Too many requests",
  "detail": "Limit of 60 requests per minute exceeded", "traceId": "d6e7f8a9" }
```
**Реакція клієнта:** зачекати `Retry-After` секунд і повторити запит. Не слати запити частіше, а послідовні повтори розносити з експоненційною паузою.

## 4. Підсумкова таблиця покриття

| Операція | Показані коди |
|---|---|
| POST /auth/register | 409, 422 |
| POST /auth/login | 401, 403, 429 |
| POST /auth/refresh | 401 |
| GET /doctors | 400 |
| GET /doctors/{id}/slots | 400, 404 |
| POST /slots/{id}/hold | 404, 409 |
| POST /appointments | 409, 422 |
| GET /appointments | 401 |
| DELETE /appointments/{id} | 403, 422 |
| PATCH /appointments/{id} | 409 |
| POST /slots/{id}/waitlist | 409, 422 |
| POST /admin/doctors | 400, 403, 409, 422 |
| POST /admin/patients/{id}/block | 404, 429 |

Усі шість обов'язкових статусів (400, 403, 404, 409, 422, 429) показані в прикладах.

## 5. Загальні правила для клієнта

1. Рішення приймати за полем **`code`**, а не за текстом.
2. Повторювати автоматично можна лише **429** (після `Retry-After`), **401** з валідним refresh-токеном (один раз) та **5xx** (з експоненційною паузою).
3. **409** і **422** автоматично не повторювати: користувач має змінити вибір чи дані.
4. `traceId` показувати в повідомленні про непередбачену помилку, щоб його можна було передати в підтримку.
