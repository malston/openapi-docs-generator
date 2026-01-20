---
name: generate-openapi
description: Generate OpenAPI documentation from Go API code and serve it via Swagger UI
allowed-args:
  - name: path
    description: Path to the Go project (defaults to current directory)
    required: false
---

Generate OpenAPI documentation for a Go API project.

**Usage:**
- `/generate-openapi` - Analyze current directory
- `/generate-openapi ~/workspace/myapi` - Analyze specific path

This command will:
1. Scan Go files to detect routes, handlers, and models
2. Generate an OpenAPI 3.0 specification (`openapi.yaml`)
3. Start a local Swagger UI server for browsing and testing the API

Use the `openapi-generator` agent to perform the analysis and generation.
