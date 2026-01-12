---
title: Compliance Dashboards
description: Building compliance visualization dashboards with db-yard.
---

# Compliance Dashboards

Building compliance visualization dashboards with db-yard.

## Overview

db-yard excels at serving compliance data through SQLPage dashboards and surveilr evidence warehouses. This use case shows how to build a complete compliance monitoring system.

## Architecture

```
+─────────────────────────────────────────────────────+
|                 Compliance System                    |
+──────────────────+──────────────────────────────────+
|   Evidence       |         Dashboards               |
|   Collection     |                                  |
|   (surveilr)     |   +──────────────────────────+  |
|                  |   | SCF Controls Dashboard   |  |
| +──────────────+ |   | (SQLPage)                |  |
| | policies.db  | |   +──────────────────────────+  |
| +──────────────+ |   +──────────────────────────+  |
| +──────────────+ |   | Audit Status Dashboard   |  |
| | evidence.db  | |   | (SQLPage)                |  |
| +──────────────+ |   +──────────────────────────+  |
| +──────────────+ |   +──────────────────────────+  |
| | audit.db     | |   | Evidence Browser         |  |
| +──────────────+ |   | (surveilr web-ui)        |  |
+──────────────────+──────────────────────────────────+
```

## Step 1: Collect Evidence

```bash
# Ingest policy documents
surveilr ingest files \
  --rssd cargo.d/compliance/policies.sqlite.db \
  /path/to/policies/

# Ingest audit evidence
surveilr ingest files \
  --rssd cargo.d/compliance/evidence.sqlite.db \
  /path/to/evidence/
```

## Step 2: Create Controls Database

```sql
-- Create SQLPage dashboard for controls
sqlite3 cargo.d/compliance/scf-dashboard.sqlite.db <<'EOF'

CREATE TABLE sqlpage_files (
    path TEXT PRIMARY KEY,
    contents TEXT,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Controls data
CREATE TABLE controls (
    control_id TEXT PRIMARY KEY,
    category TEXT,
    title TEXT,
    description TEXT,
    status TEXT DEFAULT 'not-started',
    evidence_count INTEGER DEFAULT 0,
    last_reviewed DATE
);

-- Insert SCF controls
INSERT INTO controls (control_id, category, title, status) VALUES
('SCF-1.1', 'Access Control', 'Account Management', 'implemented'),
('SCF-1.2', 'Access Control', 'Access Enforcement', 'in-progress'),
('SCF-2.1', 'Audit', 'Audit Events', 'implemented'),
('SCF-2.2', 'Audit', 'Audit Review', 'not-started');

-- Dashboard pages
INSERT INTO sqlpage_files (path, contents) VALUES
('index.sql', '
SELECT ''shell'' AS component,
       ''Compliance Dashboard'' AS title;

SELECT ''card'' AS component, 2 AS columns;

SELECT ''Controls Overview'' AS title,
       ''View all security controls'' AS description,
       ''controls.sql'' AS link,
       ''shield'' AS icon;

SELECT ''Audit Status'' AS title,
       ''Current audit progress'' AS description,
       ''audit.sql'' AS link,
       ''clipboard'' AS icon;

SELECT ''chart'' AS component,
       ''pie'' AS type,
       ''Control Status'' AS title;
SELECT status, COUNT(*) as count
FROM controls
GROUP BY status;
'),

('controls.sql', '
SELECT ''shell'' AS component,
       ''Security Controls'' AS title;

SELECT ''table'' AS component,
       ''Controls'' AS title,
       TRUE AS sort,
       TRUE AS search;
SELECT control_id, category, title, status, evidence_count, last_reviewed
FROM controls
ORDER BY control_id;
'),

('audit.sql', '
SELECT ''shell'' AS component,
       ''Audit Status'' AS title;

SELECT ''metric'' AS component;
SELECT ''Implemented'' AS title,
       (SELECT COUNT(*) FROM controls WHERE status = ''implemented'') AS value,
       ''green'' AS color;
SELECT ''In Progress'' AS title,
       (SELECT COUNT(*) FROM controls WHERE status = ''in-progress'') AS value,
       ''yellow'' AS color;
SELECT ''Not Started'' AS title,
       (SELECT COUNT(*) FROM controls WHERE status = ''not-started'') AS value,
       ''red'' AS color;
');

EOF
```

## Step 3: Deploy with db-yard

```bash
# Start all compliance services
./bin/yard.ts start

# Start web UI
./bin/web-ui/serve.ts
```

Access points:
- SCF Dashboard: `http://localhost:8787/apps/sqlpage/compliance/scf-dashboard/`
- Policy Browser: `http://localhost:8787/apps/surveilr/compliance/policies/`
- Evidence Browser: `http://localhost:8787/apps/surveilr/compliance/evidence/`

## Step 4: Create Composite Views

Query across all compliance databases:

```bash
./bin/yard.ts cc \
  --volume-root $(pwd)/cargo.d \
  --scope admin \
  > compliance-composite.sql
```

```sql
-- Add to composite.sql
CREATE VIEW compliance_overview AS
SELECT
    c.control_id,
    c.title,
    c.status,
    (SELECT COUNT(*) FROM policies.uniform_resource
     WHERE uri LIKE '%' || c.control_id || '%') as policy_count,
    (SELECT COUNT(*) FROM evidence.uniform_resource
     WHERE uri LIKE '%' || c.control_id || '%') as evidence_count
FROM controls.controls c;
```

## Step 5: Automate Updates

```bash
#!/bin/bash
# update-compliance.sh

# Re-ingest evidence
surveilr ingest files \
  --rssd cargo.d/compliance/evidence.sqlite.db \
  /path/to/new-evidence/

# Update control status
sqlite3 cargo.d/compliance/scf-dashboard.sqlite.db <<'EOF'
UPDATE controls
SET evidence_count = (
    SELECT COUNT(*)
    FROM evidence.uniform_resource
    WHERE uri LIKE '%' || control_id || '%'
),
last_reviewed = date('now')
WHERE status != 'not-started';
EOF
```

## Features

### Control Status Dashboard

- Pie chart of control status
- Sortable/searchable control table
- Evidence count per control
- Last review dates

### Evidence Browser

- Full-text search of policy documents
- View original files
- Track ingestion history

### Audit Trail

- All changes logged to ledger
- Context JSON for each service
- stdout/stderr logs

## Best Practices

1. **Separate databases by purpose** - policies, evidence, controls
2. **Regular ingestion** - Schedule evidence updates
3. **Version control SQL** - Track dashboard changes
4. **Use composites** - Query across databases
5. **Automate reporting** - Export data for stakeholders

---

See also: [SQLPage Applications](../how-to-guides/sqlpage-apps.md), [Surveilr RSSDs](../how-to-guides/surveilr-rssds.md)