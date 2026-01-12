---
title: SQLPage Applications
description: Building and deploying SQLPage apps with db-yard.
---

# SQLPage Applications

Building and deploying SQLPage apps with db-yard.

## Overview

SQLPage is a web application framework where pages are defined as SQL queries stored in a SQLite database. db-yard automatically discovers and serves these applications.

## Prerequisites

- SQLPage binary installed and in PATH
- db-yard installed
- SQLite3 for database creation

## Creating a SQLPage Database

### Step 1: Create the Database

```bash
sqlite3 cargo.d/my-app.sqlite.db <<'EOF'
-- Create the sqlpage_files table (required for detection)
CREATE TABLE sqlpage_files (
    path TEXT PRIMARY KEY,
    contents TEXT,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOF
```

### Step 2: Add Pages

```bash
sqlite3 cargo.d/my-app.sqlite.db <<'EOF'
-- Add an index page
INSERT INTO sqlpage_files (path, contents) VALUES
('index.sql', '
SELECT ''hero'' AS component,
       ''My Application'' AS title,
       ''Built with SQLPage and db-yard'' AS description;

SELECT ''card'' AS component;
SELECT ''Dashboard'' AS title,
       ''View metrics and statistics'' AS description,
       ''dashboard.sql'' AS link;
SELECT ''Reports'' AS title,
       ''Generate reports'' AS description,
       ''reports.sql'' AS link;
');

-- Add a dashboard page
INSERT INTO sqlpage_files (path, contents) VALUES
('dashboard.sql', '
SELECT ''text'' AS component,
       ''Welcome to the Dashboard'' AS contents;

SELECT ''chart'' AS component,
       ''line'' AS type,
       ''Monthly Sales'' AS title;
SELECT month, sales FROM sales_data ORDER BY month;
');
EOF
```

### Step 3: Add Data Tables

```bash
sqlite3 cargo.d/my-app.sqlite.db <<'EOF'
-- Create application data tables
CREATE TABLE sales_data (
    month TEXT,
    sales INTEGER
);

INSERT INTO sales_data VALUES
    ('Jan', 1200),
    ('Feb', 1500),
    ('Mar', 1800),
    ('Apr', 2100);
EOF
```

## Deploying the App

### Start db-yard

```bash
./bin/yard.ts start
```

Output:
```
[yard] Discovered: cargo.d/my-app.sqlite.db
[yard] Classified as: sqlpage
[yard] Spawning on port 3000
```

### Access the App

Direct access:
```bash
open http://localhost:3000
```

Via proxy:
```bash
./bin/web-ui/serve.ts
open http://localhost:8787/apps/sqlpage/my-app/
```

## Updating Pages

SQLPage reads from the database on each request, so updates are immediate:

```bash
sqlite3 cargo.d/my-app.sqlite.db <<'EOF'
UPDATE sqlpage_files
SET contents = '
SELECT ''text'' AS component,
       ''Updated content!'' AS contents;
'
WHERE path = 'index.sql';
EOF
```

Refresh the page to see changes.

## Adding New Pages

```bash
sqlite3 cargo.d/my-app.sqlite.db <<'EOF'
INSERT INTO sqlpage_files (path, contents) VALUES
('about.sql', '
SELECT ''text'' AS component,
       ''About Us'' AS title,
       ''This is the about page'' AS contents;
');
EOF
```

Access at: `http://localhost:3000/about.sql`

## Organizing Pages

Use subdirectories in the path:

```sql
INSERT INTO sqlpage_files (path, contents) VALUES
('admin/users.sql', 'SELECT ''list'' AS component...');
INSERT INTO sqlpage_files (path, contents) VALUES
('admin/settings.sql', 'SELECT ''form'' AS component...');
```

Access at:
- `http://localhost:3000/admin/users.sql`
- `http://localhost:3000/admin/settings.sql`

## Configuration

### SQLPage Settings

SQLPage can be configured via the database:

```sql
-- Create configuration table
CREATE TABLE IF NOT EXISTS sqlpage_settings (
    key TEXT PRIMARY KEY,
    value TEXT
);

-- Set configuration
INSERT INTO sqlpage_settings VALUES
    ('site_title', 'My Application'),
    ('max_rows', '1000');
```

### Environment Variables

Pass environment variables through SQLPage:

```sql
SELECT sqlpage.environment_variable('API_KEY') AS api_key;
```

## Multiple Apps

Deploy multiple SQLPage apps:

```
cargo.d/
├── dashboard/
│   └── main.sqlite.db
├── reports/
│   └── analytics.sqlite.db
└── admin/
    └── management.sqlite.db
```

Each gets its own port and proxy prefix:
- `/apps/sqlpage/dashboard/main`
- `/apps/sqlpage/reports/analytics`
- `/apps/sqlpage/admin/management`

## Common Patterns

### Navigation Menu

```sql
SELECT 'shell' AS component,
       'My App' AS title,
       'Home' AS menu_item,
       '/' AS link,
       'Dashboard' AS menu_item,
       '/dashboard.sql' AS link,
       'Reports' AS menu_item,
       '/reports.sql' AS link;
```

### Data Table

```sql
SELECT 'table' AS component,
       'Users' AS title;
SELECT id, name, email, created_at
FROM users
ORDER BY created_at DESC
LIMIT 100;
```

### Form Input

```sql
SELECT 'form' AS component,
       'Add User' AS title;
SELECT 'name' AS name, 'text' AS type, 'Name' AS label;
SELECT 'email' AS name, 'email' AS type, 'Email' AS label;

-- Handle submission
INSERT INTO users (name, email)
SELECT $name, $email
WHERE $name IS NOT NULL;
```

## Debugging

### Check Database Structure

```bash
sqlite3 cargo.d/my-app.sqlite.db ".schema"
```

### List All Pages

```bash
sqlite3 cargo.d/my-app.sqlite.db "SELECT path FROM sqlpage_files"
```

### View Page Content

```bash
sqlite3 cargo.d/my-app.sqlite.db "SELECT contents FROM sqlpage_files WHERE path='index.sql'"
```

### Check Logs

```bash
cat ledger.d/*/my-app.sqlite.db.stderr.log
```

## Best Practices

1. **Use meaningful filenames** - They become URL paths
2. **Organize by feature** - Use subdirectories
3. **Keep SQL readable** - Format with proper indentation
4. **Test locally** - Use direct port access during development
5. **Version control** - Export and track SQL files

---

See also: [Your First Cargo](../getting-started/first-cargo.md), [SQLPage Documentation](https://sql-page.com/)