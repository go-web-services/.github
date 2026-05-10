# go-service-user

**GitHub:** `github.com/go-web-services/go-service-user`
**Module:** `github.com/go-web-services/go-service-user`
**Local path:** `/Users/lomank/Workspace/go-web/service/go-service-user/`

## Role

User accounts and authentication service. Owns the full auth lifecycle: email+password signup with activation, JWT-based login with fingerprint cookie for refresh, logout (session invalidation), password reset via email token, OTP (one-time password) flows, Google OAuth2 SSO, and user profile CRUD. Also exposes internal endpoints for token authorization/refresh used by go-gateway-template's `UserInfoMiddleware`. Stores data in PostgreSQL; calls go-integration-email for transactional mail. All auth tokens are JWTs; sessions are server-side (stored in `sessions` table) and linked to a fingerprint cookie.

## Place in architecture

| | |
|---|---|
| **Layer** | Service — internal, not exposed to the internet |
| **Called by** | go-gateway-template exclusively — all auth endpoints + `/tokens/authorize` + `/tokens/refresh` via `pkg/client` |
| **Calls** | go-integration-email (transactional mail), go-web-platform (library) |
| **Own database** | PostgreSQL — dedicated instance, not shared with any other service |

This service is the identity authority of the ecosystem. The gateway delegates all authentication decisions here — it never inspects JWT secrets itself. If you need to validate or refresh a token, call this service's `/tokens/authorize` or `/tokens/refresh` endpoints via `pkg/client`.

## Directory structure

```
service/go-service-user/
├── cmd/app/
│   └── main.go                              # Entry point
├── config/
│   ├── config.go                            # App config — JWT secret, token TTLs, feature flags, OAuth
│   └── postgres_config.go                   # PostgreSQL DSN from env
├── internal/
│   ├── clients/
│   │   └── email/                           # go-integration-email client wrapper
│   ├── domain/
│   │   ├── user.go                          # User domain model
│   │   ├── session.go                       # Session domain model
│   │   └── auth_provider.go                 # AuthProvider domain model (Google SSO link)
│   ├── repository/                          # PostgreSQL repos (uses db/session for tx)
│   ├── service/
│   │   ├── auth_service.go                  # AuthService — login/logout/signup/activate/OTP/forgot-password/tokens
│   │   ├── google_sso_service.go            # GoogleSSOAuthService — OAuth2 flow
│   │   ├── user_service.go                  # UserService — CRUD
│   │   └── email_whitelist_service.go       # EmailWhitelistService — whitelist check
│   ├── transport/http/
│   │   ├── handler/
│   │   │   ├── auth_handler.go              # AuthHandler — all auth endpoints
│   │   │   ├── google_sso_handler.go        # GoogleSSOHandler — SSO get-link + callback
│   │   │   ├── user_handler.go              # UserHandler — CRUD
│   │   │   └── email_whitelist_handler.go   # EmailWhitelistHandler — /whitelist/check
│   │   └── router.go                        # Route registration
│   └── types/                               # Internal types
├── migrations/
│   ├── 00001_init_schema.sql                # users, sessions, auth_providers tables
│   └── 00002_create_email_whitelist_table.sql
└── pkg/client/                              # Importable by other services
    ├── constants/
    │   └── constants.go                     # AuthTokenExpired, SessionExpired, InvalidAuthToken, etc.
    ├── dto/
    │   ├── user_dto.go                      # UserDTO, CRUD DTOs
    │   └── auth_dto.go                      # All auth DTOs (login/signup/OTP/SSO/tokens)
    ├── enum/
    │   └── enum.go                          # UserStatus, AuthProviderType
    └── service/
        ├── auth_api_service.go              # AuthAPIService HTTP client
        ├── google_sso_api_service.go        # GoogleSSOAPIService HTTP client
        └── user_api_service.go              # UserAPIService HTTP client
```

## Key patterns

- JWT access token + server-side session + fingerprint cookie for refresh. The fingerprint is a random string stored in a `sessions` row and in an httpOnly secure cookie. Token authorize checks both JWT validity and fingerprint-session match.
- Token expiry is configurable via `ACCESS_TOKEN_EXP_DELTA_IN_SECS` (default 15 min). Sessions expire after `SESSION_EXP_DELTA_IN_SECS` (default 30 days).
- OTP codes expire after `OTP_EXP_SECS` (default 10 min).
- Whitelist: when `WHITELIST_ENABLED=true`, only emails in `email_whitelists` table can sign up.
- Product scoping: every user record carries a `product` field. Email uniqueness is per (product, email) pair — the same email can exist in two products. All auth endpoints that create/look up users require `product`.
- Error pattern: `c.Error(err)` → ErrorHandlerMiddleware (go-web-platform).
- DB transactions: via `db/session.Transaction`.

