# go-service-event

**GitHub:** `github.com/go-web-services/go-service-event`
**Module:** `github.com/go-web-services/go-service-event`

## Role

Analytics event collection and query service. Stores structured events in PostgreSQL with full idempotency (project_id + message_id unique constraint). Supports single-event ingest, atomic batch ingest (up to 100 events per request), soft-delete, update, point lookup, and paginated filtering queries. Events are scoped to a `project_id`. go-gateway-template forwards frontend events here; backend services can also call it directly via `pkg/client`.

## Place in architecture

| | |
|---|---|
| **Layer** | Service — internal, not exposed to the internet |
| **Called by** | go-gateway-template (forwards frontend analytics events via `pkg/client`). Backend services can also call it directly. |
| **Calls** | Nothing — only uses go-web-platform library |
| **Own database** | PostgreSQL — dedicated instance, not shared with any other service |

The gateway acts as the public-facing proxy for analytics: the frontend sends events to `POST /api/v1/events/send` on the gateway, which enriches the payload with `user_id` (if authenticated) and forwards it here.

## Directory structure

```
service/go-service-event/
├── cmd/app/
│   └── main.go                          # Entry point
├── config/
│   ├── config.go                        # App config (port, env)
│   └── postgres_config.go               # PostgreSQL DSN from env
├── internal/
│   ├── clients/                         # Clients for other services (if needed)
│   ├── domain/
│   │   └── event.go                     # Event domain model
│   ├── repository/                      # PostgreSQL repository (uses db/session for tx)
│   ├── service/                         # Business logic, idempotency check
│   ├── transport/http/
│   │   ├── handler/
│   │   │   └── event_handler.go         # All HTTP handlers
│   │   └── router.go                    # Route registration
│   └── types/                           # Internal types
├── migrations/
│   └── 000001_create_events_table.sql   # Goose migration (up/down)
└── pkg/client/                          # Importable by other services
    ├── dto/
    │   └── dto.go                       # All DTOs (EventDTO, input/output types)
    └── service/
        └── event_api_service.go         # EventAPIService HTTP client
```

## Key patterns

- All endpoints accept/return JSON, use `POST` (except no auth middleware — caller decides auth).
- Validation via `go-playground/validator`. After `ShouldBindJSON`, call `validate.Struct(input)`.
- Error pattern: `c.Error(err)` → `ErrorHandlerMiddleware` serializes (from go-web-platform).
- Idempotency: `UNIQUE(project_id, message_id)` — duplicate ingest returns the existing row, not an error.
- Soft delete: `deleted_at` column; all indexes are partial (`WHERE deleted_at IS NULL`).
- Batch ingest is atomic (single PostgreSQL transaction via `db/session`).
- Swagger: regenerate with `swag init -g cmd/app/main.go -o docs --parseDependency --parseInternal`.

## API contracts

### EventDTO (response shape for all endpoints)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "project_id": "proj_live_01",
  "message_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "distinct_id": "anon_6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "user_id": "usr_42",              // nullable, omitted if null
  "session_id": "sess_01j8xyz",     // nullable, omitted if null
  "ip": "203.0.113.10",            // nullable, omitted if null
  "user_agent": "Mozilla/5.0 ...", // nullable, omitted if null
  "name": "page_viewed",
  "payload": { "path": "/home", "referrer": "https://google.com" },
  "occurred_at": "2026-04-12T10:30:00Z",
  "received_at": "2026-04-12T10:30:00.456Z",
  "deleted_at": null               // omitted if not deleted
}
```

### POST /api/v1/events/create

Idempotent. Same `project_id` + `message_id` returns the existing row.

```json
// Request
{
  "project_id": "proj_live_01",      // required
  "message_id": "7c9e6679-...",      // required, UUID — use new UUID per request for idempotent retries
  "distinct_id": "anon_abc123",      // required — stable analytics identity
  "name": "page_viewed",             // required, max 255 chars (whitespace trimmed)
  "payload": { "path": "/home" },    // required, JSON object
  "occurred_at": "2026-04-12T10:30:00Z",  // required, RFC3339
  "user_id": "usr_42",              // optional
  "session_id": "sess_xyz",         // optional
  "ip": "1.2.3.4",                  // optional
  "user_agent": "Mozilla/5.0 ..."   // optional
}
// Response
{ "event": { /* EventDTO */ } }
```

### POST /api/v1/events/create-batch

Atomic — all or nothing. Max 100 events per request. Duplicate `project_id+message_id` within the batch is rejected.

```json
// Request
{ "events": [ /* array of EventCreateInputDTO, same fields as single create */ ] }
// Response
{ "events": [ /* array of EventDTO in same order */ ] }
```

### POST /api/v1/events/update

```json
// Request
{
  "id": "550e8400-...",       // required, UUID
  "name": "new_name",         // optional
  "payload": { "key": "v" }, // optional
  "user_id": "usr_42",       // optional
  "session_id": "sess_xyz"   // optional
}
// Response
{ "event": { /* EventDTO */ } }
```

### POST /api/v1/events/delete

Soft delete — sets `deleted_at`, does not remove row.

```json
// Request
{ "id": "550e8400-..." }  // required, UUID
// Response
{ "message": "Event deleted successfully", "event": { /* EventDTO with deleted_at set */ } }
```

### POST /api/v1/events/detail

```json
// Request
{ "id": "550e8400-..." }  // required, UUID
// Response
{ "event": { /* EventDTO */ } }
```

### POST /api/v1/events/query

Paginated filtering. All filter fields are optional and use OR within a field, AND across fields. `names` uses substring ILIKE per term. All other list fields use exact match (SQL IN).

```json
// Request
{
  "ids":          ["550e8400-..."],      // optional, UUIDs
  "project_ids":  ["proj_live_01"],      // optional
  "distinct_ids": ["anon_abc123"],       // optional
  "names":        ["page_view"],         // optional, substring ILIKE
  "message_ids":  ["7c9e6679-..."],      // optional, UUIDs
  "user_ids":     ["usr_42"],            // optional
  "session_ids":  ["sess_xyz"],          // optional
  "ips":          ["1.2.3.4"],           // optional, exact
  "user_agents":  ["Mozilla/5.0"],       // optional, exact
  "page":  1,
  "limit": 20
}
// Response
{
  "events": [ /* array of EventDTO */ ],
  "meta": {
    "pagination": {
      "page": 1,
      "total_pages": 5,
      "per_page": 20,
      "total": 87
    }
  }
}
```

### pkg/client usage

```go
import eventClient "github.com/go-web-services/go-service-event/pkg/client/service"
import eventDTO    "github.com/go-web-services/go-service-event/pkg/client/dto"

