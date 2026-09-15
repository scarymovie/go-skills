---
name: openapi-codegen
description: >
  Настройка oapi-codegen v2 и генерация Go-сервера из OpenAPI-спеки: схема конфига
  (ровно 7 ключей, `generate` — объект булевых полей), связка `std-http-server` +
  `strict-server`, установка через tool-директиву go.mod, раскладка сгенерированного
  кода по слоям, версии и незакрытая инъекция через `x-go-type-import`. Configuring
  oapi-codegen v2 and generating a Go server from an OpenAPI spec: config schema,
  std-http + strict server, go tool install, layering, version and security notes.
when_to_use: >
  Когда генерируешь сервер или клиент из спеки, правишь или чинишь конфиг кодогена,
  пишешь хендлер под сгенерированный интерфейс, обновляешь oapi-codegen. Триггеры:
  "oapi-codegen", "openapi codegen", "generate server", "generate from spec",
  "кодогенерация", "сгенерировать сервер", "strict-server", "std-http-server",
  "gen.go", "конфиг кодогена", "field not found in type codegen.GenerateOptions",
  "cannot unmarshal !!seq into codegen.GenerateOptions", "error parsing configuration
  style". Правила самой спеки — скилл `openapi-rpc-conventions`, раскладка каталогов —
  `go-project-structure`, форма HTTP-стека — `go-application-architecture`.
---

# Кодогенерация из OpenAPI через oapi-codegen

Go 1.27, ниже не поддерживается (политика версий — скилл `go-quality`); oapi-codegen v2.8.0 требует Go 1.25+.

Имена в примерах (`bonds`, `ListBonds`, `NwkBond`) — из демо-домена облигаций, подставляй
свои. Что не пример, а правило: пакет сгенерированного кода зовётся `api` и лежит в
каталоге `api/` — имя пакета и имя каталога должны совпадать, иначе импорт
`.../http/bonds/api` придётся обращать как `bonds.X`, и читатель каждый раз спотыкается.

## Конфиг: ровно 7 ключей верхнего уровня

Допустимы только `package`, `output`, `generate`, `output-options`, `compatibility`,
`import-mapping`, `additional-imports`. `package` пиши всегда, но не потому что без него
ошибка: `detectPackageName` отрабатывает до `Validate` и молча выводит имя из пакета каталога
`output`, а если Go-файлов там ещё нет — из имени файла спеки, так что `bonds.yaml` даёт
`package bonds` вместо `api`. Без `output` код уходит в stdout, то есть он тоже обязателен.
Декодер включает `KnownFields(true)`, поэтому любой лишний или опечатанный ключ — жёсткая
ошибка, а не молчаливое игнорирование.

```yaml
# docker/images/codegen/oapi-config/bonds.yaml
package: api
output: internal/infrastructure/http/bonds/api/bonds.gen.go
generate:
  std-http-server: true
  strict-server: true
  models: true
```

Путь в `output` считается от рабочего каталога вызова, а `go tool` резолвит пиннутую версию
по `go.mod` и потому запускается изнутри модуля. Модуль у нас в корне репозитория (скилл
`go-project-structure`), поэтому вызов идёт оттуда же и путь пишется от корня: сместил каталог
вызова — сместились все относительные пути.

`generate` — **объект булевых полей**, не список строк. Список тоже проходит, но по другой
ветке: это устаревший «старый» стиль конфига со своим набором ключей (`templates`,
`include-tags`, `exclude-schemas`, …) и своими псевдонимами целей (`types` вместо `models`,
`spec` вместо `embedded-spec`). Стиль определяется по файлу целиком, поэтому смесь
взрывается сразу двумя ошибками парсинга:

```
error parsing configuration style as old version or new version

error when parsing using old config version:
yaml: unmarshal errors:
  line 6: field output-options not found in type main.oldConfiguration

error when parsing using new config version:
yaml: unmarshal errors:
  line 4: cannot unmarshal !!seq into codegen.GenerateOptions
```

Лечение всегда одно: перевести файл целиком в новый стиль, `generate` — объектом.

Остальные четыре ключа по назначению: `output-options` — тонкая настройка генерируемого кода
(`skip-prune`, `nullable-type`, `name-normalizer`, `user-templates`); `compatibility` —
возврат исторического поведения (`always-prefix-enum-values`, `enable-auth-scopes-on-context`);
`import-mapping` — куда ведут `$ref` во внешние документы (ключ — путь или URL документа, не
JSON-pointer, иначе `Validate` ругнётся);
`additional-imports` — дописать импорт в `.gen.go`. Флаг `-package` в командной строке
перебивает конфиг, `-o` — наоборот, работает только если `output` в конфиге пуст.

