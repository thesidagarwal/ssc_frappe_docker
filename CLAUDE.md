# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

frappe_docker provides Docker-based container infrastructure for [Frappe Framework](https://github.com/frappe/frappe) and [ERPNext](https://github.com/frappe/erpnext). It includes production deployment configurations, development environments via VSCode Dev Containers, and build tooling for custom Docker images.

## This fork — team-specific layout

This is `thesidagarwal/ssc_frappe_docker`, a fork of upstream `frappe/frappe_docker`. It is used to run the dev container for the private `ssc_custom` Frappe app on top of Frappe + ERPNext **version-16**.

Team-specific additions on top of upstream:

- `devcontainer-ssc/` — committed dev-container preset. Copied to `.devcontainer/` on first setup (`.devcontainer/` itself stays gitignored). Includes a `~/.ssh` bind mount so the private `ssc_custom` repo can be cloned over SSH from inside the container.
- `development/apps-ssc.json` — apps list consumed by `installer.py`: erpnext (version-16, https) + ssc_custom (develop, ssh).
- `SETUP.md` — onboarding guide for new collaborators (links host prereqs, SSH key setup, copy presets, run installer, `bench start`).

Active dev versions: frappe `version-16`, erpnext `version-16`, ssc_custom branch `develop`. QA and prod parallel CI in the `ssc_custom` repo also build against `v16` (see `ssc_custom/docker/Containerfile` and `ssc_custom/.github/workflows/`), but use ssc_custom branch `prod-parallel`, not `develop`.

## Common Commands

### Linting

```bash
# Install pre-commit hooks
pip install pre-commit
pre-commit install

# Run all linters (Black, isort, Prettier, ShellCheck, shfmt, codespell)
pre-commit run --all-files
```

### Building Docker Images

```bash
# Build with Docker Buildx Bake (see docker-bake.hcl for available targets)
FRAPPE_VERSION=v16 ERPNEXT_VERSION=v16 docker buildx bake <target>

# Available targets: erpnext, base, build, bench, bench-test

# ARM64 build
docker buildx bake --no-cache --set "*.platform=linux/arm64"

# Quick build custom image with apps (uses pre-built layers)
docker build \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=custom:1.0.0 \
  --file=images/layered/Containerfile .
```

### Testing

```bash
# Install test dependencies
python3 -m venv venv && source venv/bin/activate
pip install -r requirements-test.txt

# Run integration tests (stops on first failure)
pytest
```

### Running Containers

```bash
# Development with devcontainer
docker compose -f .devcontainer/docker-compose.yml up -d

# Quick start (Play With Docker style)
docker compose -f pwd.yml up -d

# Production setup
docker compose -f compose.yaml up -d
```

## Architecture

### Container Services (compose.yaml)

- **configurator**: Init container that sets up bench config
- **backend**: Gunicorn serving Frappe (port 8000)
- **frontend**: Nginx reverse proxy
- **websocket**: Node.js Socket.IO server (port 9000)
- **queue-short/queue-long**: RQ background workers
- **scheduler**: Background job scheduler

### Docker Images (images/)

| Image | File | Purpose |
|-------|------|---------|
| bench | `images/bench/Dockerfile` | Bench CLI tool |
| production | `images/production/Containerfile` | Base + ERPNext images |
| layered | `images/layered/Containerfile` | Quick custom builds (uses pre-built layers) |
| custom | `images/custom/Containerfile` | Full custom builds (configurable Python/Node) |

### Directory Structure

```
.devcontainer/          # VSCode Dev Container config
development/            # Local development mount (gitignored)
  frappe-bench/         # Bench installation with apps
docs/                   # Setup and operations documentation
images/                 # Dockerfiles/Containerfiles
overrides/              # Compose override files for different setups
tests/                  # pytest integration tests
```

### Compose Override Files (overrides/)

Combine with compose.yaml for specific setups:
- `compose.mariadb.yaml` / `compose.postgres.yaml`: Database services
- `compose.redis.yaml`: Redis services
- `compose.https.yaml` / `compose.traefik-ssl.yaml`: TLS termination
- `compose.multi-bench.yaml`: Multiple bench instances

## Development Workflow

### VSCode Dev Container Setup

For this fork, use the team preset (not the upstream example):

1. Copy `devcontainer-ssc` to `.devcontainer`
2. Copy `development/vscode-example` to `development/.vscode`
3. Open in VSCode and "Reopen in Container"

See `SETUP.md` for the full guided walkthrough including host prereqs and SSH key setup.

### Inside Dev Container

```bash
# Initialize bench
bench init --skip-redis-config-generation frappe-bench
cd frappe-bench

# Configure external services
bench set-config -g db_host mariadb
bench set-config -g redis_cache redis://redis-cache:6379
bench set-config -g redis_queue redis://redis-queue:6379
bench set-config -g redis_socketio redis://redis-queue:6379

# Create site (must end with .localhost)
bench new-site --mariadb-user-host-login-scope=% --db-root-password 123 --admin-password admin dev.localhost

# Install apps
bench get-app --branch version-16 erpnext
bench --site dev.localhost install-app erpnext

# Install ssc_custom (private repo — requires SSH key reachable in container)
bench get-app --branch develop git@github.com:thesidagarwal/ssc_custom.git
bench --site dev.localhost install-app ssc_custom

# Start development server
bench start
```

### Automated Setup

```bash
# Use installer script (inside dev container) with the team apps list and v16 frappe.
# -t version-16 overrides the installer's default (version-15).
python installer.py -j apps-ssc.json -t version-16
```

## Key Configuration

| File | Purpose |
|------|---------|
| `docker-bake.hcl` | Buildx Bake targets and variables |
| `compose.yaml` | Production compose template |
| `example.env` | Environment variable reference |
| `setup.cfg` | pytest, isort, codespell config |

### Build Variables (docker-bake.hcl)

- `PYTHON_VERSION`: Default 3.11.6
- `NODE_VERSION`: Default 18.18.2
- `FRAPPE_VERSION` / `ERPNEXT_VERSION`: Branch or tag to build
