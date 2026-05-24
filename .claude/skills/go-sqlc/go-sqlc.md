---
name: go-sqlc
description: >
  sqlc code generation for Go — setup, query writing, schema validation, and migration from manual pgx
  to sqlc-generated repositories. Use when user asks about sqlc, database code generation, converting
  pgx repositories to sqlc, writing SQL queries for sqlc, setting up sqlc.yaml configuration, type
  overrides for custom PostgreSQL types, sqlc.embed for JOIN queries, enum generation, debugging
  sqlc generate errors, or integrating sqlc with avito-tech/go-transaction-manager.
  Also trigger when user mentions "type-safe SQL", "pgx boilerplate", "sqlc diff in CI",
  or wants to remove manual row scanning from repository code.
version: 1.1.0
---

# go-sqlc skill

Помощь с sqlc — генератором типобезопасного Go-кода из SQL-запросов.

> **Минимальная версия sqlc: v1.26+**

## Когда использовать

- Настройка sqlc в проекте (создание `sqlc.yaml`)
- Написание SQL-запросов с аннотациями sqlc (`-- name: ... :one/:many/:exec`)
- Миграция существующих pgx-репозиториев на sqlc
- Отладка ошибок генерации sqlc
- Настройка `overrides` для кастомных PostgreSQL-типов (uuid, numeric, enum)
- Использование `sqlc.embed` для вложенных структур в JOIN-запросах
- Интеграция sqlc с transaction-manager (avito-tech/go-transaction-manager)
- Настройка CI/CD для проверки актуальности сгенерированного кода

## Принципы работы с sqlc

### 1. Структура проекта

```
project/
├── sqlc.yaml                          # Конфигурация sqlc
├── migrations/                        # Кастомные SQL-миграции (схема БД)
│   ├── 001_initial.sql
│   └── 002_add_column.sql
├── internal/infrastructure/postgres/
│   ├── queries/                       # SQL-запросы с аннотациями sqlc
│   │   ├── user.sql
│   │   └── order.sql
│   ├── sqlc/                          # Сгенерированный код (коммитить в git)
│   │   ├── db.go                      # Интерфейс Querier
│   │   ├── models.go                  # Структуры таблиц
│   │   ├── user.sql.go
│   │   └── order.sql.go
│   └── repository/                    # Тонкие обёртки над sqlc (domain mapping)
│       ├── user_repository.go
│       └── order_repository.go
```

**Важно:** sqlc читает схему из файлов `migrations/` — не требует запущенной БД для генерации.
Формат файлов миграций: любые `.sql` файлы с `CREATE TABLE`, `ALTER TABLE` и т.д.
sqlc рекурсивно обходит указанную директорию.

### 2. Конфигурация sqlc.yaml

**Базовая конфигурация (парсинг миграций):**

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "internal/infrastructure/postgres/queries"
    schema: "migrations"  # Путь к .sql файлам схемы
    gen:
      go:
        package: "sqlc"
        out: "internal/infrastructure/postgres/sqlc"
        emit_json_tags: false
        emit_interface: true
        emit_exact_table_names: false
        emit_pointers_for_null_types: true
        query_parameter_limit: 10
        overrides:
          - db_type: "uuid"
            go_type:
              import: "github.com/google/uuid"
              type: "UUID"
          - db_type: "numeric"
            go_type:
              import: "github.com/shopspring/decimal"
              type: "Decimal"
```

**С подключением к БД (если схема сложная или использует расширения):**

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "internal/infrastructure/postgres/queries"
    database:
      uri: "${DATABASE_URL}"  # Читает из env
    gen:
      go:
        package: "sqlc"
        out: "internal/infrastructure/postgres/sqlc"
        emit_json_tags: false
        emit_interface: true
        emit_pointers_for_null_types: true
        overrides:
          - db_type: "uuid"
            go_type:
              import: "github.com/google/uuid"
              type: "UUID"
```

**Ключевые опции:**

