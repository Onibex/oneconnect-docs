# Smart Gateway Configuration Manual

## Generating a SAP OTel Bridge Endpoint (Traceability)

The **SAP OTel Bridge** (also referred to as **Traceability**) is an endpoint that, once generated, is placed in your SAP system. When errors occur in your SAP processes, those errors — along with their traceability data and logger information — become visible in the **Logs** section of the OneConnect platform. This allows you to correlate SAP-side issues with Kafka-side logs in a single view.

> 📋 **Prerequisites:**
>
> - An existing **SAP Link** created in your OneConnect environment. If you don't have one, see the [Creating a SAP Link manual](./02-Creating_a_SAP_Link.md).
> - The SAP Link must be accessible from your user session.

---

## Step 1: Navigate to Your SAP Link

1. Log in to the OneConnect platform.
2. From the sidebar, locate your SAP Link under the **SAP Links** section.
3. Click on the SAP Link name to open its dashboard.

---

## Step 2: Open the Add Menu

In the SAP Link dashboard, locate the **"Add"** button in the upper-right area of the header and click it.

A dropdown menu will appear with the available add-on options.

---

## Step 3: Select "Add Traceability"

From the dropdown menu, select the **"Add Traceability"** option.

> ℹ️ **Note:** If the option appears as **"Traceability Configured"** and is disabled, it means this SAP Link already has a Traceability endpoint configured. Each SAP Link can only have one Traceability endpoint.
<img width="1637" height="431" alt="image" src="https://github.com/user-attachments/assets/5e647537-06fb-4c64-9b98-7eb429f92381" />

---

## Step 4: Configure the Traceability Wizard

The Traceability configuration wizard consists of **2 stages**:

### Stage 1: General Information

| Field | Description |
|---|---|
| **Workspace ID** | Automatically filled with the current SAP Link ID. This field is **read-only**. |
| **Username** | Automatically filled with your email address. This field is **read-only**. |
| **Name** | A descriptive name for this Traceability bridge configuration. |

Fill in the **Name** field and click **"Next"**.

<img width="1623" height="583" alt="image" src="https://github.com/user-attachments/assets/3befab61-a3ef-490b-8e36-a555f4c8a80d" />


### Stage 2: Authentication Credentials

In this stage, provide the credentials that SAP will use to authenticate when sending data to this Traceability endpoint:

| Field | Description |
|---|---|
| **SAP User** | The username SAP must send when calling this endpoint. |
| **SAP Password** | The password SAP must send alongside the username. |

> ⚠️ **Important:** Save these credentials carefully, as they **cannot be viewed again** through the web application once the configuration is saved.

Enter both fields and click **"Save Configuration"**.

<img width="1317" height="565" alt="image" src="https://github.com/user-attachments/assets/795ee90c-66f7-46df-8156-2186dba91630" />


---

## Step 5: Confirmation

Once the configuration is saved successfully, a confirmation screen will appear displaying:

- The **Bridge name** that was created.
- The **SAP Link** to which the bridge is associated.

Click **"Return"** to go back to the SAP Link dashboard.

---

## Step 6: Retrieve the Traceability URL

After creating the Traceability endpoint:

1. Go back to your SAP Link dashboard.
2. You will see a new **"Traceability"** tab next to the **"SAP Link"** tab.
3. Click the **"Traceability"** tab to view the generated endpoint URL.
4. Use the **copy icon** 📋 to copy the URL to your clipboard.

<img width="1626" height="567" alt="image" src="https://github.com/user-attachments/assets/fcbbf490-fab9-4d11-8fb6-f9477f035dde" />


> 📌 **Important:** This URL must be configured in your SAP system’s settings. Once configured, SAP errors will automatically be sent through this bridge and become visible in the **Logs** section of OneConnect.

---

## Step 7: Verify in Logs

To confirm the Traceability bridge is working:

1. Trigger an error scenario in your SAP system that uses this bridge.
2. Navigate to the **Logs** section of your SAP Link.
3. Filter by **Source = SAP** to view SAP-side errors with full traceability data.

> 💡 **Tip:** For detailed information on how to work with logs, see the [Working with Logs manual](./06-Working_with_Logs.md).

---

## Summary

The complete workflow for generating a SAP OTel Bridge endpoint is:

1. Access your SAP Link dashboard.
2. Click **Add → Add Traceability**.
3. **Stage 1:** Enter a descriptive **Name**.
4. **Stage 2:** Provide your **SAP User** and **SAP Password** credentials.
5. Click **Save Configuration**.
6. Copy the generated **Traceability URL** from the Traceability tab.
7. Configure the URL in your SAP system's RFC destination.
8. Verify SAP errors appear in the **Logs** section (filtered by Source = SAP).
