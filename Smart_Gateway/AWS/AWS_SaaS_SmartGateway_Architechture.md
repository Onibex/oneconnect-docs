# Smart Gateway SaaS. AWS Architecture Overview

Onibex hosts the Smart Gateway in its own AWS Cloud infrastructure. Two connectivity patterns are available depending on the customer's SAP environment and network topology.

---

## Option A. Secure Internet Connection

> **Best for:** Customers with on-premise SAP or SAP outside of AWS.

<img width="1564" height="864" alt="smartgateway_NT_ALL_onibex-NEW AWS-SasS" src="https://github.com/user-attachments/assets/5be3d85b-1ed4-4ae6-980a-f4d53a096465" />

## Overview

In this deployment model, Onibex hosts the Smart Gateway in its own AWS Cloud. SAP
ECC/S4HANA connects directly to Onibex's VPC over the internet using **RFC Type G +
HTTP/HTTPS**. The ALB is configured with a **private IP only**, meaning no
internet-facing endpoint is exposed at any point.

This option supports two connectivity modes depending on the customer's security
requirements: **Private IP only** or **Private IP + SSL/TLS**.

## How It Works

SAP ECC/S4HANA initiates an outbound **RFC Type G** connection over **HTTP/HTTPS**
toward Onibex's AWS Cloud. The connection originates entirely from SAP with no
additional network infrastructure required on the customer side.

### Traffic flow

1. **SAP ECC/S4HANA** sends data via RFC Type G + HTTP/HTTPS toward Onibex's VPC.
2. An **ENI** receives the inbound traffic and forwards it to the **private ALB**.
3. The **ALB** (private IP only, no public endpoint) distributes traffic across
   **two private subnets** in separate Availability Zones, each running a
   **Smart Gateway + Kafka Connect / Onibex Connector** pair.
4. **Onibex Premium Connectors** deliver the processed data to the **Target Cloud**
   via **AWS Private Link**.

## ALB Connectivity Modes

This option supports two modes. The choice depends on the customer's internal
security policy and whether end-to-end encryption is required.

### Mode 1 - Private IP only (HTTP)

SAP communicates with the ALB over plain HTTP. No SSL certificate is required.
The private IP ALB ensures the endpoint is not reachable from the public internet.

| Aspect | Detail |
|---|---|
| Protocol | HTTP |
| Encryption in transit | No |
| Certificate required | No |
| Complexity | Low |
| Best for | Customers where endpoint isolation is sufficient |

### Mode 2 - Private IP + SSL/TLS (HTTPS)

SAP communicates with the ALB over HTTPS. SSL/TLS is terminated at the ALB layer,
adding encryption in transit on top of the private endpoint. This is recommended
for regulated industries or customers with policies requiring end-to-end encryption.

| Aspect | Detail |
|---|---|
| Protocol | HTTPS |
| Encryption in transit | Yes (TLS terminated at ALB) |
| Certificate required | Yes (managed at ALB) |
| Complexity | Low - Medium |
| Best for | Regulated industries, compliance-sensitive workloads |

> Both modes use the same architecture. The only difference is whether SSL/TLS
> is enabled at the ALB layer. No additional infrastructure is required to switch
> between modes.

---

## Components

| Component | Location | Role |
|---|---|---|
| **SAP ECC / S4HANA** | Customer SAP environment (private subnet) | Data source - initiates RFC Type G connection |
| **ENI** | Onibex VPC - private subnet | Private ingress point |
| **ALB (private IP)** | Onibex VPC - private subnet | Load balancing across AZs; SSL/TLS optional |
| **Smart Gateway** | Onibex VPC - private subnet | SAP data extraction and transformation |
| **Kafka Connect / Onibex Connector** | Onibex VPC - private subnet | Event streaming and target delivery |
| **Private Link** | Onibex VPC | Private egress to Target Cloud |
| **Onibex Premium Connectors** | Onibex VPC | Downstream integration to Target Cloud |

---

## Network and Security

| Dimension | Detail |
|---|---|
| Network path | SAP outbound -> Onibex VPC private ALB |
| Internet exposure | None (ALB is private IP only) |
| Public IPs | None assigned to any component |
| ALB endpoint | Private IP only |
| SAP connectivity | RFC Type G + HTTP or HTTPS |
| Encryption in transit | Optional (SSL/TLS at ALB) |
| Customer infra required | None |
| Access control | NACLs + Security Groups per subnet |

---

## Key Characteristics

| Dimension | Detail |
|---|---|
| Smart Gateway hosted by | Onibex |
| SAP environment | On-premise / non-AWS |
| Customer AWS infra required | None |
| Internet exposure | None (ALB private IP only) |
| SSL/TLS | Optional at ALB layer |
| Deployment complexity | Low |

---

## Notes

- This option requires no AWS account or network infrastructure on the customer side.
  SAP only needs outbound connectivity to Onibex's endpoint.
- SSL/TLS mode requires a valid certificate to be provisioned at the ALB. Onibex
  can manage this certificate.
- This is the simplest and fastest onboarding path for customers with on-premise
  or non-AWS SAP environments.

---

## Option B. Transit Gateway

> **Best for:** Enterprise customers on SAP RISE in AWS requiring private routing.

<img width="1771" height="1134" alt="smartgateway_NT_ALL_onibex-AWS-SasS" src="https://github.com/user-attachments/assets/fddb385b-884d-454c-8188-0a814f8ebc3e" />


### How it works

The **Transit Gateway** is the single private backbone connecting three parties: **SAP RISE**,
the **customer's AWS account**, and **Onibex's Cloud**. All traffic stays within the AWS network, 
no public internet involved.

1. **SAP RISE** connects to the TGW and sends data to Onibex's VPC via RFC Type G / HTTP/HTTPS.
2. **Onibex** receives the data through its TGW Attachment → ENI → private ALB → Smart Gateway
   + Kafka Connect (across two AZs).
3. **Onibex** then delivers the processed data to the **customer's Target Cloud**
   (Snowflake, Databricks, SAP HANA, ClickHouse) through Private Link.

The customer's VPC also attaches to the TGW, enabling corporate users to connect, 
optionally via a **VPN Site-to-Site** tunnel, and giving the customer's own applications
access to the same private routing fabric.

### Key characteristics

| Dimension | Detail |
|---|---|
| Network path | AWS backbone only — no public internet |
| SAP connectivity | RFC Type G + HTTP/HTTPS via TGW + VNet Peering |
| Customer AWS infra required | TGW attachment + route tables in customer VPC |
| On-premise / non-AWS SAP | Supported via VPN S2S or AWS Direct Connect |
| Security | No public IPs anywhere, NACLs + Security Groups |
| Target cloud egress | AWS Private Link |

---
