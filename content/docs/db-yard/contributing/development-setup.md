---
title: Development Setup
description: Setting up your environment for db-yard development.
---

# Development Setup

Setting up your environment for db-yard development.

## Prerequisites

### Deno

db-yard requires Deno 2.x:

```bash
# Install Deno
curl -fsSL https://deno.land/install.sh | sh

# Verify installation
deno --version
```

### Git

```bash
git --version
```

### Optional: SQLPage and Surveilr

For testing service spawning:

```bash
# Install SQLPage
# See: https://sql-page.com/

# Install surveilr
# See: https://surveilr.com/
```

## Clone Repository

```bash
git clone https://github.com/netspective-labs/db-yard.git
cd db-yard
```

## Project Structure

```
db-yard/
├── bin/
│   ├── yard.ts           # Main CLI entry point
│   └── web-ui/
│       ├── serve.ts      # Web UI server
│       ├── app.ts        # UI routes
│       ├── proxy.ts      # Proxy implementation
│       └── asset/        # Static assets
├── lib/
│   ├── discover.ts       # File discovery
│   ├── tabular.ts        # Database classification
│   ├── exposable.ts      # Exposable filtering
│   ├── spawn.ts          # Process management
│   ├── materialize.ts    # Materialization
│   ├── composite.ts      # Composite SQL
│   ├── reverse-proxy-conf.ts  # Proxy config
│   ├── path.ts           # Path utilities
│   ├── spawn-event.ts    # Event streaming
│   └── *_test.ts         # Test files
├── cargo.d/              # Example databases
├── support/
│   ├── rfc/              # Design documents
│   └── fixtures/         # Test fixtures
├── deno.jsonc            # Deno configuration
├── import_map.json       # Import mappings
└── README.md
```

## Running the CLI

```bash
# Show help
./bin/yard.ts --help

# Start services
./bin/yard.ts start

# List processes
./bin/yard.ts ps

# Start web UI
./bin/web-ui/serve.ts
```

## Running Tests

```bash
# Run all tests
deno test

# Run specific test file
deno test lib/discover_test.ts

# Run with coverage
deno test --coverage=coverage
deno coverage coverage
```

## Development Workflow

### 1. Create a Feature Branch

```bash
git checkout -b feature/my-feature
```

### 2. Make Changes

Edit files in `lib/` or `bin/`.

### 3. Run Tests

```bash
deno test
```

### 4. Format Code

```bash
deno fmt
```

### 5. Lint Code

```bash
deno lint
```

### 6. Commit Changes

```bash
git add .
git commit -m "feat: add my feature"
```

## Editor Setup

### VS Code

Install the Deno extension:

```json
// .vscode/settings.json
{
  "deno.enable": true,
  "deno.lint": true,
  "deno.unstable": true,
  "editor.formatOnSave": true,
  "[typescript]": {
    "editor.defaultFormatter": "denoland.vscode-deno"
  }
}
```

### Vim/Neovim

Use deno-vim or coc-deno for LSP support.

## Debugging

### Debug CLI

```bash
deno run --inspect-brk -A ./bin/yard.ts start
```

Then connect Chrome DevTools to `chrome://inspect`.

### Debug Tests

```bash
deno test --inspect-brk lib/discover_test.ts
```

### Logging

Use `console.log` for debugging. Consider adding `--verbose comprehensive` for detailed output.

## Common Tasks

### Add a New Command

1. Edit `bin/yard.ts`
2. Add command using Cliffy
3. Implement in `lib/`
4. Add tests

### Add a New Module

1. Create `lib/new-module.ts`
2. Create `lib/new-module_test.ts`
3. Export from module
4. Import where needed

### Update Dependencies

Edit `import_map.json`:

```json
{
  "imports": {
    "@std/": "jsr:@std@1.0.0/"
  }
}
```

## Troubleshooting

### Permission Errors

Ensure you're running with necessary permissions:

```bash
deno run -A ./bin/yard.ts
```

### Module Not Found

Check `import_map.json` and `deno.jsonc` for correct mappings.

### Test Failures

Run individual tests to isolate issues:

```bash
deno test lib/specific_test.ts --filter "test name"
```

---

Next: [Code Guidelines](./code-guidelines.md)