---
title: Proxy Prefixes
description: How db-yard derives URL paths for services.
---

# Proxy Prefixes

How db-yard derives URL paths for services.

## Overview

Each exposable service is assigned a proxy prefix - a URL path that uniquely identifies the service. This prefix is derived deterministically from the file's location in the cargo directory.

## Prefix Derivation

The proxy prefix combines:
1. Service type prefix (`/apps/sqlpage/` or `/apps/surveilr/`)
2. Relative directory path
3. Normalized filename (without database extension)

### Formula

```
/apps/{kind}/{relative-dir}/{normalized-name}
```

### Examples

| Cargo Path | Kind | Proxy Prefix |
|------------|------|--------------|
| `cargo.d/controls/scf-2025.sqlite.db` | sqlpage | `/apps/sqlpage/controls/scf-2025` |
| `cargo.d/two/example-two.db` | sqlpage | `/apps/sqlpage/two/example-two` |
| `cargo.d/evidence/audit.sqlite.db` | surveilr | `/apps/surveilr/evidence/audit` |
| `cargo.d/deep/nested/path/app.db` | sqlpage | `/apps/sqlpage/deep/nested/path/app` |

## Filename Normalization

Database extensions are stripped from filenames:

| Original | Normalized |
|----------|------------|
| `my-app.db` | `my-app` |
| `my-app.sqlite` | `my-app` |
| `my-app.sqlite.db` | `my-app` |
| `my-app.sqlite3` | `my-app` |

Compound extensions like `.sqlite.db` are fully removed.

## Using Prefixes

### Direct Access

Services are accessible directly via their port:

```
http://localhost:3000/
```

### Proxied Access

Via the db-yard proxy (web UI), services are accessible via prefix:

```
http://localhost:8787/apps/sqlpage/controls/scf-2025/
```

### Prefix Benefits

1. **Stable URLs** - Port may change, prefix stays the same
2. **Self-documenting** - Path indicates source location
3. **Multiplexing** - Many services through one port (8787)
4. **Reverse proxy ready** - NGINX/Traefik can route by path

## Prefix in Context JSON

The proxy prefix is recorded in each service's context file:

```json
{
  "serviceId": "scf-2025.sqlite.db",
  "proxyPrefix": "/apps/sqlpage/controls/scf-2025",
  "port": 3000,
  ...
}
```

## Prefix Routing

The db-yard proxy routes requests by matching prefixes:

```
Request: GET /apps/sqlpage/controls/scf-2025/index.sql
         ↓
Match:   /apps/sqlpage/controls/scf-2025 → port 3000
         ↓
Forward: GET http://localhost:3000/index.sql
```

The prefix is stripped when forwarding to the upstream service.

## Generating Proxy Configuration

Use the `proxy-conf` command to generate configuration for production proxies:

```bash
# NGINX configuration
./bin/yard.ts proxy-conf --type nginx

# Traefik configuration
./bin/yard.ts proxy-conf --type traefik
```

Example NGINX output:

```nginx
location /apps/sqlpage/controls/scf-2025/ {
    proxy_pass http://127.0.0.1:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

## Prefix Conflicts

db-yard ensures prefixes are unique:

- Different paths → different prefixes (guaranteed)
- Same filename in different directories → different prefixes

```
cargo.d/a/app.db → /apps/sqlpage/a/app
cargo.d/b/app.db → /apps/sqlpage/b/app
```

## Debugging Prefixes

### View in ps output

```bash
./bin/yard.ts ps
```

Shows proxy prefix for each service.

### Check context file

```bash
cat ledger.d/*/controls/scf-2025.sqlite.db.context.json | jq '.proxyPrefix'
```

### Test proxy routing

```bash
# Start web UI
./bin/web-ui/serve.ts

# Test route
curl http://localhost:8787/apps/sqlpage/controls/scf-2025/
```

## Best Practices

1. **Organize cargo by purpose** - Directory structure becomes URL structure
2. **Use meaningful names** - Filenames become URL segments
3. **Avoid deep nesting** - Keeps URLs manageable
4. **Consider URL design** - End users may see these paths

---

See also: [Reverse Proxy Configuration](../advanced/reverse-proxy-conf.md)