# PP Module: Real-Time Events Reference

This document lists the **Production Planning (PP)** entities supported by OneConnect for real-time data extraction, along with their underlying SAP tables, CDS Views, and the event mechanisms (BOR and RAP) that trigger the streaming.

When configuring an entity in the SAP Data Modeler, the values below are used in the **Main Customizing Data** section to define the **Change Document Object** and the event that will trigger real-time data transmission.
---

## Entity Reference Table

| Entity | CDS View | Tables | Change Doc Object | BOR Event | RAP Event | Actions |
|---|---|---|---|---|---|---|
| **Bill of Material (BOM)** |  | `STKO`<br>`STPO`<br>`MAST`<br>`STAS`<br>`STZU`<br>`STST` | `AUFTRAG` | `CL_BILLOFMATERIAL_EVENT` |  | CRE, UPD |
| **Production Confirmation** | `I_PRODUCTIONORDER`<br>`I_MATERIALDOCUMENTITEM`<br>`I_PRODUCTIONORDEROPERATION`<br>`I_MAINTENANCEORDER`<br>`I_INTERNALORDER`<br>`I_MFGORDERWITHSTATUS` | `AFKO`<br>`AFFW`<br>`AFRU`<br>`AFVC`<br>`AFPO`<br>`AFIH`<br>`AUFK`<br>`AFRC` |  | `BUS2116` |  |  |
| **Production Order** | `I_ORDER`<br>`I_ORDERITEM`<br>`I_ORDERBASIC`<br>`I_ORDERCOMPONENT`<br>`I_ORDERCONFIRMATION`<br>`I_WORKCENTER` | `AFKO`<br>`AUFK`<br>`AFVC`<br>`AFFH`<br>`AFVV`<br>`AFPO`<br>`AFIH`<br>`RESB`<br>`CRHD`<br>`AFRU`<br>`CRCO` | `AUFTRAG` | `CL_CO_WF_PRODUCTION_ORDER` | `R_PRODUCTIONORDERTP` | CRE, UPD |
| **Production Version** |  |  | `AUFTRAG` | `CL_PP_PROD_VERS_EVENT` |  | CRE, UPD, DEL |
| **Routing** |  |  | `AUFTRAG` | `CL_PP_ROUTING_EVENT` |  | CRE, UPD, DEL |
| **Work Center** | `I_WORKCENTER`<br>`I_WORKCENTERCAPACITY`<br>`I_WORKCENTERTEXT` | `CRHD`<br>`CRTX`<br>`CRCA`<br>`CRHH` | `AUFTRAG` | `CL_PP_WORK_CENTER_EVENT` | `R_WORKCENTERTP` |  |

---

## Action Codes

| Code | Meaning |
|---|---|
| **CRE** | Creation |
| **UPD** | Change / Update |
| **DEL** | Delete |

---

## Event Type Reference

OneConnect supports two SAP-standard event mechanisms for real-time data extraction:

| Event Type | Available In | Description |
|---|---|---|
| **BOR Event** | SAP ECC and S/4HANA | Classic Business Object Repository event mechanism. |
| **RAP Event** | SAP S/4HANA only | Modern RESTful ABAP Programming event mechanism. |

When both are listed for an entity, you can choose whichever fits your SAP version and architectural preferences. When only one is listed, that is the recommended (or only available) trigger for that entity.

> ℹ️ For an overview of all event mechanisms supported by OneConnect (BOR, RAP, BTE, PPF), see the [Architecture and Components manual](../../Technical_Information/02-Architecture_and_Components.md).
