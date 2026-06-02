# Sending Data with the "ANY" Entity

## Introduction

This document provides a step-by-step guide for creating and configuring the **"ANY"** entity within Onibex's SAP Data Modeler using transaction `ZONT_ONECM`. It outlines the essential processes, including selecting the appropriate domain and data type, defining entity attributes, configuring the Main Customizing Data, and executing the entity to send data from any SAP table.

By following the outlined procedures, users can effectively structure and optimize the ANY entity for flexible data transmission, enabling on-demand delivery of any SAP table without predefining its structure in the modeler.

---

## Creating the Entity

### 1. Access the OneConnect Transaction

Access SAP using transaction **`ZONT_ONECM`**.

<img width="342" height="241" alt="image" src="https://github.com/user-attachments/assets/16c05cd4-caa8-4760-bfce-4e785507c67b" />

### 2. Create a New Entity

Click on **"Create"**.
<img width="921" height="374" alt="image" src="https://github.com/user-attachments/assets/d59cfb47-c072-40af-b014-102a87a3540f" />

After clicking **"Create"**, you will be taken to the entity configuration screen.

### 3. Define the Entity Name

In the **"Entity"** field, enter the name of the entity. This entity must be named **`ANY`**.
<img width="473" height="47" alt="image" src="https://github.com/user-attachments/assets/3daf89ac-f98c-4735-ba02-cf5697588218" />

### 4. Configure Tag Data and Tag Metadata

In the **"Tag Data"** and **"Tag Metadata"** sections, check each box as needed to include the data contained in the tags located in the specified rows of the configuration.

Select these options only if your use case requires sending either the tag values or their associated metadata. ***(It is recommended to check them to easily track information.)***

- **Tag 1** is **mandatory**. It should contain the **Domain: `ANY`**. You can directly type it or select it from the available **dropdown list**.

<img width="197" height="291" alt="image" src="https://github.com/user-attachments/assets/a049e246-3e94-404a-9655-c34ba66b1117" />

