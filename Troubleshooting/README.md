# Troubleshooting

Diagnostic guides for OneConnect. This folder is organized by **failure layer**, following the path data takes through the platform:

```
SAP  ──HTTP/Basic──▶  Producer  ──JSON──▶  Kafka  ──Avro──▶  Consumer  ──▶  Kafka Connect  ──▶  Destination
       (auth)          (00)                (topics)          (01)           (sink)             (DB / lake)
```

Most issues trace back to a single one of those arrows. Identify the arrow, then open the matching guide.

---

## 🚑 Start Here: Symptom Triage

| Symptom | Most likely layer | Go to |
|---|---|---|
| `helm install` fails, or pods do not reach `Running` | Deployment | [02 — Deployment & Installation](./02-Deployment_and_Installation.md) |
| Pod stuck in `ImagePullBackOff` / `ErrImagePull` | Docker Hub secret | [02 — Deployment & Installation](./02-Deployment_and_Installation.md#2-imagepullbackoff--errimagepull) |
| SAP Link created in the UI, but no Producer pod appears | RBAC / Builder | [02 — Deployment & Installation](./02-Deployment_and_Installation.md#5-a-sap-link-was-created-but-no-producer-pod-appears) |
| SAP returns **401** when calling the endpoint | Authentication | [03 — Authentication & Credentials](./03-Authentication_and_Credentials.md) |
| SAP returns **400** or **500** on a request that used to work | Authentication / payload | [03 — Authentication & Credentials](./03-Authentication_and_Credentials.md) |
| Cannot log in to the platform UI | Authentication | [03 — Authentication & Credentials](./03-Authentication_and_Credentials.md#7-cannot-log-in-to-the-platform-ui) |
| Producer starts but no topic is ever created | Kafka connectivity | [04 — Kafka Connectivity](./04-Kafka_Connectivity.md) |
| `TimeoutException`, `SaslAuthenticationException`, `SSLHandshakeException` | Kafka connectivity | [04 — Kafka Connectivity](./04-Kafka_Connectivity.md) |
| Records land in `*_PROCESSING_ERRORS` | Avro / serialization | [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md) |
| `NullPointerException`, `ClassCastException`, `NoSuchElementException` in the Consumer | Avro / metadata | [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md) |
| Schema Registry returns **401** or **409 Conflict** | Avro / Schema Registry | [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md) |
| SAP says "sent OK", Producer topic has data, but destination is empty | End-to-end flow | [06 — No Data Reaching Destination](./06-No_Data_Reaching_Destination.md) |
| Consumer runs, logs nothing, consumes nothing | Topic name mismatch | [06 — No Data Reaching Destination](./06-No_Data_Reaching_Destination.md#2-the-consumer-is-subscribed-to-a-topic-nobody-writes-to) |
| The Logs panel in the UI is empty | Observability | [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md#3-the-logs-panel-is-empty) |
| Need an exact count of failed records | Observability | [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md#counting-occurrences) |

> 💡 Start with [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md). It covers where errors surface, which log levels reach each place, and how to raise the verbosity when you need more detail.

---

## 📚 Guides in This Folder

### [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md)
How to see what is actually happening: the Logs panel in the UI, `kubectl logs`, the error topics, and the log-level rules that decide what reaches you. **Read this first.**

### [02 — Deployment & Installation](./02-Deployment_and_Installation.md)
Helm install/upgrade failures, the Docker Hub image pull secret, RBAC for dynamically created Producer/Consumer workloads, PVCs, LoadBalancers, and MySQL startup.

### [03 — Authentication & Credentials](./03-Authentication_and_Credentials.md)
Every credential in the chain: SAP → Producer Basic auth, Kafka SASL, Schema Registry basic auth, the internal soft security token, and platform UI login.

### [04 — Kafka Connectivity](./04-Kafka_Connectivity.md)
Bootstrap and DNS, SASL/SSL handshakes, topic auto-creation, partitions and replication factor, message size limits, and ACLs.

### [05 — Avro Serialization & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md)
SAP type → Avro type mapping, null handling, required key fields, missing metadata, schema evolution and compatibility conflicts.

### [06 — No Data Reaching Destination](./06-No_Data_Reaching_Destination.md)
The end-to-end checklist when every component looks healthy but the destination stays empty — including the topic-naming contract between Producer and Consumer, and how to reprocess from the error topics.

---

## 🧭 Component Reference

Quick orientation for the names used throughout these guides.

| Name in docs | What it is | Default port | Topic it writes |
|---|---|---|---|
| **Producer** | Receives HTTP payloads from SAP and writes raw JSON to Kafka | `50501` | `<CLIENT>_<ENV>_<POSTFIX>` |
| **Consumer** | Reads raw JSON, converts to Avro, writes one topic per SAP table | — | `[<CLIENT>_<ENV>_]<TABLE>` |
| **Builder** | Creates the Producer/Consumer workloads in Kubernetes when a SAP Link is created | `7071` | — |
| **Logs service** | Receives batched logs from Producer/Consumer and serves the UI Logs panel | `7072` | — |
| **Frontend** | The web UI | `5050` | — |

A Producer and a Consumer are **created per SAP Link** by the Builder — they are not part of the base Helm install. See [02](./02-Deployment_and_Installation.md#5-a-sap-link-was-created-but-no-producer-pod-appears) if they do not appear.

---

## 🔗 Related Documentation

- [Working with Logs](../Smart_Gateway/Using_the_Smart_Gateway/07-Working_with_Logs.md) — how to *use* the Logs panel (filters, export, alerts). This folder covers what to do when the logs show a problem.
- [Generating a SAP OTel Bridge Endpoint](../Smart_Gateway/Using_the_Smart_Gateway/08-Generating_SAP_OTel_Bridge_Endpoint.md) — surfacing SAP-side errors in the same panel.
- [Reprocessing Records](../SAP_Data_Modeler/Data_Modeler_Manuals/12-Reprocessing_Records.md) — re-sending data from the SAP side.
- [SmartGateway BTP/Kyma Deployment](../Smart_Gateway/BTP/SmartGateway_BTP_Kyma_Deployment.md#troubleshooting) — Kyma/Cloud Connector specific troubleshooting.
- [OneConnect FAQ](../Technical_Information/06-OneConnect_FAQ.md) — product capabilities and limits.

---

## 📩 Escalating to Support

If a guide does not resolve the issue, contact **contact@onibex.com** with:

1. The symptom and when it started.
2. `kubectl get pods -n oneconnect` output.
3. Producer and Consumer logs (see [01](./01-Diagnostic_Toolkit.md#2-reading-logs-directly-from-kubernetes)), **captured at `DEBUG` level** where possible.
4. A sample of the failing payload, with business data redacted.
5. Your SAP Link name and workspace ID.
