# Licensing, Cost, and Recommendation Analysis — Grafana on ClickHouse

**Version:** 1.2 · **Language:** English · **Verified as of:** July 2026
**Reference scenario:** approx. 15 users · ClickHouse already deployed and reachable over the network · Goal: fast, low-cost dashboards on top of ClickHouse.

> **Companion document:** the technical implementation (installation with Docker Compose, the ClickHouse data source, TLS/nginx) is detailed in the manual `01_Grafana_ClickHouse_Integration_Manual_EN.md`.

---

## Summary and recommendation

**We recommend self-hosted Grafana OSS, in the same network or region as the existing ClickHouse.**

The analysis rests on three reasons:

1. **The license cost is USD 0**, whether for 15 or 150 users: Grafana OSS is free under AGPLv3, and this is the determining factor.
2. **The ClickHouse connector is official and free.** Grafana Enterprise is not required to connect to ClickHouse. Enterprise only adds value for native SAML, fine-grained RBAC, scheduled PDF reports, and support with an SLA — none of which are required here.
3. **Minimal latency and a private data path.** A Grafana instance in the same network as ClickHouse queries it over a **private IP**, without going out to the Internet. Grafana Cloud is also viable in terms of speed, since it runs on AWS/GCP with only tens of milliseconds of additional latency, so the advantage of being on the same network is real but moderate. The decisive differentiator remains the license cost (USD 0), not speed.

**When to reconsider the recommendation:** if the client contractually requires support with an SLA from Grafana Labs, strict corporate SAML, or automatic scheduled PDF reports → evaluate **Grafana Cloud Pro** (approx. USD 115/mo for 15 users) or **Enterprise** (quote required). See Section 8.

**The number of users (15) does not change the recommendation**; it only makes it possible to attach concrete figures to the Cloud option, which is the only one sensitive to the user count.

---

## 1. Deployment and licensing models

Grafana is offered in three forms. It is important not to confuse "Cloud" with "paid": OSS can be self-hosted for free, and Cloud has a free tier.

| | **Grafana OSS** | **Grafana Cloud** | **Grafana Enterprise (self-managed)** |
|---|---|---|---|
| Where it runs | Own server / EC2 | Grafana Labs SaaS | Own server / EC2 |
| License | AGPLv3, **free** | Free / Pro / Enterprise (Advanced) | Commercial, **paid** |
| Users | **Unlimited, free** | Charged per active user (except 3 free) | License per user/instance |
| Operations (patches, backups, TLS) | On the customer | On Grafana Labs | On the customer |
| Enterprise features (SAML, RBAC, reports) | Not native | Yes (depending on tier) | Yes |
| ClickHouse connector | Yes (free) | Yes (free) | Yes |

**AGPLv3 license (clarification):** since April 2021, the Grafana core has been AGPLv3. This **permits commercial and internal use** and deploying the unmodified binary. The obligation to publish the source code only applies if Grafana is *modified* and *offered as a service to third parties over a network* — which is not the case for internal dashboards. For a deployment of internal dashboards, AGPLv3 represents no practical restriction.

---

## 2. Does ClickHouse require buying Enterprise? — No

The "**premium data sources**" that Grafana reserves for Enterprise are proprietary connectors such as **Splunk, Oracle, ServiceNow, New Relic, Datadog, Snowflake, Dynatrace**, etc. **ClickHouse is not on that list**: its plugin is official, open-source, signed by Grafana Labs, and free, and it works in OSS.

**Conclusion:** the chosen data source (ClickHouse) is 100% compatible with the free edition. This is the point that rules out Enterprise as a technical requirement.

---

## 3. Grafana Cloud costs

Grafana Cloud bills by **consumption** (metrics/logs/traces ingested) **+ per active viewing user**. Official tiers:

