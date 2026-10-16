# Avro Serialization & Schema Registry

The Consumer reads the raw JSON that the Producer wrote, builds an Avro schema from the SAP metadata, converts every field, and writes an Avro key/value pair per row to the table's topic. Most data-quality questions in OneConnect are answered at this layer.

Typical indicators for this layer: the raw topic keeps growing, the per-table topics do not, and records appear in `PROCESSING_ERRORS`.

---

## 1. How the Conversion Works

Knowing the sequence makes every case in this guide easier to read.

1. A message arrives with three blocks: `properties` (`messageType`, `delete`, and for `kdoc` messages `entity`), `metadata` (the SAP field definitions per table), and `body` (`table` and the `data` rows).
2. The Consumer builds **two** Avro schemas for the table: a **key** schema from the fields flagged as key (`keyflag = "X"`), and a **value** schema from all fields.
3. Every row is converted to those schemas. **A message is processed as a unit**: all rows are converted first, and only then are they sent. If one row cannot be converted, the whole message goes to the error topic with the reason, and you correct and resend it as one piece.
4. Each key/value pair is serialized and sent. The schemas are registered in the Schema Registry on first use. When `properties.delete` is `X`, the value is sent as `null` (a tombstone) and only the key is converted.

### SAP type → Avro type

| SAP type | Avro type |
|---|---|
| `C` (character) | `string` |
| `D` (date) | `string` |
| `T` (time) | `string` |
| `P` (packed decimal) | `double` |
| `N` (numeric text) | `long` |
| `I`, `s` (integer) | `int` |
| `X` (hex/raw) | `bytes` |
| `table` (nested) | `array` of records |
| anything else | `string` |

### What every row needs

