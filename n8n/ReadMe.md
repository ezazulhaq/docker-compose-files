# n8n Self-Hosted Stack for Dokploy (PostgreSQL + Task Runners + Sandbox + SearXNG)

Self-contained [n8n](https://n8n.io/) workflow automation stack for **Dokploy** (where Traefik/TLS is managed by Dokploy and no external helper files are required), combining:
- **PostgreSQL 18** (`postgres`) on an internal-only network
- **External Task Runners** (`n8n-runner`) for Code node isolation
- **n8n Assistant Sandbox** (`sandbox-certs`, `sandbox-api`, `sandbox-runner-1`) + **SearXNG** (`searxng`) per n8n's [Docker Compose guide](https://docs.n8n.io/deploy/host-n8n/install-options/install-using-docker-compose) and [Production requirements](https://docs.n8n.io/deploy/host-n8n/install-options/production-requirements)

> [!WARNING]
> **Sandbox Security Note (`sandbox-runner-1`)**: `sandbox-runner-1` uses `privileged: true` and Docker-in-Docker (as specified in n8n's Docker Compose guide for development/testing). n8n recommends Kubernetes with Sysbox runners (or Daytona) for hardened production multi-tenant sandboxing. Keep `sandbox-runner-1` and `sandbox-api` private (never expose ports) and do not pass `env_file: .env` to them.

## Architecture & Services

| Service | Image | Network(s) | Ports | Purpose |
| --- | --- | --- | --- | --- |
| `n8n` | `docker.n8n.io/n8nio/n8n:2.49.6` | `n8n-public`, `n8n-internal`, `n8n-runners`, `n8n-sandbox`, `n8n-search` | `expose: 5678` | Main n8n editor, API, webhooks, and task broker (`5679` internal) |
| `n8n-runner` | `ghcr.io/n8n-io/runners:2.49.6` | `n8n-runners` (`internal: true`) | None | Isolated external task runner sidecar for Code nodes |
| `postgres` | `postgres:18` | `n8n-internal` (`internal: true`) | `expose: 5432` | Dedicated PostgreSQL database (no host ports, no public/runner/sandbox access) |
| `sandbox-certs` | `ghcr.io/n8n-io/n8n-sandbox-service-api:1.16.0` | `n8n-sandbox` | None | One-shot init container generating mTLS certs into `sandbox_tls` |
| `sandbox-api` | `ghcr.io/n8n-io/n8n-sandbox-service-api:1.16.0` | `n8n-sandbox` | `expose: 8080, 9090` | Sandbox control plane for n8n Assistant |
| `sandbox-runner-1` | `ghcr.io/n8n-io/n8n-sandbox-service-runner-dind:1.16.0` | `n8n-sandbox` | None | Privileged DinD runner executing sandbox containers |
| `searxng` | `ghcr.io/searxng/searxng:latest` | `n8n-search` | `expose: 8080` | Internal web search engine (inline `searxng_settings` config enables JSON + HTML) |

### Network Isolation

- **`n8n-public`**: External-facing bridge network for `n8n` (`expose: 5678` for Dokploy's Traefik).
- **`n8n-internal` (`internal: true`)**: Isolates `postgres` strictly to `n8n`.
- **`n8n-runners` (`internal: true`)**: Isolates `n8n-runner` strictly to `n8n` port `5679`.
- **`n8n-sandbox`**: Isolates `sandbox-certs`, `sandbox-api`, and `sandbox-runner-1` with `n8n` (no access to `postgres`, `n8n-runner`, or `searxng`).
- **`n8n-search`**: Connects `n8n` and `searxng` (allows `searxng` outbound search queries with no public ports).

## Directory Structure

```text
n8n/
├── .env
├── ReadMe.md
└── docker-compose.yaml
```

## Dokploy Deployment

1. Create a **Compose** service in Dokploy (either pointing to `n8n/docker-compose.yaml` in Git or pasting its contents into **Raw** mode).
2. Copy the variables from [`.env`](.env) into Dokploy's **Environment** tab and replace every `change-me-*` value (generate 32-byte hex keys with `openssl rand -hex 32`):
   - Keep `SANDBOX_API_KEYS` and `N8N_SANDBOX_SERVICE_API_KEY` identical.
   - Keep `SANDBOX_API_RUNNER_REGISTRATION_TOKEN` and `SANDBOX_RUNNER_REGISTRATION_TOKEN` identical.
   - Keep `SANDBOX_API_RUNNER_API_KEY` and `SANDBOX_RUNNER_API_KEYS` identical.
3. In Dokploy's **Domains** tab, map your domain (e.g. `n8n.example.com`) to service **`n8n`** on port **`5678`** with HTTPS enabled.
4. Deploy the stack (host requires at least **4 GB RAM** and **2 vCPUs** for `sandbox-runner-1`).
5. Add your LLM provider key in the n8n UI (**Settings → Assistant**) or via `N8N_INSTANCE_AI_MODEL_API_KEY` in `.env` to activate n8n Assistant.

## Verification

```bash
# Verify sandbox-api is reachable from n8n
docker compose exec n8n wget -qO- http://sandbox-api:8080/healthz

# Verify sandbox-runner-1 registered with sandbox-api
docker compose logs sandbox-api | grep -i runner

# Verify SearXNG JSON API responds internally from n8n
docker compose exec n8n wget -qO- "http://searxng:8080/search?q=n8n&format=json" | head -c 200
```

## Maintenance & Backups

- **Backups**: Back up `postgres_data` (`pg_dump`), `n8n_data` (`/home/node/.n8n`), `n8n_files` (`/files`), and `N8N_ENCRYPTION_KEY`.
- **Sandbox TLS Certs**: `sandbox-certs` does not auto-renew mTLS certificates; if certs expire, remove the `sandbox_tls` volume and redeploy so `sandbox-certs` regenerates them.
