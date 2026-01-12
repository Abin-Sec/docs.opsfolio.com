---
title: Your First Cargo
description: This guide walks through deploying your own database as cargo in db-yard.
---

# Your First Cargo

This guide walks through deploying your own database as cargo in db-yard.

## Understanding Cargo

In db-yard terminology:
- **Cargo** = Database files eligible for deployment
- **Cargo Directory** = Root folder db-yard scans for databases
- **Exposable Cargo** = Databases that can be spawned as web services

## Step 1: Prepare Your Database

db-yard can expose two types of databases:

### SQLPage Applications

A SQLPage app is a SQLite database with a `sqlpage_files` table containing SQL files:

```sql
-- Create the sqlpage_files table
CREATE TABLE sqlpage_files (
    path TEXT PRIMARY KEY,
    contents TEXT,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Add an index page
INSERT INTO sqlpage_files (path, contents) VALUES
('index.sql', 'SELECT ''text'' AS component, ''Hello from SQLPage!'' AS contents;');
```

### Surveilr RSSDs

A surveilr RSSD is a SQLite database with a `uniform_resource` table:

```sql
-- RSSDs are typically created by surveilr CLI
-- surveilr ingest files ...
```

## Step 2: Choose Your Cargo Location

Create a subdirectory structure that reflects your organization:

```bash
mkdir -p cargo.d/my-apps
mkdir -p cargo.d/my-evidence
```

The directory structure becomes the proxy prefix:
- `cargo.d/my-apps/dashboard.db` → `/apps/sqlpage/my-apps/dashboard`
- `cargo.d/my-evidence/audit.db` → `/apps/surveilr/my-evidence/audit`

## Step 3: Deploy Your Database

Copy your database into the cargo directory:

```bash
cp my-sqlpage-app.sqlite.db cargo.d/my-apps/
```

Or create a SQLPage app from scratch:

```bash
sqlite3 cargo.d/my-apps/hello.sqlite.db <<'EOF'
CREATE TABLE sqlpage_files (
    path TEXT PRIMARY KEY,
    contents TEXT,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO sqlpage_files (path, contents) VALUES
('index.sql', 'SELECT ''text'' AS component, ''Hello, World!'' AS contents;');
EOF
```

## Step 4: Start db-yard

```bash
./bin/yard.ts start --verbose essential
```

You should see:

```
[yard] Discovered: cargo.d/my-apps/hello.sqlite.db
[yard] Classified as: sqlpage
[yard] Assigned port: 3000
[yard] Proxy prefix: /apps/sqlpage/my-apps/hello
[yard] Spawning with: sqlpage
[yard] PID: 12345
```

## Step 5: Verify Deployment

Check the process list:

```bash
./bin/yard.ts ps
```

Access your application:

```bash
curl http://localhost:3000
```

Or via the proxy:

```bash
# Start web UI first
./bin/web-ui/serve.ts &

# Access via proxy
curl http://localhost:8787/apps/sqlpage/my-apps/hello/
```

## Step 6: Inspect the Ledger

The ledger contains all operational state:

```bash
ls ledger.d/
```

For each spawned service:

```
ledger.d/2026-01-12-10-30-00/
  my-apps/
    hello.sqlite.db.context.json  # Service metadata
    hello.sqlite.db.stdout.log    # Process output
    hello.sqlite.db.stderr.log    # Process errors
```

View the context:

```bash
cat ledger.d/*/my-apps/hello.sqlite.db.context.json
```

```json
{
  "serviceId": "hello.sqlite.db",
  "sessionId": "2026-01-12-10-30-00",
  "provenance": "/path/to/cargo.d/my-apps/hello.sqlite.db",
  "port": 3000,
  "proxyPrefix": "/apps/sqlpage/my-apps/hello",
  "pid": 12345,
  "kind": "sqlpage"
}
```

## Step 7: Update Your Cargo

With watch mode, updates are automatic:

```bash
# Start in watch mode
./bin/yard.ts start --watch
```

Then modify your database:

```bash
sqlite3 cargo.d/my-apps/hello.sqlite.db <<'EOF'
UPDATE sqlpage_files
SET contents = 'SELECT ''text'' AS component, ''Updated content!'' AS contents;'
WHERE path = 'index.sql';
EOF
```

The changes are reflected immediately (SQLPage reads from the database on each request).

## Best Practices

### Naming Conventions

Use descriptive names with the appropriate extension:

- `my-app.sqlite.db` - Preferred for SQLite
- `my-app.db` - Also supported
- `my-app.sqlite` - Also supported

### Directory Organization

Organize by purpose:

```
cargo.d/
  compliance/
    scf-2025.sqlite.db
    hipaa-audit.sqlite.db
  dashboards/
    metrics.sqlite.db
    status.sqlite.db
  evidence/
    q1-2026-audit.sqlite.db
```

### Database Extensions

Compound extensions are normalized:
- `my-app.sqlite.db` → proxy prefix uses `my-app` (not `my-app.sqlite`)

## Next Steps

- [Core Concepts](../core-concepts/README.md) - Understand discovery, classification, and ledger
- [How-To: SQLPage Apps](../how-to-guides/sqlpage-apps.md) - Build complete SQLPage applications
- [Watch Mode](../how-to-guides/watch-mode.md) - Continuous operation guide

---

Having issues? See [Troubleshooting](../support/troubleshooting.md).