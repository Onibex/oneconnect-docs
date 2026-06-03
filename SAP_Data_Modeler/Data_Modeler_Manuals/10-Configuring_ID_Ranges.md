# How to Configure the Z Range When Creating Entities

If you encounter the error message **"PLEASE CONFIGURE Z RANGE"** while creating an entity, follow this guide to correctly configure the number range before proceeding.

---

## Purpose

This configuration establishes the internal number ranges required by Onibex's SAP Data Modeler for creating entities and columns.

---

## Steps to Configure

### 1. Open the SNRO Transaction

In SAP, go to transaction **`SNRO`**.

<img width="381" height="238" alt="image" src="https://github.com/user-attachments/assets/181e353f-1462-4d2f-87cd-c2d1fff66a37" />

### 2. Configure the Object Names

In the transaction screen, under **Object Name**, configure the following objects:

| Object | Purpose |
|---|---|
| **`ZONR_COL_A`** | Number range for columns. |
| **`ZONR_ENTIT`** | Number range for entities. |
<img width="921" height="540" alt="image" src="https://github.com/user-attachments/assets/3ce0f8ef-0194-4efa-9b6e-ec4ba9ac2794" />

### 3. Open Interval Editing

Select one of the objects, then click on **"Interval Editing"**.
<img width="788" height="464" alt="image" src="https://github.com/user-attachments/assets/92b7793b-1a50-4e01-826b-76c9d5a2050e" />

### 4. Open the Change Intervals Screen

Click on **"Change Intervals"**, represented by the **pencil icon**.
<img width="921" height="292" alt="image" src="https://github.com/user-attachments/assets/aa6fe778-1ed4-481a-b7a5-873499095702" />

### 5. Define the Range Values

In the table that appears, fill in the following values:

| Field | Value |
|---|---|
| **Number Range No.** | `01` |
| **From No.** | `000001` |
| **To Number** | `999999` |
<img width="921" height="527" alt="image" src="https://github.com/user-attachments/assets/acf1363a-65fe-4df3-b555-9b22c1f49572" />

### 6. Save the Configuration

After entering the values, click on **"Save"**.

### 7. Confirm the Configuration

A confirmation message will appear. Click the **check mark** or press **`Enter`** to confirm.
<img width="921" height="666" alt="image" src="https://github.com/user-attachments/assets/bd4d5a97-5112-4ee0-89ec-0f9e58fcf047" />

> ⚠️ **Repeat steps 3 through 7 for both objects** (`ZONR_COL_A` and `ZONR_ENTIT`).

---

## Expected Result

> ✅ Once completed, the ID ranges for both entities and columns will be properly configured. You can now create new entities using **Onibex's SAP Data Modeler** without encountering the **"PLEASE CONFIGURE Z RANGE"** message.

---

## Notes

> ⚠️ **Important considerations:**
>
> - This setup needs to be done only **once per system**, unless the configuration is reset or deleted.
> - Make sure to repeat the steps for **both objects** (`ZONR_COL_A` and `ZONR_ENTIT`).
> - Ensure you have sufficient **authorization** to edit number range intervals in SAP.