## Девять генераторов серверов, одновременно допустим один

`std-http-server`, `chi-server`, `echo-server`, `echo5-server`, `gin-server`,
`gorilla-server`, `fiber-server`, `fiber-v3-server`, `iris-server`. Два и больше — ошибка
`only one server type is supported at a time`.

Дефолт проекта — `std-http-server`, потому что HTTP-стек у нас `net/http` + `http.ServeMux`
(владелец правила — `go-application-architecture`), и сторонний роутер тянуть не за чем.

`strict-server` в этом перечне не участвует: это **обёртка поверх выбранного сервера**, а не
сервер. Одна её включённость без сервера бессмысленна, зато сочетается с любым из девяти.

## Что даёт strict-server

Без обёртки хендлер получает голые `(w http.ResponseWriter, r *http.Request)` и сам
занимается разбором тела и сериализацией. С `strict-server: true` генерируются:

```go
type ListBondsRequestObject struct {
	Body *ListBondsJSONRequestBody
}

type ListBondsResponseObject interface {
	VisitListBondsResponse(w http.ResponseWriter) error
}

type ListBonds200JSONResponse ListBondsOutput
type ListBonds400JSONResponse NwkError
type ListBonds500JSONResponse NwkError

// StrictServerInterface represents all server handlers.
type StrictServerInterface interface {
	// (POST /bond/list)
	ListBonds(ctx context.Context, request ListBondsRequestObject) (ListBondsResponseObject, error)
}
```

То есть: типизированный `RequestObject` (тело уже распарсено), закрытый набор
`ResponseObject` — по одному типу на объявленный в спеке статус, интерфейс, где хендлер
возвращает `(Response, error)`, и автоматическая сериализация ответа с проставленным
`Content-Type` и кодом. Статус, не описанный в спеке, вернуть просто нечем — компилятор не
даст разойтись коду и контракту.

**Ошибку из хендлера возвращать нельзя.** Ненулевой `error` уходит в
`ResponseErrorHandlerFunc`, а он по умолчанию отвечает `http.Error` — plain text 500 мимо
`NwkError`. Ошибка домена должна стать `...400JSONResponse`/`...500JSONResponse` с `nil`
вторым значением; `error` оставляем на случаи, когда ответ уже нечем отдать.

## Установка: tool-директива go.mod, не Docker

```bash
go get -tool github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@v2.8.0
go tool oapi-codegen -config docker/images/codegen/oapi-config/bonds.yaml api/bonds.yaml
```

Обе команды — из корня репозитория, он же корень модуля. Если модуль живёт в подкаталоге,
каждая идёт через `go -C <dir>`, а пути в аргументах разворачиваются на `../`.

`go get -tool` добавляет в `go.mod` строку `tool github.com/...` и пиннит версию в `go.mod`
и `go.sum`. Значит версия генератора воспроизводима у всех и в CI, а `.gen.go` перестаёт
меняться от того, чей ноутбук его собрал. `@latest` в установке генератора — источник
неповторяемых диффов; пиннить обязательно.

```makefile
API_DIR := api
CFG_DIR := docker/images/codegen/oapi-config

generate-server:
	@set -e; \
	for spec in $(API_DIR)/*.yaml; do \
		name=$$(basename $$spec .yaml); \
		cfg=$(CFG_DIR)/$$name.yaml; \
		[ -f "$$cfg" ] || { echo "нет конфига $$cfg"; exit 1; }; \
		echo ">>> $$name"; \
		go tool oapi-codegen -config $$cfg $$spec; \
	done
```

Конфиг — по одному на спеку, потому что `package` и `output` живут внутри него; общего
конфига на все модули не бывает.

Docker нужен только там, где в раннере нет Go:

```dockerfile
FROM golang:1.27-alpine3.24
RUN go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@v2.8.0
ENTRYPOINT ["oapi-codegen"]
```

Версия дублируется в двух местах — при обновлении правь оба, иначе локальный и CI-код
разъедутся.

## Версия: минимум 2.7.2, и одна дыра не закрыта

В 2.7.2 закрыты две уязвимости инъекции в генерируемый код через поля спеки (GHSA-jwgf-6xff-4hvp
и GHSA-xrqw-w576-xgj2; CVE-идентификаторов у них нет). Всё, что ниже 2.7.2, — обновлять.

