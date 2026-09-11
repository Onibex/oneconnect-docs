# Avro Serialization & Schema Registry

The Consumer reads raw JSON from SAP, builds an Avro schema from the SAP metadata, converts every field, and writes an Avro key/value pair per record. Most data-quality failures in OneConnect happen here.

Typical indicators for this layer: the raw topic keeps growing, the per-table topics do not, and records appear in `PROCESSING_ERRORS`.

---

## 1. How the Conversion Works

Understanding the sequence makes every error in this guide self-explanatory.

1. A record arrives containing `metadata` (SAP field definitions) and `body` (the rows).
2. The Consumer builds **two** Avro schemas per table: a **key** schema from fields flagged as key (`keyflag = "X"`), and a **value** schema from all fields.
3. For every row, each schema field is looked up in the row data and converted to the Avro type.
4. The pair is serialized and sent; the schema is registered in the Schema Registry.

### SAP type → Avro type

| SAP type | Avro type | Java type |
|---|---|---|
| `C` (character) | `string` | String |
| `D` (date) | `string` | String |
| `T` (time) | `string` | String |
| `P` (packed decimal) | `double` | Double |
| `N` (numeric text) | `long` | Long |
| `I`, `s` (integer) | `int` | Integer |
| `X` (hex/raw) | `bytes` | ByteBuffer |
| `table` (nested) | `array` of records | List |
| anything else | `string` | String |

### Key fields versus non-key fields

The distinction matters when diagnosing:

| | Key fields (`keyflag = "X"`) | Non-key fields |
|---|---|---|
| Avro declaration | `noDefault()` — **required** | `optional()` — nullable union |
| A null value | **Fails serialization** | Accepted as `null` |
| Appears in | Key schema **and** value schema | Value schema only |

> 📌 Key fields are required, because the key becomes the primary key in the destination. A field marked as key needs a value in every record; a blank one will stop at serialization.

### Topic and subject naming

| Item | Pattern |
|---|---|
| Output topic | `[<CLIENT>_<ENV>_]<TABLE>` — prefix applied only when `useprefix = ENABLE` |
| Output topic (kdoc messages) | `[<CLIENT>_<ENV>_]<ENTITY>` |
| Error topic | `[<CLIENT>_<ENV>_]PROCESSING_ERRORS` |
| Schema subjects | `<topic>-key` and `<topic>-value` |
| Avro namespace | `com.onibex.<tablename-lowercase>` |

---

## 2. Start Here: Read the Error Topic

Every failed record is written to `PROCESSING_ERRORS` **with its payload**, which is far more useful than the log line alone.

In AKHQ, open **Topics** → `PROCESSING_ERRORS` → the **Data** tab. Get the AKHQ address with:

```bash
kubectl get svc -n kafka | grep akhq
```

Each entry looks like:

```
Failed to send message
Error: <exception message>
Data: <the original record>
```

Remember the prefix: if `useprefix = ENABLE`, the topic is `<CLIENT>_<ENV>_PROCESSING_ERRORS`.

Then match the `Error:` line against the sections below.

---

## 3. `NullPointerException` on a Field That Is Empty in SAP

### Symptom

```
errorType=NullPointerException, cause=Cannot invoke "java.lang.Object.getClass()" because "object" is null
```

The stack trace points into the type converter.

### Cause

When a schema field has no corresponding value in the row — the field is absent from the JSON, or explicitly `null` — the converter is still asked to convert it. It has no branch for `null` and fails while building its own error message.

### Diagnosis

Pull the failing record from `PROCESSING_ERRORS` and identify which field is involved:

1. List the field names in the record's `metadata` block.
2. List the keys actually present in `body.data`.
3. The field present in `metadata` but missing (or null) in `body.data` is the one that stopped the conversion.
4. Check that field's `keyflag` in the same `metadata` block. `"X"` means it is a key field, and therefore required.

The outcome tells you what to take back to the entity definition: either the field needs a value in every record, or it should not be part of the entity or its key. Entity configuration is covered in the [SAP Data Modeler manuals](../SAP_Data_Modeler/Data_Modeler_Manuals/).

