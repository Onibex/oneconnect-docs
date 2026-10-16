# Authentication & Credentials

There are **five independent credentials** in a OneConnect deployment. Each one is checked in a different place and produces a distinct symptom, so identify which one is involved before changing anything.

| # | Credential | Between | Typical symptom |
|---|---|---|---|
| 1 | **SAP → Producer Basic auth** | SAP RFC destination → Producer HTTP endpoint | SAP receives `401 Unauthorized` when sending data |
| 2 | **Kafka SASL** | Producer/Consumer → Kafka brokers | `SaslAuthenticationException`, no topics created |
| 3 | **Schema Registry basic auth** | Consumer → Schema Registry | Consumer log shows `Unauthorized; error code: 401` |
| 4 | **Soft security token** | Producer/Consumer → API Gateway reporter routes (logs and metrics) | Logs panel stays empty when log shipping goes through the API Gateway |
| 5 | **Platform login (JWT)** | Browser → API Gateway → Auth service | Cannot log in to the UI |

Credentials 2 and 3 are covered in depth in [04 — Kafka Connectivity](./04-Kafka_Connectivity.md) and [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md); this guide covers where they are set and how to tell them apart.

---

## 1. SAP → Producer: HTTP Basic Authentication

### How it works

The Producer uses standard HTTP Basic authentication; user and password must match the SAP Link credentials (`SAPPROPERTIES_USER` and `SAPPROPERTIES_PASSWORD` in the Producer pod) exactly, including case.

The endpoints are:

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/v1/saplistener/<workspace-id>` | Connection test (reachability) |
| `POST` | `/api/v1/saplistener/<workspace-id>` | Asynchronous ingestion |
| `GET` | `/api/v1/saplistener/<workspace-id>/synchronous` | Connection test (reachability) |
| `POST` | `/api/v1/saplistener/<workspace-id>/synchronous` | Synchronous ingestion (recommended), returns the Kafka offset |

The URL shown on the SAP Link screen already points to the `/synchronous` endpoint; use it as-is in SM59 (see [Choosing the Endpoint](../SAP_Data_Modeler/Data_Modeler_Manuals/15-Endpoint_Customization_SM59.md#choosing-the-endpoint-synchronous-vs-asynchronous)).

> 📌 The SM59 **Connection Test** uses `GET`, which confirms that SAP can reach the Producer on the right path. Credentials are validated when SAP sends data with `POST`. A passing connection test followed by a `401` on data transfer therefore points to the credentials.

### Reading the HTTP status code

This is the reference table for every status the Producer returns to SAP.

| Status | Request | Meaning | Where to look |
|---|---|---|---|
| `200` | `GET` (connection test) | The endpoint is reachable on this path | Nothing to fix; credentials are checked on `POST` |
| `201` | `POST` | Accepted. On the synchronous endpoint the body contains the Kafka offset | Data reached the Producer; any further question is downstream |
| `400` | `POST` | The `Authorization` header is missing | [§3](#3-400-and-500-malformed-headers-and-payloads) |
| `400` | `POST` synchronous | Credentials were accepted, but no Kafka offset was produced: the payload is not valid JSON, or the Kafka send did not complete | [§3](#3-400-and-500-malformed-headers-and-payloads) |
| `401` | `POST` | The header was read, but the user or password does not match the SAP Link | [§2](#2-diagnosing-a-401-from-sap) |
| `404` | Any | The path does not match this Producer: check the path prefix and the workspace ID | Copy the URL from the SAP Link screen; see [SM59 Step 5](../SAP_Data_Modeler/Data_Modeler_Manuals/15-Endpoint_Customization_SM59.md#step-5-configure-technical-settings) |
| `500` | `POST` | The `Authorization` header or the payload does not have the expected shape | [§3](#3-400-and-500-malformed-headers-and-payloads) |

> ℹ️ Any other code (or no HTTP response at all) comes from a network component between SAP and the Smart Gateway, such as the SAP ICM, a proxy, the ingress, or the load balancer. Review host, port, SSL settings and the network path.

---

## 2. Diagnosing a `401` from SAP

**Symptom** — SAP receives `401 Unauthorized` when sending data. On the synchronous endpoint the Producer also logs, at `ERROR` level:

```
-->Authentication error. Incorrect username and/or password.
```

**Cause** — The user or password sent by SM59 does not match the credentials configured for this SAP Link.

### Step 1 — Find the configured credentials

The Producer deployment of a SAP Link is named `<workspace-id>-producer` in the `oneconnect` namespace.

```bash
kubectl get deploy -n oneconnect -l oneconnect=<workspace-id>-producer
kubectl exec -n oneconnect deploy/<workspace-id>-producer -- env | grep SAPPROPERTIES_
```

`SAPPROPERTIES_USER` and `SAPPROPERTIES_PASSWORD` are the values SAP must send.

### Step 2 — Reproduce the call yourself

This removes SAP from the equation. Forward the Producer service port to your workstation:

```bash
kubectl port-forward -n oneconnect svc/producer-<workspace-id> 8080:8080
```

Then, from a second terminal:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -u '<user>:<password>' \
  -X POST -H 'Content-Type: application/json' \
  -d '{"test":"ping"}' \
  http://localhost:8080/api/v1/saplistener/<workspace-id>/synchronous
```

