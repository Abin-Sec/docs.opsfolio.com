---
title: Production Deployment
description: Deploying db-yard to production environments.
---

# Production Deployment

Deploying db-yard to production environments.

## Overview

This guide covers deploying db-yard services to production with proper security, monitoring, and reliability considerations.

## Architecture

### Production Stack

```
                    Internet
                       |
                       v
              +─────────────────+
              |  Load Balancer  |
              |  (TLS, Auth)    |
              +────────+────────+
                       |
              +────────v────────+
              |  Reverse Proxy  |
              |  (NGINX/Traefik)|
              +────────+────────+
                       |
        +──────────────+──────────────+
        |              |              |
        v              v              v
   +─────────+   +─────────+   +─────────+
   | Service |   | Service |   | Service |
   | :3000   |   | :3001   |   | :3002   |
   +─────────+   +─────────+   +─────────+
        |              |              |
        v              v              v
   +─────────────────────────────────────+
   |           db-yard Manager           |
   |         (watch mode)                |
   +─────────────────────────────────────+
```

## Deployment Steps

### Step 1: Prepare the Server

```bash
# Create directories
sudo mkdir -p /var/db-yard/{cargo,ledger}
sudo chown -R dbyard:dbyard /var/db-yard

# Install Deno
curl -fsSL https://deno.land/install.sh | sh

# Install dependencies
# SQLPage, surveilr, etc.
```

### Step 2: Deploy db-yard

```bash
# Clone repository
git clone https://github.com/netspective-labs/db-yard.git /opt/db-yard
cd /opt/db-yard

# Verify installation
./bin/yard.ts --version
```

### Step 3: Configure Environment

```bash
# /etc/db-yard/env
export DB_YARD_CARGO_HOME=/var/db-yard/cargo
export DB_YARD_LEDGER_HOME=/var/db-yard/ledger
export DB_YARD_PORT_BASE=9000
export DB_YARD_WEB_HOST=127.0.0.1
export DB_YARD_WEB_PORT=8787
```

### Step 4: Create Systemd Service

```ini
# /etc/systemd/system/db-yard.service
[Unit]
Description=db-yard Process Manager
After=network.target

[Service]
Type=simple
User=dbyard
Group=dbyard
EnvironmentFile=/etc/db-yard/env
WorkingDirectory=/opt/db-yard
ExecStart=/opt/db-yard/bin/yard.ts start --watch
ExecStop=/bin/kill -SIGINT $MAINPID
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/db-yard-web.service
[Unit]
Description=db-yard Web UI
After=db-yard.service

[Service]
Type=simple
User=dbyard
Group=dbyard
EnvironmentFile=/etc/db-yard/env
WorkingDirectory=/opt/db-yard
ExecStart=/opt/db-yard/bin/web-ui/serve.ts
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Step 5: Enable and Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable db-yard db-yard-web
sudo systemctl start db-yard db-yard-web
```

## Reverse Proxy Configuration

### Generate Configuration

```bash
./bin/yard.ts proxy-conf --type nginx > /etc/nginx/conf.d/db-yard.conf
```

### NGINX Configuration

```nginx
# /etc/nginx/conf.d/db-yard.conf

upstream dbyard_web {
    server 127.0.0.1:8787;
}

server {
    listen 443 ssl http2;
    server_name dbyard.example.com;

    ssl_certificate /etc/ssl/certs/dbyard.crt;
    ssl_certificate_key /etc/ssl/private/dbyard.key;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    # Authentication
    auth_basic "db-yard";
    auth_basic_user_file /etc/nginx/.htpasswd;

    # Proxy to db-yard
    location / {
        proxy_pass http://dbyard_web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Include generated service routes
    include /etc/nginx/conf.d/db-yard-services.conf;
}
```

### Auto-Update Configuration

```bash
#!/bin/bash
# /opt/db-yard/scripts/update-proxy.sh

./bin/yard.ts proxy-conf --type nginx > /etc/nginx/conf.d/db-yard-services.conf
nginx -t && nginx -s reload
```

Add to cron or trigger on ledger changes.

## Security

### Network Security

