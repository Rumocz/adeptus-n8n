# adeptus-n8n

Self-hosted [n8n](https://n8n.io/) running on Docker Compose, with config in Git so
the same stack runs locally (WSL2) and on a remote Ubuntu server.

- **n8n** + **Postgres** (data lives in Postgres, not SQLite)
- Persistent named volumes — workflows & credentials survive restarts
- One base compose file for both environments; a prod overlay adds HTTPS

## What's in Git vs not

Committed (the deployable config):

```
adeptus-n8n/
├── docker-compose.yml        # base stack (n8n + postgres) — both environments
├── docker-compose.prod.yml   # server overlay: Caddy + automatic HTTPS
├── caddy/Caddyfile           # reverse-proxy config
├── .env.example              # template for environment values
├── .gitignore
└── README.md
```

**Not** committed: `.env` (your secrets) and the Docker volumes (your actual data).
Data persistence comes from named volumes (`n8n_data`, `postgres_data`), which are
deliberately outside Git — see "Backups & migration".

---

## Phase 1 — Run locally in WSL2

Docker Engine + Compose are already installed in this WSL2 instance. If you start
fresh elsewhere, see [Installing Docker](#installing-docker-on-ubuntu) below.

1. Create your env file and generate secrets:

   ```bash
   cp .env.example .env
   # then fill in:
   openssl rand -hex 32   # paste as N8N_ENCRYPTION_KEY
   openssl rand -hex 24   # paste as POSTGRES_PASSWORD
   ```

2. Start the stack:

   ```bash
   docker compose up -d
   ```

3. Check it's healthy:

   ```bash
   docker compose ps
   curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5678/healthz   # 200
   ```

4. Open **http://localhost:5678** in your Windows browser and create the owner
   account. WSL2 forwards `localhost`, so no WSL IP is needed.

Common commands:

```bash
docker compose logs -f n8n     # tail logs
docker compose restart n8n     # restart after an env change
docker compose down            # stop (keeps data in volumes)
docker compose pull && docker compose up -d   # upgrade n8n
```

> The host port is bound to `127.0.0.1` only. To reach n8n from other devices on
> your LAN via the WSL IP, change the port mapping to `"5678:5678"` in
> `docker-compose.yml`.

---

## Phase 2 — Deploy to a remote Ubuntu server

The base stack is unchanged. Only three things differ: **env values**, an **HTTPS
overlay**, and **firewall/DNS**.

### 1. Push, then pull on the server

```bash
# local
git add -A && git commit -m "n8n stack" && git push

# server
git clone <your-repo-url> adeptus-n8n && cd adeptus-n8n
```

### 2. Server `.env`

Copy the template and use the **server values** (commented at the bottom of
`.env.example`):

```bash
cp .env.example .env
```

```ini
N8N_HOST=n8n.example.com
N8N_PROTOCOL=https
N8N_EDITOR_BASE_URL=https://n8n.example.com
WEBHOOK_URL=https://n8n.example.com
N8N_SECURE_COOKIE=true
N8N_DOMAIN=n8n.example.com
ACME_EMAIL=you@example.com
```

Generate a **fresh** `POSTGRES_PASSWORD` on the server. For the encryption key:
use a fresh one for a brand-new instance, **or reuse the local key** if you're
migrating existing workflows/credentials (otherwise saved credentials won't
decrypt).

### 3. DNS + firewall

- Point an **A record** for `n8n.example.com` at the server's public IP.
- Open ports **80** and **443** (e.g. `sudo ufw allow 80,443/tcp`, plus your
  cloud provider's security group). Do **not** expose 5678 publicly — Caddy
  proxies to it over the internal network.

### 4. Start with the prod overlay

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Caddy fetches a Let's Encrypt certificate automatically. Browse to
**https://n8n.example.com**.

> Tip: create an alias so you don't repeat the `-f` flags:
> `alias dcp='docker compose -f docker-compose.yml -f docker-compose.prod.yml'`

### Updating the server later

```bash
git pull
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Installing Docker on Ubuntu

For a fresh server (or any Ubuntu without Docker):

```bash
# Docker's official convenience script
curl -fsSL https://get.docker.com | sudo sh

# Run docker without sudo (log out/in afterward, or run: newgrp docker)
sudo usermod -aG docker $USER

# Verify
docker --version && docker compose version
```

---

## Backups & migration

Your data is in the `postgres_data` and `n8n_data` volumes. The Postgres dump is
the source of truth for workflows and credentials:

```bash
# Back up
docker compose exec -T postgres pg_dump -U n8n n8n > backup.sql

# Restore (into a fresh stack with the SAME N8N_ENCRYPTION_KEY)
docker compose exec -T postgres psql -U n8n -d n8n < backup.sql
```

Keep `N8N_ENCRYPTION_KEY` safe and identical wherever you restore — without it,
encrypted credentials in the dump cannot be decrypted.
