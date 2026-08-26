---
name: go-project-structure
description: >
  Раскладка каталогов Go-сервиса: код в app/, слои domain/usecase/infrastructure,
  все Dockerfile в docker/, спеки в api/. Отдельно — два места для интерфейсов
  репозиториев и критерий выбора между ними. Directory layout for a Go service and
  where repository interfaces belong.
when_to_use: >
  Когда создаёшь новый проект, раскладываешь каталоги, решаешь, куда положить новый
  файл или пакет, и где объявить интерфейс репозитория. Триггеры: "create go project",
  "scaffold", "project structure", "new service", "create file", "куда положить",
  "создай файл", "структура проекта", "где объявить интерфейс", "repository interface".
---

# Структура Go-проекта

Go 1.27 (минимум 1.26).

## Корень

```
project/
├── api/                  # OpenAPI/proto-спеки, source of truth
├── app/                  # Весь Go-код, включая go.mod и go.sum
├── docker/               # Все Dockerfile и compose-файлы (деплой, кодоген, тулинг)
├── test/                 # Тестовые артефакты (например, http/ с .http-файлами)
├── Makefile
└── .ci.yaml
```

- Go-код лежит в `app/`, а не в корне: корень принадлежит инфраструктуре репозитория,
  и модуль не должен тянуть её в сборку.
- `docker/` — для ВСЕХ Dockerfile, не только деплойных.
- `api/` хранит только спеки; сгенерированный по ним код лежит не здесь, а рядом со
  своей реализацией — в `app/internal/infrastructure/` (скилл `openapi-codegen`).

---

## `app/` — исходники

```
app/
├── cmd/
│   ├── app/              # Точка входа сервиса
│   └── migration/        # Точка входа раннера миграций
├── config/               # Структуры конфига и его загрузка
├── internal/
│   ├── app/              # Сборка приложения (DI, старт)
│   ├── domain/
│   │   ├── api/          # Общие входные типы usecase: filter, pagination, sort, search_result
│   │   │   └── filter/   # Дробится на подпапки, когда типов становится много
│   │   ├── model/        # Чистые доменные сущности, без внешних зависимостей
│   │   └── repository/   # Вариант A: интерфейсы, у которых несколько потребителей
│   ├── usecase/          # Бизнес-логика. Может иметь свои model/ и mapper/
│   │   └── order/
│   │       ├── order.go       # Вариант B: интерфейс объявлен здесь же, у потребителя
│   │       └── mapper/
│   ├── infrastructure/
│   │   ├── http/
│   │   │   └── <name>/   # Папка на серверный модуль, имя = имя yaml-файла в api/
│   │   │       ├── api/      # Сгенерированные интерфейсы сервера
│   │   │       ├── handler/  # HTTP-хендлеры, реализующие эти интерфейсы
│   │   │       └── mapper/   # Мапперы запросов и ответов
│   │   ├── clients/
│   │   │   └── <name>/   # Внешние HTTP-клиенты, внутренняя структура свободная
│   │   ├── postgres/     # Реализации репозиториев — для обоих вариантов
│   │   └── db/           # Подключение к базе
│   ├── version/
│   └── worker/
└── migrations/           # SQL-файлы миграций
```

- `usecase/` вместо «service»: имя точнее, внутри допустимы свои `model/` и `mapper/`.
- `<name>` в `infrastructure/http/<name>/` совпадает с именем yaml-спеки в `api/` —
  по имени папки сразу видно, какой контракт она реализует.

---

## Интерфейс репозитория живёт там, где его потребители

Два допустимых размещения, выбор — за автором сервиса.

**Один потребитель → интерфейс объявляет сам usecase-пакет**, тот, который его зовёт;
общего `domain/repository/` для него не заводим. Контракт остаётся ровно таким узким,
как нужно вызывающему, и ни один пакет не накапливает знание обо всех хранилищах
сервиса. Это «accept interfaces, return structs» в применении к слоям.

```go
// internal/usecase/order/order.go
package order

// Ровно те методы, которые зовёт этот usecase, и ни одного лишнего.
type orderRepo interface {
	ByID(ctx context.Context, id model.OrderID) (*model.Order, error)
	Save(ctx context.Context, o *model.Order) error
}

type UseCase struct {
	repo orderRepo
}

// Интерфейс неэкспортируемый, поэтому единственный способ подставить реализацию —
// конструктор: в internal/app сюда передаётся *postgres.OrderRepo.
func New(repo orderRepo) *UseCase {
	return &UseCase{repo: repo}
}
```

**Несколько потребителей → общий `domain/repository/`.** Один и тот же контракт,
размноженный по пакетам, расходится при первой же правке; в общем пакете он один.

Реализация в обоих случаях — `infrastructure/postgres/`, и она не импортирует пакет
с интерфейсом: соответствие проверяется там, где конкретный тип подставляется в
usecase, то есть в `internal/app`.

Переход дешёвый и односторонний: как только у интерфейса появляется второй
потребитель, он переезжает в `domain/repository/`, реализация не меняется.

---

## `docker/` — все Dockerfile и compose-файлы

```
docker/
├── images/
│   ├── app/              # Production-образ сервиса
│   ├── codegen/          # Контейнер кодогенерации
│   │   ├── Dockerfile
│   │   └── oapi-config/  # Конфиги oapi-codegen — рядом со своим Dockerfile
│   └── <other>/          # redis, nextjs и прочее — по подпапке на образ
├── compose.production.yaml
└── compose.development.yaml
```

Подпапка на образ; конфиги, которые нужны только этому Dockerfile, лежат в ней же —
чтобы образ переносился одним каталогом.

---

## `api/` — спеки

```
api/                      # Может содержать подпапки под pb/openapi
├── <name>.yaml           # OpenAPI-спека. Имя совпадает с infrastructure/http/<name>/
└── <name>.proto          # gRPC-спека. Имя совпадает с infrastructure/grpc/<name>/
```
