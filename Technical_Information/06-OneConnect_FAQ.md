# OneConnect: Frequently Asked Questions

## General

### What is OneConnect?

OneConnect is an enterprise-grade integration platform that extracts SAP data in real time and delivers it as governed Data Products to modern cloud platforms such as Databricks, Snowflake, and ClickHouse. See the [Product Overview manual](./01-product-overview.md) for details.

### Is OneConnect SAP-certified?

Yes. OneConnect is the **only technology SAP-verified** for extracting real-time data and metadata from SAP. It is available in the SAP Store and complies with SAP standards for security, scalability, and licensing.

### What SAP systems does OneConnect support?

OneConnect supports:

- **SAP ECC** (all supported versions)
- **SAP S/4HANA** (any version, on-premise or cloud)
- **SAP BW** (Standard and Custom BW Extractors)
- **SAP RISE on AWS**

### Does OneConnect require ABAP development?

**No.** OneConnect is a **low-code / no-code** platform. All data modeling is performed visually through the SAP Data Modeler (transaction `ZONT_ONECM`) by connecting tables and CDS Views with Inner or Left Outer Joins. No ABAP development is required.

For advanced use cases (e.g., sending custom Z tables programmatically), Onibex provides reusable ABAP templates that customers adapt to their tables. See the [Sending Z Tables manual](../SAP_Data_Modeler/12-sending-z-tables.md).

---

## Architecture

### What are the main components of OneConnect?

OneConnect consists of three components:

1. **One Connect Data marketplace:** Catalog of 150+ pre-packaged SAP Data Products.
2. **One Connect Connector for SAP:** Composed of the SAP Data Modeler and the Smart Gateway.
3. **One Connect Premium Kafka Connectors:** Confluent Gold-Verified connectors to Databricks, Snowflake, and ClickHouse.

See the [Architecture and Components manual](./02-architecture_and_components.md) for the full picture.

### What technology powers the Smart Gateway?

The Smart Gateway is built on **Kubernetes** and uses **Apache Kafka 4.2** (KRaft mode), **Confluent Schema Registry 7.6.0**, and **Apache Avro** serialization. It runs as a set of microservices deployed via Helm Chart.

### What clouds does OneConnect support?

OneConnect is deployable on any major Kubernetes-compatible cloud:

- **AWS EKS** (Elastic Kubernetes Service)
- **Azure AKS** (Azure Kubernetes Service)
- **Google Cloud GKE** (Google Kubernetes Engine)
- **SAP BTP** (Business Technology Platform)

The architecture is cloud-agnostic, with no vendor lock-in.

---

## Data Flow

### Does OneConnect support real-time data?

**Yes.** OneConnect supports both real-time streaming and batch extraction.

Real-time streaming uses SAP standard event mechanisms:

- **BOR** (Business Object Repository)
- **RAP** (RESTful ABAP Programming)
- **BTE** (Business Transaction Events)
- **PPF** (Post Processing Framework)

When a record changes in SAP, the event triggers an immediate push to the Smart Gateway, which streams it to Kafka and onward to the downstream platform.

### Does OneConnect support INSERT, UPDATE, and DELETE?

**Yes.** Unlike append-only batch extractions, OneConnect natively propagates all data change types:

- **INSERT:** New records appear downstream as they are created in SAP.
- **UPDATE:** Existing records are updated in place downstream.
- **DELETE:** Records flagged for deletion in SAP are removed downstream.

This ensures the downstream platform always reflects the true current state of SAP data.

### What is the difference between Table Format and Document Format (KDOC)?

| Format | Description | Use Case |
|---|---|---|
| **Table Format** | Each SAP table becomes a separate Kafka topic (CDC-like). | Granular field-level access, normalized analytics. |
| **Document Format (KDOC)** | Entire SAP documents become nested Kafka topics in JSON, structured like IDOCs. | Document-level processing, denormalized analytics, integration with document-oriented systems. |

Both formats can be used together if needed.

### How often does data refresh?

In real-time mode, data is pushed from SAP to the downstream platform **as the change happens** in SAP. The Smart Gateway dashboard reflects new incoming data every **10 seconds**.

In batch mode, the refresh interval is configured by the customer (hourly, daily, etc.) via scheduled SAP jobs.

---

## Data marketplace

### What is the Data marketplace?

The Data marketplace is a catalog of **150+ pre-packaged SAP Data Products** covering Sales, Inventory, Production, Procurement, Finance, Controlling, Plant Maintenance, Warehouse Management, and Transportation Management. See the [Data marketplace manual](./03-data-marketplace.md).