| Опция | Значение | Зачем |
|-------|----------|-------|
| `emit_interface: true` | Генерирует интерфейс `Querier` | Для моков в тестах |
| `emit_exact_table_names: false` | `User` вместо `Users` | Singular naming |
| `emit_pointers_for_null_types: true` | `*string` вместо `sql.NullString` | Удобнее работать |
| `emit_json_tags: false` | Без `json:` тегов | sqlc-структуры не для JSON |
| `query_parameter_limit: 10` | Лимит параметров в запросе | Защита от раздутых запросов |

### 3. Overrides — маппинг PostgreSQL-типов на Go-типы

По умолчанию sqlc использует `pgtype.*` для многих типов. Чтобы получить идиоматичные Go-типы:

```yaml
overrides:
  # uuid → github.com/google/uuid
  - db_type: "uuid"
    go_type:
      import: "github.com/google/uuid"
      type: "UUID"

  # numeric → github.com/shopspring/decimal
  - db_type: "numeric"
    go_type:
      import: "github.com/shopspring/decimal"
      type: "Decimal"

  # Nullable uuid → *uuid.UUID (если emit_pointers_for_null_types недостаточно)
  - db_type: "uuid"
    nullable: true
    go_type:
      import: "github.com/google/uuid"
      type: "UUID"
      pointer: true

  # Кастомный тип для конкретной колонки
  - column: "users.status"
    go_type: "github.com/yourorg/project/internal/domain.UserStatus"
```

### 4. Enum-типы PostgreSQL

sqlc автоматически генерирует Go-типы для PostgreSQL `ENUM`:

**Схема:**
```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'cancelled');

CREATE TABLE orders (
    id         uuid        PRIMARY KEY,
    status     order_status NOT NULL DEFAULT 'pending'
);
```

**Сгенерированный Go-код (`models.go`):**
```go
type OrderStatus string

const (
    OrderStatusPending   OrderStatus = "pending"
    OrderStatusPaid      OrderStatus = "paid"
    OrderStatusCancelled OrderStatus = "cancelled"
)

func (e *OrderStatus) Scan(src interface{}) error { ... }
func (e OrderStatus) Value() (driver.Value, error) { ... }
```

Поле `status` в структуре `Order` будет иметь тип `OrderStatus` — не `string`.

### 5. Написание SQL-запросов

**Аннотации sqlc:**

```sql
-- name: <MethodName> :<type>
-- :one    — возвращает одну строку (error если нет)
-- :many   — возвращает []Row
-- :exec   — без возврата данных
-- :execrows — возвращает количество затронутых строк
```

**Базовые примеры:**

```sql
-- queries/user.sql

-- name: GetUser :one
SELECT id, name, email, created_at
FROM users
WHERE id = $1;

-- name: ListUsers :many
SELECT id, name, email
FROM users
ORDER BY created_at DESC
LIMIT $1 OFFSET $2;

-- name: CreateUser :one
INSERT INTO users (id, name, email)
VALUES ($1, $2, $3)
RETURNING id, created_at;

-- name: UpdateUserEmail :exec
UPDATE users
SET email = $2, updated_at = NOW()
WHERE id = $1;

-- name: DeleteUser :execrows
DELETE FROM users WHERE id = $1;

-- name: CountActiveUsers :one
SELECT COUNT(*) FROM users WHERE status = 'active';
```

**Nullable поля:**

```sql
-- name: GetUserWithOptionalPhone :one
SELECT id, name, phone  -- phone NULL в схеме → *string в Go
FROM users
WHERE id = $1;
```

Генерируется:
```go
type GetUserWithOptionalPhoneRow struct {
    ID    uuid.UUID
    Name  string
    Phone *string  // nullable
}
```

**Явный CAST для предотвращения `interface{}`:**

```sql
-- Без CAST sqlc может не вывести тип COALESCE
SELECT COALESCE(phone, '')::text AS phone FROM users;
SELECT COALESCE(amount, 0)::numeric AS amount FROM orders;
```

