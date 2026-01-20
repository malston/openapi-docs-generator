# OpenAPI Generator Plugin Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Claude Code plugin that analyzes Go API code and generates OpenAPI specs with a local Swagger UI viewer.

**Architecture:** Slash command `/generate-openapi` invokes an agent that scans Go files, extracts routes/handlers/models, generates an OpenAPI 3.0 YAML spec, and serves it via Swagger UI on a local Python HTTP server.

**Tech Stack:** Claude Code plugin (Markdown), Python 3 HTTP server, Swagger UI (static HTML/JS/CSS)

---

## Task 1: Create Plugin Manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

**Step 1: Create the plugin directory structure**

```bash
mkdir -p .claude-plugin commands agents assets/swagger-ui
```

**Step 2: Write the plugin manifest**

Create `.claude-plugin/plugin.json`:

```json
{
  "name": "openapi-skill",
  "version": "1.0.0",
  "description": "Generate OpenAPI/Swagger documentation from Go API code",
  "author": {
    "name": "Mark Alston"
  },
  "license": "MIT",
  "keywords": ["openapi", "swagger", "go", "api", "documentation"]
}
```

**Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "Add plugin manifest"
```

---

## Task 2: Create the Slash Command

**Files:**
- Create: `commands/generate-openapi.md`

**Step 1: Write the command file**

Create `commands/generate-openapi.md`:

```markdown
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
```

**Step 2: Commit**

```bash
git add commands/generate-openapi.md
git commit -m "Add /generate-openapi slash command"
```

---

## Task 3: Download and Bundle Swagger UI

**Files:**
- Create: `assets/swagger-ui/index.html`
- Create: `assets/swagger-ui/swagger-ui.css`
- Create: `assets/swagger-ui/swagger-ui-bundle.js`
- Create: `assets/swagger-ui/swagger-ui-standalone-preset.js`

**Step 1: Download Swagger UI assets from CDN**

```bash
cd assets/swagger-ui

# Download the main files from unpkg CDN
curl -sL "https://unpkg.com/swagger-ui-dist@5/swagger-ui.css" -o swagger-ui.css
curl -sL "https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js" -o swagger-ui-bundle.js
curl -sL "https://unpkg.com/swagger-ui-dist@5/swagger-ui-standalone-preset.js" -o swagger-ui-standalone-preset.js

cd ../..
```

**Step 2: Create the index.html template**

Create `assets/swagger-ui/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>API Documentation - Swagger UI</title>
  <link rel="stylesheet" href="swagger-ui.css">
</head>
<body>
  <div id="swagger-ui"></div>
  <script src="swagger-ui-bundle.js"></script>
  <script src="swagger-ui-standalone-preset.js"></script>
  <script>
    window.onload = function() {
      window.ui = SwaggerUIBundle({
        url: "openapi.yaml",
        dom_id: '#swagger-ui',
        presets: [
          SwaggerUIBundle.presets.apis,
          SwaggerUIStandalonePreset
        ],
        layout: "StandaloneLayout"
      });
    };
  </script>
