<img width="1175" height="431" alt="image" src="https://github.com/user-attachments/assets/53e906b2-d9f2-4633-8392-d48145593fb4" />

# OneConnect

**OneConnect** is a real-time data integration platform that connects SAP systems to modern cloud destinations with minimal latency. It replaces slow, resource-intensive batch jobs with a continuous, event-driven flow of data, so your SAP information becomes instantly available to data lakes, analytics platforms, AI models, and third-party applications, without stressing your SAP core.

This repository is the central documentation hub for the OneConnect suite. It walks you through the platform end to end, from modeling your SAP data all the way to delivering it into your target cloud environment.

---

## How OneConnect Works: The End-to-End Story

OneConnect data flows through three main stages, each handled by a dedicated component. Read them in order to understand the full journey of your SAP data:

### 1.  [SAP Data Modeler](./SAP_Data_Modeler)

**Where you decide what SAP data to extract and how it should look.**

The SAP Data Modeler is a low-code / no-code tool that lives inside your SAP system. It's the starting point of every OneConnect deployment. From here, you design your **SAP Data Products**, which are business-meaningful entities (like Customer, Sales Order, Delivery, or Invoice) built by joining SAP tables and CDS Views through simple drag-and-drop configuration.

**What you can do with the Data Modeler:**

- Define which SAP entities, tables, and CDS Views are exposed for integration.
- Choose between real-time (event-driven) or batch extraction modes.
- Assign SAP-native event triggers (BOR, RAP, BTE, PPF) to stream data as soon as it changes in SAP.
- Reprocess historical data, monitor logs, and manage variants.

📖 Explore the [SAP Data Modeler documentation](./SAP_Data_Modeler) to learn how to install, configure, and operate it.

---

### 2.  [Smart Gateway](./Smart_Gateway)

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

📖 Explore the [Smart Gateway documentation](./Smart_Gateway) for architecture guides, deployment manuals, and configuration references.

---

### 3. 🔗 Kafka Connectors

**Where your data lands in its final destination.**

Once your SAP data is flowing through Kafka topics inside the Smart Gateway, the final step is delivering it to your target platform. This is where **Kafka Connectors** come in: they consume the Kafka topics and write the data into destinations like Databricks, PostgreSQL, Snowflake, ClickHouse, and more.

Onibex offers **Premium Kafka Connectors** that are **Confluent Gold-Verified**, meaning they meet Confluent's highest standards for reliability, performance, and enterprise readiness. These connectors support:

- **Automatic table creation and schema evolution** in the target system.
- **INSERT, UPDATE, DELETE** operations (full CDC support).
- **Idempotent writes** to avoid duplicates on retries.
- **OAuth-based security** for enterprise authentication.

Supported destinations include **Databricks**, **PostgreSQL** **Snowflake**, **ClickHouse**, and many more.

📖 Kafka Connectors configuration is covered as part of the [Smart Gateway](./Smart_Gateway) deployment.

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

This repository is organized as a step-by-step journey. Follow the folders in order to go from designing your first SAP Data Product all the way to consuming your data in production:

~~~
1. Model  →  2. Activate  →  3. Connect 
~~~

### 1. [SAP Data Modeler](./SAP_Data_Modeler)

Start here to model and prepare your SAP data. Learn how to define entities, configure event triggers, and design SAP Data Products directly inside your SAP system.

### 2. [Smart Gateway Deployment](./Smart_Gateway)

Once your data is modeled, deploy the Smart Gateway to stream it. This folder covers architecture, deployment guides for AWS, Azure, GCP, and BTP Kyma, and the SaaS vs BYOC options.

### 3. [Using the Smart Gateway](./Smart_Gateway/Using_the_Smart_Gateway)

With the Smart Gateway deployed, learn how to operate it day-to-day: manage users, configure SAP Connectors, set up Kafka Connect, add specific connectors to your destinations (Snowflake, Databricks, ClickHouse), and grant access to SAP Links.

### 4. [Technical Information](./Technical_Information)

Refer to this folder anytime you have general questions about the platform, its architecture, its components, or the business value it delivers. It also contains the OneConnect FAQ.

### 5. [Business Solutions](./Business_Solutions)

Once your data is flowing, explore ready-to-run solution stacks and reference architectures that turn OneConnect data into real business value on the consumption side (visualization, analytics, alerting).

### 6. [Troubleshooting](./Troubleshooting)

A reference section rather than a sequential step. The guides are organized by layer — deployment, authentication, connectivity, serialization, and end-to-end data flow — and open with a symptom triage table that points you to the relevant one.

All documents are in Markdown (`.md`) format and render directly on GitHub. No downloads needed.

---

## Related Solutions

OneConnect is part of the broader Onibex ecosystem. Learn more about our other products:

- **[ASK. Agentic Semantic Knowledge](https://github.com/Onibex/agentic-semantic-knowledge-ask)** — AI-powered semantic querying over your SAP Data Products.
- **[eCommerce 360°](https://github.com/Onibex/ecommerce-docs)** — The B2B/B2C commerce platform natively integrated with SAP.

---

_Maintained by [Onibex](https://github.com/Onibex)._