client := eventClient.NewEventAPIService("http://go-service-event:8020")

result, err := client.CreateV1(ginCtx, eventDTO.EventCreateInputDTO{
    ProjectID:  "proj_live_01",
    MessageID:  uuid.New().String(),
    DistinctID: "user-stable-id",
    Name:       "signup_completed",
    Payload:    map[string]any{"plan": "pro"},
    OccurredAt: time.Now(),
})
```

## Configuration

| Env var | Type | Default | Description |
|---|---|---|---|
| `APP_PORT` | int | `8020` | Gin listen port |
| `APP_ENV` | string | `dev` | `dev` or `production` |
| `POSTGRES_USER` | string | `go-service-event` | DB user |
| `POSTGRES_PASSWORD` | string | `go-service-event-password` | DB password |
| `POSTGRES_HOST` | string | `host.docker.internal` | DB host |
| `POSTGRES_PORT` | string | `5437` | DB port |
| `POSTGRES_DB` | string | `go-service-event-db` | DB name |
| `POSTGRES_SSL_MODE` | string | `disable` | `disable` or `require` |

## Database schema

```sql
CREATE TABLE events (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id  TEXT NOT NULL,
    message_id  TEXT NOT NULL,
    distinct_id TEXT NOT NULL,
    user_id     TEXT,
    session_id  TEXT,
    ip          TEXT,
    user_agent  TEXT,
    name        TEXT NOT NULL,
    payload     JSONB NOT NULL DEFAULT '{}',
    occurred_at TIMESTAMPTZ NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ,
    CONSTRAINT events_project_message_unique UNIQUE (project_id, message_id)
);
-- Partial indexes (WHERE deleted_at IS NULL) on:
-- (project_id, occurred_at DESC)
-- (project_id, distinct_id, occurred_at DESC)
-- (project_id, session_id, occurred_at DESC)
-- (project_id, user_id, occurred_at DESC)
-- (project_id, ip, occurred_at DESC)
-- (project_id, user_agent, occurred_at DESC)
```

Migrations use Goose format. Run: `goose -dir migrations postgres "<DSN>" up`.

## Inter-service dependencies

| Dependency | How |
|---|---|
| go-web-platform | Library — SetupPlatform, error handling, logging, db/session |

**Consumed by:**
- go-gateway-template — forwards frontend events via `pkg/client`

## Extension guide

**Adding a new endpoint:**
1. Add handler method to `EventHandler` in `internal/transport/http/handler/event_handler.go`
2. Add input/output DTOs to `pkg/client/dto/dto.go`
3. Add service method to `service.EventService` interface and implementation
4. Add repository method if DB access is needed
5. Register route in `internal/transport/http/router.go`
6. Add method to `EventAPIService` interface + implementation in `pkg/client/service/event_api_service.go`
7. Run `swag init` to regenerate swagger docs

**Adding a new query filter:**
1. Add field to `EventQueryInputDTO` in `pkg/client/dto/dto.go` with `validate:"omitempty,..."` tag
2. Update `internal/types/` filter struct if separate from DTO
3. Update repository query builder to include the new filter condition

## Common pitfalls

- **`message_id` not a UUID** — the `message_id` field requires `validate:"required,uuid"`. Passing a non-UUID string causes a validation error, not idempotency lookup.
- **Reusing `message_id` across different events** — `UNIQUE(project_id, message_id)` means the second ingest of the same pair returns the first event silently. Always generate a fresh UUID per logical event.
- **`occurred_at` vs `received_at`** — `occurred_at` is client-supplied (when it happened), `received_at` is server-set (when stored). Do not confuse them in queries.
- **`payload` is required** — even if empty, send `"payload": {}`. A missing `payload` key fails validation.
- **Batch limit** — max 100 items per batch request (validated). Split larger batches.
- **Soft delete doesn't filter automatically** — repositories must add `WHERE deleted_at IS NULL` to all non-detail queries. Indexes are partial; full-table queries that bypass partial indexes will miss the optimization.
- **`distinct_id` required** — unlike `user_id` (optional), `distinct_id` is required on every event. For anonymous visitors, generate a UUID in localStorage and send it as `distinct_id`.
