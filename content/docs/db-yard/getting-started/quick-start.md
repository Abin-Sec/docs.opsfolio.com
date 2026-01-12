---
title: Quick Start Tutorial
description: Get db-yard running in 5 minutes.
---

# Quick Start Tutorial

Get db-yard running in 5 minutes.

## Step 1: Start db-yard

From the db-yard directory:

```bash
./bin/yard.ts start
```

This will:
1. Scan `./cargo.d` for database files
2. Classify each database
3. Spawn exposable services
4. Write state to `./ledger.d`

You should see output like:

```
[yard] Discovering cargo in ./cargo.d
[yard] Found 5 databases
[yard] Spawning: scf-2025.3.sqlite.db on port 3000
[yard] Spawning: evidence-warehouse.db on port 3001
[yard] Materialization complete
```

## Step 2: Check Running Services

List all spawned processes:

```bash
./bin/yard.ts ps
```

Output:

```
PID    PORT   TYPE      DATABASE                          PROXY PREFIX
12345  3000   sqlpage   cargo.d/apps/scf-2025.3.sqlite.db /apps/sqlpage/apps/scf-2025.3
12346  3001   surveilr  cargo.d/evidence/warehouse.db     /apps/surveilr/evidence/warehouse
```

## Step 3: Access Your Services

Open a browser and navigate to your services:

```bash
# Direct access
open http://localhost:3000

# Or via web UI
open http://localhost:8787
```

## Step 4: Start the Web UI

For a graphical administration interface:

```bash
./bin/web-ui/serve.ts
```

Then open: `http://127.0.0.1:8787`

The web UI shows:
- Running services with PID, port, and proxy URLs
- Ledger browser for context files and logs
- Reconciliation table comparing live vs ledger state

## Step 5: Watch Mode (Optional)

For continuous operation with automatic reconciliation:

```bash
./bin/yard.ts start --watch
```

In watch mode:
- New cargo files are automatically spawned
- Removed cargo files trigger process cleanup
- Crashed processes are respawned

Press `Ctrl+C` to stop (all spawned processes will be cleaned up).

## Step 6: Kill Services

Stop all spawned services:

```bash
./bin/yard.ts kill
```

With cleanup:

```bash
./bin/yard.ts kill --clean
```

## What Just Happened?

1. **Discovery**: db-yard scanned `./cargo.d` using glob patterns (`**/*.db`, `**/*.sqlite`, etc.)

2. **Classification**: Each database was inspected:
   - Has `sqlpage_files` table? → SQLPage app
   - Has `uniform_resource` table? → surveilr RSSD
   - Neither? → Plain SQLite (not exposed)

3. **Port Allocation**: Ports assigned starting at 3000, skipping in-use ports

4. **Spawning**: Services started with appropriate executables (`sqlpage` or `surveilr web-ui`)

5. **Ledger Writing**: State written to `./ledger.d/`:
   - `<name>.context.json` - Service metadata
   - `<name>.stdout.log` - Process output
   - `<name>.stderr.log` - Process errors

## Next Steps

- [Your First Cargo](./first-cargo.md) - Create your own database cargo
- [Core Concepts](../core-concepts/README.md) - Understand how db-yard works
- [CLI Reference](../reference/cli.md) - Complete command documentation

---

Having issues? See [Troubleshooting](../support/troubleshooting.md).