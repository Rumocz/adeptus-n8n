# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`adeptus-n8n` is a self-hosted [n8n](https://n8n.io/) deployment managed with Docker Compose. Config is kept in Git so the **same stack runs locally (WSL2) and on a remote Ubuntu server** — environment differences are isolated to `.env`, never the compose files.

## Architecture

- `docker-compose.yml` — base stack (n8n + Postgres 16), used in **both** environments. n8n stores everything in Postgres (`DB_TYPE=postgresdb`), not SQLite. Data persists in named volumes `n8n_data` and `postgres_data`. The host port is bound to `127.0.0.1:5678` only.
- `docker-compose.prod.yml` — **server-only** overlay adding a Caddy reverse proxy with automatic Let's Encrypt HTTPS. Caddy reaches n8n over the internal compose network (`n8n:5678`), so 5678 is never publicly exposed.
- `caddy/Caddyfile` — proxy + TLS config, parameterized by `$N8N_DOMAIN` / `$ACME_EMAIL`.
- `.env` (gitignored) holds all secrets and per-environment values; `.env.example` is the committed template carrying both local and server value sets.

Key invariant: `N8N_ENCRYPTION_KEY` encrypts saved credentials. It must stay stable and identical anywhere data is restored — if it changes or is lost, stored credentials cannot be decrypted.

## Commands

```bash
# Local (WSL2)
docker compose up -d
docker compose logs -f n8n
docker compose down                          # stops; volumes (data) persist

# Server (base + prod overlay)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Validate compose without starting
docker compose config --quiet

# Upgrade n8n
docker compose pull && docker compose up -d

# Backup / restore (Postgres is the source of truth)
docker compose exec -T postgres pg_dump -U n8n n8n > backup.sql
docker compose exec -T postgres psql -U n8n -d n8n < backup.sql
```

Local UI: http://localhost:5678 (WSL2 forwards `localhost` to Windows).

See `README.md` for the full local-setup and server-deploy walkthrough.