Третья, **GHSA-9c2f-gr95-7wqw** (инъекция через расширение `x-go-type-import`), **фикса не
имеет**. Отсюда рабочее правило: генератор запускается только по спекам из своего репозитория.
Спека, пришедшая от подрядчика, из внешнего реестра или из PR неизвестного автора, сначала
читается глазами на предмет `x-go-type-import` и только потом попадает в `make generate-server`.
Кодоген — это выполнение кода, а не парсинг данных.

Ломающие изменения 2.8.0, на которые натыкаются при обновлении:

- Путь спеки, заканчивающийся слэшем, теперь якорится: `/foo/` генерируется как `/foo/{$}`.
  Раньше такой паттерн в `ServeMux` ловил всё поддерево ниже, из-за чего перекрывающиеся
  ветки роняли регистрацию паникой. Касается только `std-http-server`.
- Scopes схем безопасности больше не эмитятся по умолчанию: пропали константы вида
  `BearerAuthScopes` и запись scopes в контекст запроса. Вернуть — `compatibility:
  enable-auth-scopes-on-context: true`, но опция помечена deprecated: плоский список scopes
  не выражает ни альтернативных схем, ни комбинаций, и авторизацию правильнее делать
  мидлварью валидации запроса.

## Раскладка и поток вызова

Каталоги — по `go-project-structure`; для одного HTTP-модуля это три пакета:

```
internal/infrastructure/http/bonds/
├── api/       # bonds.gen.go, пакет api — только генерация, руками не трогать
├── handler/   # реализация StrictServerInterface
└── mapper/    # Nwk*/*Params <-> domain
```

Поток строго `handler → usecase → repository`. Хендлер, дёргающий репозиторий напрямую,
выносит бизнес-логику в транспортный слой и делает её непереиспользуемой из воркера или
другого протокола, поэтому репозиторий в конструкторе хендлера — сигнал ошибки.

```go
package handler

import (
	"context"
	"errors"
	"fmt"

	"bonds/internal/infrastructure/http/bonds/api"
	"bonds/internal/infrastructure/http/bonds/mapper"
	"bonds/internal/usecase/bond"

	"github.com/scarymovie/scarylog/v2"
)

type Handler struct {
	bonds *bond.UseCase
}

func NewHandler(bonds *bond.UseCase) *Handler {
	return &Handler{bonds: bonds}
}

func (h *Handler) ListBonds(ctx context.Context, req api.ListBondsRequestObject) (api.ListBondsResponseObject, error) {
	items, total, err := h.bonds.List(ctx, mapper.ToBondFilter(req.Body))
	switch {
	case errors.Is(err, bond.ErrInvalidFilter):
		return api.ListBonds400JSONResponse{Code: "VALIDATION_ERROR", Message: err.Error()}, nil
	case err != nil:
		scarylog.FromContext(ctx).Error(fmt.Errorf("list bonds: %w", err))
		return api.ListBonds500JSONResponse{Code: "INTERNAL_ERROR", Message: "internal error"}, nil
	}

	return api.ListBonds200JSONResponse{
		Total: total,
		Items: mapper.ToNwkBonds(items),
	}, nil
}

var _ api.StrictServerInterface = (*Handler)(nil)
```

Строка `var _ api.StrictServerInterface = (*Handler)(nil)` стоит того: без неё расхождение
хендлера с перегенерированным интерфейсом всплывёт не здесь, а в месте связывания.

Маппер — единственное место, где `Nwk*` встречаются с доменом; в `usecase` и `domain` они не
проникают.

```go
package mapper

import (
	"bonds/internal/domain/api/filter"
	"bonds/internal/domain/model"
	"bonds/internal/infrastructure/http/bonds/api"

	openapi_types "github.com/oapi-codegen/runtime/types"
)

func ToBondFilter(in *api.ListBondsInput) filter.Bond {
	out := filter.Bond{}
	if in == nil {
		return out
	}
	if p := in.Pagination; p != nil {
		out.Page, out.Limit = p.Page, p.Limit
	}
	if f := in.Filter; f != nil {
		if f.Type != nil && *f.Type == api.Ofz {
			out.IsOFZ = new(true) // new(expr) — Go 1.26
		}
		if f.Currency != nil {
			out.Currency = *f.Currency
		}
	}
	return out
}

func ToNwkBonds(bonds []model.Bond) []api.NwkBond {
	out := make([]api.NwkBond, len(bonds))
	for i, b := range bonds {
		out[i] = ToNwkBond(b)
	}
	return out
}

func ToNwkBond(b model.Bond) api.NwkBond {
	nwk := api.NwkBond{
		Uid:    b.UID(),
		Ticker: b.Ticker(),
	}
	if d, ok := b.MaturityDate(); ok {
		nwk.MaturityDate = &openapi_types.Date{Time: d}
	}
	return nwk
}
```