### 6. sqlc.embed — вложенные структуры для JOIN

Вместо плоской `GetOrderWithUserRow` можно получить структуру с вложенными объектами:

**SQL:**
```sql
-- name: GetOrderWithUser :one
SELECT sqlc.embed(orders), sqlc.embed(users)
FROM orders
JOIN users ON users.id = orders.user_id
WHERE orders.id = $1;
```

**Сгенерированный Go-код:**
```go
type GetOrderWithUserRow struct {
    Order Order  // полная структура orders
    User  User   // полная структура users
}
```

**Без sqlc.embed (плоская структура — избегать):**
```sql
-- name: GetOrderWithUser :one
SELECT
    o.id AS order_id,
    o.total,
    u.id AS user_id,
    u.name AS user_name
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.id = $1;
-- → GetOrderWithUserRow с плоскими полями OrderID, Total, UserID, UserName
```

`sqlc.embed` — предпочтительный подход для JOIN, маппинг на domain-объекты становится чище.

### 7. Генерация кода

```bash
# Установка
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest

# Генерация
sqlc generate

# Проверка актуальности (для CI)
sqlc diff

# Дополнительные проверки
sqlc vet
```

**Makefile:**

```makefile
.PHONY: generate-sqlc sqlc-check

generate-sqlc:
	sqlc generate

sqlc-check:
	sqlc diff
	sqlc vet
```

### 8. Интеграция с transaction-manager

sqlc генерирует `*Queries`, принимающую `DBTX` интерфейс:

```go
type DBTX interface {
    Exec(context.Context, string, ...interface{}) (pgconn.CommandTag, error)
    Query(context.Context, string, ...interface{}) (pgx.Rows, error)
    QueryRow(context.Context, string, ...interface{}) pgx.Row
}
```

`pgxpool.Pool` и `pgx.Tx` реализуют этот интерфейс → работает с `trmpgxv5.CtxGetter` без изменений:

```go
package repository

import (
    "context"
    "errors"
    "fmt"

    trmpgxv5 "github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2"
    "github.com/google/uuid"
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"

    "gitlab.yoso.ru/platform/project/internal/domain"
    "gitlab.yoso.ru/platform/project/internal/infrastructure/postgres/sqlc"
)

type UserRepository struct {
    pool   *pgxpool.Pool
    getter *trmpgxv5.CtxGetter
}

func NewUserRepository(pool *pgxpool.Pool) *UserRepository {
    return &UserRepository{
        pool:   pool,
        getter: trmpgxv5.DefaultCtxGetter,
    }
}

func (r *UserRepository) GetUser(ctx context.Context, id uuid.UUID) (*domain.User, error) {
    db := r.getter.DefaultTrOrDB(ctx, r.pool)
    q := sqlc.New(db)

    row, err := q.GetUser(ctx, id)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, fmt.Errorf("user %s not found", id)
    }
    if err != nil {
        return nil, fmt.Errorf("GetUser: %w", err)
    }

    return toDomainUser(row), nil
}
```

**Ключевой момент:** `sqlc.New(db)` принимает `DBTX`, который может быть:
- `*pgxpool.Pool` — обычный запрос
- `pgx.Tx` — запрос внутри транзакции (извлекается через `trmpgxv5.CtxGetter`)

### 9. Паттерн репозитория

**Три слоя:**

1. **sqlc-структуры** (`sqlc/models.go`) — wire-формат БД, не покидают инфраструктурный слой
2. **Репозиторий** (`repository/`) — маппинг sqlc ↔ domain
3. **Domain** (`domain/`) — бизнес-логика

```go
// domain/user.go
type User struct {
    id    uuid.UUID
    name  string
    email string
}

func NewUser(id uuid.UUID, name, email string) *User {
    return &User{id: id, name: name, email: email}
}

func (u *User) Id() uuid.UUID { return u.id }
func (u *User) Name() string  { return u.name }
func (u *User) Email() string { return u.email }
```

