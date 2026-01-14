# Dev2 Environment Setup Guide

Complete setup for the second development environment with ssc_custom app.

## Prerequisites

1. Open this directory in VSCode: `C:\Users\Sid\projects\ssc_frappe_docker_dev2`
2. Reopen in container: **F1** → "Dev Containers: Reopen in Container"
3. Wait for container to start

## Step 1: Initialize Bench

```bash
# Inside the devcontainer terminal
cd /workspace/development

# Initialize bench (Frappe v15 by default)
bench init --skip-redis-config-generation frappe-bench

cd frappe-bench
```

## Step 2: Configure Services

```bash
# Point to containerized MariaDB and Redis
bench set-config -g db_host mariadb
bench set-config -g redis_cache redis://redis-cache:6379
bench set-config -g redis_queue redis://redis-queue:6379
bench set-config -g redis_socketio redis://redis-queue:6379
```

## Step 3: Create Site

```bash
# Create site with admin credentials
bench new-site dev2.localhost \
  --mariadb-user-host-login-scope=% \
  --db-root-password 123 \
  --admin-password admin

# Enable developer mode
bench --site dev2.localhost set-config developer_mode 1
bench --site dev2.localhost clear-cache
```

## Step 4: Install ERPNext

```bash
# Get ERPNext (version 15)
bench get-app --branch version-15 erpnext

# Install on site (this will take a few minutes)
bench --site dev2.localhost install-app erpnext
```

## Step 5: Get ssc_custom App

You have two options:

### Option A: From Local Directory (if you have it locally)

```bash
# If ssc_custom is in the original dev environment
# Copy from the other bench
cp -r /path/to/original/frappe-bench/apps/ssc_custom /workspace/development/frappe-bench/apps/

# Or symlink (better for keeping in sync)
ln -s /workspace/development/frappe-bench-original/apps/ssc_custom /workspace/development/frappe-bench/apps/ssc_custom

# Install the app
bench --site dev2.localhost install-app ssc_custom
```

### Option B: From Git Repository

```bash
# Clone from your git repo
bench get-app https://github.com/[your-org]/ssc_custom --branch develop

# Install on site
bench --site dev2.localhost install-app ssc_custom
```

## Step 6: Verify Installation

```bash
# Check installed apps
bench --site dev2.localhost list-apps

# Should show:
# frappe
# erpnext
# ssc_custom

# Check developer mode
bench --site dev2.localhost get-config developer_mode
# Should output: 1
```

## Step 7: Apply Fixtures (Automatic)

When you installed ssc_custom, all fixtures were automatically applied!

To verify:
```bash
# Check custom fields were added
bench --site dev2.localhost console

# In the console:
frappe.get_all("Custom Field", filters={"module": "SSC Custom"}, fields=["dt", "fieldname"])
# Press Ctrl+D to exit
```

## Step 8: Start Development Server

```bash
# Start all services
bench start

# Your site will be available at:
# http://localhost:8010
# (Note: Port 8010, not 8000, because of dev2 port mapping)

# Login with:
# Username: Administrator
# Password: admin
```

## Alternative: Start with Debugger

For Python debugging in VSCode:

```bash
# Start services except web (web will be started by VSCode debugger)
honcho start socketio watch schedule worker_short worker_long

# Then in VSCode, go to Run & Debug and start "Python: Frappe Web"
```

## Accessing Both Environments Simultaneously

You can now run both dev environments at the same time:

| Environment | Web URL | MariaDB Port | Branch |
|-------------|---------|--------------|--------|
| Original | http://localhost:8000 | 3306 | dev-airline-tutorial-agsidd |
| Dev2 | http://localhost:8010 | 3307 | main |

## Common Commands for Dev2

```bash
# Clear cache
bench --site dev2.localhost clear-cache

# Migrate after pulling updates
bench --site dev2.localhost migrate

# Export fixtures (after making customizations)
bench --site dev2.localhost export-fixtures

# Run tests
bench --site dev2.localhost run-tests --app ssc_custom

# Install pre-commit hooks in ssc_custom
cd apps/ssc_custom
pre-commit install

# Run linters
pre-commit run --all-files
```

## Syncing with Original Environment

If you want to copy data from your original dev site to dev2:

```bash
# In original environment, backup site
bench --site development.localhost backup --with-files

# Copy backup to dev2 (find latest backup)
ls -lt sites/development.localhost/private/backups/

# In dev2, restore
bench --site dev2.localhost restore /path/to/backup.sql.gz
```

## Troubleshooting

### Port conflicts
- Make sure original dev environment is stopped, or
- Check that dev2 uses ports 8010, 9010 (already configured)

### Fixtures not applying
```bash
# Manually import fixtures
bench --site dev2.localhost migrate
bench --site dev2.localhost install-app ssc_custom --force
```

### Permission errors
```bash
# Clear cache and rebuild
bench --site dev2.localhost clear-cache
bench build --app ssc_custom
```

### Database connection issues
```bash
# Check MariaDB is running
docker ps | grep mariadb

# Test connection
bench --site dev2.localhost console
# frappe.db.sql("SELECT 1")
```

## Next Steps

1. Make your customizations in dev2
2. Test thoroughly
3. Export fixtures: `bench --site dev2.localhost export-fixtures`
4. Commit changes to git
5. Pull into original environment and migrate
