# Installation Guide — SmartGateway Helm Chart (Azure AKS, BYOC)

This Helm Chart deploys SmartGateway on Kubernetes (microservices + MySQL + required configuration).


Optimized for:

1. **AKS (Azure)**

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

## 1) AKS (Azure)

Select your cluster.
<img width="922" height="272" alt="image" src="https://github.com/user-attachments/assets/bbc94370-2212-4a55-8163-9483f051d875" />

Run the deployment by clicking the **Connect** button.
<img width="847" height="206" alt="image" src="https://github.com/user-attachments/assets/54e44c21-e7dc-4323-b56b-e5a4657b9212" />

Next, click on **Open Cloud Shell**.
<img width="944" height="200" alt="image" src="https://github.com/user-attachments/assets/d9990e01-cea2-4b58-b0ad-5f1221b0793e" />

Select the option **Manage files**.
<img width="891" height="197" alt="image" src="https://github.com/user-attachments/assets/3f89ca58-9dac-4d79-a59f-790eeccb1dcb" />

Upload the ZIP folder.
<img width="978" height="66" alt="image" src="https://github.com/user-attachments/assets/8482939e-440d-4bbe-a9e6-c684a1115200" />

Run the following command to extract the ZIP file:

```bash
unzip SmartGatewayHelmChart.zip
cd kafka-aws-test   # folder where Chart.yaml exists
```

---

## 2) Install Strimzi Operator

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

## 3) Deploy Kafka Stack

Deploy the full Kafka stack: Kafka 4.2 in KRaft mode (not compatible with ZooKeeper), Confluent Schema Registry 7.6.0, and AKHQ (Kafka UI).

AKHQ is the web UI used to inspect topics, consumer groups, and messages in the Kafka cluster.

> ℹ️ **Note**
> Deployment order matters. Always follow the two phases below in sequence.

### Phase 1 — Deploy the Kafka Cluster

The `--set loadBalancer.type` flag controls whether AKHQ is accessible internally (within the VNet) or externally (from the internet).

```bash
helm install kafka-platform ./kafka-platform \
  --namespace kafka \
  --values kafka-platform/values.yaml \
  --values kafka-platform/values-azure.yaml \
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

## 4) Install OneConnect (choose your cloud)

> **Note:** The `datasynchub` namespace is created automatically.

Replace `[Docker-token]` with your Docker Hub personal access token (e.g. `dckr_pat_xxxxxxxxxxxx`), then execute this command:

```bash
helm install oneconnect . \
  --namespace oneconnect \
  --create-namespace \
  --values values-azure.yaml \
  --set dockerHub.token=[Docker-token] \
  --set networking.loadBalancer.type=external \
  --timeout 15m
```

> ⚠️ *Note: Don't forget to insert your token in the command.*
<img width="1524" height="681" alt="image" src="https://github.com/user-attachments/assets/e846ee8f-8979-4027-aa5d-0b16aaca1bbc" />

### Verify the deployment

First, verify that the pods are healthy. To verify this, execute the following command:

```bash
# List all pods in the oneconnect namespace
kubectl get pods -n oneconnect

# For additional details (including restart count and current status)
kubectl get pods -n oneconnect -w
```
<img width="881" height="391" alt="image" src="https://github.com/user-attachments/assets/de996791-d9af-4aa7-abd0-ada6a6be2433" />

### Verify the services and get the external IP

```bash
# View all services
kubectl get svc -n oneconnect
```

Look specifically for the **`internal-frontend`** service.
<img width="922" height="250" alt="image" src="https://github.com/user-attachments/assets/2123f8de-535e-48d7-bf69-a68c9206de4a" />

### Access the platform

Once you have the `EXTERNAL-IP`, open it in your browser on port `5050`.

*For example*, if the IP is `172.184.99.191`:
```
http://172.184.99.191:5050
```
<img width="922" height="488" alt="image" src="https://github.com/user-attachments/assets/417c6701-1944-4469-9826-13edeeef0531" />

---

## Upgrade

To update an existing deployment:

```bash
helm upgrade oneconnect . \
  --namespace oneconnect \
  --values values-azure.yaml \
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

> ℹ️ **NOTE:** These same steps are valid for the multi-cloud support functionality in ***AKS (Azure)*** or ***GKE (GCP)***.
