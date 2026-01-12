---
title: Installation Guide
description: This guide covers installing db-yard and its dependencies.
---

# Installation Guide

This guide covers installing db-yard and its dependencies.

## Prerequisites

### Deno Runtime

db-yard requires Deno 2.x or later.

**Install Deno:**

```bash
# macOS/Linux
curl -fsSL https://deno.land/install.sh | sh

# Windows (PowerShell)
irm https://deno.land/install.ps1 | iex

# Homebrew
brew install deno
```

Verify installation:

```bash
deno --version
```

### SQLPage (Optional)

If you plan to run SQLPage applications:

```bash
# Download SQLPage binary
# See: https://sql-page.com/

# Or via package managers
brew install sqlpage  # macOS
```

### Surveilr (Optional)

If you plan to run surveilr RSSDs:

```bash
# See: https://surveilr.com/
```

## Installing db-yard

### Clone the Repository

```bash
git clone https://github.com/netspective-labs/db-yard.git
cd db-yard
```

### Verify Installation

```bash
# Show help
./bin/yard.ts --help

# Check version
./bin/yard.ts --version
```

## Directory Structure

After installation, you'll have:

```
db-yard/
├── bin/
│   ├── yard.ts           # Main CLI
│   └── web-ui/
│       └── serve.ts      # Web UI server
├── lib/                  # Core library
├── cargo.d/              # Default cargo directory (example databases)
├── ledger.d/             # Default ledger directory (created on first run)
└── support/              # Fixtures and documentation
```

## Configuration

db-yard uses sensible defaults but supports customization via CLI flags:

| Flag | Default | Description |
|------|---------|-------------|
| `--cargo-home` | `./cargo.d` | Cargo directory path |
| `--ledger-home` | `./ledger.d` | Ledger directory path |
| `--port-base` | `3000` | Starting port for allocation |

Example:

```bash
./bin/yard.ts start \
  --cargo-home /var/db-yard/cargo \
  --ledger-home /var/db-yard/ledger
```

## Next Steps

- [Quick Start Tutorial](./quick-start.md) - Run db-yard in 5 minutes
- [Your First Cargo](./first-cargo.md) - Deploy a complete workflow

---

See also: [Configuration Reference](../reference/configuration.md)