- **Tags 2 and 3** contain **prefilled values** that are configured in the parameter tables (as explained in the [Customizing Parameters manual](https://help.onibex.com/portal/en/kb/articles/sap-data-modeler-customizing-parameters)). These values are used to **identify the SAP version and instance** from which this entity is sending data.

- **Tags 4 and 5** are **optional**. They can contain any type of information that is relevant to the entity and may help further identify or describe it in more detail.

> ⚠️ **Important:** For the ANY entity, there is no need to select tables in the **Build Join** function or columns in the **Column Definition** function.

---

## Configuring Main Customizing Data

Just before saving the entity, certain customizations must be defined to extract the data selected in the entity. These options are found in the **"Main Customizing Data"** section.


Here is a general overview of each segment:

### 1. Data Type

Select **Transactional Data (T)**.

<img width="346" height="79" alt="image" src="https://github.com/user-attachments/assets/f6a8710c-3456-4b9c-959e-83bad239c06d" />


### 2. Description

Provide a general description of the entity and its contents. This section is for informational purposes only.
<img width="348" height="40" alt="image" src="https://github.com/user-attachments/assets/1328b6ea-e3cb-4668-9e05-cb41f86229d6" />


### 3. Number of Records

Select the number of records the entity will send per request. The recommended value is **100** for optimal performance and stability during data transmission.

### 4. Event ID

Check this box if the information being sent requires an **Event ID** to identify the executed event. Enabling this option is recommended to ensure traceability and proper event identification.

### 5. Auth Object

Here, you can assign an authorization object to the specific entity. This ensures that only SAP users with the defined authorization object can **modify or delete** the entity.

To configure it:

1. Enter the **Authorization Object** in the corresponding field.
2. Click on **"Set Values"**.
3. In the **Value** field, type the required value for the authorization object.

### 6. Log

Here, you can choose where to store the process log of this entity. There are two available options:

- **Standard Log:** Saves all logs in the **standard OneConnect log table**.
- **Server File:** Stores all logs in a `.txt` file located in an **application server directory** (this option requires a valid directory path to be specified).

---

## Save Entity Customization

After finishing the configuration of the entity, to save all the changes, go to **"Save Customizing"** in the header of the entity configuration.
<img width="1018" height="394" alt="image" src="https://github.com/user-attachments/assets/30537f98-dc4a-415b-af84-83d2bb7a9ade" />


> ✅ After clicking **"Save"**, the system will begin processing the entity, and it will be created in your entity list.

---

## Executing the "ANY" Entity

After the entity is created, it can be executed.

### 1. Select and Execute

Double-click on the **ANY** entity and select **"Execute"**.
<img width="1088" height="446" alt="image" src="https://github.com/user-attachments/assets/886b5551-bf1b-42d4-aa74-9200499ce40e" />


### 2. Define the Transmission Parameters

Select the parameters for data transmission:

| Parameter | Description |
|---|---|
| **Output Format** | Select between **KDOC** (Kafka document, structured as an IDOC-like XML string), **Table format** (structured with records and columns in a table format), or **both** (sending KDOC and table structure). |
| **Endpoint Connection** | Select the previously created RFC connection from transaction **`SM59`**. |
| **Table Name** | Select the technical name of the table that you want to send. |

Click **"Execute"** at the bottom of the popup, or press **`F8`**.
<img width="850" height="647" alt="image" src="https://github.com/user-attachments/assets/2f00b7df-f6c4-40eb-badc-0e1101b38020" />


### 3. Select Table Alias and Fields

In the next screen, you will be prompted to select the following information:

| Field | Description |
|---|---|
| **Table Alias** | Type the alias of the table that you wish to use (if you previously selected to send the alias fields). |
| **Fields Selection** | Select the specific fields from the table that you want to transfer. Review the list of available fields and check the **"Include JSON"** box next to each field you wish to include in the data transfer. |
| **Selection** | If you wish to use a field as a selection criterion for sending information in batch mode, check the box labeled **"Selection"** next to the corresponding field. |

After finishing the selection and configuration, click on **"Execute"** at the top of the screen, or press **`F8`**.
<img width="639" height="690" alt="image" src="https://github.com/user-attachments/assets/f5c2aab0-3b2f-4de4-9334-36cff96a179f" />



---

## Batch Sending Criteria Selection

On the next page, all possible batch selection criteria will appear on the left side. These are columns marked as **"Selection"** in the previous screen.

To configure the criteria:

1. Double-click the field to be set as the sending criterion to display the selection options.
2. Select the criterion for sending the entity.
3. Once criteria are defined, click **"Save"**.
<img width="1363" height="765" alt="image" src="https://github.com/user-attachments/assets/6ed63cbf-0de2-47d8-bac8-eaa551783e02" />


---

## Choosing Execution Mode

Select the execution mode for data sending:

- **Online:** Recommended when the data volume is small, or when the process does not exceed the available SAP user memory.
- **Background:** Recommended for large data volumes to prevent memory issues. Also useful for scheduling execution at a specific date and time.
<img width="860" height="738" alt="image" src="https://github.com/user-attachments/assets/a417ed1a-91bb-4cc4-a276-63928f046450" />


### Online Mode Execution

If executed in Online mode, a summary log will appear at the end, displaying:

| Field | Description |
|---|---|
| **GV_ENTITY** | Should indicate **`ANY`**, meaning the entity that was executed. |
| **TOTAL_SIZE_SENT** | Total bytes sent. |
| **TOTAL_NUMBER_OF_REQUEST** | Number of requests made. |
| **TOTAL_NUMBER_OF_RECORDS** | Number of records sent for the entity. |
<img width="739" height="317" alt="image" src="https://github.com/user-attachments/assets/aae45433-6035-4a45-b625-bbe0b6d9a77e" />

> ✅ Check the destination system to verify that the information was received correctly.
