---
title: CLI Command Reference
description: Complete reference for all db-yard CLI commands.
---

# CLI Command Reference

Complete reference for all db-yard CLI commands.

## Overview

db-yard provides a command-line interface through `bin/yard.ts`. All commands follow the pattern:

```bash
./bin/yard.ts <command> [options]
```

## Global Options

Options available across all commands:

```
--help, -h      Show help message
--version, -V   Show version
```

## Commands

### start

Scan cargo, spawn exposable services, and write state to ledger.

```bash
./bin/yard.ts start [OPTIONS]
```

**Options:**

| Option | Description |
|--------|-------------|
| `--cargo-home <path>` | Cargo directory path (default: `./cargo.d`) |
| `--ledger-home <path>` | Ledger directory path (default: `./ledger.d`) |
| `--verbose <level>` | Output verbosity: `essential` or `comprehensive` |
| `--watch` | Enable continuous watch and reconcile mode |
| `--port-base <port>` | Starting port for allocation (default: 3000) |

**Examples:**

```bash
# Basic start
./bin/yard.ts start

# With custom paths
./bin/yard.ts start \
  --cargo-home /var/db-yard/cargo \
  --ledger-home /var/db-yard/ledger

# Verbose output
./bin/yard.ts start --verbose comprehensive

# Watch mode (continuous)
./bin/yard.ts start --watch
```

**Behavior:**

- **Without --watch**: Materializes services and exits
- **With --watch**: Runs continuously, reconciling filesystem changes

**Watch Mode:**

In watch mode:
- New cargo files are automatically spawned
- Removed cargo files trigger process cleanup
- Crashed processes are respawned
- Press `Ctrl+C` to stop (cleans up all spawned processes)

---

### ps

List spawned processes (similar to Linux `ps`).

```bash
./bin/yard.ts ps [OPTIONS]
```

**Options:**

| Option | Description |
|--------|-------------|
| `--extended` | Show extended information |
| `--reconcile` | Compare ledger with live processes |

**Examples:**

```bash
# Basic listing
./bin/yard.ts ps

# Extended info
./bin/yard.ts ps --extended

# Show drift between ledger and live
./bin/yard.ts ps --reconcile
```

**Output Fields:**

| Field | Description |
|-------|-------------|
| PID | Process ID |
| PORT | Assigned port |
| TYPE | Service type (sqlpage, surveilr) |
| DATABASE | Path to source database |
| PROXY PREFIX | URL path for proxy routing |

**Reconcile Output:**

The `--reconcile` flag shows:
- Ledger entries without running processes (crashed?)
- Processes without ledger entries (orphaned?)

---

### ls

List all discovered cargo (without spawning).

```bash
./bin/yard.ts ls [OPTIONS]
```

**Options:**

| Option | Description |
|--------|-------------|
| `--cargo-home <path>` | Cargo directory path |

**Examples:**

```bash
# List all cargo
./bin/yard.ts ls

# Custom cargo location
./bin/yard.ts ls --cargo-home /var/db-yard/cargo
```

**Output:**

Shows all discovered database files with:
- Relative path
- File size
- Classification (if determinable)

---

### kill

Stop all spawned processes.

```bash
./bin/yard.ts kill [OPTIONS]
```

**Options:**

| Option | Description |
|--------|-------------|
| `--clean` | Also remove ledger session directories |

**Examples:**

```bash
# Kill processes only
./bin/yard.ts kill

# Kill and cleanup ledger
./bin/yard.ts kill --clean
```

**Behavior:**

- Sends SIGTERM to all tagged db-yard processes
- Waits for graceful shutdown
- With `--clean`, removes session directories from ledger

---

### proxy-conf

Generate reverse proxy configuration from ledger state.

```bash
./bin/yard.ts proxy-conf [OPTIONS]
```

**Options:**

| Option | Description |
|--------|-------------|
| `--type <type>` | Proxy type: `nginx`, `traefik`, or `both` |
| `--ledger-home <path>` | Ledger directory path |
| `--prefix <prefix>` | Override base prefix |
| `--strip-prefix` | Strip prefix when forwarding |

**Examples:**

```bash
# Generate NGINX config
./bin/yard.ts proxy-conf --type nginx

# Generate Traefik config
./bin/yard.ts proxy-conf --type traefik

# Generate both
./bin/yard.ts proxy-conf --type both
```

**NGINX Output Example:**

```nginx
location /apps/sqlpage/controls/scf-2025/ {
    proxy_pass http://127.0.0.1:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

**Traefik Output Example:**

```yaml
http:
  routers:
    scf-2025:
      rule: "PathPrefix(`/apps/sqlpage/controls/scf-2025`)"
      service: scf-2025
  services:
    scf-2025:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:3000"
```

---

### cc (Composite Connections)

Generate SQL DDL for composite database connections.

```bash
./bin/yard.ts cc [OPTIONS]
```

**Options:**

| Option | Description |
|--------|-------------|
| `--volume-root <path>` | Root directory for database volumes |
| `--scope <scope>` | Scope: `admin`, `tenant`, or `cross-tenant` |
| `--tenant-id <id>` | Tenant ID (required for tenant scope) |
| `--dialect <dialect>` | SQL dialect: `SQLite` or `DuckDB` |
| `--duckdb-sqlite-ext` | Include DuckDB SQLite extension preamble |

**Examples:**

```bash
# SQLite composite for admin scope
./bin/yard.ts cc --volume-root /var/db-yard --scope admin

# Tenant-specific composite
./bin/yard.ts cc --volume-root /var/db-yard --scope tenant --tenant-id tenant-123

# DuckDB composite with SQLite extension
./bin/yard.ts cc --dialect DuckDB --duckdb-sqlite-ext --volume-root /var/db-yard
```

**Output Example (SQLite):**

```sql
-- Composite connection for admin scope
-- Generated by db-yard

ATTACH DATABASE '/var/db-yard/embedded/admin/db0.sqlite.db' AS db0;
ATTACH DATABASE '/var/db-yard/embedded/admin/db1.sqlite.db' AS db1;

-- Optional views can be added here
```

**Output Example (DuckDB):**

```sql
-- Composite connection for cross-tenant scope
-- Generated by db-yard

INSTALL sqlite;
LOAD sqlite;

ATTACH '/var/db-yard/embedded/tenant/a/db.sqlite.db' AS tenant_a (TYPE sqlite);
ATTACH '/var/db-yard/embedded/tenant/b/db.sqlite.db' AS tenant_b (TYPE sqlite);
```

---

## Web UI Server

The web UI is started separately:

```bash
./bin/web-ui/serve.ts [OPTIONS]
```

**Options:**

| Option | Description |
|--------|-------------|
| `--host <host>` | Bind address (default: `127.0.0.1`) |
| `--port <port>` | Listen port (default: `8787`) |
| `--ledger-dir <path>` | Ledger directory path |
| `--assets-dir <path>` | Static assets directory |
| `--no-proxy` | Disable built-in proxy |

**Examples:**

```bash
# Default start
./bin/web-ui/serve.ts

# Custom port
./bin/web-ui/serve.ts --port 9000

# Bind to all interfaces
./bin/web-ui/serve.ts --host 0.0.0.0
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Invalid arguments |
| 130 | Interrupted (Ctrl+C) |

---

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `DB_YARD_CARGO_HOME` | Default cargo directory |
| `DB_YARD_LEDGER_HOME` | Default ledger directory |
| `DB_YARD_PORT_BASE` | Default starting port |
| `NO_COLOR` | Disable colored output |

---

See also: [Configuration Reference](./configuration.md), [Web UI Reference](./web-ui.md)