# Architecture – Golden Record Rollback Framework

## Overview

The **Golden Record Rollback Framework** ensures that when a workflow rejection occurs, the Article record is restored to its **previous valid state** using historical change data.

This architecture is built using:

* **Dataverse**
* **Power Automate Cloud Flows**
* **Office Script (TypeScript)**
* **Dataverse Web API**

The design prioritizes:

* Data integrity
* Transaction safety
* Clear audit traceability
* Scalable field restoration

---

# High-Level Architecture

```
User Rejects Article Task
          │
          ▼
Power Automate Flow Triggered
          │
          ▼
Retrieve Article Change Log Entries
          │
          ▼
Construct Rollback Object
          │
          ▼
Office Script Transformation Layer
          │
          ▼
Dataverse Web API PATCH Request
          │
          ▼
Article Restored to Previous State
```

---

# Core Architectural Components

## 1. Dataverse Tables

### Article Table

The primary entity containing the Golden Record data.

Typical fields include:

* Article Type
* Product Hierarchy
* Logistical Variant Description
* Average Weight
* MOQ
* Batch Expiry Control

This table is updated during the rollback process.

---

### Article Change Log Entry Table

This table stores the **history of field changes**.

Each record represents a single field modification.

| Column             | Purpose                   |
| ------------------ | ------------------------- |
| Article            | Reference to Article      |
| Field Logical Name | Column that changed       |
| Old Value          | Value before modification |
| New Value          | Updated value             |
| Created On         | Timestamp of change       |

During rollback, the **Old Value** is used to restore the Article.

---

# Power Automate Flow Architecture

The rollback flow orchestrates the entire process.

### Flow Responsibilities

1. Detect rejection event
2. Retrieve change history
3. Build rollback object
4. Transform payload
5. Execute Dataverse PATCH

---

## Flow Execution Steps

### Step 1 – Retrieve Change Log Records

The flow retrieves all change log entries associated with the rejected Article.

```
Dataverse Action
List Rows – Article Change Log Entry
```

Filter Query:

```
_mdm_article_value eq <ArticleID>
```

Order By:

```
createdon asc
```

This ensures the **earliest historical value** is used.

---

### Step 2 – Construct Rollback Object

The flow builds a dynamic object representing the record state before modification.

Example object:

```json
{
  "mdm_articletype": "GUID",
  "mdm_batchexpirycontrol": "false",
  "mdm_averageweightunit": "GUID",
  "mdm_averageweight": null
}
```

This object contains all fields that require restoration.

---

### Step 3 – Transformation Layer (Office Script)

The rollback object is passed to an **Office Script written in TypeScript**.

Purpose of the transformation layer:

* Convert lookup values to Dataverse format
* Normalize boolean values
* Preserve null values
* Generate a valid PATCH payload

Example transformation:

Input:

```json
{
  "mdm_articletype": "GUID"
}
```

Output:

```json
{
  "mdm_ArticleType@odata.bind": "/mdm_lookuparticletypes(GUID)"
}
```

---

# Dataverse Web API Integration

The final rollback operation uses the **Dataverse Web API PATCH method**.

Endpoint:

```
/api/data/v9.2/mdm_articles(<ArticleID>)
```

Example request body:

```json
{
  "mdm_ArticleType@odata.bind": "/mdm_lookuparticletypes(GUID)",
  "mdm_batchexpirycontrol": false,
  "mdm_averageweight": null
}
```

This performs a **partial update**, restoring only the modified fields.

---

# Lookup Field Architecture

Lookup fields require a specific syntax when updating through the Web API.

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

This format ensures the lookup relationship is properly restored.

---

# Data Type Handling

The rollback framework handles multiple field types.

| Data Type | Handling Method         |
| --------- | ----------------------- |
| Lookup    | `SchemaName@odata.bind` |
| Boolean   | true / false            |
| Number    | numeric value           |
| Text      | string                  |
| Null      | clears the field        |

This prevents payload validation errors during PATCH operations.

---

# Error Handling Strategy

The architecture includes several error-handling safeguards.

### Flow-Level Handling

* Flow run failure logging
* Conditional validation of payload data

### API-Level Handling

* Dataverse error responses captured
* Retry logic available for transient failures

---

# Performance Considerations

The architecture minimizes system load by:

* Using **single PATCH requests**
* Avoiding repeated record updates
* Selecting only required fields during queries
* Performing payload transformation in a lightweight script

---

# Security Model

API calls to Dataverse use **Azure Active Directory OAuth authentication**.

Security elements include:

* Tenant authentication
* Client application registration
* Secure secret storage
* Role-based access to Dataverse tables

---

# Future Architecture Enhancements

Possible improvements to the framework:

* Metadata-driven lookup mapping
* Automatic datatype detection
* Plugin-based rollback service
* Bulk rollback support
* Monitoring dashboards

---

# Summary

The Golden Record Rollback Framework provides a reliable mechanism to restore data integrity after workflow rejections.

Key architectural benefits:

* Deterministic rollback behavior
* Controlled transformation layer
* Efficient Dataverse updates
* Maintainable and scalable design

