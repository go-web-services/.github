# go-gateway-template

**GitHub:** `github.com/go-web-services/go-gateway-template`
**Module:** `github.com/go-web-services/go-gateway-template`

## Role

HTTP API gateway. The only publicly-exposed service. Proxies auth and user-profile flows to go-service-user and analytics events to go-service-event. Handles Cloudflare Turnstile captcha verification on sensitive auth endpoints, JWT extraction from `Authorization: Bearer` header, fingerprint cookie management, and automatic token refresh. Ships an Nginx reverse proxy config in `proxy/` for production.

## Place in architecture

| | |
|---|---|
| **Layer** | Gateway — the **only publicly-exposed service** |
| **Called by** | go-fe-website (frontend client) via HTTP. All external traffic enters the system here. |
| **Calls** | go-service-user (auth + user flows), go-service-event (analytics events) |
| **Sits behind** | Nginx reverse proxy (`proxy/` directory handles TLS termination and upstream routing in production) |
| **Does NOT call** | go-integration-email or go-integration-minio directly — those are internal-only and called by backend services |

This is the boundary between the public internet and the internal service mesh. Never expose go-service-user or go-service-event to the internet directly — all client requests must go through this gateway.

## Directory structure

```
gateway/go-gateway-template/
├── cmd/app/
│   └── main.go                      # Entry point
├── config/
│   └── config.go                    # LoadConfig() — all env vars
├── internal/
│   ├── constants/
│   │   ├── auth_constants.go        # Cookie names, header keys, context keys, error codes
│   │   └── event_constants.go       # Event-specific constants (EventProjectID)
│   ├── dto/                         # Gateway-layer request/response DTOs
│   │   └── *.go                     # LoginInputDTO, SignupInputDTO, UserDTO, OTPLoginInputDTO, etc.
│   ├── error/
│   │   └── errors.go                # ErrTokenRefreshed, ErrAuthTokenMissing, ErrFingerprintMissing, ErrCSRFTokenMissing, ErrInvalidStateFormat
│   ├── transport/http/
│   │   ├── handler/
│   │   │   ├── auth_handler.go      # AuthHandler — login/logout/signup/OTP/Google SSO/forgot-password
│   │   │   ├── user_handler.go      # UserHandler — /me, /update, /providers/list
│   │   │   └── event_handler.go     # EventHandler — /events/send
│   │   ├── middleware/
│   │   │   ├── user_info_middleware.go  # UserInfoMiddleware — soft auth, extracts + validates JWT
│   │   │   ├── auth_middleware.go       # AuthMiddleware — hard auth, aborts if no user
│   │   │   └── captcha_middleware.go    # CaptchaMiddleware (Turnstile) — verifies cf_turnstile_response
│   │   └── router.go                # Route registration
│   ├── utils/
│   │   └── utils.go                 # IsTokenExpiredError, IsAuthenticationError, ExtractAuthToken, ExtractFingerprint, SetFingerprintCookie, GenerateCSRFToken
│   └── validation/                  # Custom validators (strong_password, username_alpha)
├── proxy/
│   ├── nginx.conf                   # Production Nginx config
│   ├── nginx-dev.conf               # Dev Nginx config
│   └── Dockerfile                   # Nginx container
└── debug/
    └── Dockerfile                   # Hot-reload dev container
```

## Key patterns

### Middleware chain

```
CaptchaMiddleware()  →  handler                      (Turnstile-protected routes)
UserInfoMiddleware() →  handler                      (optional auth — events/send)
UserInfoMiddleware() →  AuthMiddleware() →  handler  (required auth — /me, /update, /providers/list, /logout)
```

**UserInfoMiddleware** (soft): extracts `Authorization: Bearer <token>` + `fingerprint` cookie, calls `AuthorizeV1` on user service. On success, sets `user_id` and `user_info` in Gin context. On token expiry, calls `RefreshTokenV1`; if refresh succeeds, sets `X-New-Auth-Token` response header and stores `ErrTokenRefreshed` in context (`auth_error` key). On any failure, sets error in context and calls `c.Next()` — does NOT abort.

**AuthMiddleware** (hard): reads `auth_error` from context. If `ErrTokenRefreshed` → returns `TOKEN_REFRESHED` error code (client must re-read cookies and retry). If missing credentials → 401. If expired/invalid → 401. If no `user_id` in context → 401.

**CaptchaMiddleware**: reads `cf_turnstile_response` field from JSON body, verifies against Cloudflare API, re-serializes body so downstream handler can bind it again. Dev bypass: send `X-Bypass-Captcha: true` header with `APP_ENV=dev`.

