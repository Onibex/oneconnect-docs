# Smart Gateway BYOC. Bring Your Own Cloud
## AWS Deployment Architecture

---

## Overview

The **BYOC (Bring Your Own Cloud)** deployment model places the entire Smart Gateway
stack directly inside the **customer's own AWS account**. Onibex delivers and configures
the deployment; the customer owns and governs the infrastructure.

This model is designed for enterprise customers who require maximum control over their
data pipeline, strict data sovereignty, or internal compliance policies that prevent
data from transiting third-party cloud environments.

---

## Architecture Diagram

<img width="1761" height="982" alt="smartgateway_NT_ALL_onibex-NEW AWS-BYOC" src="https://github.com/user-attachments/assets/172584b6-0c34-45c0-bd90-ce5772f09d01" />

---

## How It Works

The **Transit Gateway** connects **SAP ECC/S4HANA** directly to the **customer's AWS VPC**,
where the Smart Gateway stack is already running. No data leaves the customer's cloud
boundary at any point during extraction or transformation.

### Traffic flow

1. **SAP ECC/S4HANA** attaches to the Transit Gateway via its own **TGW Attachment**
   and initiates an outbound **RFC Type G** connection over **HTTP/HTTPS**.
2. Corporate users can connect into the same TGW through an optional **VPN Site-to-Site**
   tunnel from the corporate data center.
3. The **Transit Gateway** routes SAP traffic directly into the customer's VPC with no
   intermediate Onibex environment involved.
4. Inside the customer's VPC, a **private ALB** receives the traffic and distributes
   load across **two private subnets** in separate Availability Zones, each running a
   **Smart Gateway + Kafka Connect / Onibex Connector** pair.
5. **Customer Applications** within the same VPC can interact with the Smart Gateway
   directly through the ALB on the private network.
6. **Onibex Premium Connectors** push the processed data to the **Target Cloud** via
   **AWS Private Link**.

---

## Components

| Component | Location | Role |
|---|---|---|
| **SAP ECC / S4HANA** | SAP VPC (private subnet) | Data source - initiates RFC Type G connection |
| **TGW Attachment (SAP)** | SAP VPC | Connects SAP to the Transit Gateway |
| **VPN Site-to-Site** | Corporate data center | Optional - corporate user access into TGW |
| **Transit Gateway** | AWS (shared) | Private routing hub between SAP and customer VPC |
| **TGW Attachment (Customer)** | Customer VPC | Connects customer VPC to the Transit Gateway |
| **ALB (private IP)** | Customer VPC - private subnet | Load balancing across AZs; SSL/TLS available |
| **Smart Gateway** | Customer VPC - private subnet | SAP data extraction and transformation |
| **Kafka Connect / Onibex Connector** | Customer VPC - private subnet | Event streaming and target delivery |
| **Customer Applications** | Customer VPC - private subnet | Internal apps with direct ALB access |
| **Private Link** | Customer VPC | Private egress to Target Cloud |
| **Onibex Premium Connectors** | Customer VPC | Downstream integration to Target Cloud |

---

## Notes

- All subnets in the customer VPC are **private** - no component is reachable from
  the internet.
- The ALB operates with a **private IP only**; SSL/TLS termination is available and
  recommended for compliance-sensitive workloads.
