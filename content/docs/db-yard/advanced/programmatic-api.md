---
title: Programmatic API
description: Using db-yard as a library in your Deno/TypeScript code.
---

# Programmatic API

Using db-yard as a library in your Deno/TypeScript code.

## Overview

db-yard's functionality is exposed as TypeScript modules that can be imported and used programmatically. This enables building custom tooling, integrations, and extensions.

## Module Structure

```
lib/
├── discover.ts       # File discovery
├── tabular.ts        # Database classification
├── exposable.ts      # Exposable service filtering
├── spawn.ts          # Process management
├── materialize.ts    # Materialization orchestration
├── composite.ts      # Composite SQL generation
├── reverse-proxy-conf.ts  # Proxy config generation
├── path.ts           # Path utilities
└── spawn-event.ts    # Event streaming
```

## Discovery API

### Basic Discovery

```typescript
import { discover } from "./lib/discover.ts";

// Discover all databases in cargo directory
for await (const encounter of discover("./cargo.d")) {
  console.log(encounter.relativePath);
  console.log(encounter.absolutePath);
  console.log(encounter.provenance);
}
```

### Custom Source

```typescript
import { createFsSource } from "./lib/discover.ts";

const source = createFsSource("/var/db-yard/cargo");

for await (const encounter of source.walk()) {
  console.log(encounter);
}
```

### Types

```typescript
interface Encounter {
  relativePath: string;  // Path relative to cargo root
  absolutePath: string;  // Full filesystem path
  provenance: string;    // Canonical identifier
}

interface EncounterSource {
  walk(): AsyncIterable<Encounter>;
}
```

## Classification API

### Classify a Database

```typescript
import { classify, hasTable } from "./lib/tabular.ts";

const classification = await classify("/path/to/database.sqlite.db");
// Returns: "sqlpage" | "surveilr" | "plain-sqlite" | "duckdb" | "excel"
```

### Table Inspection

```typescript
import { Database } from "jsr:@db/sqlite";
import { hasTable } from "./lib/tabular.ts";

const db = new Database("/path/to/database.db");
if (hasTable(db, "sqlpage_files")) {
  console.log("This is a SQLPage app");
}
```

### Types

```typescript
type Classification =
  | "sqlpage"
  | "surveilr"
  | "plain-sqlite"
  | "duckdb"
  | "excel";
```

## Exposable API

### Filter Exposables

```typescript
import { discover } from "./lib/discover.ts";
import { exposables } from "./lib/exposable.ts";

const encounters = discover("./cargo.d");

for await (const exposable of exposables(encounters)) {
  console.log(exposable.kind);        // "sqlpage" or "surveilr"
  console.log(exposable.proxyPrefix); // URL path
  console.log(exposable.encounter);   // Original encounter
}
```

### Types

```typescript
interface Exposable {
  encounter: Encounter;
  kind: "sqlpage" | "surveilr";
  proxyPrefix: string;
}
```

## Spawn API

### Spawn a Service

```typescript
import { spawn } from "./lib/spawn.ts";

const result = await spawn({
  executable: "sqlpage",
  args: ["--port", "3000", "--database", "/path/to/db.sqlite"],
  cwd: "/working/dir",
  env: {
    DB_YARD_SERVICE_ID: "my-service",
    DB_YARD_SESSION_ID: "session-1",
  },
  stdout: "piped",
  stderr: "piped",
});

console.log(result.pid);
console.log(result.process);
```

### Smart Spawn

```typescript
import { smartSpawn } from "./lib/spawn.ts";

await smartSpawn({
  exposables: myExposables,
  ledgerHome: "./ledger.d",
  sessionId: "my-session",
  portBase: 3000,
  onSpawn: (context) => {
    console.log(`Spawned ${context.serviceId} on port ${context.port}`);
  },
});
```

### Kill All

```typescript
import { killAll } from "./lib/spawn.ts";

await killAll();
```

### Types

```typescript
interface SpawnConfig {
  executable: string;
  args: string[];
  cwd?: string;
  env?: Record<string, string>;
  stdout?: "piped" | "inherit" | "null";
  stderr?: "piped" | "inherit" | "null";
}

interface SpawnResult {
  pid: number;
  process: Deno.ChildProcess;
}

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

## Materialization API

### One-Shot Materialization

```typescript
import { materialize } from "./lib/materialize.ts";

