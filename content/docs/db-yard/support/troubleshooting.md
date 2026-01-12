---
title: Troubleshooting
description: Common issues and their solutions.
---

# Troubleshooting

Common issues and their solutions.

## Discovery Issues

### Database Not Found

**Symptom:** Your database isn't showing up in `yard.ts ls`

**Solutions:**

1. Check file extension:
   ```bash
   # Supported extensions
   *.db, *.sqlite, *.sqlite3, *.sqlite.db, *.duckdb
   ```

2. Check location:
   ```bash
   # File must be in cargo directory
   ls cargo.d/your-database.db
   ```

3. Check for hidden directories:
   ```bash
   # Hidden directories are skipped
   cargo.d/.hidden/app.db  # Won't be found
   cargo.d/visible/app.db  # Will be found
   ```

4. Check permissions:
   ```bash
   chmod 644 cargo.d/your-database.db
   ```

### Database Not Classified Correctly

**Symptom:** Database shows as "plain-sqlite" instead of "sqlpage"

**Solutions:**

1. Check for required tables:
   ```bash
   # For SQLPage
   sqlite3 your.db "SELECT name FROM sqlite_master WHERE type='table' AND name='sqlpage_files'"

   # For surveilr
   sqlite3 your.db "SELECT name FROM sqlite_master WHERE type='table' AND name='uniform_resource'"
   ```

2. Verify table isn't empty:
   ```bash
   sqlite3 your.db "SELECT COUNT(*) FROM sqlpage_files"
   ```

## Spawning Issues

### Service Won't Start

**Symptom:** `yard.ts start` fails or service doesn't appear in `ps`

**Solutions:**

1. Check if executable is installed:
   ```bash
   which sqlpage
   which surveilr
   ```

2. Check port availability:
   ```bash
   lsof -i :3000
   netstat -tlnp | grep 3000
   ```

3. Check stderr log:
   ```bash
   cat ledger.d/*/your-app.db.stderr.log
   ```

4. Run with verbose output:
   ```bash
   ./bin/yard.ts start --verbose comprehensive
   ```

### Port Already in Use

**Symptom:** Error about port being in use

**Solutions:**

1. Find what's using the port:
   ```bash
   lsof -i :3000
   ```

2. Use a different base port:
   ```bash
   ./bin/yard.ts start --port-base 9000
   ```

3. Kill existing db-yard processes:
   ```bash
   ./bin/yard.ts kill
   ```

### Process Crashes Immediately

**Symptom:** Service starts but exits immediately

**Solutions:**

1. Check stderr log:
   ```bash
   cat ledger.d/*/app.db.stderr.log
   ```

2. Check database integrity:
   ```bash
   sqlite3 your.db "PRAGMA integrity_check"
   ```

3. Test with executable directly:
   ```bash
   sqlpage --port 3000 --database cargo.d/app.db
   ```

## Proxy Issues

### 502 Bad Gateway

**Symptom:** Proxy returns 502 error

**Solutions:**

1. Check if service is running:
   ```bash
   ./bin/yard.ts ps
   ```

2. Check upstream URL:
   ```bash
   curl http://localhost:3000  # Direct access
   ```

3. Check proxy debug:
   ```bash
   curl "http://localhost:8787/.db-yard/api/proxy-debug.json?path=/apps/sqlpage/my-app/"
   ```

### Wrong Route

**Symptom:** Requests go to wrong service

**Solutions:**

1. Check prefix mapping:
   ```bash
   cat ledger.d/*/app.db.context.json | jq '.proxyPrefix'
   ```

2. Verify directory structure:
   ```bash
   # Path becomes prefix
   cargo.d/a/app.db → /apps/sqlpage/a/app
   ```

3. Enable tracing:
   ```bash
   curl -H "X-Db-Yard-Trace: 1" http://localhost:8787/apps/sqlpage/my-app/
   ```

### Timeout

**Symptom:** Proxy requests timeout

**Solutions:**

1. Check service health:
   ```bash
   curl http://localhost:3000  # Direct access
   ```

2. Check for slow queries:
   ```bash
   tail -f ledger.d/*/app.db.stdout.log
   ```

## Watch Mode Issues

### Changes Not Detected

**Symptom:** New files not spawned automatically

**Solutions:**

1. Verify file is in cargo directory:
   ```bash
   ls -la cargo.d/
   ```

2. Check file extension matches patterns

3. Check if watch mode is running:
   ```bash
   ps aux | grep yard
   ```

### Cleanup on Exit Fails

**Symptom:** Processes remain after Ctrl+C

**Solutions:**

1. Kill manually:
   ```bash
   ./bin/yard.ts kill
   ```

2. Find orphaned processes:
   ```bash
   ps aux | grep sqlpage
   ps aux | grep surveilr
   ```

3. Kill by PID:
   ```bash
   kill <pid>
   ```

## Ledger Issues

### Missing Context Files

**Symptom:** Context JSON files don't exist

**Solutions:**

1. Check ledger directory:
   ```bash
   ls -la ledger.d/
   ```

2. Check permissions:
   ```bash
   chmod 755 ledger.d/
   ```

3. Verify service was spawned:
   ```bash
   ./bin/yard.ts ps
   ```

### Stale Ledger Entries

**Symptom:** Ledger shows services that aren't running

**Solutions:**

1. Run reconciliation:
   ```bash
   ./bin/yard.ts ps --reconcile
   ```

2. Clean and restart:
   ```bash
   ./bin/yard.ts kill --clean
   ./bin/yard.ts start
   ```

## Web UI Issues

### Web UI Won't Start

**Symptom:** `serve.ts` fails to start

**Solutions:**

1. Check port:
   ```bash
   lsof -i :8787
   ```

2. Use different port:
   ```bash
   ./bin/web-ui/serve.ts --port 9000
   ```

### Empty Service List

**Symptom:** Web UI shows no services

**Solutions:**

1. Check if services are running:
   ```bash
   ./bin/yard.ts ps
   ```

2. Check ledger path:
   ```bash
   ./bin/web-ui/serve.ts --ledger-dir ./ledger.d
   ```

## Performance Issues

### Slow Discovery

**Symptom:** `yard.ts start` takes a long time

**Solutions:**

1. Reduce cargo directory scope
2. Remove unnecessary files
3. Use subdirectories for organization

### High Memory Usage

**Symptom:** Services consuming too much memory

**Solutions:**

1. Check individual service memory:
   ```bash
   ps aux | grep sqlpage
   ```

2. Reduce number of concurrent services

3. Check for memory leaks in applications

## Getting More Help

### Enable Verbose Logging

```bash
./bin/yard.ts start --verbose comprehensive
```

### Check Deno Permissions

```bash
deno run --allow-all ./bin/yard.ts start
```

### Debug Mode

```bash
deno run --inspect-brk -A ./bin/yard.ts start
```

### Report an Issue

If you've tried these solutions and still have problems:

1. Search existing issues
2. Open a new issue with:
   - Steps to reproduce
   - Expected behavior
   - Actual behavior
   - Logs and error messages
   - Environment details

---

See also: [FAQ](./faq.md), [Resources](./resources.md)