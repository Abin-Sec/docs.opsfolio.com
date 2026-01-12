---
title: Code Guidelines
description: Coding standards and conventions for db-yard.
---

# Code Guidelines

Coding standards and conventions for db-yard.

## TypeScript Style

### General Principles

1. **Type everything** - Avoid `any`, use explicit types
2. **Prefer immutability** - Use `Readonly<>` and `const`
3. **Small functions** - Single responsibility
4. **Clear naming** - Descriptive, not abbreviated

### Formatting

Use Deno's built-in formatter:

```bash
deno fmt
```

Configuration in `deno.jsonc`:

```json
{
  "fmt": {
    "indentWidth": 2,
    "lineWidth": 100,
    "singleQuote": false
  }
}
```

### Linting

Use Deno's linter:

```bash
deno lint
```

## Type Definitions

### Interfaces Over Types

Prefer interfaces for object shapes:

```typescript
// Good
interface ServiceContext {
  serviceId: string;
  port: number;
}

// Avoid for simple objects
type ServiceContext = {
  serviceId: string;
  port: number;
};
```

### Use Types for Unions

```typescript
type Classification = "sqlpage" | "surveilr" | "plain-sqlite";
```

### Readonly by Default

```typescript
interface Encounter {
  readonly relativePath: string;
  readonly absolutePath: string;
}

// Or use Readonly<>
function process(ctx: Readonly<ServiceContext>) {
  // ctx.port = 3000; // Error!
}
```

## Async Patterns

### Use Async Generators

For streaming data:

```typescript
async function* discover(root: string): AsyncIterable<Encounter> {
  for (const file of files) {
    yield { relativePath: file, ... };
  }
}
```

### Handle Errors

```typescript
try {
  await operation();
} catch (error) {
  if (error instanceof SpecificError) {
    // Handle specific error
  }
  throw error; // Re-throw if not handled
}
```

## Module Structure

### Single Responsibility

Each module should do one thing:

```
lib/
├── discover.ts      # Only discovery logic
├── tabular.ts       # Only classification logic
├── spawn.ts         # Only process management
```

### Clear Exports

```typescript
// Good: explicit exports
export { discover, createFsSource };
export type { Encounter, EncounterSource };

// Avoid: export everything
export * from "./internal.ts";
```

### Test Files

Co-locate tests with modules:

```
lib/
├── discover.ts
├── discover_test.ts
├── spawn.ts
└── spawn_test.ts
```

## Naming Conventions

### Files

- Lowercase with hyphens: `reverse-proxy-conf.ts`
- Test files: `module_test.ts`

### Variables and Functions

- camelCase: `serviceId`, `allocatePort`
- Descriptive: `getNextAvailablePort` not `getPort`

### Types and Interfaces

- PascalCase: `ServiceContext`, `Encounter`
- Descriptive: `EncounterSource` not `Source`

### Constants

- UPPER_SNAKE_CASE for true constants:
  ```typescript
  const DEFAULT_PORT = 3000;
  ```

## Error Handling

### Custom Errors

```typescript
class PortAllocationError extends Error {
  constructor(public port: number, message: string) {
    super(message);
    this.name = "PortAllocationError";
  }
}
```

### Error Messages

Be descriptive:

```typescript
// Good
throw new Error(`Port ${port} is already in use by process ${pid}`);

// Avoid
throw new Error("Port error");
```

## Documentation

### JSDoc Comments

Document public APIs:

```typescript
/**
 * Discovers database files in the given cargo directory.
 *
 * @param root - The cargo directory to scan
 * @yields Encounter objects for each discovered file
 *
 * @example
 * ```ts
 * for await (const file of discover("./cargo.d")) {
 *   console.log(file.relativePath);
 * }
 * ```
 */
export async function* discover(root: string): AsyncIterable<Encounter> {
  // ...
}
```

### Inline Comments

Explain "why", not "what":

```typescript
// Good: explains why
// Skip hidden files to avoid .git, .DS_Store, etc.
if (filename.startsWith(".")) continue;

// Avoid: explains what (obvious from code)
// Check if filename starts with dot
if (filename.startsWith(".")) continue;
```

## Testing

### Test Structure

```typescript
Deno.test("discover - finds sqlite files", async () => {
  // Arrange
  const tempDir = await Deno.makeTempDir();
  await Deno.writeFile(`${tempDir}/test.db`, new Uint8Array());

  // Act
  const results = [];
  for await (const encounter of discover(tempDir)) {
    results.push(encounter);
  }

  // Assert
  assertEquals(results.length, 1);
  assertEquals(results[0].relativePath, "test.db");

  // Cleanup
  await Deno.remove(tempDir, { recursive: true });
});
```

### Test Naming

Describe behavior:

```typescript
Deno.test("allocatePort - skips ports in use", ...);
Deno.test("classify - returns sqlpage for databases with sqlpage_files", ...);
Deno.test("spawn - writes context.json to ledger", ...);
```

## Dependencies

### Prefer Standard Library

Use Deno std when possible:

```typescript
import { join } from "@std/path";
import { walk } from "@std/fs";
```

### Pin Versions

In `import_map.json`:

```json
{
  "imports": {
    "@std/": "jsr:@std@1.0.0/",
    "@hono/": "jsr:@hono@4.11.3/"
  }
}
```

## Git Commits

### Conventional Commits

```
feat: add composite SQL generation
fix: handle port collision in watch mode
docs: update CLI reference
test: add spawn_test coverage
refactor: extract port allocation logic
```

### Commit Messages

- Present tense: "add feature" not "added feature"
- Imperative: "fix bug" not "fixes bug"
- Brief first line, details in body if needed

---

Next: [Testing Strategy](./testing.md)