# go-integration-minio

**GitHub:** `github.com/go-web-services/go-integration-minio`
**Module:** `github.com/go-web-services/go-integration-minio`
**Local path:** `/Users/lomank/Workspace/go-web/integration/go-integration-minio/`

## Role

MinIO (S3-compatible) object storage integration service. Provides a stable JSON/HTTP API over MinIO so internal services do not need a MinIO SDK dependency. Handles upload, delete, URL generation, content retrieval, and direct file streaming. Other services call this via `pkg/client` instead of talking to MinIO directly.

Allowed upload file types: `.jpg`, `.jpeg`, `.png`, `.svg`, `.pdf`, `.doc`, `.docx`, `.webp`. Max upload size: 100 MB.

## Place in architecture

| | |
|---|---|
| **Layer** | Integration — internal only, **not exposed to the internet** |
| **Called by** | Any backend service that needs file storage (not wired in go-gateway-template by default — add as needed per project) |
| **Calls** | MinIO instance (external S3-compatible object storage) |
| **Does NOT receive calls from** | go-gateway-template or the frontend — file operations must go through a backend service that calls this |

Internal services call this instead of using the MinIO SDK directly, so MinIO credentials and SDK version are isolated to one place.

## Directory structure

```
integration/go-integration-minio/
├── cmd/app/
│   └── main.go                        # Entry point
├── config/
│   └── config.go                      # MinIO + app config from env
├── internal/
│   ├── service/                       # MinioService — wraps minio-go SDK calls
│   ├── transport/http/
│   │   ├── handler/
│   │   │   └── minio_handler.go       # UploadFileV1, DeleteFileV1, GetFileContentV1, GetFileURLV1, GetFile
│   │   └── router.go                  # Route registration
│   └── types/
│       └── types.go                   # BucketName type
└── pkg/client/                        # Importable by other services
    ├── dto/
    │   └── dto.go                     # All request/response DTOs
    ├── types/
    │   └── types.go                   # BucketName type alias
    ├── service/
    │   └── minio_api_service.go       # MinioAPIService HTTP client
    └── go.mod                         # Separate module: go-integration-minio/pkg/client
```

## Key patterns

### File naming on upload

When no custom `filename` is provided, the service prepends a timestamp prefix to the original filename (format `20060102150405-<original>`). Callers should store the returned `fileName` in their database — it is the key for all subsequent operations.

### Direct download vs. content endpoint

- `GET /api/v1/minio/file/:filename?bucket_name=X` — streams file directly, sets `Content-Type` from extension. Images: `Content-Disposition: inline`. Other types: `Content-Disposition: attachment`. Use this for user-facing downloads.
- `POST /api/v1/minio/get-file-content` — returns raw bytes as a JSON string. Use for internal processing only (e.g., reading a PDF for text extraction).

### pkg/client upload

Upload uses multipart form (not JSON) because it streams the file. The `pkg/client` has a dedicated `SendMultipartRequest` helper and handles Brotli-encoded responses.

## API contracts

### POST /api/v1/minio/upload

Multipart form upload.

```
Form fields:
  file        (file, required)     — the file to upload
  bucket_name (string, required)   — target MinIO bucket
  filename    (string, optional)   — override filename; if omitted, timestamped original name is used

Response 200:
{ "fileName": "20260101120000-report.pdf", "message": "File uploaded successfully" }

Errors:
  400 — no file, missing bucket_name, file >100MB, unsupported extension
  500 — MinIO write failure
```

Allowed extensions: `.jpg`, `.jpeg`, `.png`, `.svg`, `.pdf`, `.doc`, `.docx`, `.webp`

### POST /api/v1/minio/delete

```json
// Request
{ "fileName": "20260101120000-report.pdf", "bucketName": "my-bucket" }
// Response
{ "message": "File deleted successfully" }
```

Note: request body uses `fileName`/`bucketName` (camelCase), consistent with DTOs.

### POST /api/v1/minio/get-file-content

```json
// Request
{ "fileName": "20260101120000-report.pdf", "bucketName": "my-bucket" }
// Response — raw bytes as string; may be large
{ "content": "<file bytes as string>" }
```

### POST /api/v1/minio/get-file-url

Returns a presigned URL for direct browser access.

```json
// Request
{ "fileName": "20260101120000-report.pdf", "bucketName": "my-bucket" }
// Response
{ "url": "https://minio.example.com/my-bucket/20260101120000-report.pdf?X-Amz-..." }
```

### GET /api/v1/minio/file/:filename?bucket_name=X

Direct file download/stream. No request body.

```
200 — file bytes; Content-Type inferred from extension
      Images: Content-Disposition: inline
      Others: Content-Disposition: attachment; filename="<filename>"
400 — missing filename path param or bucket_name query param
404 — file not found in MinIO
```

