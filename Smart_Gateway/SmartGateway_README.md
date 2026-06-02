# Smart Gateway

Documentation and manuals for **Onibex's Smart Gateway**.

This folder contains step-by-step deployment guides for installing and
configuring the Smart Gateway across the following cloud platforms:

- ☁️ **AWS** — Amazon Web Services
- 🔷 **Azure** — Microsoft Azure
- 🟡 **GCP** — Google Cloud Platform
- 🟦 **BTP** — SAP Business Technology Platform

Each guide walks you through the prerequisites, deployment process, and
initial configuration required to get the Smart Gateway up and running in
the target environment.

# Introduction

The **Onibex One Connect Smart Gateway** is the integration engine of the One Connect suite, built on **Kubernetes** for cloud-native scalability and portability. It receives data and metadata from the SAP Data Modeler and translates them into **Kafka schemas and topics**, serializing the payloads in **Avro** format for downstream consumption.

The Smart Gateway is deployable across all major cloud platforms (**AWS EKS**, **Azure AKS**, **Google Cloud GKE**, and **SAP BTP**), and provides a complete web-based control panel for monitoring, administration, and observability of all connected SAP systems and Kafka topics.

The table below outlines key features of the **Smart Gateway**, showcasing out-of-the-box functionalities that support user management, system monitoring, SAP connectivity, and real-time data operations.

| No | Function | Description |
|---|---|---|
| 1 | Kubernetes-Native Architecture | The Smart Gateway is built on **Kubernetes**, ensuring cloud-native scalability, high availability, and portability across environments. |
| 2 | Multi-Cloud Deployment | Deploys seamlessly on **AWS EKS**, **Azure AKS**, **Google Cloud GKE**, and **SAP BTP** through a unified Helm Chart. |
| 3 | Kafka Topic and Schema Generation | Automatically translates SAP data and metadata received from the Data Modeler into **Kafka topics** and **Schema Registry** entries, ready for downstream consumption. |
| 4 | Avro Serialization | All Kafka payloads are serialized in **Apache Avro** format, ensuring compact, schema-aware messages that are compatible with Confluent and standard Kafka ecosystems. |
| 5 | Visual Dashboard for Metrics | Incorporates a visual dashboard that displays metrics such as data processed, average API latency, requests, and errors. Users can filter this data by specific dates. |
| 6 | Warnings, Errors, and Logs Alerts Dashboard | Features a dedicated dashboard for warnings, errors, and log alerts, providing an easy way to detect and address specific issues in real time. |
| 7 | Multiple Users Management | Allows multiple users to log in to the platform and manage their own SAP Connectors, with role-based access. |
| 8 | Multiple Email Subscriptions for Log Alerts | Multiple email subscriptions can be set up for specific workspaces, enabling proactive monitoring and alert notifications. |
| 9 | Admin Panel: Customer Account Management | Features an admin panel that allows the creation of accounts for specific clients and the setup of custom workspaces. Admins can modify, delete, and create customer accounts with ease.<br><br>**Admin Panel Features:**<br>1. **Account Management:** Modify, delete, and create customer accounts.<br>2. **SAP Connector Creation:** Set up specific workspaces for each customer. |
| 10 | Endpoint Configuration to Connect with SAP | Once a SAP Connector is created, the Smart Gateway automatically generates an endpoint configuration that can be seamlessly set up in SAP using the **SM59** transaction. |
| 11 | Topics Visualization | Visualize the topics processed on Kafka, including information about possible errors and the timestamp of the last update. |
| 12 | Log Filters and Download | Provides filters to easily navigate log data by date, level, and category. Users can download logs in Excel format for further analysis. |
| 13 | Data Refresh | Data is automatically updated every **10 seconds** to reflect new incoming information from the connected SAP systems. |
| 14 | Data SyncHub | Visualize the consumption of each connector and the number of active data assets per month, enabling capacity planning and usage tracking. |
| 15 | Real-Time Inserts, Updates, and Deletes | Propagates all SAP data changes (**INSERT**, **UPDATE**, **DELETE**) downstream in real time, eliminating the reconciliation issues typical of batch-based integrations. |
| 16 | Compatibility with Premium Kafka Connectors | Works natively with the **One Connect Premium Kafka Connectors** (Confluent-Gold Verified) to deliver data into **Databricks**, **Snowflake**, and **ClickHouse**. |
