# Frappe / ERPNext / HRMS on Coolify

> One-click Docker Compose deployment for [Frappe](https://frappeframework.com/) apps (ERPNext, HRMS, etc.) on [Coolify](https://coolify.io/) — a self-hosted PaaS.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Frappe Version](https://img.shields.io/badge/Frappe-v15-blue)](https://github.com/frappe/frappe)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)](https://docs.docker.com/compose/)
[![Coolify](https://img.shields.io/badge/Coolify-Self--hosted-6C47FF)](https://coolify.io/)

---

## What is this?

This repository provides a `docker-compose.yaml` designed to deploy **Frappe Framework** apps (ERPNext, HRMS, etc.) via **Coolify** using Docker Swarm mode replica toggles.

This is not meant for production. You are solely responsible for data losses, and any damages occurred due to using this repository.

### Included services

| Service | Description |
|---|---|
| `backend` | Frappe gunicorn Python web server |
| `frontend` | Nginx reverse proxy + static assets |
| `websocket` | Socket.IO server for real-time features |
| `queue-default` | Default background job worker |
| `queue-long` | Long-running job worker |
| `queue-short` | Short job worker |
| `scheduler` | Frappe cron/scheduler process |
| `configurator` | One-shot: writes `common_site_config.json` |
| `create-site` | One-shot: creates a new Frappe site |
| `migration` | One-shot: runs `bench migrate` |
| `db` | MariaDB 10.6 (optional, can use external DB) |
| `redis-cache` | Redis for caching |
| `redis-queue` | Redis for background queues |
| `redis-socketio` | Redis for Socket.IO |

---

## Prerequisites

- A server with [Coolify](https://coolify.io/docs/installation) installed
- A domain pointed at your server (for SSL via Coolify)

> **Swarm vs Standalone:** This compose file uses `deploy.replicas` to toggle one-shot services (configurator, create-site, migration). This pattern works in two ways:
> - **Docker Swarm mode** — add your server to Coolify as a Swarm manager. Note: [Coolify's Swarm support is experimental](https://coolify.io/docs/knowledge-base/docker/swarm).
> - **Standalone + Raw Compose** — use Coolify's "Raw Compose Deployment" mode. On standard standalone deployments, Coolify injects fixed container names which conflicts with replicas.

---

## Quick Start

### 1. Fork or clone this repo

```bash
git clone https://github.com/AbdurrehmanSubhani/frappe-coolify-compose.git
```

### 2. Add to Coolify

1. In Coolify, go to **Projects → New Resource → Docker Compose**
2. Point it at this repository (or paste the compose file)
3. Set your environment variables (see below)

### 3. Configure environment variables

Copy `.env.example` and fill in your values:

```bash
cp .env.example .env
```

Set these in Coolify's **Environment Variables** panel. See [Environment Variables](#environment-variables) for full details.

### 4. Initial setup (first deploy)

Run these in order using Coolify's replica controls:

| Step | Service | Set replica to |
|---|---|---|
| 1 | `db` | `1` (if using bundled MariaDB) |
| 2 | `configurator` | `1` (then back to `0` after done) |
| 3 | `create-site` | `1` (then back to `0` after done) |
| 4 | All core services | stays `1` permanently |

> **Tip:** One-shot services (`configurator`, `create-site`, `migration`) should be set back to `0` replicas after they complete — Coolify makes this easy via its UI.

---

## Environment Variables

Copy `.env.example` to `.env` and configure:

```env
# Image
IMAGE_NAME=ghcr.io/frappe/hrms
VERSION=version-15

# Site
SITE_NAME=mysite.example.com
ADMIN_PASSWORD=changeme
INSTALL_APP_ARGS=--install-app hrms

# Database
DB_ROOT_PASSWORD=supersecretpassword
DB_HOST=db
DB_PORT=3306

# Toggle one-shot services (0=off, 1=run once)
ENABLE_DB=1
CONFIGURE=1
CREATE_SITE=1
MIGRATE=0
```

See [`.env.example`](.env.example) for all available variables and documentation.

---

## Running migrations

When you update the image version, run migrations:

1. Set `MIGRATE=1` in Coolify → redeploy
2. After the migration service completes, set `MIGRATE=0` → redeploy

---

## Using an external database

If you have an existing MariaDB/MySQL server:

1. Set `ENABLE_DB=0` to disable the bundled `db` service
2. Set `DB_HOST` to your external database hostname
3. Set `DB_PORT` accordingly (default `3306`)
4. Ensure your DB user has the required privileges

---

## Switching Frappe apps

This compose file defaults to `ghcr.io/frappe/hrms`. To deploy a different app, change `IMAGE_NAME`:

| App | Image |
|---|---|
| ERPNext | `ghcr.io/frappe/erpnext` |
| HRMS | `ghcr.io/frappe/hrms` |
| Custom | Your own built image |

---

## Architecture

```
                        ┌─────────────┐
         Internet ──────►  frontend   │ (Nginx :8080)
                        │  (Coolify   │
                        │  proxy SSL) │
                        └──────┬──────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
         ┌─────────┐    ┌───────────┐    ┌──────────┐
         │ backend │    │ websocket │    │ queues   │
         │ :8000   │    │ :9000     │    │ x3       │
         └────┬────┘    └─────┬─────┘    └────┬─────┘
              │               │               │
              └───────────────┼───────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         ┌─────────┐   ┌───────────┐   ┌──────────┐
         │ MariaDB │   │   Redis   │   │  Redis   │
         │  10.6   │   │  cache    │   │  queue   │
         └─────────┘   └───────────┘   └──────────┘
```

---

## Troubleshooting

**Site not found / 404**
- Check `SITE_NAME` matches the domain Coolify proxies to
- Ensure `create-site` ran successfully (check its logs in Coolify)

**configurator exits immediately**
- This is normal if `common_site_config.json` already exists — it's idempotent

**DB connection refused**
- If using bundled DB, ensure `ENABLE_DB=1` and the `db` service is healthy before running `configurator`

**Workers not processing jobs**
- Check `redis-queue` is healthy
- Restart `queue-*` services after initial setup

---

## Contributing

Contributions, issues, and feature requests are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

[MIT](LICENSE)

---

## Related projects

- [frappe/frappe_docker](https://github.com/frappe/frappe_docker) — Official Frappe Docker setup
- [coollabsio/coolify](https://github.com/coollabsio/coolify) — The self-hosted PaaS this is built for
- [frappe/hrms](https://github.com/frappe/hrms) — Frappe HRMS app
- [frappe/erpnext](https://github.com/frappe/erpnext) — ERPNext ERP app
