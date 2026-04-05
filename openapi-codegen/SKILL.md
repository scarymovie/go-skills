---
name: openapi-codegen
description: Use this skill when generating server/client code from OpenAPI specs, setting up codegen infrastructure, or working with oapi-codegen. Triggers on: "generate server", "openapi codegen", "oapi-codegen", "generate from spec", "кодогенерация", "сгенерировать сервер".
---

# OpenAPI Code Generation

This skill describes the process of generating Go server code from OpenAPI specifications using oapi-codegen.

## Prerequisites

1. **Docker image for codegen** (`docker/images/codegen/Dockerfile`):
```dockerfile
FROM golang:1.26-alpine

RUN go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest

WORKDIR /app

ENTRYPOINT ["oapi-codegen"]
```

2. **Codegen config** (`docker/images/codegen/server.yaml`):
```yaml
generate:
  - types
  - echo-server
  - embedded-spec
```

3. **OpenAPI spec** in `api/*.yaml` following [openapi-rpc-conventions](../openapi-rpc-conventions/SKILL.md)

## Build Docker Image

```bash
docker build -t bonds-codegen-oapi:latest -f docker/images/codegen/Dockerfile docker/images/codegen/
```

This needs to be done once or when updating oapi-codegen version.

## Generate Server Code

Use the Makefile target:

```bash
make generate-server
```

This command:
1. Finds all `*.yaml` and `*.yml` files in `api/`
2. For each spec file (e.g., `api/bonds.yaml`):
   - Extracts base name: `bonds`
   - Creates output directory: `app/internal/infrastructure/http/bonds/`
   - Generates code: `app/internal/infrastructure/http/bonds/api/bonds.gen.go`
   - Uses package name from spec base name

## Generated Structure

```
app/internal/infrastructure/http/bonds/
├── api/
│   └── bonds.gen.go          # Generated types, interfaces, routes
├── handler/
│   └── handler.go            # Manual: ServerInterface implementation
└── mapper/
    └── mapper.go             # Manual: Nwk* ↔ domain conversions
```

### Generated Code (`api/bonds.gen.go`)

Contains:
- **Types**: All schemas from OpenAPI (Input/Output/Params/Nwk*)
- **ServerInterface**: Interface with methods for each operation
- **RegisterHandlers**: Function to register routes with Echo router
- **Embedded spec**: OpenAPI spec embedded as base64

Example ServerInterface:
```go
type ServerInterface interface {
    ListBonds(ctx echo.Context) error
    CalculateYTM(ctx echo.Context) error
    SetRateSchedule(ctx echo.Context) error
    GetRateSchedule(ctx echo.Context) error
}
```

## Implementation Pattern

### 1. Create Handler (`handler/handler.go`)

```go
package handler

import (
    "bonds/internal/domain/repository"
    "bonds/internal/infrastructure/http/bonds/api"
    "bonds/internal/infrastructure/http/bonds/mapper"
    
    "github.com/labstack/echo/v4"
)

type Handler struct {
    bondRepo   repository.BondRepository
    couponRepo repository.CouponRepository
    priceRepo  repository.PriceRepository
}

func NewHandler(
    bondRepo repository.BondRepository,
    couponRepo repository.CouponRepository,
    priceRepo repository.PriceRepository,
) *Handler {
    return &Handler{
        bondRepo:   bondRepo,
        couponRepo: couponRepo,
        priceRepo:  priceRepo,
    }
}

// Implement ServerInterface
func (h *Handler) ListBonds(ctx echo.Context) error {
    var input bonds.ListBondsInput
    if err := ctx.Bind(&input); err != nil {
        return ctx.JSON(400, bonds.NwkError{
            Code:    "VALIDATION_ERROR",
            Message: err.Error(),
        })
    }
    
    // Convert input to domain filter
    filter := mapper.ToRepositoryFilter(input.Filter)
    
    // Call repository
    domainBonds, total, err := h.bondRepo.List(ctx.Request().Context(), filter)
    if err != nil {
        return ctx.JSON(500, bonds.NwkError{
            Code:    "INTERNAL_ERROR",
            Message: "Failed to fetch bonds",
        })
    }
    
    // Convert domain to network DTOs
    nwkBonds := mapper.ToNwkBonds(domainBonds)
    
    output := bonds.ListBondsOutput{
        Total: total,
        Page:  input.Pagination.Page,
        Limit: input.Pagination.Limit,
        Items: nwkBonds,
    }
    
    return ctx.JSON(200, output)
}
```

### 2. Create Mapper (`mapper/mapper.go`)

