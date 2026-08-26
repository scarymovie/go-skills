---
name: docker-go
description: >
  Docker-образы и Compose для Go-проектов: multi-stage сборка, BuildKit-кеши, .dockerignore,
  выбор рантайм-базы (alpine или distroless), production/development окружения.
  Docker images and Compose files for Go services.
when_to_use: >
  Пишешь или ревьюишь Dockerfile и compose-файлы, ускоряешь сборку образа, готовишь деплой.
  Триггеры: "dockerfile", "docker", "docker-compose", "compose.yaml", "контейнер", "образ",
  "деплой", "production image", "сборка образа", "медленная сборка", "dockerignore",
  "distroless", "build image", "multi-stage".
---

# Docker для Go-проектов

Целевая версия: Go 1.27 (минимум 1.26). Compose — мажор v5.

## Структура

```
docker/
├── images/
│   ├── app/
│   │   └── Dockerfile.production
│   └── codegen/
│       └── Dockerfile
├── compose.production.yaml
└── compose.development.yaml
```

Плюс `.dockerignore` — в корне контекста сборки (здесь контекст — `app/`), не в `docker/`.

---

## Production Dockerfile (multi-stage)

```dockerfile
# syntax=docker/dockerfile:1
ARG GO_VERSION=1.27
ARG BUILDER_ALPINE=3.24
ARG RUNTIME_ALPINE=3.24

# --- Builder ---
FROM golang:${GO_VERSION}-alpine${BUILDER_ALPINE} AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -buildvcs=false \
    -ldflags="-s" \
    -o /go/bin/app ./cmd/app

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -buildvcs=false \
    -ldflags="-s" \
    -o /go/bin/migrate ./cmd/migration

# --- Runtime ---
FROM alpine:${RUNTIME_ALPINE}

RUN apk add --no-cache ca-certificates

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --link --from=builder /go/bin/app .
COPY --link --from=builder /go/bin/migrate .
COPY --link --from=builder /app/migrations ./migrations

USER appuser

EXPOSE 8080

CMD ["./app"]
```

### Теги базовых образов

- `golang:1.27` существует, но alpine-варианты собраны только для `-alpine3.24` и `-alpine3.23`.
  Поэтому `BUILDER_ALPINE` не свободен: `golang:1.27-alpine3.22` не существует, сборка упадёт
  на pull. Рантайм разведён в отдельный `RUNTIME_ALPINE` именно поэтому: там ограничения нет,
  теги `alpine:3.21`/`3.22`/`3.23` живы и обновляются — снятыми их не считать.
- `golang:latest` теперь на **trixie** (Debian 13), а не bookworm — ещё одна причина не брать
  плавающий тег в builder.

### Distroless вместо alpine

Бинарь с `CGO_ENABLED=0` статический, shell и пакетный менеджер ему не нужны:

```dockerfile
FROM gcr.io/distroless/static-debian13:nonroot

WORKDIR /app
COPY --link --from=builder /go/bin/app .
COPY --link --from=builder /app/migrations ./migrations

USER nonroot
CMD ["./app"]
```

Внутри уже есть `ca-certificates`, `/etc/passwd` с пользователем `nonroot` и полная база tzdata;
`RUN apk add` и `adduser` не нужны, поверхность атаки меньше. Цена: **нет shell** — `HEALTHCHECK`
через `wget`/`CMD-SHELL` не работает, и `docker exec sh` для отладки тоже. Здоровье проверять
снаружи (в compose — `test: ["CMD", "./app", "healthcheck"]`, если такая подкоманда есть)
либо оркестратором. Для отладки существует тег `:debug` с busybox.

### Если нужен Yandex Cloud CA-сертификат

Когда приложение подключается к Managed PostgreSQL или другим сервисам Yandex Cloud,
нужен их корневой сертификат. Добавить перед `USER appuser`:

```dockerfile
# --- Yandex Cloud CA (добавить перед USER appuser) ---
RUN wget -q "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
        -O /usr/local/share/ca-certificates/yandex-ca.crt && \
    update-ca-certificates
```

Если Yandex Cloud не используется — этот блок не нужен. В distroless этот способ не работает
(нет `update-ca-certificates`): клади PEM в образ и указывай приложению путь через `SSL_CERT_FILE`.

### GOMAXPROCS в контейнере

Ничего не делать: с Go 1.25 рантайм сам выставляет `GOMAXPROCS` по cgroup-лимиту и affinity mask
и перечитывает их на ходу, поэтому `go.uber.org/automaxprocs` больше не нужен (поведение
включается директивой `go 1.25` и выше в `go.mod`; при `go 1.24` оно выключено).

### BuildKit-кеши: где они ломаются

Без `--mount=type=cache` кеш компиляции живёт внутри слоя `RUN go build`, а его инвалидирует
`COPY . .` — при каждой правке кода компиляция идёт с нуля. Модулей это не касается:
`go mod download` стоит до `COPY . .` и переживает правки кода, инвалидируется только по
`go.mod`/`go.sum`; маунт на `/go/pkg/mod` нужен ради другого — после смены зависимостей
докачиваются только новые модули, а не весь набор. В образ ни один из кешей не попадает.

