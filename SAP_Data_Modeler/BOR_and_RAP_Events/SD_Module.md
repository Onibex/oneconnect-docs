# SD Module: Real-Time Events Reference

This document lists the **Sales and Distribution (SD)** entities supported by OneConnect for real-time data extraction, along with their underlying SAP tables, CDS Views, and the event mechanisms (BOR and RAP) that trigger the streaming.

When configuring an entity in the SAP Data Modeler, the values below are used in the **Main Customizing Data** section to define the **Change Document Object** and the event that will trigger real-time data transmission.

---

## Entity Reference Table

| Entity | CDS View | Tables | Change Doc Object | BOR Event | RAP Event | Actions |
|---|---|---|---|---|---|---|
| **Customer Master** | `I_CUSTOMER`<br>`I_CUSTOMERTOBUSINESSPARTNER`<br>`I_CUSTSALESPARTNERFUNC`<br>`I_BUSINESSPARTNER`<br>`I_BUSINESSPARTNERBANK`<br>`I_BUSINESSPARTNERADDRESS`<br>`I_BPCONTACT`<br>`I_BPCONTACTTOADDRESS`<br>`I_BPCONTACTTOFUNCANDDEPT` | `KNA1`<br>`KNB1`<br>`KNVK`<br>`KNVV`<br>`KNVP`<br>`KNBK`<br>`KNB5`<br>`KNB4`<br>`ADRC`<br>`ADR2`<br>`KNKK`<br>`KNKA`<br>`ADR6` | `DEBI` | `BUS1006` | `I_BUSINESSPARTNERTP_2` | CRE, UPD |
| **Sales Quote** |  |  | `VERKBELEG` | `BUS2031` | `R_SALESQUOTATIONTP` | CRE, UPD, DEL |
| **Sales Order** | `I_SALESORDER`<br>`I_SALESORDERITEM`<br>`I_SALESDOCUMENTPARTNER`<br>`I_SALESORDERITEMPRICINGELEMENT`<br>`I_SALESDOCUMENTITEM`<br>`I_SALESDOCUMENTSCHEDULELINE` | `VBAK`<br>`VBAP`<br>`VBPA`<br>`VBFA`<br>`PRCD_ELEMENTS`<br>`MARD`<br>`FPLA`<br>`VBKD`<br>`VBEP` | `VERKBELEG` | `BUS2032` | `R_SALESORDERTP` | CRE, UPD, DEL |
| **Consignment Fill-up** |  | `MKPF`<br>`MSEG`<br>`MARD` | `VERKBELEG` | `BUS2032` |  | CRE, UPD, DEL |
| **Sales Contract** |  | `VBAK`<br>`VBAP`<br>`VKDFS`<br>`VBEP`<br>`VBKD`<br>`VBPA`<br>`PRCD_ELEMENTS`<br>`FPLA`<br>`VBFA` | `VERKBELEG` | `BUS2034` | `R_SALESCONTRACTTP` | CRE, UPD, DEL |
| **Sales Scheduling Agreement** |  |  | `VERKBELEG` | `BUS2035` | `R_SALESSCHEDGAGRMTTP` | CRE, UPD, DEL |
| **Credit Memo** |  | `VBRK`<br>`VBRP`<br>`BKPF`<br>`BSEG`<br>`VKDFS` | `VERKBELEG` | `BUS2076` |  | CRE, UPD, DEL |
| **Credit Memo Request** |  |  | `VERKBELEG` | `BUS2076` | `R_CREDITMEMOREQUESTTP` | CRE, UPD, DEL |
| **Debit Memo** |  | `VBRK`<br>`VBRP`<br>`BKPF`<br>`BSEG`<br>`VKDFS` | `VERKBELEG` | `BUS2077` |  | CRE, UPD, DEL |
| **Debit Memo Request** |  |  | `VERKBELEG` | `BUS2077` | `R_DEBITMEMOREQUESTTP` | CRE, UPD, DEL |
| **Returns Order** | `I_SALESORDER`<br>`I_SALESORDERITEM`<br>`I_SALESDOCUMENTPARTNER`<br>`I_SALESORDERITEMPRICINGELEMENT`<br>`I_SALESDOCUMENTITEM`<br>`I_SALESDOCUMENTSCHEDULELINE` | `VBAK`<br>`VBAP`<br>`VBPA`<br>`VBFA`<br>`PRCD_ELEMENTS`<br>`MARD`<br>`FPLA`<br>`VBKD`<br>`VBEP` | `VERKBELEG` | `BUS2078` |  | CRE, UPD, DEL |
| **Billing Document** | `I_BILLINGDOCUMENT`<br>`I_BILLINGDOCUMENTITEM`<br>`I_JOURNALENTRY`<br>`I_JOURNALENTRYITEM`<br>`I_OPERATIONALACCTGDOCITEM` | `VBRK`<br>`VBFA`<br>`VBPA`<br>`VBRP`<br>`VKDFS`<br>`BSEG`<br>`BKPF` | `VERKBELEG` | `CL_SD_BIL_BD_EVENT` | `R_BILLINGDOCUMENTTP` | CRE, UPD |
| **Delivery Note** | `I_DELIVERYDOCUMENT`<br>`I_DELIVERYDOCUMENTITEM`<br>`I_DELIVERYDOCUMENTTYPE`<br>`I_DELIVERYDOCUMENTTYPETEXT` | `LIKP`<br>`LIPS`<br>`VBFA`<br>`LQUA`<br>`LTAP`<br>`SER01`<br>`MBEW`<br>`MARD` | `LIEFERUNG` | `LIKP` |  | CRE, UPD, DEL |
| **Free of Charge Delivery** |  |  | `LIEFERUNG` | `LIKP` |  | CRE, UPD, DEL |
| **Invoice List** | `I_BILLINGDOCUMENT`<br>`E_BILLINGDOCUMENT`<br>`I_BILLINGDOCUMENTITEM`<br>`I_PRELIMBILLINGDOCUMENTITEM`<br>`E_BILLINGDUELISTITEM` | `VBRL`<br>`VBRK`<br>`VBRP`<br>`VBFA`<br>`VBPA`<br>`BKPF`<br>`BSEG` | `BILLING` | `VBRK` | `R_BILLINGDOCUMENTTP` | COMP |

---

## Action Codes

| Code | Meaning |
|---|---|
| **CRE** | Creation |
| **UPD** | Change / Update |
| **DEL** | Delete |
| **COMP** | Complete |

---

## Event Type Reference

OneConnect supports two SAP-standard event mechanisms for real-time data extraction:

| Event Type | Available In | Description |
|---|---|---|
| **BOR Event** | SAP ECC and S/4HANA | Classic Business Object Repository event mechanism. |
| **RAP Event** | SAP S/4HANA only | Modern RESTful ABAP Programming event mechanism. |

When both are listed for an entity, you can choose whichever fits your SAP version and architectural preferences. When only one is listed, that is the recommended (or only available) trigger for that entity.

> ℹ️ For an overview of all event mechanisms supported by OneConnect (BOR, RAP, BTE, PPF), see the [Architecture and Components manual](../../Technical_Information/02-Architecture_and_Components.md).
