---
title: Reverse Proxy Configuration
description: Generating NGINX and Traefik configuration from db-yard state.
---

# Reverse Proxy Configuration

Generating NGINX and Traefik configuration from db-yard state.

## Overview

db-yard includes a built-in transparent proxy for development, but production deployments should use industrial-grade reverse proxies. The `proxy-conf` command generates configuration from ledger state.

## When to Use

| Scenario | Recommended |
|----------|-------------|
| Local development | Built-in proxy |
| Testing/debugging | Built-in proxy |
| Production | NGINX/Traefik |
| High traffic | NGINX/Traefik |
| TLS termination | NGINX/Traefik |
| Authentication | NGINX/Traefik |

## Generating Configuration

### NGINX

```bash
./bin/yard.ts proxy-conf --type nginx
```

### Traefik

```bash
./bin/yard.ts proxy-conf --type traefik
```

### Both

```bash
./bin/yard.ts proxy-conf --type both
```

## NGINX Configuration

### Generated Output

```nginx
# db-yard generated configuration
# Generated: 2026-01-12T10:30:00.000Z

upstream dbyard_dashboard {
    server 127.0.0.1:3000;
}

upstream dbyard_metrics {
    server 127.0.0.1:3001;
}

server {
    listen 80;
    server_name _;

    # Dashboard (SQLPage)
    location /apps/sqlpage/apps/dashboard/ {
        proxy_pass http://dbyard_dashboard/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
    }

    # Metrics (SQLPage)
    location /apps/sqlpage/apps/metrics/ {
        proxy_pass http://dbyard_metrics/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
    }
}
```

### Using the Configuration

1. Generate config:
   ```bash
   ./bin/yard.ts proxy-conf --type nginx > /etc/nginx/conf.d/db-yard.conf
   ```

2. Test config:
   ```bash
   nginx -t
   ```

3. Reload NGINX:
   ```bash
   nginx -s reload
   ```

### NGINX Best Practices

Add these settings for production:

```nginx
# Timeouts
proxy_connect_timeout 60s;
proxy_read_timeout 300s;
proxy_send_timeout 300s;

# Buffering
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 4k;

# WebSocket support (if needed)
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

## Traefik Configuration

### Generated Output (YAML)

```yaml
# db-yard generated configuration
# Generated: 2026-01-12T10:30:00.000Z

http:
  routers:
    dbyard-dashboard:
      rule: "PathPrefix(`/apps/sqlpage/apps/dashboard`)"
      service: dbyard-dashboard
      middlewares:
        - dbyard-dashboard-strip

    dbyard-metrics:
      rule: "PathPrefix(`/apps/sqlpage/apps/metrics`)"
      service: dbyard-metrics
      middlewares:
        - dbyard-metrics-strip

  middlewares:
    dbyard-dashboard-strip:
      stripPrefix:
        prefixes:
          - "/apps/sqlpage/apps/dashboard"

    dbyard-metrics-strip:
      stripPrefix:
        prefixes:
          - "/apps/sqlpage/apps/metrics"

  services:
    dbyard-dashboard:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:3000"

    dbyard-metrics:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:3001"
```

### Using the Configuration

1. Generate config:
   ```bash
   ./bin/yard.ts proxy-conf --type traefik > /etc/traefik/dynamic/db-yard.yml
   ```

2. Traefik will auto-reload if file watching is enabled.

### Traefik Best Practices

Add health checks:

```yaml
services:
  dbyard-dashboard:
    loadBalancer:
      healthCheck:
        path: /
        interval: 10s
        timeout: 3s
      servers:
        - url: "http://127.0.0.1:3000"
```

## Command Options

### --ledger-home

Specify ledger location:

```bash
./bin/yard.ts proxy-conf --type nginx --ledger-home /var/db-yard/ledger
```

### --prefix

Override base URL prefix:

```bash
./bin/yard.ts proxy-conf --type nginx --prefix /databases
```

Generates:
```nginx
location /databases/sqlpage/apps/dashboard/ {
    ...
}
```

### --strip-prefix

Control prefix stripping:

```bash
./bin/yard.ts proxy-conf --type nginx --strip-prefix
```

## Automation

### Watch and Regenerate

Automate config updates when ledger changes:

```bash
#!/bin/bash
# watch-and-generate.sh

while true; do
  inotifywait -r -e modify,create,delete ledger.d/
  ./bin/yard.ts proxy-conf --type nginx > /etc/nginx/conf.d/db-yard.conf
  nginx -s reload
done
```

### CI/CD Integration

Generate config as part of deployment:

```yaml
# .github/workflows/deploy.yml
- name: Generate proxy config
  run: |
    ./bin/yard.ts start
    ./bin/yard.ts proxy-conf --type nginx > nginx.conf

- name: Deploy config
  run: |
    scp nginx.conf server:/etc/nginx/conf.d/db-yard.conf
    ssh server nginx -s reload
```

## Comparison: Built-in vs External

| Feature | Built-in Proxy | NGINX/Traefik |
|---------|----------------|---------------|
| Setup | Zero config | Requires config |
| Performance | Development | Production |
| TLS | No | Yes |
| Auth | No | Yes |
| Load balancing | No | Yes |
| Rate limiting | No | Yes |
| Caching | No | Yes |
| Debugging | Excellent | Standard |

## Common Patterns

### TLS Termination

```nginx
server {
    listen 443 ssl;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location /apps/ {
        proxy_pass http://127.0.0.1:8787;
    }
}
```

### Basic Authentication

```nginx
location /apps/ {
    auth_basic "db-yard";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:8787;
}
```

### Rate Limiting

```nginx
limit_req_zone $binary_remote_addr zone=dbyard:10m rate=10r/s;

location /apps/ {
    limit_req zone=dbyard burst=20 nodelay;
    proxy_pass http://127.0.0.1:8787;
}
```

## Troubleshooting

### 502 Bad Gateway

Service not running:
```bash
./bin/yard.ts ps  # Check if service is running
```

### Wrong Upstream

Regenerate config after changes:
```bash
./bin/yard.ts proxy-conf --type nginx > config.conf
```

### Path Issues

Check prefix stripping:
```bash
# Debug with curl
curl -v http://localhost/apps/sqlpage/my-app/
```

---

See also: [Web UI Reference](../reference/web-ui.md), [Production Deployment](../how-to-guides/production-deployment.md)