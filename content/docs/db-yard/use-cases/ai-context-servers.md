---
title: AI Context Servers
description: Serving structured data to AI systems with db-yard.
---

# AI Context Servers

Serving structured data to AI systems with db-yard.

## Overview

AI systems often need access to structured context data - documentation, code examples, patterns, and domain knowledge. db-yard can serve multiple SQLite databases as context sources for AI applications.

## Architecture

```
+─────────────────────────────────────────────────────+
|                   AI Application                     |
|  +────────────────────────────────────────────────+ |
|  |           Context Retrieval Layer              | |
|  +─────────────────────┬──────────────────────────+ |
+────────────────────────┼────────────────────────────+
                         |
              +──────────v──────────+
              |      db-yard        |
              |   Proxy (8787)      |
              +──────────┬──────────+
                         |
        +────────────────┼────────────────+
        |                |                |
        v                v                v
+───────────────+ +───────────────+ +───────────────+
| Engineering   | | Middleware    | | Domain        |
| Patterns      | | Examples      | | Knowledge     |
| context.db    | | context.db    | | context.db    |
+───────────────+ +───────────────+ +───────────────+
```

## Implementation

### Step 1: Create Context Databases

```bash
# Engineering patterns database
sqlite3 cargo.d/ai-context/engineering.sqlite.db <<'EOF'
CREATE TABLE patterns (
    id INTEGER PRIMARY KEY,
    name TEXT,
    category TEXT,
    description TEXT,
    example_code TEXT,
    use_case TEXT
);

CREATE TABLE best_practices (
    id INTEGER PRIMARY KEY,
    topic TEXT,
    practice TEXT,
    rationale TEXT
);

-- Add surveilr tables for web UI
CREATE TABLE uniform_resource (
    id INTEGER PRIMARY KEY,
    uri TEXT,
    content TEXT,
    nature TEXT
);
EOF
```

```bash
# Middleware examples database
sqlite3 cargo.d/ai-context/middleware.sqlite.db <<'EOF'
CREATE TABLE examples (
    id INTEGER PRIMARY KEY,
    framework TEXT,
    pattern TEXT,
    code TEXT,
    explanation TEXT
);

CREATE TABLE uniform_resource (
    id INTEGER PRIMARY KEY,
    uri TEXT,
    content TEXT,
    nature TEXT
);
EOF
```

### Step 2: Populate with Context Data

```bash
# Add engineering patterns
sqlite3 cargo.d/ai-context/engineering.sqlite.db <<'EOF'
INSERT INTO patterns (name, category, description, example_code, use_case) VALUES
('Repository Pattern', 'Data Access',
 'Abstracts data storage and retrieval behind a collection-like interface',
 'class UserRepository {
    async findById(id: string): Promise<User> {
      return await this.db.query("SELECT * FROM users WHERE id = ?", [id]);
    }
  }',
 'Use when you need to decouple domain logic from data access'),

('Factory Pattern', 'Creational',
 'Creates objects without exposing instantiation logic',
 'function createLogger(type: string): Logger {
    switch(type) {
      case "console": return new ConsoleLogger();
      case "file": return new FileLogger();
    }
  }',
 'Use when object creation is complex or varies based on conditions');
EOF
```

### Step 3: Create API SQLPage Interface

```bash
sqlite3 cargo.d/ai-context/api.sqlite.db <<'EOF'
CREATE TABLE sqlpage_files (
    path TEXT PRIMARY KEY,
    contents TEXT,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- JSON API for patterns
INSERT INTO sqlpage_files (path, contents) VALUES
('api/patterns.sql', '
SELECT ''json'' AS component;
SELECT json_group_array(
    json_object(
        ''name'', name,
        ''category'', category,
        ''description'', description,
        ''example'', example_code
    )
) AS contents
FROM engineering.patterns;
'),

-- Search endpoint
('api/search.sql', '
SELECT ''json'' AS component;
SELECT json_group_array(
    json_object(
        ''source'', source,
        ''content'', content
    )
) AS contents
FROM (
    SELECT ''pattern'' as source, name || '': '' || description as content
    FROM engineering.patterns
    WHERE name LIKE ''%'' || $q || ''%'' OR description LIKE ''%'' || $q || ''%''
    UNION ALL
    SELECT ''example'' as source, framework || '': '' || pattern as content
    FROM middleware.examples
    WHERE pattern LIKE ''%'' || $q || ''%''
);
');

-- Attach other databases for cross-queries
ATTACH DATABASE ''cargo.d/ai-context/engineering.sqlite.db'' AS engineering;
ATTACH DATABASE ''cargo.d/ai-context/middleware.sqlite.db'' AS middleware;
EOF
```