### Token refresh flow

```
Client → Gateway (Authorization: Bearer <expired_token>, Cookie: fingerprint=<fp>)
Gateway → UserService /auth/tokens/authorize  →  token_expired error
Gateway → UserService /auth/tokens/refresh    →  new_token
Gateway → Client: X-New-Auth-Token: <new_token> + error TOKEN_REFRESHED (code 400 gateway-side)
Client retries with new token from header, re-sets fingerprint cookie
```

### Google SSO CSRF protection

Gateway generates a random CSRF token, stores it in `google_sso_state` cookie (httpOnly, secure, 10 min TTL). On callback, splits state param (`<original_state>:<csrf_token>`), compares cookie, deletes cookie on success.

## API contracts

All routes are under `/api/v1`.

### Auth routes

#### POST /api/v1/auth/login
Middleware: `CaptchaMiddleware`
```json
// Request
{
  "email": "user@example.com",
  "password": "P@ssw0rd!",
  "cf_turnstile_response": "<token>"
}
// Response — auth_token in X-New-Auth-Token header, fingerprint in Set-Cookie
{ "user": { "id": "...", "email": "...", "username": "...", "status": "active", "created_at": "...", "has_password": true } }
```

#### POST /api/v1/auth/logout
Middleware: `UserInfoMiddleware → AuthMiddleware`
```json
// Request: empty body
// Response
{ "message": "Logged out successfully" }
```

#### POST /api/v1/auth/signup
Middleware: `CaptchaMiddleware`
```json
// Request
{
  "email": "user@example.com",
  "username": "johndoe",
  "password": "P@ssw0rd!",
  "cf_turnstile_response": "<token>"
}
// Response
{ "message": "Account created successfully. Please check your email to activate your account." }
```

#### POST /api/v1/auth/activate-account
```json
// Request
{ "activation_token": "<token>" }
// Response — auth_token in header, fingerprint in cookie
{ "user": { ... } }
```

#### POST /api/v1/auth/activate-account/resend
Middleware: `CaptchaMiddleware`
```json
// Request
{ "email": "user@example.com", "cf_turnstile_response": "<token>" }
// Response
{ "message": "Activation email resent." }
```

#### POST /api/v1/auth/forgot-password/start
Middleware: `CaptchaMiddleware`
```json
// Request
{ "email": "user@example.com", "cf_turnstile_response": "<token>" }
// Response
{ "message": "Password reset email sent." }
```

#### POST /api/v1/auth/forgot-password/finish
Middleware: `CaptchaMiddleware`
```json
// Request
{
  "forgot_password_token": "<token>",
  "new_password": "NewP@ss1!",
  "delete_sessions": true,
  "cf_turnstile_response": "<token>"
}
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

#### POST /api/v1/auth/google-sso/get-link
```json
// Request: empty body
// Response
{ "auth_url": "https://accounts.google.com/o/oauth2/auth?..." }
// Side effect: sets google_sso_state cookie
```

#### POST /api/v1/auth/google-sso/callback
```json
// Request
{ "code": "<google_auth_code>", "state": "<state>" }
// Response — auth_token in header, fingerprint in cookie
{ "user": { ... }, "is_new_user": true }
```

#### POST /api/v1/auth/otp/signup
Middleware: `CaptchaMiddleware`
```json
// Request
{ "email": "user@example.com", "cf_turnstile_response": "<token>" }
// Response
{ "user": { "id": "...", "email": "...", "username": "...", "status": "pending", "created_at": "...", "has_password": false } }
```

#### POST /api/v1/auth/otp/login
Middleware: `CaptchaMiddleware`
```json
// Request
{ "email": "user@example.com", "otp": "123456", "cf_turnstile_response": "<token>" }
// Response — auth_token in header, fingerprint in cookie
{ "user": { ... } }
```

### User routes

#### GET /api/v1/users/me
Middleware: `UserInfoMiddleware → AuthMiddleware`
```json
// Response
{ "user": { "id": "...", "email": "...", "username": "...", "product": "...", "status": "active", "created_at": "...", "updated_at": "...", "is_deleted": false, "has_password": true } }
```

#### POST /api/v1/users/update
Middleware: `UserInfoMiddleware → AuthMiddleware`
```json
// Request (all fields optional)
{ "username": "newname", "email": "new@example.com" }
// Response
{ "user": { ... } }
```

#### POST /api/v1/users/providers/list
Middleware: `UserInfoMiddleware → AuthMiddleware`
```json
// Response
{ "auth_providers": [{ "type": "google" }] }
```

### Event routes

#### POST /api/v1/events/send
Middleware: `UserInfoMiddleware` (optional — user_id enriched if authenticated)
```json
// Request
{
  "project_id": "proj_live_01",
  "message_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "distinct_id": "anon_abc123",
  "name": "page_viewed",
  "payload": { "path": "/home" },
  "occurred_at": "2026-01-01T12:00:00Z",
  "session_id": "sess_xyz",
  "ip": "1.2.3.4",
  "user_agent": "Mozilla/5.0 ..."
}
// Response
{ "event": { ... } }   // EventDTO from go-service-event
```

### Common response headers

| Header | When set |
|---|---|
| `X-New-Auth-Token` | On successful login, OTP login, activate-account, Google SSO callback, or token refresh |
| `Set-Cookie: fingerprint=<fp>; HttpOnly; Secure` | Same as above |

### Error codes specific to gateway

| Code | Meaning |
|---|---|
| `TOKEN_REFRESHED` | Token was auto-refreshed; client must retry with `X-New-Auth-Token` value |
| `CSRF_ERROR` | Google SSO CSRF validation failed |
| `UNAUTHORIZED_ERROR` | Missing/invalid/expired token |

## Configuration

| Env var | Type | Default | Description |
|---|---|---|---|
| `APP_PORT` | int | `8080` | Gin listen port |
| `APP_ENV` | string | `dev` | `dev` or `production`; affects captcha bypass and Gin mode |
| `ALLOW_ORIGINS` | comma-sep string | `http://localhost:3000,...` | CORS allowed origins |
| `USER_SERVICE_URL` | string | `http://host.docker.internal:8007` | Base URL for go-service-user |
| `EVENT_SERVICE_URL` | string | `http://host.docker.internal:8010` | Base URL for go-service-event |
| `TURNSTILE_SECRET_KEY` | string | `""` | Cloudflare Turnstile server-side secret |
| `AUTH_FINGERPRINT_COOKIE_DOMAIN` | string | `""` | Cookie domain (empty = current host) |
| `AUTH_FINGERPRINT_COOKIE_EXPIRATION_SEC` | int | `2592000` (30 days) | Fingerprint cookie max-age |

