---
title: Port Allocation
description: How db-yard assigns ports to services.
---

# Port Allocation

How db-yard assigns ports to services.

## Overview

db-yard automatically assigns ports to spawned services, starting from a configurable base and incrementing as needed. The allocation is deterministic and collision-aware.

## Allocation Strategy

1. Start at base port (default: 3000)
2. Check if port is in use
3. If in use, increment and retry
4. Assign first available port
5. Record in context JSON

## Default Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Base port | 3000 | Starting port number |
| Max search | 1000 | How many ports to try |

## Collision Avoidance

db-yard checks for port collisions in two ways:

### 1. System Port Check

Before assigning, db-yard verifies the port is not bound:

```typescript
// Attempt to bind the port
const listener = Deno.listen({ port });
listener.close();  // Release immediately
// If no error, port is available
```

### 2. Ledger Check

In smart-spawn mode, db-yard also checks existing ledger entries:

```typescript
// Read existing context files
// Avoid ports already allocated this session
```

## Example Allocation

```
Service 1: apps/dashboard.db
  → Check port 3000... available
  → Assigned port 3000

Service 2: apps/metrics.db
  → Check port 3001... available
  → Assigned port 3001

Service 3: apps/status.db
  → Check port 3002... in use (external process)
  → Check port 3003... available
  → Assigned port 3003
```

## Configuring Base Port

Override the default base port:

```bash
./bin/yard.ts start --port-base 8000
```

Now allocation starts at 8000:

```
Service 1 → 8000
Service 2 → 8001
...
```

## Port Persistence

Once a port is assigned, it's recorded in the context JSON:

```json
{
  "serviceId": "dashboard.db",
  "port": 3000,
  ...
}
```

### Smart Spawn Port Reuse

In smart-spawn mode, if a service is already running with a recorded port, db-yard reuses that port rather than allocating a new one.

This ensures:
- Stable URLs between restarts
- Proxy configuration remains valid
- No unnecessary port churn

## Watch Mode Behavior

In watch mode:

1. New cargo → Allocate new port
2. Existing cargo → Reuse assigned port
3. Removed cargo → Port becomes available for future use

## Port Ranges

Consider these typical ranges:

| Range | Typical Use |
|-------|-------------|
| 1-1023 | Privileged (requires root) |
| 1024-49151 | Registered ports |
| 49152-65535 | Dynamic/private ports |

db-yard defaults to 3000+ which is in the registered range and commonly used for development.

## Debugging Port Issues

### Check what's using a port

```bash
# macOS/Linux
lsof -i :3000

# Or
netstat -tlnp | grep 3000
```

### View allocated ports

```bash
./bin/yard.ts ps
```

### Check port in context

```bash
cat ledger.d/*/apps/dashboard.db.context.json | jq '.port'
```

## Best Practices

1. **Use high base ports** in production to avoid conflicts
2. **Reserve port ranges** for db-yard in your infrastructure
3. **Monitor port exhaustion** if running many services
4. **Use proxy** for stable external URLs (ports change, proxy paths don't)

---

Next: [Proxy Prefixes](./proxy-prefixes.md) - URL path derivation.