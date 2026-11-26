# n8n Self-Hosted Stack (Traefik + PostgreSQL)

Self-hosted [n8n](https://n8n.io/) workflow automation stack based on the [official n8n Docker Compose guide](https://docs.n8n.io/deploy/host-n8n/install-options/use-a-cloud-provider/use-docker-compose), extended with **PostgreSQL** for production persistence and **Traefik** for automatic Let's Encrypt HTTPS.

## Services

| Service | Image | Ports | Purpose |
| --- | --- | --- | --- |
| `traefik` | `traefik:latest` | `80:80`, `443:443` | Reverse proxy and automatic TLS/SSL certificate management via ACME TLS challenge |
| `n8n` | `docker.n8n.io/n8nio/n8n:2.49.6` | `127.0.0.1:5678:5678` | Workflow automation engine (routed via Traefik at `https://${SUBDOMAIN}.${DOMAIN_NAME}`) |
| `postgres` | `postgres:18` | Internal only (`5432` on `n8n-internal`) | PostgreSQL database for n8n workflows, executions, and credentials (not exposed to host/public network) |

## Directory Structure

```text
n8n/
├── .env
├── ReadMe.md
├── docker-compose.yaml
├── init-data.sh
└── local-files/
```

## Prerequisites

1. **DNS**: Point an `A` record for `${SUBDOMAIN}.${DOMAIN_NAME}` (e.g. `n8n.example.com`) to your server's public IP address.
2. **Firewall**: Ensure inbound TCP ports `80` and `443` are open for HTTP→HTTPS redirection and Let's Encrypt TLS-ALPN-01 certificate validation.

## Environment Variables (`.env`)

Update [`.env`](.env) before starting the stack:

| Variable | Description |
| --- | --- |
| `N8N_VERSION` | n8n Docker image tag (e.g. `2.49.6`) |
| `DOMAIN_NAME` | Base domain name (e.g. `example.com`) |
| `SUBDOMAIN` | Subdomain for n8n (e.g. `n8n`) |
| `SSL_EMAIL` | Email address used for Let's Encrypt TLS certificates |
| `GENERIC_TIMEZONE` | Timezone applied to both `TZ` and `GENERIC_TIMEZONE` (e.g. `Asia/Kolkata`) |
| `N8N_ENCRYPTION_KEY` | Secret key used to encrypt stored n8n credentials |
| `POSTGRES_USER` | PostgreSQL superuser for initial DB setup and healthchecks |
| `POSTGRES_PASSWORD` | PostgreSQL superuser password |
| `POSTGRES_DB` | PostgreSQL database name for n8n |
| `POSTGRES_NON_ROOT_USER` | Dedicated least-privilege PostgreSQL user used by n8n at runtime |
| `POSTGRES_NON_ROOT_PASSWORD` | Password for `POSTGRES_NON_ROOT_USER` |

Generate strong secrets for `N8N_ENCRYPTION_KEY` and database passwords:

```bash
openssl rand -hex 32
```

## Deployment

```bash
cd n8n
docker compose up -d
```

To view logs:

```bash
docker compose logs -f n8n
```

To stop the stack:

```bash
docker compose down
```

## Volumes & File Access

| Mount / Volume | Container Path | Purpose |
| --- | --- | --- |
| `n8n_data` | `/home/node/.n8n` | n8n configuration and local instance files |
| `postgres_data` | `/var/lib/postgresql/data` | Persistent PostgreSQL database storage |
| `traefik_data` | `/letsencrypt` | Traefik `acme.json` TLS certificates |
| `./local-files` | `/files` | Shared host directory (`N8N_RESTRICT_FILE_ACCESS_TO=/files`) |