</body>
</html>
```

**Step 3: Commit**

```bash
git add assets/swagger-ui/
git commit -m "Add bundled Swagger UI assets"
```

---

## Task 4: Create the OpenAPI Generator Agent

**Files:**
- Create: `agents/openapi-generator.md`

**Step 1: Write the agent file**

Create `agents/openapi-generator.md`:

````markdown
---
name: openapi-generator
description: Use this agent when generating OpenAPI documentation from Go code. Examples:

<example>
Context: User wants to generate API docs for a Go project
user: "/generate-openapi"
assistant: "I'll use the openapi-generator agent to analyze your Go API code and generate OpenAPI documentation."
<commentary>
The user invoked the generate-openapi command, so this agent should handle the analysis and generation.
</commentary>
</example>

<example>
Context: User wants to generate docs for a specific project
user: "/generate-openapi ~/workspace/my-api"
assistant: "I'll analyze the Go API code at ~/workspace/my-api and generate OpenAPI documentation."
<commentary>
The user specified a path, so the agent should analyze that directory.
</commentary>
</example>

model: inherit
color: cyan
tools: ["Read", "Glob", "Grep", "Bash", "Write"]
---

You are an expert Go developer specializing in API documentation. Your task is to analyze Go API code and generate accurate OpenAPI 3.0 specifications.

## Analysis Process

### Step 1: Detect Go Framework

Scan imports to determine framework:
- `net/http` only → Go standard library
- `github.com/gin-gonic/gin` → Gin framework

### Step 2: Find Route Definitions

**For standard library**, look for patterns:
- `http.HandleFunc("/path", handler)`
- `mux.HandleFunc("/path", handler)`
- `router.Handle("/path", handler)`
- Declarative route tables (slices of Route structs with Path/Method/Handler fields)

**For Gin**, look for patterns:
- `router.GET("/path", handler)`
- `router.POST("/path", handler)`
- `group.GET("/path", handler)`

### Step 3: Analyze Handlers

For each handler, extract:
- **Request body**: Look for `json.NewDecoder(r.Body).Decode(&var)` or `c.ShouldBindJSON(&var)`
- **Response body**: Look for `json.Encode(w, data)` or `c.JSON(status, data)`
- **Path parameters**: Extract from route path (`:id`, `{id}`)
- **Query parameters**: Look for `r.URL.Query().Get()` or `c.Query()`
- **Status codes**: Look for `w.WriteHeader()` or `c.JSON(status, ...)`

### Step 4: Parse Models

Find structs with `json:` tags:
- Extract field names and JSON keys
- Map Go types to OpenAPI types:
  - `string` → `string`
  - `int`, `int32`, `int64` → `integer`
  - `float32`, `float64` → `number`
  - `bool` → `boolean`
  - `[]T` → `array` with items
  - `map[string]T` → `object` with additionalProperties
  - `time.Time` → `string` with format `date-time`
- Use struct comments for descriptions

### Step 5: Generate OpenAPI Spec

Create `openapi.yaml` with:
- `openapi: "3.0.3"`
- `info` with title (from go.mod module name or directory name) and version "1.0.0"
- `paths` with all discovered endpoints
- `components/schemas` with all model definitions

### Step 6: Start Swagger UI Server

1. Create temp directory `.openapi-server/`
2. Copy Swagger UI assets from `${CLAUDE_PLUGIN_ROOT}/assets/swagger-ui/`
3. Copy generated `openapi.yaml` to `.openapi-server/`
4. Find available port (try 8080, 8081, 8082...)
5. Start Python HTTP server: `python3 -m http.server <port> --directory .openapi-server`
6. Report URL to user

## Output Format

After generation, provide:
1. Summary of discovered endpoints (count by method)
2. Summary of discovered models (count)
3. Path to generated `openapi.yaml`
4. URL for Swagger UI (e.g., `http://localhost:8080`)

## Edge Cases

- **No Go files found**: Report error and stop
- **Can't detect routes**: Ask user which file contains route definitions
- **Nested/embedded structs**: Flatten or create separate schema refs
- **Interface types**: Skip or use `object` type
- **Unexported fields**: Skip (they won't have json tags anyway)

## Quality Standards

- All paths must have at least one response defined
- All request/response bodies must reference schemas
- Use `$ref` for reusable schemas
- Include descriptions from comments where available
- Validate generated YAML is syntactically correct
````

**Step 2: Commit**

```bash
git add agents/openapi-generator.md
git commit -m "Add openapi-generator agent"
```

---

## Task 5: Add .gitignore

**Files:**
- Create: `.gitignore`

**Step 1: Write .gitignore**

Create `.gitignore`:

```
# OpenAPI server temp directory
.openapi-server/

# Generated spec (optional - user may want to commit)
# openapi.yaml

# OS files
.DS_Store

# Editor files
*.swp
*~
```

**Step 2: Commit**

```bash
git add .gitignore
git commit -m "Add .gitignore"
```

---

## Task 6: Test on diego-capacity-analyzer

**Files:**
- None (testing only)

**Step 1: Enable the plugin**

```bash
# From openapi-skill directory
claude mcp add-plugin .
```

Or manually add to Claude Code settings.

**Step 2: Test the command**

Open Claude Code and run:
```
/generate-openapi ~/workspace/diego-capacity-analyzer
```

**Step 3: Verify output**

Check that:
- [ ] Routes from `backend/handlers/routes.go` are detected
- [ ] Handler request/response types are extracted
- [ ] Models from `backend/models/` are parsed
- [ ] `openapi.yaml` is generated
- [ ] Swagger UI server starts and URL is printed
- [ ] Swagger UI loads and displays the API

**Step 4: Fix any issues**

Iterate on the agent prompt based on test results.

---

## Task 7: Documentation

**Files:**
- Create: `README.md`

**Step 1: Write README**

Create `README.md`:

```markdown
# OpenAPI Skill

A Claude Code plugin that generates OpenAPI/Swagger documentation from Go API code.

## Features

- Analyzes Go standard library (`net/http`) and Gin APIs
- Auto-detects routes, handlers, and request/response models
- Generates OpenAPI 3.0 specification
- Serves interactive Swagger UI for API browsing and testing

## Installation

```bash
claude mcp add-plugin /path/to/openapi-skill
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
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "Add README documentation"
```

---

## Summary

| Task | Description | Files |
|------|-------------|-------|
| 1 | Plugin manifest | `.claude-plugin/plugin.json` |
| 2 | Slash command | `commands/generate-openapi.md` |
| 3 | Swagger UI assets | `assets/swagger-ui/*` |
| 4 | Generator agent | `agents/openapi-generator.md` |
| 5 | Gitignore | `.gitignore` |
| 6 | Integration test | (testing) |
| 7 | Documentation | `README.md` |

Total: 7 tasks, ~6 files to create
