---
name: go-config
description: >
  Конфигурация Go-приложения одним YAML-файлом вместо env-переменных: типизированные
  структуры, строгий декодер, Validate на старте, секреты и рантайм-переключатели.
  Single YAML file config for Go apps: typed structs, strict decoding, startup validation,
  secrets, runtime switches.
when_to_use: >
  Создаёшь или правишь конфиг, добавляешь параметр или секцию, грузишь настройки, решаешь
  куда положить секрет или флаг, включающий этап работы.
  Триггеры: "config", "конфиг", "настройки", "yaml", "config.yaml", "загрузка конфига",
  "параметры приложения", "environment", "env", "секрет", "переключатель", "фича-флаг",
  "config loading", "app settings", "feature flag", "secret".
---

# Go Config: YAML-файлы вместо переменных окружения

## Принцип

Конфигурация хранится в `config.yaml` и загружается в типизированные Go-структуры.
Никаких `os.Getenv`, никаких env-переменных, никаких `.env` файлов. Единственное
исключение — секреты, см. раздел «Секреты».

**Почему:**
- Типизация — ошибки видны при старте, а не в рантайме когда `os.Getenv` вернёт пустую строку
- Вложенность — YAML естественно группирует параметры, env требует плоских имён вроде `APP_HTTP_SERVER_READ_TIMEOUT`
- Один файл — все настройки в одном месте, не размазаны по десяткам переменных
- Дефолты — лежат в `config.example.yaml` рядом с параметром, а не размазаны по `getEnvOrDefault` хелперам
- В контейнере `config.yaml` монтируется как volume

---

## Структура проекта

```
project/
├── config/
│   └── config.go              # Структуры, Validate и функция Load
├── go.mod
├── config.example.yaml        # Шаблон конфига с плейсхолдерами (коммитится в git)
└── config.yaml                # Локальный конфиг для разработки (в .gitignore)
```

Оба yaml лежат в корне модуля, рядом с `go.mod`: путь в `config.Load("config.yaml")`
относительный, а рабочий каталог процесса при `go run ./cmd/app` — именно корень модуля.

- `config.yaml` — рабочий конфиг, содержит реальные значения, **не коммитится** (добавить в `.gitignore`)
- `config.example.yaml` — шаблон с плейсхолдерами вместо секретов, **коммитится в git**
- При клонировании проекта: `cp config.example.yaml config.yaml` и заполнить реальные значения

В production `config.yaml` монтируется снаружи через docker volume:

```yaml
volumes:
  - /var/project/config.yaml:/app/config.yaml:ro
```

`/app` справа — `WORKDIR` внутри образа (скилл `docker-go`), а не каталог репозитория.

---

## Библиотека: go.yaml.in/yaml/v3, не gopkg.in/yaml.v3

`gopkg.in/yaml.v3` архивирован 01.04.2025 и не получает даже security-фиксов. Поддержку
забрала официальная YAML-организация, форк — `go.yaml.in/yaml/v3` (v3.0.5). API идентичен:
миграция = смена строки импорта, код не трогается.

Оговорка на будущее: в форке v1–v3 объявлены frozen legacy — только security-фиксы,
развитие идёт в v4 (сейчас rc). В v4 API ломающий, так что «просто сменить импорт»
работает при переезде на v3 и **не** сработает при переезде на v4.

---

## Структуры конфига

Каждая секция — отдельная структура. Корневая структура `Config` собирает всё вместе.

```go
package config

import (
	"errors"
	"fmt"
	"os"
	"strings"
	"time"

	"go.yaml.in/yaml/v3"
)

type Config struct {
	App        App        `yaml:"app"`
	HTTP       HTTP       `yaml:"http"`
	PostgreSQL PostgreSQL `yaml:"postgresql"`
}

type App struct {
	Name            string        `yaml:"name"`
	Env             string        `yaml:"env"`
	ShutdownTimeout time.Duration `yaml:"shutdownTimeout"`
}

type HTTP struct {
	Addr              string        `yaml:"addr"`
	ReadHeaderTimeout time.Duration `yaml:"readHeaderTimeout"`
	ReadTimeout       time.Duration `yaml:"readTimeout"`
	WriteTimeout      time.Duration `yaml:"writeTimeout"`
	IdleTimeout       time.Duration `yaml:"idleTimeout"`
}

type PostgreSQL struct {
	DSN               string        `yaml:"dsn"`
	MaxConns          int32         `yaml:"maxConns"`
	MinConns          int32         `yaml:"minConns"`
	MaxConnLifetime   time.Duration `yaml:"maxConnLifetime"`
	MaxConnIdleTime   time.Duration `yaml:"maxConnIdleTime"`
	HealthCheckPeriod time.Duration `yaml:"healthCheckPeriod"`
}
```

### Правила для структур

- Поля публичные — config не доменный объект, геттеры избыточны
- Теги `yaml:"camelCase"` — единый стиль в YAML-файле
- `time.Duration` — декодер парсит `30s`, `5m`, `1h` автоматически
- Вложенные структуры — `PostgreSQL`, `HTTP`, а не плоский список полей

