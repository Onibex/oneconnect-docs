# Mass Field Configuration via Excel Upload

This manual explains the steps to correctly prepare and upload an Excel file to mass configure fields/columns for the tables used in specific entities in your system. Follow these instructions carefully to avoid errors.

> ℹ️ **Note:** Make sure to ask the Onibex team to provide you with the correct Excel file for this process and to have followed the previous steps:
> - [Entity Creation and Main Customizing](./Entity_Creation_and_Main_Customizing.md)
> - [Uploading Prepackaged Entities](./Uploading_Prepackaged_Entities.md)

---

## 1. Preparing the Excel File

When creating the Excel file, adhere to the following column definitions:

| Column | Name | Description |
|---|---|---|
| **A** | Table | Specify the name of the table you wish to use. |
| **B** | Table Alias | This column must contain the exact alias used for the target table. *Keep in mind that a single table may have multiple aliases, so make sure you are referencing the correct alias corresponding to the specific instance or context within your entity.* |
| **C** | Field Name | Enter the technical name of the field to be used. |
| **D** | Alias | Define an alias for the field. *Avoid using spaces or special characters in the alias.* |
| **E** | Primary Key | Mark the primary keys with an **"x"** in this column. |
| **F** | Filter | Mark fields that can be used as filters or criteria for data transfer with an **"x"** in this column. This configuration can also be adjusted directly in the entity later. |
| **G** | Description | Provide a description for the field. Ensure this is not confused with the alias in Column D. |

---

## 2. Uploading the Excel File

Once the Excel file is prepared, follow these steps to upload it:

### Step 1: Navigate to Entity Actions

In OneConnect, navigate to **"Entity Actions"** and select **"Upload from Excel"**.
<img width="1097" height="425" alt="image" src="https://github.com/user-attachments/assets/151c26fb-b5cf-4f59-971c-cb87b2621fea" />

### Step 2: Locate the File and Specify the Sheet

In the pop-up window, locate the Excel file on your system and specify the sheet name containing the data (e.g., **`COLUMNS`**).

### Step 3: Select the Correct Upload Option

Select the option labeled **"Load Entity Column Data"**.

> ⚠️ **Important:** Do **NOT** select **"Load Entity Relations"**. That option is used for uploading entity structures and tables, which is previously explained in the [Uploading Prepackaged Entities manual](./02-uploading-prepackaged-entities.md).
<img width="1228" height="447" alt="image" src="https://github.com/user-attachments/assets/30948237-2e88-4f02-8116-e5993fd7821a" />

### Step 4: Execute the Upload

After selecting the correct file and upload option, execute the upload process. Wait a few seconds for the configuration to be saved.

### Step 5: Verify and Save

Navigate to the entity in your system for which you uploaded the fields/columns and verify that the fields have been uploaded correctly. After that, **save the customizing** to confirm the addition or modification of the fields. The entity should regenerate.

---

## 3. Modifying Fields (if needed)

If modifications are necessary, you can adjust the fields directly in the entity. Ensure that any changes are reviewed before proceeding.

For this process, please refer to the [Configuring Table and Field Aliases section](./Entity_Creation_and_Main_Customizing.md) of the Entity Creation manual.

---

## Important Notes

> ⚠️ **This file upload targets a table by its specific alias.**
> For example, if multiple entities include the same table (e.g., `VBAK`) with the alias `TABLE_TEST`, and your Excel file references this table and alias in columns A and B, the upload will apply changes to **all entities** that use that exact table-alias combination.
>
> **To avoid unintentionally modifying multiple entities**, assign unique aliases to tables when you intend to target only a specific instance of that table within a particular entity.

Additional considerations:

- **Avoid spaces or special characters in the Alias (Column D).** If a space is needed, use an underscore `_` instead.
- **Descriptions (Column G)** are used for clarity and interpretation but do not replace the Alias.

By following these instructions, you will successfully configure the fields for your entity using the Excel upload process.