> 📌 Changing which fields are keys also changes the key schema. Review [§8](#8-schema-evolution-and-409-conflict) first if the topic already holds data.

---

## 4. `NoSuchElementException` — Missing Table Metadata

### Symptom

```
errorType=NoSuchElementException, cause=No value present
```

### Cause

The Consumer looks up the metadata entry matching the table named in the record body. If no entry matches, the lookup fails outright.

This occurs when:

- The record's `body.table` does not match any `metadata[].table` (typically a case or naming mismatch).
- A **nested table** is declared as a field of type `table`, but its own metadata block was not included in the payload.
- The payload was truncated or hand-edited during a replay.

### Diagnosis

From the failing record in `PROCESSING_ERRORS`:

1. Note the value of `body.table`.
2. Confirm a `metadata` entry exists whose `table` matches it.
3. For every field of type `table`, confirm a metadata entry exists for **that** name too.

### What the result tells you

| Finding | Meaning |
|---|---|
| No `metadata` entry matches `body.table` | The payload's table name and its metadata block disagree |
| A field of type `table` has no metadata entry of its own | The nested table's metadata block is absent from the payload |
| The payload was reconstructed by hand | Replay the original record from the error topic rather than a rebuilt one |

---

## 5. `ClassCastException` and `NumberFormatException` — Type Mismatches

### Symptoms

```
cause=Cannot convert java.lang.String to Long.
cause=For input string: "1.234,56"
cause=Cannot convert String to byte: "FF"
cause=Value out of byte range (-128 to 127): 255
cause=Cannot convert empty String to byte.
```

### Cause

The SAP type in the record's `metadata` and the actual value in `body.data` disagree. The converter accepts numeric strings, but only in a strict format.

| Declared | Accepts | Rejects |
|---|---|---|
| `N` → long | `"12345"` | `"12,345"`, `"1.2"`, `"12 "`, `""` |
| `P` → double | `"123.45"` | `"123,45"` (comma decimal separator) |
| `I` → int | `"42"` | `"42.0"`, values beyond int range |
| `X` → bytes | `"0"`–`"127"`, `"-128"`–`"-1"` | `"FF"`, `"255"`, `""` |

### Identifying which case you have

Read the `Error:` line together with the `Data:` block from `PROCESSING_ERRORS`:

| What the value looks like | What it indicates |
|---|---|
| `1.234,56` on a `P` field | Comma decimal separator; the converter expects `1234.56` |
| Letters on an `N` field | The value is not numeric despite the numeric type |
| `""` on `N`, `P`, `I`, or `X` | Empty string, which has no numeric equivalent |
| `FF` or `255` on an `X` field | Outside the single-byte decimal range the converter accepts |
| A value above 2,147,483,647 on an `I` field | Exceeds the `int` range |

The field's type comes from the SAP data dictionary definition of the source field, and is reported in the record's `metadata` block. Compare the type there against the value that failed — that pairing is what you take to whoever owns the entity definition.

---

## 6. `AvroRuntimeException` — Field Not Set

### Symptom

```
org.apache.avro.AvroRuntimeException: Field <NAME> type:STRING pos:3 not set and has no default value
```

### Cause

A **required** (key) field received no value. This is the same root cause as [§3](#3-nullpointerexception-on-a-field-that-is-empty-in-sap), surfacing at the Avro layer instead of the converter, depending on where the null lands.

### Diagnosis

The exception names the field and its position. Locate that field in the record's `metadata` block and confirm its `keyflag` is `"X"` — that is why it has no default and cannot be left unset.

From there the question is whether the field should be a key at all. If the key definition changes, review [§8](#8-schema-evolution-and-409-conflict) first.

---

## 7. Schema Registry Returns `401`

### Symptom

```
io.confluent.kafka.schemaregistry.client.rest.exceptions.RestClientException:
  Unauthorized; error code: 401
```

### Cause

Two settings are worth checking first:

| Property | Must be | Frequent mistake |
|---|---|---|
| `kafkaproperties.schemauser` | The literal string `USER_INFO` | Setting an actual username |
| `kafkaproperties.schemapassword` | `<KEY>:<SECRET>` | Supplying only the secret |

`schemauser` names the *credential source*, not a user. `schemapassword` carries **both** halves of the credential, colon-separated.

```bash
kubectl exec -n oneconnect <consumer-pod> -- env | grep -i schema
```

> 📌 Schema Registry credentials are applied when `deploymenttype` is `CLOUD`. For an on-premise registry that requires authentication, run the registry without authentication or place a proxy in front of it that supplies the credentials.

### Other Schema Registry connection errors

| Symptom | Cause | Fix |
|---|---|---|
| `Connection refused` | Registry not running or wrong URL | `kubectl get pods -n kafka \| grep schema-registry` |
| `UnknownHostException` | Unqualified service name | Use `http://schema-registry.kafka.svc.cluster.local:8081` |
| `403 Forbidden` | Credentials valid, subject permissions missing | Grant write access on the `<topic>-key` / `<topic>-value` subjects |
| Timeouts under load | Registry undersized | Raise its resources; check its logs |

To test reachability and credentials from inside the cluster, call the registry from its own pod:

```bash
kubectl exec -it $(kubectl get pod -l app=schema-registry -n kafka -o name) \
  -n kafka -- curl http://localhost:8081/subjects
```

A subject list means the registry is up. If the registry answers there but the Consumer still reports `401`, the issue is the Consumer's credentials rather than the registry itself.

---

## 8. Schema Evolution and `409 Conflict`

### Symptom

```
RestClientException: Schema being registered is incompatible with an earlier schema
  for subject "732_DEV_MARA-value"; error code: 409
```

### Cause

The schema is regenerated from the SAP metadata on every record. When the entity definition changes, the new schema must satisfy the registry's compatibility rule against the existing one (`BACKWARD` by default).

| Change | Compatible? | Why |
|---|---|---|
| Adding a non-key field | ✅ Yes | Nullable with a default |
| Removing a non-key field | ✅ Usually | Readers fall back to the default |
| Adding or removing a **key** field | ❌ No | Key fields are required, no default |
| Changing a field's type (`C` → `P`) | ❌ No | Incompatible types |
| Renaming a field | ❌ No | Equivalent to removing one and adding another |
| Changing `keyflag` on an existing field | ❌ No | Moves it between required and optional |

> 📌 The key schema contains only key fields, so changing which fields are keys rewrites it entirely. This is a common source of `409`.

### Diagnosis

In AKHQ, open **Schema Registry** and search for your topic. For the `<TOPIC>-value` subject:

- The subject view shows the **current schema** with every field and its type.
- The **Versions** view lists all registered versions, so you can compare the previous one against the latest.
- The **Config** view shows the compatibility level in force for that subject.

The difference between the last two versions is the incompatible change.

### Fixes

| Option | When to use | Cost |
|---|---|---|
| **Revert the entity change** | The change was unintended | None |
| **Use a new topic** | A deliberate redesign of the table | Destination must be pointed at the new topic; history stays on the old one |
| **Relax compatibility to `NONE`** for that subject | Controlled migration, you accept the break | Downstream consumers may fail on old records |
| **Delete the subject** | Development environments only | Destroys schema history |

Both are done in AKHQ, under **Schema Registry** → the subject:

- **Relax compatibility:** open the subject's **Config** view and change the compatibility level (for example to `NONE`).
- **Delete a subject:** use the delete action on the subject. Development environments only.

> 📌 Reserve subject deletion for development environments. The destination table's structure is derived from the schema, so re-registering an incompatible version affects the sink connector and may require rebuilding the destination table.

---

## 9. Records Serialize but the Destination Has Wrong Values

Serialization succeeded, so nothing lands in the error topic. Use the type mapping in [§1](#sap-type--avro-type) to trace the value back to the type it was mapped from.

| Symptom | What it indicates |
|---|---|
| Leading zeros missing (`000123` → `123`) | The field is typed `N`, which maps to `long` |
| Decimals truncated | The field is typed `N` or `I` rather than `P` |
| Dates arrive as `20260911` | `D` maps to `string`; no date conversion is applied in transit |
| Times arrive as `143052` | `T` maps to `string`, same as above |
| Rounding on amounts | `P` maps to `double`. For exact decimals, cast to `DECIMAL` in the destination |
| A nested table arrives as a string | The child table's metadata was absent, so the field fell back to `string` — see [§4](#4-nosuchelementexception--missing-table-metadata) |
| Deletes do not remove rows | The sink connector is not applying tombstones |

> 📌 **Tombstones.** When SAP flags a record as deleted, the Consumer emits the key with a null value, following standard Kafka delete semantics. For the destination to apply it, the sink connector needs tombstone handling enabled — check `delete.enabled` and the insert mode. See [Creating Connectors](../Smart_Gateway/Using_the_Smart_Gateway/05-Creating_Connectors.md).

---

## 10. Inspecting Schemas in AKHQ

AKHQ is deployed alongside the Kafka platform and is the place to review topics, schemas, and decoded messages. Get its address with:

```bash
kubectl get svc -n kafka | grep akhq
```

Open the external IP in a browser.

### Check the topic is receiving data

1. Open **Topics** and select your table's topic.
2. The topic list shows the record count per topic, so you can confirm it is growing.
3. Open the **Data** tab to read the most recent messages. AKHQ decodes Avro against the Schema Registry, so records display as readable fields rather than binary.

### Review the registered schemas

1. Open the **Schema Registry** section in the left-hand menu.
2. Search for your topic name. Each topic has two subjects: **`<TOPIC>-key`** and **`<TOPIC>-value`**.
3. Select a subject to view the current schema: every field, its Avro type, and whether it is nullable.
4. Key fields appear as plain types; non-key fields appear as a union with `null`.

### Compare versions after a change

1. With the subject open, use the **Versions** view to list every registered version.
2. Select two versions to compare the field lists and spot what changed.
3. The **Config** view shows the compatibility level in force for that subject.

A subject accumulating many versions indicates the entity definition is changing frequently; see [§8](#8-schema-evolution-and-409-conflict).

### Read the error topic

The same **Data** tab works for `PROCESSING_ERRORS`. Each entry shows the `Error:` line and the `Data:` payload together, which is the fastest way to pair a failure with the record that caused it.

---

## Summary

1. Read **`PROCESSING_ERRORS` first** — it carries the failing payload next to the error.
2. **Key fields are required**, so a key field with no value surfaces as a `NullPointerException` or an Avro "field not set" error.
3. `NoSuchElementException` points to **metadata missing for a table**, commonly a nested child table.
4. Type errors come from a mismatch between the field's SAP type and the value sent; the `metadata` block reports the type.
5. For Schema Registry `401`, check that `schemauser` is `USER_INFO` and `schemapassword` is `KEY:SECRET`; these apply in `CLOUD` mode.
6. `409 Conflict` indicates an incompatible change; key fields always alter the key schema.
7. Use **AKHQ** to read schemas, compare versions, and inspect decoded records.
