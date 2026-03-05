# Flow Design – Golden Record Rollback Framework

## Overview

This document explains the **Power Automate Cloud Flow design** used to implement the Golden Record rollback process.

The flow is responsible for detecting a rejection event and restoring the **Article record** to its previous state using the **Article Change Log Entry** table.

The flow performs the following main tasks:

* Detect rejection events
* Retrieve change history
* Construct rollback payload
* Transform lookup values
* Execute Dataverse PATCH request

---

# Flow Trigger

The flow starts when an **Article Task is rejected by an SCM user**.

Typical trigger source:

```id="gt8v3p"
Dataverse – When a row is added or modified
```

Trigger Table:

```id="u14m7e"
mdm_pmtasks
```

Trigger Condition Example:

```id="g88v5u"
mdm_actioncode eq 'Reject'
```

When the rejection occurs, the related **Article ID** is extracted from the task.

---

# Flow Architecture

```id="m0d3k1"
Trigger (Task Rejected)
        │
        ▼
Get Article Record
        │
        ▼
Check Golden Record Flag
        │
        ├── Non-Golden Record
        │        ▼
        │     Deactivate Article
        │
        ▼
Golden Record
        │
        ▼
Retrieve Change Log Entries
        │
        ▼
Build Rollback Object
        │
        ▼
Office Script Transformation
        │
        ▼
Dataverse PATCH Update
        │
        ▼
Rollback Completed
```

---

# Step-by-Step Flow Logic

## Step 1 – Retrieve Article Record

The flow retrieves the Article associated with the rejected task.

```id="pfp4n7"
Dataverse – Get a Row by ID
Table: mdm_articles
```

This step retrieves:

* Golden Record flag
* Article identifiers
* Required metadata for rollback.

---

# Step 2 – Determine Record Type

A condition checks whether the record is a **Golden Record**.

Condition:

```id="j2xq5d"
mdm_goldenflag = true
```

### Non-Golden Record

If the record is **not golden**, the flow performs a simple action:

```id="r4z9sq"
Update Row
statecode = Inactive
```

This deactivates the record without rollback.

---

### Golden Record

If the record **is golden**, the rollback logic begins.

---

# Step 3 – Retrieve Change Log Entries

The flow retrieves all change logs associated with the article.

```id="7szxgn"
Dataverse – List Rows
Table: mdm_articlechangelogentries
```

Filter Query:

```id="lpr0vc"
_mdm_article_value eq <ArticleID>
```

Order By:

```id="m4gb9g"
createdon asc
```

Sorting by creation date ensures the **original field value** is restored.

---

# Step 4 – Construct Rollback Object

The flow builds a dynamic JSON object representing the previous record state.

This object is constructed using:

```id="sq84s3"
Apply to Each
```

Each change log entry contributes:

* Field Logical Name
* Old Value

Example rollback object:

```json id="81m2za"
{
  "mdm_articletype": "GUID",
  "mdm_batchexpirycontrol": "false",
  "mdm_averageweightunit": "GUID",
  "mdm_averageweight": null
}
```

This object represents the state before modification.

---

# Step 5 – Transform Payload Using Office Script

The rollback object is sent to an **Office Script** to prepare it for Dataverse.

```id="tw8trd"
Excel Online – Run Script
```

The script performs:

* Lookup transformation
* Boolean normalization
* Null preservation
* Dataverse API formatting

Example transformation:

Input:

```json id="6f4v2m"
{
  "mdm_articletype": "GUID"
}
```

Output:

```json id="c9kgjq"
{
  "mdm_ArticleType@odata.bind": "/mdm_lookuparticletypes(GUID)"
}
```

---

# Step 6 – Execute Dataverse PATCH

The transformed payload is sent to Dataverse using an HTTP request.

```id="p5e5p0"
HTTP Action
Method: PATCH
```

Endpoint:

```id="ktjz70"
https://<environment>.crm.dynamics.com/api/data/v9.2/mdm_articles(<ArticleID>)
```

Headers:

```id="0y5ycm"
Content-Type: application/json
Accept: application/json
```

Body:

```id="5m6gk4"
json(outputs('Run_script')?['body/result'])
```

This updates the record with the restored values.

---

# Flow Variables

The flow uses variables to construct the rollback payload.

### Example Variables

| Variable           | Type   | Purpose                         |
| ------------------ | ------ | ------------------------------- |
| varRollbackObject  | Object | Stores rollback payload         |
| varProcessedFields | Array  | Tracks processed fields         |
| varTempObject      | Object | Temporary transformation object |

These variables ensure that only the **earliest change entry per field** is used.

---

# Concurrency Control

The **Apply to Each loop runs sequentially**.

Reason:

* Prevent race conditions
* Maintain deterministic rollback
* Ensure correct field ordering

Concurrency settings:

```id="62pqv6"
Concurrency Control: Disabled
```

---

# Error Handling

The flow includes error-handling mechanisms.

### Scope Structure

```id="cl48r0"
Scope – Try
Scope – Catch
```

If an error occurs:

* Flow run logs capture the issue
* Error notifications can be triggered

---

# Performance Optimization

The flow is optimized by:

* Selecting only required columns
* Using a single PATCH request
* Avoiding repeated updates
* Performing transformation in Office Script

These improvements reduce API usage and improve execution speed.

---

# Summary

The rollback flow ensures that Golden Records are safely restored when workflow rejection occurs.

Key benefits:

* Deterministic rollback behavior
* Minimal Dataverse API usage
* Scalable architecture
* Maintainable flow design
* Strong audit traceability