## API contracts

### Shared UserDTO

```json
{
  "id": "550e8400-...",
  "email": "user@example.com",
  "username": "johndoe",
  "product": "main",
  "status": "active",      // enum: pending | active | blocked
  "created_at": "2026-01-01T00:00:00Z",
  "updated_at": "2026-01-01T00:00:00Z",
  "is_deleted": false,
  "has_password": true     // false for OTP/SSO-only users
}
```

### Auth endpoints (internal — called by go-gateway-template)

#### POST /api/v1/auth/signup
```json
// Request
{
  "email": "user@example.com",
  "username": "johndoe",
  "password": "P@ssw0rd!",
  "product": "main"
}
// Response
{ "activation_token": "<token>" }
// Side effect: sends AuthEmailConfirm email via go-integration-email
```

#### POST /api/v1/auth/activate-account
```json
// Request
{ "activation_token": "<token>" }
// Response — token + fingerprint for immediate login
{ "auth_token": "<jwt>", "fingerprint": "<fp>", "user": { /* UserDTO */ } }
```

#### POST /api/v1/auth/activate-account/resend
```json
// Request
{ "email": "user@example.com", "product": "main" }
// Response
{ "activation_token": "<new_token>" }
// Side effect: sends AuthEmailConfirm email
```

#### POST /api/v1/auth/login
```json
// Request
{ "email": "user@example.com", "password": "P@ssw0rd!", "product": "main" }
// Response
{ "auth_token": "<jwt>", "fingerprint": "<fp>", "user": { /* UserDTO */ } }
```

#### POST /api/v1/auth/logout
```json
// Request
{ "auth_token": "<jwt>" }
// Response
{ "message": "Logged out successfully" }
// Side effect: deletes session row
```

#### POST /api/v1/auth/forgot-password/start
```json
// Request
{ "email": "user@example.com", "product": "main" }
// Response
{ "forgot_password_token": "<token>" }
// Side effect: sends AuthForgotPassword email
```

#### POST /api/v1/auth/forgot-password/finish
```json
// Request
{ "forgot_password_token": "<token>", "new_password": "NewP@ss1!", "delete_sessions": true }
// Response
{ "message": "Password changed successfully." }
```

#### POST /api/v1/auth/forgot-password/check-token
```json
// Request
{ "forgot_password_token": "<token>" }
// Response
{ "valid": true }
```

#### POST /api/v1/auth/tokens/authorize
Used by gateway's `UserInfoMiddleware` to validate a JWT + fingerprint pair.
```json
// Request
{ "auth_token": "<jwt>", "fingerprint": "<fp>" }
// Response
{ "user": { /* UserDTO */ } }
// Errors: AuthTokenExpired, InvalidAuthToken, FingerprintMismatch, SessionExpired
```

#### POST /api/v1/auth/tokens/refresh
Used by gateway when authorize returns `AuthTokenExpired`.
```json
// Request
{ "auth_token": "<expired_jwt>", "fingerprint": "<fp>" }
// Response
{ "auth_token": "<new_jwt>" }
```

#### POST /api/v1/auth/compare-password
Internal — compare plaintext password against stored hash.
```json
// Request
{ "user_id": "<uuid>", "password": "plaintext" }
// Response
{ "is_match": true }
```

#### POST /api/v1/auth/otp/signup
Creates or looks up user, sends OTP email.
```json
// Request
{ "email": "user@example.com", "product": "main" }
// Response
{ "user": { /* UserDTO */ } }
// Side effect: sends AuthOTPSignin email with OTP code
```

#### POST /api/v1/auth/otp/login
Validates OTP, issues token + fingerprint.
```json
// Request
{ "email": "user@example.com", "otp": "123456", "product": "main" }
// Response
{ "access_token": "<jwt>", "fingerprint": "<fp>", "user": { /* UserDTO */ } }
```

#### POST /api/v1/auth/sso/google/get-link
```json
// Request: empty body or { "product": "main" }
// Response
{ "auth_url": "https://accounts.google.com/o/oauth2/auth?..." }
```

#### POST /api/v1/auth/sso/google/callback
```json
// Request
{ "code": "<google_code>", "state": "<state>" }
// Response
{ "auth_token": "<jwt>", "fingerprint": "<fp>", "user": { /* UserDTO */ }, "is_new_user": true }
```