Первая ловушка: пути прибиты к `GOPATH=/go` и `HOME=/root` builder-стадии. Если в builder добавить
`USER` (или переопределить `GOPATH`/`HOME`), Go начнёт писать в `$HOME/.cache/go-build` нового
пользователя, а смонтированный `/root/.cache/go-build` останется пустым. Ошибки не будет —
сборка просто молча замедлится обратно. Проверять надо не глазами, а временем повторной сборки.

Для сборки от непривилегированного пользователя монтировать по фактическим путям и отдать права:
`--mount=type=cache,target=/home/build/.cache/go-build,uid=1000,gid=1000`.

Вторая ловушка: cache mount живёт в кеше локального демона. На эфемерном CI-раннере он пуст
при каждом запуске, и ускорения не будет вообще — там нужен persistent buildx-билдер либо
`--cache-to`/`--cache-from` во внешний реестр.

### .dockerignore

`COPY . .` тянет в образ весь контекст, включая локальные бинарники и рабочий `config.yaml`
с реальными паролями — он ляжет в слой. Файл кладётся в корень контекста, при `context: ../app`
это `app/.dockerignore`:

```gitignore
bin/
config.yaml
*.local.yaml
*.md
```

Список короткий именно потому, что контекст — подкаталог `app/`: `.git`, `.idea/` и `docker/`
лежат выше и в контекст не попадают вовсе; расширишь контекст до корня репозитория — добавляй их.

Кроме безопасности это скорость: исключённые файлы не уезжают демону, и слой `COPY . .`
не инвалидируется правкой того, что в сборке не участвует.

### -buildvcs=false: зачем на самом деле

Дефолт — `auto`. Если `.git` в контекст не попал (при контексте `app/` он и не попадает —
`.git` лежит выше), сборка проходит **успешно**, просто не проставляет VCS-штамп в
`debug.ReadBuildInfo()`. Ошибки тут нет и флаг не нужен.

Жёсткая ошибка `error obtaining VCS status: exit status 128` возникает в другом случае:
`.git` в контексте **есть**, git в образе установлен, но каталог принадлежит другому
пользователю — git ругается на dubious ownership и отказывается читать репозиторий.
В Docker это ровно типичная ситуация (копирование меняет владельца, сборка идёт не от того uid).
Вот тогда и нужен `-buildvcs=false` — либо `git config --global --add safe.directory /app`,
если штамп нужен.

В шаблоне выше флаг стоит именно как страховка от этого отказа: терять нечего, версию
в бинарь всё равно кладём через `-X` (ниже).

### Почему каждый флаг здесь нужен

| Правило | Зачем |
|---|---|
| `CGO_ENABLED=0` | Статическая линковка, бинарник работает в `alpine`/`distroless`/`scratch` без libc |
| `-trimpath` | Убирает абсолютные пути из бинарника — безопасность + воспроизводимость |
| `-ldflags="-s"` | `-s` уже подразумевает `-w`, писать оба не нужно. Стектрейсы остаются (они из pclntab), теряются `.symtab` и DWARF — перестаёт работать Delve и symbol-resolving профайлеры |
| `--mount=type=cache` на обе директории | Иначе компиляция идёт с нуля после каждой правки кода, а модули перекачиваются целиком после каждой правки `go.mod` |
| `.dockerignore` | `COPY . .` иначе тащит `bin/` и локальный `config.yaml` с паролями внутрь образа |
| `COPY go.mod go.sum` перед `COPY . .` | Кеширование слоя с зависимостями — `go mod download` не перезапускается при изменении кода |
| `COPY --link` в финальной стадии | Слой не зависит от предыдущего состояния ФС: при смене базового образа копии переиспользуются, а стадии собираются параллельно |
| `GO_VERSION`/`BUILDER_ALPINE`/`RUNTIME_ALPINE` через ARG | Фиксированные версии — воспроизводимые сборки; builder и рантайм разведены, потому что набор alpine-тегов у них разный |
| `USER appuser` один раз в конце | Контейнер не работает от root, не переключать user туда-обратно |

### Версия приложения через build arg

```dockerfile
ARG VERSION=0.0.0

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -buildvcs=false \
    -ldflags="-s -X 'project/internal/version.Version=${VERSION}'" \
    -o /go/bin/app ./cmd/app
```

```bash
docker build --build-arg VERSION=$(git describe --tags --always) ...
```

Здесь `git describe` считается на хосте, поэтому `.git` внутри образа не нужен. Compose ниже
передаёт `VERSION` через `build.args` — без этого `ARG` в Dockerfile аргумент до сборки не дойдёт.

---

## Codegen Dockerfile

