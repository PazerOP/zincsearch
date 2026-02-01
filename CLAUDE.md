# CLAUDE.md - ZincSearch Development Guide

This document provides guidance for AI assistants working with the ZincSearch codebase.

## Project Overview

ZincSearch is a lightweight full-text search engine written in Go, designed as a drop-in alternative to Elasticsearch. It uses Bluge (forked as zincsearch/bluge) as the underlying indexing library and includes a Vue 3 web UI.

**Key characteristics:**
- Single binary deployment with embedded web UI
- Elasticsearch-compatible API alongside native Zinc API
- Schema-less indexing with sharding support
- Built-in authentication with role-based access control
- Write-ahead log (WAL) for data durability

## Technology Stack

### Backend (Go 1.25)
- **Web Framework:** Gin (gin-gonic/gin)
- **Full-text Search:** Bluge (zincsearch/bluge fork)
- **Metadata Storage:** BoltDB (default), Badger, or etcd
- **Logging:** zerolog
- **Metrics:** Prometheus client
- **Profiling:** Pyroscope
- **Error Tracking:** Sentry

### Frontend (Vue 3)
- **Build Tool:** Vite
- **UI Framework:** Quasar 2.8.4
- **State Management:** Vuex 4
- **Testing:** Vitest (unit), Cypress (e2e)

## Directory Structure

```
zincsearch/
├── cmd/zincsearch/          # Application entry point
│   └── main.go              # Server bootstrap, telemetry, graceful shutdown
├── pkg/                     # Core Go packages
│   ├── auth/                # User authentication and management
│   ├── bluge/               # Bluge customizations (aggregations, analysis, search)
│   ├── config/              # Configuration via environment variables
│   ├── core/                # Core indexing engine (Index, Shards, WAL, Search)
│   ├── handlers/            # HTTP request handlers
│   │   ├── auth/            # Login, user, role endpoints
│   │   ├── document/        # CRUD, bulk operations
│   │   ├── index/           # Index management
│   │   └── search/          # Search endpoints (v1 and v2/ES-compatible)
│   ├── ider/                # ID generation (Snowflake IDs)
│   ├── meta/                # Metadata models, version info
│   ├── metadata/            # Metadata storage layer (multi-backend)
│   ├── routes/              # HTTP routing and middleware
│   ├── wal/                 # Write-ahead log implementation
│   └── zutils/              # Utility functions
├── web/                     # Vue 3 frontend
│   ├── src/
│   │   ├── views/           # Page components
│   │   ├── components/      # Reusable components
│   │   ├── store/           # Vuex state
│   │   ├── router/          # Vue Router config
│   │   └── locales/         # i18n translations
│   └── package.json
├── test/                    # Go integration tests
│   └── api/                 # API endpoint tests
├── docs/                    # Generated Swagger documentation
├── k8s/                     # Kubernetes deployment manifests
└── helm/                    # Helm charts
```

## Development Workflow

### Quick Start

```bash
# 1. Build web assets (required before Go build)
cd web && npm install && npm run build && cd ..

# 2. Run the server
ZINC_FIRST_ADMIN_USER=admin ZINC_FIRST_ADMIN_PASSWORD=Complexpass#123 go run cmd/zincsearch/main.go
# Server runs on http://localhost:4080
```

### Development with Hot Reload

```bash
# Terminal 1: Backend
ZINC_FIRST_ADMIN_USER=admin ZINC_FIRST_ADMIN_PASSWORD=Complexpass#123 go run cmd/zincsearch/main.go

# Terminal 2: Frontend (hot reload)
cd web && npm run dev
# UI runs on http://localhost:8080
```

### Building

```bash
# Simple build
go build -o zincsearch cmd/zincsearch/main.go

# Full build with version info (use build.sh)
./build.sh

# Docker build
docker build --tag zincsearch:latest . -f Dockerfile
```

## Testing

### Running Tests

```bash
# Run all Go tests
./test.sh

# Run specific test pattern
./test.sh TestDocument

# Run benchmarks
./test.sh bench

# Generate coverage report (must be >= 81%)
./coverage.sh

# Frontend tests
cd web
npm run test          # Vitest unit tests
npm run test-ui       # Vitest with UI
npm run e2e           # Cypress e2e tests
npm run lint          # ESLint check
npm run lint-autofix  # Auto-fix lint issues
```

### Test Environment

Tests use these environment variables (set in test.sh):
```bash
ZINC_FIRST_ADMIN_USER=admin
ZINC_FIRST_ADMIN_PASSWORD=Complexpass#123
ZINC_WAL_SYNC_INTERVAL=10ms
ZINC_WAL_REDOLOG_NO_SYNC=true
```

### Test Patterns

Tests use Go's `testing` package with `testify/assert`:

```go
func TestDocument(t *testing.T) {
    t.Run("PUT /api/:target/_doc", func(t *testing.T) {
        t.Run("create document with exist indexName", func(t *testing.T) {
            body := bytes.NewBuffer(nil)
            body.WriteString(indexData)
            resp := request("PUT", "/api/"+indexName+"/_doc", body)
            assert.Equal(t, http.StatusOK, resp.Code)
        })
    })
}
```

