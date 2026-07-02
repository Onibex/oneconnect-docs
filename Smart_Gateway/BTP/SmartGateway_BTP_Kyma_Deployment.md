## Overview

This manual guides you through the fully private deployment of OneConnect on **SAP BTP Kyma (AWS EKS)**. All access is routed through **SAP Cloud Connector**. No services are exposed to the public internet.

---

## Installation Flow Overview

Follow the steps below in order.

| Step | Section | What you will do |
|---|---|---|
| 0 | Prerequisites | Verify tools, cluster access, and Docker credentials. |
| 1 | Section 1 | Install the Strimzi Kafka Operator. |
| 2 | Section 2 | Deploy the full Kafka Stack. |
| 3 | Section 3 | Deploy the OneConnect Platform (Cloud Connector mode). |
| 4 | Section 4 | Deploy the Observability Stack (monitoring and logging). |
| 5 | Section 5 | Enable Istio Sidecar Injection. |
| 6 | Section 6 | Activate all required Kyma Modules. |
| 7 | Section 7 | Configure SAP Cloud Connector. |
| ✓ | Section 8 | Verification and Health Checks. |

---

## 0. Prerequisites

It is necessary to install **kubectl** and **Helm v3**.

For detailed instructions, please refer to: [How to connect Kubectl to BTP Kyma environment using AWS EC2](https://github.com/Onibex/OnibexOneConnect/blob/main/Smart_Gateway/BTP/Connect_Kubectl_to_Kyma.md).

---

## 1. Install Strimzi Operator

Strimzi is the Kubernetes operator that manages Kafka clusters declaratively. It must be installed and running before any Kafka resources are applied.

### Step 1.1: Install via Helm

~~~bash
helm install strimzi-operator strimzi-kafka-operator \
  --repo https://strimzi.io/charts/ \
  --version 0.51.0 \
  --namespace kafka \
  --create-namespace
~~~

### Step 1.2: Verify Operator Health

~~~bash
kubectl get pods -n kafka | grep strimzi-cluster-operator
~~~

Expected result:

~~~
strimzi-cluster-operator-xxxx   1/1   Running
~~~

> ⚠️ **Important:** Do not proceed to Section 2 until the `strimzi-cluster-operator` pod shows `1/1 Running`. The operator must be healthy before it can process Kafka resources.

---

## 2. Deploy Kafka Stack

Deploy the full Kafka stack: **Kafka 4.2** in KRaft mode (not compatible with ZooKeeper), **Confluent Schema Registry 7.6.0**, and **Kafka Connect**.

Kafka Connect is the component that runs connectors to external databases and platforms such as MySQL and Databricks.

> ℹ️ **Note:** Deployment order matters. Always follow the phases below in sequence.

### Phase 1: Deploy the Kafka Cluster

Replace `[Docker-token]`, `[SUBACCOUNT-ID]`, and `[LOCATION-ID]` with your actual values before running the command.

| Placeholder | Description |
|---|---|
| `[Docker-token]` | Your Docker Hub personal access token (e.g. `dckr_pat_xxxxxxxxxxxx`). |
| `[SUBACCOUNT-ID]` | SAP BTP Subaccount ID. |
| `[LOCATION-ID]` | Cloud Connector Location ID. |

Run the following command:

~~~bash
cd kafka-platform/

helm install kafka-platform ./kafka-platform \
  --namespace kafka \
  --values kafka-platform/values.yaml \
  --set dockerHub.token=[Docker-token] \
  --set akhq.serviceMapping.enabled=true \
  --set akhq.serviceMapping.subaccountId="[SUBACCOUNT-ID]" \
  --set akhq.serviceMapping.locationId="[LOCATION-ID]"
~~~

Create the AKHQ service:

~~~bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: akhq
  namespace: kafka
spec:
  selector:
    app: akhq
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
EOF
~~~

Wait until all four pods are **Running** before moving to the next phase:

~~~bash
kubectl get pods -n kafka -w
~~~

All four pods must show `1/1 Running`:

~~~
kafka-cluster-main-pool-0           1/1   Running
kafka-cluster-main-pool-1           1/1   Running
kafka-cluster-main-pool-2           1/1   Running
kafka-cluster-entity-operator-xxx   1/1   Running
~~~

---

## 3. Deploy OneConnect Platform

Deploy all OneConnect application services in **Cloud Connector mode**. No services are exposed to the public internet, since all access flows through SAP Cloud Connector.

> ⚠️ **Important:** Before running this step, make sure Sections 5 (Istio Sidecar) and 6 (Kyma Modules) are complete. If you have not done those yet, go to Section 5 first and come back here afterward.

### Step 3.1: Install

You will need the three values below. Replace each placeholder with your actual value before running the command:

| Placeholder | Description |
|---|---|
| `[Docker-token]` | Your Docker Hub personal access token. |
| `[LOCATION-ID]` | Cloud Connector Location ID, e.g. `onibex-1`. Must match the value you set when configuring Cloud Connector in Section 7. |
| `[SUBACCOUNT-ID]` | SAP BTP Subaccount ID, e.g. `6c7a9dbc-8590-479e-9cec-69bf38a9a822`. Found in BTP Cockpit → Subaccount overview. |

Run the following command:

~~~bash
cd helm/one_source-helm-deployment-*/

helm install oneconnect . \
  --namespace oneconnect \
  --create-namespace \
  --values values-aws.yaml \
  --set dockerHub.token=[Docker-token] \
  --set cloudConnector.enabled=true \
  --set cloudConnector.locationId="[LOCATION-ID]" \
  --set cloudConnector.subaccountId="[SUBACCOUNT-ID]"
~~~

> ℹ️ **Note:** The `locationId` and `subaccountId` flags are both required. If either is missing, the Helm chart will display a clear error message.

### Step 3.2: Resources Created by This Deployment

| Resource | Name | Details |
|---|---|---|
| ServiceMapping | `apigateway-mapping` | Maps `apigateway.oneconnect.svc.cluster.local:9000`. |
| ServiceMapping | `frontend-mapping` | Maps `frontendoneconnect.oneconnect.svc.cluster.local:5050`. |
| ServiceMapping | `akhq-mapping` | Maps `akhq.kafka.svc.cluster.local:8080`. |
| Frontend Env | `NEXT_PUBLIC_API_BASE` | Automatically set to `http://localhost:9000` (no manual update needed). |
| LoadBalancers | (none) | No LoadBalancer services are created in this mode. |

### Step 3.3: (Optional) Use Custom Service IDs

Service IDs default to `apigateway-oneconnect` and `frontend-oneconnect`. Change them only if your Cloud Connector setup requires different names:

~~~bash
--set cloudConnector.serviceIdApigateway="my-apigateway" \
--set cloudConnector.serviceIdFrontend="my-frontend"
~~~

### Step 3.4: Retrieve ServiceMapping Endpoints

After installation, retrieve the tunnel endpoints. You will need these in Section 7 when configuring the Cloud Connector service channels.

~~~bash
# apigateway endpoint
kubectl get servicemappings.connectivityproxy.sap.com apigateway-mapping \
  -n oneconnect -ojsonpath='{.status.endpoint}'

# frontend endpoint
kubectl get servicemappings.connectivityproxy.sap.com frontend-mapping \
  -n oneconnect -ojsonpath='{.status.endpoint}'
~~~

Result format:

~~~
tcp://cp.c-XXXXX.kyma.ondemand.com:443/[service-id]
~~~

> ℹ️ Save both values, you will need them in Section 7.

---

## 4. Deploy Observability Stack

The Observability Stack provides monitoring, log aggregation, and distributed tracing for the OneConnect platform.

> ℹ️ **Note:** Deploy this after the OneConnect platform (Section 3) is running. The platform's telemetry collector will push data to this stack automatically.

### Step 4.1: Install

~~~bash
cd helm/one_source-oneconnect-observability-*/

helm install oneconnect-observability . \
  --namespace observability \
  --create-namespace \
  --values values-aws.yaml
~~~

### Step 4.2: Verify Pods

~~~bash
kubectl get pods -n observability
~~~

### Step 4.3: Access the Monitoring Dashboard (Local Port-Forward)

~~~bash
kubectl port-forward svc/grafana 3000:3000 -n observability
~~~

Open in your browser: `http://localhost:3000`

---

## 5. Enable Istio Sidecar Injection

The Cloud Connector tunnel uses **Istio mTLS** to authenticate traffic between services. Without the Istio sidecar running inside each pod, the Connectivity Proxy will reject all connections.

Think of the Istio sidecar as a security co-pilot alongside each pod. It handles encrypted routing automatically. After enabling it, each pod will show `2/2 READY` (app container + Istio sidecar).

### Step 5.1: Activate the Istio Module in Kyma

1. Log in to **BTP Cockpit** and navigate to your **Subaccount → Kyma Environment → Dashboard URL**.
2. Go to **Modules → Modify Modules → Add**.
3. Select **istio** and click **Add / Save**.
4. Wait until the module status shows **Ready**.
<img width="1053" height="408" alt="image" src="https://github.com/user-attachments/assets/4ec07b51-8213-4b75-b814-668bb90217eb" />
<img width="1050" height="396" alt="image" src="https://github.com/user-attachments/assets/339d93e9-7769-4f11-82af-b57d2c39f26d" />
<img width="1050" height="212" alt="image" src="https://github.com/user-attachments/assets/0b7d26d9-1784-429c-bf16-7b083830c2c8" />
<img width="1050" height="221" alt="image" src="https://github.com/user-attachments/assets/a4a7d3f3-ec69-4dd3-9ff6-de4c6ea22364" />

> ℹ️ Activating the Istio module in Kyma Dashboard is a key step to enable sidecar injection and ensure mTLS authentication in the Cloud Connector. Each pod displays `2/2 READY` once the application container and the security sidecar are running together.

### Step 5.2: Label the Namespaces for Sidecar Injection

Run for the `oneconnect` namespace:

| Placeholder | Description |
|---|---|
| `Namespace` | Kubernetes namespace, use `oneconnect` or `kafka`. |

~~~bash
kubectl label namespace oneconnect istio-injection=enabled
kubectl rollout restart deployment -n oneconnect
~~~

Wait until all pods show `2/2 READY`:

~~~bash
kubectl get pods -n oneconnect
~~~

> 💡 **Tip:** If pods still show `1/1` after the restart, verify the label was applied:
>
> ~~~bash
> kubectl get namespace oneconnect --show-labels | grep istio
> ~~~

---

## 6. Activate Kyma Modules

The **Connectivity Proxy** manages the secure tunnel from Cloud Connector to your cluster. The **Transparent Proxy** exposes BTP destinations as internal Kubernetes services. Both components depend on Istio (Section 5).

### Step 6.1: Activate All Five Required Modules

In **Kyma Dashboard → Modules → Modify Modules → Add**, add all five modules listed below:

| Module | Namespace | Purpose |
|---|---|---|
| `api-gateway` | (default) | API gateway for Kyma services. |
| `btp-operator` | `kyma-system` | SAP BTP service operator. |
| `connectivity-proxy` | `kyma-system` | Secure tunnel between Cloud Connector and the cluster. |
| `istio` | `kyma-system` | Service mesh and mTLS (activated in Section 5). |
| `transparent-proxy` | `sap-transp-proxy-system` | Exposes BTP destinations as Kubernetes services. |

> ⚠️ **Important:** All five modules must reach **Ready** status before you continue. If any module is missing, the Cloud Connector tunnel will not work.

### Step 6.2: Verify All Modules are Ready

~~~bash
# Transparent Proxy namespace must exist
kubectl get ns sap-transp-proxy-system

# Transparent Proxy pods must be Running
kubectl -n sap-transp-proxy-system get deploy,pod

# Connectivity Proxy must be Running
kubectl get pods -n kyma-system | grep connectivity
~~~

---

## 7. Configure SAP Cloud Connector

The **SAP Cloud Connector** is an agent you install in your on-premise network. It opens a secure outbound tunnel to SAP BTP. No inbound firewall ports are required.

**Traffic flow:**

~~~
SAP ECC / S4HANA
  → http://[cc-server-host]:[local-port]
  → Cloud Connector (Service Channel)
  → Connectivity Proxy (kyma-system)
  → service.oneconnect.svc.cluster.local:[port]
~~~

### Step 7.1: Create the Connectivity Service Instance and Binding

The Connectivity Proxy needs BTP credentials. Run the command below to create them.

> ⚠️ **Important:** Use `kubectl`. Do **NOT** use the BTP Cockpit Service Marketplace for this step.

| Placeholder | Description |
|---|---|
| `[NAMESPACE]` | Kubernetes namespace. Use `oneconnect` or `kafka`. |

~~~bash
kubectl apply -f - <<EOF
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: connectivity-instance
  namespace: oneconnect
spec:
  serviceOfferingName: connectivity
  servicePlanName: connectivity_proxy
---
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: connectivity-binding
  namespace: oneconnect
spec:
  serviceInstanceName: connectivity-instance
EOF
~~~

Verify both reach **Ready / Created** status:

~~~bash
kubectl get serviceinstances,servicebindings -n oneconnect
~~~

### Step 7.2: Retrieve ServiceMapping Endpoints

If you already ran Step 3.4, skip this. Otherwise retrieve the endpoints now:

~~~bash
kubectl get servicemappings.connectivityproxy.sap.com apigateway-mapping \
  -n oneconnect -ojsonpath='{.status.endpoint}'

kubectl get servicemappings.connectivityproxy.sap.com frontend-mapping \
  -n oneconnect -ojsonpath='{.status.endpoint}'

kubectl get servicemappings.connectivityproxy.sap.com akhq-mapping \
  -n oneconnect -ojsonpath='{.status.endpoint}'
~~~

Result format:

~~~
tcp://cp.c-XXXXX.kyma.ondemand.com:443/[service-id]
~~~

> ℹ️ Save all three values, you will need them in Step 7.5.

### Step 7.3: Install SAP Cloud Connector

1. Download the installer from [https://tools.hana.ondemand.com](https://tools.hana.ondemand.com). Search for **"Cloud Connector"** on the page (`Ctrl + F`). Download the **Windows MSI installer**. If the MSI fails, use the **Portable** version as a fallback.
2. Install on a server that has network access to your SAP on-premise system.
3. Open the admin UI at: `https://localhost:8443`
4. Log in with default credentials: **`Administrator / manage`**. Change your password immediately.

### Step 7.4: Connect Cloud Connector to the BTP Subaccount

In the Cloud Connector admin UI, go to **Subaccounts → + (Add)** and fill in the following fields:

| Field | Value |
|---|---|
| **Region** | Your BTP subaccount region (e.g. `cf-us10`). |
| **Subaccount** | Your BTP Subaccount ID. |
| **Location ID** | Must exactly match the value used in Step 3.1. |
| **Login** | Your BTP email address. |
| **Password** | Your BTP password. |

| Placeholder | Description |
|---|---|
| `[your-btp-region]` | BTP subaccount region, e.g. `cf-us10`. |
| `[SUBACCOUNT-ID]` | SAP BTP Subaccount ID. Found in BTP Cockpit → Subaccount overview. |
| `[LOCATION-ID]` | Must be the exact same value you used in Step 3.1 (case-sensitive). |

> ℹ️ **Note:** After saving, verify the status shows **Connected** in green before continuing.

### Step 7.5: Create Service Channels

Create one service channel per ServiceMapping. In the Cloud Connector admin UI, go to **On-Premise to Cloud → + (Add)**.

#### Channel 1: apigateway

| Placeholder | Description |
|---|---|
| `[apigateway-endpoint]` | The endpoint retrieved in Step 7.2. Remove the `tcp://` prefix before pasting. |
| `[local-port]` | A free port on your CC server, e.g. `9000`. |

| Field | Value |
|---|---|
| **Type** | K8s Cluster. |
| **Endpoint** | apigateway endpoint from Step 7.2 (remove the `tcp://` prefix). |
| **Local Port** | `9000` (or another free port). |
| **Enabled** | Yes. |

#### Channel 2: frontend

| Placeholder | Description |
|---|---|
| `[frontend-endpoint]` | The endpoint retrieved in Step 7.2. Remove the `tcp://` prefix before pasting. |
| `[local-port]` | A different free port, e.g. `5050` (must differ from Channel 1). |

| Field | Value |
|---|---|
| **Type** | K8s Cluster. |
| **Endpoint** | frontend endpoint from Step 7.2 (remove the `tcp://` prefix). |
| **Local Port** | `5050` (must be different from Channel 1). |
| **Enabled** | Yes. |

#### Channel 3: AKHQ

Only required if you completed the AKHQ ServiceMapping in Section 3.

| Field | Value |
|---|---|
| **Type** | K8s Cluster. |
| **Endpoint** | AKHQ endpoint from Step 7.2 (remove the `tcp://` prefix). |
| **Local Port** | `8080` (must differ from Channels 1 and 2). |
| **Enabled** | Yes. |

> 💡 **Tip:** After activating each channel, the status should show green with `1/1` connections. If it shows `0/1`, see the [Troubleshooting](#troubleshooting) section.

### Step 7.6: Configure SAP Connection (SM59)

With the service channels active, create an HTTP destination in your SAP system:

| Placeholder | Description |
|---|---|
| `[cc-server-host]` | IP address or hostname of the Cloud Connector server. |
| `[local-port]` | The local port you set in the service channel (e.g. `9000`). |

| Field | Value |
|---|---|
| **Connection Type** | HTTP Connection to External Server (Type **G**). |
| **Host** | `[cc-server-host]` |
| **Port** | `[local-port]` |
| **Protocol** | HTTP |
| **Path Prefix** | `/your-endpoint-path` |

> ℹ️ For step-by-step instructions on configuring `SM59`, see the [Endpoint Customization (SM59) manual](../SAP_Data_Modeler/14-endpoint-customization-sm59.md).

### Step 7.7: Test the Tunnel

Before going live, test connectivity from the Cloud Connector server. The `Host` header is required for the Connectivity Proxy to route requests correctly.

| Placeholder | Description |
|---|---|
| `[local-port]` | The local port of the apigateway service channel (e.g. `9000`). |
| `[local-port-frontend]` | The local port of the frontend service channel (e.g. `5050`). |
| `[local-port-akhq]` | The local port of the AKHQ service channel (e.g. `8080`). |

~~~bash
# Test apigateway tunnel
curl -vvv http://localhost:[local-port]/ \
  -H "Host: apigateway.oneconnect.svc.cluster.local"

# Test frontend tunnel
curl -vvv http://localhost:[local-port-frontend]/ \
  -H "Host: frontendoneconnect.oneconnect.svc.cluster.local"

# Test AKHQ tunnel (if configured)
curl -vvv http://localhost:[local-port-akhq]/ \
  -H "Host: akhq.kafka.svc.cluster.local"
~~~

> ✅ Any HTTP response (including `404` or `302`) means the tunnel is working.
>
> ❌ "Empty reply from server" means the tunnel is **not** working. See [Troubleshooting](#troubleshooting).

---

## ✓ Verification and Health Checks

Use the commands below to confirm every component is running correctly.

### Platform Pods

~~~bash
kubectl get pods -n oneconnect
~~~

> ℹ️ All pods must show `2/2 READY` (app + Istio sidecar).

### Services

~~~bash
kubectl get services -n oneconnect
~~~

> ℹ️ You should see only **ClusterIP** services, no LoadBalancer entries.

### Kafka

~~~bash
kubectl get kafka,deployment -n kafka
kubectl get pods -n kafka
~~~

> ℹ️ `READY` must be `True` for both resources, and 3 pool nodes + entity-operator must all be `Running`.

### ServiceMappings

~~~bash
kubectl get servicemappings.connectivityproxy.sap.com -n oneconnect
~~~

### AKHQ Kafka UI

~~~bash
kubectl get pods,svc -n kafka | grep akhq
~~~

> ℹ️ AKHQ shows `type=ClusterIP`. Access is through Cloud Connector only.

For local access via port-forward:

~~~bash
kubectl port-forward svc/akhq 8080:8080 -n kafka
~~~

Open `http://localhost:8080` (default login: `admin / admin123`).

### Istio Sidecar

~~~bash
kubectl get namespace oneconnect --show-labels | grep istio
kubectl get pods -n oneconnect
~~~

> ℹ️ All pods must show `2/2 READY`.

### Observability Stack

~~~bash
kubectl get pods -n observability
~~~

---

## Troubleshooting

### "Empty reply from server" when testing the tunnel

- Verify Istio sidecar is injected. All pods must show `2/2 READY`.
- Restart the Connectivity Proxy:

  ~~~bash
  kubectl rollout restart statefulset connectivity-proxy -n kyma-system
  ~~~

- Check proxy logs:

  ~~~bash
  kubectl logs -n kyma-system -l app=connectivity-proxy --tail=50
  ~~~

### ACCESS_DENIED in Connectivity Proxy logs

- Confirm the `istio-injection=enabled` label is set on the namespace.
- Verify the Location ID matches exactly between Cloud Connector and your Helm install (case-sensitive).
- Restart deployments:

  ~~~bash
  kubectl rollout restart deployment -n oneconnect
  ~~~

### Cloud Connector cannot connect to the BTP Subaccount

- Verify the BTP region exactly matches your subaccount region.
- Check your BTP credentials (email and password).
- Confirm the CC server has outbound internet access to `*.ondemand.com`.

### Service Channel shows 0/1 connections

- Verify the endpoint was copied without the `tcp://` prefix.
- Verify the CC is connected to the correct subaccount.
- Re-retrieve the endpoint:

  ~~~bash
  kubectl get servicemappings.connectivityproxy.sap.com apigateway-mapping \
    -n oneconnect -ojsonpath='{.status.endpoint}'
  ~~~

### Kafka pods not starting

- Ensure the Strimzi operator shows `1/1 Running` before applying Kafka resources (Section 1).
- Check operator logs:

  ~~~bash
  kubectl logs -n kafka -l strimzi.io/kind=cluster-operator --tail=100
  ~~~

### Cloud Connector services are timing out / not resolving

Istio is preventing all traffic that is not configured as mTLS. Apply a PERMISSIVE mode policy:

~~~bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: PERMISSIVE
EOF
~~~

### Helm install fails: missing cloudConnector values

Re-run with all required flags:

~~~bash
helm upgrade oneconnect . \
  --namespace oneconnect \
  --values values-aws.yaml \
  --set dockerHub.token=[your-docker-token] \
  --set cloudConnector.enabled=true \
  --set cloudConnector.locationId="[LOCATION-ID]" \
  --set cloudConnector.subaccountId="[SUBACCOUNT-ID]"
~~~

---

## Quick Reference: Useful Commands

### Platform Pods and Services

~~~bash
kubectl get pods -n oneconnect
kubectl get services -n oneconnect
~~~

### Connectivity Proxy

~~~bash
kubectl get pods -n kyma-system | grep connectivity
kubectl logs -n kyma-system -l app=connectivity-proxy --tail=50
kubectl rollout restart statefulset connectivity-proxy -n kyma-system
~~~

### ServiceMappings

~~~bash
kubectl get servicemappings.connectivityproxy.sap.com -n oneconnect
kubectl get servicemappings.connectivityproxy.sap.com apigateway-mapping \
  -n oneconnect -ojsonpath='{.status.endpoint}'
~~~

### Service Instances

~~~bash
kubectl get serviceinstances,servicebindings -n oneconnect
~~~

### Kafka Connection

~~~
kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092
http://schema-registry.kafka.svc.cluster.local:8081
~~~

### Observability

~~~bash
kubectl get pods -n observability
kubectl port-forward svc/grafana 3000:3000 -n observability
~~~

---

## Key Concepts and Glossary

| Term | Definition |
|---|---|
| **Kubernetes (K8s)** | System for running and managing containerized applications. The orchestrator for your app containers. |
| **Kyma** | SAP-managed Kubernetes environment running in SAP BTP (Business Technology Platform). |
| **Namespace** | A logical partition inside Kubernetes, like a separate workspace (e.g. `kafka`, `oneconnect`, `observability`). |
| **Strimzi** | A Kubernetes operator that makes it easy to deploy and manage Apache Kafka clusters. |
| **Kafka** | A distributed messaging system that enables data streaming between services. |
| **Helm** | A package manager for Kubernetes. Installs a full app with one command instead of many YAML files. |
| **Pod** | A running container inside Kubernetes. Similar to a lightweight virtual machine. |
| **SAP Cloud Connector** | A secure tunnel agent installed in your on-premise network. Bridges local SAP systems to SAP BTP without opening inbound firewall ports. |
| **Istio** | A service mesh that adds encrypted traffic control between pods. Required for Cloud Connector access. |
| **ServiceMapping** | A Kyma resource that links a Kubernetes service to the Cloud Connector tunnel. |
| **Observability** | A monitoring stack providing dashboards, logs, and distributed tracing. |
| **Docker Hub** | Container image registry, an "app store" for Docker images. Requires an access token. |
| **AKHQ** | A web-based dashboard for Kafka. Used to browse topics, inspect messages, and monitor consumers. |
| **Kafka Connect** | A Kafka component that runs connectors to external systems such as databases. |

---

## Frequently Asked Questions

### What if pods do not start?

Always verify that Strimzi is running (`1/1`) before installing Kafka. For other pods, check logs with `kubectl logs [pod-name] -n [NAMESPACE]`. Common causes are an incorrect or expired Docker token, or a previous Helm release that was not properly uninstalled.

### Why am I getting ACCESS_DENIED?

The Istio sidecar is either not injected or the Location ID does not match between your Helm install and the Cloud Connector configuration. Check that the namespace has the `istio-injection=enabled` label and that the Location ID is identical in both places (case-sensitive).

### How do I know the tunnel is working?

Run a `curl` command from the CC server to the local port with the appropriate `Host` header. Any HTTP response (even a `404` or `302`) confirms the tunnel is active. "Empty reply from server" means the tunnel is not working.

### Is the Observability stack mandatory?

It is optional from a technical standpoint, but highly recommended for production deployments. It consumes additional cluster resources, so confirm with the customer whether to include it.

### Where do I find the Subaccount ID?

Log in to the **SAP BTP Cockpit** and navigate to your Subaccount overview page. The Subaccount ID (a UUID) is displayed in the header area of the subaccount page.

### How do I test if the tunnel is working end to end?

Run the `curl` test in Step 7.7 from the CC server. Any HTTP response (even `404`) confirms the tunnel is active. Then create the `SM59` destination in SAP and call a known OneConnect endpoint to confirm application-level connectivity.