### User CRUD endpoints (internal — service-to-service)

#### POST /api/v1/users/create
```json
// Request
{
  "id": "optional-uuid",       // optional, generated if absent
  "email": "user@example.com", // required
  "username": "johndoe",       // required
  "product": "main",           // required
  "status": "active"           // required, enum
}
// Response
{ "user": { /* UserDTO */ } }
```

#### POST /api/v1/users/detail
```json
// Request
{ "id": "<uuid>" }
// Response
{ "user": { /* UserDTO */ } }
```

#### POST /api/v1/users/update
```json
// Request (all fields optional except id)
{
  "id": "<uuid>",
  "email": "new@example.com",
  "username": "newname",
  "product": "main",
  "status": "blocked",
  "is_deleted": false
}
// Response
{ "user": { /* UserDTO */ } }
```

#### POST /api/v1/users/query
Paginated filtering.
```json
// Request
{
  "ids": ["<uuid>", ...],          // optional
  "email": "substring",            // optional
  "username": "substring",         // optional
  "product": "main",               // optional
  "status": "active",              // optional
  "is_deleted": false,             // optional
  "page": 1,
  "limit": 20
}
// Response
{
  "users": [ /* array of UserDTO */ ],
  "meta": { "pagination": { "page": 1, "total_pages": 3, "per_page": 20, "total": 55 } }
}
```

### Email whitelist

#### POST /api/v1/whitelist/check
```json
// Request
{ "email": "user@example.com" }
// Response
{ "is_whitelisted": true }
```

### pkg/client usage

```go
import userClient "github.com/go-web-services/go-service-user/pkg/client/service"
import userDTO    "github.com/go-web-services/go-service-user/pkg/client/dto"

authClient := userClient.NewAuthAPIService("http://go-service-user:8006")
result, err := authClient.AuthorizeV1(ginCtx, userDTO.AuthorizeInputDTO{
    AuthToken:   token,
    Fingerprint: fingerprint,
})
```

### Error codes (pkg/client/constants)

| Constant | Meaning |
|---|---|
| `AuthTokenExpired` | JWT expired — trigger token refresh |
| `SessionExpired` | Server-side session invalidated |
| `InvalidAuthToken` | JWT signature/format invalid |
| `InvalidAuthTokenClaims` | JWT claims malformed |
| `FingerprintMismatch` | Fingerprint doesn't match session |
| `UserNotActive` | User is `pending` or `blocked` |

## Configuration

| Env var | Type | Default | Description |
|---|---|---|---|
| `APP_PORT` | int | `8006` | Gin listen port |
| `APP_ENV` | string | `dev` | `dev` or `production` |
| `SECRET_KEY` | string | `secret-key` | JWT signing secret — change in production |
| `ACCESS_TOKEN_EXP_DELTA_IN_SECS` | int | `900` | JWT TTL (15 min) |
| `SESSION_EXP_DELTA_IN_SECS` | int | `2592000` | Session TTL (30 days) |
| `RESET_PASSWORD_EXP_SECS` | int | `43200` | Forgot-password token TTL (12 hours) |
| `ACTIVATION_TOKEN_EXP_SECS` | int | `86400` | Activation token TTL (24 hours) |
| `OTP_EXP_SECS` | int | `600` | OTP code TTL (10 min) |
| `EMAIL_SERVICE_HOST` | string | `http://127.0.0.1:8005` | go-integration-email base URL |
| `WEBSITE_URL` | string | `https://example.com` | Used in email template links |
| `WHITELIST_ENABLED` | bool | `false` | Enable email whitelist gate on signup |
| `EXPIRED_USERS_WORKER_INTERVAL_SECS` | int64 | `3600` | Background worker interval for cleaning expired sessions |
| `GOOGLE_OAUTH_CLIENT_ID` | string | `""` | Google OAuth2 client ID |
| `GOOGLE_OAUTH_CLIENT_SECRET` | string | `""` | Google OAuth2 client secret |
| `GOOGLE_OAUTH_REDIRECT_URL` | string | `""` | Google OAuth2 redirect URL |
| `POSTGRES_USER` | string | `go-service-user` | DB user |
| `POSTGRES_PASSWORD` | string | `go-service-user-password` | DB password |
| `POSTGRES_HOST` | string | `host.docker.internal` | DB host |
| `POSTGRES_PORT` | string | `5435` | DB port |
| `POSTGRES_DB` | string | `go-service-user-db` | DB name |
| `POSTGRES_SSL_MODE` | string | `disable` | `disable` or `require` |

## Database schema