## Code Conventions

### Go Code Style

1. **License Header:** All Go files must include the Apache 2.0 license header at the top
2. **Package Organization:** Handlers in `pkg/handlers/`, core logic in `pkg/core/`
3. **Error Handling:** Return errors to handlers, which render HTTP responses
4. **JSON Binding:** Use `zutils.GinBindJSON()` for request parsing
5. **Response Rendering:** Use `zutils.GinRenderJSON()` for responses
6. **Logging:** Use `github.com/rs/zerolog/log` for structured logging

### Handler Pattern

```go
func CreateUpdate(c *gin.Context) {
    indexName := c.Param("target")
    docID := c.Param("id")

    var doc map[string]interface{}
    if err := zutils.GinBindJSON(c, &doc); err != nil {
        zutils.GinRenderJSON(c, http.StatusBadRequest, meta.HTTPResponseError{Error: err.Error()})
        return
    }

    // Business logic...

    zutils.GinRenderJSON(c, http.StatusOK, meta.HTTPResponseID{...})
}
```

### Swagger Annotations

API handlers use Swagger annotations for documentation:

```go
// @Id IndexDocument
// @Summary Create or update document
// @security BasicAuth
// @Tags    Document
// @Accept  json
// @Produce json
// @Param   index     path  string  true  "Index"
// @Param   document  body  map[string]interface{}  true  "Document"
// @Success 200 {object} meta.HTTPResponseID
// @Failure 400 {object} meta.HTTPResponseError
// @Router /api/{index}/_doc [post]
func CreateUpdate(c *gin.Context) { ... }
```

After updating annotations, regenerate docs:
```bash
./swagger.sh
```

### Configuration

Configuration is via environment variables with struct tags:

```go
type config struct {
    ServerPort    string `env:"ZINC_SERVER_PORT,default=4080"`
    DataPath      string `env:"ZINC_DATA_PATH,default=./data"`
    MetadataStorage string `env:"ZINC_METADATA_STORAGE,default=bolt"`
}
```

## Key Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ZINC_SERVER_PORT` | 4080 | HTTP server port |
| `ZINC_DATA_PATH` | ./data | Data storage directory |
| `ZINC_METADATA_STORAGE` | bolt | Metadata backend (bolt, badger, etcd) |
| `ZINC_FIRST_ADMIN_USER` | - | Initial admin username (required first run) |
| `ZINC_FIRST_ADMIN_PASSWORD` | - | Initial admin password (required first run) |
| `ZINC_SHARD_NUM` | 3 | Number of shards per index |
| `ZINC_WAL_SYNC_INTERVAL` | 1s | WAL sync frequency |
| `ZINC_BATCH_SIZE` | 1024 | Batch operation size |
| `ZINC_MAX_RESULTS` | 10000 | Maximum search results |
| `ZINC_LOG_LEVEL` | debug | Log level (debug, info, warn, error) |
| `GIN_MODE` | - | Set to "release" for production |

## API Structure

### Native Zinc API (`/api/`)
- `POST /api/login` - Authentication
- `POST /api/:target/_doc` - Create document
- `PUT /api/:target/_doc/:id` - Create/update document
- `DELETE /api/:target/_doc/:id` - Delete document
- `POST /api/:target/_search` - Search (v1 format)
- `POST /api/_bulk` - Bulk operations

### Elasticsearch-Compatible API (`/es/`)
- `POST /es/:target/_search` - Search (ES DSL)
- `POST /es/:target/_msearch` - Multi-search
- `POST /es/_bulk` - Bulk operations (ES format)
- `PUT /es/:target` - Create index
- `GET /es/:target/_mapping` - Get mapping

## CI/CD Requirements

1. **Go test coverage >= 81%** (checked via codecov)
2. **ESLint validation** for JavaScript/Vue code
3. All tests must pass

## Common Tasks

### Adding a New API Endpoint

1. Create handler function in `pkg/handlers/`
2. Add Swagger annotations
3. Register route in `pkg/routes/routes.go`
4. Regenerate Swagger: `./swagger.sh`
5. Add tests in `test/api/`

### Adding a New Configuration Option

1. Add field to struct in `pkg/config/config.go` with `env` tag
2. Use `config.Global.FieldName` to access the value

### Modifying the Web UI

1. Make changes in `web/src/`
2. Test with `cd web && npm run dev`
3. Rebuild: `cd web && npm run build`
4. The built assets are embedded into the Go binary

## Important Dependencies

| Dependency | Purpose |
|-----------|---------|
| `zincsearch/bluge` | Full-text search indexing (forked from blugelabs/bluge) |
| `gin-gonic/gin` | HTTP web framework |
| `go.etcd.io/bbolt` | BoltDB for metadata storage |
| `rs/zerolog` | Structured logging |
| `swaggo/gin-swagger` | Swagger API documentation |

## Architecture Notes

- **Sharding:** Each index is divided into configurable shards (ZINC_SHARD_NUM)
- **WAL:** Write-ahead logging ensures durability before acknowledgment
- **Embedded UI:** Frontend assets are compiled into the Go binary via `embed`
- **Graceful Shutdown:** Server handles SIGTERM/SIGINT with proper cleanup
