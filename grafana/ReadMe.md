# Grafana Observability Stack

Self-hosted observability stack for Docker and host monitoring using **Grafana**, **Prometheus**, **Loki**, **Grafana Alloy**, **Node Exporter**, and **cAdvisor**. Configured with `expose` for reverse-proxy routing (e.g. Dokploy / Traefik).

## Services

| Service | Image | Internal Port | Purpose |
| --- | --- | --- | --- |
| `grafana` | `grafana/grafana:11.6.0` | `3000` | Visualization UI and dashboards (`expose: 3000`) |
| `prometheus` | `prom/prometheus:v3.2.1` | `9090` | Time-series metrics storage and scraping |
| `loki` | `grafana/loki:3.4.2` | `3100` | Log aggregation (single-binary TSDB, `v13` schema) |
| `alloy` | `grafana/alloy:v1.7.5` | `12345` | Collects Docker container logs via `/var/run/docker.sock` and ships to Loki |
| `node-exporter` | `prom/node-exporter:v1.9.0` | `9100` | Host CPU, memory, disk, and network metrics |
| `cadvisor` | `gcr.io/cadvisor/cadvisor:v0.52.1` | `8080` | Per-container CPU, memory, disk, and network metrics |

## Directory Structure

```text
grafana/
├── .env
├── ReadMe.md
├── docker-compose.yaml
└── config/
    ├── alloy/
    │   └── config.alloy
    ├── grafana/
    │   ├── dashboards/
    │   └── provisioning/
    │       ├── dashboards/
    │       │   └── dashboards.yaml
    │       └── datasources/
    │           └── datasources.yaml
    ├── loki/
    │   └── loki-config.yaml
    └── prometheus/
        └── prometheus.yml
```

## Environment Variables (`.env`)

Update [`.env`](.env) before deploying to production:

| Variable | Default | Description |
| --- | --- | --- |
| `TZ` | `Asia/Kolkata` | Container timezone |
| `GRAFANA_ADMIN_USER` | `admin` | Initial Grafana admin username |
| `GRAFANA_ADMIN_PASSWORD` | `change-me-secure-password` | Initial Grafana admin password (change before deploying) |
| `GRAFANA_ROOT_URL` | `https://grafana.yourdomain.com` | Public URL used by Grafana for links and redirects |
| `PROMETHEUS_RETENTION` | `15d` | Prometheus TSDB metric retention period |
| `LOKI_RETENTION` | `168h` | Loki log retention period (e.g. `168h` = 7 days) |
| `DOCKER_DATA` | `/mnt/data/docker` | Host Docker `data-root` mounted read-only into `cadvisor` (falls back to `/var/lib/docker` if unset) |

## Deployment

### Dokploy

1. Create a Compose service in Dokploy pointing to `grafana/docker-compose.yaml` and copy the variables from [`.env`](.env) into Dokploy's environment configuration.
2. In the **Domains** tab, map your domain (e.g. `grafana.yourdomain.com`) to service **`grafana`** on port **`3000`**.
3. Deploy the stack.

### Docker Compose CLI

```bash
cd grafana
docker compose up -d
```

To stop the stack:

```bash
docker compose down
```

## Pre-Provisioned Data Sources & Dashboards

- **Data sources** are auto-provisioned via [`datasources.yaml`](config/grafana/provisioning/datasources/datasources.yaml):
  - **Prometheus** (`uid: prometheus`, default data source) -> `http://prometheus:9090`
  - **Loki** (`uid: loki`) -> `http://loki:3100`
- **Dashboards**: Place dashboard JSON files inside `config/grafana/dashboards/` (subfolders are preserved) or import dashboards directly in the Grafana UI. Grafana checks for file updates every 30 seconds.
- **Log Labels**: Alloy attaches `container`, `compose_project`, and `compose_service` labels to container logs for filtering in Grafana Explore (e.g. `{compose_project="supabase"}`).

## Persistent Volumes

All stateful data is stored in Docker named volumes:

- `grafana-data` (`/var/lib/grafana`)
- `prometheus-data` (`/prometheus`)
- `loki-data` (`/loki`)
- `alloy-data` (`/var/lib/alloy/data`)

## Notes & Troubleshooting

- **cAdvisor container metrics**: If Docker on your host uses `/var/lib/docker` instead of `/mnt/data/docker`, update `DOCKER_DATA` in [`.env`](.env) to match `docker info --format '{{.DockerRootDir}}'`.
- **Scraped Targets**: Prometheus scrapes `prometheus:9090`, `grafana:3000`, `loki:3100`, `alloy:12345`, `node-exporter:9100`, and `cadvisor:8080` every `15s` as defined in [`prometheus.yml`](config/prometheus/prometheus.yml).
- **Loki retention**: Retention is enforced by Loki's compactor using `LOKI_RETENTION` in [`loki-config.yaml`](config/loki/loki-config.yaml) via `-config.expand-env=true`.