| Tier | Base price | What it includes | Retention |
|---|---|---|---|
| **Free** | **USD 0** (forever) | All services with limited usage; **3 active users**; 10k metric series; 50 GB logs/traces/profiles; community support | 14 days |
| **Pro** | **From USD 19/mo** + consumption | Pay-as-you-go on top of Free; 8×5 email support | 13 months metrics / 30 days logs-traces |
| **Enterprise (Advanced)** | **From USD 25,000/yr** commitment | SAML, RBAC, auditing, premium sources, SLA, flexible deployment (public/federal cloud/BYOC) | Custom |

**Key detail:** in a BI deployment that only queries an existing ClickHouse — without ingesting metrics, logs, or traces into Grafana Cloud — the consumption meters (series, GB) barely apply, and the Free tier's 14-day retention limits are irrelevant, since the data and its retention are managed by ClickHouse and not Grafana. In practice, the Cloud cost in this scenario boils down to the per-active-user fee plus the platform charge.

**Per-user fee (Pro):** approximately **USD 8 per month per active viewing user** above the 3 free ones, plus a platform charge of **USD 19 per month** (per market sources verified as of June 2026). Grafana restructured its pricing toward consumption, so **it is advisable to confirm the exact per-user rate in the official calculator** (`grafana.com/pricing`) before issuing a firm quote.

> Cloud's "Free" tier is limited to **3 active users**. With 15 users the Free tier **is not enough**; moving to Pro would be necessary.

