# provisioning/ — Grafana configuration as code

Grafana loads this folder on startup (mounted at `/etc/grafana/provisioning`).
Anything placed here is version-controlled and reproducible across environments.

| Folder | Status | What to put here | Reference |
|---|---|---|---|
| `datasources/` | **2 versions included** | ClickHouse data source (HTTP active + Native alternative). | Manual Section A5 |
| `dashboards/`  | empty | A YAML *provider* pointing to a folder of `.json` dashboards. | Manual Section 3 |
| `alerting/`    | empty | Contact points, templates, and notification policies (email uses `../templates/`). | Manual Section A9 |

The data source ships in two versions (same `name`/`uid`):

| File | Protocol | Port | Status |
|---|---|---|---|
| `datasources/clickhouse.yaml` | HTTP | 8123 (8443 TLS) | **active** |
| `datasources/clickhouse-native.yaml.example` | Native | 9000 (9440 TLS) | inactive (`.example`) |

To switch to Native: delete/rename `clickhouse.yaml` and rename `clickhouse-native.yaml.example` → `clickhouse.yaml`
(only one active `.yaml`). In both, the password is **not** stored in the YAML: it is read from `CLICKHOUSE_PASSWORD` (`.env`).
The concrete content of each version lives in the files themselves; adjust `host`/`port`/`defaultDatabase` for your ClickHouse.

> It can also be created manually from the UI (*Connections → Data sources*) if you prefer not to use provisioning.