## Inter-service dependencies

| Service | How | Import |
|---|---|---|
| go-service-user | HTTP via `pkg/client` | `github.com/go-web-services/go-service-user/pkg/client` |
| go-service-event | HTTP via `pkg/client` | `github.com/go-web-services/go-service-event/pkg/client` |
| go-web-platform | Library | `github.com/go-web-services/go-web-platform` |

The gateway uses `AuthAPIService` (user service client) inside `UserInfoMiddleware` and auth handlers. It uses `EventAPIService` (event service client) in `EventHandler`.

## Extension guide

**Adding a new proxied route:**
1. Add handler in `internal/transport/http/handler/<resource>_handler.go`
2. Add method to the appropriate `APIService` client interface (or create new client)
3. Register route in `internal/transport/http/router.go` with correct middleware chain
4. Add service URL to `config/config.go` + `ServicesConfig` struct if new upstream
5. Add env var to `LoadConfig()`

**Adding a new captcha-protected route:**
- Wrap handler with `middleware.CaptchaMiddleware()` in router registration
- Request body MUST have a `cf_turnstile_response` string field
- Frontend must send valid Turnstile widget token

**Adding a new auth-required route:**
- Chain: `middleware.UserInfoMiddleware(log, userAuthAPIClient), middleware.AuthMiddleware(log), handler`
- Handler reads user ID with: `userID, _ := c.Get(authConstants.AuthUserIDContextKey)`

## Common pitfalls

- **Calling user service directly from frontend** — all requests must go through the gateway; user service is internal-only.
- **Missing `cf_turnstile_response` on Turnstile routes** — `CaptchaMiddleware` aborts with 400 if field is absent. Frontend must include it.
- **Not handling `TOKEN_REFRESHED` on client** — when gateway returns `TOKEN_REFRESHED` error code, the new token is in `X-New-Auth-Token` response header. Client must update its stored token and retry.
- **Hardcoding `AuthProduct`** — the gateway uses `AuthProduct = "main"` as the product identifier sent to user service. If deploying a second product, change this constant or make it configurable.
- **Fingerprint cookie on wrong domain** — `AUTH_FINGERPRINT_COOKIE_DOMAIN` must match the frontend domain in production; empty string restricts cookie to the exact host.
- **CSRF state cookie** — Google SSO state cookie expires in 10 minutes; if the OAuth callback takes longer, CSRF validation fails.
- **Event `project_id`** — comes from `EventProjectID` constant in `event_constants.go`; update it when deploying for a new project.
