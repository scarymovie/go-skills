---
name: pgx-txmanager
description: >
  Работа с PostgreSQL в Go через pgxpool (pgx v5) и avito-tech/go-transaction-manager:
  инициализация и устойчивый старт пула, структуры с тегами db, CollectRows,
  прозрачные транзакции через DefaultTrOrDB, propagation и savepoint, классификация
  ошибок, батч и CopyFrom, nullable через pgtype. PostgreSQL access in Go via pgxpool
  and go-transaction-manager.
when_to_use: >
  Триггеры: "как сделать транзакцию", "как описать структуру для БД", "как использовать
  пул соединений", "pgxpool", "CollectRows", "DefaultTrOrDB", "batch", "pgtype",
  "transaction", "connection pool", "savepoint".
---

# pgx + go-transaction-manager — паттерны работы с PostgreSQL

## Стек

Go 1.27 (минимум 1.26).

- `github.com/jackc/pgx/v5` — драйвер PostgreSQL, **не ниже v5.9.2**
- `github.com/jackc/pgx/v5/pgxpool` — пул соединений
- `github.com/avito-tech/go-transaction-manager/trm/v2` — ядро менеджера транзакций
- `github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2` — адаптер для pgx v5

Нижняя граница pgx не формальность: в v5.9.2 закрыта GHSA-j88v-2chj-qfwx — SQL-инъекция
через dollar-quoted литералы при simple protocol. В v5.9.1 до этого чинили порчу
результатов батча (регрессия v5.9.0). Актуальная на сейчас — v5.10.0.

Модуль trm версионируется отдельными тегами вида `trm/v2.0.4`, а не общим тегом
репозитория. В 2.0.4 из драйверов убрали фоновую горутину `awaitDone`: отмена контекста
больше **не** вызывает автоматический rollback из другой горутины (`pgx.Tx` не
потокобезопасен, и это приводило к панике на выполняющемся запросе). Rollback теперь
выдаётся после возврата из транзакционной функции.

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
	"github.com/jackc/pgx/v5/pgxpool"
	"time"
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

	cfg.MaxConns = c.MaxConns
	cfg.MinConns = c.MinConns
	cfg.MaxConnLifetime = c.MaxConnLifetime
	cfg.MaxConnIdleTime = c.MaxConnIdleTime
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

> Дефолт `MaxConns` в pgxpool — `max(4, runtime.NumCPU())`, а не просто 4: на любой
> многоядерной машине это число ядер. Задавать явно всё равно стоит — размер пула должен
> зависеть от того, сколько соединений выдержит база, а не от того, сколько ядер у пода.

---

## Устойчивость подключения на старте

`pgxpool.NewWithConfig` **не дозванивается** до базы: пул ленивый, prefill и health-check
уходят в фоновую горутину. Значит единственная точка, где недоступность базы видна на
старте, — первый `Ping` (или первый запрос). Без него приложение поднимается «здоровым»
при мёртвой базе, а оркестратор считает его готовым.

Отсюда два обязательных элемента инициализации:

- **`poolCfg.ConnConfig.ConnectTimeout`** — дефолтный таймаут дозвона в pgx равен
  **двум минутам**, и на blackhole-адресе воркер повиснет ровно на столько.
- **`Ping` с ретраем и экспоненциальным backoff под общим бюджетом** — переживает блип,
  но всё-таки выходит с ошибкой, если база действительно мертва, отдавая решение
  оркестратору. Ретраить только сетевые отказы: неверный пароль или отсутствующая база
  не починятся, и жечь на них весь бюджет незачем.

В рантайме всё это уже не нужно — пул самовосстанавливается на следующем `Acquire`.

Полная реализация (`InitDB`, `pingWithRetry`, классификатор `isRetryableConnErr`,
гигиена пула через `PrepareConn` и `AfterRelease`) — в
[`references/pool-startup.md`](references/pool-startup.md).

---

## Инициализация менеджера транзакций

```go
import (
	trmpgxv5 "github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2"
	"github.com/avito-tech/go-transaction-manager/trm/v2/manager"
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
	ID        int64              `db:"id"`
	Name      string             `db:"name"`
	Email     string             `db:"email"`
	Bio       pgtype.Text        `db:"bio"`        // nullable
	DeletedAt pgtype.Timestamptz `db:"deleted_at"` // nullable, колонка timestamptz
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
	trmpgxv5 "github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
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
	trmpgxv5 "github.com/avito-tech/go-transaction-manager/drivers/pgxv5/v2"
	"github.com/avito-tech/go-transaction-manager/trm/v2/manager"
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

### Вложенный Do переиспользует транзакцию, а не создаёт savepoint

Частое и дорогое заблуждение. Дефолтная propagation — `PropagationRequired`: вложенный
`Do` **присоединяется к уже открытой транзакции**. Никакого savepoint не создаётся, и
ошибка внутри вложенного вызова откатывает всё целиком, а не только вложенную часть.

```go
return s.trManager.Do(ctx, func(ctx context.Context) error {
	if err := r.Save(ctx, u); err != nil {
		return err
	}
	// Тот же самый Do — та же самая транзакция. Ошибка здесь откатит и Save выше.
	return s.trManager.Do(ctx, func(ctx context.Context) error {
		return r.SaveProfile(ctx, p)
	})
})
```

Настоящий savepoint надо просить явно — через `DoWithSettings` с
`trm.PropagationNested`. Тогда драйвер откроет вложенную транзакцию через `SAVEPOINT`,
и её откат не тронет внешнюю:

```go
nested := settings.Must(settings.WithPropagation(trm.PropagationNested))

