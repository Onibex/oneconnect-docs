# Smart Gateway Configuration Manual

## Creating SAP Connectors on the Smart Gateway Platform

> 📌 **Note:** Each account and user has the ability to create SAP Connectors in their designated environment. The following steps show how a SAP Connector is created from an **administrator account**. Non-administrator users can also create SAP Connectors in their environment following these same steps.

---

## Step 1: Access the Smart Gateway Platform

Enter the Smart Gateway platform with your **username** and **password**.
<img width="1048" height="477" alt="image" src="https://github.com/user-attachments/assets/ca4c9ecc-da2b-4715-80bb-faca40f7f7a8" />

---

## Step 2: Navigate to the Users Section

Go to the **"USERS"** section, accessible from the menu button in the upper-left corner (the icon with three horizontal lines).

Click on the name of the user you want to assign the new SAP Connector to, or click the right-pointing arrow in the **"Actions"** column.
<img width="1308" height="305" alt="image" src="https://github.com/user-attachments/assets/ff0b56cf-e083-493c-b44f-3f60fb24c3b7" />

---

## Step 3: Add a New SAP Connector

Select the **"SAP Links"** option and click the **"Add SAP Link"** button.
<img width="1363" height="432" alt="image" src="https://github.com/user-attachments/assets/1b4b4b80-b9af-436f-9ffb-cdca97d69422" />

The process of creating an SAP Connector consists of **4 stages**:

1. Definition of the SAP Connector.
2. Connection Details.
3. Topic Settings.
4. Advanced Settings.

---

## Stage 1: SAP Connector Definition

Fill in the following information:

| Field | Description |
|---|---|
| **SAP Connector Name (Topic Prefix)** | The name used to identify the connector. Also serves as the prefix for every topic sent through it. |
| **SAP Environment** | The SAP environment to which this connector will be added. |
| **SAP Connector Username** | The username used in the SAP RFC configuration. |
| **SAP Connector Password** | The password for the previously created user. Used in the SAP RFC configuration. |

> ⚠️ **Important:** Save the username and password information carefully, as these **cannot be changed later**. Neither you nor the Onibex team will be able to access the password once it has been created.
>
> There is an option to **download the password in `.txt` format**. We strongly recommend doing so to ensure you have a secure copy.

After adding the required information, click **"Next"**.
<img width="890" height="412" alt="image" src="https://github.com/user-attachments/assets/e199a301-1c83-42fe-b83f-541015447e5e" />

---

## Stage 2: Connection Details

In the **Connection Details** section, you will configure how the SAP Connector communicates with your Kafka environment.

### Creating a Secret

Before entering the Kafka connection parameters, you must create a **Secret** that will store the connection details.

> ℹ️ **Reusable Resources:** Connection details are stored as **secure secrets**. Once created, a secret can be **reused across different workspaces or components**, for example when creating a new SAP Link or a new SAP Connector, without having to re-enter the connection parameters.

Fill in the following fields under **New Secret Details**:

| Field | Description |
|---|---|
| **Secret Name** | A unique identifier for the secret. Only **lowercase letters, numbers, and dashes** are allowed. Example: `dev-kafka`. |
| **Secret Description** *(optional)* | A short description to help identify the purpose of the secret. Example: `Development Connection`. |

Once the secret is created, proceed to configure the connection parameters below. Select the deployment type: **Confluent Cloud** or **Confluent Platform**.
<img width="813" height="436" alt="image" src="https://github.com/user-attachments/assets/c2c0b4a3-b978-4973-9013-1f654f6b3321" />

### Option A: Confluent Cloud

If you select **Confluent Cloud**, enter the following information:

| Field | Description |
|---|---|
| **Bootstrap Server** | The Kafka bootstrap server address. |
| **Schema Registry URL** | The URL of the Confluent Schema Registry. |
| **Cluster ID** | The unique identifier of your Confluent Cloud cluster. |
| **API Key** and **API Secret** | Credentials for Kafka access. |
| **Schema Registry Key** and **Schema Registry Secret** | Credentials for Schema Registry access. |

