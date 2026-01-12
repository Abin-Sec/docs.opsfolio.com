---
title: Surveilr RSSDs
description: Working with Resource Surveillance State Databases.
---

# Surveilr RSSDs

Working with Resource Surveillance State Databases.

## Overview

Surveilr is a tool for ingesting files and resources into SQLite databases called RSSDs (Resource Surveillance State Databases). db-yard automatically discovers and serves these databases with surveilr's web UI.

## Prerequisites

- Surveilr binary installed and in PATH
- db-yard installed
- Files or resources to ingest

## Creating an RSSD

### Step 1: Ingest Files

```bash
# Create RSSD by ingesting files
surveilr ingest files \
  --rssd cargo.d/evidence/project-docs.sqlite.db \
  /path/to/documents/
```

### Step 2: Ingest from Multiple Sources

```bash
# Ingest from multiple directories
surveilr ingest files \
  --rssd cargo.d/evidence/audit.sqlite.db \
  /path/to/policies/ \
  /path/to/procedures/ \
  /path/to/evidence/
```

### Step 3: Verify RSSD Structure

```bash
sqlite3 cargo.d/evidence/audit.sqlite.db ".tables"
```

You should see tables including:
- `uniform_resource` (main resource table)
- `uniform_resource_transform`
- `device`
- `behavior`

## Deploying the RSSD

### Start db-yard

```bash
./bin/yard.ts start
```

Output:
```
[yard] Discovered: cargo.d/evidence/audit.sqlite.db
[yard] Classified as: surveilr
[yard] Spawning on port 3000
```

### Access the Web UI

Direct access:
```bash
open http://localhost:3000
```

Via proxy:
```bash
./bin/web-ui/serve.ts
open http://localhost:8787/apps/surveilr/evidence/audit/
```

## RSSD Features

### Browse Resources

The surveilr web UI provides:
- File browser with directory structure
- Content preview for text files
- Metadata display (size, type, hash)
- Full-text search

### Query Resources

SQL queries against the RSSD:

```sql
-- List all resources
SELECT uri, content_digest, size_bytes
FROM uniform_resource
ORDER BY last_modified DESC;

-- Find files by type
SELECT uri
FROM uniform_resource
WHERE nature = 'text/markdown';

-- Search content
SELECT uri, content
FROM uniform_resource
WHERE content LIKE '%compliance%';
```

## Updating RSSDs

### Re-ingest Files

```bash
# Run ingestion again to update
surveilr ingest files \
  --rssd cargo.d/evidence/audit.sqlite.db \
  /path/to/documents/
```

Surveilr handles:
- New files → added
- Modified files → updated
- Removed files → optionally marked

### Watch Mode

In db-yard watch mode, RSSD changes trigger service restart:

```bash
./bin/yard.ts start --watch
```

## Multiple RSSDs

Organize by purpose:

```
cargo.d/
├── evidence/
│   ├── q1-audit.sqlite.db
│   ├── q2-audit.sqlite.db
│   └── annual-review.sqlite.db
├── compliance/
│   ├── policies.sqlite.db
│   └── procedures.sqlite.db
└── projects/
    ├── project-a.sqlite.db
    └── project-b.sqlite.db
```

Each RSSD gets its own:
- Port (3000, 3001, 3002, ...)
- Proxy prefix (`/apps/surveilr/evidence/q1-audit`, etc.)

## Combining with Composites

Query across multiple RSSDs:

```bash
# Generate composite SQL
./bin/yard.ts cc \
  --volume-root /var/db-yard \
  --scope cross-tenant \
  > composite.sql
```

Use the composite:

```sql
-- After running composite.sql
SELECT tenant, uri, size_bytes
FROM all_resources
WHERE size_bytes > 1000000
ORDER BY size_bytes DESC;
```

## Common Workflows

### Evidence Collection

```bash
# 1. Collect evidence
surveilr ingest files \
  --rssd cargo.d/evidence/$(date +%Y-%m).sqlite.db \
  /path/to/monthly-evidence/

# 2. Start db-yard
./bin/yard.ts start

# 3. Review in browser
open http://localhost:3000
```

### Compliance Audit

```bash
# 1. Ingest policies and procedures
surveilr ingest files \
  --rssd cargo.d/compliance/controls.sqlite.db \
  /path/to/policies/ \
  /path/to/procedures/

# 2. Ingest evidence
surveilr ingest files \
  --rssd cargo.d/compliance/evidence.sqlite.db \
  /path/to/audit-evidence/

# 3. Deploy both
./bin/yard.ts start
```

### Project Documentation

```bash
# Ingest project files
surveilr ingest files \
  --rssd cargo.d/projects/my-project.sqlite.db \
  ./docs/ \
  ./src/ \
  ./tests/

# Include specific patterns only
surveilr ingest files \
  --rssd cargo.d/projects/my-project.sqlite.db \
  --include "*.md" \
  --include "*.ts" \
  ./
```

## Advanced Usage

### Custom Ingestion

Surveilr supports various ingestion modes:

```bash
# Ingest from URLs
surveilr ingest http \
  --rssd cargo.d/web/external.sqlite.db \
  https://example.com/api/data

# Ingest with transformations
surveilr ingest files \
  --rssd cargo.d/data/processed.sqlite.db \
  --transform markdown \
  /path/to/documents/
```

### RSSD Analysis

Query the RSSD directly:

```bash
# File type distribution
sqlite3 cargo.d/evidence/audit.sqlite.db <<'EOF'
SELECT nature, COUNT(*) as count
FROM uniform_resource
GROUP BY nature
ORDER BY count DESC;
EOF

# Total size
sqlite3 cargo.d/evidence/audit.sqlite.db <<'EOF'
SELECT SUM(size_bytes) / 1024 / 1024 as total_mb
FROM uniform_resource;
EOF
```

## Debugging

### Check RSSD Contents

```bash
# Count resources
sqlite3 cargo.d/evidence/audit.sqlite.db \
  "SELECT COUNT(*) FROM uniform_resource"

# List recent resources
sqlite3 cargo.d/evidence/audit.sqlite.db \
  "SELECT uri FROM uniform_resource ORDER BY last_modified DESC LIMIT 10"
```

### Check Service Logs

```bash
cat ledger.d/*/evidence/audit.sqlite.db.stderr.log
```

### Verify Classification

```bash
./bin/yard.ts ls
# Should show "surveilr" classification
```

## Best Practices

1. **Organize by purpose** - Separate RSSDs for different audit periods or projects
2. **Use meaningful names** - Names become URL paths
3. **Regular ingestion** - Schedule re-ingestion for updated content
4. **Archive old RSSDs** - Move completed audits out of cargo.d
5. **Version control metadata** - Track RSSD creation scripts

---

See also: [Your First Cargo](../getting-started/first-cargo.md), [Surveilr Documentation](https://surveilr.com/)