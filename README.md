# OpenAPI Docs Generator Agent & Claude Plugin

A Claude Code plugin that generates OpenAPI/Swagger documentation from Go API code.

## Features

- Analyzes Go standard library (`net/http`) and Gin APIs
- Auto-detects routes, handlers, and request/response models
- Generates OpenAPI 3.0 specification
- Serves interactive Swagger UI for API browsing and testing

## Installation

```bash
claude plugin marketplace add /path/to/openapi-docs-generator
claude plugin install openapi@openapi-dev
```

## Usage

```bash
# Analyze current directory
/generate-openapi

# Analyze specific path
/generate-openapi ~/workspace/my-api
```

## Output

- `openapi.yaml` - Generated OpenAPI specification
- Swagger UI server at `http://localhost:8080` (or next available port)

## Supported Frameworks

- Go standard library (`net/http`, `http.ServeMux`)
- Gin (`github.com/gin-gonic/gin`)

## Requirements

- Python 3 (for HTTP server)
