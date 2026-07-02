# Requirements Manual: Multi-Cloud Kubernetes POC

This manual defines the **minimum infrastructure requirements** to deploy a Proof of Concept (POC) on managed Kubernetes across the three major cloud providers.

The POC is designed to validate OneConnect functionality with a compact, cost-efficient Kubernetes cluster. For production deployments, refer to the corresponding cloud-specific deployment guides.

---

## AWS (EKS)

| Component | AWS Service | Specification |
|---|---|---|
| **Worker nodes** | EC2 `m6a.xlarge` | 2 nodes, 4 vCPU / 16 GB RAM each. |
| **Control plane** | Amazon EKS | Managed control plane. |

---

## Azure (AKS)

| Component | Azure Service | Specification |
|---|---|---|
| **Worker nodes** | VM `Standard_D4ds_v5` | 2 nodes, 4 vCPU / 16 GB RAM each. |
| **Control plane** | Azure Kubernetes Service (AKS) | Managed control plane. |

---

## GCP (GKE)

| Component | GCP Service | Specification |
|---|---|---|
| **Worker nodes** | Compute Engine `n2d-standard-4` | 2 nodes, 4 vCPU / 16 GB RAM each. |
| **Control plane** | Google Kubernetes Engine (GKE) | Managed control plane. |

---

## Total Resources per Environment

Regardless of the cloud provider chosen, the POC cluster will provide the following aggregate resources:

| Resource | Value |
|---|---|
| **Total nodes** | 2 |
| **Total vCPUs** | 8 |
| **Total RAM** | 32 GB |
| **Control plane** | Fully managed by the cloud provider (no user administration required). |

---

## Cost Estimation

> ℹ️ For an economic estimate, use each provider's official pricing calculator:
>
> - **AWS Pricing Calculator:** [https://calculator.aws](https://calculator.aws)
> - **Azure Pricing Calculator:** [https://azure.microsoft.com/pricing/calculator](https://azure.microsoft.com/pricing/calculator)
> - **Google Cloud Pricing Calculator:** [https://cloud.google.com/products/calculator](https://cloud.google.com/products/calculator)

---

## Additional Support

For specific data or sizing questions for your scenario, please contact the Onibex team at [contact@onibex.com](mailto:contact@onibex.com).
