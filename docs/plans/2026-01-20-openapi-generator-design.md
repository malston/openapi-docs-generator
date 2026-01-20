# OpenAPI Generator Plugin Design

## Overview

A Claude Code plugin that analyzes Go API code and generates OpenAPI/Swagger documentation with a local browser-based viewer.

## Goals

- Analyze existing Go API code (standard library `net/http` and Gin)
- Auto-detect routes, handlers, and request/response models
- Generate valid OpenAPI 3.0 specification
- Serve interactive Swagger UI for browsing and testing

## Plugin Structure

```
openapi-skill/
├── plugin.json              # Plugin manifest
├── skills/
│   └── generate-openapi.md  # User-facing slash command
├── agents/
│   └── openapi-generator.md # Agent that performs analysis
└── assets/
    └── swagger-ui/          # Bundled Swagger UI (HTML/JS/CSS)
```

## Command Interface

### `/generate-openapi`

```
/generate-openapi                    # Analyze current directory
/generate-openapi ~/workspace/myapi  # Analyze specific path
```

**Output:**
- `openapi.yaml` - Generated OpenAPI spec in project root
- `.openapi-server/` - Temp directory for Swagger UI (gitignored)
- Prints URL to access Swagger UI (e.g., `http://localhost:8080`)

## Go Code Analysis

### Discovery Strategy

1. **Find route definitions** by scanning for patterns:
   - `http.HandleFunc` / `http.Handle` / `mux.HandleFunc` (stdlib)
   - `router.GET/POST/PUT/DELETE` (Gin)
   - Declarative route tables (slice of Route structs)

2. **Trace handlers** from routes to extract:
   - HTTP method and path
   - Request body type (`json.NewDecoder(r.Body).Decode(&var)`)
   - Response type (`json.Encode` or `c.JSON` calls)
   - Error responses

3. **Parse models** by finding structs with `json:` tags:
   - Extract field names, types, JSON keys
   - Use struct comments for descriptions
   - Handle nested types

### Framework Detection

Detect framework based on imports:
- `net/http` only → standard library
- `github.com/gin-gonic/gin` → Gin framework

### Fallback Behavior

If auto-discovery fails:
- No Go files → Error: "No Go files found"
- Can't detect routes → Prompt: "Which file contains your route definitions?"

## Swagger UI Server

### Implementation

1. Write `openapi.yaml` to project root
2. Create temp `.openapi-server/` directory containing:
   - `index.html` - Swagger UI configured to load spec
   - `openapi.yaml` - Copy of generated spec
3. Start `python3 -m http.server <port>` in background
4. Print URL to console

### Port Selection

Try ports in order: 8080, 8081, 8082, etc. until one is available.

### Why Python?

- Pre-installed on macOS and most Linux systems
- No external dependencies
- Simple one-liner to start

## Generated OpenAPI Spec

Example output:

```yaml
openapi: "3.0.3"
info:
  title: "API Title"
  version: "1.0.0"
paths:
  /api/v1/health:
    get:
      summary: "Health check"
      responses:
        "200":
          description: "Success"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/HealthResponse"
components:
  schemas:
    HealthResponse:
      type: object
      properties:
        status:
          type: string
```

## Scope

### In Scope (v1)

- Go standard library `net/http`
- Go Gin framework
- Auto-discovery with fallback prompts
- OpenAPI 3.0 spec generation
- Local Swagger UI server

### Out of Scope (Future)

- FastAPI/Python support
- Auto-browser opening
- Configuration file
- Watch mode / auto-regenerate
- Authentication documentation

## Test Target

Initial testing against: `~/workspace/diego-capacity-analyzer`

This project uses Go standard library with:
- Declarative route table in `backend/handlers/routes.go`
- Handler functions in `backend/handlers/*.go`
- Models with JSON tags in `backend/models/*.go`
