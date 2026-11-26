# n8n Production Stack (Traefik + PostgreSQL + External Task Runners)

Production-ready self-hosted [n8n](https://n8n.io/) workflow automation stack aligned with n8n's [Docker Compose guide](https://docs.n8n.io/deploy/host-n8n/install-options/use-a-cloud-provider/use-docker-compose) and [Production requirements](https://docs.n8n.io/deploy/host-n8n/install-options/production-requirements).

## Architecture & Services

| Service | Image | Network(s) | Ports | Purpose |
| --- | --- | --- | --- | --- |
| `traefik` | `traefik:v3` | `n8n-public` | `80:80`, `443:443` | Reverse proxy, HTTP→HTTPS redirect, and Let's Encrypt ACME TLS termination |
| `n8n` | `docker.n8n.io/n8nio/n8n:2.49.6` | `n8n-public`, `n8n-internal`, `n8n-runners` | `127.0.0.1:5678:5678` | Main n8n editor, API, webhook handler, and task broker (`5679` internal) |
| `n8n-runner` | `n8nio/runners:2.49.6` | `n8n-runners` (`internal: true`) | None | Isolated external task runner sidecar for Code node execution |
| `postgres` | `postgres:18` | `n8n-internal` (`internal: true`) | Internal `5432` | Dedicated PostgreSQL database (no host ports, no public/runner access) |

### Network Isolation

- **`n8n-public`**: Bridges `traefik` and `n8n` for external HTTPS routing and outbound workflow HTTP requests. `traefik.docker.network=n8n-public` ensures Traefik routes only over this network.
- **`n8n-internal` (`internal: true`)**: Isolates `postgres` so only `n8n` can reach it; `postgres` has no published ports and no internet/proxy access.
- **`n8n-runners` (`internal: true`)**: Isolates `n8n-runner` so it can only communicate with `n8n`'s task broker on port `5679` and cannot reach `postgres` or the public network.

## Directory Structure

```text
n8n/
├── .env
├── ReadMe.md
├── docker-compose.yaml
├── init-data.sh
└── local-files/
```

## Production Pre-Deployment Checklist

1. **64-bit Linux host** with Docker Engine and Docker Compose v2.
2. **DNS**: Point an `A` record for `${SUBDOMAIN}.${DOMAIN_NAME}` to your server's public IP.
3. **Firewall**:
   - Allow inbound TCP `80` and `443` for HTTP→HTTPS redirection and Let's Encrypt TLS-ALPN-01 validation.
   - Keep `5678` (bound to `127.0.0.1`), `5679`, and `5432` closed to the public internet.
4. **Secrets in [`.env`](.env)**:
   - Generate distinct 32-byte hex keys for `N8N_ENCRYPTION_KEY` and `N8N_RUNNERS_AUTH_TOKEN`:
     ```bash
     openssl rand -hex 32
     ```
   - Set strong, unique passwords for `POSTGRES_PASSWORD` and `POSTGRES_NON_ROOT_PASSWORD`.
   - Back up `N8N_ENCRYPTION_KEY` securely; losing it prevents decrypting stored credentials.
5. **Proxy hops**: Keep `N8N_PROXY_HOPS=1` when Traefik is directly in front of n8n; increase if you add an upstream CDN/load balancer (e.g. Cloudflare proxy).
6. **Backup strategy**: Schedule automated backups of:
   - PostgreSQL (`docker compose exec -T postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" > backup.sql`)
   - `n8n_data` volume (`/home/node/.n8n`, which stores local binary data in `filesystem` mode)
   - Your `.env` file (`N8N_ENCRYPTION_KEY`)

## Environment Variables (`.env`)

| Variable | Description |
| --- | --- |
| `N8N_VERSION` | Matching image tag for `n8nio/n8n` and `n8nio/runners` |
| `DOMAIN_NAME` | Base domain name (e.g. `example.com`) |
| `SUBDOMAIN` | Subdomain for n8n (e.g. `n8n`) |
| `SSL_EMAIL` | Email address for Let's Encrypt TLS certificates |
| `N8N_PROXY_HOPS` | Number of trusted reverse-proxy hops (`1` for Traefik) |
| `TRAEFIK_LOG_LEVEL` | Traefik log level (`WARN` recommended for production) |
| `GENERIC_TIMEZONE` | Timezone applied to `TZ` and `GENERIC_TIMEZONE` |
| `N8N_ENCRYPTION_KEY` | Secret key used to encrypt stored n8n credentials |
| `N8N_RUNNERS_AUTH_TOKEN` | Shared authentication secret between `n8n` and `n8n-runner` |
| `POSTGRES_USER` | PostgreSQL superuser for DB initialization and healthchecks |
| `POSTGRES_PASSWORD` | PostgreSQL superuser password |
| `POSTGRES_DB` | PostgreSQL database name for n8n |
| `POSTGRES_NON_ROOT_USER` | Least-privilege PostgreSQL user used by n8n at runtime |
| `POSTGRES_NON_ROOT_PASSWORD` | Password for `POSTGRES_NON_ROOT_USER` |
| `EXECUTIONS_DATA_MAX_AGE` | Retention window in hours before execution pruning (`336` = 14 days) |
| `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | Maximum retained execution records |

## Deployment

```bash
cd n8n
docker compose up -d
```

Check service health:

```bash
docker compose ps
docker compose logs -f n8n n8n-runner
```

To stop the stack:

```bash
docker compose down
```

## Scaling to Queue Mode (Optional)

This setup runs in `regular` execution mode with an external task runner and `filesystem` binary data storage. If your workload outgrows a single instance, add a persistent **Redis** service, switch `EXECUTIONS_MODE=queue`, run dedicated `worker` (and optional `webhook`) containers, and move binary data to shared storage or supported external storage (S3 / Azure Blob Storage).
