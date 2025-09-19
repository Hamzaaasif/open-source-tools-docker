## Open Source Tools with Docker

This repository provides ready-to-run Docker Compose setups for:

- **Directus**: Headless CMS/API
- **n8n**: Workflow automation (single-node and worker/queue mode)
- **Vaultwarden**: Lightweight Bitwarden-compatible password manager

Each tool is isolated in its own directory with a `docker-compose.yml` and supporting files.

### Prerequisites

- Docker Desktop 4.x+
- Docker Compose v2 (bundled with Docker Desktop)
- macOS, Linux, or Windows with WSL2

### Directory Layout

- `directus/`
  - `docker-compose.yml`: Directus + Postgres + Redis
  - `env`: Example environment file (copy to `.env`)
  - `uploads/`, `extensions/`, `migrations/`: Persistent app data and extensions
- `n8n/`
  - `withPostgres/`: Single-node n8n + Postgres
    - `docker-compose.yml`, `init-data.sh`, `nginx.conf`, `README.md`
  - `withPostgresAndWorker/`: n8n with queue worker + Redis + Postgres
    - `docker-compose.yml`, `init-data.sh`, `README.md`
- `vaultwarden/`
  - `docker-compose.yml`: Vaultwarden server
  - `env`: Example environment file (copy to `.env`)
  - `vault.nginx.conf`: Example reverse-proxy config

---

## Quick Start

You can run any stack independently. In each section, replace placeholders to match your environment.

### 1) Directus

Location: `directus/`

1. Create environment file:
   - Copy the provided sample file: `cp directus/env directus/.env`
   - Open `directus/.env` and set at minimum:
     - `APP_NAME`, `APP_ENV`
     - `POSTGRES_EXPOSE_PORT` (host port for Postgres, e.g. 5433)
     - `DIRECTUS_PORT` (host port for Directus, defaults to 8055 on container)
     - Database credentials (user, password, name) expected by the compose file
     - `REDIS_HOST_PASSWORD`

2. Start services:
   - `cd directus`
   - `docker compose up -d`

3. Access:
   - Directus UI/API: `http://localhost:${DIRECTUS_PORT}`
   - Postgres: `localhost:${POSTGRES_EXPOSE_PORT}`

4. First-time setup:
   - Open the Directus URL and complete the admin user creation.

5. Data & volumes:
   - Persistent data is stored in:
     - `directus/data/database` (Postgres)
     - `directus/uploads` (file storage)
     - `directus/extensions` (extensions)
     - `directus/migrations` (schema migrations)

6. Common commands:
   - Stop: `docker compose down`
   - Logs: `docker compose logs -f directus`
   - Update image: `docker compose pull && docker compose up -d`

Notes:
- CORS, token TTLs, websocket logs, and storage paths are preconfigured in the compose environment section. Adjust in `.env` if needed.

### 2) n8n

n8n is provided in two variants. Choose one.

#### Option A: Single-node (with Postgres)

Location: `n8n/withPostgres/`

1. Create environment variables (shell or `.env` used by Docker):
   - Set/Export at minimum before running:
     - `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
     - `POSTGRES_NON_ROOT_USER`, `POSTGRES_NON_ROOT_PASSWORD`
     - `SUBDOMAIN`, `DOMAIN_NAME` (for webhook and host configuration)
     - Optional for auth: `N8N_BASIC_AUTH_USER`, `N8N_BASIC_AUTH_PASSWORD`

2. Start services:
   - `cd n8n/withPostgres`
   - `docker compose up -d`

3. Access:
   - UI: `http://localhost:5678`
   - Health: check logs if container restarts

4. Data & volumes:
   - Postgres data in a named volume (`db_storage`)
   - n8n config/data in a named volume (`n8n_storage`)

5. Common commands:
   - Stop: `docker compose down`
   - Logs: `docker compose logs -f n8n`
   - Update image: `docker compose pull && docker compose up -d`

Notes:
- Compose sets `N8N_PROTOCOL=https` and `WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/`. If running locally without HTTPS, adjust or set appropriate hosts.

#### Option B: Queue mode with worker (with Postgres + Redis)

Location: `n8n/withPostgresAndWorker/`

1. Create environment variables (shell or `.env` used by Docker):
   - Set/Export at minimum before running:
     - `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
     - `POSTGRES_NON_ROOT_USER`, `POSTGRES_NON_ROOT_PASSWORD`
     - `ENCRYPTION_KEY` (required for credential encryption)

2. Start services:
   - `cd n8n/withPostgresAndWorker`
   - `docker compose up -d`

3. Access:
   - UI: `http://localhost:5678`

4. Data & volumes:
   - Postgres data: `db_storage`
   - Redis data: `redis_storage`
   - n8n config/data: `n8n_storage`

5. Common commands:
   - Stop: `docker compose down`
   - Logs (web): `docker compose logs -f n8n`
   - Logs (worker): `docker compose logs -f n8n-worker`
   - Update image: `docker compose pull && docker compose up -d`

Notes:
- Execution mode is set to queue; worker is started with `command: worker`.
- Ensure `ENCRYPTION_KEY` is a strong, persistent value. Changing it will invalidate existing credentials.

### 3) Vaultwarden

Location: `vaultwarden/`

1. Create environment file:
   - Copy the provided sample file: `cp vaultwarden/env vaultwarden/.env`
   - Open `vaultwarden/.env` and set at minimum:
     - `PORT` (host port to expose the service)
     - Recommended: `ADMIN_TOKEN` for the admin panel
     - Optional: SMTP and domain-related settings if enabling invites or email

2. Start services:
   - `cd vaultwarden`
   - `docker compose up -d`

3. Access:
   - UI/API: `http://localhost:${PORT}`

4. Data & volumes:
   - Persistent data in: `vaultwarden/data`

5. Common commands:
   - Stop: `docker compose down`
   - Logs: `docker compose logs -f vaultwarden`
   - Update image: `docker compose pull && docker compose up -d`

Reverse Proxy (optional):
- An example Nginx config is provided at `vaultwarden/vault.nginx.conf`. Adapt domains, TLS certificates, and upstream ports.

---

## Backups

- Directus
  - Database: back up `directus/data/database`
  - Files: back up `directus/uploads`, `directus/extensions`, `directus/migrations`
- n8n
  - Database: named volume `db_storage`
  - App data: named volume `n8n_storage`; plus `redis_storage` (queue mode)
- Vaultwarden
  - Data directory: `vaultwarden/data`

Use `docker run --rm -v <volume_or_path>:/data -v "$PWD":/backup alpine tar -czf /backup/<name>.tar.gz /data` for quick tarball backups, or your usual backup tooling.

## Updates

- Pull new images and recreate:
  - `docker compose pull && docker compose up -d`
- For stacks with multiple services, run the commands inside the respective directories.

## Troubleshooting

- Port already in use: Change exposed host ports in the compose files or `.env`.
- Environment not loaded: Verify you created `.env` (not `env`) in the service directory or exported variables in your shell.
- Stuck on starting:
  - Check dependencies via `depends_on` health checks (Postgres/Redis must be healthy).
  - View logs: `docker compose logs -f`
- File permissions:
  - On Linux, you may need to adjust ownership of data directories (UID/GID) based on container users.

## Security Notes

- Always set strong passwords/secrets in `.env` files and do not commit them.
- Use HTTPS behind a reverse proxy for public exposure.
- Regularly update images and rotate credentials.

## License

This repo aggregates open-source services. Each service is licensed by its respective project. Review individual licenses before production use.
