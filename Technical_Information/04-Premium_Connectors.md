# OneConnect Premium Kafka Connectors
 
## What Are the Premium Kafka Connectors?
 
The **OneConnect Premium Kafka Connectors** are a suite of **Confluent Gold Verified Kafka Connectors** that deliver SAP data securely and reliably from Kafka into modern enterprise data platforms.
 
These connectors complete the OneConnect data flow. After data is extracted from SAP by the Data Modeler and translated by the Smart Gateway into Avro serialized Kafka topics, the Premium Kafka Connectors deliver that data into the target platform with full schema governance and idempotent writes.
 
Each connector is engineered for enterprise grade reliability, security, and performance, with cloud agnostic architecture and full schema governance.
 
---
 
## Supported Destinations
 
OneConnect currently ships **15 Premium Kafka Connectors**, covering data lakehouses, cloud warehouses, relational databases, object storage, and generic protocols.
 
| Destination | Connector Class | Type | Primary Use Case |
|---|---|---|---|
| **Snowflake** | OnibexSnowflakeSinkConnector | Sink | Cloud data warehouse, business intelligence, enterprise reporting. |
| **Databricks** | OnibexDataLakeSinkConnector | Sink | Data lakehouse, AI/ML workloads, advanced analytics, GenAI initiatives. |
| **SAP HANA** | OnibexHanaSinkConnector | Sink | Real time replication into SAP HANA for hybrid SAP landscapes. |
| **ClickHouse** | Contact Onibex for connector class | Sink | Real time OLAP, operational dashboards, high throughput analytics. |
| **SQL Server** | JdbcSinkConnector | Sink | Relational reporting and integration with existing SQL Server estates. |
| **PostgreSQL** | JdbcSinkConnector | Sink | Relational reporting and general purpose PostgreSQL integration. |
| **Presto** | JdbcSinkConnector | Sink | Federated SQL analytics across distributed data sources. |
| **DB2** | Db2SinkConnector | Sink | Integration with IBM DB2 estates. |
| **BigQuery** | BigQuerySinkConnector | Sink | Google Cloud data warehouse and analytics. |
| **Amazon S3** | S3SinkConnector | Sink | Object storage, data lake landing zone, archival. |
| **Google Cloud Storage** | GcsSinkConnector | Sink | Object storage and data lake landing zone on GCP. |
| **HTTP** | HttpSinkConnector | Sink | Generic delivery to any HTTP or REST endpoint. |
| **Microsoft Fabric** | KustoSinkConnector | Sink | Real time analytics in Microsoft Fabric via Kusto. |
| **Iceberg** | IcebergSinkConnector | Sink | Open table format for lakehouse architectures. |
| **PostgreSQL Stored Procedure** | PostgresqlStoredProcedureSourceConnector | Source | Trigger PostgreSQL stored procedures from Kafka events. |
 
> ℹ️ All connectors are production ready and available for use today. None are currently in a beta or preview state.
 <img width="1013" height="451" alt="image" src="https://github.com/user-attachments/assets/56aada22-1d06-41fe-941d-2a1d08d31d72" />

---
 
## Key Features
 
### Cloud Agnostic Architecture
 
The connectors run on any Kubernetes compatible cloud (AWS, Azure, GCP, SAP BTP), ensuring no vendor lock in and full portability between cloud providers.
 
### Table and Column Auto Creation and Evolution
 
When new tables or columns appear in the source SAP system, the connectors automatically:
 
- Create the corresponding target tables in the destination platform.
- Add new columns to existing tables.
- Evolve schemas while maintaining backward compatibility.
> ℹ️ This eliminates manual DDL maintenance when SAP entities are extended.
 
### Full Support for INSERT, UPDATE, and UPSERT
 
Unlike many batch based connectors that only append records, the Premium Kafka Connectors natively support:
 
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
### OAuth Based Security
 
Authentication uses **OAuth 2.0**, providing:
 
- Token based authentication with automatic refresh.
- Integration with enterprise identity providers.
- No hard coded credentials in connector configurations.
---
 
## Confluent Gold Verification
 
The Confluent Gold Verified status is awarded by Confluent (the company that maintains Apache Kafka) only to connectors that pass rigorous testing for:
 
- Functional correctness across all CDC operations.
- Performance under high throughput loads.
- Reliability during failure scenarios.
- Compliance with Kafka Connect best practices.
- Documentation and support readiness.
**All 15 OneConnect Premium Connectors carry Confluent Gold Verified status**, meaning enterprises can deploy any of them with the same confidence as any first party Confluent integration.
 
---
 
## Connector Comparison
 
| Feature | All 15 Connectors |
|---|---|
| **Auto table creation** | ✅ |
| **Auto schema evolution** | ✅ |
| **INSERT / UPDATE / DELETE** | ✅ |
| **Idempotent writes** | ✅ |
| **Confluent Gold verified** | ✅ |
| **Schema Registry support** | ✅ |
| **OAuth security** | ✅ |
 
---
 
## Why This Matters
 
Traditional SAP to cloud integrations often fall short in one or more areas. The table below shows how OneConnect closes those gaps.
 
| Common Limitation | How OneConnect Solves It |
|---|---|
| Append only architecture loses update/delete semantics | Native CDC support (INSERT, UPDATE, DELETE) |
| Manual schema migrations break pipelines | Automatic schema evolution |
| Duplicates after retries | Idempotent writes guarantee correctness |
| Vendor lock in to a specific cloud | Cloud agnostic Kubernetes deployment |
| Credentials stored in plaintext configs | OAuth based authentication |
| Limited destination coverage | 15 Gold Verified connectors across warehouses, lakehouses, databases, and object storage |
 
The Premium Kafka Connectors provide a production ready bridge between SAP and the modern cloud data stack, with a destination for nearly every enterprise data platform.
