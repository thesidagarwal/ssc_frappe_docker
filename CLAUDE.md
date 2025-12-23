# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **frappe_docker** - a containerized setup for [Frappe Framework](https://github.com/frappe/frappe) and [ERPNext](https://github.com/frappe/erpnext). It provides Docker-based production deployments and development environments for Frappe applications.

## Repository Structure

- `images/` - Docker image definitions
  - `production/` - Production-ready images (Frappe + ERPNext)
  - `custom/` - Custom image builds using apps.json
  - `layered/` - Quick builds using pre-built base layers
  - `bench/` - Development bench image
- `overrides/` - Docker Compose override files for different configurations
- `docs/` - Documentation for setup, deployment, and operations
- `development/` - Development environment setup (has its own CLAUDE.md)
- `tests/` - pytest-based integration tests
- `.devcontainer/` - VSCode Dev Container configuration

## Technology Stack

- **Containers**: Docker, Docker Compose v2, Docker Buildx
- **Web Server**: nginx (frontend proxy and static assets)
- **Application Server**: Gunicorn + Werkzeug (Python)
- **Real-time**: Socket.IO via Node.js
- **Database**: MariaDB 10.6+ or PostgreSQL (Frappe only)
- **Cache**: Redis (separate instances for cache and queue)
- **Task Queue**: Python RQ with worker processes
- **Scheduler**: Python schedule
- **Proxy**: Traefik (optional, for HTTPS/Let's Encrypt)

## Common Commands

### Quick Start (Play With Docker / Testing)

```bash
# Start complete ERPNext stack with database and Redis
docker compose -f pwd.yml up -d

# Wait ~5 minutes for site creation, then access at localhost:8080
# Username: Administrator, Password: admin
```

### Production Setup

```bash
# 1. Copy and configure environment variables
cp example.env .env
# Edit .env: set ERPNEXT_VERSION, DB_PASSWORD, LETSENCRYPT_EMAIL, SITES

# 2. Generate compose configuration
docker compose -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.https.yaml \
  config > ~/gitops/docker-compose.yml

# 3. Start containers
docker compose --project-name <project-name> -f ~/gitops/docker-compose.yml up -d

# 4. Create first site
docker compose --project-name <project-name> exec backend \
  bench new-site --mariadb-user-host-login-scope=% \
  --db-root-password <db-password> \
  --admin-password <admin-password> \
  mysite.example.com
```

### Site Operations

```bash
# All commands run in backend container
docker compose exec backend <command>

# Create new site
bench new-site --mariadb-user-host-login-scope=% \
  --db-root-password 123 --admin-password admin sitename

# Migrate site after updates
bench --site sitename migrate

# Backup site
bench --site sitename backup

# Restore backup
bench --site sitename restore /path/to/backup

# Enable/disable maintenance mode
bench --site sitename set-maintenance-mode on

# Install app on site
bench --site sitename install-app app_name
```

### Building Custom Images

```bash
# 1. Create apps.json with your apps
cat > apps.json << 'EOF'
[
  {
    "url": "https://github.com/frappe/erpnext",
    "branch": "version-15"
  },
  {
    "url": "https://github.com/user/custom_app",
    "branch": "main"
  }
]
EOF

# 2. Encode to base64
export APPS_JSON_BASE64=$(base64 -w 0 apps.json)

# 3. Build using layered method (faster, uses pre-built base)
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=ghcr.io/user/repo/custom:1.0.0 \
  --file=images/layered/Containerfile .

# 4. Build using custom method (slower, full control over Python/Node versions)
docker build \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=PYTHON_VERSION=3.11.9 \
  --build-arg=NODE_VERSION=20.19.2 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=ghcr.io/user/repo/custom:1.0.0 \
  --file=images/custom/Containerfile .

# 5. Push to registry
docker push ghcr.io/user/repo/custom:1.0.0

# 6. Use in compose by setting environment variables
export CUSTOM_IMAGE='ghcr.io/user/repo/custom'
export CUSTOM_TAG='1.0.0'
export PULL_POLICY='never'  # For local images
```

### Building Production Images (Maintainers)

```bash
# Build using docker-bake.hcl
FRAPPE_VERSION=version-15 ERPNEXT_VERSION=v15.70.1 \
  docker buildx bake

# Available targets: erpnext, base, build, bench
# See docker-bake.hcl for configuration
```

### Testing and Linting

```bash
# Install pre-commit hooks
pre-commit install

# Run linters on all files
pre-commit run --all-files

# Install test dependencies
python3 -m venv venv
source venv/bin/activate
pip install -r requirements-test.txt

# Run integration tests
pytest
```

## Docker Compose Override System

The repository uses a modular compose file system. Start with `compose.yaml` (base) and add overrides:

### Available Overrides

- `compose.mariadb.yaml` - Add MariaDB database service
- `compose.postgres.yaml` - Add PostgreSQL database service (Frappe only, not ERPNext)
- `compose.redis.yaml` - Add Redis cache and queue services
- `compose.proxy.yaml` - Add Traefik reverse proxy
- `compose.noproxy.yaml` - Publish frontend directly without proxy
- `compose.https.yaml` - Enable Let's Encrypt SSL certificates via Traefik
- `compose.backup-cron.yaml` - Add automated backup cron job
- `compose.multi-bench.yaml` - Multi-bench setup on same server
- `compose.multi-bench-ssl.yaml` - Multi-bench with SSL

### Common Combinations

```bash
# Local development (no SSL, containerized DB/Redis)
docker compose -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  up -d

# Production with SSL (Let's Encrypt)
docker compose -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.https.yaml \
  config > ~/gitops/docker-compose.yml

# Production with external DB/Redis
docker compose -f compose.yaml \
  -f overrides/compose.proxy.yaml \
  config > ~/gitops/docker-compose.yml
# Make sure DB_HOST, DB_PORT, REDIS_CACHE, REDIS_QUEUE are set in .env
```

## Service Architecture

The main services defined in `compose.yaml`:

- **configurator** - Runs once on startup to configure `common_site_config.json` with DB/Redis connection details
- **backend** - Gunicorn/Werkzeug Python application server (port 8000)
- **frontend** - nginx server for static assets and request routing (port 8080 default)
- **websocket** - Node.js Socket.IO server for real-time features (port 9000)
- **queue-short** - RQ worker for short-running tasks (default, short queues)
- **queue-long** - RQ worker for long-running tasks (long, default, short queues)
- **scheduler** - Runs scheduled tasks via `bench schedule`

Optional services (added via overrides):

- **db** - MariaDB or PostgreSQL database
- **redis-cache** / **redis-queue** - Redis instances
- **proxy** - Traefik reverse proxy with optional SSL

## Environment Variables

Key variables in `.env` (see `example.env` and `docs/environment-variables.md`):

```bash
# Version control
ERPNEXT_VERSION=v15.70.1          # ERPNext version tag
CUSTOM_IMAGE=frappe/erpnext       # Custom image name
CUSTOM_TAG=${ERPNEXT_VERSION}     # Custom image tag
PULL_POLICY=always                # always|never|missing

# Database (if using external)
DB_HOST=
DB_PORT=
DB_PASSWORD=123

# Redis (if using external)
REDIS_CACHE=
REDIS_QUEUE=

# HTTPS configuration
LETSENCRYPT_EMAIL=mail@example.com
SITES=`site1.example.com`,`site2.example.com`

# Networking
HTTP_PUBLISH_PORT=8080
FRAPPE_SITE_NAME_HEADER=$$host    # Or set to specific site name

# Nginx tuning
PROXY_READ_TIMEOUT=120
CLIENT_MAX_BODY_SIZE=50m
UPSTREAM_REAL_IP_ADDRESS=127.0.0.1
UPSTREAM_REAL_IP_HEADER=X-Forwarded-For
UPSTREAM_REAL_IP_RECURSIVE=off
```

## Development Workflow

For development, use the devcontainer setup:

```bash
# Copy devcontainer configuration
cp -R devcontainer-example .devcontainer
cp -R development/vscode-example development/.vscode

# Open in VSCode and reopen in container
code .
# Command Palette: "Dev Containers: Reopen in Container"

# Inside container, create bench
bench init --skip-redis-config-generation frappe-bench
cd frappe-bench

# Configure for containerized services
bench set-config -g db_host mariadb
bench set-config -g redis_cache redis://redis-cache:6379
bench set-config -g redis_queue redis://redis-queue:6379
bench set-config -g redis_socketio redis://redis-queue:6379

# Create site
bench new-site --mariadb-user-host-login-scope=% development.localhost

# Get and install apps
bench get-app --branch version-15 erpnext
bench --site development.localhost install-app erpnext

# Start development server
bench start
```

**Note:** For detailed development instructions, see `docs/development.md` or `development/CLAUDE.md`.

## Build Arguments

### Custom/Layered Images

- `FRAPPE_PATH` - Frappe framework git URL (default: https://github.com/frappe/frappe)
- `FRAPPE_BRANCH` - Frappe branch (default: version-15)
- `APPS_JSON_BASE64` - Base64-encoded apps.json file
- `PYTHON_VERSION` - Python version (custom only, default: 3.11.6)
- `NODE_VERSION` - Node.js version (custom only, default: 20.19.2)
- `DEBIAN_BASE` - Debian version (custom only, default: bookworm)
- `WKHTMLTOPDF_VERSION` - wkhtmltopdf version (custom only, default: 0.12.6.1-3)

### Production Images (docker-bake.hcl)

- `FRAPPE_VERSION` - Frappe framework version
- `ERPNEXT_VERSION` - ERPNext version
- `PYTHON_VERSION` - Python version
- `NODE_VERSION` - Node.js version
- `REGISTRY_USER` - Docker registry username (default: frappe)

## Key Files

- `compose.yaml` - Base Docker Compose configuration with all core services
- `pwd.yml` - Play With Docker quick-start compose file (includes DB, Redis, site creation)
- `example.env` - Environment variable template
- `docker-bake.hcl` - Docker Buildx Bake configuration for building production images
- `images/production/Containerfile` - Multi-stage production image build
- `images/custom/Containerfile` - Custom image with full build control
- `images/layered/Containerfile` - Quick custom image using pre-built layers
- `images/bench/Dockerfile` - Development bench container

## Testing

Tests use pytest and require running containers:

```bash
# Start test environment
docker compose -f tests/compose.ci.yaml up -d

# Run tests
pytest tests/

# Key test files:
# - tests/test_frappe_docker.py - Main integration tests
# - tests/conftest.py - Pytest fixtures and configuration
```

## Git Workflow

- **Main branch**: `main` - stable releases
- **Development branch**: Contribute via PRs to `main`
- Pre-commit hooks enforce linting (shellcheck, yamllint, etc.)

## ARM64 Support

```bash
# Build multi-architecture images for ARM64
docker buildx bake --no-cache --set "*.platform=linux/arm64"

# In compose files, add:
# platform: linux/arm64
# And use :latest tags instead of specific versions
```

## Maintenance Notes

**On Debian version updates** (e.g., bullseye → bookworm):
- Update `images/production/Containerfile`, `images/custom/Containerfile`
- Update Python version, apt packages, wkhtmltopdf version
- Update `images/bench/Dockerfile` similarly

**On ERPNext version releases**:
- Update `.github/workflows/build_stable.yml` to add new version, remove EOL version
- Ensure helm charts are bumped

## Additional Documentation

- [docs/development.md](docs/development.md) - Complete development setup guide
- [docs/custom-apps.md](docs/custom-apps.md) - Building images with custom apps
- [docs/site-operations.md](docs/site-operations.md) - Site management commands
- [docs/environment-variables.md](docs/environment-variables.md) - All available environment variables
- [docs/setup-options.md](docs/setup-options.md) - Production deployment options
- [docs/troubleshoot.md](docs/troubleshoot.md) - Common issues and solutions