Предпочтительный способ — не Docker: `go get -tool github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen`
и вызов через `go tool oapi-codegen`, версия фиксируется в `go.mod` (см. скилл `openapi-codegen`).
Отдельный образ нужен только там, где Go не установлен — например в CI-раннере без тулчейна.

```dockerfile
# syntax=docker/dockerfile:1
ARG GO_VERSION=1.27
ARG BUILDER_ALPINE=3.24

FROM golang:${GO_VERSION}-alpine${BUILDER_ALPINE}

ARG OAPI_CODEGEN_VERSION=v2.8.0

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@${OAPI_CODEGEN_VERSION}

WORKDIR /app

ENTRYPOINT ["oapi-codegen"]
```

`@latest` здесь запрещён: образ, собранный в разное время, даст разный сгенерированный код —
и диффы поедут в PR, которые кодоген не трогали. Версия в ARG и меняется осознанно.

---

## compose.production.yaml

Имя файла: Compose сам ищет `compose.yaml`, это предпочтительное имя; окружения разводить
суффиксом (`compose.production.yaml`) и передавать через `-f`. Расширение `.yaml`, не `.yml`.
Ключ `version:` объявлен устаревшим — Compose его игнорирует и предупреждает; не добавлять.

```yaml
name: project

services:
  app:
    image: registry.example.com/project:${TAG}
    build:
      context: ../app
      dockerfile: ../docker/images/app/Dockerfile.production
      args:
        VERSION: ${VERSION}
    container_name: project-app
    restart: always
    volumes:
      - /var/project/config.yaml:/app/config.yaml:ro
    networks:
      - internal
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    logging:
      options:
        max-size: 10m
        max-file: "3"

  migrate:
    image: registry.example.com/project:${TAG}
    container_name: project-migrator
    command: ["./migrate", "up"]
    volumes:
      - /var/project/config.yaml:/app/config.yaml:ro
    networks:
      - internal
    restart: "no"
    logging:
      options:
        max-size: 10m
        max-file: "3"

networks:
  internal:
    name: project_internal
```

У `migrate` нет `depends_on`: в production БД внешняя (Managed PostgreSQL или отдельный хост),
сервиса `postgres` в этом файле нет, а `depends_on` на несуществующий сервис — ошибка валидации
Compose, файл просто не запустится. Ждать готовности внешней БД нечем: это делает сам мигратор
ретраями подключения на старте. Если БД всё же поднимается этим же файлом — добавить сервис
`postgres` с `healthcheck` и тогда уже `depends_on: {postgres: {condition: service_healthy}}`.

### Почему compose устроен так

| Правило | Зачем |
|---|---|
| `healthcheck` на app | Позволяет другим сервисам ждать реальной готовности через `condition: service_healthy` |
| `depends_on` только на сервисы этого файла, и всегда с `condition: service_healthy` | Ссылка на отсутствующий сервис роняет валидацию; локальный postgres надо ждать по healthcheck, а не по старту контейнера; внешнюю БД ждёт ретрай в приложении |
| Volumes с `:ro` | Config монтируется read-only — контейнер не может его перезаписать |
| `restart: "no"` для migrate | Мигратор выполняется один раз и завершается |
| `max-file: "3"` в кавычках | Compose ожидает строку, без кавычек значение станет числом |

---

## compose.development.yaml

```yaml
name: project-development

services:
  postgres:
    image: postgres:18-alpine
    container_name: project-postgres-dev
    restart: unless-stopped
    environment:
      POSTGRES_USER: db_user
      POSTGRES_PASSWORD: db_password
      POSTGRES_DB: db_database
    networks:
      - internal
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U db_user -d db_database"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  internal:
    name: project_internal

volumes:
  postgres_data:
```

Только инфраструктурные зависимости. Приложение запускается локально через `go run`.

Мажорную версию Postgres менять нельзя на живом томе: 18-й сервер не поднимется на каталоге
данных 17-го. Для dev проще удалить том (`docker compose down -v`) и накатить миграции заново.

---

## Антипаттерны

```dockerfile
# Плохо: сборка без CGO_ENABLED=0 — динамический бинарь не запустится в distroless/scratch
RUN go build -o /go/bin/app ./cmd/app

# Плохо: alpine:latest — невоспроизводимые сборки
FROM alpine:latest

# Плохо: переключение USER туда-обратно
USER appuser
WORKDIR /app
USER root
RUN ...
USER appuser

# Плохо: git в production builder (если не нужен для приватных модулей)
RUN apk add --no-cache git

# Плохо: миграция собирается без флагов оптимизации, а основное приложение — с ними
RUN go build -o /go/bin/migrate ./cmd/migration

# Плохо: cache mount добавлен, а USER в builder-стадии сменён — кеш молча не работает
USER build
RUN --mount=type=cache,target=/root/.cache/go-build go build ./cmd/app
```

```yaml
# Плохо: migrate зависит от app вместо БД
depends_on:
  app:
    condition: service_healthy

# Плохо: depends_on на сервис, которого нет в этом compose-файле
depends_on:
  postgres:
    condition: service_healthy
```
