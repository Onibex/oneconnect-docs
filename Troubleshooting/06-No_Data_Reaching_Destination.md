# No Data Reaching Destination

When every pod is `Running` and SAP reports success but the destination table is empty, this guide walks the pipeline stage by stage to find where records stop.

---

## 1. Localize the Break First

Do not guess. Run these four counts and find the first stage with no data.

**Stage A — did SAP reach the Producer?**

```bash
kubectl logs -n oneconnect <producer-pod> --tail=100
```

**Stages B to D** are read in AKHQ. Get its address with:

```bash
kubectl get svc -n kafka | grep akhq
```

| Stage | Question | Where in AKHQ |
|---|---|---|
| B | Is the raw topic growing? | **Topics** → `<CLIENT>_<ENV>_<POSTFIX>` — watch the record count |
| C | Is the Consumer reading? | **Consumer Groups** → `group_0` — check members and lag |
| C | Are per-table topics growing? | **Topics** — one topic per SAP table, with its record count |
| D | Is anything failing? | **Topics** → `PROCESSING_ERRORS` — check the record count |

| First stage with no data | Break is | Go to |
|---|---|---|
| A — no requests logged | SAP → Producer | [§3](#3-sap-is-not-reaching-the-producer) |
| B — raw topic empty or missing | Producer → Kafka | [§4](#4-the-producer-is-not-writing-to-kafka) |
| C — raw topic grows, group missing or lag static | Consumer not consuming | [§2](#2-the-consumer-is-subscribed-to-a-topic-nobody-writes-to) |
| C — lag drops but no per-table topics | Consumer failing to produce | [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md) |
| D — `PROCESSING_ERRORS` growing | Data quality | [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md) |
| All topics grow, destination empty | Kafka Connect / sink | [§5](#5-kafka-connect-is-not-delivering-to-the-destination) |

---

## 2. The Consumer Is Subscribed to a Topic Nobody Writes To

A frequent explanation when the pipeline is idle and no errors are reported.

### The naming contract

The Producer and the Consumer each build the raw topic name independently, from separate configuration:

| Component | Property | Builds |
|---|---|---|
| Producer **writes to** | `sapproperties.client`, `sapproperties.environment`, `sapproperties.postfix` | `<CLIENT>_<ENV>_<POSTFIX>` |
| Consumer **reads from** | `sapproperties.client`, `sapproperties.environment`, `sapproperties.regexsource` | `<CLIENT>_<ENV>_<REGEXSOURCE>` |

All three parts must match. In particular, the Consumer's **`regexsource` must equal the Producer's `postfix`** — they are different property names for the same value, which is exactly why they drift.

> 📌 The Consumer builds its **input** topic name without the `useprefix` flag, and applies `useprefix` to its **output** and **error** topics. `useprefix` therefore affects where results are written, not where input is read from.

### Diagnosis

```bash
kubectl exec -n oneconnect <producer-pod> -- env | grep -iE 'client|environment|postfix'
kubectl exec -n oneconnect <consumer-pod> -- env | grep -iE 'client|environment|regexsource|useprefix'
```

Compose both names by hand and compare them against the topic list in **AKHQ → Topics**.

A raw topic that exists and grows, alongside a consumer group whose `CURRENT-OFFSET` stays flat — or that is absent — points to a mismatch.

### Typical mismatches

| Producer | Consumer | Result |
|---|---|---|
| `postfix = RAWDATA` | `regexsource = RAW_DATA` | Consumer waits on a topic nothing writes to |
| `environment = DEV` | `environment = QAS` | Different topic entirely |
| `client = 732` | `client = 0732` | Different topic entirely |
| `postfix = RAWDATA` | `regexsource = rawdata` | **Kafka topic names are case-sensitive** — different topic |

### Fix

Align the values and restart the Consumer:

```bash
kubectl rollout restart deployment/<consumer-deployment> -n oneconnect
```

Because the Consumer starts from `earliest`, it will pick up everything already accumulated in the raw topic once pointed at the right name — no data is lost.

---

## 3. SAP Is Not Reaching the Producer

### Check whether requests arrive at all

```bash
kubectl logs -n oneconnect <producer-pod> --tail=200
```

Successful requests are logged at `DEBUG` (`-->Request has been received, Starting to process`), so at the default `INFO` level a healthy Producer produces little output. To confirm traffic is arriving, raise the level temporarily or test the endpoint directly.

```bash
# Does the endpoint answer?
kubectl exec -n oneconnect <producer-pod> -- \
  curl -s -o /dev/null -w "%{http_code}\n" \
  http://localhost:50501/api/v1/saplistener/<workspace-id>
```

`200` means the Producer is up and routing correctly.

### Then send a real payload

```bash
kubectl exec -n oneconnect <producer-pod> -- \
  curl -s -w "\n%{http_code}\n" \
  -u '<user>:<password>' \
  -X POST -H 'Content-Type: application/json' \
  -d '{"test":"ping"}' \
  http://localhost:50501/api/v1/saplistener/<workspace-id>/synchronous
```

| Result | Meaning | Next |
|---|---|---|
| `201` plus an offset | The whole Producer path works — the problem is SAP-side or in the network | Check SM59 |
| `401` | Credentials | [03 — Authentication](./03-Authentication_and_Credentials.md) |
| `400` / `500` | Header or Kafka problem | [03 — Authentication](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| No response | The Producer is not listening | Check pod status and port |

### If the endpoint works but SAP traffic is not arriving

| Cause | Check |
|---|---|
| SM59 destination misconfigured | Host, port, full path including workspace ID — see [Endpoint Customization (SM59)](../SAP_Data_Modeler/Data_Modeler_Manuals/15-Endpoint_Customization_SM59.md) |
| Ingress or LoadBalancer not exposing the Producer | `kubectl get ingress -n oneconnect`, `kubectl get svc -n oneconnect` |
| SAP cannot reach the cluster | Firewall, proxy, or Cloud Connector — on BTP see the [Kyma troubleshooting](../Smart_Gateway/BTP/SmartGateway_BTP_Kyma_Deployment.md#troubleshooting) |
| The entity is not scheduled or activated in SAP | See [Entity Execution (Batch Mode)](../SAP_Data_Modeler/Data_Modeler_Manuals/04-Entity_Execution_Batch_Mode.md) |
| SAP is sending to a different SAP Link | Compare the workspace ID in the SM59 path with the SAP Link |

---

## 4. The Producer Is Not Writing to Kafka

SAP gets `201`, but the raw topic is missing or static.

`201` confirms the request was accepted. The asynchronous endpoint returns before the Kafka send completes, so use the synchronous endpoint when you need confirmation that the record reached Kafka.

| Check | Command / Action |
|---|---|
| Does the topic exist? | Check **AKHQ → Topics**; if absent, enable `DEBUG` to see the topic creation outcome. See [04 — Kafka Connectivity](./04-Kafka_Connectivity.md#0-before-you-start-turn-on-the-detail) |
| Are sends succeeding? | Raise to `DEBUG` and watch the send outcome; the async path reports at `INFO`, so check `kubectl logs` |
| Is the payload valid JSON? | Invalid JSON goes to `<CLIENT>_<ENV>_INBOUND_ERRORS`, not the raw topic |
| Are records too large? | Look for `RecordTooLargeException` — see [04 §5](./04-Kafka_Connectivity.md#5-message-size-limits) |

Malformed payloads from SAP land in `<CLIENT>_<ENV>_INBOUND_ERRORS`. Read it in **AKHQ → Topics → that topic → Data**.

Use the synchronous endpoint to test: it returns the actual Kafka offset, so a `201` with an offset proves the record was persisted.

---

## 5. Kafka Connect Is Not Delivering to the Destination

The per-table topics are growing, so the OneConnect side is healthy. The break is in the sink.

Kafka Connect runs as a Strimzi `KafkaConnect` resource in the `kafka` namespace:

```bash
kubectl get kafkaconnect -n kafka
kubectl get pods -n kafka | grep -i connect
```

If Kafka Connect has been provisioned for the SAP Link, its connectors appear in AKHQ under **Connect**, showing each connector, its state, and its tasks.

To query the Connect REST API directly from its pod:

```bash
kubectl exec -it connect-cluster-connect-0 -n kafka -- \
  curl http://localhost:8083/connectors

kubectl exec -it connect-cluster-connect-0 -n kafka -- \
  curl http://localhost:8083/connectors/<connector-name>/status
```

| `state` | Meaning | Action |
|---|---|---|
| `RUNNING` with running tasks | Healthy — the problem is elsewhere | Verify you are querying the right destination table |
| `FAILED` | Read `trace` in the status output | Usually credentials, schema, or a missing table |
| `PAUSED` | Manually paused | Resume it from the **Connect** section in AKHQ |
| Tasks `FAILED`, connector `RUNNING` | Task-level failure | Restart the task from the **Connect** section in AKHQ |
| Connector not listed | Not yet created | See below |

### Common sink failures

| Symptom in `trace` | Cause | Fix |
|---|---|---|
| Authentication / login failed | Destination credentials wrong | Update the connector configuration |
| Table or schema does not exist | Auto-create disabled | Enable auto-create, or pre-create the table |
| Column type mismatch | Avro type differs from the existing column | See [05 §9](./05-Avro_Serialization_and_Schema_Registry.md#9-records-serialize-but-the-destination-has-wrong-values) |
| Primary key missing | Upsert mode needs a key | Confirm the entity has key fields defined |
| Schema Registry unreachable from Connect | Connect needs its own registry configuration | Verify Connect's `value.converter.schema.registry.url` |
| Topic not in the connector's topic list | The connector was created before this table existed | Add the topic to the connector configuration |

> 📌 The Consumer creates a topic per table on first sight, and the sink connector writes the topics it is configured for. After adding a table in SAP, add its topic to the connector's topic list — see [Creating Connectors](../Smart_Gateway/Using_the_Smart_Gateway/05-Creating_Connectors.md).

### On-premise automatic connector creation

In `ONPREMISE` mode the Consumer creates a sink connector automatically for each new topic, reporting the outcome at `DEBUG`. Enable `DEBUG` to see it:

```bash
kubectl set env deployment/<consumer> -n oneconnect LOGGING_LEVEL_ROOT=DEBUG
kubectl logs -n oneconnect <consumer-pod> -f | grep -i 'SinkConnector'
```

Look for `-->Failed to create SinkConnector for table=...`. Typical causes: `kafkaproperties.connecturl` wrong or unreachable, or the destination database configuration (`connecttargetdatabaseurl`, `dbuser`, `dbpassword`) invalid.

---

## 6. Data Arrives, but Only Partially

| Symptom | Cause | Fix |
|---|---|---|
| Only some tables arrive | The sink connector's topic list is incomplete | Add the missing topics |
| Only some rows arrive | The rest failed conversion | Check `PROCESSING_ERRORS` — [05](./05-Avro_Serialization_and_Schema_Registry.md) |
| Roughly half the records arrive | Two Consumers sharing `group_0` split the partitions | Give each SAP Link its own `groupid` — [04 §7](./04-Kafka_Connectivity.md#two-consumers-one-group) |
| Data stops after a fixed period | Topic retention expired before the sink read it | Default retention is 7 days; raise `retentiontime` or fix the sink |
| Duplicates in the destination | Consumer evicted mid-batch and reprocessed | Use `upsert` mode; see [04 §8](./04-Kafka_Connectivity.md#8-rebalances-and-poll-timeouts) |
| Deleted rows persist | Tombstones ignored by the sink | Enable delete handling on the connector |
| Data stopped at a specific timestamp | Consumer crashed or lag is growing | Check pod restarts and consumer lag |

```bash
# Has the Consumer been restarting?
kubectl get pods -n oneconnect | grep -i consumer
kubectl describe pod -n oneconnect <consumer-pod> | grep -A5 'Last State'
```

---

## 7. Recovering Failed Records

Once the root cause is fixed, recover what failed.

### Option A — re-send from SAP

The authoritative recovery. See [Reprocessing Records](../SAP_Data_Modeler/Data_Modeler_Manuals/12-Reprocessing_Records.md).

### Option B — extract payloads from the error topic

The error topics carry the original payload in their `Data:` block. For a small number of records, open **AKHQ → Topics → `PROCESSING_ERRORS` → Data**, copy the `Data:` payload from each entry, and POST it back to the Producer endpoint. AKHQ's download option on the Data tab exports the displayed messages when you need more than a few.

> 📌 Resolve the root cause before replaying, so the replayed records process successfully.

---

## 8. End-to-End Verification After a Fix

Confirm the repair with a single traceable record.

Work through this in **AKHQ → Topics**, which lists every topic with its current record count.

| # | Step | Expected |
|---|---|---|
| 1 | Note the record count of the raw topic `<CLIENT>_<ENV>_<POSTFIX>` | — |
| 2 | Trigger a small extraction for one table from the SAP side | — |
| 3 | Re-check the raw topic | Count increased |
| 4 | Open **Consumer Groups** → `group_0` | `LAG` returns to 0 |
| 5 | Check the per-table topic for that table | Count increased |
| 6 | Check `PROCESSING_ERRORS` | Count unchanged |
| 7 | Query the destination table | New rows present |

Counts advancing in step at 3, 5, and 7, with the error topic flat at 6, means the pipeline is healthy end to end.

---

## Summary

1. **Localize before diagnosing** — count records at each of the four stages and find the first one that is empty.
2. Check the **topic-naming contract** early: the Consumer's `regexsource` must equal the Producer's `postfix`, with `client` and `environment` matching too. Topic names are case-sensitive.
3. A `201` from the asynchronous endpoint confirms acceptance; use the synchronous endpoint when you need the Kafka offset as confirmation.
4. Per-table topics growing while the destination stays empty points to **Kafka Connect**; new tables need adding to the connector's topic list.
5. In `ONPREMISE` mode, automatic sink connector creation reports its outcome at `DEBUG`.
6. Resolve the root cause before replaying, so the replayed records process successfully.