### Step 4: Deploy Context Servers

```bash
./bin/yard.ts start
./bin/web-ui/serve.ts
```

Access points:
- Engineering patterns (surveilr): `http://localhost:8787/apps/surveilr/ai-context/engineering/`
- Middleware examples (surveilr): `http://localhost:8787/apps/surveilr/ai-context/middleware/`
- API (SQLPage): `http://localhost:8787/apps/sqlpage/ai-context/api/`

## AI Integration

### Python Client Example

```python
import requests

class ContextClient:
    def __init__(self, base_url="http://localhost:8787"):
        self.base_url = base_url

    def get_patterns(self):
        resp = requests.get(f"{self.base_url}/apps/sqlpage/ai-context/api/api/patterns.sql")
        return resp.json()

    def search(self, query):
        resp = requests.get(
            f"{self.base_url}/apps/sqlpage/ai-context/api/api/search.sql",
            params={"q": query}
        )
        return resp.json()

# Usage
client = ContextClient()
patterns = client.get_patterns()
results = client.search("repository")
```

### LangChain Integration

```python
from langchain.tools import Tool
from langchain.agents import initialize_agent

def search_patterns(query: str) -> str:
    client = ContextClient()
    results = client.search(query)
    return str(results)

pattern_tool = Tool(
    name="pattern_search",
    func=search_patterns,
    description="Search engineering patterns and middleware examples"
)

agent = initialize_agent(
    tools=[pattern_tool],
    llm=llm,
    agent="zero-shot-react-description"
)
```

### Direct SQL Access

For AI systems that can query SQLite directly:

```python
import sqlite3

def get_context(query: str) -> list:
    conn = sqlite3.connect("cargo.d/ai-context/engineering.sqlite.db")
    cursor = conn.execute(
        "SELECT * FROM patterns WHERE description LIKE ?",
        [f"%{query}%"]
    )
    return cursor.fetchall()
```

## Use Cases

### Code Generation Context

Provide patterns and examples to help AI generate better code:

```sql
-- Query relevant patterns for a task
SELECT name, example_code
FROM patterns
WHERE use_case LIKE '%' || $task || '%';
```

### Documentation Retrieval

Store and query documentation:

```sql
CREATE TABLE documentation (
    id INTEGER PRIMARY KEY,
    section TEXT,
    content TEXT,
    embedding BLOB  -- Optional: store embeddings
);
```

### Domain Knowledge Base

Store domain-specific knowledge:

```sql
CREATE TABLE domain_concepts (
    id INTEGER PRIMARY KEY,
    term TEXT,
    definition TEXT,
    related_terms TEXT,
    examples TEXT
);
```

## Scaling

### Multiple Context Categories

```
cargo.d/
├── ai-context/
│   ├── engineering/
│   │   └── patterns.sqlite.db
│   ├── frameworks/
│   │   ├── react.sqlite.db
│   │   └── vue.sqlite.db
│   ├── languages/
│   │   ├── typescript.sqlite.db
│   │   └── python.sqlite.db
│   └── domain/
│       ├── healthcare.sqlite.db
│       └── finance.sqlite.db
```

### Composite Queries

```bash
./bin/yard.ts cc --scope cross-tenant > context-composite.sql
```

Query across all context sources:

```sql
SELECT * FROM all_patterns
WHERE category = 'Data Access'
ORDER BY relevance DESC;
```

## Best Practices

1. **Structured data** - Use consistent schemas
2. **JSON APIs** - Provide machine-readable endpoints
3. **Version context** - Track changes to knowledge bases
4. **Refresh regularly** - Keep context up to date
5. **Embed for search** - Consider vector embeddings for semantic search

---

See also: [Composite Connections](../advanced/composites.md), [SQLPage Applications](../how-to-guides/sqlpage-apps.md)