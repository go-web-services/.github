# go-web-platform

**GitHub:** `github.com/go-web-services/go-web-platform`
**Module:** `github.com/go-web-services/go-web-platform`

## Role

Shared Go library imported by every service in the ecosystem. Provides common plumbing — Gin bootstrap, structured logging, request/response logging middleware, error serialization middleware, PostgreSQL transaction abstraction, inter-service HTTP client, env helpers, shared constants, and shared types. Services implement only business logic; everything else is delegated here.

## Place in architecture

| | |
|---|---|
| **Layer** | Shared — compiled Go library, **not a deployed service** |
| **Called by** | Every service in the ecosystem: go-gateway-template, go-service-user, go-service-event, go-integration-email, go-integration-minio |
| **Calls** | Nothing — this is the leaf dependency; no outbound HTTP, no DB |
| **Deployment** | Not deployed. Imported via `go get github.com/go-web-services/go-web-platform` |

If you are adding a new service to the ecosystem, your first dependency is this library. If you are changing this library, every service that imports it must be retested and re-deployed.

## Directory structure

```
shared/go-web-platform/
├── entrypoint/
│   └── platform.go            # SetupPlatform — single call wires all platform concerns
├── logger/
│   └── logger.go              # Logger interface + zerolog implementation
├── middleware/
│   ├── logging_middleware.go  # LoggingMiddleware — request/response logging
│   └── error_handler_middleware.go  # ErrorHandlerMiddleware — c.Error(err) → JSON
├── error/
│   ├── base_error.go          # BaseError struct
│   ├── errors.go              # Sentinel error vars
│   └── dto.go                 # ErrorResponseDTO, ValidationErrorResponseDTO
├── transport/http/
│   ├── response.go            # Ok(c, data) helper
│   ├── router.go              # AddPlatformRoutes — wires /health /ready /swagger
│   └── handler/
│       └── health_handler.go  # /health and /ready handlers
├── db/session/
│   ├── session.go             # Session interface + context helpers
│   └── postgres.go            # NewPostgres(pool) — pgxpool-backed implementation
├── utils/
│   └── utils.go               # GetEnv, SendRequest
├── constants/
│   └── constants.go           # Error code strings, environment names, header names
└── types/
    └── types.go               # ErrorCode, Environment, PaginationInputParams, PaginationOutputParams, LoggingConfig
```

## Key patterns

### Bootstrap

Call once in `main.go` before registering application routes:

```go
import (
    platform "github.com/go-web-services/go-web-platform/entrypoint"
    "github.com/go-web-services/go-web-platform/logger"
    mw       "github.com/go-web-services/go-web-platform/middleware"
)

func main() {
    cfg  := config.Load()
    logg := logger.NewLogger(cfg.App.Env)
    router := gin.New()

    platform.SetupPlatform(
        router,
        logg,
        db.Ping,                  // readiness func — nil = always ready
        mw.DefaultLoggingConfig(),
        cfg.App.Env,
    )

    api.SetupRoutes(router, ...)
    router.Run(":" + cfg.App.Port)
}
```

`SetupPlatform` registers: Gin release mode, panic recovery, `LoggingMiddleware`, `ErrorHandlerMiddleware`, `/health`, `/ready`, `/swagger/*any`.

### Error handling

Handlers **never** write error responses directly. They push to Gin context; `ErrorHandlerMiddleware` serializes.

```go
func (h *myHandler) CreateV1(c *gin.Context) {
    var payload dto.CreateInput
    if err := c.ShouldBindJSON(&payload); err != nil {
        _ = c.Error(err)  // validation errors serialized as ValidationErrorResponseDTO
        return
    }
    result, err := h.service.Create(c.Request.Context(), payload)
    if err != nil {
        _ = c.Error(err)  // *BaseError → status from BaseError.Status
        return
    }
    platformResponse.Ok(c, result)
}
```

### DB transaction pattern

Session is injected via constructor. Repos call `session.GetTxFromContext(ctx)` — picks up tx automatically or falls back to pool.

```go
// Service layer
func (s *svc) CreateWithProfile(ctx context.Context, ...) error {
    return s.session.Transaction(ctx, func(ctx context.Context) error {
        user, err := s.userRepo.Create(ctx, ...)
        if err != nil { return err }  // auto-rollback
        return s.profileRepo.Create(ctx, user.ID, ...)
        // auto-commit on nil
    })
}

// Repository layer
func (r *repo) Create(ctx context.Context, ...) (*domain.User, error) {
    tx := session.GetTxFromContext(ctx)
    if tx != nil {
        return scan(tx.QueryRow(ctx, query, args...))
    }
    return scan(r.pool.QueryRow(ctx, query, args...))
}
```

Nested `Transaction` calls are safe: inner calls participate in the outer tx; their `Commit`/`Rollback` are no-ops.

### Inter-service HTTP

```go
import "github.com/go-web-services/go-web-platform/utils"

var result MyResponseDTO
err := utils.SendRequest("POST", url, payload, &result, ginCtx)
```

Propagates request headers (including auth) from the incoming Gin context.

## API contracts

### Packages and import paths

