# OneConnect Premium Kafka Connectors

## What Are the Premium Kafka Connectors?

The **One Connect Premium Kafka Connectors** are a suite of **Confluent Gold-Verified Kafka Connectors** that deliver SAP data securely and reliably from Kafka into modern enterprise data platforms.

These connectors complete the OneConnect data flow. After data is extracted from SAP by the Data Modeler and translated by the Smart Gateway into Avro-serialized Kafka topics, the Premium Kafka Connectors deliver that data into the target cloud platform with full schema governance and idempotent writes.

Each connector is engineered for enterprise-grade reliability, security, and performance, with cloud-agnostic architecture and full schema governance.

---

## Supported Destinations

| Destination | Primary Use Case |
|---|---|
| **Databricks** | Data lakehouse, AI/ML workloads, advanced analytics, GenAI initiatives. |
| **Snowflake** | Cloud data warehouse, business intelligence, enterprise reporting. |
| **ClickHouse** | Real-time OLAP, operational dashboards, high-throughput analytics. |

---

## Key Features

### Cloud-Agnostic Architecture

The connectors run on any Kubernetes-compatible cloud (AWS, Azure, GCP, SAP BTP), ensuring no vendor lock-in and full portability between cloud providers.

### Table and Column Auto-Creation and Evolution

When new tables or columns appear in the source SAP system, the connectors automatically:

- Create the corresponding target tables in the destination platform.
- Add new columns to existing tables.
- Evolve schemas while maintaining backward compatibility.

> ℹ️ This eliminates manual DDL maintenance when SAP entities are extended.

### Full Support for INSERT, UPDATE, and UPSERT

Unlike many batch-based connectors that only append records, the Premium Kafka Connectors natively support:

| Operation | Behavior |
|---|---|
| **INSERT** | New records are appended to the target table. |
| **UPDATE** | Existing records are updated in place. |
| **DELETE** | Records flagged for deletion are removed from the target. |
| **UPSERT** | Insert or update based on primary key matching. |

This ensures the downstream platform always reflects the true current state of SAP data, not a historical accumulation.

### Idempotent Writes for Data Consistency

The connectors guarantee **idempotent writes**: if the same Kafka message is delivered multiple times (for example, after a retry or recovery), the result in the destination is identical to a single delivery.

> ✅ This eliminates duplicate records during connector restarts, network blips, or partition rebalancing.

### Schema Registry Integration

All connectors integrate natively with the **Confluent Schema Registry**, providing:

- Strong schema enforcement at write time.
- Automatic schema evolution tracking.
- Decoupled producer and consumer schemas with backward and forward compatibility.

### OAuth-Based Security

Authentication uses **OAuth 2.0**, providing:

- Token-based authentication with automatic refresh.
- Integration with enterprise identity providers.
- No hard-coded credentials in connector configurations.

---

## Confluent Gold Verification

The Confluent Gold-Verified status is awarded by Confluent (the company that maintains Apache Kafka) only to connectors that pass rigorous testing for:

- Functional correctness across all CDC operations.
- Performance under high-throughput loads.
- Reliability during failure scenarios.
- Compliance with Kafka Connect best practices.
- Documentation and support readiness.

This verification means enterprises can deploy the OneConnect Premium Connectors with the same confidence as any first-party Confluent integration.

---

## Connector Comparison

| Feature | Databricks | Snowflake | ClickHouse |
|---|---|---|---|
| **Auto table creation** | ✅ | ✅ | ✅ |
| **Auto schema evolution** | ✅ | ✅ | ✅ |
| **INSERT / UPDATE / DELETE** | ✅ | ✅ | ✅ |
| **Idempotent writes** | ✅ | ✅ | ✅ |
| **Confluent Gold-verified** | ✅ | ✅ | ✅ |
| **Schema Registry support** | ✅ | ✅ | ✅ |
| **OAuth security** | ✅ | ✅ | ✅ |

---

## Why This Matters

Traditional SAP-to-cloud integrations often fall short in one or more areas. The table below shows how OneConnect closes those gaps.

| Common Limitation | How OneConnect Solves It |
|---|---|
| Append-only architecture loses update/delete semantics | Native CDC support (INSERT, UPDATE, DELETE) |
| Manual schema migrations break pipelines | Automatic schema evolution |
| Duplicates after retries | Idempotent writes guarantee correctness |
| Vendor lock-in to a specific cloud | Cloud-agnostic Kubernetes deployment |
| Credentials stored in plaintext configs | OAuth-based authentication |

The Premium Kafka Connectors close all of these gaps, providing a production-ready bridge between SAP and the modern cloud data stack.