```bash
# Bind services to localhost only
export DB_YARD_WEB_HOST=127.0.0.1

# Firewall rules
sudo ufw allow 443/tcp  # HTTPS only
sudo ufw deny 3000:4000/tcp  # Block direct service access
```

### File Permissions

```bash
# Restrict cargo directory
chmod 750 /var/db-yard/cargo
chmod 640 /var/db-yard/cargo/*.db

# Restrict ledger
chmod 750 /var/db-yard/ledger
```

### Authentication

Use reverse proxy authentication:

```bash
# Create htpasswd file
htpasswd -c /etc/nginx/.htpasswd admin
```

Or integrate with SSO/OAuth.

## Monitoring

### Health Checks

```bash
#!/bin/bash
# /opt/db-yard/scripts/health-check.sh

# Check db-yard service
if ! systemctl is-active --quiet db-yard; then
    echo "db-yard service is down"
    exit 1
fi

# Check web UI
if ! curl -sf http://127.0.0.1:8787/.db-yard/api/services.json > /dev/null; then
    echo "Web UI is not responding"
    exit 1
fi

# Check service count
count=$(curl -sf http://127.0.0.1:8787/.db-yard/api/services.json | jq length)
if [ "$count" -eq 0 ]; then
    echo "No services running"
    exit 1
fi

echo "OK: $count services running"
```

### Log Aggregation

```bash
# Forward logs to syslog
logger -t db-yard -f /var/db-yard/ledger/watch/**/*.log
```

Or use journald:

```bash
journalctl -u db-yard -f
```

### Metrics

Export metrics from the API:

```bash
# Prometheus endpoint (custom implementation)
curl http://127.0.0.1:8787/.db-yard/api/metrics
```

## Backup and Recovery

### Backup Script

```bash
#!/bin/bash
# /opt/db-yard/scripts/backup.sh

BACKUP_DIR=/var/backups/db-yard
DATE=$(date +%Y%m%d)

# Backup cargo databases
tar -czf $BACKUP_DIR/cargo-$DATE.tar.gz /var/db-yard/cargo

# Backup ledger (optional, can be regenerated)
tar -czf $BACKUP_DIR/ledger-$DATE.tar.gz /var/db-yard/ledger

# Cleanup old backups
find $BACKUP_DIR -mtime +30 -delete
```

### Recovery

```bash
# Stop services
sudo systemctl stop db-yard db-yard-web

# Restore cargo
tar -xzf /var/backups/db-yard/cargo-20260112.tar.gz -C /

# Restart
sudo systemctl start db-yard db-yard-web
```

## Scaling

### Multiple Instances

For high availability:

```
+─────────────────+
|  Load Balancer  |
+────────+────────+
         |
    +────+────+
    |         |
    v         v
+───────+ +───────+
| Node1 | | Node2 |
+───────+ +───────+
```

Each node has its own:
- Cargo directory (synced via rsync/NFS)
- Ledger directory
- Service ports

### Shared Storage

```bash
# Mount shared cargo
mount -t nfs nas:/db-yard/cargo /var/db-yard/cargo
```

## Updating

### Rolling Update

```bash
# Pull latest
cd /opt/db-yard
git pull

# Restart gracefully
sudo systemctl restart db-yard

# Regenerate proxy config
./bin/yard.ts proxy-conf --type nginx > /etc/nginx/conf.d/db-yard-services.conf
nginx -s reload
```

### Database Updates

```bash
# Add new database
cp new-app.sqlite.db /var/db-yard/cargo/

# Watch mode automatically picks it up
```

## Troubleshooting

### Service Won't Start

```bash
# Check logs
journalctl -u db-yard -n 50

# Check permissions
ls -la /var/db-yard/
```

### Port Conflicts

```bash
# Find what's using ports
netstat -tlnp | grep 9000
```

### Memory Issues

```bash
# Monitor memory
ps aux | grep sqlpage

# Limit memory per service (via systemd)
MemoryMax=512M
```

---

See also: [Configuration Reference](../reference/configuration.md), [Reverse Proxy Configuration](../advanced/reverse-proxy-conf.md)