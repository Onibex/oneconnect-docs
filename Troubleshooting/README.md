# Troubleshooting

Diagnostic guides for OneConnect. This folder is organized by **layer**, following the path data takes through the platform:

```
SAP  ──HTTP/Basic──▶  Producer  ──JSON──▶  Kafka  ──Avro──▶  Consumer  ──▶  Kafka Connect  ──▶  Destination
       (auth)                              (topics)                          (sink)             (DB / lake)
```

Most issues trace back to a single one of those arrows. Find your symptom below, or look up the exact message in the [Error Reference](./07-Error_Reference.md).

---

## 🚑 Start Here: Symptom Triage

### Installation and platform

| Symptom | Go to |
|---|---|
| `helm install` fails, or pods do not reach `Running` | [02 §1 — `helm install` fails immediately](./02-Deployment_and_Installation.md#1-helm-install-fails-immediately) |
| Pod in `ImagePullBackOff` / `ErrImagePull` | [02 §2 — Image pull](./02-Deployment_and_Installation.md#2-imagepullbackoff--errimagepull) |
| Pods or PVCs stuck in `Pending` | [02 §3 — Pods `Pending`](./02-Deployment_and_Installation.md#3-pods-stuck-in-pending) · [02 §4 — Storage class](./02-Deployment_and_Installation.md#pvcs-pending-the-storage-class-does-not-exist-in-the-cluster) |
| `auth` works, but `builder` / `cwcback` / `logs` / `metrics` report `Access denied` | [02 §6 — MariaDB grants](./02-Deployment_and_Installation.md#auth-works-but-builder--cwcback--logs--metrics-report-access-denied) |
| No external IP, or the UI does not load | [02 §7 — Cannot reach the UI](./02-Deployment_and_Installation.md#7-no-external-ip--cannot-reach-the-ui) |
| Helm reports `failed post-install: timed out waiting for the condition` | [02 §9 — Post-install timeout](./02-Deployment_and_Installation.md#9-helm-reports-failed-post-install-timed-out-waiting-for-the-condition) |
| Cannot log in to the platform UI | [03 §7 — Platform login](./03-Authentication_and_Credentials.md#7-cannot-log-in-to-the-platform-ui) |

### SAP Links

| Symptom | Go to |
|---|---|
| The SAP Link appears in the UI, but its Producer/Consumer pods are not running | [06 §8 — SAP Link without running pods](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) |
| Creating a SAP Link returns `Workspace Max limit (N) has been reached` | [06 §9 — Workspace limit](./06-No_Data_Reaching_Destination.md#9-creating-a-sap-link-fails-workspace-limit-reached) |
| The SAP Link's Kafka Connect is not reachable | [04 §10 — Kafka Connect per SAP Link](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |

### SAP → Producer

| Symptom | Go to |
|---|---|
| Need to know what an HTTP code returned to SAP means | [03 — HTTP status table](./03-Authentication_and_Credentials.md#reading-the-http-status-code) |
| SAP receives **401** | [03 §2 — Diagnosing a `401`](./03-Authentication_and_Credentials.md#2-diagnosing-a-401-from-sap) |
| SAP receives **400** or **500** | [03 §3 — Headers and payloads](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| SAP receives **404** | [03 — HTTP status table](./03-Authentication_and_Credentials.md#reading-the-http-status-code) |
| No request from SAP appears in the Producer logs | [06 §3 — SAP is not reaching the Producer](./06-No_Data_Reaching_Destination.md#3-sap-is-not-reaching-the-producer) |
| SAP waits a long time for the response | [04 §9 — Producer response time](./04-Kafka_Connectivity.md#9-sap-waits-a-long-time-for-the-producers-response) |

### Kafka

| Symptom | Go to |
|---|---|
| `TimeoutException`, `Broker may not be available`, `No resolvable bootstrap urls` | [04 §2 — Cannot reach the brokers](./04-Kafka_Connectivity.md#2-cannot-reach-the-brokers) |
| `SaslAuthenticationException`, `SSLHandshakeException` | [04 §3 — Authentication rejected](./04-Kafka_Connectivity.md#3-authentication-rejected-by-the-brokers) |
| No topic is ever created | [04 §4 — Topics are never created](./04-Kafka_Connectivity.md#4-topics-are-never-created) |
| `RecordTooLargeException` | [04 §5 — Message size limits](./04-Kafka_Connectivity.md#5-message-size-limits) |
| `TopicAuthorizationException`, `GroupAuthorizationException` | [04 §6 — ACLs](./04-Kafka_Connectivity.md#6-acls-and-authorization) |
| The Consumer connects but consumes nothing | [04 §7](./04-Kafka_Connectivity.md#7-the-consumer-connects-but-consumes-nothing) · [06 §2 — Topic naming](./06-No_Data_Reaching_Destination.md#2-the-consumer-is-subscribed-to-a-topic-nobody-writes-to) |

### Data conversion (Consumer)

| Symptom | Go to |
|---|---|
| Records land in `*_PROCESSING_ERRORS` | [05 §2 — Read the error topic](./05-Avro_Serialization_and_Schema_Registry.md#2-start-here-read-the-error-topic) |
| `NullPointerException` (a field without a value) | [05 §3 — Field without a value](./05-Avro_Serialization_and_Schema_Registry.md#3-a-field-without-a-value-in-a-row) |
| `NoSuchElementException` | [05 §4 — Missing table metadata](./05-Avro_Serialization_and_Schema_Registry.md#4-nosuchelementexception--missing-table-metadata) |
| `ClassCastException`, `NumberFormatException` | [05 §5 — Type mismatches](./05-Avro_Serialization_and_Schema_Registry.md#5-classcastexception-and-numberformatexception--type-mismatches) |
| `Error processing OneConnect.typeFinder` | [05 §6 — Field not found in the metadata](./05-Avro_Serialization_and_Schema_Registry.md#6-error-processing-oneconnecttypefinder--field-not-found-in-the-metadata) |
| Schema Registry returns **401** or **409** | [05 §7](./05-Avro_Serialization_and_Schema_Registry.md#7-schema-registry-returns-401) · [05 §8](./05-Avro_Serialization_and_Schema_Registry.md#8-schema-evolution-and-409-conflict) |
| `InvalidTopicException` on `/BIC/` tables | [05 §12 — Table names with slashes](./05-Avro_Serialization_and_Schema_Registry.md#12-invalidtopicexception--table-names-with-slashes-bic-tables) |

### Destination

| Symptom | Go to |
|---|---|
| SAP reports success, but the destination stays empty | [06 §1 — Localize the break first](./06-No_Data_Reaching_Destination.md#1-localize-the-break-first) |
| Topics grow, but Kafka Connect does not deliver | [06 §5 — Kafka Connect](./06-No_Data_Reaching_Destination.md#5-kafka-connect-is-not-delivering-to-the-destination) |
| Only part of the data arrives | [06 §6 — Partial data](./06-No_Data_Reaching_Destination.md#6-data-arrives-but-only-partially) |
| Need to reprocess records | [06 §7 — Error topics](./06-No_Data_Reaching_Destination.md#7-reviewing-and-reprocessing-records-from-the-error-topics) |

### Logs

| Symptom | Go to |
|---|---|
| The Logs panel is empty | [01 §3 — Logs panel empty](./01-Diagnostic_Toolkit.md#3-the-logs-panel-is-empty) |
| SAP entries do not appear in the Logs panel | [01 §3 — SAP entries](./01-Diagnostic_Toolkit.md#sap-entries-are-missing) |
| Need more detail in the logs | [01 §4 — Raising the log level](./01-Diagnostic_Toolkit.md#4-raising-the-log-level-to-see-more-detail) |

> 💡 Start with [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md). It covers where errors surface, which log levels reach each place, and how to raise the verbosity when you need more detail.

---

## 📚 Guides in This Folder

### [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md)
Where to look: the Logs panel, `kubectl logs`, the error topics, the log-level rules that decide what reaches each place, and a five-minute health check. **Read this first.**

### [02 — Deployment & Installation](./02-Deployment_and_Installation.md)
Helm install and upgrade, the Docker Hub image pull secret, storage classes per cloud, MariaDB startup and grants, load balancers, and the post-install hook.

### [03 — Authentication & Credentials](./03-Authentication_and_Credentials.md)
The HTTP status codes returned to SAP, SAP → Producer Basic auth, Kafka SASL, Schema Registry basic auth, the internal soft security token, and platform UI login.

### [04 — Kafka Connectivity](./04-Kafka_Connectivity.md)
Bootstrap and DNS, SASL/SSL handshakes, topic creation, message size limits, ACLs, consumer groups and offsets, Producer response times, and the Kafka Connect instance of each SAP Link.

### [05 — Avro Serialization & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md)
How the Consumer converts SAP data, how to read the error topic, and every conversion message it produces: empty fields, metadata, types, nested tables, `/BIC/` tables, and schema compatibility.

### [06 — No Data Reaching Destination](./06-No_Data_Reaching_Destination.md)
The end-to-end checklist when the destination stays empty: localizing the break, topic naming, Kafka Connect, SAP Links without running pods, the workspace limit, and reprocessing from the error topics.

### [07 — Error Reference](./07-Error_Reference.md)
Every message, HTTP code, and state covered by these guides, grouped by component, with its meaning and a link to the case that resolves it.

---

## 🧭 Component Reference

Quick orientation for the names used throughout these guides.

| Name in docs | What it is | Port |
|---|---|---|
| **Producer** | Receives HTTP payloads from SAP and writes raw JSON to Kafka. One per SAP Link (`<workspace-id>-producer`) | `8080` |
| **Consumer** | Reads raw JSON, converts to Avro, writes one topic per SAP table. One per SAP Link (`<workspace-id>-consumer`) | — |
| **Kafka Connect** | Delivers the per-table topics to the destination. One per SAP Link (`connect-<workspace-id>`, namespace `kafka`) | `8083` |
| **Builder** | Creates the Producer, Consumer, and Kafka Connect workloads when a SAP Link is created | `7071` |
| **API Gateway** | Routes platform traffic and validates the internal soft security token | `9000` |
| **Auth service** | Platform login and users | `9001` |
| **Logs service** | Receives log entries from the Producer, Consumer, and SAP OTel Bridge and serves the Logs panel | `7072` |
| **Metrics service** | Receives Producer metrics and serves the dashboard | `7070` |
| **SAP OTel Bridge** | Receives SAP-side events (Traceability) and sends them to the Logs panel | — |
| **Frontend** | The web UI | `5050` |

The Producer, Consumer, and Kafka Connect are **created per SAP Link** by the Builder — they are not part of the base Helm install. See [06 §8](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) if they do not appear.

---

## 🔗 Related Documentation

- [Working with Logs](../Smart_Gateway/Using_the_Smart_Gateway/07-Working_with_Logs.md) — how to *use* the Logs panel (filters, export, alerts). This folder covers what to do when the logs show a problem.
- [Generating a SAP OTel Bridge Endpoint](../Smart_Gateway/Using_the_Smart_Gateway/08-Generating_SAP_OTel_Bridge_Endpoint.md) — surfacing SAP-side errors in the same panel.
- [Endpoint Customization (SM59)](../SAP_Data_Modeler/Data_Modeler_Manuals/15-Endpoint_Customization_SM59.md) — configuring the RFC destination and choosing the synchronous endpoint.
- [Reprocessing Records](../SAP_Data_Modeler/Data_Modeler_Manuals/12-Reprocessing_Records.md) — re-sending data from the SAP side.
- [SmartGateway BTP/Kyma Deployment](../Smart_Gateway/BTP/SmartGateway_BTP_Kyma_Deployment.md#troubleshooting) — Kyma/Cloud Connector specific troubleshooting.
- [OneConnect FAQ](../Technical_Information/06-OneConnect_FAQ.md) — product capabilities and limits.

---

## 📩 Escalating to Support

If a guide does not resolve the issue, contact **contact@onibex.com** to open a support request. Our team will guide you from there.