### What is the difference between a Foundational and a Business Data Product?

| Type | Description | Examples |
|---|---|---|
| **Foundational Data Product** | A precise representation of an SAP business entity. | Customer, Sales Order, Invoice, Product, Vendor. |
| **Business Data Product** | A composition of multiple Foundational Data Products enriched with business logic. | Real-Time Supply and Demand Inventory Situation, Sales Performance, Accounts Receivable Visibility. |

### Can I customize pre-packaged Data Products?

**Yes.** Every pre-packaged Data Product can be extended through the SAP Data Modeler by:

- Adding custom Z\* tables.
- Adding custom CDS Views.
- Including or excluding specific fields.
- Applying business-friendly aliases.
- Defining filters and selection criteria.

All customizations are low-code / no-code.

---

## Premium Kafka Connectors

### What destinations are supported?

The Premium Kafka Connectors deliver SAP data to:

- **Databricks** (data lakehouse, AI/ML)
- **Snowflake** (cloud data warehouse, BI)
- **ClickHouse** (real-time OLAP)

All connectors are **Confluent Gold-Verified**.

### Do the connectors handle schema changes automatically?

**Yes.** When new tables or columns appear in SAP, the connectors automatically:

- Create new tables in the target platform.
- Add new columns to existing tables.
- Evolve schemas while preserving backward compatibility.

This eliminates manual DDL maintenance.

### Are the connectors idempotent?

**Yes.** All connectors guarantee idempotent writes: if the same Kafka message is delivered multiple times, the result is identical to a single delivery. This prevents duplicate records during retries, restarts, or partition rebalancing.

---

## Security and Compliance

### How does OneConnect authenticate?

- **SAP to Smart Gateway:** HTTP RFC connection (Type G) configured in SAP transaction `SM59`, with optional TLS encryption.
- **Smart Gateway to Kafka:** Internal Kubernetes networking with mutual TLS.
- **Kafka to Downstream Platforms:** OAuth 2.0 with automatic token refresh.

### Does OneConnect preserve the SAP Clean Core?

**Yes.** OneConnect operates as an external data layer and does not modify the SAP core. All data extraction uses standard SAP mechanisms (RFC, BOR, RAP, BTE, PPF), and no custom enhancements to standard SAP objects are required.

### Can OneConnect handle authorization restrictions?

**Yes.** OneConnect respects SAP authorization objects. Each entity can be assigned an SAP authorization object, ensuring that only users with the required permissions can modify or delete it.

For CDS Views, OneConnect is compatible with all access control values **except** `#PRIVILEGED_ONLY`. Acceptable values are `#CHECK`, `#NOT_REQUIRED`, and `#MANDATORY`.

---

## Deployment

### How long does it take to deploy OneConnect?

The full installation for a proof of concept can be completed in approximately **10 hours**, allowing customers to test the tool on their premises within a couple of business days.

For a full Business Value delivery (live data flowing into Databricks or Snowflake with pre-packaged Business Data Products), see the [15-Hour Business Value Challenge manual](./05-15_Hour_Business_Value_Challenge.md).

### Does OneConnect support high availability?

**Yes.** OneConnect supports automatic failover between a primary and secondary Smart Gateway endpoint. If the primary becomes unavailable, all transmissions automatically redirect to the secondary, with no manual intervention.
### Can OneConnect run alongside existing SAP integrations?

**Yes.** OneConnect is non-disruptive and runs alongside any existing SAP integrations (BAPIs, IDocs, custom ABAP extractions, third-party tools). It does not interfere with existing data flows.

---

## Support and Contact

### How do I get help with OneConnect?

For technical support, integration questions, or to start the 15-Hour Business Value Challenge, contact the Onibex team:

- **Website:** [www.onibex.com](https://www.onibex.com)
- **Email:** contact@onibex.com
- **Knowledge Base:** [help.onibex.com](https://help.onibex.com)

### Where can I find the latest documentation?

This GitHub repository (`Onibex/OnibexOneConnect`) is the central documentation hub for OneConnect. It contains:

- The [SAP_Data_Modeler folder](../SAP_Data_Modeler/) with detailed manuals for the modeling component.
- The [Smart_Gateway folder](../Smart_Gateway/) with deployment guides for the gateway component.
- This `OneConnect_Overview` folder with product-level documentation, architecture, and FAQs.
