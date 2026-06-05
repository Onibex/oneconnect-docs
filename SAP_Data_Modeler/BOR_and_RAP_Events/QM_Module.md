# QM Module: Real-Time Events Reference

This document lists the **Quality Management (QM)** entities supported by OneConnect for real-time data extraction, along with their underlying SAP tables, CDS Views, and the event mechanisms (BOR and RAP) that trigger the streaming.

When configuring an entity in the SAP Data Modeler, the values below are used in the **Main Customizing Data** section to define the **Change Document Object** and the event that will trigger real-time data transmission. 

---

## Entity Reference Table

| Entity | CDS View | Tables | Change Doc Object | BOR Event | RAP Event | Actions |
|---|---|---|---|---|---|---|
| **Inspection Lot** |  | `QALS`<br>`QAMV`<br>`QAVE`<br>`QASR`<br>`QMAT`<br>`MSEG`<br>`MKPF`<br>`QAMR` | `MELDUNG` | `BUS2045` | `R_INSPECTIONLOTTP` | CRE, UPD, POSTING START |
| **Quality Certificate** |  |  |  | `BUS2117` |  | CRE, RELEASE, COMP |
| **Quality Notification** | `I_QLTYNOTIFICATION`<br>`I_QUALITYNOTIFICATIONITEM`<br>`I_QLTYNOTIFICATIONTASK`<br>`I_QLTYNOTIFICATIONCAUSE`<br>`I_QLTYNOTIFICATIONACTIVITY` | `QMEL`<br>`QMSM`<br>`QMFE`<br>`QMUR`<br>`QMMA` |  | `BUS2078` | `R_QLTYNOTIFICATIONTP` | CRE, RELEASE, COMP |
| **Stability Study** |  |  | `MELDUNG` | `BUS7051` |  | CRE, RELEASE, COMP |

---

## Action Codes

| Code | Meaning |
|---|---|
| **CRE** | Creation |
| **UPD** | Change / Update |
| **RELEASE** | Release |
| **COMP** | Complete |
| **POSTING START** | Posting process started |

---

## Event Type Reference

OneConnect supports two SAP-standard event mechanisms for real-time data extraction:

| Event Type | Available In | Description |
|---|---|---|
| **BOR Event** | SAP ECC and S/4HANA | Classic Business Object Repository event mechanism. |
| **RAP Event** | SAP S/4HANA only | Modern RESTful ABAP Programming event mechanism. |

When both are listed for an entity, you can choose whichever fits your SAP version and architectural preferences. When only one is listed, that is the recommended (or only available) trigger for that entity.

> ℹ️ For an overview of all event mechanisms supported by OneConnect (BOR, RAP, BTE, PPF), see the [Architecture and Components manual](../../Technical_Information/02-Architecture_and_Components.md).