```sql
-- users
CREATE TABLE users (
    id                       UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username                 TEXT NOT NULL,
    email                    TEXT NOT NULL,
    password                 TEXT,           -- nullable (OTP/SSO users have no password)
    product                  TEXT NOT NULL,
    status                   TEXT NOT NULL,  -- pending | active | blocked
    otp_code                 TEXT DEFAULT '',
    otp_generated_at         TIMESTAMPTZ,
    reset_password_token     TEXT DEFAULT '',
    reset_password_expire_at TIMESTAMPTZ,
    activation_token         TEXT DEFAULT '',
    activation_token_expire_at TIMESTAMPTZ DEFAULT '0001-01-01 00:00:00Z',
    last_login_at            TIMESTAMPTZ,
    activated_at             TIMESTAMPTZ,
    is_deleted               BOOLEAN NOT NULL DEFAULT FALSE,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- Unique: (product, email) WHERE is_deleted = false
-- Unique: id WHERE is_deleted = false
-- Unique: activation_token WHERE activation_token != ''
-- Unique: reset_password_token WHERE reset_password_token != ''

-- sessions
CREATE TABLE sessions (
    id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    expired_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- auth_providers (SSO links)
CREATE TABLE auth_providers (
    id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id          UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type             TEXT NOT NULL,    -- "google"
    provider_user_id TEXT NOT NULL,
    metadata         JSONB,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- Unique: (type, provider_user_id)
-- Unique: (user_id, type)

-- email_whitelists
CREATE TABLE email_whitelists (
    id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email      TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);
```

Migrations use Goose. Run: `goose -dir migrations postgres "<DSN>" up`.

## Inter-service dependencies

| Service | How | Purpose |
|---|---|---|
| go-integration-email | HTTP via `pkg/client` | Sends activation, forgot-password, OTP emails |
| go-web-platform | Library | SetupPlatform, error handling, logging, db/session, utils.SendRequest |

**Consumed by:**
- go-gateway-template — all auth endpoints + `/tokens/authorize` + `/tokens/refresh` via `pkg/client`

## Extension guide

**Adding a new auth endpoint:**
1. Add method to `AuthService` interface + implementation in `internal/service/auth_service.go`
2. Add handler method to `AuthHandler` in `internal/transport/http/handler/auth_handler.go`
3. Add input/output DTOs to `pkg/client/dto/auth_dto.go`
4. Register route in `internal/transport/http/router.go` under `newAuthRouterGroup`
5. Add method to `AuthAPIService` in `pkg/client/service/auth_api_service.go`
6. If new email needed: add email type to go-integration-email and call it via `EmailAPIService`

**Adding a new user field:**
1. Add migration SQL (new column with default)
2. Update `domain.User` struct
3. Update repository scan + insert/update queries
4. Update `UserDTO` in `pkg/client/dto/user_dto.go`
5. Update `UserUpdateInputDTO` if the field should be updateable
6. Tag and release new pkg/client version

**Enabling Google SSO:**
- Set `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`, `GOOGLE_OAUTH_REDIRECT_URL`
- The redirect URL must match the one registered in Google Cloud Console

## Common pitfalls

- **`product` field missing** — every auth operation (signup, login, OTP, resend) requires `product`. The gateway hardcodes `AuthProduct = "main"`; if deploying a second product, this must be changed.
- **Reusing `activation_token` after expiry** — expired tokens return an error, not a re-send. Frontend must call `/resend` to get a new token.
- **`has_password: false` users** — OTP and SSO users have no password. Attempting `/login` with password for these users fails. Check `has_password` before showing password fields.
- **Partial unique index on soft-deleted users** — `(product, email)` unique only `WHERE is_deleted = false`. Soft-deleted users free up their email for re-registration.
- **`SECRET_KEY` default in production** — default is `"secret-key"`. Any service running with the default can forge tokens. Always override in production.
- **Session cleanup** — expired sessions are cleaned by a background worker. The interval is `EXPIRED_USERS_WORKER_INTERVAL_SECS`. Expired sessions do not cause active requests to fail (they fail at fingerprint validation), but they accumulate until the worker runs.
- **Google SSO state is validated by the gateway** (CSRF cookie), not by this service. The callback endpoint here trusts the `state` param passed through. CSRF enforcement is in go-gateway-template.
- **`pkg/client` is a separate concern** — callers import `pkg/client`, not the service internals. Never reference `internal/` types from outside the service.
- **`access_token` vs `auth_token`** — OTP login response uses `access_token`; all other login/activate responses use `auth_token`. Be consistent when reading from the pkg/client DTO.
