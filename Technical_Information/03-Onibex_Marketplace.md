# OneConnect Data Marketplaceplace

## What is the Data Marketplaceplace?

The **One Connect Data Marketplace** is a catalog of **150+ pre-packaged SAP Data Products** structured like a "digital superMarketplace". It eliminates the need to build SAP data extractions from scratch by providing ready-to-use, SAP-verified Data Products that cover the most common business processes.

The Data Marketplace accelerates time-to-value: instead of spending months building integrations, enterprises can pick the Data Products they need from the catalog and have them streaming into Kafka and downstream platforms within hours.

---

## Sections of the Data Marketplace

The catalog is organized by SAP business module, mirroring the structure of an SAP system:

| Section | Description |
|---|---|
| **Sales & Distribution (SD)** | Sales orders, deliveries, billing documents, customer master. |
| **Inventory Management (MM-IM)** | Stock levels, movements, valuations. |
| **Production (PP)** | Production orders, BOMs, work centers, capacities. |
| **Procurement (MM)** | Purchase orders, purchase requisitions, vendor master. |
| **Finance (FI)** | Journal entries, accounts payable, accounts receivable, GL accounts. |
| **Controlling (CO)** | Cost centers, profit centers, internal orders. |
| **Plant Maintenance (PM)** | Maintenance orders, equipment, functional locations. |
| **Extended Warehouse Management (EWM)** | Warehouse tasks, handling units, putaway and picking. |
| **Transportation Management (TM)** | Freight orders, shipment documents, transportation lanes. |

Additional sections cover Quality Management, Human Resources, and other modules.

---

## Foundational vs. Business Data Products

The Data Marketplace distinguishes between two levels of Data Products:

### Foundational Data Products

**SAP transactional documents and master data** in their raw, structured form.

Examples:

- Customer
- Sales Order
- Delivery
- Invoice
- Product
- Purchase Order
- Vendor

Each Foundational Data Product is a precise, governed representation of an SAP business entity, ready to be consumed downstream.

### Business Data Products

**Composed of multiple Foundational Data Products enriched with business logic**, designed to power real-time business use cases.

Examples:

- Real-Time Supply and Demand Inventory Situation
- Sales Performance
- Accounts Receivable Visibility
- Production Monitoring

Business Data Products power multiple outcomes:

- Real-time experiences for eCommerce
- Operational intelligence in CRM
- Predictive analytics and AI in Data Warehouses

---

## What's Included in Each Foundational Data Product

Every pre-packaged Foundational Data Product ships with:

| Component | Description |
|---|---|
| **Excel Documentation** | Lists all tables and CDS Views, with relationships defined (e.g., Sales Order Header, Items, Pricing). |
| **Triggering Event** | Defines the BOR, RAP, BTE, or PPF event that streams data in real time from SAP to the OneConnect Smart Gateway and onward to Kafka. |
| **Customization Hooks** | Customers can extend pre-packaged products by adding custom tables, CDS Views, and fields using the low-code SAP Data Modeler, without coding. |

---

## Example: Sales & Distribution (SD) Pre-Packaged Data Products

The following table shows examples of Foundational Data Products available in the SD section, with their underlying SAP tables and CDS Views:

| Entity | S/4HANA Tables | CDS View Ingredients | ECC Tables | Sync Mode |
|---|---|---|---|---|
| **Sales Order** | VBAK, VBAP, VBPA, VBFA, PRCD_ELEMENTS, VBKD, VBEP, MARD | I_SALESORDER, I_SALESORDERITEM, I_SALESORDERITEMPRICINGELEMENT, I_SALESDOCUMENTITEM, I_SALESDOCUMENTPARTNER, I_SALESORDERPARTNER, I_SALESDOCUMENTSCHEDULELINE | VBAK, VBAP, VBPA, VBFA, KONV, VBKD, VBEP, MARD, VBUP, VBUK | Real-Time |
| **Billing Document** | VBRK, VBRP, VBFA, VBPA, BKPF, BSEG | I_BILLINGDOCUMENT, I_BILLINGDOCUMENTITEM, I_JOURNALENTRY, I_JOURNALENTRYITEM, I_OPERATIONALACCTGDOCITEM, I_BILLINGDOCUMENTITEMBASIC | VBRK, VBRP, VBFA, VBPA, BKPF, BSEG | Real-Time |
| **Customer Master** | KNA1, KNB1, KNVK, KNBK, ADRC, KNKK, etc. | I_CUSTOMER, I_BUSINESSPARTNER, I_BPCONTACT, etc. | KNA1, KNB1, KNVK, ADR6, etc. | Real-Time |
| **Delivery Note** | LIKP, LIPS, VBFA, LQUA, LTAP, SER01, MBEW, MARD | I_DELIVERYDOCUMENT, I_DELIVERYDOCUMENTITEM, etc. | LIKP, LIPS, VBFA, LQUA, LTAP, SER01, MBEW, MARD | Real-Time |

> ℹ️ This is a representative sample. Each section of the Data Marketplace contains similarly detailed Data Products for its respective business domain.

---

## Extending the Data Marketplace

Customers are not limited to the pre-packaged products. The SAP Data Modeler allows full customization:

- Add custom **Z\* tables** to an existing Data Product.
- Add custom **CDS Views** (Standard, Custom, Private, Interface, or Consumption).
- Add or remove specific fields from any table.
- Apply business-friendly aliases for fields and tables.
- Define batch filters to extract subsets of data.

All extensions are **low-code / no-code**, performed through the visual modeler interface, without writing ABAP.

---

## The Governed Data Foundation

Together, these Data Products form a **Democratized, Hyperconnected, Governed Data Foundation**: a single trusted layer where SAP data, IoT signals, customer interactions, and AI analytics converge.

This foundation enables enterprises to stop reinventing integrations for each new use case and instead build new business outcomes on top of reusable, governed Data Products.
