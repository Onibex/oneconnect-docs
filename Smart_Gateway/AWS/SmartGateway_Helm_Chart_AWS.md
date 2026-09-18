# Installation Guide — SmartGateway Helm Chart (AWS EKS, BYOC)

This Helm Chart deploys SmartGateway on Kubernetes (microservices + MySQL + required configuration).

Optimized for:

1. **EKS (AWS)**

---

## Before you start (requirements)

You need:

- **Compressed folder:** `SmartGatewayHelmChart.zip`
- Kubernetes **1.19+**
- Helm **3.0+**
- Cluster access (`kubectl` configured)
- Docker Hub token (**Docker Hub Token will be provided by the Onibex Team**)
- **PersistentVolume provisioner** (MySQL and DataSyncHub)
- Cloud-specific controllers / CSI drivers (LoadBalancer + Storage)

Verify tooling:

```bash
kubectl version --client
helm version
kubectl get nodes
```

---

## 1. EKS (AWS)

Go to your EKS resources and connect to **CloudShell** by selecting the icon as shown in the image.

<img width="922" height="464" alt="image" src="https://github.com/user-attachments/assets/aa3ac3ee-98d6-40a2-aabb-5684e964c0ce" />

Verify your AWS identity. Execute the following command:

```bash
aws sts get-caller-identity
```

<img width="922" height="205" alt="image" src="https://github.com/user-attachments/assets/b7bd35d1-2772-4dfe-ac5a-2e13c039867c" />

List your EKS clusters in the region where they were created.

Run the following command, replacing the region with the one where your cluster is located:

```bash
aws eks list-clusters --region us-east-2
```

<img width="557" height="167" alt="image" src="https://github.com/user-attachments/assets/dd994d50-cd6c-4586-bf40-858e1b6ff392" />

Configure `kubectl` access to your EKS cluster (update kubeconfig).

Run the following command and replace `CLUSTER_NAME` with the name of the cluster you want to select, as shown in the image:

```bash
aws eks update-kubeconfig --region us-east-2 --name CLUSTER_NAME
```

<img width="922" height="100" alt="image" src="https://github.com/user-attachments/assets/8d894883-9933-483e-83f8-2188107bec59" />

### 1.2) Prepare the chart

- Select the option **Manage files**

<img width="922" height="230" alt="image" src="https://github.com/user-attachments/assets/aeffea57-8d9c-408b-b6b6-23b70e7cfa58" />

- Upload the ZIP folder

<img width="713" height="68" alt="image" src="https://github.com/user-attachments/assets/fb10a57b-8be4-40ee-9ce9-c02b4366478f" />

- Run the following command to extract the ZIP file:

```bash
unzip SmartGatewayHelmChart.zip
cd kafka-aws-test   # folder where Chart.yaml exists
```

---

## 2. Install Strimzi Operator

Strimzi is the Kubernetes operator that manages Kafka clusters declaratively. It must be installed and running before any Kafka resources are applied.

### Step 2.1 — Install via Helm

```bash
helm install strimzi-operator strimzi-kafka-operator \
  --repo https://strimzi.io/charts/ \
  --version 0.51.0 \
  --namespace kafka \
  --create-namespace
```

### Step 2.2 — Verify Operator Health

```bash
kubectl get pods -n kafka | grep strimzi-cluster-operator
```

**Expected result:** `strimzi-cluster-operator-xxxx 1/1 Running`

> ℹ️ **Important**
> Do not proceed to Step 3 until the `strimzi-cluster-operator` pod shows `1/1 Running`. The operator must be healthy before it can process Kafka resources.

---

## 3. Deploy Kafka Stack

Deploy the full Kafka stack: Kafka 4.2 in KRaft mode (not compatible with ZooKeeper), Confluent Schema Registry 7.6.0, and AKHQ (Kafka UI).

AKHQ is the web UI used to inspect topics, consumer groups, and messages in the Kafka cluster.

> ℹ️ **Note**
> Deployment order matters. Always follow the two phases below in sequence.

### Phase 1 — Deploy the Kafka Cluster

The `--set loadBalancer.type` flag controls whether AKHQ is accessible internally (within the VPC) or externally (from the internet).

```bash
helm install kafka-platform ./kafka-platform \
  --namespace kafka \
  --values kafka-platform/values.yaml \
  --values kafka-platform/values-aws.yaml \
  --set loadBalancer.type=external
```

### Phase 2 — Wait for Kafka Pods

```bash
kubectl get pods -n kafka -w
```

All pods must show `1/1 Running` before proceeding:

```
kafka-cluster-main-pool-0           1/1   Running
kafka-cluster-entity-operator-xxx   1/1   Running
schema-registry-xxx                 1/1   Running
akhq-xxx                            1/1   Running
```

---

## 4. Install OneConnect (choose your cloud)

> **Note:** The `datasynchub` namespace is created automatically.

Replace `[Docker-token]` with your Docker Hub personal access token (e.g. `dckr_pat_xxxxxxxxxxxx`), then execute this command:

```bash
helm install oneconnect . \
  --namespace oneconnect \
  --create-namespace \
  --values values-aws.yaml \
  --set dockerHub.token=[Docker-token] \
  --set networking.loadBalancer.type=external \
  --timeout 15m
```

> ⚠️ *Note: Don't forget to insert your token in the command.*

<img width="1524" height="681" alt="image" src="https://github.com/user-attachments/assets/429fa9b8-e0ae-491d-ac03-6852baf11989" />

### Verify the deployment

First, verify that the pods are healthy. Execute the following command:

```bash
# List all pods in the oneconnect namespace
kubectl get pods -n oneconnect

# For additional details (including restart count and current status)
kubectl get pods -n oneconnect -w
```

You should see something like:

<img width="922" height="342" alt="image" src="https://github.com/user-attachments/assets/80894d78-aef4-4bd4-9d20-098600e58067" />

### Verify the services and get the external IP

```bash
# View all services
kubectl get svc -n oneconnect
```

Look specifically for the ***`internal-frontend`*** service. You should see something like:

<img width="922" height="189" alt="image" src="https://github.com/user-attachments/assets/cf93ecd8-a899-4c2b-9aed-68a8654b4154" />

### Access the platform

Once you have the `EXTERNAL-IP`, open it in your browser.

*For example*, if the IP is `a33bd3f701d5e442abc9828d940699d7-289592184.us-east-2.elb.amazonaws.com`:
http://a33bd3f701d5e442abc9828d940699d7-289592184.us-east-2.elb.amazonaws.com:5050

<img width="922" height="708" alt="image" src="https://github.com/user-attachments/assets/6eb31cd3-5ea1-422e-a5f0-e4fadd8dbbda" />

![Login screen](./img/11-login-screen.png)

---

## Upgrade

To update an existing deployment:

```bash
helm upgrade oneconnect . \
  --namespace oneconnect \
  --values values-aws.yaml \
  --set dockerHub.token=[Docker-token]
```

---

## Uninstall

```bash
helm uninstall oneconnect --namespace oneconnect
helm uninstall kafka-platform --namespace kafka
helm uninstall strimzi-operator --namespace kafka
```

To also delete the namespaces:

```bash
kubectl delete namespace oneconnect
kubectl delete namespace kafka
kubectl delete namespace datasynchub
```

---

> ℹ️ **NOTE:** These same steps are valid for the multi-cloud support functionality in ***EKS (AWS)*** or ***GKE (GCP)***.
