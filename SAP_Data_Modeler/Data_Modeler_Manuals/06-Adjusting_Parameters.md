# OneConnect Parameter Adjustments

**The OneConnect solution includes a parameter table used for sending information, especially real-time data.**

It is **important to configure this table beforehand**, prior to sending any data, to ensure proper communication and data handling.

---

## Accessing the Parameter Table

Follow these steps to configure your parameter table:

### 1. Open the OneConnect Transaction

Enter transaction **`ZONT_ONECM`**.

<img width="534" height="377" alt="image" src="https://github.com/user-attachments/assets/6327d0db-a221-4b56-ad14-0eb13ddc2f9e" />

### 2. Navigate to the Parameter Table

Within the OneConnect transaction, open the **"Execution"** menu at the top and select **"Parameter Table"**.
<img width="921" height="363" alt="image" src="https://github.com/user-attachments/assets/c52485f9-7911-49e2-b77f-927ffd7917cf" />

---

## Parameter Options

### RFC_DESTINATION

This is the default endpoint where the information will be sent in real time. The endpoint must exist in **`SM59`**, and it must have the same name and a proper communication setup with the target system.

### SYSID

This is marked with an **`X`** in the parameter value field and is used to identify and send the system ID information along with the data sent from an entity.

### SYSTEM_OPEN

This is marked with an **`X`** in the parameter value field and is used to indicate whether the OneConnect solution is open for any user to create, modify, or delete entities.

> ℹ️ If the value is left blank, the system is considered **closed**, and users will only be allowed to execute existing entities.

### TRANSMISSION_MEDIUM

This defines the method for sending information from SAP. The available options are:

| Value | Mode | Description |
|---|---|---|
| **`T`** | Table mode | Sends data structured with records and columns in a table format. |
| **`K`** | KDocs mode | Sends data as Kafka documents in an IDOC-like XML structure. |
| **`B`** | Both | Sends data using both Table and KDoc modes simultaneously. |

### USE_ALIAS

This parameter defines whether table and column aliases are sent. If marked with an **`X`**, the values will be sent with their aliases; otherwise, the technical names of tables and columns will be sent.

### TAG2 and TAG3

These are prefilled values that appear in each entity. They help identify the **SAP version** and the **client number** used to send information from the entity.
<img width="1340" height="351" alt="image" src="https://github.com/user-attachments/assets/d177e5a6-9e83-44a5-b1eb-1ecefb4c2250" />

---

## Adjusting Parameters

To modify a parameter value, follow these steps:

1. Ensure that **edit mode** is enabled (top-right of the screen).
2. Modify the **"Field Value"** field with the required value, ensuring correctness.
3. Press **`Enter`** to apply the change.
4. Click **"Save"** to confirm the update.
<img width="909" height="307" alt="image" src="https://github.com/user-attachments/assets/43e674d2-c6cc-4a22-94d6-418dfde96394" />

5. Select the request where this change will be stored, then click **"Continue"**.
<img width="794" height="431" alt="image" src="https://github.com/user-attachments/assets/e68600e1-40f9-4378-adfe-a75a637aeedd" />

> ⚠️ **Caution:** Incorrect parameter values can directly affect real-time data extraction.

---

## Confirmation

Upon successful parameter adjustment, a confirmation message will appear at the bottom of the screen.
<img width="825" height="192" alt="image" src="https://github.com/user-attachments/assets/1408c8ab-ee70-4a6b-8d49-1641cb51d04f" />
