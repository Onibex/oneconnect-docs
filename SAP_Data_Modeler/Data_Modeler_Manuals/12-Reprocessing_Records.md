# JSON Reprocessing

This reprocessing procedure is carried out when an error occurs during data transmission through entities, whether via **batch** or **real-time**. It also applies to cases where no error occurred, but there is a specific need to resend a JSON from one of the tables instead of performing a full traditional submission.

> ℹ️ Should an error occur within the **RFC Destination**, whether due to connection issues or incorrect configuration, there is no need for concern. The JSON files are stored and flagged with an error status (**`E`**) within a transaction that allows for individual resubmission, ensuring data backup and integrity.

---

## Reprocessing Steps

### 1. Access the Reprocessing Transaction

Enter transaction code **`ZONT_REPRO`**.
<img width="856" height="448" alt="image" src="https://github.com/user-attachments/assets/5e72da1f-72d1-43f7-9cc6-20d3718f39c6" />

### 2. Define the Selection Criteria

Select the date and time range when the data transmission failure occurred.

> ℹ️ By default, the current date will be displayed.

Then, click **"Execute"**.
<img width="866" height="455" alt="image" src="https://github.com/user-attachments/assets/556f2bb0-124c-4ad9-9ff6-072c7b58f401" />

### 3. Review the Status Overview

The system will display the JSON information labeled with three distinct statuses:

| Status | Code | Description |
|---|---|---|
| **Processed** | `P` | The submission was successful and reached its RFC Destination correctly. |
| **Error** | `E` | The submission failed and did not reach its RFC Destination. |
| **Reprocessed** | `R` | Applies to JSONs that were resent through this program and successfully reached their RFC Destination. |
<img width="892" height="135" alt="image" src="https://github.com/user-attachments/assets/63d971bc-d012-4164-8b8d-c854fa1092fd" />

### 4. Preview the JSON Data

To view the JSON details, double-click the icon in the **"Preview JSON"** column for the specific record you wish to visualize.
<img width="897" height="470" alt="image" src="https://github.com/user-attachments/assets/e51aea9f-e80a-47da-b688-ca036b348cc0" />

### 5. Modify the RFC Destination (Optional)

To change the destination, select the row of the JSON you want to reprocess and click the **"Change Destination"** button in the menu.
<img width="939" height="491" alt="image" src="https://github.com/user-attachments/assets/92c95ad9-a460-44bc-bfee-1868f4d0ae15" />

> ℹ️ You can either enter the destination manually or search for it.

### 6. Execute the Reprocessing

Once the JSON has the correct destination and is verified for resubmission, select the row and click the **"Reprocessing"** button.
<img width="939" height="150" alt="image" src="https://github.com/user-attachments/assets/cceac129-9d2a-4b87-acd9-88355155bc13" />

> ✅ Upon a successful execution, the status will change from **`E`** to **`R`**, and a reprocessing date will be assigned.

### 7. Delete a Record (Optional)

To delete a JSON, simply select the row and click the **"Delete"** option. The row will be automatically removed.
<img width="939" height="142" alt="image" src="https://github.com/user-attachments/assets/e1ba3e98-2074-4fb0-895a-4144c19a5f1d" />
