# Smart Gateway on AWS

This folder contains the **infrastructure deployment guides** for running the Smart Gateway on Amazon EKS: choosing a network/hosting model and installing the Helm chart.

> ℹ️ Looking for how to operate the platform once it's deployed (users, SAP Links, connectors, logs)? See [Using the Smart Gateway](../Using_the_Smart_Gateway/) instead.

---

## 📚 Manuals in This Folder

### Step 1 — Choose who hosts the platform

#### 1. [SaaS Architecture](./AWS_SaaS_SmartGateway_Architecture.md) — *Onibex hosts it*

Onibex-hosted model: the Smart Gateway runs in Onibex's AWS Cloud. Nothing to install on your side. Covers both connectivity options — a secure internet connection (FQDN + HTTPS) for on-premise/non-AWS SAP, and a Transit Gateway option for SAP RISE customers requiring fully private routing.

#### 2. [BYOC Architecture](./AWS_BYOC_SmartGateway_Architecture.md) — *you host it*

Bring Your Own Cloud model: the Smart Gateway stack runs entirely inside the customer's own AWS account, connected to SAP via a private Transit Gateway. Best for customers requiring maximum data sovereignty.

### Step 2 — Install the platform (BYOC only)

#### 3. [Installation Guide](./SmartGateway_Helm_Chart_AWS.md)

Step-by-step installation into your own EKS cluster using **Helm** (the standard package manager for Kubernetes): connecting `kubectl` via CloudShell, installing the Strimzi Kafka operator, deploying the Kafka stack, and installing OneConnect via Helm.

> ⚠️ You only need this guide if you chose **BYOC**. With **SaaS**, Onibex performs this deployment for you — skip straight to [Using the Smart Gateway](../Using_the_Smart_Gateway/).

---

## 🚀 Recommended Reading Path

1. Read **SaaS** and **BYOC Architecture** to decide which hosting and network model fits your customer's requirements.
2. **If you chose BYOC:** follow the **[Installation Guide](./SmartGateway_Helm_Chart_AWS.md)** to perform the actual installation on EKS.
3. Once deployed, continue to [Using the Smart Gateway](../Using_the_Smart_Gateway/) to configure users, SAP Links, and connectors.

---

## 🔗 External Resources

- [Minimum Requirements](../Minimum_Requirements.md) — EKS sizing reference for a Proof of Concept.
- [Using the Smart Gateway](../Using_the_Smart_Gateway/) — day-to-day operation guide, once deployed.
