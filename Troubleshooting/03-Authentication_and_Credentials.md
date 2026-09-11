# Authentication & Credentials

There are **five independent credentials** in a OneConnect pipeline. They fail in different places and produce very different symptoms. Identify which one is involved before changing anything.

| # | Credential | Between | Typical failure |
|---|---|---|---|
| 1 | **SAP → Producer Basic auth** | SAP RFC destination → Producer HTTP endpoint | SAP shows `401 Unauthorized` |
| 2 | **Kafka SASL** | Producer/Consumer → Kafka brokers | `SaslAuthenticationException`, no topics created |
| 3 | **Schema Registry basic auth** | Consumer → Schema Registry | `401 Unauthorized` from the registry, records fail to serialize |
| 4 | **Soft security token** | Producer/Consumer → Logs & Metrics services | Logs panel stays empty |
| 5 | **Platform login (JWT)** | Browser → Auth service | Cannot log in to the UI |

Credentials 2 and 3 are covered in depth in [04 — Kafka Connectivity](./04-Kafka_Connectivity.md) and [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md); this guide covers where they are set and how to tell them apart.

---

## 1. SAP → Producer: HTTP Basic Authentication

### How it works

SAP calls the Producer with an HTTP `Basic` authorization header. The Producer:

1. Strips the first 6 characters (`Basic `).
2. Base64-decodes the remainder.
3. Splits on the first `:` into user and password.
4. Compares both against `sapproperties.user` and `sapproperties.password`.

Both must match **exactly** — the comparison is case-sensitive and whitespace-sensitive.

The endpoints are:

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/<prefix>/<workspace-id>` | Connection test — **no authentication**, always returns `200` |
| `POST` | `/<prefix>/<workspace-id>` | Asynchronous ingestion |
| `GET` | `/<prefix>/<workspace-id>/synchronous` | Connection test — **no authentication** |
| `POST` | `/<prefix>/<workspace-id>/synchronous` | Synchronous ingestion, returns the Kafka offset |

The default prefix is `api/v1/saplistener`.

> 📌 The `GET` endpoints do not authenticate, so a passing SM59 "Connection Test" confirms reachability rather than credentials. A passing test alongside a `401` on data transfer is consistent.

### Reading the HTTP status code

| Status | Meaning | Where to look |
|---|---|---|
| `401` | Credentials were parsed but did not match | [§2](#2-diagnosing-a-401-from-sap) |
| `400` | The `Authorization` header was **absent**, or the synchronous call produced no offset | The header is mandatory; a missing header is rejected before the credential check |
| `500` | The header was present but malformed | [§3](#3-400-and-500-malformed-headers-and-payloads) |
| `201` | Accepted | Data reached the Producer; any further problem is downstream |

---

## 2. Diagnosing a `401` from SAP

### Step 1 — Find the configured credentials

```bash
kubectl exec -n oneconnect <producer-pod> -- env | grep -i sapproperties
```

If they are set in the properties file rather than the environment:

```bash
kubectl exec -n oneconnect <producer-pod> -- \
  sh -c 'grep sapproperties /app/application.properties 2>/dev/null'
```

### Step 2 — Reproduce the call yourself

This removes SAP from the equation:

```bash
kubectl exec -n oneconnect <producer-pod> -- \
  curl -s -o /dev/null -w "%{http_code}\n" \
  -u '<user>:<password>' \
  -X POST -H 'Content-Type: application/json' \
  -d '{"test":"ping"}' \
  http://localhost:50501/api/v1/saplistener/<workspace-id>/synchronous