---

## Загрузка: строгий декодер плюс Validate

Два отказа, которые обязаны случиться на старте, а не в рантайме: неизвестный ключ
(опечатка) и пустое обязательное значение.

```go
func Load(path string) (*Config, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, fmt.Errorf("open config: %w", err)
	}
	defer f.Close()

	dec := yaml.NewDecoder(f)
	dec.KnownFields(true) // опечатка в ключе — ошибка старта, а не тихий zero value

	var cfg Config
	if err := dec.Decode(&cfg); err != nil {
		return nil, fmt.Errorf("parse config: %w", err)
	}

	if err := cfg.Validate(); err != nil {
		return nil, fmt.Errorf("validate config: %w", err)
	}

	return &cfg, nil
}
```

`yaml.Unmarshal` не годится: у него нет `KnownFields`, и `shutdwonTimeout` вместо
`shutdownTimeout` он молча проигнорирует — сервис поднимется с нулевым таймаутом. Ровно от
этого класса ошибок скилл и уходит, отказываясь от `os.Getenv`. С `KnownFields(true)`
получаем `line 3: field shutdwonTimeout not found in type config.App`.

Одного декодера мало: отсутствующая секция целиком декодируется в zero value без ошибки,
и пустой обязательный DSN пройдёт молча. Отсюда `Validate` на корневом `Config`:

```go
func (c *Config) Validate() error {
	if c.App.Name == "" {
		return errors.New("app.name is required")
	}
	if c.HTTP.Addr == "" {
		return errors.New("http.addr is required")
	}
	if c.PostgreSQL.DSN == "" {
		return errors.New("postgresql.dsn is required")
	}
	if strings.Contains(c.PostgreSQL.DSN, "${") {
		return errors.New("postgresql.dsn: unresolved placeholder")
	}
	if c.PostgreSQL.MinConns > c.PostgreSQL.MaxConns {
		return fmt.Errorf("postgresql.minConns %d > maxConns %d", c.PostgreSQL.MinConns, c.PostgreSQL.MaxConns)
	}

	return nil
}
```

- Метод один, на корневом `Config` — единая точка, её невозможно забыть вызвать. Когда
  секция разрастается, у неё заводится свой `Validate`, вызываемый из корневого
- Проверяются обязательность и согласованность полей между собой. Доступность внешних
  систем не проверяется — это дело конструктора компонента, а не парсинга файла

Использование в `main.go`:

```go
cfg, err := config.Load("config.yaml")
if err != nil {
	fmt.Fprintln(os.Stderr, "load config:", err)
	os.Exit(1)
}

log := initLogger(cfg)
```

`panic` на старте формально разрешён (владелец правила — `go-code-style`: `panic` только
в `Must*`-обёртке, в тестовых фикстурах и на старте до приёма трафика), но здесь он даёт
бесполезный стектрейс: виновата строка в YAML, а не код. Поэтому сообщение и `os.Exit(1)`.
Пишем в `os.Stderr`, потому что конфиг грузится ДО логгера — логгер сам настраивается из
конфига; дальше по `main` ошибки старта идут через `log.Error(err)`, см.
`go-application-architecture`. `os.Exit` не выполняет `defer`, на этом шаге отложенных
вызовов ещё нет.

---

## config.example.yaml (коммитится в git)

```yaml
app:
  name: my-service
  env: development
  shutdownTimeout: 15s

http:
  addr: ":8080"
  readHeaderTimeout: 5s
  readTimeout: 10s
  writeTimeout: 30s
  idleTimeout: 120s

postgresql:
  dsn: "postgres://USER:PASSWORD@HOST:5432/DATABASE?sslmode=disable"
  maxConns: 25
  minConns: 5
  maxConnLifetime: 1h
  maxConnIdleTime: 30m
  healthCheckPeriod: 1m
```

`config.yaml` — тот же файл ключ в ключ, отличаются только значения: вместо плейсхолдеров
`USER`/`PASSWORD`/`HOST` реальные креды. Набор ключей обязан совпадать: с `KnownFields(true)`
ключ, которого нет в структуре, роняет старт, а секция, которую забыли перенести, молча
уезжает в zero value и ловится уже `Validate`.

---

## Секреты

Секреты лежат в смонтированном снаружи `config.yaml` (`:ro`, права `0600`, файла нет в
образе и нет в git). Отдельного механизма секретов скилл не вводит, и **исключений для env
здесь тоже нет**: приложение не читает `os.Getenv` ни для чего, включая пароли.

Хранилище секретов (Vault, docker secret, CI) при этом не мешает — оно просто работает на
шаг раньше. Подстановку делает деплой, отдавая контейнеру уже готовый файл: рендерит
`config.yaml` из шаблона перед стартом либо монтирует секрет файлом и собирает конфиг из
него. Приложение получает конфиг в единственном виде — как файл с реальными значениями, —
и не знает, кто и чем его заполнил.