return s.trManager.Do(ctx, func(ctx context.Context) error {
	if err := r.Save(ctx, u); err != nil {
		return err
	}
	// Ошибка внутри откатится до savepoint, внешняя транзакция продолжится.
	if err := s.trManager.DoWithSettings(ctx, nested, func(ctx context.Context) error {
		return r.SaveOptionalPart(ctx, p)
	}); err != nil {
		s.log.Warn("optional part skipped", "err", err)
	}
	return nil
})
```

Если не нужен именно частичный откат — не усложняй: обычного вложенного `Do` достаточно,
просто понимай, что он ничего не изолирует.

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

## Классификация ошибок

Возвращать `err` как есть из репозитория — значит заставить каждый вызывающий разбирать
строку ошибки. Два случая различаются на уровне типов и должны разбираться в репозитории.

```go
import (
	"errors"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"
)

var (
	ErrNotFound = errors.New("not found")
	ErrConflict = errors.New("already exists")
)

func (r *UserRepository) Create(ctx context.Context, name, email string) (int64, error) {
	q := r.getter.DefaultTrOrDB(ctx, r.pool)

	var id int64
	err := q.QueryRow(ctx,
		`INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id`, name, email,
	).Scan(&id)

	var pgErr *pgconn.PgError
	if errors.As(err, &pgErr) && pgErr.Code == "23505" { // unique_violation
		return 0, fmt.Errorf("create user %q: %w", email, ErrConflict)
	}
	if err != nil {
		return 0, fmt.Errorf("create user: %w", err)
	}
	return id, nil
}
```

`pgx.CollectOneRow` на пустой выборке отдаёт `pgx.ErrNoRows` — его тоже стоит перевести
в доменную `ErrNotFound`, чтобы usecase не зависел от драйвера:

```go
user, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[User])
if errors.Is(err, pgx.ErrNoRows) {
	return User{}, fmt.Errorf("user %d: %w", id, ErrNotFound)
}
```

Коды PostgreSQL, которые различают чаще всего: `23505` unique_violation,
`23503` foreign_key_violation, `23502` not_null_violation, `40001` serialization_failure
(единственный из перечисленных, который имеет смысл ретраить), `40P01` deadlock_detected.

---

## CopyFrom — массовая загрузка

`pgx.Batch` шлёт N отдельных INSERT одним раундтрипом. `CopyFrom` использует протокол
`COPY` и на больших объёмах быстрее на один-два порядка: разбор запроса происходит
один раз, а не N.

```go
func (r *OrderRepository) BulkInsert(ctx context.Context, orders []Order) (int64, error) {
	q := r.getter.DefaultTrOrDB(ctx, r.pool)

	return q.CopyFrom(ctx,
		pgx.Identifier{"orders"},
		[]string{"user_id", "total"},
		pgx.CopyFromSlice(len(orders), func(i int) ([]any, error) {
			return []any{orders[i].UserID, orders[i].Total}, nil
		}),
	)
}
```

Границы применимости: `CopyFrom` не умеет `ON CONFLICT` и не возвращает
сгенерированные id. Нужен upsert или `RETURNING` — грузи `CopyFrom` во временную
таблицу и делай `INSERT ... SELECT` из неё, либо оставайся на батче.

Батч по-прежнему уместен, когда запросы **разные** — это его смысл, а не объём.

---

## Шпаргалка: когда что использовать

| Задача | Инструмент |
|--------|-----------|
| Устойчивый connect на старте | `pingWithRetry` + backoff, `ConnConfig.ConnectTimeout`, `isRetryableConnErr` |
| Список записей | `CollectRows` + `RowToStructByName` |
| Одна запись | `CollectOneRow` + `RowToStructByName` |
| INSERT/UPDATE/DELETE | `getter.DefaultTrOrDB(ctx, pool).Exec(...)` |
| Транзакция | `trManager.Do(ctx, fn)` |
| Вложенный вызов в той же транзакции | вложенный `trManager.Do(ctx, fn)` |
| Частичный откат (savepoint) | `DoWithSettings` + `trm.PropagationNested` |
| Нет строки / дубль по unique | `errors.Is(err, pgx.ErrNoRows)` / `pgconn.PgError.Code == "23505"` |
| Массовая загрузка | `CopyFrom` |
| Несколько вставок за раз | `pgx.Batch` + `SendBatch` |
| Nullable колонка | `pgtype.Text / Timestamptz / Numeric / ...` |
| Вложенные связи | отдельный запрос + ручная сборка |
