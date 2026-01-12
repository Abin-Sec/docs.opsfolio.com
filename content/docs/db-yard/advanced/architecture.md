---
title: Architecture Overview
description: Understanding how db-yard processes and manages database cargo.
---

# Architecture Overview

Understanding how db-yard processes and manages database cargo.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI (yard.ts)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  start   │  │    ps    │  │   kill   │  │    cc    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────┐
│                         lib/                                 │
│  ┌───────────┐  ┌────────────┐  ┌─────────────┐            │
│  │ discover  │→ │  tabular   │→ │  exposable  │            │
│  │  (glob)   │  │(classify)  │  │  (filter)   │            │
│  └───────────┘  └────────────┘  └──────┬──────┘            │
│                                         │                    │
│  ┌───────────────────────────────────────────────────┐      │
│  │                     spawn                          │      │
│  │  ┌─────────┐  ┌──────────┐  ┌────────────────┐   │      │
│  │  │ process │  │  ledger  │  │ reconciliation │   │      │
│  │  │ mgmt    │  │  writer  │  │    (smart)     │   │      │
│  │  └─────────┘  └──────────┘  └────────────────┘   │      │
│  └───────────────────────────────────────────────────┘      │
│                                                              │
│  ┌─────────────┐  ┌────────────────┐  ┌──────────────┐     │
│  │ composite   │  │ reverse-proxy  │  │ materialize  │     │
│  │   (SQL)     │  │    -conf       │  │   (watch)    │     │
│  └─────────────┘  └────────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
        ↑                           ↑
        │                           │
┌───────┴───────────────────────────┴─────────────────────────┐
│                       Web UI                                 │
│  ┌─────────┐  ┌───────────┐  ┌──────────┐                  │
│  │  serve  │  │    app    │  │  proxy   │                  │
│  │ (Hono)  │  │ (routes)  │  │(forward) │                  │
│  └─────────┘  └───────────┘  └──────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

## Processing Pipeline

### Stage 1: Discovery

**Location:** `lib/discover.ts`

Recursively walks cargo directories using glob patterns:

```typescript
// Discovery yields encounters
for await (const encounter of discover(cargoHome)) {
  console.log(encounter.relativePath);
}
```

Key types:

```typescript
interface Encounter {
  relativePath: string;
  absolutePath: string;
  provenance: string;
}

interface EncounterSource {
  walk(): AsyncIterable<Encounter>;
}
```

### Stage 2: Classification

**Location:** `lib/tabular.ts`

Inspects each database to determine its type:

```typescript
const classification = await classify(encounter);
// Returns: "sqlpage" | "surveilr" | "plain-sqlite" | "duckdb" | "excel"
```

Classification logic:

```
Open SQLite connection
  → Query sqlite_master for tables
  → If uniform_resource exists → surveilr
  → Else if sqlpage_files exists → sqlpage
  → Else → plain-sqlite
```

### Stage 3: Exposable Filtering

**Location:** `lib/exposable.ts`

Filters to only exposable services:

```typescript
for await (const exposable of exposables(encounters)) {
  // Only sqlpage and surveilr services
}
```

### Stage 4: Port Allocation

**Location:** `lib/spawn.ts`

Assigns ports incrementally:

```typescript
const port = await allocatePort(basePort);
// Checks system availability
// Checks existing ledger entries
// Returns first available
```

### Stage 5: Process Spawning

**Location:** `lib/spawn.ts`

Spawns service processes:

```typescript
const process = await spawn({
  executable: "sqlpage",
  args: ["--port", port, "--database", dbPath],
  env: {
    DB_YARD_SERVICE_ID: serviceId,
    DB_YARD_SESSION_ID: sessionId,
  }
});
```

### Stage 6: Ledger Writing

**Location:** `lib/spawn.ts`

Writes operational state to disk:

```typescript
await writeLedger({
  sessionDir,
  relativePath,
  context: { serviceId, port, pid, ... },
  process: { stdout, stderr }
});
```

## Key Modules

### lib/discover.ts

Generic discovery adapter system.

```typescript
// Main exports
export function discover(root: string): AsyncIterable<Encounter>;
export function createFsSource(root: string): EncounterSource;
```