| Package | Import path |
|---|---|
| Bootstrap | `github.com/go-web-services/go-web-platform/entrypoint` |
| Logger | `github.com/go-web-services/go-web-platform/logger` |
| Middleware | `github.com/go-web-services/go-web-platform/middleware` |
| Errors + DTOs | `github.com/go-web-services/go-web-platform/error` |
| HTTP response | `github.com/go-web-services/go-web-platform/transport/http` |
| DB transactions | `github.com/go-web-services/go-web-platform/db/session` |
| Utils | `github.com/go-web-services/go-web-platform/utils` |
| Constants | `github.com/go-web-services/go-web-platform/constants` |
| Types | `github.com/go-web-services/go-web-platform/types` |

### Error types

```go
// BaseError — domain error carrying HTTP status
type BaseError struct {
    Code    types.ErrorCode
    Message string
    Status  int
}

// Constructors
func NewError(code types.ErrorCode, msg string) *BaseError          // status 400
func NewErrorWithStatus(code, msg string, status int) *BaseError

// Sentinel errors (pre-built BaseError values)
var (
    ErrEntityNotFound        // 404  ENTITY_NOT_FOUND
    ErrInvalidRequestPayload // 400  INVALID_REQUEST_PAYLOAD
    ErrUnauthorized          // 401  UNAUTHORIZED_ERROR
    ErrInternalServerError   // 500  INTERNAL_SERVER_ERROR
    ErrForbidden             // 403  FORBIDDEN_ERROR
    ErrValidation            // 400  VALIDATION_ERROR
)
```

### Error response shapes

```json
// Standard error (ErrorResponseDTO)
{ "code": "ENTITY_NOT_FOUND", "message": "entity not found" }

// Validation error (ValidationErrorResponseDTO) — from ShouldBindJSON failures
{
  "code": "VALIDATION_ERROR",
  "message": "validation failed",
  "errors": [
    { "field": "email", "message": "email is required" }
  ]
}
```

### Platform routes (registered by SetupPlatform)

```
GET /health   → 200 {"status":"ok"}
GET /ready    → 200 {"status":"ok"} or 503 if readiness func returns error
GET /swagger/*any → Swagger UI
```

### Session interface

```go
type Session interface {
    Begin(ctx context.Context) (Session, error)
    Transaction(ctx context.Context, f func(context.Context) error) error
    Rollback() error
    Commit() error
    Context() context.Context
}

// Helpers
func NewPostgres(pool *pgxpool.Pool) Session
func GetTxFromContext(ctx context.Context) pgx.Tx        // nil if no tx
func GetOrCreateTx(ctx context.Context, pool *pgxpool.Pool) (pgx.Tx, error)
```

### utils.SendRequest signature

```go
func SendRequest(
    method  string,
    url     string,
    payload any,          // JSON-marshaled as body; nil = no body
    target  any,          // response JSON decoded into this
    ctx     *gin.Context, // for header propagation
) error
```

### types

```go
type PaginationInputParams struct {
    Page  int64 `json:"page,omitempty"`
    Limit int64 `json:"limit,omitempty"`
}

type PaginationOutputParams struct {
    Page       int64 `json:"page"`
    TotalPages int64 `json:"total_pages"`
    PerPage    int64 `json:"per_page"`
    Total      int64 `json:"total"`
}

type LoggingConfig struct { ... }  // see middleware.DefaultLoggingConfig()
```

### constants (error code strings)

```go
const (
    UnauthorizedError      = "UNAUTHORIZED_ERROR"
    ForbiddenError         = "FORBIDDEN_ERROR"
    NotFoundError          = "ENTITY_NOT_FOUND"
    InternalServerError    = "INTERNAL_SERVER_ERROR"
    InvalidRequestPayload  = "INVALID_REQUEST_PAYLOAD"
    ValidationError        = "VALIDATION_ERROR"
    EntityNotFound         = "ENTITY_NOT_FOUND"
)

const (
    Development = types.Environment("dev")
    Production  = types.Environment("production")
)
```

## Configuration

No env vars — this is a library, not a service. Consumers pass config through function arguments.

## Inter-service dependencies

None. This is the leaf dependency that all other services import.

## Extension guide

**Adding a new sentinel error:**
1. Open `error/errors.go`
2. Add: `var ErrMyError = NewErrorWithStatus("MY_ERROR_CODE", "human message", httpStatus)`

**Adding a new package:**
1. Create directory under `shared/go-web-platform/<pkg>/`
2. Use module path `github.com/go-web-services/go-web-platform/<pkg>`
3. Avoid importing sibling packages in ways that create import cycles — use `types` package for shared primitives

## Common pitfalls

- **Writing response in handler before calling `c.Error`** — `ErrorHandlerMiddleware` only fires if no response has been written yet. Always return after `c.Error(err)`, never write `c.JSON` on the error path.
- **Not returning after `c.Error`** — execution continues after `c.Error`; always `return` immediately after.
- **Bypassing tx context** — passing a plain `context.Background()` into a repo inside a `Transaction` call breaks the tx chain. Always thread the same `ctx` received by the service method.
- **Calling `NewError` with wrong status** — `NewError` defaults to 400. Use `NewErrorWithStatus` for 401/403/500.
- **Logging raw errors without structured fields** — use `log.Error("msg", err)` not `log.Error(err.Error())`.