> ⚠️ Run this test on a non-production SAP Link. On production, use the `GET` connection test, or send a valid message from SAP.

| Result | Conclusion |
|---|---|
| `201` | Credentials are correct; review the SM59 configuration (Step 4) |
| `401` | The credentials do not match what the Producer expects |
| `400` | The header is not arriving; check how you are passing it |
| `404` | The workspace ID in the path does not match this Producer |

### Step 3 — Common causes

| Cause | Detail | Fix |
|---|---|---|
| Credentials changed but the Producer still runs with the previous values | The Producer reads its configuration at startup | Confirm the values with Step 1, then `kubectl rollout restart deployment/<workspace-id>-producer -n oneconnect` |
| Leading or trailing whitespace in the SM59 user or password | The values must match exactly | Retype the values in SM59 without spaces |
| Password characters | — | Use passwords with letters, digits, and common symbols, excluding `:` |
| SM59 targets a different SAP Link | Each SAP Link has its own Producer and its own credentials | Confirm that the workspace ID in the SM59 path matches the SAP Link whose credentials you entered |

> 📌 The synchronous endpoint logs `-->Authentication error. Incorrect username and/or password.`, so the Logs panel is the quickest check there. On the asynchronous endpoint, read the status code returned to SAP or reproduce the call with `curl` as in Step 2.

### Step 4 — Check the SAP side

The RFC destination is configured in `SM59`. See [Endpoint Customization (SM59)](../SAP_Data_Modeler/Data_Modeler_Manuals/15-Endpoint_Customization_SM59.md) for the full procedure. Verify:

- Host, port, and path prefix come from the URL shown on the SAP Link screen, used as-is.
- Logon user and password are set under the **Logon & Security** tab, with Basic authentication selected.
- SSL settings match the endpoint scheme (`http` vs `https`).

---

## 3. `400` and `500`: Malformed Headers and Payloads

### `400 Bad Request`

