# FI Module: Real-Time Events Reference

This document lists the **Financial Accounting (FI)** entities supported by OneConnect for real-time data extraction, along with their underlying SAP tables, CDS Views, and the event mechanisms (BOR and RAP) that trigger the streaming.

When configuring an entity in the SAP Data Modeler, the values below are used in the **Main Customizing Data** section to define the **Change Document Object** and the event that will trigger real-time data transmission.

---

## Entity Reference Table

| Entity | CDS View | Tables | Change Doc Object | BOR Event | RAP Event | Actions |
|---|---|---|---|---|---|---|
| **Accounts Payable** |  | `BKPF`<br>`BSEG`<br>`LFA1`<br>`LFB1`<br>`LFM1`<br>`LFC1`<br>`LFBK`<br>`BSIK`<br>`BSAK` |  | `BUS6002`<br>`BUS6003`<br>`BUS6030`<br>`BUS6035` |  | CRE, UPD |
| **Accounts Receivable** | `I_ACCOUNTINGDOCUMENTJOURNAL`<br>`I_JOURNALENTRY`<br>`I_OPERATIONALACCTGDOCITEM`<br>`I_APAROPENITEMHISTORY`<br>`I_ARLINEITEM`<br>`I_COMPANYCODE` | `BKPF`<br>`BSEG`<br>`BSID`<br>`BSAD`<br>`BSET` |  | `BUS6021`<br>`BUS6022`<br>`BUS6035` |  | CRE, UPD |
| **Dunning Notice** |  |  | `DUNNING` | `FIODUNNING` |  | COMP |
| **Financial Statement** |  |  | `SRV_CONF` | `BUS4498`<br>`BUS4499` |  | POSTING START, POSTING DONE |
| **Fixed Asset** |  |  |  | `BUS2040` |  | CRE, UPD |
| **Invoice** | `I_BILLINGDOCUMENT`<br>`E_BILLINGDOCUMENT`<br>`I_BILLINGDOCUMENTITEM`<br>`I_PRELIMBILLINGDOCUMENTITEM`<br>`E_BILLINGDUELISTITEM` | `VBRK`<br>`VBRP`<br>`BKPF`<br>`BSEG`<br>`VKDFS` | `BILLING` | `CL_COLL_PR_EVT_INV_MEM_UPD` |  | COMP |
| **Payment Advice** |  |  | `PAYMEMNT` | `FIOPAYAVIS` |  | COMP |

---

## Action Codes

| Code | Meaning |
|---|---|
| **CRE** | Creation |
| **UPD** | Change / Update |
| **DEL** | Delete |
| **COMP** | Complete |
| **POSTING START** | Posting process started |
| **POSTING DONE** | Posting process finished |

---

## Event Type Reference

OneConnect supports two SAP-standard event mechanisms for real-time data extraction:

| Event Type | Available In | Description |
|---|---|---|
| **BOR Event** | SAP ECC and S/4HANA | Classic Business Object Repository event mechanism. |
| **RAP Event** | SAP S/4HANA only | Modern RESTful ABAP Programming event mechanism. |

When both are listed for an entity, you can choose whichever fits your SAP version and architectural preferences. When only one is listed, that is the recommended (or only available) trigger for that entity.

> ℹ️ For an overview of all event mechanisms supported by OneConnect (BOR, RAP, BTE, PPF), see the [Architecture and Components manual](../../Technical_Information/02-Architecture_and_Components.md).
