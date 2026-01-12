---
title: Frequently Asked Questions
description: Common questions about db-yard.
---

# Frequently Asked Questions

Common questions about db-yard.

## General

### What is db-yard?

db-yard is a file-driven process orchestration platform for managing embedded database cargo (SQLite, DuckDB) as web services. It treats your filesystem as the control plane, automatically discovering databases, spawning services, and maintaining operational state on disk.

### Why use db-yard?

- **No daemon required** - Filesystem is the API
- **Automatic discovery** - Drop files in, services appear
- **Inspectable state** - All operational data in readable files
- **Simple operations** - One command to start everything

### What databases are supported?

Currently:
- **SQLite** - Full support (SQLPage apps, surveilr RSSDs)
- **DuckDB** - Discovery and composite support
- **Excel** - Discovery only

### What's the difference between db-yard and Docker?

| Feature | db-yard | Docker |
|---------|---------|--------|
| Focus | Embedded databases | General containers |
| State | Filesystem | Container registry |
| Orchestration | File-based | YAML/CLI |
| Overhead | Minimal | Container runtime |

db-yard is designed specifically for embedded database workflows.

## Installation

### What are the requirements?

- Deno 2.x
- Optional: SQLPage binary (for SQLPage apps)
- Optional: surveilr binary (for RSSDs)

### How do I install db-yard?

```bash
git clone https://github.com/netspective-labs/db-yard.git
cd db-yard
./bin/yard.ts --help
```

### Does db-yard work on Windows?

db-yard is primarily tested on Linux and macOS. Windows support via WSL should work.

## Usage

### How do I start services?

```bash
./bin/yard.ts start
```

### How do I add a new database?

Drop it in the cargo directory:

```bash
cp my-app.sqlite.db cargo.d/
```

If using watch mode, it's automatically spawned.

### How do I remove a service?

Remove the database from cargo directory:

```bash
rm cargo.d/my-app.sqlite.db
```

If using watch mode, the process is automatically killed.

### What's the difference between start and start --watch?

| Mode | Behavior |
|------|----------|
| `start` | One-shot: spawn all, exit |
| `start --watch` | Continuous: spawn, monitor, reconcile |

### How do I access my services?

Directly by port:
```
http://localhost:3000
```

Via proxy:
```
http://localhost:8787/apps/sqlpage/my-app/
```

## Classification

### How does db-yard detect SQLPage apps?

It looks for a `sqlpage_files` table in the SQLite database.

### How does db-yard detect surveilr RSSDs?

It looks for a `uniform_resource` table in the SQLite database.

### My database isn't being classified correctly

Check that:
1. The required table exists
2. The table isn't empty
3. The file extension is correct

## Ports

### How are ports allocated?

Starting from the base port (default 3000), incrementing for each service, skipping ports in use.

### Can I specify a port for a service?

Currently, ports are allocated automatically. Use the proxy for stable URLs.

### What if I run out of ports?

Increase the port range or reduce the number of services.

## Ledger

### What is the ledger?

The ledger is db-yard's operational state store - JSON files and logs that record everything about spawned services.

### Where is the ledger?

By default: `./ledger.d/`

### Can I delete the ledger?

Yes, but running services won't be affected. The ledger is for inspection and tracking.

### How do I clean up old sessions?

```bash
./bin/yard.ts kill --clean
```

Or manually delete session directories from `ledger.d/`.

## Composite Connections

### What are composite connections?

A way to query multiple SQLite databases through a single connection using ATTACH DATABASE.

### How do I create a composite?

```bash
./bin/yard.ts cc --volume-root /path/to/dbs --scope admin
```

### Can I use DuckDB for composites?

Yes:
```bash
./bin/yard.ts cc --dialect DuckDB --duckdb-sqlite-ext
```

## Proxy

### What is the built-in proxy?

A transparent HTTP proxy that routes requests to spawned services based on URL paths.

### Should I use the built-in proxy in production?

No. Use NGINX/Traefik with generated configuration:

```bash
./bin/yard.ts proxy-conf --type nginx
```

### How do I debug proxy routing?

```bash
curl "http://localhost:8787/.db-yard/api/proxy-debug.json?path=/apps/sqlpage/my-app/"
```

## Watch Mode

### Is watch mode safe for production?

Watch mode is designed for development. For production:
- Use systemd to manage db-yard
- Use external proxies (NGINX/Traefik)
- Implement proper monitoring

### What happens when I Ctrl+C in watch mode?

All spawned processes are killed and db-yard exits cleanly.

### Can I run multiple db-yard instances?

Not recommended. Use a single instance with multiple cargo directories if needed.

## Troubleshooting

### Why isn't my service starting?

1. Check if the executable (sqlpage/surveilr) is installed
2. Check if the port is available
3. Check stderr logs in the ledger

### Why is my service crashing?

Check `ledger.d/*/service.db.stderr.log` for error messages.

### How do I report a bug?

Open an issue at [github.com/netspective-labs/db-yard/issues](https://github.com/netspective-labs/db-yard/issues) with:
- Steps to reproduce
- Expected vs actual behavior
- Logs and error messages

---

See also: [Troubleshooting](./troubleshooting.md), [Resources](./resources.md)