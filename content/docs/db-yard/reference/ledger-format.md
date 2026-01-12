---
title: Ledger Format
description: Schema and structure of db-yard's operational state files.
---

# Ledger Format

Schema and structure of db-yard's operational state files.

## Overview

The ledger is db-yard's complete operational record. All state is written as plain files: JSON for structured data, plain text for logs.

## Directory Structure

```
ledger.d/
├── 2026-01-12-10-30-00/              # Session directory
│   ├── apps/                          # Mirrors cargo structure
│   │   ├── my-app.db.context.json     # Service context
│   │   ├── my-app.db.stdout.log       # Process stdout
│   │   └── my-app.db.stderr.log       # Process stderr
│   └── evidence/
│       ├── audit.db.context.json
│       ├── audit.db.stdout.log
│       └── audit.db.stderr.log
└── watch/                             # Watch mode session
    └── ...
```

## Session Naming

### Materialize Mode

Session directories are named with timestamps:

```
YYYY-MM-DD-HH-MM-SS
```

Example: `2026-01-12-10-30-00`

### Watch Mode

Watch mode uses a stable session name:

```
watch
```

This allows continuous reconciliation without creating new directories.

## Context JSON

Each spawned service has a `*.context.json` file.

### Schema

```typescript
interface ServiceContext {
  // Identity
  serviceId: string;           // Unique identifier (usually filename)
  sessionId: string;           // Session directory name

  // Source
  provenance: string;          // Absolute path to source database
  relativePath: string;        // Path relative to cargo root

  // Classification
  kind: "sqlpage" | "surveilr"; // Service type

  // Network
  port: number;                // Assigned port number
  proxyPrefix: string;         // URL path for proxy routing

  // Process
  pid: number;                 // Process ID
  executable: string;          // Command used to start
  args: string[];              // Command line arguments

  // Timestamps
  spawnedAt: string;           // ISO 8601 timestamp
}
```

### Example

```json
{
  "serviceId": "dashboard.sqlite.db",
  "sessionId": "2026-01-12-10-30-00",
  "provenance": "/home/user/db-yard/cargo.d/apps/dashboard.sqlite.db",
  "relativePath": "apps/dashboard.sqlite.db",
  "kind": "sqlpage",
  "port": 3000,
  "proxyPrefix": "/apps/sqlpage/apps/dashboard",
  "pid": 12345,
  "executable": "sqlpage",
  "args": [
    "--port", "3000",
    "--database", "/home/user/db-yard/cargo.d/apps/dashboard.sqlite.db"
  ],
  "spawnedAt": "2026-01-12T10:30:00.000Z"
}
```

### Field Details

#### serviceId

Unique identifier for the service within a session. Derived from the database filename.

```
dashboard.sqlite.db → serviceId: "dashboard.sqlite.db"
```

#### sessionId

Name of the session directory. Used to group related services.

#### provenance

Absolute filesystem path to the source database. Used for:
- Smart spawn identity checking
- Log correlation
- Debugging

#### relativePath

Path relative to the cargo root. Used for:
- Ledger directory mirroring
- Proxy prefix derivation

#### kind

Service classification:
- `sqlpage` - SQLPage application
- `surveilr` - Surveilr RSSD

#### port

Allocated network port for the service.

#### proxyPrefix

URL path for proxy routing. Derived from:
```
/apps/{kind}/{relativePath-without-extension}
```

#### pid

Operating system process ID. Used for:
- Process management
- Reconciliation checks
- Log correlation

#### executable

The command used to spawn the service:
- `sqlpage` for SQLPage apps
- `surveilr` for surveilr RSSDs

#### args

Full command line arguments passed to the executable.

#### spawnedAt

ISO 8601 timestamp of when the process was spawned.

## Log Files

### stdout.log

Standard output from the spawned process.

**Format:** Plain text, appended in real-time.

**Example:**
```
[2026-01-12T10:30:01Z] SQLPage server starting
[2026-01-12T10:30:01Z] Listening on http://0.0.0.0:3000
[2026-01-12T10:30:01Z] Database: /path/to/dashboard.sqlite.db
[2026-01-12T10:30:05Z] Request: GET /
[2026-01-12T10:30:05Z] Response: 200 OK (45ms)
```

### stderr.log

Standard error from the spawned process.

**Format:** Plain text, appended in real-time.

**Example:**
```
[2026-01-12T10:35:00Z] Warning: Query took 500ms
[2026-01-12T10:40:00Z] Error: Connection reset by peer
```

## Mirrored Layout

The ledger mirrors the cargo directory structure:

```
cargo.d/                          ledger.d/session/
├── apps/                         ├── apps/
│   ├── dashboard.db        →     │   ├── dashboard.db.context.json
│   └── metrics.db          →     │   ├── dashboard.db.stdout.log
└── evidence/                     │   ├── dashboard.db.stderr.log
    └── audit.db            →     │   └── metrics.db.*
                                   └── evidence/
                                       └── audit.db.*
```

This makes finding the ledger entry for any cargo file trivial.

## File Naming

### Context Files

```
{original-filename}.context.json
```

Example: `dashboard.sqlite.db.context.json`

### Log Files

```
{original-filename}.stdout.log
{original-filename}.stderr.log
```

Example: `dashboard.sqlite.db.stdout.log`

## Parsing the Ledger

### List All Services

```bash
cat ledger.d/*/**.context.json | jq -s '.'
```

### Find by Port

```bash
cat ledger.d/*/**.context.json | jq 'select(.port == 3000)'
```

### Find by Kind

```bash
cat ledger.d/*/**.context.json | jq 'select(.kind == "sqlpage")'
```

### Extract Ports

```bash
cat ledger.d/*/**.context.json | jq -r '.port' | sort -n
```

### Check Process Status

```bash
for ctx in ledger.d/*/**.context.json; do
  pid=$(jq -r '.pid' "$ctx")
  if kill -0 "$pid" 2>/dev/null; then
    echo "Running: $ctx (PID $pid)"
  else
    echo "Dead: $ctx (PID $pid)"
  fi
done
```

## Ledger Lifecycle

### Creation

1. Service spawned
2. Context JSON written
3. Log files created (empty)

### Runtime

1. Logs appended as process outputs
2. Context JSON unchanged

### Cleanup

With `kill --clean`:
1. Processes terminated
2. Session directory removed

Without `--clean`:
1. Processes terminated
2. Ledger preserved for audit

## Best Practices

1. **Version control ledger** - Track operational history
2. **Rotate old sessions** - Clean up stale materialize sessions
3. **Monitor log size** - Long-running services accumulate logs
4. **Parse with jq** - JSON is queryable and scriptable
5. **Use for debugging** - Logs contain service output

---

See also: [Ledger System](../core-concepts/ledger-system.md), [CLI Reference](./cli.md)