# Introduction

The **Onibex One Connect SAP Data Modeler** is a **low-code / no-code** tool designed to model SAP Entities (Foundational Data Products) by connecting SAP tables and CDS Views through Inner or Left Outer Joins, without requiring ABAP development or custom coding.

As a core component of the One Connect Connector for SAP, the Data Modeler pushes data and metadata to the Smart Gateway in either **batch** or **real-time** mode, leveraging SAP standard mechanisms such as **BOR**, **RAP**, **BTE**, and **PPF**. It supports the extension of any of the 150+ pre-packaged Foundational Data Products available in the One Connect Data Market, as well as the creation of fully custom entities tailored to specific business requirements.

The table below outlines key features of the **SAP Data Modeler**, showcasing out-of-the-box functionalities that support entity modeling, real-time replication, and seamless integration with SAP environments (ECC, S/4HANA, and BW).

| No | Function | Description |
|---|---|---|
| 1 | Low-Code / No-Code Modeling | Design SAP Entities visually by connecting tables and CDS Views through **Inner Join** or **Left Outer Join** operations, eliminating the need for ABAP coding or custom development. |
| 2 | Direct Data Transmission by Specific Table | Send data from any specified SAP table within various business processes, including the ability to transmit customer data stored in **custom Z\* tables**. |
| 3 | Send CDS View Information | Transmit data from any CDS View created in the HANA database, including both **standard** and **customized** views. |
| 4 | Flexible Table Grouping by Business Entity | Group the necessary tables to extract information from a business entity (such as Material, Order, or Customer), allowing the inclusion of both standard and customized (Z\*) tables. |
| 5 | Customized Field Extraction from Tables | Filter the list of fields extracted from each SAP table, enhancing the efficiency of both data extraction and downstream analysis. |
| 6 | Flexible Integration of Customized (Z\*) Tables | Incorporate customized (Z\*) tables into the data extraction process, ensuring tailored data sets that meet specific business requirements. |
| 7 | Data Extraction Filter by Business Entity | Select from a wide range of filters for each business entity, allowing users to send only the specific data they need. |
| 8 | Business-Friendly Name Alias Customization | Use a ubiquitous language by creating custom aliases for SAP fields and tables, simplifying interpretation for users without SAP expertise across the business. |
| 9 | Table Format Replication | Converts each SAP table or CDS View into an individual Kafka topic in a CDC-like (Change Data Capture) pattern, ideal for granular, field-level downstream consumption. |
| 10 | Document Format Replication (KDOC) | Converts entire SAP documents into nested Kafka topics in an IDOC-like structure. Introduces the **KDOC** concept to send complex and nested data in JSON format based on IDOCs. |
| 11 | Real-Time Data Streaming | Push data and metadata to the Smart Gateway in real time using SAP standard event mechanisms: **BOR**, **RAP**, **BTE**, and **PPF**. |
| 12 | Batch Mode Replication | Supports batch extraction for initial loads, historical synchronization, or scheduled refreshes when real-time streaming is not required. |
| 13 | Real-Time Inserts, Updates, and Deletes | Ensures every connected downstream system reflects the most accurate SAP information by propagating all data changes (**INSERT**, **UPDATE**, **DELETE**) in real time. |
| 14 | Metadata Propagation | Automatically extracts and propagates SAP metadata to the Smart Gateway, where it is translated into Kafka schemas and serialized in **Avro** format. |
| 15 | Send Data from Standard and Custom BW Extractors | Activate a unique tool that retrieves data created by **Standard and Custom BW Extractors** and sends it to One Connect. |
| 16 | Inbound Document Creation in SAP | Built-in interfaces and APIs enable the creation of documents inside SAP from external systems such as **Salesforce**, **VTEX**, and others. |
| 17 | Easy Configuration with Central Transaction | A single, central SAP transaction manages all One Connect customization and execution processes, allowing users to install, configure, and execute One Connect functions from one convenient location. |
| 18 | Friendly Log Extraction | Based on the SAP Log concept, the tool provides a user-friendly log within SAP to validate the data extraction and sending procedures. |
| 19 | Prepackaged SAP Data Operational Model | The prepackaged SAP data model covers more than **100 SAP business processes**, addressing a wide range of tables to ensure thorough and efficient implementation. |
| 20 | Extension of Pre-Packaged Data Products | Extend any of the **150+ pre-packaged Foundational Data Products** from the One Connect Data Market by adding custom tables, CDS Views, and fields, without coding. |
| 21 | Rapid Installation for Proof of Concept | The prepackaged installation for a proof of concept can be completed in approximately **10 hours**, allowing users to test the tool on their premises within a couple of business days. |
| 22 | SAP-Verified Integration | The only SAP-verified technology for extracting real-time data and metadata, available in the **SAP Store** and compliant with SAP standards for security, scalability, and licensing. |
