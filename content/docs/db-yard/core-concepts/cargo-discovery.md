---
title: Cargo Discovery
description: How db-yard finds databases in your filesystem.
---

# Cargo Discovery

How db-yard finds databases in your filesystem.

## Overview

Discovery is the first step in db-yard's pipeline. It recursively walks one or more cargo roots using deterministic glob patterns to find all potential database files.

## Glob Patterns

By default, db-yard looks for these file patterns:

| Pattern | Description |
|---------|-------------|
| `**/*.db` | Generic database files |
| `**/*.sqlite` | SQLite databases |
| `**/*.sqlite3` | SQLite3 databases |
| `**/*.sqlite.db` | Compound SQLite extension |
| `**/*.duckdb` | DuckDB databases |
| `**/*.xlsx` | Excel files (discovered but not exposable) |

The `**` glob means recursive - db-yard searches all subdirectories.

## Discovery Process

```
1. Receive cargo root(s)
   cargo.d/

2. Apply glob patterns
   **/*.db, **/*.sqlite, ...

3. Yield encounters
   cargo.d/apps/dashboard.db
   cargo.d/evidence/audit.sqlite.db
   cargo.d/reports/metrics.duckdb
```

## Cargo Roots

A cargo root is the base directory db-yard scans. You can specify multiple roots:

```bash
./bin/yard.ts start \
  --cargo-home ./cargo.d \
  --cargo-home /var/db-yard/cargo \
  --cargo-home ~/my-databases
```

Each root is scanned independently.

## Relative Paths

File paths are tracked relative to their cargo root. This relative path is used for:

- Proxy prefix derivation
- Ledger directory structure
- Service identification

Example:
```
Cargo root: ./cargo.d
Absolute path: ./cargo.d/apps/compliance/scf.sqlite.db
Relative path: apps/compliance/scf.sqlite.db
```

## Discovery Adapters

db-yard uses a generic discovery adapter system that can source files from:

- Local filesystem (default)
- Any "path-like" source

The adapter yields `Encounter` objects:

```typescript
interface Encounter {
  relativePath: string;
  absolutePath: string;
  provenance: string;
}
```

## Lazy Content Loading

Discovery reports file locations without loading content until explicitly needed. This makes discovery fast even with large numbers of files.

Classification happens lazily - a file is only opened when db-yard needs to inspect its tables.

## Filtering

Some files are always excluded:

- Hidden files (starting with `.`)
- Files in hidden directories
- Common build artifacts (`node_modules`, `target`, etc.)

## Determinism

Discovery is deterministic:

1. Same cargo root → same files found
2. Same files → same ordering (sorted by path)
3. No randomness or external state

This makes operations reproducible and predictable.

## Watching for Changes

In watch mode, db-yard monitors the cargo directory for:

- New files matching patterns → trigger spawn
- Removed files → trigger kill
- Modified files → depends on service type

```bash
./bin/yard.ts start --watch
```

## Example

```
cargo.d/
├── apps/
│   ├── dashboard.sqlite.db     # Discovered
│   ├── metrics.db              # Discovered
│   └── config.json             # Ignored (no matching pattern)
├── evidence/
│   ├── q1-audit.sqlite.db      # Discovered
│   └── q2-audit.sqlite.db      # Discovered
├── analytics/
│   └── reports.duckdb          # Discovered (but not exposable)
└── .hidden/
    └── secret.db               # Ignored (hidden directory)
```

Discovery yields:
```
apps/dashboard.sqlite.db
apps/metrics.db
evidence/q1-audit.sqlite.db
evidence/q2-audit.sqlite.db
analytics/reports.duckdb
```

## Debugging Discovery

Use the `ls` command to see what db-yard discovers:

```bash
./bin/yard.ts ls
```

This shows all cargo without spawning anything.

---

Next: [Service Classification](./service-classification.md) - How discovered files are categorized.