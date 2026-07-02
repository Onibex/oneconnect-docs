# Smart Gateway Configuration Manual

## Adding a Connector to Kafka Connect

This manual describes how to add a **specific connector** (such as Snowflake, Databricks, Hana, SQL Server, PostgreSQL, DB2, BigQuery, Amazon S3, or Microsoft Fabric) to a Kafka Connect deployment on the Smart Gateway platform.

Once a connector is created, it will start streaming SAP data from the selected Kafka topics into the target destination in near real time.

> 📋 **Prerequisites:**
>
> - The **SAP Link** must already exist.
> - **Kafka Connect** must be configured for the SAP Link. If it is not, the **"+ Add Connector"** option will be disabled. See the [Configure Kafka Connect manual](./03-Configure_Kafka_Connect.md).
> - A **Kafka Secret** must exist with the connection details of the target Kafka environment. See the [Create SAP Link manual](./02-Creating_a_SAP_Link.md) for details.
> - You must have the **connection details for the target destination** (e.g., JDBC URL, username, password) and a valid **Onibex License** if required by the connector.

---

## Step 1: Launch the Add Connector Wizard

Enter the SAP Link where Kafka Connect has been configured. In the upper-right corner, click the **"Add"** dropdown and select **"+ Add Connector"**.

> ℹ️ **Note:** The dropdown will display **"Kafka Connect Configured"** (in gray) as confirmation that Kafka Connect is already set up. The **"+ Add Connector"** option will now be active.
<img width="1338" height="459" alt="image" src="https://github.com/user-attachments/assets/a1e3b5e8-241b-4ef3-948d-a6c6b896c437" />

---

## Step 2: Wizard Overview

The Add Connector wizard consists of **5 steps**:

1. **Select Connector:** Choose the connector type and give it a name.
2. **Kafka Configuration:** Select the Kafka Secret.
3. **Connection Configuration:** Provide the destination-specific connection details.
4. **Additional Settings:** Configure advanced behavioral and performance settings.
5. **Review and Create:** Review the full configuration and confirm.

---

## Step 2.1: Select Connector

Choose the connector type that matches your target destination.

<img width="892" height="586" alt="image" src="https://github.com/user-attachments/assets/c0342aab-f265-4cbe-b239-a7ff77ba97df" />


### General Information

Under **General Information**, provide a name for the connector:

| Field | Description |
|---|---|
| **Connector Instance Name** | A unique identifier for this connector instance (e.g., `Snowflake-Prod-Sales`). |

Click **"Next"** to proceed.

---

## Step 2.2: Kafka Configuration

Select the **Kafka Secret** that contains the connection details for your Confluent cluster.

| Field | Description |
|---|---|
| **Kafka Secret** | The name of the previously created secret containing the Kafka connection details (bootstrap server, credentials, schema registry, etc.). |

> ℹ️ **Reusable Resources:** The same secret used for the Kafka Connect deployment can be reused here. Secrets are stored securely and can be reused across different components.

Click **"Next"** to proceed.
<img width="963" height="359" alt="image" src="https://github.com/user-attachments/assets/e6c7f0b7-b2d4-4ef2-bb29-0a46fabc2d2a" />

---

## Step 2.3: Connection Configuration

Provide the connection details for your target destination. **The fields shown will vary depending on the connector type selected in Step 2.1.**

The example below shows the configuration for the **Snowflake** connector. Similar fields apply to other JDBC-based connectors (Databricks, Hana, SQL Server, PostgreSQL, etc.).

### Snowflake Connection Fields

| Field | Description | Required |
|---|---|---|
| **JDBC Connection URL** | The full JDBC URL for the Snowflake instance. | ✅ |
| **Connection User** | The username for authenticating to Snowflake. |  |
| **Connection Password** | The password for the connection user. |  |
| **Onibex License** | The Onibex license key required to activate the connector. | ✅ |

### Topic Selection

You must specify which Kafka topics the connector should consume from. There are **two mutually exclusive options**:

| Option | Description |
|---|---|
| **Topics Regex** | A regular expression pattern that matches multiple topic names. Useful when you want to consume all topics for a SAP Link, excluding auxiliary topics. |
| **Topics** | A specific list of topic names to consume. |