> 💡 **Tip:** After filling in the **Bootstrap Server**, **Schema Registry URL**, and **Cluster ID**, you can execute a test connection to confirm communication with your Confluent Cloud environment.
<img width="813" height="361" alt="image" src="https://github.com/user-attachments/assets/b8058e16-c3da-4988-ba19-6a9ecbebee88" />

### Option B: Confluent Platform

If you select **Confluent Platform**, enter the following information:

| Field | Description |
|---|---|
| **Security Protocol** | Select between `Standard`, `Plain Text`, or `SASL_SSL`. |
| **Bootstrap Server** | The Kafka bootstrap server address. |
| **Schema Registry URL** | The URL of the Confluent Schema Registry. |
| **API Key** and **API Secret** | Credentials for Kafka access. Only required if using `Plain Text` or `SASL_SSL`. |
| **Schema Registry Key** and **Schema Registry Secret** | Credentials for Schema Registry access. Only required if using `Plain Text` or `SASL_SSL`. |

> 💡 **Tip:** After filling in the **Bootstrap Server** and **Schema Registry URL**, you can execute a test connection to confirm communication with your Confluent Platform environment.
<img width="810" height="414" alt="image" src="https://github.com/user-attachments/assets/95e49757-a12c-467e-9c99-2f8dc0399307" />


> 📌 **Note:** The connection test used to verify Kafka connectivity points to the default port of the **Kafka REST Proxy** (`8082`). If the test fails when using **Confluent Platform**, this may be because the service is not available or is running on a different port.

---

## Stage 3: Topic Settings

In the **Topic Settings** section, enter the required information and click **"Next"**.

The system will automatically populate some default values, which can be modified as needed:

| Field | Description |
|---|---|
| **Number of partitions** | The number of partitions for each Kafka topic created by the connector. |
| **Number of replicas** | The replication factor for each Kafka topic. |
| **Retention period (in days)** | How long messages are retained in each topic before being deleted. |
<img width="816" height="318" alt="image" src="https://github.com/user-attachments/assets/13b13c4d-ce30-44a5-8432-5da28824a570" />

---

## Stage 4: Advanced Settings

In the **Advanced Settings** section, select the desired resources for the SAP Connector container:

| Setting | Description |
|---|---|
| **Request CPU** | The guaranteed CPU allocation for the connector container. |
| **Request Memory** | The guaranteed memory allocation for the connector container. |
| **Limit CPU** | The maximum CPU the connector container can consume. |
| **Limit Memory** | The maximum memory the connector container can consume. |

Click **"Finish"** to complete the process. You will be redirected to the **"Users"** section, where the new SAP Connector has been created.

<img width="813" height="324" alt="image" src="https://github.com/user-attachments/assets/fe50ac87-d096-4a25-9632-06bd27a823b5" />

---

## Step 4: Access the New SAP Connector

To access your new SAP Connector:

1. Click on the **username** or the **action button**.
2. Navigate to the **"SAP Connectors"** section.
3. You will see a list with your new connector. Click the **"GO"** button to access it.
<img width="1360" height="384" alt="image" src="https://github.com/user-attachments/assets/d9b3ddfd-004e-4122-ae2f-51c1e3ac4df8" />

Here, you can:

- View information related to your SAP Connector.
- Retrieve the **URL needed for the RFC connection in SAP**.
<img width="1149" height="591" alt="image" src="https://github.com/user-attachments/assets/50626a99-0b8b-4f47-8fb0-d90c349daaf0" />

> ℹ️ For step-by-step instructions on configuring the RFC destination in SAP, see the [Endpoint Customization (SM59) manual](../SAP_Data_Modeler/15-Endpoint_Customization_SM59.md).

---

## Summary

The complete workflow for creating an SAP Connector is:

1. Log in to the Smart Gateway platform.
2. Navigate to **Users** and select the target user.
3. Go to **SAP Connectors → Add SAP Connector**.
4. **Stage 1:** Define the connector name, environment, username, and password.
5. **Stage 2:** Configure the Confluent Cloud or Confluent Platform connection.
6. **Stage 3:** Set the topic partitions, replicas, and retention.
7. **Stage 4:** Set the container CPU and memory resources.
8. Click **Finish**.
9. Access the connector to retrieve the RFC URL for SAP.