Если деплой рендерит шаблон, в `config.yaml` до подстановки лежит
`postgres://user:${POSTGRES_PASSWORD}@host:5432/db`. Тогда `Validate` обязан проверять
`strings.Contains(dsn, "${")`: не подставилось — падаем на старте, а не на первом коннекте
с паролем `${POSTGRES_PASSWORD}`. Проверка стоит строчку и ловит целый класс тихих отказов.

Что здесь запрещено: env-оверрайд «любой ключ конфига можно переопределить переменной».
Он возвращает ровно ту невидимость, ради ухода от которой выбран YAML: чтобы узнать
действующее значение, приходится обойти файл, окружение контейнера и манифест деплоя.

---

## Рантайм-переключатели

Флаг, включающий разрушительный этап (миграция данных, чистка, массовая рассылка), обязан
быть **fail-closed**: отсутствующее значение читается как ВЫКЛЮЧЕНО. `KnownFields(true)`
ловит опечатку внутри известной секции, но не спасает, если секции нет вовсе или опечатка
в имени самой секции — она декодируется в zero value. Значит zero value обязан означать
«выключено»:

```go
type Cleanup struct {
	Enabled bool          `yaml:"enabled"` // нет ключа — false — этап не запускается
	Period  time.Duration `yaml:"period"`
}
```

Поэтому `Enabled bool`, а не `Disabled bool`: у второго zero value означает «включено», и
любая потеря ключа сама включает разрушительный этап.

Два разных уровня, их нельзя смешивать в одном флаге:

- **Компонент сконструирован** — читается один раз в `Load`/при сборке зависимостей
  (пул, слушающий сокет, горутина воркера). Меняется только рестартом. Сюда идут адреса,
  DSN, размеры пулов
- **Этап активен** — перечитывается на ходу, на каждой итерации цикла. Компонент при этом
  жив и сконструирован, он просто не делает работу. Сюда идут выключатели этапов

```go
type Worker struct {
	active atomic.Bool // «этап активен»: перечитывается на каждом тике
	do     func(context.Context) error
}

func (w *Worker) tick(ctx context.Context) error {
	if !w.active.Load() {
		return nil // выключено — тик пустой, воркер продолжает жить
	}

	return w.do(ctx)
}
```

Смешение уровней даёт две типичные аварии: выключатель, который на самом деле требует
рестарта (успели напортить, пока катится деплой), и адрес пула, который «перечитывается»,
но реально ни на что не влияет, потому что соединения уже открыты.

---

## Добавление новой секции

1. Создать структуру в `config.go`:

```go
type Redis struct {
	Addr     string `yaml:"addr"`
	Password string `yaml:"password"`
	DB       int    `yaml:"db"`
}
```

2. Добавить в `Config` поле `Redis Redis` с тегом `yaml:"redis"`.

3. Добавить обязательные поля секции в `Validate`.

4. Добавить секцию в `config.yaml` и `config.example.yaml` — именно в этом порядке, после
   правки структуры: с `KnownFields(true)` ключ, для которого ещё нет поля, роняет старт.

```yaml
redis:
  addr: "localhost:6379"
  password: ""
  db: 0
```

5. Использовать через `cfg.Redis.Addr` — типизированный доступ, автокомплит в IDE.

---

## Передача конфига в приложение

Целиком `*config.Config` доходит ровно до одного места — `NewApplication(cfg *config.Config,
log *scarylog.Logger)` (сигнатура и порядок сборки — в `go-application-architecture`).
Дальше по цепочке зависимостей идут секции, а не корневой конфиг:

```go
func NewPool(ctx context.Context, cfg config.PostgreSQL) (*pgxpool.Pool, error) {
	// ...
}

func NewServer(cfg config.HTTP, handler http.Handler) *http.Server {
	return &http.Server{
		Addr:              cfg.Addr,
		Handler:           handler,
		ReadHeaderTimeout: cfg.ReadHeaderTimeout, // обязателен, см. go-application-architecture
		ReadTimeout:       cfg.ReadTimeout,
		WriteTimeout:      cfg.WriteTimeout,
		IdleTimeout:       cfg.IdleTimeout,
	}
}
```

---

## Антипаттерны

```go
// Плохо: глобальная переменная конфига
var GlobalConfig Config

// Плохо: передача всего конфига туда, где нужна одна секция
func NewPool(cfg *config.Config) // нужен только cfg.PostgreSQL

func main() {
	// Плохо: os.Getenv — нет типизации, нет дефолтов, нет структуры
	port := os.Getenv("APP_PORT")
	if port == "" {
		port = "8080"
	}

	// Плохо: .env файлы и godotenv
	godotenv.Load(".env")
	os.Getenv("DB_HOST")

	// Плохо: конфиг через флаги для десятков параметров
	flag.String("db-host", "localhost", "")
	flag.Int("db-port", 5432, "")
	flag.String("db-user", "postgres", "")
	// ...20 строк флагов
}
```
