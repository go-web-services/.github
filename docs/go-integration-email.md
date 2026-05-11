# go-integration-email

**GitHub:** `github.com/go-web-services/go-integration-email`
**Module:** `github.com/go-web-services/go-integration-email`

## Role

Outbound transactional email service. Callers POST `{emailType, recipients, params}` → service looks up template files by email type → renders HTML + plain-text + subject with Go `text/template` → sends via SMTP (STARTTLS on port 587). No inbox, no webhooks, no retry queue. Internal only — not exposed to the internet.

## Place in architecture

| | |
|---|---|
| **Layer** | Integration — internal only, **not exposed to the internet** |
| **Called by** | go-service-user (activation emails, forgot-password, OTP sign-in). Any other internal service that needs outbound email may also call it. |
| **Calls** | External SMTP server only |
| **Does NOT receive calls from** | go-gateway-template or the frontend — this service is internal service-to-service only |

This service is a thin wrapper over SMTP. It should never be placed in a public subnet or behind a public load balancer.

## Directory structure

```
integration/go-integration-email/
├── cmd/app/
│   └── main.go                          # Entry point
├── config/
│   └── config.go                        # SMTP config from env
├── internal/
│   ├── constants/
│   │   └── constants.go                 # (empty — email type constants live in pkg/client/constants)
│   ├── mappings/
│   │   └── email_mapping.go             # EmailTypeMapping: emailType → {html, txt, subject} paths
│   ├── service/
│   │   └── email_service.go             # EmailService.SendEmail — template render + smtp.SendMail
│   ├── templates/
│   │   └── auth/
│   │       ├── forgot-password/
│   │       │   ├── body.html
│   │       │   ├── body.txt
│   │       │   └── subject.txt
│   │       ├── email-confirm/
│   │       │   ├── body.html
│   │       │   ├── body.txt
│   │       │   └── subject.txt
│   │       └── otp-signin/
│   │           ├── body.html
│   │           ├── body.txt
│   │           └── subject.txt
│   ├── transport/http/
│   │   ├── handler/                     # HTTP handler for POST /api/v1/send
│   │   └── router.go                    # Route registration
│   ├── types/
│   │   └── types.go                     # EmailTemplateMapping struct
│   └── utils/
│       └── utils.go                     # BuildTemplatePath helper
├── pkg/client/                          # Importable by other services
│   ├── constants/
│   │   └── constants.go                 # AuthForgotPassword, AuthEmailConfirm, AuthOTPSignin
│   ├── dto/
│   │   ├── dto.go                       # BaseSendEmailDTO, GeneralSendEmailDTO, SendEmailOutputDTO
│   │   ├── auth_dto.go                  # AuthForgotPasswordInputDTO, AuthEmailConfirmInputDTO, AuthOTPSigninInputDTO + Params
│   │   └── website_dto.go               # (website-specific DTOs if present)
│   ├── types/
│   │   └── types.go                     # EmailType = string
│   └── service/
│       └── email_api_service.go         # EmailAPIService HTTP client
└── info/
    ├── AI_DOCS.md                       # Step-by-step guide for adding new email types
    └── EMAILS_DESCRIPTION.md            # Human descriptions of existing email types
```

## Key patterns

### Template rendering

Templates use Go's `text/template` syntax. Params are passed as `map[string]any`. The service always injects `{{.Year}}` (current year, for copyright lines).

```html
<!-- body.html example -->
<p>Click <a href="{{.ForgotPasswordLink}}">here</a> to reset your password.</p>
<p>Link expires in {{.ExpirationTimeInMinutes}} minutes.</p>
<small>&copy; {{.Year}}</small>
```

Warning: extra newlines in template files (especially `subject.txt`) corrupt the MIME message. Keep subject files to exactly one line with no trailing newline.

### EmailTypeMapping

Mapping lives in `internal/mappings/email_mapping.go`. Key is `pkg/client/types.EmailType` (string), value is `internal/types.EmailTemplateMapping` with three file paths relative to `internal/templates/`.

```go
var EmailTypeMapping = map[clientTypes.EmailType]types.EmailTemplateMapping{
    clientConsts.AuthForgotPassword: {
        HTMLTemplateFileName: "auth/forgot-password/body.html",
        TextTemplateFileName: "auth/forgot-password/body.txt",
        SubjectFileName:      "auth/forgot-password/subject.txt",
    },
    clientConsts.AuthEmailConfirm: {
        HTMLTemplateFileName: "auth/email-confirm/body.html",
        TextTemplateFileName: "auth/email-confirm/body.txt",
        SubjectFileName:      "auth/email-confirm/subject.txt",
    },
    clientConsts.AuthOTPSignin: {
        HTMLTemplateFileName: "auth/otp-signin/body.html",
        TextTemplateFileName: "auth/otp-signin/body.txt",
        SubjectFileName:      "auth/otp-signin/subject.txt",
    },
}
```

## API contracts

### POST /api/v1/send

Generic send — for callers that build params manually:

```json
// Request
{
  "emailType": "AuthForgotPassword",
  "recipients": ["user@example.com"],
  "params": {
    "ForgotPasswordLink": "https://example.com/reset?token=abc",
    "ExpirationTimeInMinutes": 720
  }
}
// Response
{ "message": "Email sent successfully" }
```

`emailType` must match a key in `EmailTypeMapping`; unknown types return 400.

### Typed DTOs (for pkg/client consumers)

Use strongly-typed DTOs instead of `GeneralSendEmailDTO.params` to avoid key typos:

