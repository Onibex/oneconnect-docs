# How to Set Up High Availability for One Connect for SAP

## Overview

One Connect's Data Modeler transmits data and metadata to the Smart Gateway via an HTTP endpoint configured in SAP transaction **`SM59`** as an RFC destination (Connection Type **G**).

To ensure uninterrupted data flow, One Connect supports a **high availability (HA)** configuration with automatic failover between a primary and secondary Smart Gateway endpoint. If the primary endpoint becomes unavailable, the system automatically redirects all transmissions to the secondary endpoint, with no manual intervention required.

This article walks through how to configure both endpoints and explains the failover behavior.

---

## Prerequisites

Before configuring high availability, ensure you have:

- Access to SAP transaction **`SM59`** (RFC destination management).
- Access to maintain entries in the custom table **`ZONTA_OC_PARAM`**.
- One Connect Data Modeler installed and operational.
- Two Smart Gateway instances deployed and accessible via HTTP.

---

## Step 1: Configure the Primary Endpoint

### 1.1 Create the RFC Destination in SM59

To enable outbound data transmission from SAP, create an RFC destination in transaction **`SM59`** with the following settings:

| Setting | Value |
|---|---|
| **Connection Type** | `G` — HTTP Connection to External Server |
| **Target Host** | Hostname or IP of the primary Smart Gateway |
| **Service / Port** | Port number of the Smart Gateway HTTP service |
| **Path Prefix** | Path as specified in your One Connect deployment |

After creating the destination, validate it using both the **Connection Test** and **HTTP Test** options within `SM59`.

> ⚠️ A valid and active endpoint is required before proceeding.

### 1.2 Register the RFC Destination in ZONTA_OC_PARAM

Create an entry in table **`ZONTA_OC_PARAM`** with the following values:

| Field | Value |
|---|---|
| **Parameter Name** | `RFC_DESTINATION` |
| **Parameter Value** | Name of the RFC destination created in `SM59` |

> ℹ️ This registers the primary endpoint. One Connect will use this destination for all data transmissions during both batch and real-time processes.

---

## Step 2: Configure the Secondary Endpoint (Failover)

### 2.1 Create a Second RFC Destination in SM59

Repeat the same process from **Step 1.1** to create a second RFC destination pointing to your secondary Smart Gateway instance. Validate it using the **Connection Test** and **HTTP Test**.

### 2.2 Register the Secondary Endpoint in ZONTA_OC_PARAM

Create an additional entry in table **`ZONTA_OC_PARAM`**:

| Field | Value |
|---|---|
| **Parameter Name** | `RFC_SECONDARY` |
| **Parameter Value** | Name of the secondary RFC destination created in `SM59` |

---

## Failover Behavior

Once both endpoints are configured, One Connect applies the following processing logic for every data transmission, whether triggered by a batch job or a real-time event:

1. The system attempts to connect to the **primary endpoint** first.
2. If the primary connection fails or returns an unsuccessful response, the system **automatically redirects** the transmission to the **secondary endpoint**.
3. On subsequent executions, if the primary endpoint has recovered, the system **automatically resumes** sending data to the primary endpoint.
4. If **both** endpoints are unavailable, the system raises an error message.

> ✅ **No manual switchover or restart is required.** The failover and recovery process is fully automatic.

---

## Summary

| Parameter | Table | Purpose |
|---|---|---|
| **`RFC_DESTINATION`** | `ZONTA_OC_PARAM` | Primary Smart Gateway endpoint |
| **`RFC_SECONDARY`** | `ZONTA_OC_PARAM` | Secondary Smart Gateway endpoint (failover) |

By configuring both parameters, you ensure higher availability and resilience for your One Connect data transmission processes. In the event of a primary endpoint outage, data delivery continues seamlessly through the secondary endpoint until the primary is restored.