Features:
- Glob-based file matching
- Lazy content loading
- Pluggable source adapters

### lib/tabular.ts

Database type classification.

```typescript
// Main exports
export async function classify(path: string): Promise<Classification>;
export function hasTable(db: Database, table: string): boolean;
```

Features:
- SQLite table inspection
- Extension-based detection
- Cheap, deterministic checks

### lib/exposable.ts

Exposable service primitives.

```typescript
// Main exports
export function exposables(
  encounters: AsyncIterable<Encounter>
): AsyncIterable<Exposable>;
```

Features:
- Classification filtering
- Metadata enrichment
- Proxy prefix derivation

### lib/spawn.ts

Process management and ledger tracking.

```typescript
// Main exports
export async function spawn(config: SpawnConfig): Promise<SpawnResult>;
export async function smartSpawn(config: SmartSpawnConfig): Promise<void>;
export async function killAll(): Promise<void>;
```

Features:
- Process spawning with tagging
- Ledger file writing
- Smart spawn reconciliation
- Process cleanup

### lib/materialize.ts

Materialization and watch orchestration.

```typescript
// Main exports
export async function materialize(config: MaterializeConfig): Promise<void>;
export async function watch(config: WatchConfig): Promise<void>;
```

Features:
- One-shot materialization
- Continuous watch mode
- SIGINT cleanup handling

### lib/composite.ts

Composite SQL generation.

```typescript
// Main exports
export function generateComposite(config: CompositeConfig): string;
```

Features:
- SQLite ATTACH statements
- DuckDB SQLite extension
- Scope-based generation

### lib/reverse-proxy-conf.ts

Proxy configuration generation.

```typescript
// Main exports
export function generateNginx(services: Service[]): string;
export function generateTraefik(services: Service[]): string;
```

Features:
- NGINX configuration
- Traefik configuration
- Prefix-based routing

## Data Flow Example

```
cargo.d/apps/dashboard.sqlite.db
    │
    ▼ [lib/discover.ts]
Encounter { relativePath: "apps/dashboard.sqlite.db", ... }
    │
    ▼ [lib/tabular.ts]
Classification: "sqlpage"
    │
    ▼ [lib/exposable.ts]
Exposable { kind: "sqlpage", proxyPrefix: "/apps/sqlpage/apps/dashboard", ... }
    │
    ▼ [lib/spawn.ts - allocatePort]
port: 3000
    │
    ▼ [lib/spawn.ts - spawn]
Process { pid: 12345, ... }
    │
    ▼ [lib/spawn.ts - writeLedger]
ledger.d/session/apps/dashboard.sqlite.db.context.json
```

## Type System

Key types used throughout:

```typescript
// Discovery
interface Encounter {
  relativePath: string;
  absolutePath: string;
  provenance: string;
}

// Classification
type Classification =
  | "sqlpage"
  | "surveilr"
  | "plain-sqlite"
  | "duckdb"
  | "excel";

// Exposable
interface Exposable {
  encounter: Encounter;
  kind: "sqlpage" | "surveilr";
  proxyPrefix: string;
}

// Spawn
interface ServiceContext {
  serviceId: string;
  sessionId: string;
  provenance: string;
  relativePath: string;
  kind: Classification;
  port: number;
  proxyPrefix: string;
  pid: number;
  executable: string;
  args: string[];
  spawnedAt: string;
}
```

## Extension Points

### 1. Custom Discovery Sources

Implement the `EncounterSource` interface:

```typescript
const mySource: EncounterSource = {
  async *walk() {
    // Yield encounters from any source
  }
};
```

### 2. Custom Classifiers

Add new classification logic:

```typescript
function classifyCustom(path: string): Classification {
  // Custom classification logic
}
```

### 3. Custom Service Types

Extend the spawn logic for new service types:

```typescript
const customSpawner = {
  kind: "my-service",
  executable: "my-executable",
  buildArgs: (config) => [...],
};
```

## Technology Stack

| Component | Technology |
|-----------|------------|
| Runtime | Deno 2.x |
| Language | TypeScript |
| CLI | Cliffy |
| Web | Hono |
| Validation | Zod |
| Patterns | ts-pattern |

---

Next: [Composite Connections](./composites.md)