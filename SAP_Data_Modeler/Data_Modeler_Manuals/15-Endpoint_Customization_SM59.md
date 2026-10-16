# Endpoint Customization (SM59)

SAP uses the **Remote Function Call (RFC)** to establish connections with external systems. This RFC serves as an exclusive interface for communication between SAP systems and external applications, enabling SAP to function as both a client and a server in familiar RFC scenarios.

To customize this asset in SAP, it is crucial to know the parameters governing the web connection to which SAP will be linked. Essential details such as the **URL**, **port**, and **RFC type** are required for its successful integration.

> 📋 **Prerequisites:** Before starting, make sure you have the host, port, path prefix, and credentials of the destination created beforehand in the Smart Gateway.

---

## Step 1: Access Transaction SM59 in SAP

Go to the **Transaction** field, enter **`SM59`**, and press **Enter** or click the checkmark.
<img width="2236" height="499" alt="image" src="https://github.com/user-attachments/assets/974aebc6-333f-4c58-8074-70289c954cd8" />

---

## Step 2: Locate HTTP Connections

Expand the **HTTP Connections to External Server** node.

- If you **already have an RFC** created, select it and skip to **Step 5** to make any necessary changes.
- If you **need to create one**, continue with the next step.
<img width="2236" height="880" alt="image" src="https://github.com/user-attachments/assets/cb9670fe-b272-4997-a000-b10072b771e1" />

---

## Step 3: Create or Select the Connection

Click **"Create"**. If you already have a connection, select it instead.
<img width="1917" height="1092" alt="image" src="https://github.com/user-attachments/assets/db738345-9fcc-495c-afa1-2e4e1d5699db" />

---

## Step 4: Define the Connection Name

Fill in the following values:

| Field | Value |
|---|---|
| **Connection Type** | **G — HTTP Connection to External Server** |
| **Destination Name** *(recommended)* | `ONIBEX_ONECONNECT` or `ONIBEX_POC` |
<img width="2201" height="1179" alt="image" src="https://github.com/user-attachments/assets/d57461a1-5a00-4136-b495-5365a9738206" />

---

## Step 5: Configure Technical Settings

Fill in the following values on the **Technical Settings** tab:

| Field | Description |
|---|---|
| **Descriptions** *(optional)* | `Onibex One Connect` |
| **Host** | IP address of the server where data is sent from SAP. |
| **Port** | Port number used to connect with the host. |
| **Path Prefix** | The path where the web/app is located. |
<img width="1334" height="279" alt="image" src="https://github.com/user-attachments/assets/747157d8-8c15-4b2c-9ffe-2caba7eb80f6" />

> 💡 **Example: How to split a URL**
>
> Full URL: `https://{DOMAIN}:5051/api/v1/saplistener/{CLIENT}`
>
> - **Host:** `{DOMAIN}`
> - **Port:** `5051`
> - **Path Prefix:** `/api/v1/saplistener/{CLIENT}`
<img width="1652" height="1064" alt="image" src="https://github.com/user-attachments/assets/ebeb2105-2324-4e58-abef-774e965e2461" />

### Choosing the Endpoint: Synchronous vs. Asynchronous

The Smart Gateway exposes two ingestion endpoints for each SAP Link. You select one through the **Path Prefix** by adding or omitting the `/synchronous` suffix.

| Mode | Method | Path Prefix | Behavior |
|---|---|---|---|
| **Synchronous** *(default, recommended)* | `POST` | `/api/v1/saplistener/{CLIENT}/synchronous` | Returns the Kafka offset of the record. If no offset is produced, it returns `400`. |
| **Asynchronous** | `POST` | `/api/v1/saplistener/{CLIENT}` | The Producer accepts the message and returns `201`. |

> ✅ **Use the synchronous endpoint by default.** Configure the **Path Prefix** as `/api/v1/saplistener/{CLIENT}/synchronous`.
>
> Full URL: `https://{DOMAIN}:5051/api/v1/saplistener/{CLIENT}/synchronous`

> 📌 **The URL shown by the platform for the SAP Link already points to the synchronous endpoint** (it ends in `/synchronous`), so use it as-is: split it into **Host**, **Port**, and **Path Prefix** without adding or removing anything. When the platform publishes SAP Links through an ingress, the ingress exposes this synchronous path for each SAP Link; when it uses a load balancer, the URL has the form `http://{LOAD_BALANCER_ADDRESS}:8080/api/v1/saplistener/{CLIENT}/synchronous`.

> ℹ️ The **Connection Test** uses `GET`, which does not authenticate on either endpoint. A `200` confirms the server is reachable, not that the credentials are valid.

---

## Step 6: Configure Logon & Security

In the **Logon & Security** tab, enter the username and password that you previously set up on the **OneConnect** site.

> ℹ️ These credentials must match the user account configured in the Smart Gateway for the corresponding workspace.
<img width="484" height="396" alt="image" src="https://github.com/user-attachments/assets/bb4575a2-4db9-4a82-8b2b-5ec53ab6e880" />

---

## Step 7: Test the Connection

Once all required fields are completed, click **"Connection Test"**.
<img width="727" height="276" alt="image" src="https://github.com/user-attachments/assets/1f6cf865-cbf8-4524-b6e8-2babf29eb1d5" />

### Interpreting the HTTP Response Status

| Status Code | Meaning | Action |
|---|---|---|
| **200** | ✅ The endpoint is reachable. | No action needed. |
| **404** | ⚠️ The server was reached, but the path does not match this SAP Link. | Review the **Path Prefix** and the workspace ID; copy them from the URL shown on the SAP Link screen. |
| **Any other code** | ℹ️ The response comes from a network component between SAP and the Smart Gateway (SAP ICM, proxy, ingress, or load balancer). | Review host, port, SSL settings (`http` vs `https`), and the network path to the Smart Gateway. |
<img width="512" height="288" alt="image" src="https://github.com/user-attachments/assets/49709354-ee7c-4506-9087-bb57994f6e52" />

> ℹ️ The credentials in **Logon & Security** are validated when SAP sends data (`POST`). See the [HTTP status table](../../Troubleshooting/03-Authentication_and_Credentials.md#reading-the-http-status-code) for the codes returned at that point.

> ✅ **You're done!** Your SAP system is now configured to communicate with the external endpoint via RFC.
