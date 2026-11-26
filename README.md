# MedSlot

Веб-застосунок для онлайн-запису пацієнтів до лікарів. Навчальний проєкт з дисципліни «Об'єктно-орієнтований аналіз і проєктування ПЗ».

Пацієнт шукає лікаря, бачить вільні слоти й записується без дзвінків. Лікар веде свій розклад, адміністратор клініки керує лікарями та акаунтами. Основний акцент проєкту: коректна робота за умови одночасних запитів на один слот.

## Документація (завдання 2)

| Вимога | Документ |
|---|---|
| Опис продукту: функціональна специфікація, UML, Identity Management | [docs/SRS.md](docs/SRS.md) (16 FR, 9 NFR, 3 актори, 3 UML-діаграми) |
| Архітектура та діаграми послідовності | [docs/architecture.md](docs/architecture.md) |
| Error Handling design (OpenAPI, 13 операцій) | [docs/error-handling.md](docs/error-handling.md), [docs/api/openapi.yaml](docs/api/openapi.yaml) |
| Concurrency Patterns usage (7 потоків) | [docs/concurrency.md](docs/concurrency.md) |

## Як переглянути OpenAPI

Відкрийте [Swagger Editor](https://editor.swagger.io/) і вставте вміст файлу `docs/api/openapi.yaml`.

## Структура репозиторію

```
medslot/
├── README.md
└── docs/
    ├── SRS.md
    ├── architecture.md
    ├── error-handling.md
    ├── concurrency.md
    └── api/
        └── openapi.yaml
```

## Статус

Проєкт містить лише документацію та моделі. Реалізація застосунку не передбачена завданням.

## Автор

Мусурівський Олег, ПМІм-12c
