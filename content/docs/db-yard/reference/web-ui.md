---
title: Web UI Reference
description: Browser-based administration interface and API endpoints.
---

# Web UI Reference

Browser-based administration interface and API endpoints.

## Overview

db-yard includes a web-based administration interface for managing services, browsing the ledger, and debugging proxy routing.

## Starting the Web UI

```bash
./bin/web-ui/serve.ts [OPTIONS]
```

**Options:**

| Option | Default | Description |
|--------|---------|-------------|
| `--host` | `127.0.0.1` | Bind address |
| `--port` | `8787` | Listen port |
| `--ledger-dir` | `./ledger.d` | Ledger directory |
| `--assets-dir` | Built-in | Static assets |
| `--no-proxy` | `false` | Disable proxy |

**Example:**

```bash
# Start on default port
./bin/web-ui/serve.ts

# Custom port and host
./bin/web-ui/serve.ts --host 0.0.0.0 --port 9000
```

## URL Structure

| Path | Purpose |
|------|---------|
| `/` | Redirects to `/.db-yard/ui/` |
| `/.db-yard/ui/` | Main UI interface |
| `/.db-yard/api/` | API endpoints |
| `/apps/sqlpage/...` | Proxied SQLPage services |
| `/apps/surveilr/...` | Proxied surveilr services |

## UI Sections

### Running Services

Shows all currently running services managed by db-yard.

**Columns:**

| Column | Description |
|--------|-------------|
| PID | Process ID |
| Upstream | Direct service URL (e.g., `http://localhost:3000`) |
| Proxied | Proxy URL (e.g., `/apps/sqlpage/my-app`) |
| Service ID | Service identifier |
| Session ID | Session identifier (tooltip) |
| Ledger Context | Link to context JSON |
| Actions | View stdout/stderr logs |

### Reconciliation Table

Compares live processes against ledger entries.

**Status Indicators:**

| Status | Meaning |
|--------|---------|
| Running | Process exists, ledger exists |
| Orphaned | Process exists, no ledger entry |
| Missing | Ledger exists, no process |

### Ledger Browser

Direct file browser for the ledger directory.

- Navigate session directories
- View context JSON files
- Download or view logs

## API Endpoints

### Service List

```
GET /.db-yard/api/services.json
```

Returns all running services:

```json
[
  {
    "serviceId": "my-app.sqlite.db",
    "pid": 12345,
    "port": 3000,
    "proxyPrefix": "/apps/sqlpage/my-app",
    "kind": "sqlpage",
    "provenance": "/path/to/cargo.d/my-app.sqlite.db"
  }
]
```

### Reconciliation

```
GET /.db-yard/api/reconcile.json
```

Returns comparison of ledger vs live processes:

```json
{
  "running": [...],
  "orphaned": [...],
  "missing": [...]
}
```

### Ledger Contents

```
GET /.db-yard/api/ledger.json
```

Returns all context files from ledger.

### Proxy Debug

```
GET /.db-yard/api/proxy-debug.json?path=/apps/sqlpage/my-app/index.sql
```

Shows how a path would be routed:

```json
{
  "requestPath": "/apps/sqlpage/my-app/index.sql",
  "matchedPrefix": "/apps/sqlpage/my-app",
  "upstreamUrl": "http://127.0.0.1:3000/index.sql",
  "serviceId": "my-app.sqlite.db",
  "headers": {
    "Host": "127.0.0.1:3000",
    "X-Forwarded-For": "[redacted]"
  }
}
```

### Proxy Roundtrip

```
GET /.db-yard/api/proxy-roundtrip.json?path=/apps/sqlpage/my-app/
```

Performs actual upstream request:

```json
{
  "requestPath": "/apps/sqlpage/my-app/",
  "upstreamUrl": "http://127.0.0.1:3000/",
  "status": 200,
  "latencyMs": 45,
  "headers": {
    "content-type": "text/html"
  },
  "preview": "<!DOCTYPE html>..."
}
```

## Proxy Features

### Transparent Proxying

All requests to `/apps/...` are proxied to upstream services:

```
Request:  GET /apps/sqlpage/my-app/page.sql
Matches:  /apps/sqlpage/my-app → port 3000
Forwards: GET http://localhost:3000/page.sql
```

### Request Tracing

Enable tracing for any proxied request:

**Query Parameter:**
```
GET /apps/sqlpage/my-app/?__db_yard_trace=1
```

**Header:**
```
X-Db-Yard-Trace: 1
```

**Trace Response Headers:**

| Header | Value |
|--------|-------|
| `X-Db-Yard-Matched-Prefix` | The matched proxy prefix |
| `X-Db-Yard-Upstream` | The upstream URL used |
| `X-Db-Yard-Service-Id` | The service identifier |

**Server Log:**

```
[proxy] TRACE path=/apps/sqlpage/my-app/ upstream=http://127.0.0.1:3000/ status=200 latency=45ms
```

### Header Forwarding

The proxy forwards these headers:

| Header | Value |
|--------|-------|
| `Host` | Original host |
| `X-Real-IP` | Client IP |
| `X-Forwarded-For` | Proxy chain |
| `X-Forwarded-Proto` | Original protocol |
| `X-Forwarded-Host` | Original host |

## Static Assets

The web UI serves static assets from `bin/web-ui/asset/`:

| File | Purpose |
|------|---------|
| `index.html` | Main UI page |
| `app.js` | Frontend JavaScript |
| `styles.css` | UI styling |

## Intended Use

The built-in proxy is designed for:

- Local development
- Testing and debugging
- Lightweight demos
- Validating routing

For production, use industrial-grade proxies (NGINX, Traefik) with generated configuration:

```bash
./bin/yard.ts proxy-conf --type nginx > /etc/nginx/conf.d/db-yard.conf
```

## Security Considerations

### Default Binding

By default, the web UI binds to `127.0.0.1` (localhost only).

To expose externally:
```bash
./bin/web-ui/serve.ts --host 0.0.0.0
```

### No Authentication

The web UI has no built-in authentication. For production:

1. Keep bound to localhost
2. Use a reverse proxy with auth
3. Restrict network access

### Sensitive Data

The proxy debug endpoints may expose:
- Internal service URLs
- Port allocations
- File paths

Consider disabling in production or restricting access.

---

See also: [CLI Reference](./cli.md), [Reverse Proxy Configuration](../advanced/reverse-proxy-conf.md)