```go
// repository/user_repository.go

func (r *UserRepository) GetUser(ctx context.Context, id uuid.UUID) (*domain.User, error) {
    q := sqlc.New(r.getter.DefaultTrOrDB(ctx, r.pool))
    row, err := q.GetUser(ctx, id)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, fmt.Errorf("user %s not found", id)
    }
    if err != nil {
        return nil, fmt.Errorf("GetUser: %w", err)
    }
    return toDomainUser(row), nil
}

func toDomainUser(row sqlc.User) *domain.User {
    return domain.NewUser(row.ID, row.Name, row.Email)
}
```

**Правила маппинга:**
- `errors.Is(err, pgx.ErrNoRows)` — всегда через `errors.Is`, не `==`
- Оборачивать ошибки через `fmt.Errorf("method: %w", err)` для трейсинга
- sqlc-структуры не должны выходить за пределы `repository/`

### 10. Миграция с ручного pgx на sqlc

**Шаг 1:** Создать `sqlc.yaml`

**Шаг 2:** Убедиться, что файлы схемы в `migrations/` содержат актуальный DDL

**Шаг 3:** Извлечь SQL из методов репозитория в `queries/*.sql`

Было:
```go
func (r *UserRepository) GetUser(ctx context.Context, id uuid.UUID) (*domain.User, error) {
    db := r.getter.DefaultTrOrDB(ctx, r.pool)
    rows, err := db.Query(ctx,
        `SELECT id, name, email FROM users WHERE id = $1`, id)
    // ручной scan...
}
```

Стало:
```sql
-- queries/user.sql
-- name: GetUser :one
SELECT id, name, email FROM users WHERE id = $1;
```

**Шаг 4:** Запустить `sqlc generate`

**Шаг 5:** Заменить ручной код:
```go
func (r *UserRepository) GetUser(ctx context.Context, id uuid.UUID) (*domain.User, error) {
    q := sqlc.New(r.getter.DefaultTrOrDB(ctx, r.pool))
    row, err := q.GetUser(ctx, id)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, fmt.Errorf("user %s: not found", id)
    }
    if err != nil {
        return nil, fmt.Errorf("GetUser: %w", err)
    }
    return toDomainUser(row), nil
}
```

**Шаг 6:** Удалить старые структуры с тегами `db:"..."`

### 11. Динамические запросы

sqlc не поддерживает запросы с переменным набором условий (динамический WHERE, IN с произвольным числом элементов).

**Стратегии внутри репозитория:**

**Вариант 1 — pgx напрямую** (для единичных сложных запросов):
```go
func (r *UserRepository) SearchUsers(ctx context.Context, filter domain.UserFilter) ([]*domain.User, error) {
    db := r.getter.DefaultTrOrDB(ctx, r.pool)

    query := `SELECT id, name, email FROM users WHERE 1=1`
    args := []any{}
    i := 1

    if filter.Name != "" {
        query += fmt.Sprintf(" AND name ILIKE $%d", i)
        args = append(args, "%"+filter.Name+"%")
        i++
    }
    if filter.Status != "" {
        query += fmt.Sprintf(" AND status = $%d", i)
        args = append(args, filter.Status)
        i++
    }

    rows, err := db.Query(ctx, query, args...)
    // ...
}
```

**Вариант 2 — sqlc с умеренно широким фильтром** (COALESCE-трюк):
```sql
-- name: ListUsersByFilter :many
SELECT id, name, email FROM users
WHERE
    ($1::text IS NULL OR name ILIKE '%' || $1 || '%')
    AND ($2::text IS NULL OR status = $2)
ORDER BY created_at DESC;
```

Передавать `nil` (`*string`) если фильтр не нужен — sqlc корректно обработает nullable параметры.

### 12. Типичные ошибки и решения

#### "column does not exist"
```bash
Error: column "phone_number" does not exist in table "users"
```
**Причина:** Опечатка в имени или DDL в `migrations/` устарел.  
**Решение:** Проверить актуальность схемы в `migrations/`.

