# Grafana + ClickHouse deployment stack

A ready-to-run stack that puts **Grafana OSS** behind **nginx (TLS)** to visualize data
from an **existing ClickHouse**. Grafana does not store the data: it queries ClickHouse
and renders it. Full documentation in [docs/01_Grafana_ClickHouse_Integration_Manual_EN.md](docs/01_Grafana_ClickHouse_Integration_Manual_EN.md)
(a Spanish version is also available under `docs/`).

## Structure

```
.
├── docker-compose.yaml   # Grafana OSS + nginx (TLS reverse proxy)
├── nginx.conf            # Virtual host + HTTP→HTTPS redirect + WebSocket
├── .env.example          # Variables template (domain, admin, SMTP) — copy to .env
├── certs/                # fullchain.crt + private.key (placeholders; add the real ones)
├── provisioning/         # Grafana configuration as code (see manual Section A5)
│   ├── datasources/      #   → clickhouse.yaml (HTTP, active) + clickhouse-native.yaml.example (Native)
│   ├── dashboards/       #   → dashboard provider (empty)
│   └── alerting/         #   → contact points / notification policies (empty)
├── templates/            # Alert email templates (branding)
└── docs/                 # Integration manual
```

## Quick start

1. **Configure variables**
   ```bash
   cp .env.example .env
   # edit .env: GRAFANA_DOMAIN + admin credentials + CLICKHOUSE_PASSWORD (and SMTP if needed)
   ```
2. **Domain in nginx** — replace `grafana.tu-dominio.com` in [nginx.conf](nginx.conf) with the same value as `GRAFANA_DOMAIN`.
3. **Certificates** — place the real `fullchain.crt` and `private.key` in [certs/](certs/) (see [certs/README.md](certs/README.md)).
4. **Start**
   ```bash
   docker compose up -d
   docker compose logs -f grafana
   ```
5. **Access** — `https://<GRAFANA_DOMAIN>/` and sign in with the admin user from `.env`.
6. **Data source** — it ships as code in [provisioning/datasources/](provisioning/datasources/) in two versions: `clickhouse.yaml` (**HTTP**, active) and `clickhouse-native.yaml.example` (**Native**, alternative — rename it to use it). Adjust `host`/`port`/`defaultDatabase` (the password comes from `CLICKHOUSE_PASSWORD`). It can also be set up from *Connections → Data sources* (manual Section A5).

## Notes

- Public access is **only through nginx (443)**. Port `3000` is published on `127.0.0.1` for local development only.
- The `provisioning/` folders ship **empty** on purpose; fill them in per project (see manual Section A5 and the alerting section).
- Update: `docker compose pull && docker compose up -d`. Backup: the `storage` volume (`/var/lib/grafana`, which holds the SQLite configuration database).
