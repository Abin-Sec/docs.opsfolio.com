---
title: Watch Mode
description: Running db-yard with continuous monitoring and reconciliation.
---

# Watch Mode

Running db-yard with continuous monitoring and reconciliation.

## Overview

Watch mode keeps db-yard running continuously, monitoring the cargo directory and automatically reconciling changes. New databases are spawned, removed databases are killed, and crashed processes are respawned.

## Starting Watch Mode

```bash
./bin/yard.ts start --watch
```

Output:
```
[yard] Starting in watch mode
[yard] Watching: ./cargo.d
[yard] Initial materialization...
[yard] Spawned: dashboard.sqlite.db on port 3000
[yard] Spawned: metrics.sqlite.db on port 3001
[yard] Ready. Press Ctrl+C to stop.
```

## How It Works

### Initial Materialization

When watch mode starts:
1. Scans cargo directory
2. Classifies all databases
3. Spawns exposable services
4. Writes to ledger

### Continuous Monitoring

After initial setup, db-yard watches for:

| Event | Action |
|-------|--------|
| New file appears | Spawn service |
| File removed | Kill service |
| Process crashes | Respawn service |
| File modified | No action (service reads from DB) |

### Cleanup on Exit

When you press `Ctrl+C`:
1. SIGINT received
2. All spawned processes killed
3. Clean exit

## Adding New Cargo

With watch mode running:

```bash
# Add a new database
cp my-new-app.sqlite.db cargo.d/apps/
```

db-yard automatically:
```
[yard] Detected: cargo.d/apps/my-new-app.sqlite.db
[yard] Classified as: sqlpage
[yard] Spawning on port 3002
```

## Removing Cargo

```bash
# Remove a database
rm cargo.d/apps/old-app.sqlite.db
```

db-yard automatically:
```
[yard] Removed: cargo.d/apps/old-app.sqlite.db
[yard] Killing process: PID 12345
```

## Process Recovery

If a service crashes:

```
[yard] Process died: dashboard.sqlite.db (PID 12345)
[yard] Respawning: dashboard.sqlite.db on port 3000
```

The same port is reused to maintain stable URLs.

## Stable Session Directory

Watch mode uses a stable session:

```
ledger.d/watch/
├── apps/
│   ├── dashboard.sqlite.db.context.json
│   └── dashboard.sqlite.db.stdout.log
...
```

Unlike materialize mode (timestamped sessions), watch mode continuously updates the same directory.

## Combining with Web UI

Run both in parallel:

```bash
# Terminal 1: Watch mode
./bin/yard.ts start --watch

# Terminal 2: Web UI
./bin/web-ui/serve.ts
```

The web UI shows live status and allows browsing the ledger.

## Configuration

### Custom Directories

```bash
./bin/yard.ts start --watch \
  --cargo-home /var/db-yard/cargo \
  --ledger-home /var/db-yard/ledger
```

### Verbose Output

```bash
./bin/yard.ts start --watch --verbose comprehensive
```

Shows detailed information about:
- Each file discovered
- Classification decisions
- Port allocation
- Process lifecycle events

## Use Cases

### Development Environment

Keep services running during development:

```bash
# Start watch mode
./bin/yard.ts start --watch

# In another terminal, work on your database
sqlite3 cargo.d/my-app.sqlite.db
# Changes are immediately available

# Add new apps as needed
cp new-app.sqlite.db cargo.d/
```

### Demo Environment

Quickly spawn/remove services:

```bash
./bin/yard.ts start --watch

# Add demo databases
cp demo-*.sqlite.db cargo.d/

# After demo, clean up
rm cargo.d/demo-*.sqlite.db
```

### Testing

Test service deployment:

```bash
./bin/yard.ts start --watch

# Deploy test database
cp test-app.sqlite.db cargo.d/

# Run tests
./run-tests.sh http://localhost:3000

# Remove test database
rm cargo.d/test-app.sqlite.db
```

## Monitoring

### Check Status

```bash
./bin/yard.ts ps
```

### Watch Logs

```bash
# Follow a specific service log
tail -f ledger.d/watch/apps/dashboard.sqlite.db.stdout.log

# Watch all logs
tail -f ledger.d/watch/**/*.log
```

### Reconciliation Status

```bash
./bin/yard.ts ps --reconcile
```

Shows any drift between expected and actual state.

## Graceful Shutdown

Press `Ctrl+C` (SIGINT) to stop:

```
^C
[yard] Received SIGINT, cleaning up...
[yard] Killing: dashboard.sqlite.db (PID 12345)
[yard] Killing: metrics.sqlite.db (PID 12346)
[yard] Cleanup complete
```

All child processes are terminated cleanly.

## Troubleshooting

### Services Not Spawning

Check classification:
```bash
./bin/yard.ts ls
```

Ensure databases have required tables:
- `sqlpage_files` for SQLPage apps
- `uniform_resource` for surveilr RSSDs

### Port Conflicts

Check what's using ports:
```bash
lsof -i :3000
```

Start with a different base port:
```bash
./bin/yard.ts start --watch --port-base 8000
```

### Watch Not Detecting Changes

Ensure the file is in the cargo directory:
```bash
ls -la cargo.d/
```

Check file permissions.

### High CPU Usage

May occur with many files. Consider:
- Reducing cargo directory scope
- Using subdirectories
- Filtering file types

## Best Practices

1. **Use for development** - Not recommended for production
2. **Monitor logs** - Watch for errors and crashes
3. **Clean exit** - Always use Ctrl+C, not kill -9
4. **Separate environments** - Use different cargo directories
5. **Regular cleanup** - Remove unused databases

---

See also: [CLI Reference](../reference/cli.md), [Production Deployment](./production-deployment.md)