---
name: pgx-txmanager
description: |
  Паттерны работы с PostgreSQL в Go через pgxpool (pgx v5) и txmanager (github.com/scarymovie/txmanager).
  Используй этот скилл при любых вопросах про: работу с БД в Go, pgxpool, транзакции, структуры с тегами db,
  CollectRows, GetQuerier, WithinTransaction, nullable-поля через pgtype, batch-запросы, миграции репозиториев на txmanager.
  Также триггерится на: "как сделать транзакцию", "как описать структуру для БД", "как использовать пул соединений".
---

# pgx + txmanager — паттерны работы с PostgreSQL

## Стек

- `github.com/jackc/pgx/v5` — драйвер PostgreSQL
- `github.com/jackc/pgx/v5/pgxpool` — пул соединений
- `github.com/scarymovie/txmanager` — менеджер транзакций поверх pgxpool

---

## Инициализация пула

Конфиг пула выносится в отдельную структуру — значения берутся снаружи (env, yaml, etc.),
ничего не захардкожено в самом конструкторе.

```go
import (
    "context"
    "time"
    "github.com/jackc/pgx/v5/pgxpool"
)

type PoolConfig struct {
    DSN                string
    MaxConns           int32
    MinConns           int32
    MaxConnLifetime    time.Duration
    MaxConnIdleTime    time.Duration
    HealthCheckPeriod  time.Duration
}

func NewPool(ctx context.Context, c PoolConfig) (*pgxpool.Pool, error) {
    cfg, err := pgxpool.ParseConfig(c.DSN)
    if err != nil {
        return nil, err
    }

    cfg.MaxConns          = c.MaxConns
    cfg.MinConns          = c.MinConns
    cfg.MaxConnLifetime   = c.MaxConnLifetime
    cfg.MaxConnIdleTime   = c.MaxConnIdleTime
    cfg.HealthCheckPeriod = c.HealthCheckPeriod

    return pgxpool.NewWithConfig(ctx, cfg)
}
```

Пример разумных значений по умолчанию — можно применять если конфиг не задан явно:

```go
func DefaultPoolConfig(dsn string) PoolConfig {
    return PoolConfig{
        DSN:               dsn,
        MaxConns:          25,
        MinConns:          5,
        MaxConnLifetime:   time.Hour,
        MaxConnIdleTime:   30 * time.Minute,
        HealthCheckPeriod: time.Minute,
    }
}
```

> `MaxConns` по умолчанию в pgxpool — 4, что почти всегда мало для продакшена.

---

## Структуры с тегами

pgx v5 **не делает автоматический маппинг по тегам** при обычном `Scan`.
`RowToStructByName` читает тег `db:"..."` — использовать его как стандарт.

```go
import "github.com/jackc/pgx/v5/pgtype"

type User struct {
    ID        int64            `db:"id"`
    Name      string           `db:"name"`
    Email     string           `db:"email"`
    Bio       pgtype.Text      `db:"bio"`        // nullable
    DeletedAt pgtype.Timestamp `db:"deleted_at"` // nullable
}

type Order struct {
    ID     int64   `db:"id"`
    UserID int64   `db:"user_id"` // FK → users.id
    Total  float64 `db:"total"`
}
```

Для полей с внешними связями (слайсы) — `db`-тег не нужен, они заполняются отдельно:

```go
type UserWithOrders struct {
    User
    Orders []Order // нет db-тега, заполняем руками
}
```

---

## CollectRows — основной способ получить список

```go
rows, err := pool.Query(ctx, `SELECT id, name, email, bio, deleted_at FROM users`)
if err != nil {
    return nil, err
}

users, err := pgx.CollectRows(rows, pgx.RowToStructByName[User])
// rows.Close() и rows.Err() — внутри CollectRows
```

Для одной записи:

```go
rows, err := pool.Query(ctx, `SELECT id, name, email, bio, deleted_at FROM users WHERE id = $1`, id)
if err != nil {
    return nil, err
}

user, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[User])
// вернёт pgx.ErrNoRows если записи нет
```

Кастомный маппер (когда нужна трансформация):

```go
users, err := pgx.CollectRows(rows, func(row pgx.CollectableRow) (User, error) {
    var u User
    err := row.Scan(&u.ID, &u.Name, &u.Email)
    return u, err
})
```

---

## Репозиторий — GetQuerier вместо прямого pool

