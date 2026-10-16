# No Data Reaching Destination

When SAP reports success but the destination table stays empty, this guide walks the pipeline stage by stage to find where records stop, and then covers the SAP Link deployment cases that keep data from flowing at all.

In this guide, `<workspace-id>` is the SAP Link ID, `<CLIENT>` is the SAP Link's **SAP Connector Name (Topic Prefix)**, and `<ENV>` is its **SAP Environment**. Each SAP Link runs as the Deployments `<workspace-id>-producer` and `<workspace-id>-consumer` in the `oneconnect` namespace, plus its own Kafka Connect, `connect-<workspace-id>`, in the `kafka` namespace.

| Topic | Name |
|---|---|
| Raw topic (Producer writes, Consumer reads) | `<CLIENT>_<ENV>_RAWDATA` |
| Per-table topics (Consumer writes, Kafka Connect reads) | `<CLIENT>_<ENV>_<TABLE>` |
| Producer error topic | `<CLIENT>_<ENV>_INBOUND_ERRORS` |
| Consumer error topic | `<CLIENT>_<ENV>_PROCESSING_ERRORS` |

---

## 1. Localize the Break First

Do not guess. Find the first stage with no data.

**Stage 0 — are the SAP Link's workloads running?**

```bash
kubectl get pods -n oneconnect | grep <workspace-id>
kubectl get pods -n kafka -l workspace=<workspace-id>
```

