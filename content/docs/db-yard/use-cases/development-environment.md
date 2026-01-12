---
title: Development Environment
description: Using db-yard for local development workflows.
---

# Development Environment

Using db-yard for local development workflows.

## Overview

db-yard's watch mode and automatic discovery make it ideal for local development. Add, modify, and remove databases as you work, and db-yard keeps everything running.

## Development Setup

### Initial Configuration

```bash
# Clone db-yard
git clone https://github.com/netspective-labs/db-yard.git
cd db-yard

# Create development cargo directory
mkdir -p cargo.d/dev

# Start in watch mode
./bin/yard.ts start --watch --verbose essential
```

### Start Web UI (separate terminal)

```bash
./bin/web-ui/serve.ts
```

Now you have:
- Watch mode managing services on ports 3000+
- Web UI at http://localhost:8787

## Development Workflow

### Adding a New App

```bash
# Create a new SQLPage app
sqlite3 cargo.d/dev/my-feature.sqlite.db <<'EOF'
CREATE TABLE sqlpage_files (
    path TEXT PRIMARY KEY,
    contents TEXT,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO sqlpage_files (path, contents) VALUES
('index.sql', 'SELECT ''text'' AS component, ''Hello!'' AS contents;');
EOF
```

Watch mode output:
```
[yard] Detected: cargo.d/dev/my-feature.sqlite.db
[yard] Classified as: sqlpage
[yard] Spawning on port 3000
```

### Modifying Pages

```bash
# Update the page
sqlite3 cargo.d/dev/my-feature.sqlite.db <<'EOF'
UPDATE sqlpage_files
SET contents = 'SELECT ''text'' AS component, ''Updated!'' AS contents;'
WHERE path = 'index.sql';
EOF

# Refresh browser - changes are immediate
```

### Adding Data Tables

```bash
# Add application data
sqlite3 cargo.d/dev/my-feature.sqlite.db <<'EOF'
CREATE TABLE items (
    id INTEGER PRIMARY KEY,
    name TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO items (name) VALUES ('Item 1'), ('Item 2'), ('Item 3');

UPDATE sqlpage_files
SET contents = '
SELECT ''table'' AS component, ''Items'' AS title;
SELECT id, name, created_at FROM items;
'
WHERE path = 'index.sql';
EOF
```

### Removing an App

```bash
# Remove the database
rm cargo.d/dev/my-feature.sqlite.db
```

Watch mode output:
```
[yard] Removed: cargo.d/dev/my-feature.sqlite.db
[yard] Killing process: PID 12345
```

## Multi-App Development

Organize multiple apps:

```
cargo.d/
├── dev/
│   ├── feature-a/
│   │   └── app.sqlite.db
│   ├── feature-b/
│   │   └── app.sqlite.db
│   └── shared/
│       └── common-data.sqlite.db
```

Each gets its own port and proxy:
- Feature A: `/apps/sqlpage/dev/feature-a/app`
- Feature B: `/apps/sqlpage/dev/feature-b/app`

## Development Tools

### Quick Database Creation Script

```bash
#!/bin/bash
# scripts/new-app.sh

NAME=$1
DB="cargo.d/dev/$NAME.sqlite.db"

sqlite3 "$DB" <<'EOF'
CREATE TABLE sqlpage_files (
    path TEXT PRIMARY KEY,
    contents TEXT,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO sqlpage_files (path, contents) VALUES
('index.sql', 'SELECT ''hero'' AS component, ''New App'' AS title;');
EOF

echo "Created $DB"
echo "Access at: http://localhost:8787/apps/sqlpage/dev/$NAME/"
```

Usage:
```bash
./scripts/new-app.sh my-new-feature
```

### Database Export/Import

```bash
# Export SQL
sqlite3 cargo.d/dev/app.sqlite.db ".dump" > app.sql

# Import to new database
sqlite3 cargo.d/dev/app-copy.sqlite.db < app.sql
```

### Quick Testing

```bash
# Test a specific page
sqlite3 cargo.d/dev/app.sqlite.db "SELECT contents FROM sqlpage_files WHERE path='index.sql'"

# Check table structure
sqlite3 cargo.d/dev/app.sqlite.db ".schema"
```

## Integration with IDE

### VS Code Tasks

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Start db-yard",
      "type": "shell",
      "command": "./bin/yard.ts start --watch",
      "isBackground": true,
      "problemMatcher": []
    },
    {
      "label": "Start Web UI",
      "type": "shell",
      "command": "./bin/web-ui/serve.ts",
      "isBackground": true,
      "problemMatcher": []
    }
  ]
}
```

### Launch Configuration

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug db-yard",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "deno",
      "runtimeArgs": ["run", "-A", "./bin/yard.ts", "start", "--watch"],
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

## Testing Workflow

### Unit Test Databases

```bash
# Create test fixtures
mkdir -p cargo.d/test

sqlite3 cargo.d/test/fixtures.sqlite.db <<'EOF'
CREATE TABLE sqlpage_files (...);
-- Test data
EOF
```

### Integration Tests

```bash
#!/bin/bash
# Run integration tests

# Start services
./bin/yard.ts start &
sleep 2

# Run tests
curl -sf http://localhost:3000/ > /dev/null && echo "PASS: App responds"

# Cleanup
./bin/yard.ts kill
```

## Debugging

### View Service Logs

```bash
# Follow logs in real-time
tail -f ledger.d/watch/dev/app.sqlite.db.stdout.log
```

### Check Service Status

```bash
./bin/yard.ts ps --extended
```

### Proxy Debugging

```bash
# Debug proxy routing
curl "http://localhost:8787/.db-yard/api/proxy-debug.json?path=/apps/sqlpage/dev/app/"
```

## Best Practices

1. **Use watch mode** - Automatic service management
2. **Organize by feature** - Separate directories for each feature
3. **Version control SQL** - Export and track database definitions
4. **Clean up regularly** - Remove unused test databases
5. **Use the web UI** - Visual overview of all services

---

See also: [Watch Mode](../how-to-guides/watch-mode.md), [Quick Start](../getting-started/quick-start.md)