> ⚠️ **Important:** You can only use **one** of these options at a time. If you fill in **Topics Regex**, the **Topics** field is disabled, and vice versa.

**Example Topics Regex** for a SAP Link named `TestOnibex`:

~~~
TestOnibex.*(?<!_RAWDATA)(?<!_PROCESSING_ERRORS)(?<!_OUTBOUND_ERRORS)
~~~

This pattern matches all topics starting with `TestOnibex` **except** the auxiliary topics that end in `_RAWDATA`, `_PROCESSING_ERRORS`, or `_OUTBOUND_ERRORS`.

> 💡 **Tip:** Using a Topics Regex is the recommended approach for most deployments, since it automatically includes new topics as they are created without requiring configuration changes.

Click **"Next"** to proceed.
<img width="942" height="449" alt="image" src="https://github.com/user-attachments/assets/1a1cd649-e510-4096-8ff1-5e304a7ed765" />

---

## Step 2.4: Additional Settings

Configure advanced behavioral settings and performance tuning for your connector.

| Setting | Description | Typical Value |
|---|---|---|
| **Tasks Max** | The maximum number of parallel tasks the connector will run. Increasing this improves throughput. | `1` |
| **Insert Mode** | How records are written to the destination. `UPSERT` inserts new records and updates existing ones based on the primary key. | `UPSERT` |
| **PK Mode** | How the primary key is determined. `record_key` uses the Kafka record's key. | `record_key` |
| **Auto Create** | Whether the connector should automatically create the target table if it does not exist. | `true` |
| **Auto Evolve** | Whether the connector should automatically add new columns to the target table when new fields appear in Kafka records. | `true` |
| **Delete Enabled** | Whether the connector should process tombstone records (null-value records) as deletes in the destination. | `true` |
| **Max Pool Size** | The maximum number of concurrent JDBC connections. | `1` |
| **Idle Timeout** | The maximum time (in milliseconds) a connection can remain idle before being closed. | `30000` |
| **Timeout** | The maximum time (in milliseconds) to wait for a database operation to complete. | `10000` |

> 💡 **Tip:** The default values are suitable for most scenarios. Adjust **Tasks Max** and **Max Pool Size** upward for high-volume workloads.

Click **"Next"** to proceed.
<img width="931" height="455" alt="image" src="https://github.com/user-attachments/assets/8cbc2bb0-cba4-47bb-a266-f333b6adf398" />

---

## Step 2.5: Review and Create

Review the full configuration before creating the connector. The summary displays:

| Section | Content |
|---|---|
| **Connector Type** | The selected connector (e.g., Snowflake). |
| **General Information** | The connector instance name. |
| **Kafka Secret** | The selected Kafka secret. |
| **Connection Summary** | A one-line summary of the destination connection details, including the JDBC URL, topic pattern, insert mode, and delete flag. |

If everything is correct, click **"Finish"** to create the connector. The connector will start running and streaming data from the configured Kafka topics to the target destination.

If you need to adjust anything, click **"Prev"** to return to the previous step.
<img width="907" height="504" alt="image" src="https://github.com/user-attachments/assets/a3056072-cd47-4f24-9bd4-479adf1ba224" />

---

## Step 3: Verify the Connector

After clicking **Finish**, you will be returned to the SAP Link view. Navigate to the **"Connectors"** tab to see your newly created connector listed.

From there, you can:

- Monitor the connector's status.
- Edit or delete the connector configuration.

---

## Summary

The complete workflow for adding a connector is:

1. Access the target **SAP Link** in Smart Gateway.
2. Confirm that **Kafka Connect** is configured.
3. Click **Add → + Add Connector**.
4. **Step 1:** Select the connector type and give it a name.
5. **Step 2:** Select the Kafka Secret.
6. **Step 3:** Provide the destination connection details and topic selection (Regex or list).
7. **Step 4:** Configure advanced settings (Insert Mode, Auto Create, Auto Evolve, etc.).
8. **Step 5:** Review the summary and click **Finish**.
9. Verify the connector in the **Connectors** tab.
