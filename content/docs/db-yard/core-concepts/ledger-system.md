---
title: Ledger System
description: The operational state layer in db-yard.
---

# Ledger System

The operational state layer in db-yard.

## Overview

The ledger is db-yard's complete operational record. Every spawned service writes its state to disk as files you can read, version, copy, audit, or generate tooling around.

There is no internal database. The filesystem is the API.

## Ledger Structure

```
ledger.d/
├── 2026-01-12-10-30-00/              # Session directory
│   ├── apps/
│   │   ├── dashboard.db.context.json # Service metadata
│   │   ├── dashboard.db.stdout.log   # Process stdout
│   │   └── dashboard.db.stderr.log   # Process stderr
│   └── evidence/
│       ├── audit.db.context.json
│       ├── audit.db.stdout.log
│       └── audit.db.stderr.log
└── 2026-01-12-11-45-00/              # Another session
    └── ...
```

## Session Directories

### Materialize Mode

In one-shot mode (`start` without `--watch`), each run creates a timestamped session:

```
ledger.d/2026-01-12-10-30-00/
```

This creates an immutable snapshot of that run.

### Watch Mode

In continuous mode (`start --watch`), a stable session is used:

```
ledger.d/watch/
```

State is continuously reconciled rather than replaced.

## Mirrored Layout

Session directories mirror the cargo directory structure:

```
cargo.d/                      ledger.d/session/
├── apps/                     ├── apps/
│   └── dashboard.db    →     │   ├── dashboard.db.context.json
│                              │   ├── dashboard.db.stdout.log
│                              │   └── dashboard.db.stderr.log
└── evidence/                 └── evidence/
    └── audit.db        →         ├── audit.db.context.json
                                   ├── audit.db.stdout.log
                                   └── audit.db.stderr.log
```

This makes tracing a running service back to its source trivial.

## Context JSON

Each spawned service has a `*.context.json` file containing all metadata:

```json
{
  "serviceId": "dashboard.db",
  "sessionId": "2026-01-12-10-30-00",
  "provenance": "/absolute/path/to/cargo.d/apps/dashboard.db",
  "relativePath": "apps/dashboard.db",
  "kind": "sqlpage",
  "port": 3000,
  "proxyPrefix": "/apps/sqlpage/apps/dashboard",
  "pid": 12345,
  "spawnedAt": "2026-01-12T10:30:00.000Z",
  "executable": "sqlpage",
  "args": ["--port", "3000", "--database", "/absolute/path/to/cargo.d/apps/dashboard.db"]
}
```

### Context Fields

| Field | Description |
|-------|-------------|
| `serviceId` | Unique identifier (usually filename) |
| `sessionId` | Session directory name |
| `provenance` | Absolute path to source database |
| `relativePath` | Path relative to cargo root |
| `kind` | Classification (sqlpage, surveilr) |
| `port` | Assigned port number |
| `proxyPrefix` | URL prefix for proxy routing |
| `pid` | Process ID |
| `spawnedAt` | ISO timestamp of spawn |
| `executable` | Command used to start service |
| `args` | Command line arguments |

## Log Files

Each service has stdout and stderr logs:

### stdout.log

```
[sqlpage] Starting on port 3000
[sqlpage] Database: /path/to/dashboard.db
[sqlpage] Ready to accept connections
```

### stderr.log

```
[sqlpage] Warning: Large query detected
[sqlpage] Error: Connection timeout
```

Logs are written in real-time as the process runs.

## Using the Ledger

### List Ledger Contents

```bash
./bin/yard.ts ls
```

### Read Context Files

```bash
cat ledger.d/*/apps/dashboard.db.context.json | jq .
```

### Tail Logs

```bash
tail -f ledger.d/watch/apps/dashboard.db.stdout.log
```

### Generate Reports

```bash
# List all ports in use
cat ledger.d/*/**.context.json | jq -r '.port'

# Find services by type
cat ledger.d/*/**.context.json | jq -r 'select(.kind == "sqlpage") | .serviceId'
```

## Reconciliation

The `ps --reconcile` command compares ledger to live processes:

```bash
./bin/yard.ts ps --reconcile
```

This highlights:
- Ledger entries without running processes (crashed?)
- Processes without ledger entries (orphaned?)

## Ledger Cleanup

Remove ledger and kill processes:

```bash
./bin/yard.ts kill --clean
```

This:
1. Kills all tagged processes
2. Removes session directories

## Ledger as API

Because the ledger is just files, you can:

1. **Version control it** - Track operational history
2. **Copy it** - Replicate state to another machine
3. **Parse it** - Build tooling with jq, scripts, etc.
4. **Generate from it** - Create NGINX configs, monitoring rules
5. **Audit it** - Review what ran and when

---

Next: [Port Allocation](./port-allocation.md) - How ports are assigned.