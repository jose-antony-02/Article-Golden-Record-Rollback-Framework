# Golden Record Rollback Framework

## Overview

The **Golden Record Rollback Framework** is designed to safely revert changes made to a Golden Article when a workflow rejection occurs.

When an SCM user rejects a task, the system restores the record to its **previous state before modification** using the change history captured in the **Article Change Log Entry** table.

This ensures:

* Data consistency
* Traceability of changes
* Controlled rollback of business-critical records
* Safe handling of lookup, boolean, numeric, and text fields

---

# Architecture

The rollback process is implemented using **Power Automate + Dataverse Web API + Office Script (TypeScript)**.

The solution follows this pipeline:

```
Article Rejected
      ↓
Retrieve Change Log Entries
      ↓
Build Rollback Object
      ↓
Office Script Transformation
      ↓
Dataverse PATCH API
      ↓
Record Restored
```

---

# Key Components

## 1. Article Change Log Entry Table

This table stores all field-level changes made to an Article.

Typical fields include:

| Field              | Description         |
| ------------------ | ------------------- |
| Article            | Related Article     |
| Field Logical Name | Column modified     |
| Old Value          | Value before change |
| New Value          | Updated value       |
| Created On         | Change timestamp    |

During rollback, the **old values** are used to restore the record.

---

# Rollback Process

## Step 1 — Retrieve Change Logs

Power Automate retrieves all change log entries for the rejected article.

```
List Rows
Table: Article Change Log Entry
Filter: _mdm_article_value eq ArticleID
Order By: createdon asc
```

This ensures the **earliest previous value** is restored.

---

## Step 2 — Build Rollback Object

A rollback object is constructed dynamically inside the flow.

Example:

```json
{
  "mdm_articletype": "GUID",
  "mdm_batchexpirycontrol": "false",
  "mdm_averageweightunit": "GUID",
  "mdm_averageweight": null
}
```

This object represents the **state of the record before modification**.

---

## Step 3 — Transform Payload (Office Script)

The rollback object is passed to an **Office Script (TypeScript)** which prepares it for Dataverse API.

The script:

* Converts lookup fields into `@odata.bind`
* Handles boolean values
* Preserves null values
* Returns a PATCH-ready payload

Example transformation:

### Input

```json
{
  "mdm_articletype": "GUID",
  "mdm_averageweightunit": "GUID"
}
```

### Output

```json
{
  "mdm_ArticleType@odata.bind": "/mdm_lookuparticletypes(GUID)",
  "mdm_AverageWeightUnit@odata.bind": "/mdm_lookupaverageweightunits(GUID)"
}
```

---

# Step 4 — Dataverse PATCH Request

The transformed payload is sent to Dataverse using the Web API.

```
PATCH
/api/data/v9.2/mdm_articles(<ArticleID>)
```

Example body:

```json
{
  "mdm_ArticleType@odata.bind": "/mdm_lookuparticletypes(GUID)",
  "mdm_batchexpirycontrol": false,
  "mdm_averageweight": null
}
```

This restores the article to its previous state.

---

# Lookup Handling

Dataverse lookup fields must use the **navigation property** with `@odata.bind`.

Format:

```
<SchemaName>@odata.bind
```

Example:

```
mdm_ArticleType@odata.bind
```

Value format:

```
/<EntitySetName>(GUID)
```

Example:

```
/mdm_lookuparticletypes(GUID)
```

---

# Handling Special Data Types

| Type    | Handling                |
| ------- | ----------------------- |
| Lookup  | `SchemaName@odata.bind` |
| Boolean | true / false            |
| Number  | numeric value           |
| Text    | string                  |
| Null    | clears field            |

---

# Error Handling

Rollback operations include validation and error handling:

* Flow run failures are logged
* PATCH errors are captured
* Invalid lookup mappings are detected

This ensures rollback failures can be investigated easily.

---

# Benefits of This Approach

* Single PATCH operation (atomic update)
* Handles dynamic field changes
* Scalable for additional fields
* Centralized transformation logic
* Fully auditable rollback process

---

# Future Improvements

Potential enhancements include:

* Automatic lookup detection
* Metadata-driven field mapping
* Plugin-based rollback engine
* Performance optimization for bulk operations

---

# Maintainers

Maintained as part of the **MDM Article Workflow Automation** framework.
