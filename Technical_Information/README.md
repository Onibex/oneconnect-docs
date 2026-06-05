# Technical Information

This folder contains the **product-level documentation** for Onibex OneConnect: what it is, how it works, what it includes, and frequently asked questions.

For component-specific technical manuals, see:

- [SAP_Data_Modeler folder](../SAP_Data_Modeler/) for detailed guides on modeling SAP Entities, configuring parameters, and executing data sending.
- [Smart_Gateway folder](../Smart_Gateway/) for deployment guides on Kubernetes across AWS, Azure, GCP, and SAP BTP.

---

## 📚 Manuals in This Folder

### 1. [OneConnect Overview](./01-OneConnect_Overview.md)

What OneConnect is, the problems it solves, and the three pillars of business value it delivers: a clean SAP core, operational intelligence for everyone, and a governed data foundation.

### 2. [Architecture and Components](./02-Architecture_and_Components.md)

A technical breakdown of the three components that make up OneConnect: the Onibex Marketplace, the Connector for SAP (Data Modeler + Smart Gateway), and the Premium Connectors. Includes data flow descriptions and the full technology stack.

### 3. [Onibex Marketplace](./03-Onibex_Marketplace.md)

A deep dive into the 150+ pre-packaged SAP Data Products. Covers the sections of the catalog (Sales, Inventory, Finance, etc.), the difference between Foundational and Business Data Products, and how to extend pre-packaged products with custom tables and CDS Views.

### 4. [Premium Connectors](./04-Premium_Connectors.md)

Technical reference for the Confluent Gold-Verified connectors that stream SAP data into Databricks, Snowflake, and ClickHouse. Covers automatic schema evolution, idempotent writes, CDC support, and OAuth-based security.

### 5. [The 15-Hour Business Value Challenge](./05-15_Hour_Business_Value_Challenge.md)

The zero-risk, hands-on initiative that delivers real-time SAP data into Databricks or Snowflake in just 15 working hours. Covers what is delivered, business outcomes, and who should take the challenge.

### 6. [OneConnect FAQ](./06-OneConnect_FAQ.md)

Common questions about OneConnect, organized by topic: General, Architecture, Data Flow, Onibex Marketplace, Premium Connectors, Security and Compliance, Deployment, and Support.

---

## 🚀 Recommended Reading Path

### If you are evaluating OneConnect

Start here to understand the value proposition and what the product delivers:

1. [OneConnect Overview](./01-OneConnect_Overview.md)
2. [The 15-Hour Business Value Challenge](./05-15_Hour_Business_Value_Challenge.md)
3. [Onibex Marketplace](./03-Onibex_Marketplace.md)
4. [OneConnect FAQ](./06-OneConnect_FAQ.md)

### If you are a technical architect

Start here to understand how OneConnect fits into your enterprise architecture:

1. [Architecture and Components](./02-Architecture_and_Components.md)
2. [Premium Connectors](./04-Premium_Connectors.md)
3. [OneConnect FAQ](./06-OneConnect_FAQ.md) (Architecture and Security sections)
4. Continue to the [Smart_Gateway folder](../Smart_Gateway/) for deployment details.

### If you are an SAP consultant or developer

Start here to understand the modeling and data flow side of OneConnect:

1. [OneConnect Overview](./01-OneConnect_Overview.md)
2. [Onibex Marketplace](./03-Onibex_Marketplace.md)
3. Continue to the [SAP_Data_Modeler folder](../SAP_Data_Modeler/) for hands-on modeling guides.

---

## 🔗 External Resources

- **Website:** [www.onibex.com](https://www.onibex.com)
- **Knowledge Base:** [help.onibex.com](https://help.onibex.com)
- **Email:** contact@onibex.com
