# Smart Gateway SaaS. AWS Architecture Overview

Onibex hosts the Smart Gateway in its own AWS Cloud infrastructure. Two connectivity patterns are available depending on the customer's SAP environment and network topology.

---

## Option A. Secure Internet Connection

> **Best for:** Customers with on-premise SAP or SAP outside of AWS.

<img width="1175" height="774" alt="smartgateway_NT_ALL_onibex-NEW AWS-SasS_Trad" src="https://github.com/user-attachments/assets/1e72e6d0-06aa-4345-9398-46ef6c12b3ec" />

## Overview

In this deployment model, Onibex hosts the Smart Gateway in its own AWS Cloud. SAP
ECC/S4HANA connects directly to Onibex's VPC using **RFC Type G + HTTPS**. The
connection is resolved through a **Fully Qualified Domain Name (FQDN)** with
**SSL/TLS** enforced at the ALB layer.

## How It Works

SAP ECC/S4HANA initiates an outbound **RFC Type G** connection over **HTTPS**
toward Onibex's AWS Cloud using a **Fully Qualified Domain Name**. The connection
originates entirely from SAP with no additional network infrastructure required on
the customer side.

### Traffic flow

1. **SAP ECC/S4HANA** sends data via RFC Type G + HTTPS to Onibex's FQDN.
2. An **ENI** receives the inbound traffic and forwards it to the **ALB**.
3. The **ALB** terminates SSL/TLS and distributes traffic across separate Availability Zones, each pointing to a
   **Smart Gateway + Kafka Connect / Onibex Connector**.
4. **Onibex Premium Connectors** deliver the processed data to the **Target Cloud**
   via **AWS Private Link**.

## Connectivity

SAP communicates with the ALB using a **Fully Qualified Domain Name (FQDN)** or
a **custom domain** over HTTPS. SSL/TLS is terminated at the ALB layer. Onibex
provides the FQDN to be configured in the SAP RFC destination (SM59) during the
onboarding process. A customer-provided domain can be used if required by internal
DNS governance or compliance policy.

| Aspect | Detail |
|---|---|
| Protocol | HTTPS |
| SAP destination | FQDN or custom domain |
| Encryption in transit | Yes (TLS terminated at ALB) |
| Certificate | Onibex managed or customer provided |
| Complexity | Low |


---

## Option B. Transit Gateway

> **Best for:** Enterprise customers on SAP RISE in AWS requiring private routing.

<img width="1651" height="1018" alt="smartgateway_NT_ALL_onibex-NEW AWS-SasS_TGW" src="https://github.com/user-attachments/assets/61ac8712-4d01-4f02-81d6-58e8ebec346b" />

### How it works

The **Transit Gateway** is the single private backbone connecting three parties: **SAP RISE**,
the **customer's AWS account**, and **Onibex's Cloud**. All traffic stays within the AWS network, 
no public internet involved.

1. **SAP RISE** connects to the TGW and sends data to Onibex's VPC via RFC Type G / HTTP/HTTPS.
2. **Onibex** receives the data through its TGW Attachment → ENI → private ALB → Smart Gateway
   + Kafka Connect.
3. **Onibex** then delivers the processed data to the **customer's Target Cloud**
   (Snowflake, Databricks, SAP HANA, ClickHouse, etc.) through Private Link.

The customer's VPC also attaches to the TGW, enabling corporate users to connect, 
optionally via a **VPN Site-to-Site** tunnel, and giving the customer's own applications
access to the same private routing fabric.

### Key characteristics

| Dimension | Detail |
|---|---|
| Network path | AWS backbone only. No public internet |
| SAP connectivity | RFC Type G + HTTP/HTTPS via TGW |
| Customer AWS infra required | TGW attachment + route tables in customer VPC |
| On-premise / non-AWS SAP | Supported via VPN S2S or AWS Direct Connect |
| Security | No public IPs anywhere, NACLs + Security Groups |
| Target cloud egress | AWS Private Link |

---
