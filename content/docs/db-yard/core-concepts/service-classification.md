---
title: Service Classification
description: How db-yard categorizes discovered databases.
---

# Service Classification

How db-yard categorizes discovered databases.

## Overview

After discovery, each database file is classified to determine if and how it can be exposed as a web service. Classification is cheap, deterministic, and based solely on table inspection.

## Classification Types

| Classification | Detection Method | Service Executable |
|---------------|------------------|-------------------|
| SQLPage App | Has `sqlpage_files` table | `sqlpage` |
| Surveilr RSSD | Has `uniform_resource` table | `surveilr web-ui` |
| Plain SQLite | Neither table present | Not exposed |
| DuckDB | `.duckdb` extension | Not exposed (discovery only) |
| Excel | `.xlsx` extension | Not exposed (discovery only) |

## Classification Logic

```
Is it SQLite-like?
  ├─ Yes → Check tables
  │     ├─ Has uniform_resource? → surveilr RSSD
  │     ├─ Has sqlpage_files? → SQLPage app
  │     └─ Neither? → Plain SQLite (ignored)
  └─ No → Other format (not exposable)
```

## SQLPage Applications

A SQLPage app is a SQLite database that contains SQL files in a special table:

```sql
-- Detection query
SELECT 1 FROM sqlite_master
WHERE type='table' AND name='sqlpage_files';
```

The `sqlpage_files` table schema:

```sql
CREATE TABLE sqlpage_files (
    path TEXT PRIMARY KEY,
    contents TEXT,
    last_modified TIMESTAMP
);
```

When exposed, SQLPage reads SQL from this table and renders web pages.

## Surveilr RSSDs

A surveilr RSSD (Resource Surveillance State Database) contains ingested resources:

```sql
-- Detection query
SELECT 1 FROM sqlite_master
WHERE type='table' AND name='uniform_resource';
```

The `uniform_resource` table is created by surveilr during ingestion:

```bash
surveilr ingest files --rssd my-evidence.db /path/to/files
```

When exposed, `surveilr web-ui` provides a browsable interface.

## Exposable vs Non-Exposable

Only classified databases become services:

| Type | Exposable | Reason |
|------|-----------|--------|
| SQLPage app | Yes | Has web interface |
| Surveilr RSSD | Yes | Has web interface |
| Plain SQLite | No | No web interface |
| DuckDB | No | Analytics engine, not web server |
| Excel | No | Tabular data, not database |

Non-exposable files are still tracked for:
- Composite connections
- Analytics queries via DuckDB
- Future extensibility

## Classification Priority

If a database has both `sqlpage_files` and `uniform_resource`:

1. `uniform_resource` wins → classified as surveilr RSSD

This is because surveilr RSSDs may include SQLPage content, but the primary interface should be surveilr's web UI.

## No Heuristics

Classification is explicit, not heuristic:

- No file name guessing
- No content inspection beyond tables
- No background indexing
- No machine learning

This keeps behavior predictable and auditable.

## Debugging Classification

Use verbose mode to see classification decisions:

```bash
./bin/yard.ts start --verbose comprehensive
```

Output:

```
[yard] Discovered: cargo.d/apps/dashboard.sqlite.db
[yard] Checking for sqlpage_files table... found
[yard] Classified as: sqlpage

[yard] Discovered: cargo.d/evidence/audit.db
[yard] Checking for uniform_resource table... found
[yard] Classified as: surveilr

[yard] Discovered: cargo.d/data/raw.db
[yard] Checking for sqlpage_files table... not found
[yard] Checking for uniform_resource table... not found
[yard] Classified as: plain-sqlite (not exposable)
```

## Extending Classification

The classification system is designed for extension. Future versions may support:

- Custom classification rules
- Additional service types
- Plugin-based classifiers

---

Next: [Ledger System](./ledger-system.md) - The operational state layer.