`txmanager.GetQuerier(ctx, pool)` возвращает активную транзакцию из контекста, если она есть,
иначе — сам пул. Благодаря этому репозиторий не знает, работает ли он внутри транзакции.

```go
type UserRepository struct {
    pool *pgxpool.Pool
}

func (r *UserRepository) Create(ctx context.Context, name, email string) (int64, error) {
    q := txmanager.GetQuerier(ctx, r.pool)

    var id int64
    err := q.QueryRow(ctx,
        `INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id`,
        name, email,
    ).Scan(&id)
    return id, err
}

func (r *UserRepository) GetByID(ctx context.Context, id int64) (User, error) {
    q := txmanager.GetQuerier(ctx, r.pool)

    rows, err := q.Query(ctx,
        `SELECT id, name, email, bio, deleted_at FROM users WHERE id = $1`, id,
    )
    if err != nil {
        return User{}, err
    }
    return pgx.CollectOneRow(rows, pgx.RowToStructByName[User])
}
```

---

## Транзакция через WithinTransaction

```go
type UserService struct {
    tm       *txmanager.TxManager
    userRepo *UserRepository
    profRepo *ProfileRepository
}

func (s *UserService) CreateWithProfile(ctx context.Context, name, email, bio string) error {
    return s.tm.WithinTransaction(ctx, func(ctx context.Context) error {
        userID, err := s.userRepo.Create(ctx, name, email)
        if err != nil {
            return err
        }

        return s.profRepo.Create(ctx, userID, bio)
    })
    // при ошибке — автоматический rollback
    // при успехе — commit
}
```

Репозитории через `GetQuerier` автоматически подхватывают транзакцию из контекста —
никаких дополнительных изменений в них не нужно.

---

## Batch-вставка

```go
func (r *OrderRepository) BulkInsert(ctx context.Context, orders []Order) error {
    q := txmanager.GetQuerier(ctx, r.pool)

    batch := &pgx.Batch{}
    for _, o := range orders {
        batch.Queue(
            `INSERT INTO orders (user_id, total) VALUES ($1, $2)`,
            o.UserID, o.Total,
        )
    }

    results := q.SendBatch(ctx, batch)
    defer results.Close()

    for range orders {
        if _, err := results.Exec(); err != nil {
            return err
        }
    }
    return nil
}
```

> `SendBatch` доступен и на `pgxpool.Pool`, и на `pgx.Tx` — `GetQuerier` это учитывает.

---

## Связи: один ко многим

```go
func (r *UserRepository) GetWithOrders(ctx context.Context, userID int64) (UserWithOrders, error) {
    q := txmanager.GetQuerier(ctx, r.pool)

    rows, err := q.Query(ctx,
        `SELECT id, name, email, bio, deleted_at FROM users WHERE id = $1`, userID,
    )
    if err != nil {
        return UserWithOrders{}, err
    }

    user, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[User])
    if err != nil {
        return UserWithOrders{}, err
    }

    orderRows, err := q.Query(ctx,
        `SELECT id, user_id, total FROM orders WHERE user_id = $1`, userID,
    )
    if err != nil {
        return UserWithOrders{}, err
    }

    orders, err := pgx.CollectRows(orderRows, pgx.RowToStructByName[Order])
    if err != nil {
        return UserWithOrders{}, err
    }

    return UserWithOrders{User: user, Orders: orders}, nil
}
```

---

## Nullable-поля: pgtype

```go
// Чтение
if user.Bio.Valid {
    fmt.Println(user.Bio.String)
}

// Запись (NULL)
var bio pgtype.Text // Valid == false → NULL в БД

// Запись (значение)
bio := pgtype.Text{String: "developer", Valid: true}
```

---

## Шпаргалка: когда что использовать

| Задача | Инструмент |
|--------|-----------|
| Список записей | `CollectRows` + `RowToStructByName` |
| Одна запись | `CollectOneRow` + `RowToStructByName` |
| INSERT/UPDATE/DELETE | `GetQuerier(ctx, pool).Exec(...)` |
| Транзакция | `tm.WithinTransaction(ctx, fn)` |
| Несколько вставок за раз | `pgx.Batch` + `SendBatch` |
| Nullable колонка | `pgtype.Text / Timestamp / Numeric / ...` |
| Вложенные связи | отдельный запрос + ручная сборка |