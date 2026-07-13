---
name: pgx-txmanager
description: |
  Паттерны работы с PostgreSQL в Go через pgxpool (pgx v5) и avito-tech/go-transaction-manager.
  Используй этот скилл при любых вопросах про: работу с БД в Go, pgxpool, транзакции, структуры с тегами db,
  CollectRows, DefaultTrOrDB, Do (транзакция), nullable-поля через pgtype, batch-запросы, миграции репозиториев на txmanager.
  Также триггерится на: "как сделать транзакцию", "как описать структуру для БД", "как использовать пул соединений".
---

# pgx + go-transaction-manager — паттерны работы с PostgreSQL

## Стек

- `github.com/jackc/pgx/v5` — драйвер PostgreSQL
- `github.com/jackc/pgx/v5/pgxpool` — пул соединений
- `github.com/avito-tech/go-transaction-manager/trm/v2` — ядро менеджера транзакций
- `github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2` — адаптер для pgx v5

### Установка

```bash
go get github.com/avito-tech/go-transaction-manager/trm/v2
go get github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2
```

---

## Инициализация пула

```go
import (
    "context"
    "time"
    "github.com/jackc/pgx/v5/pgxpool"
)

type PoolConfig struct {
    DSN               string
    MaxConns          int32
    MinConns          int32
    MaxConnLifetime   time.Duration
    MaxConnIdleTime   time.Duration
    HealthCheckPeriod time.Duration
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

Разумные значения по умолчанию:

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

## Устойчивость подключения на старте (retry + backoff)

`pgxpool.NewWithConfig` **не подключается сразу** — пул создаётся лениво (prefill и
health-check уходят в фоновую горутину), фактический дозвон форсирует **первый `Ping()`**
(или первый запрос). Значит единственная «жадная» точка отказа — стартовый `Ping`.

- **В рантайме** пул самовосстанавливается: при следующем `Acquire` он передозванивается,
  а мёртвые соединения выбрасываются (`PrepareConn`/`AfterRelease`, см. ниже). Специальная
  обработка тут не нужна.
- **На старте** без повтора один временный `connection refused` (failover БД, под
  стартовал раньше базы) роняет процесс. Чинить нужно **только старт**.

Две вещи: (1) выставить таймаут одного дозвона — без него pgx подставляет дефолт
**2 минуты**, и «чёрная дыра» повесит воркер надолго; (2) повторять `Ping` с ограниченным
backoff'ом — переждать блип, но не висеть вечно (бюджет исчерпан → выходим, оркестратор
перезапустит под — прежнее поведение для реально лежащей БД).

```go
const (
    connectDialTimeout    = 5 * time.Second  // потолок одного дозвона (вместо дефолтных 2 мин)
    connectInitialBackoff = 1 * time.Second
    connectMaxBackoff     = 5 * time.Second
    connectMaxWait        = 90 * time.Second // общий бюджет; обычный failover укладывается
)

func InitDB(/* cfg, logger */) (*pgxpool.Pool, error) {
    poolCfg, err := pgxpool.ParseConfig(dsn)
    // ... MaxConns/MinConns/MaxConnLifetime/... ...
    poolCfg.ConnConfig.ConnectTimeout = connectDialTimeout

    // NewWithConfig не дозванивается; Background — чтобы фоновый prefill/health-check
    // не отменился, когда InitDB вернётся (не передавай сюда короткий ctx с defer cancel).
    pool, err := pgxpool.NewWithConfig(context.Background(), poolCfg)
    if err != nil {
        return nil, err
    }
    if err := pingWithRetry(pool, logger); err != nil {
        pool.Close()
        return nil, err
    }
    return pool, nil
}