### pkg/client interface

```go
type MinioAPIService interface {
    UploadFileV1(ctx *gin.Context, file *multipart.FileHeader, bucketName string, fileName string) (dto.FileUploadV1ResponseDTO, error)
    DeleteFileV1(ctx *gin.Context, payload dto.FileDeleteV1RequestDTO) (dto.FileDeleteV1ResponseDTO, error)
    GetFileContentV1(ctx *gin.Context, payload dto.FileContentV1RequestDTO) (dto.FileContentV1ResponseDTO, error)
    GetFileURLV1(ctx *gin.Context, payload dto.FileURLV1RequestDTO) (dto.FileURLV1ResponseDTO, error)
    GetFileV1(ctx *gin.Context, payload dto.FileGetV1RequestDTO) (io.ReadCloser, error)
}

// Constructor
func NewMinioAPIService(host string) MinioAPIService
```

### DTO shapes

```go
// Upload response
type FileUploadV1ResponseDTO struct {
    FileName string `json:"fileName"`
    Message  string `json:"message"`
}

// Delete
type FileDeleteV1RequestDTO struct {
    FileName   string `json:"fileName" binding:"required"`
    BucketName string `json:"bucketName" binding:"required"`
}
type FileDeleteV1ResponseDTO struct {
    Message string `json:"message"`
}

// Content
type FileContentV1RequestDTO struct {
    FileName   string `json:"fileName" binding:"required"`
    BucketName string `json:"bucketName" binding:"required"`
}
type FileContentV1ResponseDTO struct {
    Content string `json:"content"`
}

// URL
type FileURLV1RequestDTO struct {
    FileName   string `json:"fileName" binding:"required"`
    BucketName string `json:"bucketName" binding:"required"`
}
type FileURLV1ResponseDTO struct {
    URL string `json:"url"`
}

// Direct get (streaming)
type FileGetV1RequestDTO struct {
    FileName   string `json:"fileName" binding:"required"`
    BucketName string `json:"bucketName" binding:"required"`
}
```

## Configuration

| Env var | Type | Default | Description |
|---|---|---|---|
| `APP_PORT` | int | `8080` | Gin listen port |
| `APP_ENV` | string | `development` | `dev` or `production` |
| `MINIO_ENDPOINT` | string | `localhost:9000` | MinIO host:port (no scheme) |
| `MINIO_ACCESS_KEY` | string | `minioadmin` | MinIO access key |
| `MINIO_SECRET_KEY` | string | `minioadmin` | MinIO secret key |
| `MINIO_USE_SSL` | bool (string) | `false` | Set `true` for TLS MinIO |
| `TEST_ENV` | string | `""` | Test environment flag |

## Inter-service dependencies

| Dependency | How |
|---|---|
| go-web-platform | Library — SetupPlatform, error handling, logging, utils.SendRequest |
| MinIO | External S3-compatible storage via `minio-go` SDK |

No other go-web services are called.

## Extension guide

**Adding a new endpoint** (e.g., copy file):

1. Add method to `MinioService` interface in `internal/service/`
2. Implement using `minio-go` SDK
3. Add handler method in `internal/transport/http/handler/minio_handler.go`
4. Register route in `internal/transport/http/router.go`
5. Add request/response DTOs to `pkg/client/dto/dto.go`
6. Add method to `MinioAPIService` interface and implementation in `pkg/client/service/minio_api_service.go`
7. Tag new pkg/client version so consumers can `go get`

**Supporting a new file type:**

In `minio_handler.go` `UploadFileV1`, add extension to `allowedTypes` map:
```go
allowedTypes := map[string]bool{
    ".jpg": true,
    // ... add here
    ".mp4": true,
}
```

Also update `GetFile` content-type mapping if the type needs inline display.

## Common pitfalls

- **camelCase vs snake_case in DTOs** — HTTP request bodies use camelCase (`fileName`, `bucketName`). The upload form fields use snake_case (`file`, `bucket_name`, `filename`). Do not mix them up.
- **Storing original filename instead of returned `fileName`** — the service may prefix a timestamp. Always store the `fileName` from the upload response, not the original filename.
- **Using `get-file-content` for large files** — returns bytes as JSON string; for files >1MB this is wasteful. Use `GET /file/:filename` for user-facing downloads instead.
- **Not creating MinIO bucket before upload** — the service does not auto-create buckets. Buckets must exist in MinIO before upload calls succeed.
- **`pkg/client` is a separate Go module** — `go.mod` inside `pkg/client/` has its own module path `github.com/go-web-services/go-integration-minio/pkg/client`. Import and version it independently from the service itself.
- **SSL mismatch** — if `MINIO_USE_SSL=true` but MinIO has no valid cert, the SDK will reject the connection. Match SSL setting to actual MinIO config.