```go
// AuthForgotPassword
dto.AuthForgotPasswordInputDTO{
    BaseSendEmailDTO: dto.BaseSendEmailDTO{
        EmailType:  constants.AuthForgotPassword,  // "AuthForgotPassword"
        Recipients: []string{"user@example.com"},
    },
    Params: dto.AuthForgotPasswordParams{
        ForgotPasswordLink:      "https://...",
        ExpirationTimeInMinutes: 720,
    },
}

// AuthEmailConfirm
dto.AuthEmailConfirmInputDTO{
    Params: dto.AuthEmailConfirmParams{
        EmailConfirmLink:        "https://...",
        ExpirationTimeInMinutes: 1440,
    },
}

// AuthOTPSignin
dto.AuthOTPSigninInputDTO{
    Params: dto.AuthOTPSigninParams{
        OTPCode:                 "123456",
        ExpirationTimeInMinutes: 10,
    },
}
```

### pkg/client usage

```go
import (
    emailClient "github.com/go-web-services/go-integration-email/pkg/client/service"
    emailDTO    "github.com/go-web-services/go-integration-email/pkg/client/dto"
    emailConsts "github.com/go-web-services/go-integration-email/pkg/client/constants"
)

client := emailClient.NewEmailAPIService("http://go-integration-email:8080")

result, err := client.SendEmailV1(ginCtx, emailDTO.AuthForgotPasswordInputDTO{...})
```

`SendEmailV1` accepts any `dto.EmailPayload` (= `any`); JSON-marshals it and POSTs to `/api/v1/send`.

### Existing email types

| Constant | emailType string | Template folder | Params |
|---|---|---|---|
| `AuthForgotPassword` | `"AuthForgotPassword"` | `auth/forgot-password/` | `ForgotPasswordLink`, `ExpirationTimeInMinutes` |
| `AuthEmailConfirm` | `"AuthEmailConfirm"` | `auth/email-confirm/` | `EmailConfirmLink`, `ExpirationTimeInMinutes` |
| `AuthOTPSignin` | `"AuthOTPSignin"` | `auth/otp-signin/` | `OTPCode`, `ExpirationTimeInMinutes` |

## Configuration

| Env var | Type | Default | Description |
|---|---|---|---|
| `APP_PORT` | int | `8080` | Gin listen port |
| `APP_ENV` | string | `development` | `dev` or `production` |
| `EMAIL_SERVER` | string | `""` | SMTP server hostname |
| `EMAIL_PORT` | int | `587` | SMTP port (STARTTLS) |
| `EMAIL_USERNAME` | string | `""` | SMTP auth username |
| `EMAIL_PASSWORD` | string | `""` | SMTP auth password |
| `EMAIL_FROM` | string | `""` | From address on sent emails |

## Inter-service dependencies

| Dependency | How |
|---|---|
| go-web-platform | Library — SetupPlatform, error handling, logging, utils.SendRequest |

No outbound calls to other go-web services. SMTP is the only external I/O.

## Extension guide

**Adding a new email type — 6 steps:**

1. **Create template directory:**
   ```
   internal/templates/<group>/<email-type>/
   ```
   Use kebab-case for the folder name. Group by domain (e.g., `auth/`, `billing/`).

2. **Create three template files:**
   - `body.html` — full HTML email (use inline styles for email client compat)
   - `body.txt` — plain text fallback
   - `subject.txt` — single line, no trailing newline

   Variables: `{{.ParamName}}` where ParamName matches a key in the params map (PascalCase). `{{.Year}}` is always available.

3. **Add constant in `pkg/client/constants/constants.go`:**
   ```go
   var MyNewEmail types.EmailType = "MyNewEmail"
   ```

4. **Add entry to `internal/mappings/email_mapping.go`:**
   ```go
   clientConsts.MyNewEmail: {
       HTMLTemplateFileName: "group/my-new-email/body.html",
       TextTemplateFileName: "group/my-new-email/body.txt",
       SubjectFileName:      "group/my-new-email/subject.txt",
   },
   ```

5. **Add typed DTO in `pkg/client/dto/`** (optional but recommended):
   ```go
   type MyNewEmailInputDTO struct {
       BaseSendEmailDTO
       Params MyNewEmailParams `json:"params" binding:"required"`
   }
   type MyNewEmailParams struct {
       SomeLink string `json:"SomeLink" binding:"required"`
   }
   ```

6. **Update `info/EMAILS_DESCRIPTION.md`** with email type name, trigger, and params.

**Button style conventions:**
- Primary button (solid background): action emails (password reset, activation)
- Secondary button (outlined): notification emails

## Common pitfalls

- **Trailing newline in `subject.txt`** — causes malformed MIME; the subject bleeds into headers. File must end with the subject text, no `\n`.
- **PascalCase params keys** — Go template fields must match exactly. `{{.forgotPasswordLink}}` fails; `{{.ForgotPasswordLink}}` works. Keep all param map keys PascalCase.
- **Missing mapping entry** — if `emailType` string has no entry in `EmailTypeMapping`, the service cannot find templates and returns 500. Always add mapping when adding a new type.
- **HTML inline styles only** — email clients strip `<style>` blocks. All CSS must be inline.
- **Not updating pkg/client** — consumers import `pkg/client`; if you add a constant/DTO there, bump and retag the module so dependent services can `go get` the update.
- **Using `constants.go` inside `internal/`** — the `internal/constants/constants.go` file is effectively empty; all email type constants live in `pkg/client/constants/constants.go`. Don't add constants to the wrong location.
