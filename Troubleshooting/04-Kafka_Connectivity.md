# Kafka Connectivity

Connectivity between the Producer/Consumer and the Kafka brokers: reaching them, authenticating, creating topics, and getting records accepted.

---

## 0. Before You Start: Turn On the Detail

Topic creation reports its outcome at `DEBUG`, and the Producer's asynchronous send path reports at `INFO`. Both are below the default root level, so enable `DEBUG` on the workload before diagnosing anything in this guide:

```bash
kubectl set env deployment/<producer> -n oneconnect LOGGING_LEVEL_ROOT=DEBUG
kubectl logs -n oneconnect <producer-pod> -f | grep -i topic
```

Revert when finished — see [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md#4-raising-the-log-level-to-see-more-detail).

---

## 1. Configuration Reference

Both components share the same connection properties.

| Property | Default | Notes |
|---|---|---|
| `kafkaproperties.bootstrapaddress` | *(required)* | `host:port`, comma-separated for multiple brokers |
| `kafkaproperties.deploymenttype` | `CLOUD` | `CLOUD` or `ONPREMISE` — **exact match, case-insensitive** |
| `kafkaproperties.securityprotocol` | `SASL_SSL` | Ignored in `CLOUD` mode |
| `kafkaproperties.saslmechanism` | `PLAIN` | — |
| `kafkaproperties.sasluser` / `saslpassword` | *(required)* | API key / secret |
| `kafkaproperties.clientdnslookup` | `use_all_dns_ips` | — |
| `kafkaproperties.clientid` | *(set per workload)* | Appears in broker logs and quotas |
| `kafkaproperties.topicpartitions` | `1` | **Cloud clusters usually need 3+** |
| `kafkaproperties.topicreplicas` | `3` | Must not exceed the broker count |
| `kafkaproperties.topicmessagesize` | `8388608` (8 MB) | Per-topic `max.message.bytes` |
| `kafkaproperties.maxrequestsize` | `8388608` (8 MB) | Producer-side `max.request.size` |
| `kafkaproperties.retentiontime` | `604800000` (7 days) | — |
| `kafkaproperties.cleanuppolicy` | `delete` | — |
| `kafkaproperties.compression` | `producer` | — |
| `kafkaproperties.groupid` | `group_0` | Consumer only |

### What `deploymenttype` controls

| Value | `security.protocol` | SASL configured? | Schema Registry auth? |
|---|---|---|---|
| `CLOUD` | Set to `SASL_SSL` | Yes | Yes |
| `ONPREMISE` | Your `securityprotocol` value | Yes — unless `securityprotocol` is `NO_SECURITY` | No |
| `ONPREMISE` + `NO_SECURITY` | Not set (plaintext) | No | No |
| Unrecognized value | Not set | No | No |

> 📌 The value must be exactly `CLOUD` or `ONPREMISE` (case-insensitive). Anything else leaves the client without security settings, which presents as a connection timeout rather than an authentication error. Worth confirming early:
>
> ```bash
> kubectl exec -n oneconnect <pod> -- env | grep -i deploymenttype
> ```

---

## 2. Cannot Reach the Brokers

### Symptoms

```
org.apache.kafka.common.errors.TimeoutException: Topic ... not present in metadata after 60000 ms
java.net.UnknownHostException: ...
org.apache.kafka.common.errors.DisconnectException
Connection to node -1 ... could not be established. Broker may not be available.
```

### Diagnosis

```bash
# What address is actually configured?
kubectl exec -n oneconnect <pod> -- env | grep -i bootstrap

# Is the name resolvable and the port open from inside the pod?
kubectl exec -n oneconnect <pod> -- getent hosts <broker-host>
kubectl exec -n oneconnect <pod> -- timeout 5 bash -c '</dev/tcp/<broker-host>/<port>' && echo OPEN || echo BLOCKED

# In-cluster Kafka: is it up?
kubectl get pods -n kafka
kubectl get svc  -n kafka
```

### Causes

| Cause | Check | Fix |
|---|---|---|
| Wrong bootstrap address | Compare against the SAP Link's Kafka configuration | Correct it and restart the workload |
| In-cluster service name incomplete | Must be fully qualified across namespaces | Use `kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092` |
| Kafka not yet running | `kubectl get pods -n kafka` | Complete the Kafka phase before OneConnect — see [02](./02-Deployment_and_Installation.md#deployment-order-was-not-respected) |
| Security group / NSG / firewall blocks the port | TCP test above returns `BLOCKED` | Open the broker port from the cluster's egress range |
| Confluent Cloud private networking | Cluster is not reachable from this VPC | Configure peering/PrivateLink, or use a public endpoint |
| Missing port | `host` without `:port` | Bootstrap must be `host:port` |

---

## 3. Authentication Rejected by the Brokers

### Symptoms

```
org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed: Invalid username or password
org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed during authentication due to invalid credentials
```

This indicates the network path is working and the brokers declined the credentials.

### Causes

| Cause | Fix |
|---|---|
| API key/secret wrong or revoked | Issue a new key pair in the Kafka cluster and update `sasluser`/`saslpassword` |
| Key belongs to a different cluster | Confluent Cloud keys are cluster-scoped — verify the key matches the bootstrap address |
| Whitespace or quotes in the secret | The credentials are interpolated into a JAAS string; stray quotes break it. Re-set cleanly |
| A single quote (`'`) in the password | The JAAS config is single-quoted, so a `'` terminates it early | Regenerate the key to avoid `'` |
| Workload not restarted after the change | `kubectl rollout restart deployment/<name> -n oneconnect` |
| `saslmechanism` mismatch | Confluent Cloud uses `PLAIN`; some on-premise clusters use `SCRAM-SHA-512` — match the broker |

```bash
kubectl exec -n oneconnect <pod> -- env | grep -iE 'sasluser|saslmechanism|securityprotocol'
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

The Producer creates its topic at startup and again on each request; the Consumer creates one per SAP table. Both report the outcome at `DEBUG` — see [§0](#0-before-you-start-turn-on-the-detail).

### Diagnosis

Check which topics exist in **AKHQ → Topics**. Then enable `DEBUG` to see the creation outcome:

```bash
kubectl set env deployment/<producer> -n oneconnect LOGGING_LEVEL_ROOT=DEBUG
kubectl logs -n oneconnect <producer-pod> -f | grep -i 'topic'
```

### Causes

| Exception at DEBUG | Cause | Fix |
|---|---|---|
| `TopicAuthorizationException` | The API key lacks `CREATE` on the cluster / `WRITE` on the topic | Grant ACLs — see [§6](#6-acls-and-authorization) |
| `InvalidReplicationFactorException` | `topicreplicas` exceeds the broker count | Set `topicreplicas` ≤ number of brokers (use `1` for a single-broker dev cluster) |
| `PolicyViolationException` | Cluster policy rejects the partition count or retention | Confluent Cloud typically requires **≥ 3 partitions**; raise `topicpartitions` |
| `TimeoutException` | Brokers unreachable | See [§2](#2-cannot-reach-the-brokers) |
| `TopicExistsException` | The topic already exists | No action; the send proceeds normally |

> 📌 The defaults are `topicpartitions: 1` and `topicreplicas: 3`. The replica default suits a 3-broker cloud cluster; a single-broker development cluster needs `1`. Confluent Cloud enforces a minimum partition count, so it typically needs `3` or more. Expect to tune one of the two for your environment.

### Topic created but with the wrong settings

If a topic was auto-created by the broker before OneConnect created it explicitly, it carries broker defaults rather than the configured retention, message size, and compression.

**To inspect it:** in AKHQ, open **Topics** → your topic → the **Configs** tab. This lists the effective settings, including `max.message.bytes`, `retention.ms`, `cleanup.policy`, and `compression.type`. The **Partitions** tab shows partition count, replicas, and in-sync replicas.

**To correct it:** edit the values in the same **Configs** tab and save.

Partition count can be increased but not reduced; replication factor changes require a partition reassignment through Strimzi.

---

## 5. Message Size Limits

### Symptom

```
org.apache.kafka.common.errors.RecordTooLargeException:
  The message is 12582912 bytes when serialized which is larger than the maximum request size
```

SAP tables with many rows per payload, or wide tables with long text fields, exceed the default 8 MB.

### The four limits that must agree

| Limit | Where | Default here |
|---|---|---|
| `max.request.size` | Producer client (`kafkaproperties.maxrequestsize`) | 8 MB |
| `max.message.bytes` | Topic (`kafkaproperties.topicmessagesize`) | 8 MB |
| `message.max.bytes` | Broker | Cluster-dependent |
| `fetch.max.bytes` | Consumer | Client default |

Raising only the client limit produces a broker-side rejection instead; raising only the topic limit still fails client-side. **Raise them together.**

> 📌 Confluent Cloud caps message size at **8 MB**, so on Confluent Cloud the adjustment is made by sending fewer records per payload rather than by raising the limit.

### Ways to resolve it

1. **Send fewer records per request.** The entity controls this through its **Number of Records** setting — see [Entity Creation and Main Customizing](../SAP_Data_Modeler/Data_Modeler_Manuals/01-Entity_Creation_and_Main_Customizing.md). This is the option available on Confluent Cloud, where the 8 MB cap cannot be raised.
2. **Raise the limits** on a self-managed cluster, in all four places above.
3. **Check whether the payload grew** — compare the current record against an earlier one in AKHQ to see whether a nested table was added.

---

## 6. ACLs and Authorization

### Symptoms

```
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [732_DEV_MARA]
org.apache.kafka.common.errors.ClusterAuthorizationException: Cluster authorization failed.
org.apache.kafka.common.errors.GroupAuthorizationException: Not authorized to access group: group_0
```

### Permissions required

| Component | Operation | Resource |
|---|---|---|
| Producer | `CREATE` | Cluster (to create its raw topic) |
| Producer | `WRITE`, `DESCRIBE` | Its raw topic and `*_INBOUND_ERRORS` |
| Consumer | `READ`, `DESCRIBE` | The raw topic |
| Consumer | `READ` | Consumer group `group_0` |
| Consumer | `CREATE` | Cluster (one topic per SAP table) |
| Consumer | `WRITE`, `DESCRIBE` | Per-table topics and `PROCESSING_ERRORS` |

> 📌 The Consumer needs **cluster-level `CREATE`**, since it creates a topic the first time it sees each SAP table. A key scoped to a fixed topic list will cover existing tables but not new ones; the outcome is reported at `DEBUG` — see [§0](#0-before-you-start-turn-on-the-detail).

Prefix-based ACLs are the practical approach, since topic names follow `<CLIENT>_<ENV>_*`.

On a Strimzi cluster, ACLs are declared on the `KafkaUser` resource, so review them with `kubectl`:

```bash
kubectl get kafkauser -n kafka
kubectl describe kafkauser -n kafka <user>
```

For an external cluster such as Confluent Cloud, manage ACLs from that provider's console.

---

## 7. The Consumer Connects but Consumes Nothing

Distinguish "cannot connect" from "connected, nothing to read".

In AKHQ, open **Consumer Groups** and select `group_0`. The view lists its members, the topics and partitions assigned, and the lag per partition.

| What you see | Meaning | Next step |
|---|---|---|
| Group not found | The Consumer never connected | [§2](#2-cannot-reach-the-brokers), [§3](#3-authentication-rejected-by-the-brokers) |
| Group exists, no members | The Consumer crashed or is not running | `kubectl get pods`, check for a crash loop |
| Members present, `LAG` = 0, `CURRENT-OFFSET` = 0 | Connected, but the topic is empty | The Producer is not writing — go upstream |
| Members present, `LAG` growing | Records exist but processing fails | [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md) |
| Group subscribed to an unexpected topic | Topic name mismatch | [06 — No Data Reaching Destination](./06-No_Data_Reaching_Destination.md#2-the-consumer-is-subscribed-to-a-topic-nobody-writes-to) |

The Consumer starts from `earliest`, so a newly deployed Consumer replays the entire topic. A large initial lag on a fresh deployment is expected and should shrink steadily.

### Two Consumers, one group

All Consumers share the group `group_0` by default. Two Consumers for different SAP Links against the same cluster will **split the partitions between them**, so each sees only part of the data — and, subscribed to different topics, they will trigger continuous rebalances.

```
Attempt to heartbeat failed since group is rebalancing
```

**Fix:** give each SAP Link a distinct `kafkaproperties.groupid`.

---

## 8. Rebalances and Poll Timeouts

```
org.apache.kafka.clients.consumer.CommitFailedException:
  Offset commit cannot be completed since the consumer is not part of an active group
```

The Consumer polls on a fixed schedule with `max.poll.interval.ms` set to **600000** (10 minutes). If processing one batch takes longer than that, the broker evicts the Consumer and the batch is reprocessed — producing duplicates and, if the slow batch repeats, a permanent loop.

| Cause | Fix |
|---|---|
| A very large payload takes minutes to convert | Reduce the SAP batch size |
| Schema Registry is slow or timing out | See [05](./05-Avro_Serialization_and_Schema_Registry.md) |
| Destination backpressure | Check Kafka Connect and the destination database |
| Consumer is CPU-throttled | Raise its CPU limit |

The Consumer also skips a poll cycle while the previous one is still running, logging at `DEBUG`:

```
Skipping this poll cycle because previous is still running
```

Frequent occurrences mean processing is slower than the poll interval — a throughput warning, not an error.

---

## 9. Quick Command Reference

### In AKHQ

| Task | Where |
|---|---|
| List topics and their record counts | **Topics** |
| Partitions, replicas, in-sync replicas | **Topics** → topic → **Partitions** |
| Effective topic configuration | **Topics** → topic → **Configs** |
| Read the newest messages (Avro decoded) | **Topics** → topic → **Data** |
| Consumer group members and lag | **Consumer Groups** → `group_0` |
| Registered schemas | **Schema Registry** |

### With kubectl

```bash
# AKHQ address
kubectl get kafka -n kafka
kubectl get pods  -n kafka
kubectl logs -n kafka -l strimzi.io/kind=cluster-operator --tail=100
```

AKHQ provides the same information in a browser:

```bash
kubectl get svc -n kafka | grep akhq
```

---

## Summary

1. **Topic creation reports at `DEBUG`** — enable it before diagnosing a missing topic.
2. `deploymenttype` must be exactly `CLOUD` or `ONPREMISE`; any other value disables all security settings and disguises itself as a network timeout.
3. Timeouts mean **network**; `SaslAuthenticationException` means **credentials**. Do not conflate them.
4. Review `topicpartitions` and `topicreplicas` against your cluster size — the defaults suit a 3-broker cloud cluster.
5. The Consumer needs **cluster-level `CREATE`**, because it creates a topic per SAP table on first sight.
6. Give each SAP Link its **own `groupid`**, or Consumers will split partitions and rebalance endlessly.