| Rule | Why |
|---|---|
| **Every field in the metadata has a value in every row** — key fields and non-key fields alike | Each field is converted to its Avro type; a field that is absent from the row or sent as JSON `null` cannot be converted ([§3](#3-a-field-without-a-value-in-a-row)) |
| **Every key in a row has a matching field in the metadata** | The type of each value is looked up in the metadata ([§6](#6-error-processing-oneconnecttypefinder--field-not-found-in-the-metadata)) |
| **Field names are lowercase**, in `metadata[].fieldname` and in the `data` rows alike | Names are matched exactly, and key fields are looked up in lowercase |
| Numeric values use a dot as decimal separator and no thousands separator | `N`, `P`, and `I` values are parsed strictly ([§5](#5-classcastexception-and-numberformatexception--type-mismatches)) |

### Key fields versus non-key fields

| | Key fields (`keyflag = "X"`) | Non-key fields |
|---|---|---|
| Avro declaration | Required (no default) | Optional (union with `null`, default `null`) |
| Appears in | Key schema **and** value schema | Value schema only |
| Value needed in every row | Yes | Yes |

> 📌 The key becomes the primary key in the destination. Choose key fields that always carry a value and that identify the row uniquely.

### Topic and subject naming

| Item | Pattern |
|---|---|
| Output topic | `[<CLIENT>_<ENV>_]<TABLE>` (the value of `body.table`) — prefix applied only when `useprefix = ENABLE` |
| Output topic (`kdoc` messages) | `[<CLIENT>_<ENV>_]<ENTITY>` (the value of `properties.entity`) |
| Error topic | `[<CLIENT>_<ENV>_]PROCESSING_ERRORS` |
| Schema subjects | `<topic>-key` and `<topic>-value` |

---

## 2. Start Here: Read the Error Topic

The error topic keeps the record and the reason together, which makes it the fastest place to start.

In AKHQ, open **Topics** → `PROCESSING_ERRORS` → the **Data** tab. Get the AKHQ address with:

```bash
kubectl get svc -n kafka | grep akhq
```

Remember the prefix: if `useprefix = ENABLE`, the topic is `<CLIENT>_<ENV>_PROCESSING_ERRORS`.

### What an entry contains

Each entry has no key, and its value is a text string with three lines:

```
Failed to send message
Error: <the reason>
Data: <the record>
```

> ℹ️ The value is stored as an Avro-encoded string (subject `PROCESSING_ERRORS-value`, with the prefix when it applies). AKHQ decodes it for you; a plain console consumer shows a few leading binary bytes before the text.

### Which stage the entry comes from

The `Data:` line tells you at which stage the record stopped, and therefore how to recover it:

| `Data:` contains | Stage | Matching Consumer log line | How to recover |
|---|---|---|---|
| The full SAP message as JSON (`{"oneconnect":{...}}`) | **Processing** — reading the message, building the schema, or converting a row | `Error processing record, will send to error topic, errorType=<X>, cause=<reason>` | Fix the cause (data, entity, or metadata), then resend from SAP. Nothing from that message was sent, so the resend is complete |
| One row in Avro JSON form (`{"matnr": "...", ...}`), or `null` for a delete | **Sending** — schema registration or delivery to the table's topic | `-->Error Sending Record to topic=<t>, key=<k>, value=<v>, errorType=<X>, cause=<reason>` | Fix the cause (registry, topic, broker), then resend the affected data from SAP. The log line also shows the key |

> ℹ️ Entries with `Error: null` and an empty `Data:` mark a Consumer restart; no action needed.

The `Error:` line carries the same text as `cause=` in the log. Match it against the sections below. To see the log lines:

```bash
kubectl logs -n oneconnect <consumer-pod> --tail=500 | grep -E 'Error processing record|Error Sending Record'
```

The full stack trace follows each log line; for Schema Registry errors, the line that starts with `Caused by:` names the registry response.

---

## 3. A Field Without a Value in a Row

### Symptom

A field defined in the metadata has no value in one or more rows. The Consumer log and the `Error:` line in `PROCESSING_ERRORS` show:

```
errorType=NullPointerException, cause=Cannot invoke "Object.getClass()" because "object" is null
```

`Data:` holds the full SAP message.

### Cause

A field defined in the `metadata` block has no value in a row: the key is absent from the row, its value is JSON `null`, or the key is present with a different letter case than the metadata field name. Each field is converted to its Avro type, so every field of the entity needs a value in every row — this applies to non-key fields as much as to key fields. Because the message is processed as a unit, one row without a value sends the whole message to the error topic.

### Confirm

Pull the entry from `PROCESSING_ERRORS` and compare:

1. List the field names in the `metadata` block for the table in `body.table`.
2. Go through the rows in `body.data` and look for a field that is missing, `null`, or spelled with different case (for example `MATNR` in the metadata and `matnr` in the row).
3. Check that both the metadata names and the row keys are lowercase.

### Fix

1. **Value missing or `null`:** in SAP, make sure the extraction fills every field of the entity in every row — an empty string `""` for character fields (`C`, `D`, `T`), and a number such as `"0"` for numeric fields (`N`, `P`, `I`). If a field has nothing to contribute, remove it from the entity. Entity configuration is covered in the [SAP Data Modeler manuals](../SAP_Data_Modeler/Data_Modeler_Manuals/).
2. **Case differs:** send field names in lowercase in both `metadata[].fieldname` and the row keys. This is what the standard OneConnect extraction does; check any custom code or hand-built payload that produced the message.
3. Resend the data from SAP.

> 📌 Changing which fields are keys also changes the key schema. Review [§8](#8-schema-evolution-and-409-conflict) first if the topic already holds data.

---

## 4. `NoSuchElementException` — Missing Table Metadata

### Symptom

```
errorType=NoSuchElementException, cause=No value present
```

### Cause

The Consumer looks for the `metadata` entry whose `table` equals `body.table` (the comparison ignores letter case). The message stops here when:

- No `metadata[].table` has the same name as `body.table`.
- The `metadata` list is empty.
- `body.table` is absent from the message.

### Confirm

From the entry in `PROCESSING_ERRORS`:

1. Note the value of `body.table`.
2. List the `table` values in the `metadata` block.
3. Confirm one of them is the same name.

### Fix

1. Make sure the message carries a `metadata` block for the table named in `body.table`, with the same name.
2. If the message was assembled or edited outside the standard extraction, rebuild it from SAP rather than by hand.
3. Resend the data from SAP.

---

## 5. `ClassCastException` and `NumberFormatException` — Type Mismatches

### Symptoms

```
errorType=NumberFormatException, cause=For input string: "1.234,56"
errorType=NumberFormatException, cause=For input string: ""
errorType=NumberFormatException, cause=empty String
errorType=NumberFormatException, cause=Cannot convert String to byte: "FF"
errorType=NumberFormatException, cause=Cannot convert empty String to byte.
errorType=ClassCastException, cause=Cannot convert java.lang.Boolean to Long.
```

### Cause

The SAP type in the `metadata` block and the value in the row disagree. Numeric values sent as strings are parsed, and the format accepted is strict:

| Declared | Accepts | Does not accept |
|---|---|---|
| `N` → long | `"12345"` | `"12,345"`, `"1.2"`, `"12 "`, `"12-"`, `""` |
| `P` → double | `"123.45"` | `"1.234,56"` (comma decimal separator), `"12-"` (trailing sign), `""` |
| `I` → int | `"42"` | `"42.0"`, `""`, values beyond the int range |
| `N`, `P`, `I` | Strings and JSON numbers | JSON `true`/`false` (`ClassCastException`) |

### Confirm

Read the `Error:` line together with the `Data:` block from `PROCESSING_ERRORS`:

| What the value looks like | What it indicates |
|---|---|
| `1.234,56` on a `P` field | Comma decimal separator; the Consumer expects `1234.56` |
| `12-` on a `P` or `N` field | SAP trailing minus sign; the Consumer expects `-12` |
| Letters on an `N` field | The value is not numeric despite the numeric type |
| `""` on `N`, `P`, or `I` | Empty string, which has no numeric equivalent |
| `true`/`false` on a numeric field | A boolean value on a numeric field |
| A value above 2,147,483,647 on an `I` field | Exceeds the `int` range |

The field's type comes from the SAP data dictionary definition of the source field and is reported in the `metadata` block. That pairing — declared type and the value that was sent — is what you take to whoever owns the entity definition.

### Fix

1. Send numeric values in plain format: dot as decimal separator, no thousands separator, leading minus sign, and `"0"` instead of an empty value.
2. If the field holds text by nature (for example, a code with letters), change its type in the source so it maps to `string`.
3. Resend the data from SAP.

### RAW fields (type `X`)

The messages `Cannot convert String to byte: "<value>"` and `Cannot convert empty String to byte.` refer to a field of type `X` (RAW), such as a GUID (`RAW16`). Recommended modeling:

1. If the RAW value is not needed in the destination, leave it out of the entity.
2. If it is needed, expose it to OneConnect as a character field (for example, its hexadecimal text form), so it maps to `string` and arrives exactly as SAP shows it.

---

## 6. `Error processing OneConnect.typeFinder` — Field Not Found in the Metadata

### Symptom

```
errorType=RuntimeException, cause=Error processing OneConnect.typeFinder dataName: <field>. Invalid or name not found in BodyMetadata
```

`<field>` is the name of the key found in the row.

### Cause

Every key in a row is looked up in the table's `metadata` block to find its SAP type. The named field is present in the data but has no field definition in the metadata. Data and metadata describe different field lists — typically because the entity changed on one side, or the payload was assembled outside the standard extraction.

### Confirm

1. In the `PROCESSING_ERRORS` entry, find `<field>` in a row of `body.data`.
2. Look for the same name in the `metadata` block for `body.table`.
3. Use the table below to read the result.

| Finding | Meaning |
|---|---|
| The name is not in the metadata at all | The row carries a field the entity does not define |
| The name contains `/` (for example `/bic/zfield`) | Field names are expected to use letters, digits, and underscores only |
| The name is in the metadata of a different table | The row belongs to another table, or a nested table's fields are at the wrong level |

### Fix

1. Align the entity so that the data rows and the `metadata` block list the same fields.
2. For fields whose names contain `/`, expose them to OneConnect under a name made of letters, digits, and underscores (this is also what Avro field names require).
3. Resend the data from SAP.

---

## 7. Schema Registry Returns `401`

### Symptom

In the Consumer log:

```
-->Error Sending Record to topic=<topic>, key=..., value=..., errorType=SerializationException, cause=Error registering Avro schema{"type":"record",...}
...
Caused by: ...RestClientException: Unauthorized; error code: 401
```

The `cause=` text starts with `Error registering Avro schema`, followed by the schema; the registry's answer is in the `Caused by:` line of the stack trace.

> ℹ️ During a Schema Registry outage, details are in the Consumer log; resend the affected period from SAP.

### Cause

Two settings are worth checking first:

| Property | Must be | Frequent mistake |
|---|---|---|
| `kafkaproperties.schemauser` | The literal string `USER_INFO` | Setting an actual username |
| `kafkaproperties.schemapassword` | `<KEY>:<SECRET>` | Supplying only the secret |

`schemauser` names the *credential source*, not a user. `schemapassword` carries **both** halves of the credential, colon-separated.

### Confirm

```bash
kubectl exec -n oneconnect <consumer-pod> -- env | grep -i schema
```

> 📌 Schema Registry credentials are applied when `deploymenttype` is `CLOUD`.

### Fix

1. Set `schemauser` to `USER_INFO` and `schemapassword` to `<KEY>:<SECRET>`.
2. Restart the Consumer pod so it picks up the values.
3. Resend from SAP the data reported in the log during the outage.

### Other Schema Registry connection errors

| Symptom (Consumer log) | Cause | Fix |
|---|---|---|
| `errorType=SerializationException, cause=Error serializing Avro message`, with `Caused by: java.net.ConnectException` | Registry not running, or wrong port | `kubectl get pods -n kafka \| grep schema-registry`; check `kafkaproperties.schemaregistryurl` |
| `errorType=SerializationException, cause=Error serializing Avro message`, with `Caused by: java.net.UnknownHostException` | Host name cannot be resolved | Use the fully qualified name, e.g. `http://schema-registry.kafka.svc.cluster.local:8081` |
| `cause=Error registering Avro schema...` with `403` in the `Caused by:` line | Credentials valid, subject permissions missing | Grant write access on the `<topic>-key` / `<topic>-value` subjects |

To test reachability from inside the cluster, call the registry from its own pod:

```bash
kubectl exec -it $(kubectl get pod -l app=schema-registry -n kafka -o name) \
  -n kafka -- curl http://localhost:8081/subjects
```

A subject list means the registry is up. If the registry answers there but the Consumer still reports `401`, the credentials configured in the Consumer are the place to look.

---

## 8. Schema Evolution and `409 Conflict`

### Symptom

In the Consumer log:

```
-->Error Sending Record to topic=732_DEV_MARA, ..., errorType=SerializationException, cause=Error registering Avro schema{"type":"record",...}
...
Caused by: ...RestClientException: Schema being registered is incompatible with an earlier schema for subject "732_DEV_MARA-value"; error code: 409
```

In `PROCESSING_ERRORS`, the `Error:` line starts with `Error registering Avro schema` and `Data:` holds the row.

### Cause

The schema is generated from the SAP metadata of each message. When the entity definition changes, the new schema must satisfy the subject's compatibility rule against the registered one (`BACKWARD` is the Schema Registry default). Key fields are required in both the key and the value schema; non-key fields are optional with a `null` default.

| Change | Compatible under `BACKWARD`? | Why |
|---|---|---|
| Adding a non-key field | ✅ Yes | Optional, with a default |
| Removing a non-key field | ✅ Yes | Removing a field is always readable |
| Renaming a non-key field | ✅ Yes | Treated as removing one optional field and adding another |
| Adding a **key** field, or making an existing field a key | ❌ No | Key fields are required and have no default |
| Renaming a **key** field | ❌ No | The new name is a required field without a default |
| Removing a key field, or turning a key field into a non-key field | ✅ Yes for the registry | The destination primary key changes too; plan it as a new table |
| `I` → `N` (int → long) or `N` → `P` (long → double) | ✅ Yes | Avro type promotion |
| Other type changes (for example `C` → `P`) | ❌ No | Incompatible types |

> 📌 The key schema contains only key fields, so any change to the set of keys rewrites it. This is a common source of `409`.

### Confirm

In AKHQ, open **Schema Registry** and search for your topic. For the subject named in the message:

- The subject view shows the **current schema** with every field and its type.
- The **Versions** view lists all registered versions, so you can compare the previous one against the new metadata.
- The **Config** view shows the compatibility level in force for that subject.

### Fix

| Option | When to use | Cost |
|---|---|---|
| **Revert the entity change** | The change was unintended | None |
| **Use a new topic** | A deliberate redesign of the table | Destination must be pointed at the new topic; history stays on the old one |
| **Relax compatibility to `NONE`** for that subject | Controlled migration, you accept the break | Downstream consumers may fail on old records |
| **Delete the subject** | Development environments only | Removes the schema history |

Both registry actions are done in AKHQ, under **Schema Registry** → the subject:

- **Relax compatibility:** open the subject's **Config** view and change the compatibility level (for example to `NONE`).
- **Delete a subject:** use the delete action on the subject. Development environments only.

After the fix, resend from SAP the data reported during the conflict.

> 📌 Reserve subject deletion for development environments. The destination table's structure is derived from the schema, so registering an incompatible version affects the sink connector and may require rebuilding the destination table.

---

## 9. Records Serialize but the Destination Has Wrong Values

Serialization succeeded, so nothing lands in the error topic. Use the type mapping in [§1](#sap-type--avro-type) to trace the value back to the type it was mapped from.

| Symptom | What it indicates | Recommended practice |
|---|---|---|
| Leading zeros missing (`000123` → `123`) | The field is typed `N`, which maps to `long` | Type the field as character if the zeros matter |
| Decimals truncated | The field is typed `N` or `I` rather than `P` | Use `P` for amounts and quantities |
| Dates arrive as `20260911` | `D` maps to `string`; no date conversion is applied in transit | Convert in the destination |
| Times arrive as `143052` | `T` maps to `string`, same as above | Convert in the destination |
| Amounts show a different number of decimal places (`10.50` → `10.5`) | `P` maps to `double` | Format the value, or cast it to `DECIMAL`, in the destination |
| A character field arrives as `123.0` | The value was sent as a JSON number rather than a JSON string | Send SAP values as JSON strings |
| Deletes do not remove rows | The sink connector is not applying tombstones | See the note below |

> 📌 **Tombstones.** When SAP flags a message as a delete (`properties.delete = "X"`), the Consumer emits the key with a null value, following standard Kafka delete semantics. For the destination to apply it, the sink connector needs tombstone handling enabled — check `delete.enabled` and the insert mode. See [Creating Connectors](../Smart_Gateway/Using_the_Smart_Gateway/05-Creating_Connectors.md).

---

## 10. Missing Message Fields: `delete`, `messageType`, `body`, `keyflag`

### Symptom

A message arrives without one of its required elements, and the Consumer log shows `errorType=NullPointerException` with a `cause=` that names the element. Elements to check:

| Element | Where |
|---|---|
| `properties` | `oneconnect.properties` |
| `delete` | `oneconnect.properties.delete` |
| `messageType` | `oneconnect.properties.messageType` |
| `body` | `oneconnect.body` |
| `data` | `oneconnect.body.data` |
| `metadata` | `oneconnect.metadata` |
| `keyflag` | Each field of the `metadata` block |

### Cause

The Consumer reads these elements from every message, so each one needs to be present:

- `properties.messageType` decides the topic name (`kdoc` uses `properties.entity`).
- `properties.delete` is `"X"` for a delete and `""` otherwise.
- `body.table` and `body.data` carry the rows (an empty `data` list is valid and produces no records).
- Each `metadata` field carries `fieldname`, `type`, and `keyflag` (`"X"` for key fields, `""` otherwise).

### Confirm

Open the entry in `PROCESSING_ERRORS` and check the `Data:` JSON for the element named in the table above.

### Fix

1. Send the complete message structure from SAP, with `delete` and `keyflag` always present (as `""` when not set).
2. If the message was built or modified outside the standard extraction, regenerate it from SAP.
3. Resend the data from SAP.

---

## 11. Nested Tables: `AvroRuntimeException: Not a record`

### Symptom

```
errorType=AvroRuntimeException, cause=Not a record: {"type":"array","items":{...}}
errorType=AvroRuntimeException, cause=Not a record: "string"
```

### Cause

- `Not a record: {"type":"array",...}` — a field of type `table` arrived as `{"table": ..., "data": [...]}`, and the Consumer could not match it to a metadata block. The nested `table` value must be exactly the same name (including case) as the `metadata[].table` entry for that child table, and that entry must be part of the message.
- `Not a record: "string"` (or another simple type) — a field defined as a simple type in the metadata received a JSON object in the row.

### Confirm

In the `Data:` JSON of the `PROCESSING_ERRORS` entry:

1. For each field of type `table`, find a `metadata` entry with the same `table` name.
2. Check that the nested `"table"` value in the row is spelled exactly like that entry.
3. Check that fields of simple types (`C`, `N`, `P`, …) carry simple values, not objects.

### Fix

1. Include the child table's `metadata` block in the message, with the same name used in the nested data.
2. Align the entity so nested structures are declared as `table` fields.
3. Resend the data from SAP.

---

## 12. `InvalidTopicException` — Table Names with Slashes (`/BIC/` Tables)

### Symptom

One log line per row:

```
-->Error Sending Record to topic=732_DEV_/BIC/AZSALES00, key=..., value=..., errorType=InvalidTopicException, cause=...
```

`PROCESSING_ERRORS` receives one entry per row, with the row in `Data:`.

### Cause

The output topic is named after `body.table` (or `properties.entity` for `kdoc` messages). Kafka topic names accept letters, digits, `.`, `_`, and `-` only, so a name such as `/BIC/AZSALES00` cannot become a topic.

### Confirm

Check the `topic=` value in the log line, or `body.table` / `properties.entity` in the message.

### Fix

1. Publish the data from SAP under a name made of letters, digits, `.`, `_`, and `-` — for example, an entity based on a view or structure with such a name, or a `kdoc` message whose `entity` defines the topic name.
2. Resend the data from SAP.

---

## 13. Field Names Rejected by Avro: `SchemaParseException` and `Duplicate field`

### Symptom

```
errorType=SchemaParseException, cause=Illegal initial character: <name>
errorType=SchemaParseException, cause=Illegal character in: <name>
errorType=AvroRuntimeException, cause=Duplicate field <name> in record ...
```

### Cause

Avro field names start with a letter or underscore and contain only letters, digits, and underscores. A name that starts with a digit, or contains `-`, `.`, a space, or `$`, is not accepted. `Duplicate field` means the same name appears twice in one table's `metadata` block.

### Confirm

Find `<name>` in the `metadata` block of the `PROCESSING_ERRORS` entry.

### Fix

1. Expose the field to OneConnect under a name that follows the Avro rules, and keep each name unique within a table.
2. Resend the data from SAP.

---

## 14. Inspecting Schemas in AKHQ

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

The same **Data** tab works for `PROCESSING_ERRORS`. Each entry shows the `Error:` line and the `Data:` payload together; see [§2](#2-start-here-read-the-error-topic) for how to read them.

---

## Summary

1. Read **`PROCESSING_ERRORS` first** — it keeps the record and the reason. A full SAP message in `Data:` means the processing stage; a single row means the sending stage.
2. A message is processed **as a unit**: every field in the metadata needs a value in every row, and names are lowercase in both metadata and data.
3. `Error processing OneConnect.typeFinder dataName: X` means the row has a field the metadata does not define.
4. `NoSuchElementException` means no `metadata` block matches `body.table`; send `delete`, `messageType`, `body`, and `keyflag` in every message.
5. Type errors come from a mismatch between the field's SAP type and the value sent; expose RAW values needed in the destination as character fields.
6. Table or entity names become topic names and accept letters, digits, `.`, `_`, `-` only.
7. Schema Registry errors surface as `Error registering Avro schema`; the `Caused by:` line shows `401` (check `schemauser` = `USER_INFO`, `schemapassword` = `KEY:SECRET`, `CLOUD` mode) or `409` (incompatible entity change).
8. Use **AKHQ** to read schemas, compare versions, and inspect decoded records.
