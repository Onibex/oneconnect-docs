# Business Events in SAP: BOR and RAP

## Introduction

A **business event** is a notification that SAP automatically generates when a relevant change occurs on a business object: its creation, modification, or deletion. This notification allows external systems (such as Onibex's Smart Gateway) to find out about that change in real time, without having to constantly query the database to check whether something has changed.

SAP has implemented this concept in two different ways throughout its technological evolution: **BOR events** (the classic approach) and **RAP events** (the modern approach, associated with S/4HANA and the RAP programming model). This document explains, in general terms, what each one is, how they are similar, how they differ, and why both are relevant for data integration.

> 📘 **Note**
> This document is a conceptual introduction. For details on the events available per module (SD, MM, PP, PM, FI, QM, EWM), see the module-specific manuals in this repository.

---

## What is a business object?

Before talking about events, it's necessary to understand what triggers those events in the first place: the **business object**.

A business object is the representation of a real business concept within SAP: a sales order, a material, a customer, a production order, an invoice, etc. A business object doesn't usually live in a single table; it's typically made up of several related tables (for example, header and item tables).

Both BOR and RAP are, at their core, two different ways of modeling and exposing these business objects, each belonging to a different technological generation of SAP.

---

## BOR Events (Business Object Repository)

### What is the BOR?

The **BOR (Business Object Repository)** is SAP's classic repository of business objects, available since very early versions of the system (R/3) and still present in SAP ECC. It's mainly administered through transaction `SWO1`.

Each business object in the BOR is defined with:

- **Attributes**: the fields that describe the object (for example, order number, date, customer)
- **Methods**: operations that can be executed on the object
- **Events**: notifications that are triggered when something relevant happens to the object (creation, change, deletion, release, etc.)

### How are BOR events triggered?

BOR events are strongly tied to SAP's **Workflow** engine (Business Workflow). When a relevant action occurs on an object (for example, a sales order is saved), SAP can trigger an event associated with that business object through:

- **Specific function modules** that explicitly call the event (for example, `SWE_EVENT_CREATE`)
- **Change documents**, which detect changes on certain fields and trigger the corresponding event
- Custom logic added in a **user exit**, **BAdI**, or **enhancement**, when the standard event doesn't cover the required case

These events are logged and can be monitored through transaction `SWEL` (Event Linkage Log), and their trigger configuration (linkage) is managed in transactions such as `SWE2` and `SWE3`.

### Key characteristics of BOR

| Characteristic | Description |
|---|---|
| Underlying technology | Classic ABAP, Business Object Repository |
| Main transaction | `SWO1` |
| Availability | SAP R/3, SAP ECC, and backward compatibility in S/4HANA |
| Tied to | SAP's Workflow engine |
| Trigger mechanism | Function modules, change documents, enhancements |
| Monitoring | `SWEL`, `SWE2`, `SWE3` |

> ⚠️ **Important**
> Not every business object in the BOR has standard events covering everything that's needed (for example, it's common to find a creation event but no change event for certain specific fields). In those cases, the event must be extended through an enhancement or BAdI so it triggers at the right moment.

---

## RAP Events (RESTful ABAP Programming Model)

### What is RAP?

The **RAP (RESTful ABAP Programming Model)** is SAP's modern programming model, introduced with S/4HANA and designed from the ground up for cloud-based architectures, APIs, and CDS views. In RAP, a business object is modeled as a **CDS-based Business Object (BO)**, with its behavior explicitly defined in modern ABAP code.

Unlike the BOR, in RAP the business object isn't just a definition of standalone attributes and methods: it's a fully modeled entity with:

- **Root entity and child entities** (for example, header and items of a document)
- **Behavior definitions**, where the allowed operations (create, update, delete) and their validations are explicitly declared
- **Determinations** and **validations**, logic that runs automatically at certain points in the object's lifecycle

### How are RAP events triggered?

In RAP, business events are declared directly within the business object's **behavior definition**, using the `events` keyword (at the root entity level or at the child entity level). These events are explicitly triggered in the implementation code of the behavior (behavior pool), typically within the methods that handle the business logic (for example, when executing an action or when a save operation completes).

These events can, in turn, integrate natively with:

- **Business Events (Event Mesh / SAP Event Broker)**, allowing the event to flow directly into a messaging channel
- **Enterprise Event Enablement**, S/4HANA's standard framework for exposing business events in a decoupled way to external consumers

### Key characteristics of RAP

| Characteristic | Description |
|---|---|
| Underlying technology | ABAP RESTful Application Programming Model, CDS views |
| Availability | SAP S/4HANA (on-premise and cloud) |
| Tied to | Behavior definitions and behavior pools |
| Trigger mechanism | Explicit event declaration in the behavior definition |
| Native integration | Enterprise Event Enablement, SAP Event Mesh |
| Approach | API-first, oriented toward cloud and decoupled architectures |

---

## Key differences between BOR and RAP

~~~
BOR                                   RAP
---------------------------------     ---------------------------------
Classic technology (R/3 / ECC)        Modern technology (S/4HANA)
Tied to the Workflow engine           Tied to behavior definitions
Events sometimes incomplete           Events explicitly defined
                                       by the BO developer
Requires enhancements/BAdIs           Native integration with Event Mesh
to fill gaps                          and Enterprise Event Enablement
Configured via SWO1/SWE2/SWE3         Configured via behavior
                                       definition and Eclipse (ADT)
~~~

In practical terms, the most relevant difference for integration purposes is this: **BOR relies on the correct event already existing or being built manually**, while **RAP is designed from the start to expose events natively and in a decoupled way**, since that is precisely one of the core principles of the RAP programming model.

---

## Why this matters for integration with Onibex OneConnect

Onibex OneConnect needs to be compatible with both models, because in the real world these coexist:

- Clients still running on **SAP ECC**, where the available mechanism is **BOR**
- Clients that have already migrated to **SAP S/4HANA**, where classic objects (BOR, for backward compatibility) and modern objects (RAP) coexist

That's why, within the **Data Modeler**, when configuring a business event for an entity, it's necessary to identify which underlying technology that business object has available in the source SAP system (BOR or RAP), since this determines:

- Which transactions or tools are used to verify or extend the event (`SWO1`/`SWE2` for BOR, behavior definitions via Eclipse/ADT for RAP)
- Whether the event already exists natively, or requires additional development (enhancement in BOR, or explicit declaration in RAP)
- Which output framework is being used to carry the event to the Smart Gateway

---

## Quick glossary

| Term | Meaning |
|---|---|
| **BOR** | Business Object Repository, SAP's classic repository of business objects |
| **RAP** | RESTful ABAP Programming Model, SAP's modern programming model for S/4HANA |
| **Business object** | Representation of a business concept in SAP (order, customer, invoice, etc.) |
| **Behavior definition** | In RAP, the explicit definition of which operations and events a business object has |
| **Change document** | Classic SAP mechanism for detecting field changes and triggering events (used in BOR) |
| **Event Mesh / Enterprise Event Enablement** | S/4HANA framework for exposing business events in a decoupled way to external systems |
| **SWO1 / SWE2 / SWE3 / SWEL** | Classic transactions for defining, configuring, and monitoring business objects and events in BOR |

---

*General reference document. For details on the events available per module (SD, MM, PP, PM, FI, QM, EWM), see the corresponding module-specific manuals in this repository.*
