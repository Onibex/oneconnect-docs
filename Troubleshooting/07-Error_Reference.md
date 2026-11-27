# Error Reference

A lookup table of the messages, HTTP codes, and states that OneConnect components produce. Find the message you see, read what it means, and follow the link to the case that resolves it.

Messages are grouped by the component that produces them, in the order data travels:

```
SAP  ──▶  Producer  ──▶  Kafka  ──▶  Consumer  ──▶  Kafka Connect  ──▶  Destination
```

> 💡 Use your browser's search (**Ctrl+F** / **Cmd+F**) with a distinctive part of the message — for example the exception name or the HTTP code. Values shown as `<...>` vary per installation.

| Component | Entries |
|---|---|
| [SAP](#sap) | 5 |
| [Producer](#producer) | 21 |
| [Kafka](#kafka) | 9 |
| [Consumer](#consumer) | 20 |
| [Schema Registry](#schema-registry) | 4 |
| [Kafka Connect](#kafka-connect) | 1 |
| [Builder](#builder) | 14 |
| [API Gateway](#api-gateway) | 2 |
| [Logs](#logs) | 4 |
| [SAP OTel Bridge](#sap-otel-bridge) | 2 |
| [Helm](#helm) | 9 |
| [Kubernetes](#kubernetes) | 11 |
| [MariaDB](#mariadb) | 5 |

---

## SAP

Responses and messages seen on the SAP side when calling the Smart Gateway.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| SM59 Connection Test returns `200` | The Producer endpoint is reachable on this path; credentials are validated later, on `POST` | Nothing to fix | [03 §1](./03-Authentication_and_Credentials.md#reading-the-http-status-code) |
| SM59 Connection Test returns `404` | The path prefix or workspace ID does not match the SAP Link | Use the URL shown on the SAP Link screen as-is | [03 §1](./03-Authentication_and_Credentials.md#reading-the-http-status-code) |
| Connection Test returns a code other than `200`/`404`, or no HTTP response | The response comes from a network component between SAP and the Smart Gateway (ICM, proxy, ingress, load balancer) | Review host, port, SSL settings and network path | [03 §1](./03-Authentication_and_Credentials.md#reading-the-http-status-code) |
| HTTP/ICM timeout in SM59 while the Producer is running | SAP stopped waiting before the Producer reported Kafka's answer | Restore Kafka connectivity; set SM59 Special Options → Timeout (e.g. 180 s) | [04 §9](./04-Kafka_Connectivity.md#9-sap-waits-a-long-time-for-the-producers-response) |
| Connection error, timeout, or `404` when SAP calls the SAP Link URL | SAP requests are not reaching the Producer | Test the Producer via port-forward, then check SM59 / ingress / load balancer | [06 §3](./06-No_Data_Reaching_Destination.md#3-sap-is-not-reaching-the-producer) |

---

## Producer

Messages from the Producer, which receives data from SAP and writes it to Kafka.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `Starting to process` (search key, `DEBUG`) | A request from SAP reached the Producer | Optional detail; shown when the log level is raised with `LOGGING_LEVEL_ROOT=DEBUG` | [01 §4](./01-Diagnostic_Toolkit.md#4-raising-the-log-level-to-see-more-detail) |
| `-->ExecutionException =<message>` | Synchronous endpoint: Kafka did not confirm the record | Read the message; check Kafka connectivity | [01 §2](./01-Diagnostic_Toolkit.md#producer-and-consumer-log-lines) |
| `-->MalformedJsonException =<message>` | Payload is not valid JSON; stored in `<CLIENT>_<ENV>_INBOUND_ERRORS` | Read the payload in AKHQ and correct it on the SAP side | [01 §5](./01-Diagnostic_Toolkit.md#5-the-error-topics) |
| `ERROR sending logs: I/O error on POST request for "<url>"` (Consumer: `ERROR sending logs : ...`) | Logs service URL cannot be reached | Check `ONECONNECT_LOGS_URL_PRODUCER`/`_CONSUMER` and the `logs` pod | [01 §3](./01-Diagnostic_Toolkit.md#3-the-logs-panel-is-empty) |
| `ERROR sending logs: 404 ...` | URL path does not match a Logs endpoint | Use `.../logs/reporter/controller-logs/logs-producer` or `logs-consumer` | [01 §3](./01-Diagnostic_Toolkit.md#3-the-logs-panel-is-empty) |
| `401 Unauthorized` returned to SAP on `POST` | User or password does not match the SAP Link credentials | Compare SM59 Logon & Security with `SAPPROPERTIES_USER` / `SAPPROPERTIES_PASSWORD`; avoid `:` in the password | [03 §2](./03-Authentication_and_Credentials.md#2-diagnosing-a-401-from-sap) |
| `-->Authentication error. Incorrect username and/or password.` | Synchronous request rejected for credentials (`401`) | Same as above | [03 §2](./03-Authentication_and_Credentials.md#2-diagnosing-a-401-from-sap) |
| `MissingRequestHeaderException` with `Required request header 'Authorization'` (`WARN`, `400`) | SAP sent no `Authorization` header | Set Basic authentication with user and password in SM59 Logon & Security | [03 §3](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| `-->MalformedJsonException =Json non valid` (`400` on the synchronous endpoint) | Payload is not valid JSON; it is written to `<CLIENT>_<ENV>_INBOUND_ERRORS` | Review the payload sent by SAP | [03 §3](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| `-->ExecutionException =<exception>: <message>` (`400` on the synchronous endpoint) | Credentials accepted; the Kafka send did not complete | Route by the Kafka exception in 04 | [03 §3](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| `500` returned to SAP on `POST` with an `Authorization` header that is not HTTP Basic `user:password` | The header is not HTTP Basic, or the password is empty | Configure Basic auth in SM59 with both user and password | [03 §3](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| `JsonSyntaxException` in the Producer log (`500`) | The payload is valid JSON but not an object | Send one JSON object per request | [03 §3](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| `-->Error Sending Record to topic=<topic>, value=..., errorType=<class>, cause=<message>` (`500` on the synchronous endpoint) | The Kafka send could not start | Check Kafka connectivity; once resolved, restart the Producer | [03 §3](./03-Authentication_and_Credentials.md#3-400-and-500-malformed-headers-and-payloads) |
| `Disconnected from metrics` (WARN, every push) | Metrics push not accepted (for example soft token rejected on `/metrics/reporter/**`) | Check `ONECONNECT_METRICS_URL` and the soft token | [03 §6](./03-Authentication_and_Credentials.md#6-internal-soft-security-token) |
| `org.apache.kafka.common.errors.TimeoutException: Topic <topic> not present in metadata after 60000 ms.` | The Producer could not reach the brokers within 60 s | Check bootstrap address, network, and Kafka status | [04 §2](./04-Kafka_Connectivity.md#2-cannot-reach-the-brokers) |
| `org.apache.kafka.common.config.ConfigException: No resolvable bootstrap urls given in bootstrap.servers` (pod in `CrashLoopBackOff`) | The bootstrap host name cannot be resolved at startup | Use a resolvable `host:port` (fully qualified for in-cluster Kafka) | [04 §2](./04-Kafka_Connectivity.md#2-cannot-reach-the-brokers) |
| `-->ExecutionException =org.apache.kafka.common.errors.SaslAuthenticationException: ...` (SAP gets `400`) | Synchronous send refused by the brokers' authentication | Correct the Kafka credentials | [04 §3](./04-Kafka_Connectivity.md#3-authentication-rejected-by-the-brokers) |
| `-->ExecutionException =org.apache.kafka.common.errors.TimeoutException: Expiring <N> record(s) for <topic>-<partition>:120000 ms has passed since batch creation` | Kafka did not confirm the write within the delivery timeout | Restore broker connectivity; re-send from SAP | [04 §9](./04-Kafka_Connectivity.md#9-sap-waits-a-long-time-for-the-producers-response) |
| `404` on `/api/v1/saplistener/<workspace-id>/synchronous` | The path or workspace ID differs from this Producer's | Use the SAP Link URL shown by the platform as-is | [06 §3](./06-No_Data_Reaching_Destination.md#3-sap-is-not-reaching-the-producer) |
| SAP receives `201`, but the raw topic does not grow | The asynchronous endpoint confirms receipt | For delivery confirmation use the synchronous endpoint (the URL shown by the platform); read the Producer log | [06 §4](./06-No_Data_Reaching_Destination.md#4-the-producer-is-not-writing-to-kafka) |
| `-->MalformedJsonException =...` | The request body was not valid JSON; it is stored in `<CLIENT>_<ENV>_INBOUND_ERRORS` | Correct the payload in SAP and re-send | [06 §7](./06-No_Data_Reaching_Destination.md#7-reviewing-and-reprocessing-records-from-the-error-topics) |

---

## Kafka

Messages from the Kafka clients when connecting to the brokers.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `Connection to node -1 (<host>/<ip>:<port>) could not be established. Broker may not be available.` | The client cannot open a connection to the broker | Check bootstrap address, firewall, Kafka pods | [04 §2](./04-Kafka_Connectivity.md#2-cannot-reach-the-brokers) |
| `SaslAuthenticationException: Authentication failed: Invalid username or password` | Brokers declined the API key/secret | Issue/enter a valid key for this cluster; restart workloads | [04 §3](./04-Kafka_Connectivity.md#3-authentication-rejected-by-the-brokers) |
| `javax.net.ssl.SSLHandshakeException: PKIX path building failed` | Broker certificate not trusted | Trust the CA or use a publicly trusted certificate | [04 §3](./04-Kafka_Connectivity.md#tls-specific-errors) |
| `TopicAuthorizationException` / `InvalidReplicationFactorException` / `PolicyViolationException` during topic creation | The topic could not be created with the requested permissions or settings | Grant ACLs; set Replicas ≤ brokers; align Topic Settings with the cluster policy | [04 §4](./04-Kafka_Connectivity.md#4-topics-are-never-created) |
| `RecordTooLargeException: The message is <N> bytes when serialized which is larger than 8388608, which is the value of the max.request.size configuration.` | The payload exceeds the 8 MB client limit | Lower the entity's Number of Records, or raise all size limits together | [04 §5](./04-Kafka_Connectivity.md#5-message-size-limits) |
| `GroupAuthorizationException: Not authorized to access group: <CLIENT>_<ENV>` | The Consumer's key lacks READ on its group | Grant READ on group `<CLIENT>_<ENV>` | [04 §6](./04-Kafka_Connectivity.md#6-acls-and-authorization) |
| `consumer poll timeout has expired. This means the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms ...` | A batch took longer than 10 minutes; the group rebalances and redelivers it | Reduce Number of Records, check Schema Registry/brokers, raise Limit CPU; use upsert | [04 §8](./04-Kafka_Connectivity.md#8-rebalances-and-poll-timeouts) |
| `-->Error Sending Record to topic=<prefix>/BIC/<table>, ... errorType=InvalidTopicException` | Table or entity name contains `/`, which Kafka topic names do not accept | Publish under a name with letters, digits, `.`, `_`, `-` | [05 §12](./05-Avro_Serialization_and_Schema_Registry.md#12-invalidtopicexception--table-names-with-slashes-bic-tables) |
| `Error while fetching metadata with correlation id <N> : {<topic>=UNKNOWN_TOPIC_OR_PARTITION}` (Consumer log) | The Consumer is subscribed to a topic that does not exist | Align `postfix` / `regexsource`, `client`, `environment` | [06 §2](./06-No_Data_Reaching_Destination.md#2-the-consumer-is-subscribed-to-a-topic-nobody-writes-to) |

---

## Consumer

Messages from the Consumer, which converts SAP data to Avro and writes one topic per table.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `Unhandled exception in consumerFactory at partition:... offset:..., errorType=..., cause=..., stackTrace=...` | The poll cycle stopped on an error; the record in progress goes to `*_PROCESSING_ERRORS` | Route by `errorType` | [01 §2](./01-Diagnostic_Toolkit.md#producer-and-consumer-log-lines) |
| `-->Critical Error: Failed to send error message to topic=..., value=..., errorType=..., cause=..., stackTrace=...` | The entry could not be written to the error topic; `value=` holds it | Keep the `value=` content for replay; check Kafka connectivity | [01 §2](./01-Diagnostic_Toolkit.md#producer-and-consumer-log-lines) |
| `Failed to send message` / `Error: <msg>` / `Data: <record>` (entry in `*_PROCESSING_ERRORS`) | Format of each processing-error entry | Fix the cause in `Error:`, replay `Data:` | [01 §5](./01-Diagnostic_Toolkit.md#5-the-error-topics) |
| `org.apache.kafka.common.config.ConfigException: No resolvable bootstrap urls given in bootstrap.servers` (pod in `CrashLoopBackOff`) | The bootstrap host name cannot be resolved at startup | Use a resolvable `host:port` | [04 §2](./04-Kafka_Connectivity.md#2-cannot-reach-the-brokers) |
| `Unhandled exception in consumerFactory at partition:0 offset:0, errorType=<X>, cause=...` (repeating) | A poll cycle was refused (ACLs, credentials) | Resolve the condition named in `errorType` | [04 §6](./04-Kafka_Connectivity.md#6-acls-and-authorization) |
| Consumer group `<CLIENT>_<ENV>` missing, without members, or with growing `LAG` in AKHQ | Consumer not connected, not running, or slower than the arrival rate | Follow the table in 04 §7 | [04 §7](./04-Kafka_Connectivity.md#7-the-consumer-connects-but-consumes-nothing) |
| `Error processing record, will send to error topic, errorType=<X>, cause=<reason>` | A SAP message stopped at the processing stage; the full message is in `PROCESSING_ERRORS` (`Data:`) | Match `cause=` against the 05 cases; fix and resend from SAP | [05 §2](./05-Avro_Serialization_and_Schema_Registry.md#2-start-here-read-the-error-topic) |
| `-->Error Sending Record to topic=<t>, key=<k>, value=<v>, errorType=<X>, cause=<reason>` | One row stopped at the sending stage; the row is in `PROCESSING_ERRORS` (`Data:`) | Fix the cause (registry, topic, broker), then resend from SAP | [05 §2](./05-Avro_Serialization_and_Schema_Registry.md#2-start-here-read-the-error-topic) |
| `PROCESSING_ERRORS` entry with `Error: null` and an empty `Data:` | Marks a Consumer restart | No action needed | [05 §2](./05-Avro_Serialization_and_Schema_Registry.md#2-start-here-read-the-error-topic) |
| `errorType=NullPointerException, cause=Cannot invoke "Object.getClass()" because "object" is null` | A metadata field has no value in a row (absent, `null`, or different letter case) | Fill every field in every row; use lowercase names in metadata and data; resend | [05 §3](./05-Avro_Serialization_and_Schema_Registry.md#3-a-field-without-a-value-in-a-row) |
| `errorType=NoSuchElementException, cause=No value present` | No `metadata[].table` matches `body.table`, or metadata is empty | Send the metadata block for the table named in `body.table` | [05 §4](./05-Avro_Serialization_and_Schema_Registry.md#4-nosuchelementexception--missing-table-metadata) |
| `errorType=NumberFormatException, cause=For input string: "<value>"` / `cause=empty String` | A value does not parse as the field's numeric SAP type (`N`, `P`, `I`) | Send plain numbers (dot decimal, leading minus, `"0"` not `""`) or retype the field | [05 §5](./05-Avro_Serialization_and_Schema_Registry.md#5-classcastexception-and-numberformatexception--type-mismatches) |
| `Cannot convert String to byte: "<value>"` / `Cannot convert empty String to byte.` | Value on a type `X` (RAW) field | Leave the RAW field out, or expose it as a character field | [05 §5](./05-Avro_Serialization_and_Schema_Registry.md#raw-fields-type-x) |
| `errorType=ClassCastException, cause=Cannot convert java.lang.Boolean to Long.` | A JSON boolean on a numeric field | Send a numeric value | [05 §5](./05-Avro_Serialization_and_Schema_Registry.md#5-classcastexception-and-numberformatexception--type-mismatches) |
| `Error processing OneConnect.typeFinder dataName: <field>. Invalid or name not found in BodyMetadata` | The row carries a field that the table's metadata does not define (including names with `/`) | Align data and metadata; use names with letters, digits, underscores | [05 §6](./05-Avro_Serialization_and_Schema_Registry.md#6-error-processing-oneconnecttypefinder--field-not-found-in-the-metadata) |
| Character field arrives as `123.0`; leading zeros missing | Value sent as JSON number, or field type mapping | Send values as JSON strings; choose the SAP type to match the data | [05 §9](./05-Avro_Serialization_and_Schema_Registry.md#9-records-serialize-but-the-destination-has-wrong-values) |
| `errorType=NullPointerException` whose `cause=` names `delete`, `messageType`, `properties`, `body`, `data`, `metadata`, or `keyflag` | A required message element is missing | Send the complete message (`delete` and `keyflag` as `""` when not set) | [05 §10](./05-Avro_Serialization_and_Schema_Registry.md#10-missing-message-fields-delete-messagetype-body-keyflag) |
| `errorType=AvroRuntimeException, cause=Not a record: {"type":"array",...}` / `Not a record: "string"` | Nested table without matching metadata block (or name case differs), or an object in a simple field | Include the child metadata with the same name; align the entity | [05 §11](./05-Avro_Serialization_and_Schema_Registry.md#11-nested-tables-avroruntimeexception-not-a-record) |
| `errorType=SchemaParseException, cause=Illegal initial character: <name>` / `Illegal character in: <name>` / `Duplicate field <name> in record ...` | Field name not valid for Avro, or repeated in the metadata | Use Avro-valid, unique field names | [05 §13](./05-Avro_Serialization_and_Schema_Registry.md#13-field-names-rejected-by-avro-schemaparseexception-and-duplicate-field) |
| `-->Failed to create SinkConnector for table=<table> Error=...` | The Consumer's optional automatic connector creation (`ONPREMISE`) did not complete | Use platform-created connectors and their status | [06 §5](./06-No_Data_Reaching_Destination.md#5-kafka-connect-is-not-delivering-to-the-destination) |

---

## Schema Registry

Messages related to schema registration and compatibility.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `Unauthorized; error code: 401` in the Consumer log | Registry credentials not accepted | `KAFKAPROPERTIES_SCHEMAUSER=USER_INFO`, `KAFKAPROPERTIES_SCHEMAPASSWORD=KEY:SECRET`, configured with `CLOUD` | [03 §5](./03-Authentication_and_Credentials.md#5-schema-registry-basic-auth) |
| `errorType=SerializationException, cause=Error registering Avro schema...` with `Caused by: ...RestClientException: Unauthorized; error code: 401` | Registry rejected the Consumer's credentials | Set `schemauser=USER_INFO`, `schemapassword=<KEY>:<SECRET>` (`CLOUD` mode) | [05 §7](./05-Avro_Serialization_and_Schema_Registry.md#7-schema-registry-returns-401) |
| `cause=Error serializing Avro message` with `Caused by: java.net.ConnectException` / `java.net.UnknownHostException` | Registry not reachable at `schemaregistryurl` | Check the registry pod and use the fully qualified URL | [05 §7](./05-Avro_Serialization_and_Schema_Registry.md#other-schema-registry-connection-errors) |
| `Schema being registered is incompatible with an earlier schema for subject "<topic>-value"; error code: 409` (in the `Caused by:` line) | Entity change not compatible with the registered schema | Revert, use a new topic, or relax compatibility for the subject | [05 §8](./05-Avro_Serialization_and_Schema_Registry.md#8-schema-evolution-and-409-conflict) |

---

## Kafka Connect

Messages related to the Kafka Connect instance of each SAP Link.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| Connector or tasks `FAILED` in AKHQ **Connect** → `connect-<workspace-id>` | The sink cannot write to the destination | Read `trace`; fix credentials, table, schema, or topic list | [06 §5](./06-No_Data_Reaching_Destination.md#5-kafka-connect-is-not-delivering-to-the-destination) |

---

## Builder

Messages from the Builder, which creates the workloads of each SAP Link.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `ERROR sending request:` / `forbidden` in Builder logs; no Producer pod after creating a SAP Link | Builder RBAC not in place | Keep `rbac.create: true` and `services.builder.serviceAccount.create: true`, then upgrade | [02 §5](./02-Deployment_and_Installation.md#5-a-sap-link-was-created-but-no-producer-pod-appears) |
| `Failed to connect to broker [<bootstrap>]: <message>` | The SAP Link connection test could not reach Kafka | Read the message; check address, network, credentials | [04 §2](./04-Kafka_Connectivity.md#2-cannot-reach-the-brokers) |
| `Kafka Connect Deployment not found for workspace: <workspace-id>. Please create it first.` | No Kafka Connect configured for this SAP Link | Run Configure Kafka Connect, then create the connector | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| `Failed to create connector: I/O error on POST request for "http://connect-<workspace-id>.kafka.svc.cluster.local:8083/connectors": ...` | The SAP Link's Kafka Connect did not answer | Check pod `connect-<workspace-id>` in `kafka` and its log | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| `Kafka Connect error: <response body>` | Kafka Connect answered with an error | Wait until the worker is Ready; read the body and worker log | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| `Unexpected error or timeout creating connector (attempt <n>/5). Retrying in 5000ms...` | A connector request is in progress | Wait; if it ends in an error, check Kafka Connect | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| `Failed to get connector status: ...` / `Failed to restart connector due to communication error: ...` | Kafka Connect REST API not reachable | Check pod and Service `connect-<workspace-id>` | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| `Failed to create/update resource 'connect-<workspace-id>': 403 FORBIDDEN` | The Builder lacks permission in the `kafka` namespace | Enable Role `smartgateway-kafka-connect-manager` (`rbac.create: true`) | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| `Kafka secret '<name>' not found in namespace 'oneconnect'` | The Kafka secret to copy for Kafka Connect is not present | Select a secret that exists for the SAP Link owner | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| `Failed to register Connect in AKHQ config for workspace <workspace-id>` | Kafka Connect is deployed but not added to AKHQ | Ensure Role `akhq-config-updater` and ConfigMap `akhq-config`; configure again | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| `LoadBalancer external address not yet assigned for workspace: <workspace-id>` | The SAP Link's load balancer has no external address yet | Wait for `EXTERNAL-IP` on `producer-<workspace-id>-loadbalancer`, then retry | [06 §3](./06-No_Data_Reaching_Destination.md#3-sap-is-not-reaching-the-producer) |
| `ERROR sending request:` followed by `HttpClientErrorException$Forbidden: 403 Forbidden ...` | Kubernetes declined creating the SAP Link's Deployments/Service (permissions) | Enable RBAC (`builder-role`), then remove the SAP Link and create it again | [06 §8](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) |
| `ERROR sending request:` followed by `HttpClientErrorException$UnprocessableEntity: 422 ...` | Kubernetes declined the Deployment (e.g. Advanced Settings resource values) | Use valid quantities with requests ≤ limits; remove the SAP Link and create it again | [06 §8](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) |
| `Workspace Max limit (N) has been reached. It is not possible to create more.` | The number of SAP Links reached `KUBERNETES_MAX_WORKSPACES` | Remove SAP Links you no longer use, or raise `KUBERNETES_MAX_WORKSPACES` | [06 §9](./06-No_Data_Reaching_Destination.md#9-creating-a-sap-link-fails-workspace-limit-reached) |

---

## API Gateway

Messages from the API Gateway, which routes platform and reporting traffic.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| Login requests fail in the browser with `net::ERR_NAME_NOT_RESOLVED` or a connection error | `NEXT_PUBLIC_API_BASE` is not a browser-reachable gateway address | Confirm the `internal-apigateway` address and the `frontend-apigateway-sync` job, or set `networking.loadBalancer.apigateway.externalIP` | [03 §7](./03-Authentication_and_Credentials.md#7-cannot-log-in-to-the-platform-ui) |
| `Failure in token verification` in the gateway log | The gateway cannot validate the JWT with Auth at `AUTH_URL` | Confirm Auth is running; set `AUTH_URL` for non-`oneconnect` namespaces; log in again after rotating `JWT_SECRET` | [03 §7](./03-Authentication_and_Credentials.md#7-cannot-log-in-to-the-platform-ui) |

---

## Logs

Messages related to the delivery of log entries to the Logs panel.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| Logs panel shows no `PRODUCER`/`CONSUMER` entries | Batch window not elapsed, nothing at `WARN`+, filters, or delivery to the Logs service | Follow the checklist | [01 §3](./01-Diagnostic_Toolkit.md#3-the-logs-panel-is-empty) |
| SAP `INFO`/`DEBUG`/`TRACE` entries not shown | Level filter set to `WARN`/`ERROR`/`FATAL` | Set Level to `ALL` | [01 §1](./01-Diagnostic_Toolkit.md#which-entries-reach-the-panel) |
| `ERROR sending logs: 500 ...` | Logs service could not store the batch | Check `kubectl logs -l app=logs` for database errors | [01 §3](./01-Diagnostic_Toolkit.md#3-the-logs-panel-is-empty) |
| `ERROR sending logs: 401 Unauthorized ...` in Producer/Consumer stdout | Log shipping goes through the API Gateway and the soft token differs from `AUTH_SOFTAUTHTOKEN` | Make `ONECONNECT_SOFTSECURITYTOKEN` equal to the gateway's `AUTH_SOFTAUTHTOKEN` and restart | [03 §6](./03-Authentication_and_Credentials.md#6-internal-soft-security-token) |

---

## SAP OTel Bridge

Messages from the SAP OTel Bridge (Traceability).

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `Failed to export SAP event <event-id> to oneconnect-logs: <reason>` | Bridge could not deliver the SAP event to the Logs service | Check the `logs` pod and `TELEMETRY_LOGS_ENDPOINT` (Helm `SAPOTELBRIDGE_LOGSURL`) | [01 §3](./01-Diagnostic_Toolkit.md#sap-entries-are-missing) |
| `telemetry.logs.enabled=true but no endpoint configured; skipping SAP log export` | No Logs endpoint configured on the bridge | Set Helm `SAPOTELBRIDGE_LOGSURL` | [01 §3](./01-Diagnostic_Toolkit.md#sap-entries-are-missing) |

---

## Helm

Messages shown by Helm during install or upgrade.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `no matches for kind "Kafka" in version "kafka.strimzi.io/v1" -- ensure CRDs are installed first` | `kafka-platform` installed before the Strimzi CRDs exist | Install the Strimzi operator first, then retry | [02 §1](./02-Deployment_and_Installation.md#deployment-order-was-not-respected) |
| `mysql.rootPassword is required` / `mysql.password is required` / `appSecrets.jwtSecret is required` / `appSecrets.softToken is required` | The install or upgrade did not receive the chart secrets | Pass `--values secrets.local.yaml` (or set `mysql.existingSecret` / `appSecrets.existingSecret`) on every install and upgrade | [02 §1](./02-Deployment_and_Installation.md#missing-required-secrets) |
| `akhq.security.jwtSecret is required` / `akhq.security.users[].password (SHA-256) is required` | `kafka-platform` did not receive the AKHQ secrets | Pass `--values kafka-platform/secrets.local.yaml` with the AKHQ JWT secret and the SHA-256 of the admin password | [02 §1](./02-Deployment_and_Installation.md#missing-required-secrets) |
| `namespaces "kafka" not found` | `oneconnect` chart installed before the `kafka` namespace exists | Complete phases 1 and 2 (Strimzi, kafka-platform) first | [02 §1](./02-Deployment_and_Installation.md#deployment-order-was-not-respected) |
| UI not reachable from the internet after `helm upgrade` | `networking.loadBalancer.type=external` not passed on upgrade | Pass the same values file and flags on every upgrade; keep `networking.loadBalancer.type` | [02 §8](./02-Deployment_and_Installation.md#8-upgrades) |
| `Job.batch "oneconnect-mysql-config" is invalid: spec.template: ... field is immutable` | The database setup Job from the previous revision is still present | Re-run the database setup | [02 §6](./02-Deployment_and_Installation.md#re-run-the-database-setup) |
| `PersistentVolumeClaim "st-datasynchub" is invalid: spec: Forbidden: spec is immutable after creation` / `StatefulSet.apps "mysql" is invalid: spec: Forbidden` | Storage values differ from the original install | Use the original storage values | [02 §8](./02-Deployment_and_Installation.md#8-upgrades) |
| `Error: INSTALLATION FAILED: failed post-install: timed out waiting for the condition` | The post-install hook did not finish within the Helm timeout | `helm upgrade` with install flags and `--timeout 15m` | [02 §9](./02-Deployment_and_Installation.md#9-helm-reports-failed-post-install-timed-out-waiting-for-the-condition) |
| `ERROR: ApiGateway LoadBalancer endpoint was not assigned after 600s.` (frontend-apigateway-sync hook) | `internal-apigateway` got no address | Resolve the load balancer, then `helm upgrade` | [02 §9](./02-Deployment_and_Installation.md#9-helm-reports-failed-post-install-timed-out-waiting-for-the-condition) |

---

## Kubernetes

Pod states and Kubernetes events.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `ErrImagePull` / `ImagePullBackOff` with `unauthorized` or `pull access denied` | Docker Hub token missing, expired, or not passed on upgrade | Always pass `dockerHub.token`: `helm upgrade` with `--set dockerHub.token=...`, then restart pods | [02 §2](./02-Deployment_and_Installation.md#2-imagepullbackoff--errimagepull) |
| `toomanyrequests: You have reached your pull rate limit` | Docker Hub pull limit on the public images of supporting components | Retry later or use a registry mirror | [02 §2](./02-Deployment_and_Installation.md#2-imagepullbackoff--errimagepull) |
| `Insufficient cpu` / `Insufficient memory` | Node pool too small for the requests | Scale nodes or lower requests | [02 §3](./02-Deployment_and_Installation.md#3-pods-stuck-in-pending) |
| `had untolerated taint` (e.g. on `mysql-0`) | Nodes are tainted | Add `scheduling.tolerations`; keep at least one node pool without taints for the database and setup jobs | [02 §3](./02-Deployment_and_Installation.md#tolerations-scope) |
| `storageclass.storage.k8s.io "<name>" not found` / `pod has unbound immediate PersistentVolumeClaims` | Storage class from the values file (`gp3`, `managed-premium`, `pd-ssd`) does not exist in the cluster | Compare with `kubectl get storageclass`; create the class or override it with `--set` | [02 §4](./02-Deployment_and_Installation.md#pvcs-pending-the-storage-class-does-not-exist-in-the-cluster) |
| `waiting for a volume to be created, either by external provisioner` | Cloud CSI driver not installed | Install the EBS / Azure Disk / PD CSI driver | [02 §4](./02-Deployment_and_Installation.md#other-storage-symptoms) |
| `EXTERNAL-IP` `<pending>` on `internal-frontend` / `internal-apigateway` | No load balancer controller, quota, or subnet tags | Fix the cloud load balancer setup | [02 §7](./02-Deployment_and_Installation.md#7-no-external-ip--cannot-reach-the-ui) |
| `WeakKeyException` or `DecodingException` in the Auth log on sign-in | `JWT_SECRET` is not Base64 or decodes to fewer than 32 bytes | Set `appSecrets.jwtSecret` (for example the output of `openssl rand -hex 32`), `helm upgrade` and restart Auth | [03 §7](./03-Authentication_and_Credentials.md#7-cannot-log-in-to-the-platform-ui) |
| Kafka Connect pod `CreateContainerConfigError`: `couldn't find key KAFKAPROPERTIES_SCHEMAREGISTRYURL in Secret ...` | The SAP Link's Kafka secret has no Schema Registry settings | Use a secret with Schema Registry URL and credentials | [04 §10](./04-Kafka_Connectivity.md#10-the-sap-links-kafka-connect-is-not-reachable) |
| SAP Link listed in the UI, no `<workspace-id>-producer` / `-consumer` pods; events show `exceeded quota` | The namespace quota does not cover the SAP Link's pods | Lower Advanced Settings / replicas or raise the quota | [06 §8](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) |
| Consumer `CreateContainerConfigError`: `couldn't find key KAFKAPROPERTIES_SCHEMAREGISTRYURL in Secret oneconnect/<secret-name>` | The SAP Link's Kafka secret has no Schema Registry settings | Create the SAP Link with a secret that includes Schema Registry URL and credentials | [06 §8](./06-No_Data_Reaching_Destination.md#8-the-sap-link-appears-in-the-ui-but-its-pods-are-not-running) |

---

## MariaDB

Messages related to the platform database.

| Message or symptom | Meaning | What to do | Guide |
|---|---|---|---|
| `ERROR: MySQL user not ready after 300 seconds` (init container `wait-for-mysql`) | MariaDB not ready yet when auth/builder/cwcback/logs/metrics started | Wait; the services continue once MariaDB is ready. If persistent, check MariaDB pod and PVC | [02 §6](./02-Deployment_and_Installation.md#other-mariadb-symptoms) |
| `Access denied for user '<mysql.user>'@'%' to database '<database>'` while `auth` works | The database setup Job did not complete | Re-run the database setup | [02 §6](./02-Deployment_and_Installation.md#re-run-the-database-setup) |
| `ERROR: MySQL did not become ready in time` (mysql-config Job) | MariaDB was not reachable within the Job window, or a retained volume holds different passwords | Check `mysql-0`; on reinstall reuse the original passwords | [02 §6](./02-Deployment_and_Installation.md#other-mariadb-symptoms) |
| `mysql-0` `OOMKilled` / `Liveness probe failed` | Memory limit too low / slow first initialization | Raise `mysql.resources.limits.memory` / `mysql.livenessProbe.initialDelaySeconds` | [02 §6](./02-Deployment_and_Installation.md#other-mariadb-symptoms) |
| `mysql-0` `CrashLoopBackOff` with InnoDB or data-dictionary errors after moving from an older chart | The `data-mysql-0` volume was created by MySQL 8, which MariaDB cannot open | Use a new installation, or move the data with a dump and import | [02 §6](./02-Deployment_and_Installation.md#other-mariadb-symptoms) |

---

## Not Listed Here?

1. Find the stage where data stops with [06 — No Data Reaching Destination](./06-No_Data_Reaching_Destination.md#1-localize-the-break-first).
2. Raise the log level as described in [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md) and look for the first `ERROR` or `WARN` line.
3. Contact **contact@onibex.com** to open a support request — see [Escalating to Support](./README.md#-escalating-to-support).
