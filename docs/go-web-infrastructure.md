# go-web-infrastructure

**GitHub:** `github.com/go-web-services/go-web-infrastructure`

## Role

Shared Docker-oriented operational stack for Go Web deployments: nginx reverse-proxy configs (TLS templates, upstream routing toward the gateway) and an observability bundle (Grafana + Loki + Grafana Alloy scraping container logs). Nothing here is imported as Go code — it complements [go-web-platform](./go-web-platform.md) by providing runtime wiring and dashboards.

## Place in architecture

| | |
|---|---|
| **Layer** | Shared ops — configs and Compose stacks, **not an application binary** |
| **Used next to** | Gateway and services composed on Docker network `go-network`; proxy terminates HTTP(S) toward `go-gateway-template`; Alloy ships logs from containers on `go-network` into Loki for Grafana |
| **Calls / connects** | Public clients → nginx proxy → gateway upstream; Grafana UI → Loki datasource |
| **Deployment** | Build/run via Compose from this repository; stacks attach to external network `go-network`, same convention as gateway and application compose files |

Treat this repo as infrastructure-as-files: fork or vendor into an environment repo, adjust hostnames (`server_name`, upstream hosts), TLS material, retention, and `GF_*` URLs for production.

## Directory structure

```
shared/go-web-infrastructure/
├── .env.sample                      # Copy to `.env`; used for Compose env interpolation (not mounted wholesale as secrets)
├── proxy/
│   ├── docker-compose.proxy.yml      # Builds/runs nginx proxy container on go-network
│   ├── Dockerfile                    # Embeds certs generation + nginx image
│   ├── nginx.conf                    # Sample TLS + upstream routing (placeholders)
│   └── nginx-dev.conf                # Local `/api` and `/swagger` → upstream `go-gateway-template:8009`
├── observability/
│   ├── docker-compose.grafana.yml    # Grafana + Loki + Alloy stack
│   ├── alloy/config.alloy            # Docker log scrape → Loki (`go-network` filtered)
│   ├── loki/loki-config.yaml         # Loki persistence / retention knobs
│   └── grafana/
│       ├── provisioning/datasources/loki.yml
│       └── dashboards/go-service-event.json   # starter dashboard JSON
└── README.md                         # Quick start in upstream repo
```

(In the multi-repo checkout used by this workspace, paths appear under `shared/go-web-infrastructure/`; upstream clone may be extracted at repo root.)

## nginx proxy (`proxy/`)

| File | Purpose |
|---|---|
| `nginx.conf` | Example TLS (`server_name`, upstream backends) — replace placeholders before production |
| `nginx-dev.conf` | Convenience routing: `/api` and `/swagger` proxy to container hostname `go-gateway-template` port `8009` |
| `Dockerfile` | Supplies self-signed material under `/etc/nginx/certs` unless PEMs are bind-mounted |

**Standalone proxy** (requires existing external network `go-network`):

```bash
docker compose -f proxy/docker-compose.proxy.yml up -d --build
```

Upstream `README.md` sometimes shows `-f docker-compose.proxy.yml`; use the path that exists on your branch (commonly `proxy/docker-compose.proxy.yml` at repository root). If the build step cannot find `Dockerfile`, set `services.proxy.build.context` in that compose file to the directory that contains the `Dockerfile` (typically `.` when the compose file lives beside it).

Production TLS typically bind-mounts real certs over `/etc/nginx/certs/server.crt` and `server.key` instead of relying on generated self-signed files.

## Observability (`observability/`)

| Component | Role |
|---|---|
| **Loki** | Log store (`lmk-obs-loki`), retention influenced by `.env` / config |
| **Alloy** | Ships Docker logs from containers attached to `go-network` into Loki; labels streams with container name as `service` |
| **Grafana** | Dashboards (`lmk-obs-grafana`), Loki datasource provisioned under `grafana/provisioning/` |

Bootstrap:

```bash
cp .env.sample .env            # edit GF_* and passwords
docker network create go-network   # once, if stacks do not create it
docker compose --env-file ./.env -f observability/docker-compose.grafana.yml up -d
```

The repository’s `.env.sample` may still reference `observability/docker-compose.yml`; use `observability/docker-compose.grafana.yml` if that is what exists on `main`.
- Alloy UI listens on `${ALLOY_UI_PORT:-4002}` mapped to Alloy’s HTTP server

When serving Grafana behind the nginx proxy, attach both stacks to `go-network`, proxy to `http://lmk-obs-grafana:3000`, and set **`GF_SERVER_ROOT_URL`** (and **`GF_SERVER_SERVE_FROM_SUB_PATH`** if applicable) per inline comments in `docker-compose.grafana.yml`.

See `.env.sample` for `LOKI_LOG_RETENTION_DAYS`, Grafana admin credentials, and Grafana server URL hints.

## Cross-repo conventions

| Convention | Detail |
|---|---|
| **Network** | External Docker network name `go-network` matches gateway/services compose defaults |
| **Upstream hostname** | Dev nginx expects gateway service hostname `go-gateway-template` |
| **Log pipeline** | Matches high-level diagrams in the docs repo README (Loki/Grafana stack alongside application containers) |

## Operational notes

- Alloy uses `pid: host` and the Docker socket to scrape logs — inappropriate for hardened multi-tenant hosts without review; tighten for production accordingly.
- Dashboard JSON under `observability/grafana/dashboards/` is starter material — extend per service labels written by Alloy (`service` matches container name unless relabel rules change).
- Prefer managing secrets (`GF_SECURITY_ADMIN_PASSWORD`, TLS keys) through your orchestration layer rather than committing real `.env` contents.
