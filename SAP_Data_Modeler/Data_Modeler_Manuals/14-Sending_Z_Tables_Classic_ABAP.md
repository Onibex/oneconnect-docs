# Sending Z Tables from an Internal Table

This manual explains how to send custom **Z tables** through OneConnect using the SAP Data Modeler's **ANY entity** combined with a custom Z program, when the data originates from a **standard ABAP internal table**.

> ℹ️ This procedure is similar to [Sending Z Tables via SAP Data Modeler and Custom Z Program](./13-Sending_Z_Tables_ABAP_RAP.md), but uses the `send_json_any_internal_table` method instead of `send_json_any_ltable_rap`. Use this approach when your data comes from a classic ABAP internal table rather than a RAP-based source.

---

## Step 1: Create an ANY Entity with the Z Tables

In the SAP Data Modeler, create an **ANY** entity that includes the desired Z tables.
<img width="1001" height="399" alt="image" src="https://github.com/user-attachments/assets/96093e27-8117-4642-b6f5-defecd658130" />
<img width="811" height="346" alt="image" src="https://github.com/user-attachments/assets/73783f39-706a-498e-8618-1af1028b8b61" />

> ⚠️ **Important:** Ensure the **Domain** is set to **`ANY`**.

### Select the Columns

Inside the entity, select the desired columns to send in the **Column Definition** section, then save the new entity.
<img width="1033" height="257" alt="image" src="https://github.com/user-attachments/assets/634fbee8-d692-4cfb-bf68-862f216ac9a2" />

---

## Step 2: Locate the Custom Z Program

In transaction **`SE38`**, locate the Z program provided by Onibex: **`Z_PROGRAM_CUD_ZTABLES`**.
<img width="983" height="503" alt="image" src="https://github.com/user-attachments/assets/623ff951-2cf7-44dc-b8f8-2b7b59368fd6" />

In **Display/Change** mode, review the program logic to confirm it sends the appropriate Z tables.
<img width="1359" height="733" alt="image" src="https://github.com/user-attachments/assets/fd712705-f4d0-4958-b735-8672da64edaa" />

> ℹ️ Update this Z program to include the necessary logic to handle the tables specific to your implementation. This ensures the program reacts to **create**, **update**, and **delete** events on the Z tables.

---

## Step 3: Configure the Operation Type

To send Z tables, use the following structure depending on the operation type:

| Operation | `iv_update` | `iv_delete` |
|---|---|---|
| **Create or Update** | `abap_true` | `abap_false` |
| **Delete** | `abap_false` | `abap_true` |

You must also set the **`iv_entity_business_proc`** parameter to the name of your ANY entity.

---

## Example ABAP Code for Sending Z Tables from Internal Tables

The example below sends a sales order structure composed of a **HEADER** table (`ZONTA_SO_H`) and an **ITEMS** table (`ZONTA_SO_ITEM`) from internal tables.

### Constants

```abap
"----------------------------------------------------------------------*
"*  CONSTANTS                                                          *
"----------------------------------------------------------------------*
CONSTANTS:
  gc_aliastab_header TYPE zonta_oc_col_all-tabname     VALUE 'HEADER',
  gc_aliastab_items  TYPE zonta_oc_col_all-tabname     VALUE 'ITEMS',
  gc_tabname_hd      TYPE zonta_oc_col_all-tabname     VALUE 'ZONTA_SO_H',
  gc_tabname_items   TYPE zonta_oc_col_all-tabname     VALUE 'ZONTA_SO_ITEM',
  gc_entity          TYPE zonta_obj_oc-business_proc   VALUE 'ANY_ZTABLE_AUTOMATIC'.
```

### Sending the HEADER Table

```abap
"----------------------------------------------------------------------*
"*  Send HEADER table                                                  *
"----------------------------------------------------------------------*
NEW zoncl_fetch_data( )->send_json_any_internal_table(
  iv_tabname              = gc_tabname_hd
  iv_aliastab             = gc_aliastab_header     " Table Alias
  iv_update               = abap_true
  iv_delete               = abap_false
  it_tables_data          = gt_data_hd
  iv_alias                = abap_false
  iv_fieldname            = abap_true
  iv_entity_business_proc = gc_entity
* iv_dest                 = 'ONIBEX_DEMO'
* iv_bothnames            = abap_false
).
```

### Sending the ITEMS Table

```abap
"----------------------------------------------------------------------*
"*  Send ITEMS table                                                   *
"----------------------------------------------------------------------*
NEW zoncl_fetch_data( )->send_json_any_internal_table(
  iv_tabname              = gc_tabname_items
  iv_aliastab             = gc_aliastab_items     " Table Alias
  iv_update               = abap_true
  iv_delete               = abap_false
  it_tables_data          = gt_data_item
  iv_alias                = abap_false
  iv_fieldname            = abap_true
  iv_entity_business_proc = gc_entity
* iv_dest                 = 'ONIBEX_DEMO'
* iv_bothnames            = abap_false
).
```

> ℹ️ **Note about commented parameters:** The `iv_dest` and `iv_bothnames` parameters are commented out (with `*`) in this example, meaning they will use the default values. Uncomment and configure them only if you need to override the defaults.

---

## Parameter Reference

| Parameter | Description |
|---|---|
| **`iv_tabname`** | Technical name of the Z table being sent. |
| **`iv_aliastab`** | Table alias used to identify the table in the entity (e.g., `HEADER`, `ITEMS`). |
| **`iv_update`** | Set to `abap_true` for create or update operations. |
| **`iv_delete`** | Set to `abap_true` for delete operations. |
| **`it_tables_data`** | Internal table containing the data to be sent. |
| **`iv_alias`** | Set to `abap_true` to send alias names; `abap_false` to send technical names. |
| **`iv_fieldname`** | Set to `abap_true` to send field names. |
| **`iv_entity_business_proc`** | Name of the ANY entity in the SAP Data Modeler. |
| **`iv_dest`** *(optional)* | RFC destination defined in transaction `SM59`. If omitted, uses the default destination. |
| **`iv_bothnames`** *(optional)* | Set to `abap_true` to send both technical name and alias concatenated. |

> ℹ️ **Operation type reference:** When sending a create or update operation, pass `iv_update = abap_true` and `iv_delete = abap_false`. For delete operations, pass `iv_update = abap_false` and `iv_delete = abap_true`. The `iv_entity_business_proc` parameter must always be set to the name of your entity.
