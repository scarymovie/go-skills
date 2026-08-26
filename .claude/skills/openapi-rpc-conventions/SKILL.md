---
name: openapi-rpc-conventions
description: >
  Конвенции OpenAPI-спеки в стиле RPC-over-HTTP: только POST, вход всегда в теле,
  именование операций и схем (`*Input`, `*Output`, `*Params`, `Nwk*`), единый `NwkError`
  и закрытый перечень статусов 200/400/500. RPC-over-HTTP OpenAPI conventions: POST-only
  endpoints, request/response schema naming, unified error format, closed status list.
when_to_use: >
  Когда пишешь или ревьюишь OpenAPI-спеку, добавляешь эндпоинт, называешь схему запроса
  или ответа, решаешь каким статусом отдать ошибку. Триггеры: "openapi", "api spec",
  "endpoint", "operation", "request schema", "response schema", "openapi conventions",
  "спека", "апи", "эндпоинт", "операция", "схема запроса", "код ошибки", "NwkError".
  Для WebSocket-контракта — скилл `websocket-openapi-guidelines`, для настройки
  генератора — `openapi-codegen`.
---

# Конвенции OpenAPI в стиле RPC-over-HTTP

Go 1.27 (минимум 1.26); oapi-codegen v2.8.0 требует Go 1.25+.

## Эндпоинт — это операция, а не ресурс

Отсюда следует всё остальное:

- **Только POST.** Ни GET, ни PUT, ни PATCH, ни DELETE. Единственное исключение —
  эндпоинт WebSocket-хендшейка: по RFC 6455 это обычный GET, и отвечает он 101/401
  (скилл `websocket-openapi-guidelines`).
- **Ни query-, ни path-параметров.** Если у операции есть вход — он целиком в теле запроса.
- Тело запроса необязательно: операция без входа — валидный случай. Но раз вход есть —
  `required: true`, иначе сгенерированный `Body` приезжает `nil` при пустом теле и
  nil-проверка расползается по всем хендлерам.

Причина: у операции нет URL-ресурса, состоянием которого можно управлять глаголами метода.
`/dialog/create` — это имя вызова, и единственное, что нужно транспорту, — донести аргументы.

## Именование: operationId и имена схем — разные идентификаторы

`operationId` — глагол+существительное с **маленькой** буквы: `createDialog`, `getUserList`,
`searchMessages`. Имена схем — **PascalCase от того же operationId** плюс суффикс:
`CreateDialogInput`, `CreateDialogOutput`.

Это не одна и та же строка, и путать их нельзя: `operationId: CreateDialog` — ошибка,
схема `createDialogInput` — тоже. Регистр несёт смысл, потому что oapi-codegen сам
поднимает первую букву `operationId` в имя метода: из `createDialog` выходит `CreateDialog`,
и `CreateDialogInput` читается как «вход метода `CreateDialog`».

У эндпоинта не больше двух собственных типов: `<OperationId>Input` и `<OperationId>Output`.

```yaml
/dialog/create:
  post:
    operationId: createDialog                            # camelCase
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/CreateDialogInput'   # PascalCase + Input
```

### Переиспользуемые группы параметров — суффикс `Params`

```yaml
PaginationParams:
  type: object
  properties:
    page: { type: integer }
    perPage: { type: integer }

UserSearchParams:       # вариант под конкретную сущность тоже допустим
  type: object
  properties:
    query: { type: string }
    role: { type: string }
```

`*Params` вкладываются внутрь `*Input`, а не подставляются телом запроса напрямую:
тело запроса всегда `*Input`, иначе имя операции перестаёт прослеживаться в типе.

### Общие DTO провода — префикс `Nwk`

Всё, что ездит по проводу и переиспользуется между операциями, называется `Nwk<Entity>`:
`NwkDialog`, `NwkUser`, `NwkError`.

```yaml
NwkDialog:
  type: object
  properties:
    id: { type: string }
    title: { type: string }
```

**Ключевое правило:** `Nwk*` не появляются в usecase и в домене. Конвертация — только в
мапперах `internal/infrastructure/http/<name>/mapper/`. Причина: эти типы генерируются
из спеки и меняются вместе с ней, поэтому любая протечка делает домен заложником формата
провода — переименование поля в JSON начинает править бизнес-логику.

## Перечень статусов закрыт: 200, 400, 500

- `200` — операция выполнена. Тело — `<OperationId>Output`.
- `400` — вход не прошёл валидацию, виноват вызывающий. Тело — `NwkError`.
- `500` — сбой на нашей стороне. Тело — `NwkError`.

**404 не используется.** Отсутствие сущности — это результат операции, а не сбой
транспорта: эндпоинт `/dialog/get` существует всегда, отсутствует лишь запрошенный диалог,
поэтому «не найдено» выражается в теле 200-ответа.

```yaml
GetDialogOutput:
  type: object
  required: [found]
  properties:
    found: { type: boolean }
    dialog:
      $ref: '#/components/schemas/NwkDialog'    # отсутствует, когда found == false
```

По той же причине не появляются 401/403/409/422: оттенок ошибки несёт поле `code` внутри
`NwkError`, а не статус; аутентификация живёт в мидлвари и отвечает до входа в хендлер.
Встретился в спеке `'404'` с телом `NwkError` или `http.StatusNotFound` в хендлере — это
отклонение от конвенции: перенеси признак в `*Output` и верни 200.

```yaml
NwkError:
  type: object
  required: [code, message]
  properties:
    code: { type: string }       # машиночитаемый: VALIDATION_ERROR, INTERNAL_ERROR
    message: { type: string }    # человекочитаемое сообщение
```

Полный ответный блок операции всегда выглядит одинаково:

```yaml
    responses:
      '200':
        description: OK
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateDialogOutput'
      '400':
        description: Validation error
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/NwkError'
      '500':
        description: Internal server error
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/NwkError'
```

## Перечень статусов — это форма сгенерированного интерфейса

При `strict-server` (кодоген настраивается в скилле `openapi-codegen`) каждый описанный
статус превращается в отдельный тип `ResponseObject`, который хендлер обязан уметь вернуть:

```go
// Сгенерировано strict-server по блоку responses выше.
type CreateDialog200JSONResponse CreateDialogOutput
type CreateDialog400JSONResponse NwkError
type CreateDialog500JSONResponse NwkError

type StrictServerInterface interface {
	CreateDialog(ctx context.Context, request CreateDialogRequestObject) (CreateDialogResponseObject, error)
}
```

То есть перечень статусов — не документация, а прямо публичный API нашего кода: добавили
404 в спеку — получили ещё один тип и ещё одну ветку в каждом хендлере этой операции.
Это и есть главный аргумент держать перечень коротким и одинаковым для всех операций.

## Сводка по именованию

| Что | Правило | Пример |
|---|---|---|
| Идентификатор операции | camelCase, первая буква маленькая | `createDialog` |
| Тело запроса | PascalCase от operationId + `Input` | `CreateDialogInput` |
| Тело ответа | PascalCase от operationId + `Output` | `CreateDialogOutput` |
| Группа параметров | `<Entity?>Params`, вкладывается в `*Input` | `PaginationParams`, `UserSearchParams` |
| DTO провода | `Nwk<Entity>`, не пересекает границу домена | `NwkDialog`, `NwkUser` |
| Ошибка | `NwkError`, один на все операции | тело 400 и 500 |