func pingWithRetry(pool *pgxpool.Pool, logger *scarylog.Logger) error {
    deadline := time.Now().Add(connectMaxWait)
    delay := connectInitialBackoff
    for attempt := 1; ; attempt++ {
        ctx, cancel := context.WithTimeout(context.Background(), connectDialTimeout)
        err := pool.Ping(ctx)
        cancel()
        if err == nil {
            return nil
        }
        if !isRetryableConnErr(err) { // неверный пароль/БД — падаем сразу, не жжём бюджет
            return err
        }
        if time.Now().Add(delay).After(deadline) {
            return err // бюджет исчерпан
        }
        logger.Warn("db connect failed, retrying", "attempt", attempt, "delay", delay, "error", err)
        time.Sleep(delay)
        delay = min(delay*2, connectMaxBackoff)
    }
}
```

**Классификатор временных ошибок** — повторяем только сеть/дозвон; на ошибках конфигурации
(логин/пароль/имя БД) падаем сразу, иначе бюджет 90с сгорит на опечатке в пароле:

```go
func isRetryableConnErr(err error) bool {
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) {
        switch pgErr.Code {
        case "28P01", "28000", "3D000", "3F000": // пароль / authorization / БД / схема
            return false
        }
        return strings.HasPrefix(pgErr.Code, "08") || pgErr.Code == "57P03" // conn_exception / cannot_connect_now
    }
    var connErr *pgconn.ConnectError // оборачивает дозвон, в т.ч. "connection refused"
    if errors.As(err, &connErr) {
        return true
    }
    var netErr net.Error
    if errors.As(err, &netErr) {
        return true
    }
    return errors.Is(err, context.DeadlineExceeded)
}
```

> Ошибка вида `failed to connect to user=… database=…: … connect: connection refused` —
> это `*pgconn.ConnectError` (форматируется именно так). Ловится через `errors.As`.

**Гигиена соединений в пуле** — выбрасывает мёртвые соединения (напр. после failover'а)
на каждом `Acquire`, поэтому длинный `MaxConnLifetime` безопасен (pgx v5.9+):

```go
poolCfg.PrepareConn = func(ctx context.Context, conn *pgx.Conn) (bool, error) {
    if conn.IsClosed() {
        return false, nil // false = выбросить соединение, пул создаст новое
    }
    if err := conn.Ping(ctx); err != nil {
        return false, nil
    }
    return true, nil
}
poolCfg.AfterRelease = func(conn *pgx.Conn) bool {
    return !conn.IsClosed()
}
```

---

## Инициализация менеджера транзакций

```go
import (
    "github.com/avito-tech/go-transaction-manager/trm/v2/manager"
    trmpgxv5 "github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2"
)

trManager := manager.Must(trmpgxv5.NewDefaultFactory(pool))
```

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
    UserID int64   `db:"user_id"`
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

---

## Репозиторий — DefaultTrOrDB вместо прямого pool

`trmpgxv5.DefaultCtxGetter.DefaultTrOrDB(ctx, pool)` возвращает активную транзакцию из контекста, если она есть,
иначе — сам пул. Благодаря этому репозиторий не знает, работает ли он внутри транзакции.

```go
import (
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
    trmpgxv5 "github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2"
)

type UserRepository struct {
    pool   *pgxpool.Pool
    getter *trmpgxv5.CtxGetter
}

func NewUserRepository(pool *pgxpool.Pool) *UserRepository {
    return &UserRepository{pool: pool, getter: trmpgxv5.DefaultCtxGetter}
}

func (r *UserRepository) Create(ctx context.Context, name, email string) (int64, error) {
    q := r.getter.DefaultTrOrDB(ctx, r.pool)

    var id int64
    err := q.QueryRow(ctx,
        `INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id`,
        name, email,
    ).Scan(&id)
    return id, err
}

func (r *UserRepository) GetByID(ctx context.Context, id int64) (User, error) {
    q := r.getter.DefaultTrOrDB(ctx, r.pool)

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

## Транзакция через Do

```go
import (
    "github.com/avito-tech/go-transaction-manager/trm/v2/manager"
    trmpgxv5 "github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2"
)

type UserService struct {
    trManager *manager.Manager
    userRepo  *UserRepository
    profRepo  *ProfileRepository
}

func (s *UserService) CreateWithProfile(ctx context.Context, name, email, bio string) error {
    return s.trManager.Do(ctx, func(ctx context.Context) error {
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

Вложенные транзакции (savepoint):

```go
return s.trManager.Do(ctx, func(ctx context.Context) error {
    checkErr(r.Save(ctx, u))

    return s.trManager.Do(ctx, func(ctx context.Context) error {
        u.Username = "new_username"
        return r.Save(ctx, u)
    })
})
```

Репозитории через `DefaultTrOrDB` автоматически подхватывают транзакцию из контекста —
никаких дополнительных изменений в них не нужно.

---

## Batch-вставка

```go
func (r *OrderRepository) BulkInsert(ctx context.Context, orders []Order) error {
    q := r.getter.DefaultTrOrDB(ctx, r.pool)

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

---

## Связи: один ко многим

```go
func (r *UserRepository) GetWithOrders(ctx context.Context, userID int64) (UserWithOrders, error) {
    q := r.getter.DefaultTrOrDB(ctx, r.pool)

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
| Устойчивый connect на старте | `pingWithRetry` + backoff, `ConnConfig.ConnectTimeout`, `isRetryableConnErr` |
| Список записей | `CollectRows` + `RowToStructByName` |
| Одна запись | `CollectOneRow` + `RowToStructByName` |
| INSERT/UPDATE/DELETE | `getter.DefaultTrOrDB(ctx, pool).Exec(...)` |
| Транзакция | `trManager.Do(ctx, fn)` |
| Вложенная транзакция | вложенный `trManager.Do(ctx, fn)` |
| Несколько вставок за раз | `pgx.Batch` + `SendBatch` |
| Nullable колонка | `pgtype.Text / Timestamp / Numeric / ...` |
| Вложенные связи | отдельный запрос + ручная сборка |
