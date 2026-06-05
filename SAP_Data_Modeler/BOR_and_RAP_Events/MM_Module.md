# MM Module: Real-Time Events Reference

This document lists the **Materials Management (MM)** entities supported by OneConnect for real-time data extraction, along with their underlying SAP tables, CDS Views, and the event mechanisms (BOR and RAP) that trigger the streaming.

When configuring an entity in the SAP Data Modeler, the values below are used in the **Main Customizing Data** section to define the **Change Document Object** and the event that will trigger real-time data transmission.
---

## Entity Reference Table

| Entity | CDS View | Tables | Change Doc Object | BOR Event | RAP Event | Actions |
|---|---|---|---|---|---|---|
| **Consumption Data** |  | `KONH`<br>`KONP` | `MSEG` | `CL_MMIM_MATDOC_EVENT` |  | CRE |
| **Contract** |  |  | `EINKBELEG` | `CL_MM_PUR_WF_OBJECT_CTR` | `R_PURCHASECONTRACTTP` | CRE, UPD, DEL |
| **Goods Receipt** |  | `MKPF`<br>`MSEG`<br>`MBEW`<br>`MARD` | `MSEG` | `BUS2017` |  | CRE, COMP |
| **Inventory Management** |  | `MKPF`<br>`MSEG`<br>`MARA`<br>`MARD`<br>`MBEW`<br>`MBEWH` | `MSEG` | `CL_MMIM_MATDOC_EVENT` |  | CRE |
| **Invoice Verification** |  |  |  | `BUS2081` |  | CRE, UPD, RELEASE, COMP |
| **Material Batch Data** |  |  | `CHARGE` | `BUS1001002` |  | CRE, UPD |
| **Material Master** | `I_PRODUCT`<br>`I_MRPMATERIAL`<br>`I_PRODUCTSALES`<br>`I_PRODUCTPLANTBASIC`<br>`I_PRODUCTVALUATION`<br>`I_PRODUCTUNITSOFMEASURE`<br>`I_MATERIAL`<br>`I_MATERIALSTOCK`<br>`I_BOMASSEMBLY`<br>`I_BATCH` | `MARA`<br>`MVER`<br>`MSTA`<br>`MLGN`<br>`MOFF`<br>`MLAN`<br>`MKAL`<br>`MARC`<br>`MARD`<br>`MVKE`<br>`MBEW`<br>`MARM`<br>`MEAN`<br>`MAKT`<br>`MLGT`<br>`MAST`<br>`MCH1`<br>`MCHA`<br>`MCHB`<br>`EBEW` | `MATERIAL` | `BUS1001006` |  | CRE |
| **Pipeline Handling** |  | `MKPF`<br>`MSEG`<br>`MBEW`<br>`MLGN`<br>`MLGT`<br>`MARD` | `MSEG` | `CL_MMIM_MATDOC_EVENT` |  | CRE |
| **Pricing Conditions** |  |  | `KONDITION` | `CL_SLSPRCGCNDNRECORD_EVENTS` |  | CRE, UPD, SET ACTIVE, POSTING DONE |
| **Purchase Order** | `I_PURCHASEORDER`<br>`I_PURCHASEORDERITEM`<br>`I_PURCHASEORDERSCHEDULELINE`<br>`I_PURORDACCOUNTASSIGNMENT`<br>`I_PURCHASEORDERPARTNER`<br>`I_PURCHASEREQUISITION`<br>`I_PURCHASEREQUISITIONITEM`<br>`I_PURCHASINGDOCUMENTTYPE`<br>`I_PURCHASINGORGANIZATION`<br>`I_PURCHASINGDOCUMENTTYPETEXT`<br>`I_PURCHASINGGROUP` | `EKKO`<br>`EKPO`<br>`EKET`<br>`EKKN`<br>`EKBE`<br>`EKES`<br>`EBKN`<br>`EBAN` | `EINKBELEG` | `BUS2012` | `R_PURCHASEORDERTP` | CRE, UPD, SET ACTIVE |
| **Purchase Requisition** | `I_PURCHASEREQUISITIONITEM`<br>`I_PURCHASEREQUISITIONITEMAPI01`<br>`E_PURCHASEREQUISITIONITEM` | `EBAN`<br>`EBKN`<br>`EBKN` | `BANF` | `BUS2105` | `R_PURCHASEREQUISITIONTP` | CRE, UPD, POSTING START, POSTING DONE |
| **Purchasing Info Record** |  |  | `EINKBELEG` | `CL_MM_PUR_OBJECT_IR` | `R_PURCHASINGINFORECORDTP` | CRE, UPD |
| **Quotation (Vendor)** |  |  | `EINKBELEG` | `BUS2012` |  | CRE, UPD, SET ACTIVE |
| **Request for Quotation (RFQ)** | `I_PURCHASEORDER`<br>`I_PURCHASEORDERITEM`<br>`I_PURCHASEORDERSCHEDULELINE`<br>`I_PURCHASEORDERSCHEDULELINE`<br>`I_PURORDACCOUNTASSIGNMENT`<br>`I_PURORDACCOUNTASSIGNMENT` | `EKKO`<br>`EKPO`<br>`EKET`<br>`EKKN` |  | `BUS2011` |  | CRE |
| **Scheduling Agreement** |  |  |  | `LIKP` | `R_SCHEDGAGRMTHDRTP` | CRE, UPD, DEL |
| **Service Entry Sheet** |  |  | `ENTRYSHEET` | `BUS2091` | `R_SERVICEENTRYSHEETTP` | CRE, COMP |
| **Source List** |  | `EORD`<br>`EINE`<br>`EINA` | `SOURCELIST` | `CL_MM_PUR_OBJECT_SL` |  | CRE, UPD, DEL |
| **Stock Transport Order** |  |  | `EINKBELEG` | `BUS2012` |  | CRE, UPD, SET ACTIVE |
| **Subcontracting Components** |  |  | `MSEG` | `CL_MMIM_MATDOC_EVENT` |  | CRE |
| **Vendor Master** | `I_SUPPLIER`<br>`I_SUPPLIERTOBUSINESSPARTNER`<br>`I_BUSINESSPARTNER`<br>`I_BUSINESSPARTNERBANK`<br>`I_BUSINESSPARTNERADDRESS` | `LFA1`<br>`LFB1`<br>`LFM1`<br>`LFBK`<br>`LFC1`<br>`LFM2`<br>`ADRC`<br>`ADR2`<br>`ADR6` | `KRED` | `LFA1` | `I_BUSINESSPARTNERTP_2` | CRE |

---

## Action Codes

| Code | Meaning |
|---|---|
| **CRE** | Creation |
| **UPD** | Change / Update |
| **DEL** | Delete |
| **RELEASE** | Release |
| **COMP** | Complete |
| **SET ACTIVE** | Activate record / status |
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