> **Connectivity:** if the ClickHouse is reachable from the Internet, Grafana Cloud can connect **directly** to its endpoint (with TLS and a firewall/Security Group restricted to Grafana Cloud's IP ranges), without needing **Private Data Source Connect (PDC)**. If exposing the port is not preferred, PDC is the more secure option. Trade-off to weigh: keeping a database endpoint accessible from the Internet, even if restricted by IP.

---

## 4. Grafana Enterprise costs and the difference between self-managed and Cloud

**Grafana Enterprise (self-managed)** — the commercial edition installed on the customer's own server:

- **Non-public pricing**: quoted through sales. Community reports vary widely: from approx. **USD 4,000–5,000/yr** (median reported by small buyers) up to a list starting point of approx. **USD 25,000/yr**, with negotiable discounts of 20–35% on annual commitments.
- It is a license (per user/instance) that is **added** to the cost of the customer's own infrastructure and operations (server, backups, staff). It does not include hosting.

**Grafana Cloud Enterprise (Advanced)** — the same Enterprise features but on the SaaS:

- Starts from a **minimum commitment of USD 25,000/yr** + consumption. It reduces the operational burden (no servers to manage) but adds pay-as-you-go ingestion costs.

**The cost does vary between Cloud and on-premise:**
- **Self-managed:** the quoted license is added to the customer's own infrastructure and staff; it offers more control in exchange for more operational work.
- **Cloud:** an annual commitment plus consumption, with no infrastructure work; it implies less control, but also less operation.
- For 15 users doing BI on ClickHouse, **neither one is economically justified** versus OSS, unless the specific Enterprise features in Section 5 are required.

---

## 5. OSS vs Enterprise/paid — matrix and pros/cons

### What Enterprise (Cloud Advanced or self-managed) adds over OSS

| Feature | OSS | Enterprise | Relevant for 15 users / ClickHouse? |
|---|---|---|---|
| Visualization, panels, alerting | Complete | Yes | — (OSS already covers it) |
| ClickHouse connector | Yes | Yes | **OSS is sufficient** |
| Unlimited users | Yes | Yes | in favor of OSS |
| SSO OAuth/OIDC (Google, Azure AD, Keycloak) | Yes | Yes | OSS supports it |
| **Native SAML** | (can be emulated with oauth2-proxy) | Yes | Only if it is a strict corporate requirement |
| **Fine-grained RBAC** (custom roles) | (only Viewer/Editor/Admin per folder/team) | Yes | With 15 users, the basic roles usually suffice |
| Per-data-source permissions | No | Yes | Normally unnecessary here |
| **Scheduled PDF reports by email** | No | Yes | **The only real gap** if the client requests them |
| Query caching | No | Yes | Desirable with many users; ClickHouse is already fast |
| Custom branding / white-label | No | Yes | Cosmetic |
| Auditing / usage insights | No | Yes | Only if there are compliance requirements |
| Premium sources (Splunk, Oracle, etc.) | No | Yes | Not applicable (they use ClickHouse) |
| **Support with SLA from Grafana Labs** | (community) | Yes | Depends on the client's risk tolerance |

### Pros and cons

**Grafana OSS**

**Strengths:**
- License cost USD 0; unlimited users.
- Full control, data and deployment in-house (good for data governance).
- Same visualization engine and the same ClickHouse connector as the paid editions.
- The fastest option for on-prem ClickHouse (no hop out to the Internet).

**Limitations:**
- All operations are in-house: patches, backups, TLS, availability.
- No native SAML (OIDC or a proxy is required), no fine-grained RBAC, no scheduled PDF reports.
- Community support, no SLA.

**Grafana Enterprise / paid (Cloud or self-managed)**

**Strengths:**
- SAML, fine-grained RBAC, scheduled PDF reports, auditing, query caching.
- Support with an SLA (vendor backing).
- Cloud removes the operational burden.

**Limitations:**
- Significant cost (thousands to tens of thousands of USD per year); high minimums in Cloud Enterprise (USD 25k/yr).
- Cloud against on-prem ClickHouse requires a tunnel (PDC) and adds network latency.
- Paying for features this type of use mostly does not need.

---

## 6. 15-user scenario — cost comparison

Assumptions: 15 viewing users, on-prem ClickHouse already in place (zero additional cost), BI usage (no telemetry ingestion into Grafana).

### Annual license/subscription cost (software only)

| Option | Calculation | **Annual cost (license)** |
|---|---|---|
| **Self-hosted OSS** | Free, any number of users | **USD 0** |
| **Grafana Cloud Free** | Limited to 3 users → **not enough** for 15 | Not viable |
| **Grafana Cloud Pro** | (USD 19 platform + 12 users × approx. USD 8) × 12 months | **approx. USD 1,380** |
| **Grafana Cloud Advanced** | approx. 15 × USD 55/mo × 12 | approx. USD 9,900 (min. commitment USD 25k/yr) → **≥USD 25,000** |
| **Enterprise self-managed** | Commercial quote | **approx. USD 5,000–25,000+** |

### Estimated total first-year cost (TCO: includes infrastructure and operations)

| Option | License | Infra (server) | Operations / setup | **1st-year TCO (approx.)** |
|---|---|---|---|---|
| **Self-hosted OSS** | USD 0 | approx. USD 200–400 (EC2 t3.medium) | Hours from the team to install and operate | **approx. USD 200–400 + internal hours** |
| **Grafana Cloud Pro** | approx. USD 1,380 | USD 0 (SaaS) | Direct connection to ClickHouse (TLS + Security Group); PDC optional | **approx. USD 1,380 + setup** |
| **Enterprise self-managed** | approx. USD 5,000–25,000+ | approx. USD 200–400 | Setup + operations (internal) | **approx. USD 5,200–25,400+** |

**Conclusion:** for 15 users, OSS is dramatically cheaper on license (USD 0). The real cost of OSS is the **implementation and operations effort** (installation, updates, backups). Cloud Pro is a reasonable alternative (approx. USD 1,380/yr) **only if** the customer prefers not to manage servers; if the ClickHouse is reachable from the Internet, the connection from Cloud is direct (no PDC). Enterprise is not justified except for the specific requirements in Section 5.

---

## 7. Considerations and "hidden" costs of the self-hosted deployment

These items should be quoted separately because they are not Grafana license costs but are real project costs:

| Item | Detail | Typical cost |
|---|---|---|
| **Server** | A modest machine (2 vCPU, 4–8 GB, SSD), e.g. a `t3.medium`–`t3.large` in AWS, in the **same network/region** as the ClickHouse | approx. USD 15–35/mo on-demand; less with savings plans |
| **Domain** | A dedicated subdomain (e.g., `grafana.tu-dominio.com`) | If they already have a domain, USD 0; a new domain approx. USD 10–15/yr |
| **DNS / static IP** | `A` record + Elastic IP in AWS | Associated Elastic IP: no charge; unassociated: minor |
| **SSL certificate** | **nginx (in Docker Compose) with your own certificate** (bring-your-own-cert); alternative: **free Let's Encrypt** with Caddy or certbot | USD 0 |
| **ClickHouse security** | If the endpoint is accessible from the Internet: a restricted firewall/Security Group (never `0.0.0.0/0`), TLS active, and a read-only user; ideally Grafana consumes it over an **internal private network** | USD 0 (configuration) |
| **Config database** | **SQLite** (the default) is sufficient and stable for approx. 15 users; PostgreSQL only if HA is required | USD 0 (SQLite) · RDS approx. USD 15–30/mo only if HA |
| **Backups** | Backup of the `storage` volume (SQLite `grafana.db`) + dashboards as code | Minor |
| **Operations** | Patches, monitoring, updates | Hours from the team (part of the service) |
| **High availability (optional)** | 2nd Grafana instance + shared Postgres + load balancer | Only if the client requires HA |

For 15 internal users, a **single-instance** deployment with **Docker Compose** (Grafana OSS + nginx/TLS, SQLite) on a `t3.medium` EC2 is sufficient and low-cost.

---

## 8. Final recommendation and decision tree

**Recommendation:** **self-hosted Grafana OSS** (deployed with **Docker Compose**: Grafana + nginx TLS + SQLite), co-located with ClickHouse. USD 0 license, maximum speed, full control of the data.

**Reconsider a paid edition only if one of these triggers is met:**

- Does the client **require support with an SLA** from the vendor? → Enterprise or Cloud.
- Do they require **native corporate SAML** (not OIDC)? → Enterprise/Cloud (or emulate it with oauth2-proxy in OSS).
- Do they need **scheduled PDF reports automatically sent by email**? → Enterprise/Cloud (the only real functional gap of OSS for BI).
- Do they require **fine-grained RBAC** or auditing for regulatory compliance? → Enterprise/Cloud.
- Do they **not want to manage servers** at all? → **Grafana Cloud Pro** (approx. USD 115/mo for 15 users). Since ClickHouse is public, the connection is direct (TLS + Security Group to Cloud's ranges); PDC optional if the endpoint should not be exposed.

If **none** apply → **OSS** is the correct answer, and the 15 users confirm it: it is the only option where the user count generates no cost.

> **Reference configuration:** self-hosted Grafana OSS with **Docker Compose** (Grafana + nginx TLS + SQLite) and the ClickHouse data source over HTTP. The details are in the manual `01_Grafana_ClickHouse_Integration_Manual_EN.md`.

---

## 9. Sources and verification notes

- Official Grafana Cloud pricing page (`grafana.com/pricing`): Free USD 0, Pro from USD 19/mo + consumption, Enterprise from USD 25,000/yr.
- Per-active-user fee (approx. USD 8/mo above the 3 free in Pro): market sources verified as of June 2026; confirm in the official calculator given the shift to consumption-based pricing.
- Enterprise-exclusive features (SAML, RBAC, reports, premium sources, caching, branding, usage insights): official Grafana Enterprise documentation.
- ClickHouse plugin: official, open-source, signed by Grafana Labs, free; works in OSS and Cloud.
- Self-managed Enterprise pricing: non-public; ranges based on buyer reports (median approx. USD 4,800/yr) and a list starting point (approx. USD 25k/yr), negotiable.
- AGPLv3 license since April 2021; commercial and internal use permitted.

*All prices and capabilities correspond to July 2026 and may change. The Enterprise figures and the per-user fee are market estimates, not firm quotes; for a contractual number, request a quote from Grafana Labs and validate the official calculator.*
