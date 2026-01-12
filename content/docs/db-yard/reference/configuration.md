---
title: Configuration Reference
description: Environment variables, defaults, and configuration options for db-yard.
---

# Configuration Reference

Environment variables, defaults, and configuration options for db-yard.

## Overview

db-yard follows a convention-over-configuration approach. Most settings have sensible defaults that can be overridden via CLI flags or environment variables.

## Configuration Hierarchy

Settings are resolved in this order (later overrides earlier):

1. Built-in defaults
2. Environment variables
3. CLI flags

## Defaults

| Setting | Default | Description |
|---------|---------|-------------|
| Cargo home | `./cargo.d` | Where to find database cargo |
| Ledger home | `./ledger.d` | Where to write operational state |
| Port base | `3000` | Starting port for allocation |
| Web UI host | `127.0.0.1` | Web server bind address |
| Web UI port | `8787` | Web server listen port |

## Environment Variables

### Core Settings

| Variable | Type | Description |
|----------|------|-------------|
| `DB_YARD_CARGO_HOME` | Path | Default cargo directory |
| `DB_YARD_LEDGER_HOME` | Path | Default ledger directory |
| `DB_YARD_PORT_BASE` | Number | Starting port for allocation |

### Web UI Settings

| Variable | Type | Description |
|----------|------|-------------|
| `DB_YARD_WEB_HOST` | String | Web UI bind address |
| `DB_YARD_WEB_PORT` | Number | Web UI listen port |

### Output Settings

| Variable | Type | Description |
|----------|------|-------------|
| `NO_COLOR` | Boolean | Disable colored output |
| `DB_YARD_VERBOSE` | String | Default verbosity level |

### Example

```bash
# Set environment variables
export DB_YARD_CARGO_HOME=/var/db-yard/cargo
export DB_YARD_LEDGER_HOME=/var/db-yard/ledger
export DB_YARD_PORT_BASE=8000

# Run with env-based config
./bin/yard.ts start
```

## CLI Flags

CLI flags always override environment variables.

### yard.ts start

| Flag | Env Equivalent | Description |
|------|----------------|-------------|
| `--cargo-home` | `DB_YARD_CARGO_HOME` | Cargo directory |
| `--ledger-home` | `DB_YARD_LEDGER_HOME` | Ledger directory |
| `--port-base` | `DB_YARD_PORT_BASE` | Starting port |
| `--verbose` | `DB_YARD_VERBOSE` | Output verbosity |
| `--watch` | - | Enable watch mode |

### web-ui/serve.ts

| Flag | Env Equivalent | Description |
|------|----------------|-------------|
| `--host` | `DB_YARD_WEB_HOST` | Bind address |
| `--port` | `DB_YARD_WEB_PORT` | Listen port |
| `--ledger-dir` | `DB_YARD_LEDGER_HOME` | Ledger directory |
| `--no-proxy` | - | Disable proxy |

## Discovery Configuration

### Glob Patterns

The following patterns are used for discovery (not currently configurable):

```
**/*.db
**/*.sqlite
**/*.sqlite3
**/*.sqlite.db
**/*.duckdb
**/*.xlsx
```

### Ignored Paths

These patterns are always ignored:

- Hidden files (`.something`)
- Hidden directories (`.git/`, `.cache/`)
- Common build artifacts (`node_modules/`, `target/`)

## Process Configuration

### Service Executables

| Classification | Executable | Notes |
|---------------|------------|-------|
| SQLPage app | `sqlpage` | Must be in PATH |
| Surveilr RSSD | `surveilr` | Must be in PATH |

### Process Tagging

db-yard tags spawned processes with environment variables:

| Variable | Value |
|----------|-------|
| `DB_YARD_SERVICE_ID` | Service identifier |
| `DB_YARD_SESSION_ID` | Session identifier |
| `DB_YARD_PROVENANCE` | Source file path |

This allows db-yard to identify and manage its processes.

## Directory Structure

### Cargo Directory

```
cargo.d/                    # Cargo root
├── apps/                   # Subdirectory (becomes URL path)
│   └── my-app.sqlite.db    # Database cargo
└── evidence/
    └── audit.db
```

### Ledger Directory

```
ledger.d/                           # Ledger root
├── 2026-01-12-10-30-00/            # Session (materialize mode)
│   └── apps/
│       ├── my-app.sqlite.db.context.json
│       ├── my-app.sqlite.db.stdout.log
│       └── my-app.sqlite.db.stderr.log
└── watch/                          # Session (watch mode)
    └── ...
```

## Port Configuration

### Allocation

- Starts at `--port-base` (default 3000)
- Increments by 1 for each service
- Skips ports already in use
- Maximum search range: 1000 ports

### Reserved Ports

Consider avoiding:

| Port | Common Use |
|------|------------|
| 80 | HTTP |
| 443 | HTTPS |
| 5432 | PostgreSQL |
| 3306 | MySQL |
| 8080 | Common HTTP alt |

## Proxy Configuration

### Built-in Proxy

The web UI includes a transparent proxy:

| Setting | Value |
|---------|-------|
| Proxy root | `/.db-yard/` |
| UI path | `/.db-yard/ui/` |
| API path | `/.db-yard/api/` |
| App paths | `/apps/...` |

### External Proxy Generation

Generate configuration for production proxies:

```bash
./bin/yard.ts proxy-conf --type nginx
./bin/yard.ts proxy-conf --type traefik
```

## Example Configurations

### Development

```bash
# Use defaults
./bin/yard.ts start
./bin/web-ui/serve.ts
```

### Production

```bash
# Environment-based config
export DB_YARD_CARGO_HOME=/var/db-yard/cargo
export DB_YARD_LEDGER_HOME=/var/db-yard/ledger
export DB_YARD_PORT_BASE=9000
export DB_YARD_WEB_HOST=0.0.0.0
export DB_YARD_WEB_PORT=8787

./bin/yard.ts start --watch
./bin/web-ui/serve.ts
```

### Container

```dockerfile
ENV DB_YARD_CARGO_HOME=/data/cargo
ENV DB_YARD_LEDGER_HOME=/data/ledger
ENV DB_YARD_WEB_HOST=0.0.0.0
```

---

See also: [CLI Reference](./cli.md), [Production Deployment](../how-to-guides/production-deployment.md)