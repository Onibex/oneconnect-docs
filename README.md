<img width="1175" height="431" alt="image" src="https://github.com/user-attachments/assets/53e906b2-d9f2-4633-8392-d48145593fb4" />

# OneConnect

**OneConnect** is a real-time data integration platform that connects SAP systems to modern cloud destinations with minimal latency. It replaces slow, resource-intensive batch jobs with a continuous, event-driven flow of data, so your SAP information becomes instantly available to data lakes, analytics platforms, AI models, and third-party applications, without stressing your SAP core.

This repository is the central documentation hub for the OneConnect suite. It walks you through the platform end to end, from modeling your SAP data all the way to delivering it into your target cloud environment.

---

## The 3 Phases of OneConnect

OneConnect moves your SAP data to its final destination in three simple phases:

~~~
Phase 1: Model  →  Phase 2: Activate  →  Phase 3: Connect
~~~

### Phase 1 — Model: [SAP Data Modeler](./SAP_Data_Modeler)

**Decide what SAP data to extract.** Using a low-code / no-code tool that lives inside your SAP system, you design **SAP Data Products**, business-meaningful entities (like Customer, Sales Order, Delivery, or Invoice) built by joining SAP tables and CDS Views. No ABAP development required.

It also supports the extension of any of the **150+ pre-packaged Foundational Data Products** available in the **One Connect Data Market**, as well as the creation of fully custom entities tailored to your business needs.

📖 Explore the [SAP Data Modeler documentation](./SAP_Data_Modeler) to learn how to install, configure, and operate it.

---

### Phase 2 — Activate: [Smart Gateway](./Smart_Gateway)

**Where your SAP data is transformed into streams the rest of the world can consume.**

Once your entities are defined in the Data Modeler, the **Smart Gateway** takes over. It's a Kubernetes-based engine that receives data and metadata from SAP and translates them into standard streaming formats: **Apache Kafka topics** and **Avro schemas** managed by the **Confluent Schema Registry**.

The Smart Gateway is the bridge between your SAP world and the modern data ecosystem. It handles high-volume, real-time data flows while remaining cloud-agnostic and horizontally scalable.

**Where the Smart Gateway can run:**

- **AWS** (EKS).
- **Azure** (AKS).
- **GCP** (GKE).
- **SAP BTP Kyma**.
- **Docker (any OS with Docker installed)**.

You can also choose between deployment models:

- **SaaS** (Onibex-hosted).
- **BYOC** (Bring Your Own Cloud, deployed in your own account).

If you run into an issue while deploying or activating the Smart Gateway, check the [Troubleshooting guide](./Troubleshooting).

📖 Explore the [Smart Gateway documentation](./Smart_Gateway) for architecture guides, deployment manuals, and configuration references.

---

### Phase 3 — Connect: Kafka Connectors

**Deliver your data to its final destination.** Once your SAP data is flowing through Kafka topics, **Kafka Connectors** consume those topics and write the data into destinations like Databricks, PostgreSQL, Snowflake, ClickHouse, and more.

Onibex offers **Premium Kafka Connectors** that are **Confluent Gold-Verified**, supporting automatic schema evolution, full CDC (INSERT/UPDATE/DELETE), idempotent writes, and OAuth-based security.

📖 Learn more in the [Premium Connectors documentation](./Technical_Information/04-Premium_Connectors.md).

---

## Reference Material

Beyond the three main components, this repository also contains reference material to support your understanding and adoption of OneConnect:

###  [Technical Information](./Technical_Information)

Product-level documentation about OneConnect as a whole, useful for evaluators, architects, and anyone new to the platform. This folder includes:

- **OneConnect Overview:** what OneConnect is, the problems it solves, and the value it delivers.
- **Architecture and Components:** technical breakdown of the platform.
- **Onibex Marketplace:** 150+ pre-packaged SAP Data Products, ready to deploy.
- **Premium Connectors:** deep dive into the Confluent Gold-Verified connectors.
- **The 15-Hour Business Value Challenge:** the zero-risk PoC offer.
- **Frequently Asked Questions.**

###  [Business Solutions](./Business_Solutions)

Ready-to-run solution stacks and reference architectures that turn the data delivered by OneConnect into real business value on the consumption side (visualization, analytics, alerting). This folder includes:

- Deployment artifacts (configuration as code, container definitions).
- Step-by-step integration manuals.
- Cost and licensing analyses.

###  [Troubleshooting](./Troubleshooting)

Diagnostic guides organized by layer: deployment, authentication, Kafka connectivity, Avro serialization, and end-to-end data flow. Start with the [symptom triage table](./Troubleshooting/README.md#-start-here-symptom-triage) to find the relevant guide.

---

## How to Use This Repository

1. Follow the **three phases** above, in order, to go from modeling your SAP data all the way to seeing it land in its destination: [SAP Data Modeler](./SAP_Data_Modeler) → [Smart Gateway](./Smart_Gateway) (including [Using the Smart Gateway](./Smart_Gateway/Using_the_Smart_Gateway) for day-to-day operation) → [Premium Connectors](./Technical_Information/04-Premium_Connectors.md).
2. Use the **Reference Material** section above whenever you need general product information, ready-to-run business solutions, or help troubleshooting an issue.

All documents are in Markdown (`.md`) format and render directly on GitHub. No downloads needed.

---

## Related Solutions

OneConnect is part of the broader Onibex ecosystem. Learn more about our other products:

- **[ASK. Agentic Semantic Knowledge](https://github.com/Onibex/agentic-semantic-knowledge-ask)** — AI-powered semantic querying over your SAP Data Products.
- **[eCommerce 360°](https://github.com/Onibex/ecommerce-docs)** — The B2B/B2C commerce platform natively integrated with SAP.

---

_Maintained by [Onibex](https://github.com/Onibex)._
