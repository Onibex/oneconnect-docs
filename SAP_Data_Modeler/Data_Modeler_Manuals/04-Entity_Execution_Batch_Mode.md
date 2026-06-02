# Batch Mode Entity Sending

> ℹ️ **Pre-execution Requirements**
>
> Before executing an entity and sending information in batch mode, ensure that the entity is correctly created and configured. Likewise, confirm that the endpoint configuration is properly set up and tested.

---

## Execution Steps

### 1. Open the OneConnect Transaction

Enter transaction **`ZONT_ONECM`**.
<img width="191" height="121" alt="image" src="https://github.com/user-attachments/assets/63d53586-d5b8-4549-9c8e-d78583b2ce94" />

### 2. Select and Execute the Entity

Double-click the entity to be sent (it should appear selected at the top), then click the **"Execute"** button.
<img width="1085" height="462" alt="image" src="https://github.com/user-attachments/assets/e69fd26c-5a0c-4f90-ad09-22f9113655af" />

### 3. Define the Transmission Parameters

In the popup window, select the parameters for data transmission:

| Parameter | Description |
|---|---|
| **Output Type** | Choose whether to send the field's technical name (*field name*), the alias/label assigned to the field (*field label*), or both formats concatenated separated by an underscore (e.g., `vbeln_sales_order`). |
| **Output Format** | Select between **KDOC** (Kafka document, structured as an IDOC-like XML string), **Table format** (structured with records and columns in a table format), or **both** (sending KDOC and table structure). |
| **Endpoint Connection** | Select the previously created RFC connection from transaction **`SM59`**. |

Click **"Execute"** at the bottom of the popup, or press **`F8`**.
<img width="846" height="613" alt="image" src="https://github.com/user-attachments/assets/cfcdbd6b-cbef-4531-8858-dce8f7849812" />

---

## Batch Sending Criteria Selection

On the next page, all possible batch selection criteria will appear on the left side. These are columns marked as **"Selection"** in the entity configuration.

To configure the criteria:

1. Double-click the field to be set as the sending criterion to display the selection options.
2. Select the criterion for sending the entity.
3. Once criteria are defined, click **"Save"**.
<img width="1359" height="760" alt="image" src="https://github.com/user-attachments/assets/1adc0510-8bd3-49ee-968e-4d094ef78d96" />

---

## Choosing Execution Mode

Select the execution mode for data sending:

- **Online:** Recommended when the data volume is small, or when the process does not exceed the available SAP user memory.
- **Background:** Recommended for large data volumes to prevent memory issues. Also useful for scheduling execution at a specific date and time.
<img width="842" height="619" alt="image" src="https://github.com/user-attachments/assets/d86a36dc-3c8c-4ac4-8661-cf46054965fc" />

### Online Mode Execution

If executed in Online mode, a summary log will appear at the end, displaying:

| Field | Description |
|---|---|
| **Sent Tables** | Each table sent, with its corresponding alias. |
| **TOTAL_SIZE_SENT** | Total bytes sent. |
| **TOTAL_NUMBER_OF_REQUEST** | Number of requests made. |
| **TOTAL_NUMBER_OF_RECORDS** | Number of records sent for the entity. |

> ✅ Check the destination system to verify that the information was received correctly.
<img width="1272" height="456" alt="image" src="https://github.com/user-attachments/assets/b106000e-ad07-460d-bcdf-906a3a4c0745" />

### Background Mode Execution

A job configuration window will appear to schedule the execution time.

1. Select the desired execution time.
2. Click **"Save"**.
<img width="836" height="767" alt="image" src="https://github.com/user-attachments/assets/96c4a17b-8cdd-4b81-a88a-b139665d03ee" />

Once completed, the data sending process will execute at the configured time.

> ℹ️ Generated jobs and their results can be reviewed in the **"Display Job Sched."** menu.
> <img width="1087" height="412" alt="image" src="https://github.com/user-attachments/assets/a3e138e2-b711-416c-9efb-f647a702a500" />