```go
package mapper

import (
    "bonds/internal/domain/model"
    "bonds/internal/domain/repository"
    "bonds/internal/infrastructure/http/bonds/api"
)

func ToRepositoryFilter(params *bonds.BondFilterParams) repository.BondFilter {
    filter := repository.BondFilter{}
    
    if params != nil {
        if params.Type != nil && *params.Type == bonds.Ofz {
            isOFZ := true
            filter.IsOFZ = &isOFZ
        }
        if params.Currency != nil {
            filter.Currency = *params.Currency
        }
    }
    
    return filter
}

func ToNwkBonds(domainBonds []repository.BondWithNextCoupon) []bonds.NwkBond {
    result := make([]bonds.NwkBond, len(domainBonds))
    for i, b := range domainBonds {
        result[i] = ToNwkBond(b)
    }
    return result
}

func ToNwkBond(b repository.BondWithNextCoupon) bonds.NwkBond {
    nwk := bonds.NwkBond{
        Uid:                   b.UID,
        Figi:                  b.FIGI,
        Ticker:                b.Ticker,
        Name:                  b.Name,
        Isin:                  b.ISIN,
        Currency:              b.Currency,
        IsOfz:                 b.IsOFZ,
        FloatingCoupon:        b.FloatingCouponFlag,
        Amortization:          b.AmortizationFlag,
        Nominal:               b.Nominal,
        AciRub:                b.ACIValue,
        CouponQuantityPerYear: b.CouponQuantityPerYear,
    }
    
    // Optional fields
    if b.MaturityDate.Valid {
        dateStr := b.MaturityDate.Time.Format("2006-01-02")
        nwk.MaturityDate = &dateStr
    }
    
    return nwk
}
```

### 3. Wire in Application (`app/app.go`)

```go
import (
    bondsapi "bonds/internal/infrastructure/http/bonds/api"
    bondshandler "bonds/internal/infrastructure/http/bonds/handler"
)

func NewApplication(ctx context.Context, cfg *config.Config, log *slog.Logger) (*Application, error) {
    // ... init repos ...
    
    // Create handler
    handler := bondshandler.NewHandler(bondRepo, couponRepo, priceRepo)
    
    // Create Echo instance
    e := echo.New()
    
    // Register routes
    bondsapi.RegisterHandlers(e, handler)
    
    // Create http.Server from Echo
    server := &http.Server{
        Addr:    cfg.HTTP.Addr,
        Handler: e,
    }
    
    return &Application{server: server, ...}, nil
}
```

## Error Handling

Always return `NwkError` for 400/500 responses:

```go
// Validation error (400)
return ctx.JSON(400, bonds.NwkError{
    Code:    "VALIDATION_ERROR",
    Message: "Invalid bond UID format",
})

// Not found (404)
return ctx.JSON(404, bonds.NwkError{
    Code:    "NOT_FOUND",
    Message: "Bond not found",
})

// Internal error (500)
return ctx.JSON(500, bonds.NwkError{
    Code:    "INTERNAL_ERROR",
    Message: "Database connection failed",
})
```

## Makefile Target

The `generate-server` target in Makefile:

```makefile
GEN_IMAGE_OAPI := bonds-codegen-oapi
GEN_TAG := latest
API_DIR := api
OUT_DIR = app/internal/infrastructure/http

generate-server:
	@set -e; \
	CONFIG="docker/images/codegen/server.yaml"; \
	SPECS="$$(find $(API_DIR) -name '*.yaml' -o -name '*.yml')"; \
	if [ -z "$$SPECS" ]; then \
		echo "No OpenAPI specs found in $(API_DIR)"; \
		exit 1; \
	fi; \
	for spec in $$SPECS; do \
		name=$$(basename $$spec | sed -E 's/\.ya?ml$$//'); \
		out_dir=$(OUT_DIR)/$$name; \
		echo ">>> Generating server code for $$name into $$out_dir..."; \
		mkdir -p $$out_dir/api; \
		docker run --rm \
			-u $$(id -u):$$(id -g) \
			-v "$$(pwd)":/app \
			-w /app \
			$(GEN_IMAGE_OAPI):$(GEN_TAG) \
			-config "$$CONFIG" \
			-o "$$out_dir/api/$$name.gen.go" \
			-package bonds \
			"$$spec"; \
	done
```

## Workflow Summary

1. **Write OpenAPI spec** in `api/bonds.yaml` following conventions
2. **Generate code**: `make generate-server`
3. **Implement handler**: Create `handler/handler.go` implementing `ServerInterface`
4. **Create mapper**: Create `mapper/mapper.go` for type conversions
5. **Wire in app**: Register handlers in `app.go`
6. **Test**: Use `.http` files in `test/http/`

## When to Regenerate

Regenerate server code when:
- Adding new endpoints
- Changing request/response schemas
- Updating error responses
- Modifying parameter types

**Important**: Never edit generated `*.gen.go` files manually — they will be overwritten on next generation.

## Common Issues

### Issue: Package name mismatch
**Solution**: Ensure `-package` flag matches the desired package name in Makefile

### Issue: Types not found
**Solution**: Check that all `$ref` references in OpenAPI spec are valid

### Issue: Echo not found
**Solution**: Add `github.com/labstack/echo/v4` to `go.mod`:
```bash
go get github.com/labstack/echo/v4
```

### Issue: Generated code doesn't compile
**Solution**: Validate OpenAPI spec first:
```bash
docker run --rm -v "$(pwd)":/app -w /app \
  openapitools/openapi-generator-cli validate -i api/bonds.yaml
```