| Cause | How to recognize it | Fix |
|---|---|---|
| The `Authorization` header is missing | The Producer logs a `WARN` line containing `MissingRequestHeaderException` and `Required request header 'Authorization'` | Set user and password under **Logon & Security** in SM59 with Basic authentication; make sure no proxy removes the header |
| Synchronous request with a payload that is not valid JSON | The Producer logs `-->MalformedJsonException =Json non valid` and writes the payload to `<CLIENT>_<ENV>_INBOUND_ERRORS` | See [06 — No Data Reaching the Destination](./06-No_Data_Reaching_Destination.md#4-the-producer-is-not-writing-to-kafka) |
| Synchronous request whose Kafka send did not complete | The Producer logs `-->ExecutionException =<exception>: <message>` | Route by the exception in [04 — Kafka Connectivity](./04-Kafka_Connectivity.md) |

In the last two cases authentication succeeded; the `400` tells SAP that no Kafka offset was produced for that request.

```bash
kubectl logs -n oneconnect deploy/<workspace-id>-producer --tail=200 \
  | grep -E 'MissingRequestHeaderException|MalformedJsonException|ExecutionException'
```

### `500 Internal Server Error` on authentication

A header that is not HTTP Basic `user:password` (or an empty password) returns `500`; configure Basic auth in SM59 with both user and password.

### `500` caused by the payload

| Cause | How to recognize it | Fix |
|---|---|---|
| The payload is valid JSON but not an object (for example a top-level array) | Producer log contains `JsonSyntaxException` | Send one JSON object per request |
| The synchronous Kafka send could not start | Producer log contains `-->Error Sending Record to topic=<topic>, value=..., errorType=<class>, cause=<message>` | Treat it as a Kafka problem: [04 — Kafka Connectivity](./04-Kafka_Connectivity.md). Once resolved, restart the Producer |

```bash
kubectl logs -n oneconnect deploy/<workspace-id>-producer --tail=200 | grep -Ei 'error|exception'
```

---

## 4. Kafka SASL Credentials

Used by both Producer and Consumer to authenticate to the brokers. The Builder stores the bootstrap address, SASL user and password, and security protocol of each SAP Link in a Kubernetes secret and injects them into the pods.

| Property (env var) | Meaning |
|---|---|
| `kafkaproperties.sasluser` (`KAFKAPROPERTIES_SASLUSER`) | API key / username |
| `kafkaproperties.saslpassword` (`KAFKAPROPERTIES_SASLPASSWORD`) | API secret / password |
| `kafkaproperties.saslmechanism` | `PLAIN` by default |
| `kafkaproperties.securityprotocol` (`KAFKAPROPERTIES_SECURITYPROTOCOL`) | `SASL_SSL` by default |
| `kafkaproperties.deploymenttype` (`KAFKAPROPERTIES_DEPLOYMENTTYPE`) | `CLOUD` or `ONPREMISE` |

### The deployment type changes which settings apply

| `deploymenttype` | Behavior |
|---|---|
| `CLOUD` | `security.protocol` is set to `SASL_SSL`; the `securityprotocol` value is not used |
| `ONPREMISE` | `securityprotocol` is used; the special value `NO_SECURITY` connects without authentication |

> 📌 Set `deploymenttype` to `CLOUD` or `ONPREMISE`.

```bash
kubectl exec -n oneconnect <pod> -- env | grep -E 'KAFKAPROPERTIES_(DEPLOYMENTTYPE|SECURITYPROTOCOL|SASLUSER)'
```

Full diagnosis in [04 — Kafka Connectivity](./04-Kafka_Connectivity.md).

---

## 5. Schema Registry Basic Auth

Used by the Consumer only, when writing Avro.

| Property (env var) | Expected value | Notes |
|---|---|---|
| `kafkaproperties.schemauser` (`KAFKAPROPERTIES_SCHEMAUSER`) | `USER_INFO` | This is the **credentials source**, not a username. The Builder sets it to `USER_INFO` |
| `kafkaproperties.schemapassword` (`KAFKAPROPERTIES_SCHEMAPASSWORD`) | `<KEY>:<SECRET>` | The **whole credential pair**, colon-separated |
| `kafkaproperties.schemaregistryurl` (`KAFKAPROPERTIES_SCHEMAREGISTRYURL`) | Registry URL | — |

Two settings are worth checking first for a Schema Registry `401`:

1. **`schemauser` holds a username.** It must stay `USER_INFO`.
2. **`schemapassword` holds only the secret.** It must be `KEY:SECRET`, both halves separated by a colon.

> 📌 Schema Registry basic authentication is configured with `CLOUD`.

See [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md#7-schema-registry-returns-401) for the full diagnosis.

---

## 6. Internal Soft Security Token

**Symptom** — The Logs panel (or the metrics dashboard) stays empty for a SAP Link, and the Producer or Consumer stdout shows:

```
ERROR sending logs: 401 Unauthorized ...
```

For metrics, the pods log `WARN Disconnected from metrics` on every push.

**Cause** — The Producer and Consumer send their logs and metrics with `Authorization: Bearer <ONECONNECT_SOFTSECURITYTOKEN>`. When these requests go through the **API Gateway**, the gateway validates the token on the `/logs/reporter/**` and `/metrics/reporter/**` routes against its own `AUTH_SOFTAUTHTOKEN`, as configured at install. The two values must match.

> ℹ️ This applies when `ONECONNECT_LOGS_URL_PRODUCER`, `ONECONNECT_LOGS_URL_CONSUMER`, or `ONECONNECT_METRICS_URL` point at the API Gateway.

**Confirm**

```bash
# Where the pods send logs and metrics, and the token they use
kubectl exec -n oneconnect deploy/<workspace-id>-producer -- env | grep -E 'ONECONNECT_(LOGS_URL|METRICS_URL|SOFTSECURITYTOKEN)'
kubectl exec -n oneconnect deploy/<workspace-id>-consumer -- env | grep -E 'ONECONNECT_(LOGS_URL|METRICS_URL|SOFTSECURITYTOKEN)'

# Rejections are visible on the sender side
kubectl logs -n oneconnect deploy/<workspace-id>-producer | grep "ERROR sending logs"

# The value the API Gateway expects
kubectl get deploy apigateway -n oneconnect -o yaml | grep -A1 AUTH_SOFTAUTHTOKEN
```

Confirm from the sender's log (`ERROR sending logs`).

**Fix**

1. Choose one token value.
2. Set it as `AUTH_SOFTAUTHTOKEN` on the API Gateway and as `ONECONNECT_SOFTSECURITYTOKEN` on the Producer and Consumer.
3. Set it as `ONECONNECT_SOFTTOKEN` on the Builder, which passes it to the Producer and Consumer of SAP Links it deploys.
4. Restart the workloads you changed ([§8](#8-credential-change-checklist)).

Full Logs panel checklist in [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md#3-the-logs-panel-is-empty).

---

## 7. Cannot Log In to the Platform UI

The browser talks to the **API Gateway**, which routes `/auth/**` to the Auth service. These are the deployment settings that affect login.

```bash
kubectl get pods -n oneconnect -l app=auth
kubectl logs  -n oneconnect -l app=auth --tail=100
kubectl logs  -n oneconnect -l app=apigateway --tail=100
```

### The UI cannot reach the API Gateway

**Cause** — The UI sends its requests to `NEXT_PUBLIC_API_BASE`, which must be a gateway address the **browser** can reach (`http://<gateway-address>:9000`). The in-cluster default `http://apigateway.oneconnect.svc.cluster.local:9000` resolves only inside Kubernetes.

**Confirm** — In the browser developer tools, login requests fail with `net::ERR_NAME_NOT_RESOLVED` or a connection error, or go to an unexpected host.

```bash
kubectl get deploy frontendoneconnect -n oneconnect -o yaml | grep -A1 NEXT_PUBLIC_API_BASE
kubectl get svc internal-apigateway -n oneconnect
```

**Fix**

1. With the load balancer enabled for the gateway (`networking.loadBalancer.apigateway.enabled`), the chart's post-install/post-upgrade job `frontend-apigateway-sync` waits for the `internal-apigateway` address and sets `NEXT_PUBLIC_API_BASE` on the frontend. Confirm the service received an external address and the job completed.
2. Otherwise, set `networking.loadBalancer.apigateway.externalIP` to a browser-reachable gateway address and upgrade the release.
3. Restart the frontend after changing the value.

### Every login is rejected

| Cause | Confirm | Fix |
|---|---|---|
| `JWT_SECRET` is not valid Base64, or decodes to fewer than 32 bytes | Auth log shows `WeakKeyException` or `DecodingException` on sign-in | Set `JWT_SECRET` to a Base64 string of at least 32 bytes and restart Auth |
| Auth cannot reach its database | Auth log shows JDBC connection errors | Confirm MySQL is running and that `mysql-secret` matches the database ([02 — Installation & Deployment](./02-Deployment_and_Installation.md)) |
| The user has not completed registration | The user exists only after finishing registration from the link in the activation email | Configure SMTP so the email is delivered: [Configuring the Email Service](../Smart_Gateway/Using_the_Smart_Gateway/01-Configuring_Email_Service_SMTP.md); check `kubectl logs -n oneconnect -l app=emailbuilder --tail=100` |

> 📌 Set `JWT_SECRET` (Base64, at least 32 bytes) at install.

### Login succeeds but secured screens return `500`

**Cause** — The API Gateway validates each JWT with the Auth service at `AUTH_URL`. Its default value targets a release installed in the `oneconnect` namespace.

**Confirm** — The gateway log shows `Failure in token verification`.

**Fix**

1. Confirm the Auth pod is running.
2. For a release in another namespace, set `AUTH_URL` on the API Gateway to the Auth service address and restart the gateway.
3. After rotating `JWT_SECRET`, log in again so the browser receives a new token.

For granting an existing user access to a SAP Link (a permissions topic), see [SAP Link Access](../Smart_Gateway/Using_the_Smart_Gateway/06-SAP_Link_Access.md#quick-troubleshooting).

---

## 8. Credential Change Checklist

Configuration is read at **startup**. Restart the workload after changing a value so it takes effect.

| Changed | Restart |
|---|---|
| SAP Link user/password | Producer |
| Kafka API key/secret | Producer **and** Consumer |
| Schema Registry credentials | Consumer |
| Soft security token | API Gateway, Producer **and** Consumer (and Builder for new SAP Links) |
| Docker Hub token | Nothing, but delete failing pods so they re-pull |
| `JWT_SECRET` | Auth (users log in again afterwards) |
| `NEXT_PUBLIC_API_BASE` | Frontend |

```bash
kubectl rollout restart deployment/<name> -n oneconnect
kubectl rollout status  deployment/<name> -n oneconnect
```

---

## Summary

1. Identify **which of the five credentials** is involved before changing anything; each has a distinct symptom.
2. The SM59 connection test (`GET`) confirms reachability: `200` = reachable, `404` = check path prefix and workspace ID. Credentials are validated on `POST`.
3. On `POST`: `401` = credentials · `400` = missing header, or no offset on the synchronous endpoint (invalid JSON or Kafka send) · `500` = malformed header or payload.
4. Use the SAP Link URL as shown (it already ends in `/synchronous`), and avoid `:` in the password.
5. Set `deploymenttype` to `CLOUD` or `ONPREMISE`.
6. Schema Registry needs `schemauser=USER_INFO` and `schemapassword=KEY:SECRET`, configured with `CLOUD`.
7. The soft security token is validated by the API Gateway on `/logs/reporter/**` and `/metrics/reporter/**`; keep `AUTH_SOFTAUTHTOKEN` and `ONECONNECT_SOFTSECURITYTOKEN` equal.
8. Login depends on a browser-reachable `NEXT_PUBLIC_API_BASE` and a valid Base64 `JWT_SECRET`.
9. Always **restart the workload** after changing a credential.
