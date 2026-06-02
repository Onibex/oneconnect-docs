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

The table below outlines key features of the **OneConnect platform**, showcasing out-of-the-box functionalities that support user management, system monitoring, SAP connectivity, and real-time data operations.

| No | Function | Description |
|---|---|---|
| 1 | OneConnect Cloud: Visual Dashboard for Metrics | OneConnect platform incorporates a visual dashboard that displays metrics such as data processed, average API latency, requests, and errors. Users can filter this data based on specific dates. |
| 2 | Warnings, Errors, and Logs Alerts Dashboard | OneConnect platform features a dashboard specifically for warnings, errors, and log alerts, providing an easy way to detect and address specific issues. |
| 3 | Multiple Users Management | OneConnect platform allows multiple users to log in to the platform as users and manage their own SAP Connectors. |
| 4 | Multiple Email Subscriptions for Log Alerts | OneConnect platform allows multiple email subscriptions to be set up for specific workspaces, enabling monitoring and alert notifications. |
| 5 | Admin Panel: Customer Account Management | OneConnect platform features an admin panel that allows the creation of accounts for specific clients and the setup of custom workspaces. Admins can modify, delete, and create customer accounts with ease.<br><br>**Admin Panel Features:**<br>1. **Account Management:** Modify, delete, and create customer accounts.<br>2. **SAP Connector Creation:** Set up specific workspaces for each customer. |
| 6 | Endpoint Configuration to connect with SAP | Once a SAP Connector is created in OneConnect platform, it automatically generates an endpoint configuration that can be seamlessly set up in SAP using the **SM59** transaction. |
| 7 | Log Filters and Download | OneConnect platform provides filters to easily navigate log data by date, level, and category. Users can download logs in Excel format for further analysis. |
| 8 | Data Refresh | Data in OneConnect platform is automatically updated every 10 seconds to reflect new incoming information. |
| 9 | Topics Visualization | You can visualize the topics processed on Kafka in OneConnect platform, including information about possible errors and the timestamp of the last update. |
| 10 | Data SyncHub | You can visualize the consumption of each of the connectors and the number of active data assets per month. |
