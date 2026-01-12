---
title: Testing Strategy
description: How to write and run tests for db-yard.
---

# Testing Strategy

How to write and run tests for db-yard.

## Overview

db-yard uses Deno's built-in test runner with a focus on unit tests and golden-string regression tests.

## Running Tests

### All Tests

```bash
deno test
```

### Specific File

```bash
deno test lib/discover_test.ts
```

### Specific Test

```bash
deno test --filter "discover - finds sqlite files"
```

### With Coverage

```bash
deno test --coverage=coverage
deno coverage coverage
```

### Watch Mode

```bash
deno test --watch
```

## Test Structure

### Basic Test

```typescript
import { assertEquals } from "@std/assert";

Deno.test("function - describes behavior", () => {
  // Arrange
  const input = "test";

  // Act
  const result = myFunction(input);

  // Assert
  assertEquals(result, "expected");
});
```

### Async Test

```typescript
Deno.test("async function - describes behavior", async () => {
  const result = await asyncFunction();
  assertEquals(result, "expected");
});
```

### Test with Cleanup

```typescript
Deno.test("spawn - creates ledger files", async () => {
  // Setup
  const tempDir = await Deno.makeTempDir();

  try {
    // Test
    await spawn({ ledgerHome: tempDir, ... });
    const files = await Deno.readDir(tempDir);
    // Assert...
  } finally {
    // Cleanup
    await Deno.remove(tempDir, { recursive: true });
  }
});
```

## Test Categories

### Unit Tests

Test individual functions in isolation:

```typescript
// lib/path_test.ts
Deno.test("normalizePath - removes compound extensions", () => {
  assertEquals(normalizePath("file.sqlite.db"), "file");
  assertEquals(normalizePath("file.db"), "file");
});
```

### Integration Tests

Test module interactions:

```typescript
// lib/spawn_test.ts
Deno.test("smartSpawn - skips already running services", async () => {
  // Setup mock ledger
  // Spawn first time
  // Spawn second time
  // Assert only one process
});
```

### Golden-String Tests

Compare output against known-good values:

```typescript
// lib/composite_test.ts
Deno.test("generateComposite - matches golden output", () => {
  const result = generateComposite({
    volumeRoot: "/var/db-yard",
    scope: "admin",
    dialect: "SQLite",
  });

  const golden = await Deno.readTextFile("support/fixtures/composite-admin.sql");
  assertEquals(result, golden);
});
```

## Test Fixtures

### Location

```
support/
└── fixtures/
    ├── sakila.db         # Sample database
    ├── chinook.db        # Sample database
    ├── composite-admin.sql   # Golden output
    └── README.md
```

### Creating Fixtures

```bash
# Create test database
sqlite3 support/fixtures/test.db < test-schema.sql

# Generate golden file
./bin/yard.ts cc --scope admin > support/fixtures/composite-admin.sql
```

## Mocking

### Mock File System

```typescript
Deno.test("discover - handles empty directory", async () => {
  const tempDir = await Deno.makeTempDir();

  const results = [];
  for await (const encounter of discover(tempDir)) {
    results.push(encounter);
  }

  assertEquals(results.length, 0);
  await Deno.remove(tempDir);
});
```

### Mock Process

```typescript
// Use actual process for integration tests
// Or create minimal test databases

Deno.test("classify - detects sqlpage", async () => {
  const tempDir = await Deno.makeTempDir();
  const dbPath = `${tempDir}/test.db`;

  // Create test database with sqlpage_files table
  const db = new Database(dbPath);
  db.execute("CREATE TABLE sqlpage_files (path TEXT)");
  db.close();

  const result = await classify(dbPath);
  assertEquals(result, "sqlpage");

  await Deno.remove(tempDir, { recursive: true });
});
```

## Test Patterns

### Table-Driven Tests

```typescript
const testCases = [
  { input: "file.db", expected: "file" },
  { input: "file.sqlite.db", expected: "file" },
  { input: "file.sqlite3", expected: "file" },
];

for (const { input, expected } of testCases) {
  Deno.test(`normalizePath - ${input}`, () => {
    assertEquals(normalizePath(input), expected);
  });
}
```

### Grouping Tests

```typescript
Deno.test("discover", async (t) => {
  await t.step("finds sqlite files", async () => {
    // ...
  });

  await t.step("skips hidden files", async () => {
    // ...
  });

  await t.step("handles nested directories", async () => {
    // ...
  });
});
```

## Assertions

### Common Assertions

```typescript
import {
  assertEquals,
  assertNotEquals,
  assertThrows,
  assertRejects,
  assertStringIncludes,
} from "@std/assert";

// Equality
assertEquals(actual, expected);
assertNotEquals(actual, unexpected);

// Strings
assertStringIncludes(output, "expected substring");

// Errors
assertThrows(() => throwingFunction(), Error, "error message");
await assertRejects(async () => await asyncThrowingFunction());
```

## Continuous Integration

### GitHub Actions Example

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: denoland/setup-deno@v1
      - run: deno test
      - run: deno lint
      - run: deno fmt --check
```

## Best Practices

1. **Test behavior, not implementation** - Focus on what, not how
2. **One assertion per test** - When practical
3. **Descriptive names** - `classify - returns surveilr for uniform_resource`
4. **Clean up resources** - Use try/finally or temp directories
5. **Fast tests** - Avoid slow operations in unit tests
6. **Deterministic** - No flaky tests

---

Next: [Pull Request Workflow](./pull-requests.md)