Даты формата `date` приезжают как `openapi_types.Date` (обёртка над `time.Time` с маршалингом
в `2006-01-02`), а не как строка — форматировать вручную не надо.

Связывание — обычный `http.ServeMux`, обёртка надевается на хендлер, маршруты вешаются на mux:

```go
import (
	"net/http"

	"bonds/config"
	// пакет во всех модулях называется api — на месте связывания нужны алиасы
	bondsapi "bonds/internal/infrastructure/http/bonds/api"
	bondshandler "bonds/internal/infrastructure/http/bonds/handler"
	"bonds/internal/usecase/bond"
)

func buildServer(cfg config.HTTP, bondUC *bond.UseCase) *http.Server {
	mux := http.NewServeMux()

	// StrictServerInterface -> ServerInterface -> маршруты в mux.
	strict := bondsapi.NewStrictHandler(bondshandler.NewHandler(bondUC), nil)
	bondsapi.HandlerFromMux(strict, mux)

	return &http.Server{
		Addr:              cfg.Addr,
		Handler:           mux,
		ReadHeaderTimeout: cfg.ReadHeaderTimeout,
	}
}
```

Второй аргумент `NewStrictHandler` — `[]StrictMiddlewareFunc`, мидлварь уровня операции
(видит `operationID` и распарсенный запрос). Обычная HTTP-мидлварь `func(http.Handler)
http.Handler` навешивается снаружи на `mux`, как описано в `go-application-architecture`.

## Статусы: только 200, 400, 500

Перечень статусов закрыт и принадлежит скиллу `openapi-rpc-conventions`: `200` с `*Output`,
`400` и `500` с `NwkError`, 404 не используется. Следствие для кодогена одно: не дописывай
статус в спеку ради того, чтобы у `ResponseObject` появился нужный тип, — это правка
контракта, а не обход компилятора.

## Регенерация

Перегенерировать после любой правки спеки: новый эндпоинт, изменение схемы, новый статус,
смена типа поля. `*.gen.go` руками не правят — правка исчезнет при следующем прогоне; нужное
поведение добавляется в спеку либо в `handler`/`mapper`. Результат коммитится в репозиторий:
ревью видит эффект правки спеки, а сборка не зависит от доступности генератора.

## Что ломается чаще всего

| Симптом | Причина |
|---|---|
| В блоке `error parsing configuration style`: `field <x> not found in type codegen.GenerateOptions` | Опечатка в имени генератора либо ключ из старого стиля (`types`, `spec`) в новом конфиге |
| В том же блоке: `cannot unmarshal !!seq into codegen.GenerateOptions` | `generate` списком в файле, где есть новые ключи |
| В том же блоке: `field <x> not found in type codegen.OutputOptions` | Ключ `compatibility` (например `always-prefix-enum-values`) положен в `output-options` |
| `only one server type is supported at a time` | Два генератора сервера сразу; `strict-server` за сервер не считается |
| Сгенерировано `package bonds` вместо `api`, ошибки нет | В конфиге нет `package` — имя выведено из имени файла спеки |
| Пакет `api`, а в коде `bonds.X` | Имя пакета в `package:` разошлось с именем каталога в `output:` |
| 500 с plain-text телом вместо `NwkError` | Хендлер вернул ненулевой `error` вместо `...500JSONResponse` |
| Паника при регистрации маршрутов | Пути с завершающим слэшем в старой версии; лечится обновлением на 2.8.0 с якорем `{$}` |
| Типы из `$ref` во внешний файл продублировались | Не заполнен `import-mapping` для этого документа |

## Если проект уже на echo

Менять рабочий сервис ради смены роутера незачем. Меняется один ключ, `strict-server`
остаётся:

```yaml
generate:
  echo-server: true
  strict-server: true
  models: true
```

Отличия: интерфейс хендлера тот же `StrictServerInterface` с `(ctx, request) (Response, error)`
— переписывать хендлеры при миграции на `std-http-server` не придётся, меняется только
связывание. Вместо `HandlerFromMux(si, mux)` вызывается `RegisterHandlers(e, si)`, где `e` —
`*echo.Echo`. Для echo v5 генератор другой — `echo5-server`. Новый модуль в таком проекте
всё равно заводится на `std-http-server`.