```

| Result | Conclusion |
|---|---|
| `201` | Credentials are correct — the problem is on the SAP side (SM59 config) or in the network path |
| `401` | Credentials genuinely do not match what the Producer expects |
| `400` | The header is not arriving — check how you are passing it |

### Step 3 — Common causes

| Cause | Detail | Fix |
|---|---|---|
| Password changed in the UI but the pod was not restarted | The Producer reads its configuration at startup | `kubectl rollout restart deployment/<producer> -n oneconnect` |
| Trailing whitespace in the SM59 password field | The comparison is exact | Retype the password in SM59, no leading/trailing spaces |
| Wrong workspace ID in the URL | Routes to a different Producer with different credentials | Copy the endpoint URL from the SAP Link screen verbatim |
| A colon inside the password | The header is split on `:`, so the password is truncated at the first colon | **Do not use `:` in the password** |
| Non-ASCII characters in the password | Base64 decoding assumes the platform charset | Use ASCII-only credentials |
| Testing against the wrong SAP Link | Each SAP Link has its own Producer and its own credentials | Confirm which SAP Link the SM59 destination targets |

> 📌 The `/synchronous` path logs `-->Authentication error. Incorrect username and/or password.` at `ERROR` level, so the Logs panel is the quickest check there. For the asynchronous endpoint, read the status code returned to SAP or reproduce the call with `curl`.

### Step 4 — Check the SAP side

The RFC destination is configured in `SM59`. See [Endpoint Customization (SM59)](../SAP_Data_Modeler/Data_Modeler_Manuals/15-Endpoint_Customization_SM59.md) for the full procedure. Verify:

- Target host and port match the SAP Link endpoint.
- Path includes the full prefix **and** the workspace ID.
- Logon user and password are set under the **Logon & Security** tab, with Basic authentication selected.
- SSL settings match the endpoint scheme (`http` vs `https`).

---

## 3. `400` and `500`: Malformed Headers and Payloads

### `400 Bad Request`

Two distinct causes:

| Cause | Distinguishing feature |
|---|---|
| The `Authorization` header is missing entirely | Rejected before any credential check; nothing is logged |
| A synchronous request authenticated successfully but produced no Kafka offset | The Producer logs a Kafka error alongside it |

The second case means authentication is fine and the failure is in the Kafka send — go to [04 — Kafka Connectivity](./04-Kafka_Connectivity.md).

### `500 Internal Server Error` on authentication

Raised when the header is present but does not have the expected shape:

| Malformed header | Result |
|---|---|
| Shorter than 6 characters | Index error while stripping the `Basic ` prefix |
| A different scheme (e.g. `Bearer <token>`) | The first 6 characters are stripped blindly and the rest fails to decode |
| Decoded value contains no `:` | Index error while splitting user from password |
| Not valid Base64 | Decoding error |

**Fix:** configure SM59 to send HTTP **Basic** authentication. A `Bearer` token or a custom scheme returns `500` rather than `401`, so a `500` here points to the authentication scheme.

### `500` on the synchronous endpoint after authentication succeeds

If the Kafka send throws, the synchronous path can fail while trying to read the resulting offset. This surfaces as a `500` to SAP, with a Kafka error in the pod logs:

```bash
kubectl logs -n oneconnect <producer-pod> --tail=200 | grep -Ei 'error|exception'
```

Treat it as a Kafka problem — [04 — Kafka Connectivity](./04-Kafka_Connectivity.md).

---

## 4. Kafka SASL Credentials

Used by both Producer and Consumer to authenticate to the brokers.

| Property | Meaning |
|---|---|
| `kafkaproperties.sasluser` | API key / username |
| `kafkaproperties.saslpassword` | API secret / password |
| `kafkaproperties.saslmechanism` | `PLAIN` by default |
| `kafkaproperties.securityprotocol` | `SASL_SSL` by default |
| `kafkaproperties.deploymenttype` | `CLOUD` or `ONPREMISE` |

### The deployment type changes which settings apply

This is the most frequent source of confusion:

| `deploymenttype` | Behavior |
|---|---|
| `CLOUD` | `security.protocol` is **forced to `SASL_SSL`**; your `securityprotocol` value is ignored |
| `ONPREMISE` | `securityprotocol` is honoured; the special value `NO_SECURITY` disables authentication entirely |
| Anything else | **No security settings are applied at all** — the client attempts a plaintext connection |

> 📌 A typo in `deploymenttype` (`Cloud ` with a trailing space, `CLOD`, `on-premise`) matches neither branch, leaving the client without `security.protocol`, SASL, or credentials. This presents as a timeout or disconnect rather than an authentication error, so check this value when a reachable cluster times out.

```bash
kubectl exec -n oneconnect <pod> -- env | grep -i 'deploymenttype\|securityprotocol\|saslmechanism'
```

Full diagnosis in [04 — Kafka Connectivity](./04-Kafka_Connectivity.md).

---

## 5. Schema Registry Basic Auth

Used by the Consumer only, when writing Avro.

| Property | Expected value | Notes |
|---|---|---|
| `kafkaproperties.schemauser` | `USER_INFO` | This is the **credentials source**, not a username. Leave it as `USER_INFO` |
| `kafkaproperties.schemapassword` | `<KEY>:<SECRET>` | The **whole credential pair**, colon-separated |
| `kafkaproperties.schemaregistryurl` | Registry URL | — |

Two settings are worth checking first for a Schema Registry `401`:

1. **Putting a username in `schemauser`.** It must stay `USER_INFO`; anything else makes the client use an unknown credential source and skip authentication.
2. **Putting only the secret in `schemapassword`.** It must be `KEY:SECRET` — both halves, separated by a colon.

> 📌 Schema Registry authentication is configured when `deploymenttype` is `CLOUD`. For an on-premise registry that requires authentication, run the registry without authentication or front it with a proxy that supplies the credentials.

See [05 — Avro & Schema Registry](./05-Avro_Serialization_and_Schema_Registry.md#7-schema-registry-returns-401) for the full diagnosis.

---

## 6. Internal Soft Security Token

`oneconnect.softsecuritytoken` (env: `ONECONNECT_SOFTSECURITYTOKEN`) is the bearer token the Producer and Consumer use to post logs and metrics to the internal Logs and Metrics services.

The usual symptom is an empty Logs panel. Since the log channel itself is involved, check the pod's stdout rather than the UI:

```bash
kubectl logs -n oneconnect <pod> | grep "ERROR sending logs"
```

Verify the token matches across components:

```bash
kubectl exec -n oneconnect <producer-pod> -- env | grep ONECONNECT_SOFTSECURITYTOKEN
kubectl exec -n oneconnect <consumer-pod> -- env | grep ONECONNECT_SOFTSECURITYTOKEN
kubectl logs -n oneconnect -l app=logs --tail=100 | grep -Ei '401|unauthor|forbidden'
```

A `401` in the Logs service confirms a token mismatch. Full checklist in [01 — Diagnostic Toolkit](./01-Diagnostic_Toolkit.md#3-the-logs-panel-is-empty).

---

## 7. Cannot Log In to the Platform UI

This is the Auth service, unrelated to the data pipeline.

```bash
kubectl get pods -n oneconnect -l app=auth
kubectl logs  -n oneconnect -l app=auth --tail=100
```

| Symptom | Cause | Fix |
|---|---|---|
| "Invalid credentials" for a user you just created | The account is not yet activated | Activation requires a working SMTP server — see [Configuring the Email Service](../Smart_Gateway/Using_the_Smart_Gateway/01-Configuring_Email_Service_SMTP.md) |
| No activation email arrives | SMTP not configured, or `emailbuilder` unhealthy | `kubectl logs -n oneconnect -l app=emailbuilder --tail=100` |
| Login works, then the session drops | JWT expiry | Default is 4 hours (`JWT_EXPIRATIONMS`); adjust in values if needed |
| Every login fails after a re-install | The Auth database holds the previous user data | Confirm the credentials in `mysql-secret` match the database |
| Login succeeds but the UI shows no data | API Gateway cannot reach Auth for token validation | `kubectl logs -n oneconnect -l app=apigateway --tail=100` |

> 📌 Set your own `JWT_SECRET` before going to production. The chart ships with a default value intended for evaluation.

For granting an existing user access to a SAP Link — a permissions issue rather than an authentication one — see [SAP Link Access](../Smart_Gateway/Using_the_Smart_Gateway/06-SAP_Link_Access.md#quick-troubleshooting).

---

## 8. Credential Change Checklist

Configuration is read at **startup**. Changing a value without restarting the workload has no effect, which regularly reads as "the fix did not work".

| Changed | Restart |
|---|---|
| SAP Link user/password | Producer |
| Kafka API key/secret | Producer **and** Consumer |
| Schema Registry credentials | Consumer |
| Soft security token | Producer **and** Consumer |
| Docker Hub token | Nothing — but delete failing pods so they re-pull |
| `JWT_SECRET` | Auth, API Gateway |

```bash
kubectl rollout restart deployment/<name> -n oneconnect
kubectl rollout status  deployment/<name> -n oneconnect
```

---

## Summary

1. Identify **which of the five credentials** is involved before changing anything — the symptoms are distinct.
2. `GET` connection tests do **not** authenticate; a passing SM59 connection test says nothing about the password.
3. `401` = wrong credentials · `400` = missing header · `500` = malformed header or wrong auth scheme.
4. For the asynchronous endpoint, read the status code returned to SAP or reproduce the call with `curl`.
5. `deploymenttype` must be exactly `CLOUD` or `ONPREMISE`; other values leave the Kafka security settings unset.
6. Schema Registry needs `schemauser=USER_INFO` and `schemapassword=KEY:SECRET`, applied in `CLOUD` mode.
7. Always **restart the workload** after changing a credential.
