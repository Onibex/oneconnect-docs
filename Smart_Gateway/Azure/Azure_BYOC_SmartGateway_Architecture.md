# Smart Gateway BYOC. Bring Your Own Cloud
## Azure Deployment Architecture

---

## Overview

The **BYOC (Bring Your Own Cloud)** deployment model places the entire Smart Gateway
stack directly inside the **customer's own Azure subscription**. Onibex delivers and
configures the deployment; the customer owns and governs the infrastructure.

This model is designed for enterprise customers who require maximum control over their
data pipeline, strict data sovereignty, or internal compliance policies that prevent
data from transiting third-party cloud environments.

---

## Architecture Diagram

<img width="1882" height="941" alt="smartgateway_NT_ALL_onibex-NEW Azure-BYOC" src="https://github.com/user-attachments/assets/60cdf7af-adee-44a8-86b7-3aecfcb9a72a" />

---

## How It Works

**ExpressRoute** connects **SAP ECC/S4HANA** directly to the **customer's Azure VNet**,
where the Smart Gateway stack is already running. No data leaves the customer's cloud
boundary at any point during extraction or transformation.

### Traffic flow

1. **SAP ECC/S4HANA** attaches to ExpressRoute via its own **VNet Peering** and
   initiates an outbound **RFC Type G** connection over **HTTP/HTTPS**.
2. Corporate users can connect into the same ExpressRoute through an optional
   **VPN Gateway** or **ExpressRoute circuit** from the corporate data center.
3. **ExpressRoute** routes SAP traffic directly into the customer's VNet through
   **VNet Peering**, with no intermediate Onibex environment involved.
4. Inside the customer's VNet, a **private ALB** receives the traffic and distributes
   load across two Availability Zones, each pointing to a
   **Smart Gateway + Kafka Connect / Onibex Connector**.
5. **Customer Applications** within the same VNet can interact with the Smart Gateway
   directly through the ALB on the private network.
6. **Onibex Premium Connectors** push the processed data to the **Target Cloud**

---

## Components

| Component | Location | Role |
|---|---|---|
| **SAP ECC / S4HANA** | SAP VNet (private subnet) | Data source. Initiates RFC Type G connection. |
| **VNet Peering (SAP)** | SAP VNet | Connects SAP to ExpressRoute. |
| **VPN Gateway / ExpressRoute** | Corporate data center | Optional. Corporate user access into the cloud environment. |
| **ExpressRoute** | Azure (shared) | Private routing hub between SAP and customer VNet. |
| **VNet Peering (Customer)** | Customer VNet | Connects customer VNet to ExpressRoute. |
| **ALB (private IP)** | Customer VNet, private subnet | Load balancing across Availability Zones. FQDN or Domain SSL available. |
| **Smart Gateway** | Customer VNet, private subnet | SAP data extraction and transformation. |
| **Kafka Connect / Onibex Connector** | Customer VNet, private subnet | Event streaming and target delivery. |
| **Customer Applications** | Customer VNet, private subnet | Internal apps with direct ALB access. |
| **Private Link** | Customer VNet | Private egress to Target Cloud. |
| **Onibex Premium Connectors** | Customer VNet | Downstream integration to Target Cloud. |

---

## Notes

- All subnets in the customer VNet are **private**. No component is reachable from
  the internet.
- The ALB operates with a **private IP only**; **FQDN or Domain SSL/TLS termination**
  is available and recommended for compliance-sensitive workloads.
- The architecture uses **Availability Zones** to distribute the Smart Gateway and
  Kafka Connect components across separate physical data centers within the same
  Azure region for high availability.