If the Producer or Consumer pods are missing or not `Running`, go to [§8](#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running).

**Stage A — did SAP reach the Producer?**

Check the response SAP receives for its requests (SM59 connection test or the SAP-side log). To test the Producer directly, use the port-forward test in [§3](#3-sap-is-not-reaching-the-producer).

For more detail, you can raise the Producer log level (see [01 §4](./01-Diagnostic_Toolkit.md#4-raising-the-log-level-to-see-more-detail)) and search for received requests:

```bash
kubectl logs -n oneconnect deploy/<workspace-id>-producer --tail=200 | grep 'Starting to process'
```

**Stages B to D** are read in AKHQ. Get its address with:

```bash
kubectl get svc -n kafka | grep akhq
```

| Stage | Question | Where in AKHQ |
|---|---|---|
| B | Is the raw topic growing? | **Topics** → `<CLIENT>_<ENV>_RAWDATA` — watch the record count |
| C | Is the Consumer reading? | **Consumer Groups** → `<CLIENT>_<ENV>` — check members and lag |
| C | Are per-table topics growing? | **Topics** — one `<CLIENT>_<ENV>_<TABLE>` topic per SAP table, with its record count |
| D | Is anything recorded for review? | **Topics** → `<CLIENT>_<ENV>_PROCESSING_ERRORS` and `<CLIENT>_<ENV>_INBOUND_ERRORS` — check the record counts |

| First stage with no data | Break is | Go to |
|---|---|---|
| 0 — pods missing or not `Running` | SAP Link deployment | [§8](#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) |
| A — SAP reports a connection error, timeout, or `404` | SAP → Producer | [§3](#3-sap-is-not-reaching-the-producer) |
| B — raw topic empty or missing | Producer → Kafka | [§4](#4-the-producer-is-not-writing-to-kafka) |
| C — raw topic grows, group missing or offsets static | Consumer not consuming | [§2](#2-the-consumer-is-subscribed-to-a-topic-nobody-writes-to), [04 §7](./04-Kafka_Connectivity.md#7-the-consumer-connects-but-consumes-nothing) |
| C — offsets advance but no per-table topics | Consumer cannot produce | [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md) |
| D — error topics growing | Records set aside for review | [§7](#7-reviewing-and-reprocessing-records-from-the-error-topics) |
| All topics grow, destination empty | Kafka Connect / sink | [§5](#5-kafka-connect-is-not-delivering-to-the-destination) |

---

## 2. The Consumer Is Subscribed to a Topic Nobody Writes To

### The naming contract

The Producer and the Consumer each build the raw topic name from their own configuration:

| Component | Property | Builds |
|---|---|---|
| Producer **writes to** | `sapproperties.client`, `sapproperties.environment`, `sapproperties.postfix` | `<CLIENT>_<ENV>_<POSTFIX>` |
| Consumer **reads from** | `sapproperties.client`, `sapproperties.environment`, `sapproperties.regexsource` | `<CLIENT>_<ENV>_<REGEXSOURCE>` |

For SAP Links created from the platform, the contract is met automatically: both workloads receive the same `client` and `environment` from the SAP Link, and both `postfix` and `regexsource` use the default `RAWDATA`. Check this section when the variables were changed on the Deployments by hand, or for workloads deployed outside the platform. The Consumer's **`regexsource` must equal the Producer's `postfix`**.

> 📌 The Consumer always reads `<CLIENT>_<ENV>_<REGEXSOURCE>`. The `useprefix` flag (default `ENABLE`) applies to the topics it **writes** — per-table topics and `PROCESSING_ERRORS` — not to the topic it reads.

### Symptom

The raw topic grows while the Consumer group's offsets stay flat or the group is absent. When the Consumer is subscribed to a topic that does not exist, the Kafka client logs a warning such as:

```
Error while fetching metadata with correlation id <N> : {<topic>=UNKNOWN_TOPIC_OR_PARTITION}
```

### Confirm

```bash
kubectl exec -n oneconnect deploy/<workspace-id>-producer -- env | grep -iE 'SAPPROPERTIES_(CLIENT|ENVIRONMENT|POSTFIX)'
kubectl exec -n oneconnect deploy/<workspace-id>-consumer -- env | grep -iE 'SAPPROPERTIES_(CLIENT|ENVIRONMENT|REGEXSOURCE|USEPREFIX)'
```

A variable that is not listed uses the image default (`RAWDATA` for `POSTFIX` and `REGEXSOURCE`). Compose both names and compare them with the topic list in **AKHQ → Topics**.

### Typical mismatches

| Producer | Consumer | Result |
|---|---|---|
| `postfix = RAWDATA` | `regexsource = RAW_DATA` | Consumer waits on a topic nothing writes to |
| `environment = DEV` | `environment = QAS` | Different topic entirely |
| `client = 732` | `client = 0732` | Different topic entirely |
| `postfix = RAWDATA` | `regexsource = rawdata` | **Kafka topic names are case-sensitive** — different topic |

### Fix

1. Align the values on both Deployments.
2. Restart the Consumer:

   ```bash
   kubectl rollout restart deployment/<workspace-id>-consumer -n oneconnect
   ```

A new consumer group starts from `earliest`, so once pointed at the right name the Consumer also processes the records already accumulated in the raw topic (within its retention period).

---

## 3. SAP Is Not Reaching the Producer

### Symptom

SAP reports a connection error, a timeout, or `404`, and the raw topic does not grow.

### Confirm — test the Producer directly

Platform-deployed Producers listen on port `8080` behind the Service `producer-<workspace-id>`:

```bash
kubectl port-forward -n oneconnect svc/producer-<workspace-id> 8080:8080
```

In a second terminal:

```bash
# Does the endpoint answer? (GET does not authenticate)
curl -s -o /dev/null -w "%{http_code}\n" \
  http://localhost:8080/api/v1/saplistener/<workspace-id>/synchronous

# Send a test request
curl -s -w "\n%{http_code}\n" \
  -u '<user>:<password>' \
  -X POST -H 'Content-Type: application/json' \
  -d '{"test":"ping"}' \
  http://localhost:8080/api/v1/saplistener/<workspace-id>/synchronous
```

| Result | Meaning | Next |
|---|---|---|
| `200` on `GET` | The Producer is up and the path is correct | Send the `POST` |
| `201` plus an offset on `POST` | The whole Producer path works — the break is between SAP and the cluster | Check the table below |
| `401` | Credentials | [03 — Authentication](./03-Authentication_and_Credentials.md) |
| `400` / `500` | Header, payload, or Kafka result | [03 — Authentication](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| `404` | The path or workspace ID differs from this Producer's | Use the SAP Link URL shown by the platform |
| No response | The Producer is not listening | Check pod status — [§8](#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) |

> ⚠️ On production, use the `GET` connection test, or send a valid message from SAP.

### If the Producer works but SAP traffic is not arriving

| Cause | Check |
|---|---|
| SM59 destination differs from the SAP Link URL | Host, port, and full path including the workspace ID — use the URL shown by the platform as-is. See [Endpoint Customization (SM59)](../SAP_Data_Modeler/Data_Modeler_Manuals/15-Endpoint_Customization_SM59.md) |
| Ingress path not present for this SAP Link (ingress mode) | `kubectl get ingress producer-oneconnect -n oneconnect -o yaml \| grep <workspace-id>`. If the path is missing, the Builder log shows `ERROR sending request:` at the time the SAP Link was created; remove the SAP Link and create it again once the cause is fixed |
| Load balancer address not yet assigned (load balancer mode) | The platform reports `LoadBalancer external address not yet assigned for workspace: <workspace-id>`. Check `kubectl get svc producer-<workspace-id>-loadbalancer -n oneconnect` and retry once `EXTERNAL-IP` is set — see [02 §7](./02-Deployment_and_Installation.md#7-no-external-ip--cannot-reach-the-ui) |
| SAP cannot reach the cluster | Firewall, proxy, or Cloud Connector — on BTP see the [Kyma troubleshooting](../Smart_Gateway/BTP/SmartGateway_BTP_Kyma_Deployment.md#troubleshooting) |
| The entity is not scheduled or activated in SAP | See [Entity Execution (Batch Mode)](../SAP_Data_Modeler/Data_Modeler_Manuals/04-Entity_Execution_Batch_Mode.md) |
| SAP is sending to a different SAP Link | Compare the workspace ID in the SM59 path with the SAP Link |
| SAP times out before the Producer answers | See [04 §9](./04-Kafka_Connectivity.md#9-sap-waits-a-long-time-for-the-producers-response) |

---

## 4. The Producer Is Not Writing to Kafka

### Symptom

SAP receives `201`, but the raw topic is missing or its record count does not change.

### Cause — choose the endpoint that matches the confirmation you need

The Producer offers two endpoints for each SAP Link:

| Endpoint | Path | Replies | Use it when |
|---|---|---|---|
| **Synchronous** *(recommended; the URL shown by the platform)* | `/api/v1/saplistener/<workspace-id>/synchronous` | `201` with the Kafka offset once Kafka has stored the record; `400` with the Kafka result otherwise | You want delivery confirmation in SAP |
| Asynchronous | `/api/v1/saplistener/<workspace-id>` | `201` on receipt | You want SAP to send as fast as possible |

The asynchronous endpoint confirms receipt; for delivery confirmation use the synchronous endpoint (the URL shown by the platform).

### Confirm

| Check | Command / Action |
|---|---|
| Which endpoint does SAP call? | Compare the SM59 **Path Prefix** with the SAP Link URL (it ends in `/synchronous`) |
| Does the topic exist? | **AKHQ → Topics**; if absent, see [04 §4](./04-Kafka_Connectivity.md#4-topics-are-never-created) |
| What did Kafka answer? | Synchronous: `kubectl logs ... \| grep -- '-->ExecutionException'`. Asynchronous: the Producer log shows Kafka's message, e.g. `Topic <topic> not present in metadata after 60000 ms.` |
| Is the payload valid JSON? | Payloads that are not valid JSON are logged as `-->MalformedJsonException =...` and stored in `<CLIENT>_<ENV>_INBOUND_ERRORS` |
| Are records too large? | Look for `RecordTooLargeException` — [04 §5](./04-Kafka_Connectivity.md#5-message-size-limits) |

```bash
kubectl logs -n oneconnect deploy/<workspace-id>-producer --since=30m | grep -iE 'ExecutionException|MalformedJson|metadata|RecordTooLarge'
```

### Fix

1. Use the SAP Link URL as shown by the platform (synchronous) in SM59, so SAP receives Kafka's answer for every request.
2. Resolve the Kafka condition reported in the log with [04 — Kafka Connectivity](./04-Kafka_Connectivity.md).
3. Re-send the affected records from SAP — [§7](#7-reviewing-and-reprocessing-records-from-the-error-topics).

---

## 5. Kafka Connect Is Not Delivering to the Destination

### Symptom

The per-table topics are growing, so the OneConnect side is healthy, but the destination table does not change.

### Confirm

Each SAP Link has its own Kafka Connect, `connect-<workspace-id>`, in the `kafka` namespace. In AKHQ, open **Connect** → `connect-<workspace-id>` to see each connector, its state, and its tasks. The same status is available from the REST API:

```bash
kubectl get pods -n kafka -l workspace=<workspace-id>

kubectl port-forward -n kafka svc/connect-<workspace-id> 8083:8083
# in a second terminal
curl http://localhost:8083/connectors
curl http://localhost:8083/connectors/<connector-name>/status
```

If Kafka Connect does not answer, see [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable).

| `state` | Meaning | Action |
|---|---|---|
| `RUNNING` with running tasks | Healthy — the problem is elsewhere | Verify you are querying the right destination table |
| `FAILED` | Read `trace` in the status output | Usually credentials, schema, or a missing table |
| `PAUSED` | Paused | Resume it from the **Connect** section in AKHQ |
| Tasks `FAILED`, connector `RUNNING` | Task-level failure | Restart the task from the **Connect** section in AKHQ, or restart the connector from the platform |
| Connector not listed | Not yet created | Create it — [Creating Connectors](../Smart_Gateway/Using_the_Smart_Gateway/05-Creating_Connectors.md) |

### Fix — common sink failures

| Symptom in `trace` | Cause | Fix |
|---|---|---|
| Authentication / login failed | Destination credentials differ from the database's | Update the connector configuration |
| Table or schema does not exist | Auto-create disabled | Enable auto-create, or pre-create the table |
| Column type mismatch | Avro type differs from the existing column | See [05 §9](./05-Avro_Serialization_and_Schema_Registry.md#9-records-serialize-but-the-destination-has-wrong-values) |
| Primary key missing | Upsert mode needs a key | Confirm the entity has key fields defined |
| Schema Registry `401` or unreachable | Kafka Connect reads the Schema Registry URL and credentials from the SAP Link's Kafka secret | Make sure the secret includes **Schema Registry URL** and credentials, then configure Kafka Connect again |
| Topic not read by the connector | The connector's `topics` list or `topics.regex` does not include this table's topic | Add the topic to `topics`, or adjust `topics.regex` so it matches the table's topic |

> 📌 The Consumer creates a topic per table the first time it sees it, and each connector reads the topics in its configuration. After adding a table in SAP, make sure the connector covers its topic — see [Creating Connectors](../Smart_Gateway/Using_the_Smart_Gateway/05-Creating_Connectors.md).

> ℹ️ In `ONPREMISE` mode the Consumer can also register a sink connector by itself at `kafkaproperties.connecturl` (outcome message: `-->Failed to create SinkConnector for table=<table> Error=...`). Connectors created from the platform are managed independently of this option; use the connector status above for them.

---

## 6. Data Arrives, but Only Partially

| Symptom | Cause | Fix |
|---|---|---|
| Only some tables arrive | The connector does not cover every per-table topic | Add the missing topics — [§5](#5-kafka-connect-is-not-delivering-to-the-destination) |
| Some SAP messages arrive, others do not | The Consumer converts each SAP message as a whole; messages it cannot convert are stored in `<CLIENT>_<ENV>_PROCESSING_ERRORS` | Review them — [§7](#7-reviewing-and-reprocessing-records-from-the-error-topics) and [05](./05-Avro_Serialization_and_Schema_Registry.md) |
| Individual rows of a message are missing | The row was converted but its send to the per-table topic was refused (size, Schema Registry, ACL) and recorded in the error topic | Check the `-->Error Sending Record` lines — [§7](#7-reviewing-and-reprocessing-records-from-the-error-topics) |
| Data stops after a fixed period | Topic retention expired before the sink read it | Retention comes from the SAP Link's Topic Settings; restore the sink and re-send the period from SAP |
| Duplicates in the destination | Records delivered again after a rebalance or Consumer restart | Use `upsert` mode; see [04 §8](./04-Kafka_Connectivity.md#8-rebalances-and-poll-timeouts) |
| Deleted rows persist | Tombstones ignored by the sink | Enable delete handling on the connector |
| Data stopped at a specific time | Consumer restarting, or lag growing | Check pod restarts and consumer lag |

```bash
# Has the Consumer been restarting?
kubectl get pods -n oneconnect | grep <workspace-id>-consumer
kubectl describe pod -n oneconnect <consumer-pod> | grep -A5 'Last State'
```

---

## 7. Reviewing and Reprocessing Records from the Error Topics

Records that cannot be processed are set aside in an error topic so that you can review them and send them again once the cause is resolved.

### Where to look

Open **AKHQ → Topics → `<topic>` → Data**:

| Topic | Written by | Entry contains |
|---|---|---|
| `<CLIENT>_<ENV>_INBOUND_ERRORS` | Producer | The request body exactly as received from SAP, when it was not valid JSON |
| `<CLIENT>_<ENV>_PROCESSING_ERRORS` | Consumer | A text entry (Avro string, decoded by AKHQ) in the form below |

```
Failed to send message
Error: <error message>
Data: <data>
```

### What the `Data:` block holds

Match the entry's time with the Consumer log to see which kind it is:

| Consumer log line | `Data:` holds | Recommended reprocessing |
|---|---|---|
| `Error processing record, will send to error topic, errorType=..., cause=...` | The **complete SAP message** (the JSON with its `oneconnect` envelope) | Re-send from SAP, or replay the `Data:` block (Option B) |
| `-->Error Sending Record to topic=<topic>, key=..., value=..., errorType=..., cause=...` | **One converted row** for that topic — not a SAP message | Re-send from SAP (Option A) |
| `Unhandled exception in consumerFactory at partition:... offset:..., errorType=...` | Empty, or the last record read — this records a poll-level condition (ACLs, connectivity) | Resolve the condition — [04 §6](./04-Kafka_Connectivity.md#6-acls-and-authorization); re-send affected periods from SAP |

### Option A — re-send from SAP (recommended)

Resolve the root cause, then use [Reprocessing Records](../SAP_Data_Modeler/Data_Modeler_Manuals/12-Reprocessing_Records.md) to send the affected records again. This path works for every kind of entry, including `INBOUND_ERRORS` payloads, which need correcting on the SAP side first.

### Option B — replay a complete SAP message from the error topic

For entries whose `Data:` block is a complete SAP message:

1. Resolve the root cause first, so the replayed message processes successfully.
2. Copy the `Data:` block into a file, for example `record.json`. AKHQ's download option on the Data tab exports the displayed messages when you need more than a few.
3. Post it to the SAP Link URL with the SAP Link's credentials:

   ```bash
   curl -s -w "\n%{http_code}\n" \
     -u '<user>:<password>' \
     -X POST -H 'Content-Type: application/json' \
     --data @record.json \
     '<SAP Link URL>'
   ```

4. A `201` with an offset confirms the message is back in the raw topic; follow it through with [§10](#10-end-to-end-verification-after-a-fix).

---

## 8. The SAP Link Appears in the UI, but Its Pods Are Not Running

### Symptom

The SAP Link was created and is listed in the platform, but `kubectl get pods -n oneconnect | grep <workspace-id>` shows no Producer/Consumer pods, or shows them in a state other than `Running`.

### Cause

The SAP Link is saved first and its workloads are then created in Kubernetes. If Kubernetes declines a request, the Builder logs `ERROR sending request:` followed by Kubernetes' answer. Pods can also be held back by namespace quotas or by the pod's configuration.

### Confirm

**Step 1 — do the Deployments exist?**

```bash
kubectl get deploy,svc -n oneconnect | grep <workspace-id>
```

If they are missing, read the Builder log around the creation time:

```bash
kubectl logs -n oneconnect deploy/builder --since=1h | grep -A3 'ERROR sending request'
```

| Builder log after `ERROR sending request:` | Cause | Fix |
|---|---|---|
| `HttpClientErrorException$Forbidden: 403 Forbidden ...` | The Builder's service account lacks permission in `oneconnect` | Install with `rbac.create: true` (Role `builder-role`: deployments, services, secrets, ingresses) — see [02 §5](./02-Deployment_and_Installation.md#5-a-sap-link-was-created-but-no-producer-pod-appears) |
| `HttpClientErrorException$UnprocessableEntity: 422 ...` mentioning the container `resources` | The SAP Link's **Advanced Settings** values are not valid Kubernetes quantities, or a request exceeds its limit | Use quantities such as `250m` / `1` for CPU and `512Mi` / `1Gi` for memory, with each request ≤ its limit |
| `java.io.IOException: The file does not exists: ...token` | The Builder pod has no service account token mounted | Keep `automountServiceAccountToken` enabled for the Builder |

**Step 2 — the Deployments exist, but there are no pods**

```bash
kubectl describe deploy -n oneconnect <workspace-id>-producer | tail -20
kubectl get events -n oneconnect --sort-by=.lastTimestamp | grep <workspace-id>
```

Messages such as `exceeded quota` or `forbidden: maximum cpu usage per Container` come from a namespace `ResourceQuota` or `LimitRange`. Each SAP Link runs `KUBERNETES_WORKSPACE_REPLICAS` replicas of both its Producer and Consumer (Helm value `services.builder.env.KUBERNETES_WORKSPACE_REPLICAS`), so the quota must cover the Advanced Settings requests multiplied by that number, for both workloads.

**Step 3 — pods exist but are not `Running`**

| Pod status / message | Go to |
|---|---|
| `ImagePullBackOff` / `ErrImagePull` | [02 §2](./02-Deployment_and_Installation.md#2-imagepullbackoff--errimagepull) |
| `Pending` | [02 §3](./02-Deployment_and_Installation.md#3-pods-stuck-in-pending) |
| Consumer `CreateContainerConfigError`: `couldn't find key KAFKAPROPERTIES_SCHEMAREGISTRYURL in Secret oneconnect/<secret-name>` (or `KAFKAPROPERTIES_SCHEMAPASSWORD`) | The Consumer reads the Schema Registry URL and credentials from the SAP Link's Kafka secret. Create the SAP Link with a secret that includes **Schema Registry URL** and credentials |
| `CrashLoopBackOff` | `kubectl logs -n oneconnect <pod> --previous`; for Kafka startup messages see [04 §2](./04-Kafka_Connectivity.md#2-cannot-reach-the-brokers) |

### Fix

1. Correct the cause identified above.
2. For Step 1 causes, retry the creation: remove the SAP Link and create it again once the cause is fixed.
3. For Step 2 and Step 3 causes, the existing Deployments start their pods once the condition is resolved (quota raised, node capacity or image available). For a Kafka secret without Schema Registry settings, create the SAP Link again with a secret that includes them.
4. Confirm with `kubectl get pods -n oneconnect | grep <workspace-id>` and continue with [§1](#1-localize-the-break-first).

---

## 9. Creating a SAP Link Fails: Workspace Limit Reached

### Symptom

Creating a SAP Link returns:

```
Workspace Max limit (N) has been reached. It is not possible to create more.
```

### Cause

The limit counts the SAP Links in the installation. It is set with the Helm value `services.builder.env.KUBERNETES_MAX_WORKSPACES`; `"0"` means no limit.

### Confirm

```bash
kubectl exec -n oneconnect deploy/builder -- env | grep KUBERNETES_MAX_WORKSPACES
```

Compare the value with the number of SAP Links listed in the platform.

### Fix

1. Remove SAP Links you no longer use, **or**
2. Raise `services.builder.env.KUBERNETES_MAX_WORKSPACES` (or set `"0"` for no limit) and upgrade the release — see [02 §8](./02-Deployment_and_Installation.md#8-upgrades).

---

## 10. End-to-End Verification After a Fix

Confirm the repair with a single traceable record. Work through this in **AKHQ**, which lists every topic with its current record count.

| # | Step | Expected |
|---|---|---|
| 1 | Note the record count of `<CLIENT>_<ENV>_RAWDATA` | — |
| 2 | Trigger a small extraction for one table from the SAP side | SAP receives `201` with an offset (synchronous endpoint) |
| 3 | Re-check the raw topic | Count increased |
| 4 | Open **Consumer Groups** → `<CLIENT>_<ENV>` | `LAG` returns to 0 |
| 5 | Check the per-table topic `<CLIENT>_<ENV>_<TABLE>` | Count increased |
| 6 | Check `<CLIENT>_<ENV>_PROCESSING_ERRORS` | Count unchanged |
| 7 | Open **Connect** → `connect-<workspace-id>` | Connector and tasks `RUNNING` |
| 8 | Query the destination table | New rows present |

Counts advancing at steps 3 and 5, the error topic flat at step 6, the connector running at step 7, and rows present at step 8 mean the pipeline is healthy end to end.

---

## Summary

1. **Localize before diagnosing** — check the SAP Link's pods, then count records at each stage and find the first one that is empty.
2. Platform SAP Links meet the **topic-naming contract** automatically (`<CLIENT>_<ENV>_RAWDATA`); check it when variables were edited by hand. Topic names are case-sensitive.
3. The synchronous endpoint (the URL shown by the platform) confirms delivery with the Kafka offset; the asynchronous endpoint confirms receipt.
4. Each SAP Link has its own Kafka Connect, `connect-<workspace-id>`; per-table topics growing while the destination stays empty points there.
5. Error topics hold records set aside for review. Re-sending from SAP is the recommended reprocessing path; entries holding a complete SAP message can also be replayed to the SAP Link URL.
6. A SAP Link without running pods: read the Builder log for `ERROR sending request:`, then check quotas, Advanced Settings values, and the Kafka secret.
7. `Workspace Max limit (N) has been reached`: remove SAP Links you no longer use, or raise `KUBERNETES_MAX_WORKSPACES`.
