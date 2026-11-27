# OneConnect Architecture and Components

OneConnect is composed of **three integrated components** that work together to extract, transform, and deliver SAP data to downstream cloud platforms in real time.

---

## High-Level Data Flow

The data journey through OneConnect follows this path:

1. **Source:** SAP S/4HANA, SAP ECC, or SAP BW.
2. **Extraction and Modeling:** One Connect Connector for SAP, composed of the **SAP Data Modeler** (low-code / no-code modeling inside SAP) and the **Smart Gateway** (Kubernetes-based engine that translates data into Kafka topics).
3. **Streaming:** Apache Kafka cluster with Confluent Schema Registry, where data is serialized in Avro format.
4. **Delivery:** Premium Kafka Connectors push the data into the target cloud platform.
5. **Destination:** Databricks, Snowflake, or ClickHouse.

The full pipeline runs end to end in real time, propagating every INSERT, UPDATE, and DELETE event from SAP to the destination within seconds.

---

## Component 1: One Connect Data Market

A catalog of **150+ pre-packaged SAP Data Products** structured like a "digital supermarket".

The Data Market provides:

- **Foundational Data Products:** SAP transactional documents and master data, such as Customer, Sales Order, Delivery, Invoice, and Product.
- **Business Data Products:** Composed of multiple Foundational Data Products enriched with business logic, such as Real-Time Supply and Demand Inventory Situation.

For full details, see the [Data Market manual](./03-Onibex_Marketplace.md).

---

## Component 2: One Connect Connector for SAP

The **only technology SAP-verified** to extract real-time data and metadata. Available in the **SAP Store**, ensuring compliance with SAP standards for security, scalability, and licensing.

This component is itself composed of two subcomponents.

### 2.1 SAP Data Modeler

A **low-code / no-code** tool to design SAP Entities (Foundational Data Products) by connecting tables and CDS Views via **Inner Join** or **Left Outer Join** relationships.

It pushes data and metadata to the Smart Gateway in two modes:

| Mode | Description |
|---|---|
| **Batch** | Bulk extraction triggered manually or via scheduled jobs. |
| **Real-Time** | Event-driven streaming using SAP standard mechanisms (BOR, RAP, BTE, PPF). |

It also supports two replication formats:

| Format | Description | Use Case |
|---|---|---|
| **Table Format** | Each SAP table or CDS View becomes a Kafka topic (CDC-like). | Granular, field-level downstream consumption. |
| **Document Format (KDOC)** | Entire SAP documents become nested Kafka topics (IDOC-like). | Aggregated, document-level downstream consumption. |

For full details on the Data Modeler, refer to the [SAP_Data_Modeler folder](../SAP_Data_Modeler/).

### 2.2 Smart Gateway

Built on **Kubernetes**, the Smart Gateway receives data and metadata from the SAP Data Modeler, translates them into **Kafka schemas and topics**, and serializes them in **Apache Avro** format.

Key capabilities:

- Multi-cloud deployment via Helm Chart (AWS EKS, Azure AKS, Google Cloud GKE, SAP BTP).
- Automatic Kafka topic and Schema Registry entry generation.
- Visual dashboard for metrics, errors, and alerts.
- Multi-user management with role-based access.
- Real-time data refresh every 10 seconds.

For full details on the Smart Gateway, refer to the [Smart_Gateway folder](../Smart_Gateway/).

---

## Component 3: One Connect Premium Kafka Connectors

A suite of **Confluent Gold-Verified Kafka Connectors** that deliver SAP data securely and reliably into modern data platforms.

Supported destinations:

| Destination | Use Case |
|---|---|
| **Databricks** | Data lakehouse, AI/ML, analytics. |
| **Snowflake** | Cloud data warehouse, BI, analytics. |
| **ClickHouse** | Real-time OLAP, operational analytics. |

For full details, see the [Premium Kafka Connectors manual](./04-Premium_connectors.md).

---

## Technology Stack

| Layer | Technology |
|---|---|
| **Source** | SAP ECC, S/4HANA (any version), SAP BW |
| **Extraction** | SAP-verified RFC connections (Type G HTTP), BOR, RAP, BTE, PPF |
| **Modeling** | Low-code / no-code visual modeler (transaction `ZONT_ONECM`) |
| **Gateway** | Kubernetes-based microservices - Onibex Smart Gateway |
| **Streaming** | Apache Kafka 4.2 (KRaft mode), Confluent Schema Registry 7.6.0 |
| **Serialization** | Apache Avro |
| **Storage** | MariaDB (operational), persistent volumes |
| **Cloud** | AWS EKS, Azure AKS, Google Cloud GKE, SAP BTP |
| **Destinations** | Databricks, Snowflake, ClickHouse, AND MORE! |

---

## Deployment Architecture

OneConnect is deployed across two main environments:

1. **Inside SAP:** The SAP Data Modeler is installed as an SAP transport package and accessed via transaction `ZONT_ONECM`.
2. **On Kubernetes:** The Smart Gateway, Kafka cluster, Schema Registry, and Kafka Connect components are deployed via Helm Chart on a managed Kubernetes service (EKS, AKS, GKE, or BTP).

Communication between the two environments occurs over an **HTTP RFC destination** (Connection Type G) configured in SAP transaction `SM59`.
