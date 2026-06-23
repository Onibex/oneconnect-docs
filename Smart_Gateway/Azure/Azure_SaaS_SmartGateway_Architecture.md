# Smart Gateway SaaS. Azure Architecture Overview

Onibex hosts the Smart Gateway in its own Azure Cloud infrastructure. Two connectivity patterns are available depending on the customer's SAP environment and network topology.

---

## Option A. Secure Internet Connection

> **Best for:** Customers with on-premise SAP or SAP outside of Azure.

<img width="1168" height="753" alt="smartgateway_NT_ALL_onibex-NEW Azure-SaaS Trad" src="https://github.com/user-attachments/assets/b33d8532-758e-4202-8efb-10c1918fce43" />

## Overview

In this deployment model, Onibex hosts the Smart Gateway in its own Azure Cloud. SAP
ECC/S4HANA connects directly to Onibex's VNet using **RFC Type G + HTTPS**. The
connection is resolved through a **Fully Qualified Domain Name (FQDN)** with
**SSL/TLS** enforced at the ALB layer.

## How It Works

SAP ECC/S4HANA initiates an outbound **RFC Type G** connection over **HTTPS**
toward Onibex's Azure Cloud using a **Fully Qualified Domain Name**. The connection
originates entirely from SAP with no additional network infrastructure required on
the customer side.

### Traffic flow

1. **SAP ECC/S4HANA** sends data via RFC Type G + HTTPS to Onibex's FQDN.
2. The **ALB** receives the inbound traffic from the public internet.
3. The **ALB** terminates SSL/TLS and distributes traffic across separate Availability
   Zones, each pointing to a **Smart Gateway + Kafka Connect / Onibex Connector**.
4. **Onibex Premium Connectors** deliver the processed data to the **Target Cloud**
   via **Azure Private Link**.

## Connectivity

SAP communicates with the ALB using a **Fully Qualified Domain Name (FQDN)** or
a **custom domain** over HTTPS. SSL/TLS is terminated at the ALB layer. Onibex
provides the FQDN to be configured in the SAP RFC destination (`SM59`) during the
onboarding process. A customer-provided domain can be used if required by internal
DNS governance or compliance policy.

| Aspect | Detail |
|---|---|
| Protocol | HTTPS |
| SAP destination | FQDN or custom domain |
| Encryption in transit | Yes (TLS terminated at ALB) |
| Certificate | Onibex managed or customer provided |
| Complexity | Low |
