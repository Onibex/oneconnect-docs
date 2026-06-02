# Uploading Prepackaged Entities

**Onibex offers prepackaged entities that include predefined structures with tables and relationships designed for multiple SAP business processes.**

Uploading a prepackaged entity helps save time during the entity creation process while ensuring that all required tables for the selected process are correctly included and aligned with best practices.

> 📌 **Note:** Before proceeding with this step, make sure you have the Excel files provided directly by the Onibex team. If these files are not available or if the functionality is not enabled in your system, please contact the Onibex team for assistance.

---

## Steps to Upload a Prepackaged Entity

### 1. Access the ZONT_ONECM Transaction

Go to the Transaction field in SAP, type **`ZONT_ONECM`** in the input box, and press **Enter**.
<img width="711" height="168" alt="image" src="https://github.com/user-attachments/assets/d1b498ac-c5af-4bf8-8325-0ede8d7685cf" />

### 2. Open the Upload Option

On the upper left side of the screen, select **"Entity Actions"** and then **"Upload from Excel"**.
<img width="1094" height="435" alt="image" src="https://github.com/user-attachments/assets/68944b2d-22f6-4552-97a0-3ec6912f5b3f" />

### 3. Configure the Upload Parameters

Upload the Excel file with the SAP Operating Data Model.

> ℹ️ The Excel file must be previously delivered by the Onibex team for use and modification.

Configure the following parameters:

1. **File Path:** Select the location of the Excel file to be used.
2. **Sheet Name:** Select the name of the Excel sheet of the entity that is going to be uploaded. The name must match exactly the one in the sheet; otherwise, the entity will not be created.
3. **Load Entity Relations:** Make sure to select this option. The other available options are explained in other manuals.
<img width="1232" height="450" alt="image" src="https://github.com/user-attachments/assets/51cc435c-14c7-4e6b-9ba5-9caca4794bae" />

### 4. Execute the Upload

After configuring the parameters, click on **"Execute"**.

After a few seconds, the system will complete the processing of the prepackaged entity. You should then see the entity listed in the **Entities** section within the OneConnect component.

---

## Verify the Uploaded Entity

> ⚠️ **Recommended:** After the entity is created, access the configuration of the newly created entity using the **"Change"** option to verify that the structure, values, and selected fields are correct. This ensures the prepackaged entity aligns with your specific requirements before execution or further integration steps.

> ✅ If needed, the user can adjust and modify the created entity by following the same steps used to create and customize an entity manually. These steps are explained in detail in the [Entity Creation and Main Customizing manual]
