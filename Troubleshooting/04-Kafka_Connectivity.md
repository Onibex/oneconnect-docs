# Kafka Connectivity

Connectivity between the Producer, the Consumer, and each SAP Link's Kafka Connect and the Kafka brokers: reaching them, authenticating, creating topics, getting records accepted, and getting timely answers back to SAP.

In this guide, `<workspace-id>` is the SAP Link ID, `<CLIENT>` is the SAP Link's **SAP Connector Name (Topic Prefix)**, and `<ENV>` is its **SAP Environment**. The platform deploys each SAP Link as the Deployments `<workspace-id>-producer` and `<workspace-id>-consumer` in the `oneconnect` namespace.

---

## 0. Getting More Detail

The default log level covers most cases in this guide. For more detail — for example the outcome of each topic creation — you can optionally raise the workload to `DEBUG`:

```bash
kubectl set env deployment/<workspace-id>-producer -n oneconnect LOGGING_LEVEL_ROOT=DEBUG
kubectl logs -n oneconnect deploy/<workspace-id>-producer -f | grep -i topic
```

Revert when finished — see [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md#4-raising-the-log-level-to-see-more-detail).

---

## 1. Configuration Reference

The Producer and the Consumer read the same connection properties. For SAP Links created from the platform, several values come from the SAP Link itself, as shown in the **Source** column; the others use the image defaults.

| Property | Default | Source for platform SAP Links | Notes |
|---|---|---|---|
| `kafkaproperties.bootstrapaddress` | *(required)* | SAP Link Kafka secret | `host:port`, comma-separated for multiple brokers |
| `kafkaproperties.deploymenttype` | `CLOUD` | SAP Link (Connection Details) | `CLOUD` or `ONPREMISE` |
| `kafkaproperties.securityprotocol` | `SASL_SSL` | SAP Link Kafka secret | Applies in `ONPREMISE` mode |
| `kafkaproperties.saslmechanism` | `PLAIN` | Image default | — |
| `kafkaproperties.sasluser` / `saslpassword` | *(required)* | SAP Link Kafka secret | API key / secret |
| `kafkaproperties.clientdnslookup` | `use_all_dns_ips` | Image default | — |
| `kafkaproperties.topicpartitions` | `1` | SAP Link (Topic Settings) | Partitions of each topic the component creates |
| `kafkaproperties.topicreplicas` | `3` | SAP Link (Topic Settings) | Must not exceed the broker count |
| `kafkaproperties.retentiontime` | `604800000` (7 days) | SAP Link (Topic Settings) | `retention.ms` of created topics |
| `kafkaproperties.topicmessagesize` | `8388608` (8 MB) | Image default | Producer: topic `max.message.bytes`. Consumer: topic `max.message.bytes` **and** its client `max.request.size` |
| `kafkaproperties.maxrequestsize` | `8388608` (8 MB) | Image default | **Producer only** — client `max.request.size` |
| `kafkaproperties.cleanuppolicy` | `delete` | Image default | — |
| `kafkaproperties.compression` | `producer` | Image default | — |
| `kafkaproperties.groupid` | `group_0` | SAP Link: `<CLIENT>_<ENV>` | **Consumer only** — its consumer group |

> 📌 Topic settings (partitions, replicas, retention, message size) apply when a topic is **created**. To change an existing topic, edit it in AKHQ — see [§4](#topic-created-but-with-the-wrong-settings).

### What `deploymenttype` controls

| Value | `security.protocol` | SASL configured? | Schema Registry auth? |
|---|---|---|---|
| `CLOUD` | Set to `SASL_SSL` | Yes | Yes |
| `ONPREMISE` | Your `securityprotocol` value | Yes — unless `securityprotocol` is `NO_SECURITY` | No |
| `ONPREMISE` + `NO_SECURITY` | Not set (plaintext) | No | No |

> 📌 Set `deploymenttype` to `CLOUD` or `ONPREMISE`. To confirm the value in use:
>
> ```bash
> kubectl exec -n oneconnect deploy/<workspace-id>-producer -- env | grep -iE 'deploymenttype|securityprotocol'
> ```

---

## 2. Cannot Reach the Brokers

### Symptoms

At runtime (Producer or Consumer logs):

```
org.apache.kafka.common.errors.TimeoutException: Topic <topic> not present in metadata after 60000 ms.
java.net.UnknownHostException: ...
Connection to node -1 (<host>/<ip>:<port>) could not be established. Broker may not be available.
```

At startup, when the bootstrap host cannot be resolved, the pod restarts (`CrashLoopBackOff`) with:

```
org.apache.kafka.common.config.ConfigException: No resolvable bootstrap urls given in bootstrap.servers
```

In the platform, the SAP Link connection test reports the Kafka check as failed; the Builder log shows the reason:

```
Failed to connect to broker [<bootstrap>]: <message>
```

### Diagnosis

```bash
# What address is configured?
kubectl exec -n oneconnect deploy/<workspace-id>-producer -- env | grep -i bootstrap

# Builder's view of the connection test
kubectl logs -n oneconnect deploy/builder --tail=200 | grep -i 'broker'

# In-cluster Kafka: is it up?
kubectl get pods -n kafka
kubectl get svc  -n kafka
```

### Causes

| Cause | Check | Fix |
|---|---|---|
| Bootstrap address differs from the cluster's | Compare against the cluster's listener address | Update the SAP Link's Kafka secret and restart the workload |
| In-cluster service name not fully qualified | Across namespaces the name must be complete | Use `kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092` |
| Kafka not yet running | `kubectl get pods -n kafka` | Complete the Kafka phase before OneConnect — see [02](./02-Deployment_and_Installation.md#deployment-order-was-not-respected) |
| Security group / NSG / firewall blocks the port | The connection test reports a timeout | Open the broker port from the cluster's egress range |
| Cloud provider private networking | The cluster is not reachable from this VPC/VNet | Configure peering/PrivateLink, or use a public endpoint |
| Port missing | `host` without `:port` | Bootstrap must be `host:port` |

---

## 3. Authentication Rejected by the Brokers

### Symptoms

```
org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed: Invalid username or password
org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed during authentication due to invalid credentials
```

This indicates the network path is working and the brokers declined the credentials. On the synchronous Producer endpoint SAP receives `400`, and the Producer logs `-->ExecutionException =org.apache.kafka.common.errors.SaslAuthenticationException: ...`.

### Causes

| Cause | Fix |
|---|---|
| API key/secret revoked or mistyped | Issue a new key pair in the Kafka cluster and update the SAP Link's Kafka secret |
| Key belongs to a different cluster | Cloud API keys are cluster-scoped — use a key issued for the cluster in the bootstrap address |
| Whitespace or quotes in the secret | Enter the secret exactly as issued, without surrounding spaces or quotes |
| Workload not restarted after the change | `kubectl rollout restart deployment/<workspace-id>-producer deployment/<workspace-id>-consumer -n oneconnect` |
| `saslmechanism` differs from the broker's | The default is `PLAIN`; match the mechanism the broker listener expects |

```bash
kubectl exec -n oneconnect deploy/<workspace-id>-producer -- env | grep -iE 'sasluser|saslmechanism|securityprotocol'
```

### TLS-specific errors

```
javax.net.ssl.SSLHandshakeException: PKIX path building failed
org.apache.kafka.common.errors.SslAuthenticationException
```

| Cause | Fix |
|---|---|
| Self-signed / private CA broker certificate | Add the CA to the pod's truststore, or use a publicly trusted certificate |
| `SASL_SSL` configured against a `SASL_PLAINTEXT` listener | Match `securityprotocol` to the listener you are connecting to |
| Expired broker certificate | Renew it |
| Hostname does not match the certificate SAN | Connect using the name in the certificate |

---

## 4. Topics Are Never Created

The Producer creates its raw topic, and the Consumer creates one topic per SAP table (and its error topic) the first time it sees them.

### Diagnosis

Check which topics exist in **AKHQ → Topics**. For the detailed creation outcome, enable `DEBUG` ([§0](#0-getting-more-detail)) and look for the exceptions below:

```bash
kubectl set env deployment/<workspace-id>-producer -n oneconnect LOGGING_LEVEL_ROOT=DEBUG
kubectl logs -n oneconnect deploy/<workspace-id>-producer -f | grep -i 'topic'
```

### Causes

| Exception | Cause | Fix |
|---|---|---|
| `TopicAuthorizationException` | The API key lacks `CREATE` on the cluster / `WRITE` on the topic | Grant ACLs — see [§6](#6-acls-and-authorization) |
| `InvalidReplicationFactorException` | `topicreplicas` exceeds the broker count | Set **Replicas** in the SAP Link's Topic Settings to ≤ number of brokers (`1` for a single-broker development cluster) |
| `PolicyViolationException` / `InvalidConfigurationException` | The cluster's policy rejects the requested partitions, replicas, retention, or other topic settings | Align the SAP Link's Topic Settings with the cluster's policy |
| `TimeoutException` | Brokers unreachable | See [§2](#2-cannot-reach-the-brokers) |

> 📌 After correcting ACLs or topic settings, restart the Consumer:
>
> ```bash
> kubectl rollout restart deployment/<workspace-id>-consumer -n oneconnect
> ```

### Topic created but with the wrong settings

If the broker auto-created a topic before OneConnect created it explicitly, the topic carries the broker's defaults rather than the configured retention, message size, and compression.

**To inspect it:** in AKHQ, open **Topics** → your topic → the **Configs** tab. This lists the effective settings, including `max.message.bytes`, `retention.ms`, `cleanup.policy`, and `compression.type`. The **Partitions** tab shows partition count, replicas, and in-sync replicas.

**To correct it:** edit the values in the same **Configs** tab and save.

Partition count can be increased but not reduced; replication factor changes require a partition reassignment.

---

## 5. Message Size Limits

### Symptom

```
org.apache.kafka.common.errors.RecordTooLargeException: The message is <N> bytes when serialized which is larger than 8388608, which is the value of the max.request.size configuration.
```

- **Producer, synchronous endpoint:** SAP receives `400`, and the Producer logs `-->ExecutionException =org.apache.kafka.common.errors.RecordTooLargeException: ...`.
- **Producer, asynchronous endpoint:** the asynchronous endpoint confirms receipt; for delivery confirmation use the synchronous endpoint (the URL shown by the platform) — see [§9](#9-sap-waits-a-long-time-for-the-producers-response).
- **Consumer:** the per-row send logs `-->Error Sending Record to topic=... errorType=RecordTooLargeException, ...` and writes an entry to `<CLIENT>_<ENV>_PROCESSING_ERRORS`.

SAP payloads with many rows, or wide tables with long text fields, can exceed the default 8 MB.

### The limits that must agree

| Limit | Where | Property | Default |
|---|---|---|---|
| `max.request.size` | Producer client | `kafkaproperties.maxrequestsize` | 8 MB |
| `max.message.bytes` | Raw topic (created by the Producer) | Producer `kafkaproperties.topicmessagesize` | 8 MB |
| `max.request.size` **and** `max.message.bytes` | Consumer client and the per-table / error topics it creates | Consumer `kafkaproperties.topicmessagesize` | 8 MB |
| `message.max.bytes` | Broker | Cluster configuration | Cluster-dependent |

Raising only a client limit produces a broker-side rejection instead; raising only a topic limit still fails client-side. **Raise them together.**

### Ways to resolve it

1. **Send fewer records per request (recommended).** The entity controls this through its **Number of Records** setting — see [Entity Creation and Main Customizing](../SAP_Data_Modeler/Data_Modeler_Manuals/01-Entity_Creation_and_Main_Customizing.md). This works on every cluster, including managed clusters whose size cap cannot be raised.
2. **Raise the limits** on a self-managed cluster, in all the places above. Set the variables on the workloads (`KAFKAPROPERTIES_MAXREQUESTSIZE` and `KAFKAPROPERTIES_TOPICMESSAGESIZE` on the Producer, `KAFKAPROPERTIES_TOPICMESSAGESIZE` on the Consumer) and update `max.message.bytes` on topics that already exist through AKHQ → **Configs**.
3. **Check whether the payload grew** — compare the current record against an earlier one in AKHQ to see whether a nested table was added.

---

## 6. ACLs and Authorization

### Symptoms

```
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [732_DEV_MARA]
org.apache.kafka.common.errors.ClusterAuthorizationException: Cluster authorization failed.
org.apache.kafka.common.errors.GroupAuthorizationException: Not authorized to access group: 732_DEV
```

When the Consumer's own read is refused, it logs the condition on every poll cycle and records it in the error topic:

```
Unhandled exception in consumerFactory at partition:0 offset:0, errorType=GroupAuthorizationException, cause=..., stackTrace=...
```

### Permissions required

| Component | Operation | Resource |
|---|---|---|
| Producer | `CREATE` | Cluster (to create its raw topic) |
| Producer | `WRITE`, `DESCRIBE` | `<CLIENT>_<ENV>_RAWDATA` and `<CLIENT>_<ENV>_INBOUND_ERRORS` |
| Consumer | `READ`, `DESCRIBE` | `<CLIENT>_<ENV>_RAWDATA` |
| Consumer | `READ` | Consumer group `<CLIENT>_<ENV>` |
| Consumer | `CREATE` | Cluster (one topic per SAP table) |
| Consumer | `WRITE`, `DESCRIBE` | Per-table topics `<CLIENT>_<ENV>_<TABLE>` and `<CLIENT>_<ENV>_PROCESSING_ERRORS` |
| Kafka Connect (per SAP Link) | `READ` | Consumer group `connect-group-<workspace-id>` and the per-table topics its connectors read |
| Kafka Connect (per SAP Link) | `CREATE`, `READ`, `WRITE`, `DESCRIBE` | Internal topics `connect-configs-<workspace-id>`, `connect-offsets-<workspace-id>`, `connect-status-<workspace-id>` |

> 📌 The Consumer needs **cluster-level `CREATE`**, since it creates a topic the first time it sees each SAP table. A key scoped to a fixed topic list covers existing tables but not new ones.

Prefix-based ACLs are the practical approach, since OneConnect topic names follow `<CLIENT>_<ENV>_*` and Kafka Connect resources follow `connect-*-<workspace-id>`.

On a Strimzi cluster, ACLs are declared on the `KafkaUser` resource:

```bash
kubectl get kafkauser -n kafka
kubectl describe kafkauser -n kafka <user>
```

For an external cluster such as Confluent Cloud, manage ACLs from that provider's console.

---

## 7. The Consumer Connects but Consumes Nothing

Distinguish "cannot connect" from "connected, nothing to read".

Each SAP Link's Consumer uses the consumer group `<CLIENT>_<ENV>` (for example `732_DEV`). In AKHQ, open **Consumer Groups** and select it. The view lists its members, the topics and partitions assigned, and the lag per partition.

| What you see | Meaning | Next step |
|---|---|---|
| Group not found | The Consumer has not connected | [§2](#2-cannot-reach-the-brokers), [§3](#3-authentication-rejected-by-the-brokers) |
| Group exists, no members | The Consumer is not running | `kubectl get pods -n oneconnect`; see [06 §8](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) |
| Members present, `LAG` = 0, `CURRENT-OFFSET` = 0 | Connected, but the raw topic is empty | The Producer is not writing — go upstream |
| Members present, `LAG` growing | Processing is slower than the arrival rate, or poll cycles are being refused | [§8](#8-rebalances-and-poll-timeouts), [§6](#6-acls-and-authorization) |
| Group subscribed to an unexpected topic | Topic name mismatch | [06 — No Data Reaching Destination](./06-No_Data_Reaching_Destination.md#2-the-consumer-is-subscribed-to-a-topic-nobody-writes-to) |

### How offsets are committed

Processed records are committed; records that cannot be processed are written to `<CLIENT>_<ENV>_PROCESSING_ERRORS` for review — see [06 §7](./06-No_Data_Reaching_Destination.md#7-reviewing-and-reprocessing-records-from-the-error-topics).

A new consumer group starts from `earliest`, so a newly deployed Consumer reads the whole raw topic. A large initial lag on a fresh deployment is expected and shrinks steadily.

### Consumer groups and replicas

Every replica of a SAP Link's Consumer joins the same group, `<CLIENT>_<ENV>`, and Kafka distributes the raw topic's partitions among them. With one partition, one replica processes the data and the others stand by. To spread the load across replicas, give the raw topic as many partitions as there are replicas (set **Partitions** in the SAP Link's Topic Settings before the topic is created, or increase it in AKHQ).

Consumers outside the platform that reuse the same group name on the same raw topic share its partitions with the SAP Link's Consumer; give each independent reader its own `kafkaproperties.groupid`.

---

## 8. Rebalances and Poll Timeouts

### Symptom

Kafka client warnings in the Consumer log:

```
consumer poll timeout has expired. This means the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms ...
Asynchronous auto-commit of offsets ... failed: ...
Attempt to heartbeat failed since group is rebalancing
```

### Cause

Processing one batch took longer than the Consumer's `max.poll.interval.ms` (**600000**, 10 minutes). The group then rebalances and the batch is delivered again.

### Confirm

Check the Consumer's CPU usage and the time between log lines for consecutive records, and look for the warnings above:

```bash
kubectl logs -n oneconnect deploy/<workspace-id>-consumer --tail=500 | grep -iE 'poll timeout|auto-commit|rebalanc'
kubectl top pod -n oneconnect | grep consumer
```

### Fix

| Cause | Fix |
|---|---|
| Very large SAP payloads take long to convert | Reduce the entity's **Number of Records** per payload |
| Schema Registry is slow or timing out | See [05](./05-Avro_Serialization_and_Schema_Registry.md) |
| Brokers slow or unreachable (topic creation and sends wait for them) | See [§2](#2-cannot-reach-the-brokers) |
| Consumer is CPU-throttled | Raise **Limit CPU** in the SAP Link's Advanced Settings |

> 💡 Use `upsert` insert mode on the sink connector so that records delivered again after a rebalance or restart update the same rows in the destination.

---

## 9. SAP Waits a Long Time for the Producer's Response

### Symptom

SAP reports a timeout (for example an ICM/HTTP timeout in SM59 or in the entity log), while the Producer is running. The Producer later logs a Kafka result such as `-->ExecutionException =org.apache.kafka.common.errors.TimeoutException: Topic <topic> not present in metadata after 60000 ms.`

### Cause

The synchronous endpoint answers SAP once Kafka has answered. When the brokers are slow or unreachable, the Producer waits for Kafka up to these limits:

| Step | Maximum wait (Kafka client defaults) |
|---|---|
| Topic check | Up to ~60 s (`default.api.timeout.ms`) |
| Send — waiting for topic metadata | Up to ~60 s (`max.block.ms`) |
| Synchronous endpoint — waiting for the broker's acknowledgment (`acks=all`) | Up to the delivery timeout (`delivery.timeout.ms`, 120 s) |

The synchronous endpoint (the URL shown by the platform) answers once Kafka has confirmed the write, so SAP receives either `201` with the offset or `400` with the Kafka result.

### Confirm

```bash
kubectl logs -n oneconnect deploy/<workspace-id>-producer --since=15m | grep -iE 'ExecutionException|not present in metadata|Expiring'
```

Messages that mention `60000 ms` or `Expiring ... record(s)` mean the Producer was waiting on Kafka.

### Fix

1. Restore broker connectivity — [§2](#2-cannot-reach-the-brokers) and [§3](#3-authentication-rejected-by-the-brokers).
2. Give SAP enough time to receive the Producer's answer: in **SM59**, open the SAP Link's HTTP destination, go to the **Special Options** tab and set **Timeout** to a value above the Producer's longest wait (for example `180` seconds). See [Endpoint Customization (SM59)](../SAP_Data_Modeler/Data_Modeler_Manuals/15-Endpoint_Customization_SM59.md).
3. Keep payloads moderate (**Number of Records**) so each request completes quickly when Kafka is healthy.
4. Records that SAP marked as failed during the outage can be sent again with [Reprocessing Records](../SAP_Data_Modeler/Data_Modeler_Manuals/12-Reprocessing_Records.md).

---

## 10. The SAP Link's Kafka Connect Is Not Reachable

Each SAP Link has its own Kafka Connect, created from [Configure Kafka Connect](../Smart_Gateway/Using_the_Smart_Gateway/04-Configure_Kafka_Connect.md). The platform deploys it in the `kafka` namespace as the Deployment and Service `connect-<workspace-id>` (pod labels `app=connect-cluster`, `workspace=<workspace-id>`) and reaches its REST API at `http://connect-<workspace-id>.kafka.svc.cluster.local:8083`.

### Symptom

Creating, updating, or restarting a connector, or reading its status, returns an error in the platform such as:

```
Kafka Connect Deployment not found for workspace: <workspace-id>. Please create it first.
Failed to create connector: I/O error on POST request for "http://connect-<workspace-id>.kafka.svc.cluster.local:8083/connectors": ...
Kafka Connect error: <response body>
Failed to get connector status: ...
Failed to restart connector due to communication error: ...
```

While a connector request is in progress, the Builder log may show:

```
Unexpected error or timeout creating connector (attempt 1/5). Retrying in 5000ms...
HTTP error 500 creating connector (attempt 1/5). Retrying in 5000ms...
Max retries reached. Unexpected error creating connector
```

### Cause

Kafka Connect has not been configured for this SAP Link yet, its pod is not running or not ready, or the worker cannot join the Kafka cluster.

### Confirm

```bash
# Is it deployed and running?
kubectl get deploy,svc,pods -n kafka -l workspace=<workspace-id>
kubectl describe pod -n kafka -l workspace=<workspace-id>

# What does the worker say?
kubectl logs -n kafka deploy/connect-<workspace-id> --tail=200

# Does the REST API answer?
kubectl port-forward -n kafka svc/connect-<workspace-id> 8083:8083
curl http://localhost:8083/connectors
```

In AKHQ, the worker appears under **Connect** as `connect-<workspace-id>`.

### Fix

| What you find | Fix |
|---|---|
| `Kafka Connect Deployment not found for workspace` | Run [Configure Kafka Connect](../Smart_Gateway/Using_the_Smart_Gateway/04-Configure_Kafka_Connect.md) for the SAP Link, then create the connector |
| Pod `Pending` | The worker requests 1 GiB memory / 250m CPU (limits 2560 MiB / 500m). Free node capacity or raise the `kafka` namespace quota — see [02 §3](./02-Deployment_and_Installation.md#3-pods-stuck-in-pending) |
| Pod `CreateContainerConfigError`: `couldn't find key KAFKAPROPERTIES_SCHEMAREGISTRYURL in Secret ...` | Kafka Connect reads the Schema Registry URL and credentials from the SAP Link's Kafka secret. Create the secret with **Schema Registry URL** and credentials, then configure Kafka Connect again |
| Worker log shows broker timeouts or `SaslAuthenticationException` | Same causes as the Producer/Consumer — [§2](#2-cannot-reach-the-brokers), [§3](#3-authentication-rejected-by-the-brokers) |
| Worker log shows it cannot create `connect-configs-<workspace-id>`, `connect-offsets-<workspace-id>`, or `connect-status-<workspace-id>` | Set **Storage Replication Factor** ≤ the broker count, and grant the ACLs in [§6](#6-acls-and-authorization) |
| `HTTP error 500 ...` while the worker is starting | The worker is joining its group; wait until the pod is `Ready` and retry |
| Configuring Kafka Connect fails with `Failed to create/update resource 'connect-<workspace-id>': 403 FORBIDDEN` | The Builder's service account needs the Role `smartgateway-kafka-connect-manager` in the `kafka` namespace (Helm `rbac.create: true`) |
| `Kafka secret '<name>' not found in namespace 'oneconnect'` | The SAP Link's Kafka secret is copied into `kafka` for Kafka Connect; select a secret that exists for the SAP Link owner |
| The worker is not listed in AKHQ, and the Builder logs `Failed to register Connect in AKHQ config for workspace <workspace-id>` | Kafka Connect itself is available (use the port-forward above). The AKHQ entry needs the Role `akhq-config-updater` and the ConfigMap `akhq-config` in `kafka`; configure Kafka Connect again once they are in place |

> ℹ️ Kafka Connect takes its security protocol from the SAP Link's security settings. Match it to the broker listener.

---

## 11. Quick Command Reference

### In AKHQ

| Task | Where |
|---|---|
| List topics and their record counts | **Topics** |
| Partitions, replicas, in-sync replicas | **Topics** → topic → **Partitions** |
| Effective topic configuration | **Topics** → topic → **Configs** |
| Read the newest messages (Avro decoded) | **Topics** → topic → **Data** |
| Consumer group members and lag | **Consumer Groups** → `<CLIENT>_<ENV>` |
| SAP Link's Kafka Connect and connectors | **Connect** → `connect-<workspace-id>` |
| Registered schemas | **Schema Registry** |

### With kubectl

```bash
# Kafka cluster and operator
kubectl get kafka -n kafka
kubectl get pods  -n kafka
kubectl logs -n kafka -l strimzi.io/kind=cluster-operator --tail=100

# SAP Link workloads
kubectl get pods -n oneconnect | grep <workspace-id>
kubectl get pods -n kafka -l workspace=<workspace-id>

# AKHQ address
kubectl get svc -n kafka | grep akhq
```

---

## Summary

1. A missing topic usually points to **ACLs, replicas, or cluster policy** — `DEBUG` is available for extra detail.
2. Set `deploymenttype` to `CLOUD` or `ONPREMISE`.
3. Timeouts mean **network**; `SaslAuthenticationException` means **credentials**.
4. Partitions, replicas, and retention come from the SAP Link's **Topic Settings**; keep replicas ≤ the broker count.
5. Message size: the Producer uses `maxrequestsize` + `topicmessagesize`; the Consumer uses `topicmessagesize` for both limits. Sending fewer records per payload is the recommended adjustment.
6. Each SAP Link's Consumer uses the group `<CLIENT>_<ENV>`; processed records are committed, and records that cannot be processed go to `<CLIENT>_<ENV>_PROCESSING_ERRORS` for review.
7. Give SAP's HTTP timeout (SM59 → Special Options) enough room for the synchronous endpoint to report Kafka's answer.
8. Each SAP Link has its own Kafka Connect, `connect-<workspace-id>` in the `kafka` namespace — check its pod and REST API when connector operations fail.