await materialize({
  cargoHome: "./cargo.d",
  ledgerHome: "./ledger.d",
  portBase: 3000,
  verbose: "essential",
});
```

### Watch Mode

```typescript
import { watch } from "./lib/materialize.ts";

const controller = new AbortController();

// Start watching
watch({
  cargoHome: "./cargo.d",
  ledgerHome: "./ledger.d",
  portBase: 3000,
  signal: controller.signal,
});

// Later: stop watching
controller.abort();
```

## Composite API

### Generate Composite SQL

```typescript
import { generateComposite } from "./lib/composite.ts";

const sql = generateComposite({
  volumeRoot: "/var/db-yard",
  scope: "admin",
  dialect: "SQLite",
});

console.log(sql);
```

### DuckDB with SQLite Extension

```typescript
const sql = generateComposite({
  volumeRoot: "/var/db-yard",
  scope: "cross-tenant",
  dialect: "DuckDB",
  duckdbSqliteExt: true,
});
```

### Types

```typescript
interface CompositeConfig {
  volumeRoot: string;
  scope: "admin" | "tenant" | "cross-tenant";
  tenantId?: string;  // Required for tenant scope
  dialect: "SQLite" | "DuckDB";
  duckdbSqliteExt?: boolean;
}
```

## Reverse Proxy API

### Generate NGINX Config

```typescript
import { generateNginx } from "./lib/reverse-proxy-conf.ts";

const services = [
  { serviceId: "app1", port: 3000, proxyPrefix: "/apps/sqlpage/app1" },
  { serviceId: "app2", port: 3001, proxyPrefix: "/apps/sqlpage/app2" },
];

const config = generateNginx(services);
console.log(config);
```

### Generate Traefik Config

```typescript
import { generateTraefik } from "./lib/reverse-proxy-conf.ts";

const config = generateTraefik(services);
console.log(config);
```

## Event API

### Listen to Spawn Events

```typescript
import { spawnEvents } from "./lib/spawn-event.ts";

for await (const event of spawnEvents()) {
  switch (event.type) {
    case "spawned":
      console.log(`Service ${event.serviceId} spawned`);
      break;
    case "killed":
      console.log(`Service ${event.serviceId} killed`);
      break;
    case "crashed":
      console.log(`Service ${event.serviceId} crashed`);
      break;
  }
}
```

## Complete Example

```typescript
import { discover } from "./lib/discover.ts";
import { exposables } from "./lib/exposable.ts";
import { smartSpawn, killAll } from "./lib/spawn.ts";
import { generateNginx } from "./lib/reverse-proxy-conf.ts";

async function main() {
  // 1. Discover and filter
  const encounters = discover("./cargo.d");
  const services: ServiceContext[] = [];

  // 2. Smart spawn
  await smartSpawn({
    exposables: exposables(encounters),
    ledgerHome: "./ledger.d",
    sessionId: new Date().toISOString(),
    portBase: 3000,
    onSpawn: (ctx) => services.push(ctx),
  });

  // 3. Generate proxy config
  const nginxConfig = generateNginx(services);
  await Deno.writeTextFile("nginx.conf", nginxConfig);

  console.log(`Spawned ${services.length} services`);
  console.log("NGINX config written to nginx.conf");

  // 4. Wait for signal
  Deno.addSignalListener("SIGINT", async () => {
    await killAll();
    Deno.exit(0);
  });
}

main();
```

## Error Handling

```typescript
import { discover } from "./lib/discover.ts";
import { classify } from "./lib/tabular.ts";

try {
  for await (const encounter of discover("./cargo.d")) {
    try {
      const classification = await classify(encounter.absolutePath);
      console.log(`${encounter.relativePath}: ${classification}`);
    } catch (err) {
      console.error(`Failed to classify ${encounter.relativePath}:`, err);
    }
  }
} catch (err) {
  console.error("Discovery failed:", err);
}
```

---

See also: [Architecture Overview](./architecture.md)