# SmartGateway Helm Chart — Quick Deployment Guide (AWS)

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

![CloudShell icon](./img/01-cloudshell-icon.png)

Verify your AWS identity. Execute the following command:

```bash
aws sts get-caller-identity
```

![AWS identity verification](./img/02-aws-identity.png)

List your EKS clusters in the region where they were created.

Run the following command, replacing the region with the one where your cluster is located:

```bash
aws eks list-clusters --region us-east-2
```

![List EKS clusters](./img/03-list-clusters.png)

Configure `kubectl` access to your EKS cluster (update kubeconfig).

Run the following command and replace `CLUSTER_NAME` with the name of the cluster you want to select, as shown in the image:

```bash
aws eks update-kubeconfig --region us-east-2 --name CLUSTER_NAME
```

![Update kubeconfig](./img/04-update-kubeconfig.png)

### 1.2) Prepare the chart

- Select the option **Manage files**

![Manage files option](./img/05-manage-files.png)

- Upload the ZIP folder

![Upload ZIP folder](./img/06-upload-zip.png)

- Run the following command to extract the ZIP file:

```bash
unzip SmartGatewayHelmChart.zip
cd helm-deployment   # folder where Chart.yaml exists
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

Deploy the full Kafka stack: Kafka 4.2 in KRaft mode (not compatible with ZooKeeper), Confluent Schema Registry 7.6.0, and Kafka Connect.

Kafka Connect is the component that runs connectors to external databases and platforms such as MySQL and Databricks.

> ℹ️ **Note**
> Deployment order matters. Always follow the three phases below in sequence.

### Phase 1 — Deploy the Kafka Cluster

Replace `[Docker-token]` with your Docker Hub personal access token before running this command.

| Placeholder | Description |
|---|---|
| `[Docker-token]` | Your Docker Hub personal access token (e.g. `dckr_pat_xxxxxxxxxxxx`) |

```bash
helm install kafka-platform ./kafka-platform \
  --namespace kafka \
  --values kafka-platform/values.yaml \
  --set dockerHub.token=[Docker-token]
```

Expose the AKHQ UI through an internal LoadBalancer:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: akhq
  namespace: kafka
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: akhq
  ports:
    - port: 8080
      targetPort: 8080
EOF
```

Wait until all four pods are **Running** before moving to Phase 2:

```bash
kubectl get pods -n kafka -w
```

All four pods must show `1/1 Running`:
kafka-cluster-main-pool-0           1/1   Running
kafka-cluster-main-pool-1           1/1   Running
kafka-cluster-main-pool-2           1/1   Running
kafka-cluster-entity-operator-xxx   1/1   Running

---

## 4. Install (choose your cloud)

> **Note:** The `datasynchub` namespace is created automatically.

Execute this command:

```bash
helm install oneconnect . \
  --namespace oneconnect \
  --create-namespace \
  --values values-aws.yaml \
  --set dockerHub.token=YOUR_TOKEN \
  --set networking.loadBalancer.type=external \
  --timeout 15m
```

> ⚠️ *Note: Don't forget to insert your token in the command.*

![Install command output](./img/07-install-command.png)

### Verify the deployment

First, verify that the pods are healthy. Execute the following command:

```bash
# List all pods in the oneconnect namespace
kubectl get pods -n oneconnect

# For additional details (including restart count and current status)
kubectl get pods -n oneconnect -w
```

You should see something like:

![Pods status](./img/08-pods-status.png)

### Verify the services and get the external IP

```bash
# View all services
kubectl get svc -n oneconnect
```

Look specifically for the ***`internal-frontend`*** service. You should see something like:

![Services list](./img/09-services-list.png)

### Access the platform

Once you have the `EXTERNAL-IP`, open it in your browser.

*For example*, if the IP is `a33bd3f701d5e442abc9828d940699d7-289592184.us-east-2.elb.amazonaws.com`:
http://a33bd3f701d5e442abc9828d940699d7-289592184.us-east-2.elb.amazonaws.com:5050

![Platform access example](./img/10-platform-access.png)

![Login screen](./img/11-login-screen.png)

---

> ℹ️ **NOTE:** These same steps are valid for the multi-cloud support functionality in ***EKS (AWS)*** or ***GKE (GCP)***.
