# Smart Gateway Configuration Manual

## Configuring Kafka Connect

This manual describes how to configure a **Kafka Connect** deployment on the Smart Gateway platform. Kafka Connect is the component that runs the connectors responsible for delivering SAP data from Kafka topics to downstream destinations such as Databricks, Snowflake, or ClickHouse.

> 📋 **Prerequisites:**
>
> - A **SAP Link** must already exist (in the example: `TestOnibex`).
> - A **Kafka Secret** must already exist with the connection details of the target Kafka environment. 

---

## Step 1: Access the SAP Link

Enter the Smart Gateway platform and navigate to the SAP Link where you want to configure Kafka Connect (in the example: **`TestOnibex`**).

Inside the SAP Link view, you will see:

- The **SAP Link endpoint URL** (used to receive data from SAP).
- A **Dashboard** showing metrics such as Requests, Processed Data, API Average Latency, and API Errors.
- Two tabs: **SAP Link** and **Connectors**.

---

## Step 2: Launch the Kafka Connect Wizard

Click the **"Add"** dropdown button in the upper-right corner and select **"+ Configure Kafka Connect"**.
<img width="1175" height="615" alt="image" src="https://github.com/user-attachments/assets/f8e665f4-d55a-4868-a323-57cb101421e2" />

The other options in the dropdown are:

| Option | Purpose |
|---|---|
| **+ Add Traceability** | Adds a traceability configuration to the SAP Link. |
| **+ Add Ask SAP** | Adds an Agentic Semantic Knowledge (ASK) configuration. |
| **+ Configure Kafka Connect** | Configures the Kafka Connect deployment (this manual). |
| **+ Add Connector** | Adds a specific connector (available after Kafka Connect is configured). |

> ℹ️ **Note:** The **"+ Add Connector"** option becomes available only after Kafka Connect has been successfully configured for the SAP Link.

---

## Step 3: Kafka Connect Wizard Overview

The Kafka Connect configuration wizard consists of **3 steps**:

1. **Connection Details:** Select the Kafka Secret.
2. **Advanced Settings:** Tune replicas and storage replication.
3. **Summary:** Review and confirm the configuration.

---

## Step 3.1: Connection Details

In this step, select the **Kafka Secret** that Kafka Connect will use to reach the Kafka cluster.

| Field | Description |
|---|---|
| **Kafka Secret** | The name of the previously created secret containing the Kafka connection details (bootstrap server, credentials, schema registry, etc.). |

> ℹ️ **Reusable Resources:** Secrets are stored securely and can be reused across different SAP Links, connectors, and Kafka Connect deployments. This avoids having to re-enter connection parameters each time.

Click **"Next"** to proceed.

<img width="722" height="329" alt="image" src="https://github.com/user-attachments/assets/f0f0a776-14b7-4af6-b598-6a13f7dd79bf" />

---

## Step 3.2: Advanced Settings

In this step, tune the deployment replicas and cluster internal replication settings.

| Field | Description | Default |
|---|---|---|
| **Replicas** | The number of Kafka Connect worker instances to run. Increasing replicas improves throughput and availability. | `1` |
| **Storage Replication Factor** | The replication factor for the internal topics used by Kafka Connect (config, offsets, and status topics). Should not exceed the number of brokers in the Kafka cluster. | `1` |

> 💡 **Tip:** For a Proof of Concept or development environment, the default values (`1` and `1`) are sufficient. For production deployments, consider increasing the **Replicas** to at least `2` for high availability, and set **Storage Replication Factor** to `3` if your Kafka cluster has at least 3 brokers.

Click **"Next"** to proceed, or **"Back"** to modify the previous step.

<img width="774" height="370" alt="image" src="https://github.com/user-attachments/assets/623cb7ab-3fa7-4696-94c6-8f3013708c1b" />

---

## Step 3.3: Summary

Review your configuration before creating the Kafka Connect deployment. The summary displays:

| Field | Description |
|---|---|
| **SAP Link** | The SAP Link this Kafka Connect will belong to. |
| **Kafka Secret** | The selected secret containing the Kafka connection details. |
| **Replicas** | The number of Kafka Connect worker instances configured. |
| **Storage Replication Factor** | The replication factor for the internal topics. |

If everything is correct, click **"Finish"** to create the Kafka Connect deployment.

If you need to adjust anything, click **"Back"** to return to the previous step.
<img width="764" height="426" alt="image" src="https://github.com/user-attachments/assets/fcb6ec77-cd24-41b3-88c0-02e41a4405b9" />

---

## Step 4: Verify the Deployment

After clicking **Finish**, the Kafka Connect deployment will be provisioned. You will be returned to the SAP Link view, where you can now:

- See Kafka Connect listed as a configured component.
- Use the **"+ Add Connector"** option to start adding specific connectors (Databricks, Snowflake, ClickHouse, etc.).

> ℹ️ For details on adding specific connectors after Kafka Connect is configured, see the next manual: **Adding Connectors to Kafka Connect**.

---

## Summary

The complete workflow for configuring Kafka Connect is:

1. Access the target **SAP Link** in Smart Gateway.
2. Click **Add → + Configure Kafka Connect**.
3. **Step 1:** Select the previously created **Kafka Secret**.
4. **Step 2:** Set the number of **Replicas** and **Storage Replication Factor**.
5. **Step 3:** Review the summary and click **Finish**.
6. Verify the deployment and proceed to add specific connectors.
