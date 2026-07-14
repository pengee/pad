# Pad — Build Instructions

## Build requirements

### Go (backend)
- Go 1.26.5 (per `go.mod`)
- `CGO_ENABLED=0` — pure Go SQLite, no C toolchain needed

### Node.js (web UI)
- Node.js (Dockerfile uses `node:26-alpine`)
- Vite + SvelteKit for the embedded SPA

## Step 1: Clone the repo

```bash
git clone https://github.com/PerpetualSoftware/pad.git
cd pad
```

## Step 2: Build web UI

```bash
cd web && npm ci && npm run build && cd ..
```

Output goes to `web/build/`. This gets embedded into the Go binary at build time.

## Step 3: Build Go binary

### Docker (recommended)
```bash
docker build --build-arg VERSION=1.0.0 \
             --build-arg COMMIT=$(git rev-parse HEAD) \
             --build-arg BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
             -t pad .
```

### Local (without Docker)
```bash
CGO_ENABLED=0 go build -ldflags="-s -w -X main.version=dev" -o pad ./cmd/pad
```

## Step 4: Run the container

```bash
docker run -d \
  --name pad \
  -p 7777:7777 \
  -v pad-data:/data \
  -e PAD_DATA_DIR=/data \
  -e PAD_HOST=0.0.0.0 \
  pad
```

### Optional environment variables
| Variable | Purpose | Example |
|----------|---------|---------|
| `PAD_URL` | Public base URL for generated links | `https://app.example.com` |
| `PAD_EMAIL_API_KEY` | Maileroo API key for transactional email | (API key) |
| `PAD_METRICS_TOKEN` | Bearer token for `/metrics` scrape endpoint | (secret) |
| `PAD_CORS_ORIGINS` | Comma-separated allowed CORS origins | `https://app.example.com` |
| `PAD_SECURE_COOKIES` | Set `Secure` flag on cookies | `true` |
| `PAD_TRUSTED_PROXIES` | CIDRs allowed to set `X-Forwarded-For` | `10.0.0.0/8` |
| `PAD_IMPORT_BUNDLE_MAX_BYTES` | Cap on workspace import bundle size | `0` (default 2 GiB) |
| `PAD_SSE_MAX_CONNECTIONS` | Global SSE connection limit | `0` (unlimited) |
| `PAD_SSE_MAX_PER_WORKSPACE` | Per-workspace SSE connection limit | `0` (unlimited) |

## Step 5: First-run setup

The container listens on port `7777`. On first startup, the bootstrap endpoint creates the admin user.

### Check health
```bash
curl http://localhost:7777/api/v1/health
```

### Create admin (self-host)
```bash
# The entrypoint prints a bootstrap token to stdout — capture it from logs
docker logs pad

# POST with the bootstrap token
curl -X POST http://localhost:7777/api/v1/auth/bootstrap \
  -H "Content-Type: application/json" \
  -H "X-Bootstrap-Token: <token-from-logs>" \
  -d '{"email":"admin@example.com","password":"<password>"}'
```

### Optional: enable bypass-setup-token (skip logs-token flow)
```bash
docker run -d \
  --name pad \
  -p 7777:7777 \
  -v pad-data:/data \
  -e PAD_BYPASS_SETUP_TOKEN=true \
  ...
```
With this flag, the first admin can be created via the web UI form without copying a token from logs. Only use on trusted networks (LAN, Tailscale, etc.).

## Step 6: Enable MCP endpoint (self-hosted)

The stock build does **not** expose a web-accessible `/mcp` endpoint — it's gated behind cloud mode. To enable it, apply these three patches before building.

### Patch 1 — Wire transport on self-hosted (`cmd/pad/main.go`)

In `cmd/pad/main.go`, find the cloud-mode block where `SetMCPTransport` is called (inside `if s.IsCloud()`). Add a parallel path for self-hosted:

```go
// After the cloud-mode SetMCPTransport call, add:
if !s.IsCloud() && os.Getenv("PAD_MCP_PUBLIC_URL") != "" {
    transport := mcpNewStreamableHTTPServer(mcpServer)  // mcp-go's NewStreamableHTTPServer
    s.SetMCPTransport(transport, os.Getenv("PAD_MCP_PUBLIC_URL"), "")
}
```

### Patch 2 — Drop cloud gate (`internal/server/handlers_mcp.go`)

In `registerMCPRoutes`, change:

```go
// Before:
if s.mcpTransport == nil || !s.IsCloud() {
    return
}

// After:
if s.mcpTransport == nil {
    return
}
```

### Patch 3 — Open discovery endpoint (`internal/server/handlers_mcp.go`)

In `registerMCPRoutes`, change:

```go
// Before:
r.With(s.requireCloudMode).Get("/.well-known/oauth-protected-resource", s.handleOAuthProtectedResource)

// After:
r.Get("/.well-known/oauth-protected-resource", s.handleOAuthProtectedResource)
```

### Run with MCP enabled

```bash
docker run -d \
  --name pad \
  -p 7777:7777 \
  -v pad-data:/data \
  -e PAD_DATA_DIR=/data \
  -e PAD_HOST=0.0.0.0 \
  -e PAD_MCP_PUBLIC_URL=https://mcp.example.com \
  pad
```

### MCP auth

Self-hosted uses **Personal Access Tokens** (PAT). Generate one from the web UI (Settings → API Tokens), then connect MCP clients with:

```json
{
  "mcpServers": {
    "pad": {
      "command": "http",
      "url": "http://localhost:7777/mcp",
      "headers": {
        "Authorization": "Bearer pad_<your-token-here>"
      }
    }
  }
}
```

The OAuth introspection path is effectively disabled (returns `401` when no OAuth server is present). PAT tokens are validated against the `api_tokens` table in SQLite.

### Optional: MCP session TTL

```bash
-e PAD_MCP_SESSION_TTL=30m \
-e PAD_MCP_SESSION_SWEEP_INTERVAL=5m \
```
Defaults to 30m TTL, 5m sweep interval.
