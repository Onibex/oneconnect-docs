# Smart Gateway Configuration Manual

## Generating an Ask SAP Endpoint

The **Ask SAP** feature allows you to integrate your SAP Link with an Ask SAP service. Once configured, a dedicated endpoint URL is generated and can be used for Ask SAP integration within your environment.

> 📋 **Prerequisites:**
>
> - An existing **SAP Link** created in your OneConnect environment. If you don't have one, see the [Creating a SAP Link manual](./02-Creating_a_SAP_Link.md).
> - At least one **Kafka Secret** created in your account. Kafka Secrets store the connection parameters needed for the Ask SAP deployment. See the [Creating a SAP Link manual](./02-Creating_a_SAP_Link.md) (Stage 2) for details on how to create secrets.
> - The SAP Link must be accessible from your user session.

---

## Step 1: Navigate to Your SAP Link

1. Log in to the OneConnect Cloud platform.
2. From the sidebar, locate your SAP Link under the **SAP Links** section.
3. Click on the SAP Link name to open its dashboard.

---

## Step 2: Open the Add Menu

In the SAP Link dashboard, locate the **"Add"** button in the upper-right area of the header and click it.

A dropdown menu will appear with the available add-on options.

---

## Step 3: Select "Add Ask Sap"

From the dropdown menu, select the **"Add Ask Sap"** option.

> ℹ️ **Note:** If the option appears as **"Ask Sap Configured"** and is disabled, it means this SAP Link already has an Ask SAP endpoint configured. Each SAP Link can only have one Ask SAP endpoint.

---

## Step 4: Configure the Ask SAP Wizard

The Ask SAP configuration wizard consists of **3 stages**:

### Stage 1: Create Ask Sap Link

Define the basic details for your Ask SAP connection:

| Field | Description |
|---|---|
| **Ask Sap Name (Topic Prefix)** | The name used to identify the Ask SAP connection. Also serves as the prefix for every Kafka topic sent through it. |
| **SAP Environment** | Select the SAP environment from the dropdown menu. |
| **Username** | The username for the Ask SAP connection. |
| **Password** | A password for the Ask SAP connection. Use the **"Generate"** button to auto-generate a secure password, or type one manually. Click the **download icon** 📥 to save the password as a `.txt` file. |

> ⚠️ **Important:** Save the password carefully. It is recommended to download the `.txt` file using the download button next to the password field.

Click **"Next"** to proceed.

### Stage 2: Connection Details

Configure the Kafka connection parameters:

| Field | Description |
|---|---|
| **Kafka Secret** | Select an existing Kafka Secret from the dropdown. This secret contains the Kafka connection parameters (bootstrap server, schema registry, API keys, etc.). |
| **Partitions** | The number of partitions for each Kafka topic created by the Ask SAP connection. Default: `1`. |
| **Replicas** | The replication factor for each Kafka topic. Default: `1`. |
| **Retention (ms)** | How long messages are retained in each topic (in milliseconds). Default: `604800000` (7 days). |

Click **"Next"** to proceed.

### Stage 3: Advanced Settings

Allocate CPU and memory resources for the Ask SAP container:

| Setting | Description | Default |
|---|---|---|
| **CPU Requests** | The guaranteed CPU allocation. | `100m` |
| **Memory Requests** | The guaranteed memory allocation. | `256Mi` |
| **CPU Limits** | The maximum CPU the container can consume. | `500m` |
| **Memory Limits** | The maximum memory the container can consume. | `512Mi` |

Click **"Create Ask Sap"** to finalize the configuration.

---

## Step 5: Confirmation

Once the configuration is created successfully, you will be redirected to the SAP Link dashboard. A success notification will confirm the Ask SAP link was created.

---

## Step 6: Retrieve the Ask SAP URL

After creating the Ask SAP endpoint:

1. Go to your SAP Link dashboard.
2. You will see a new **"Ask Sap"** tab next to the **"SAP Link"** tab.
3. Click the **"Ask Sap"** tab to view the generated endpoint URL.
4. Use the **copy icon** 📋 to copy the URL to your clipboard.

From this tab, you can also:

- Click **"View Config"** to see the full configuration details of your Ask SAP setup.
- Click **"Delete Ask Sap"** to remove the Ask SAP endpoint if it is no longer needed.

> ⚠️ **Warning:** Deleting an Ask SAP endpoint is irreversible. Make sure you no longer need the endpoint before deleting it.

---

## Summary

The complete workflow for generating an Ask SAP endpoint is:

1. Access your SAP Link dashboard.
2. Click **Add → Add Ask Sap**.
3. **Stage 1:** Define the name, SAP environment, username, and password.
4. **Stage 2:** Select a Kafka Secret and configure topic settings (partitions, replicas, retention).
5. **Stage 3:** Allocate CPU and memory resources for the container.
6. Click **Create Ask Sap**.
7. Copy the generated **Ask SAP URL** from the Ask Sap tab.