#### "syntax error at or near"
```sql
SELECT * FROM users WERE id = $1;  -- WERE вместо WHERE
```
**Решение:** Исправить синтаксис SQL.

#### "query parameter limit exceeded"
```sql
INSERT INTO orders (col1, ..., col15) VALUES ($1, ..., $15)
```
**Решение:** Увеличить `query_parameter_limit` в `sqlc.yaml` или разбить на несколько запросов.

#### sqlc генерирует `interface{}` вместо конкретного типа
**Причина:** sqlc не вывел тип для выражения (COALESCE, CASE и т.д.).  
**Решение:** Добавить явный CAST:
```sql
SELECT COALESCE(phone, '')::text AS phone FROM users;
```

#### `pgx.ErrNoRows` не перехватывается
**Причина:** Прямое сравнение `err == pgx.ErrNoRows` ломается при обёрнутых ошибках.  
**Решение:** Всегда использовать `errors.Is(err, pgx.ErrNoRows)`.

### 13. CI/CD интеграция

```yaml
# .gitlab-ci.yml
sqlc-check:
  stage: lint
  script:
    - go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
    - sqlc diff   # Проверка что сгенерированный код актуален
    - sqlc vet    # Дополнительные проверки запросов
```

**Pre-commit hook:**

```bash
#!/bin/bash
# .git/hooks/pre-commit
sqlc diff
if [ $? -ne 0 ]; then
    echo "sqlc code is out of date. Run 'make generate-sqlc'"
    exit 1
fi
```

### 14. Тестирование с sqlc

**Моки через интерфейс `Querier` (unit-тесты):**

```go
// Сгенерированный sqlc/db.go
type Querier interface {
    GetUser(ctx context.Context, id uuid.UUID) (User, error)
    ListUsers(ctx context.Context, arg ListUsersParams) ([]User, error)
}

// В тестах:
type mockQuerier struct {
    getUserFn func(ctx context.Context, id uuid.UUID) (sqlc.User, error)
}

func (m *mockQuerier) GetUser(ctx context.Context, id uuid.UUID) (sqlc.User, error) {
    return m.getUserFn(ctx, id)
}
```

**Интеграционные тесты (testcontainers):**

```go
func TestUserRepository_GetUser(t *testing.T) {
    pool := setupTestDB(t)  // запускает PostgreSQL в Docker
    repo := NewUserRepository(pool)

    user, err := repo.GetUser(context.Background(), testUserID)
    require.NoError(t, err)
    assert.Equal(t, "John", user.Name())
}
```

## Когда НЕ использовать sqlc

- **Динамические запросы с непредсказуемым числом условий** — использовать ручной pgx (см. раздел 11)
- **Сложная бизнес-логика в SQL** (оконные функции с переменной логикой) — лучше в Go-коде
- **Легаси-проект с ORM** — миграция с GORM требует переписывания всех запросов, взвесить трудозатраты

## Checklist для настройки sqlc в проекте

- [ ] Создать `sqlc.yaml` с правильными путями
- [ ] Убедиться что `migrations/` содержит актуальный DDL
- [ ] Настроить `overrides` для uuid, numeric и кастомных типов
- [ ] Создать директорию `queries/`
- [ ] Извлечь SQL в `.sql` файлы с аннотациями `-- name: ... :`
- [ ] Запустить `sqlc generate`
- [ ] Обновить репозитории: заменить ручной scan на `sqlc.New(db).Method()`
- [ ] Проверить все `err == pgx.ErrNoRows` → заменить на `errors.Is`
- [ ] Добавить `sqlc diff` в CI
- [ ] Добавить `generate-sqlc` в Makefile
- [ ] Не добавлять `sqlc/` в `.gitignore` — коммитить сгенерированный код

## Ссылки

- Документация: https://docs.sqlc.dev/
- Примеры: https://github.com/sqlc-dev/sqlc/tree/main/examples
- Playground: https://play.sqlc.dev/