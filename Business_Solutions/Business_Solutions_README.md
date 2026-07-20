# Business Solutions

Ready-to-run **solution stacks and reference architectures** that turn the
data delivered by OneConnect into tangible business value.

Where the [SAP Data Modeler](../SAP_Data_Modeler) and the
[Smart Gateway](../Smart_Gateway) focus on **extracting and streaming** SAP
data in real time, the assets in this folder focus on the **consumption
side**: they show how to deploy, secure, and operate the downstream tools
that visualize, analyze, and act on that data once it lands in a target
platform (data lakes, analytics engines, dashboards, and alerting).

Each solution is self-contained and includes the deployment artifacts
(configuration as code, container definitions) together with step-by-step
manuals and, where relevant, cost and licensing analyses.

## Contents

### [grafana-clickhouse-stack](./grafana-clickhouse-stack)

A ready-to-run stack that puts **Grafana OSS** behind **nginx (TLS)** to
visualize data stored in an **existing ClickHouse** instance — one of the
OneConnect Premium Connector destinations. Grafana does not store the data;
it queries ClickHouse and renders it in dashboards and alerts. This folder
includes:

- Deployment artifacts — `docker-compose.yaml`, `nginx.conf`, TLS certificate
  placeholders, and Grafana provisioning as code (data sources, dashboards,
  alerting).
- **Integration manual** (EN/ES) — end-to-end setup of Grafana against
  ClickHouse.
- **Licensing & cost analysis** (EN/ES) — Grafana OSS vs. commercial editions.

See [grafana-clickhouse-stack/README.md](./grafana-clickhouse-stack/README.md)
for the quick start and full structure.

## How to use this folder

1. Open the folder of the solution you want to deploy.
2. Start with its `README.md`, which serves as the index and quick start.
3. Follow the detailed manual under `docs/` for the complete procedure.

All documents are in Markdown (`.md`) format, so they render directly on
GitHub with no need to download them.

_Maintained by [Onibex](https://github.com/onibex)._
