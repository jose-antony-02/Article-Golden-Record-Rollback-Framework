# Office Script – Payload Transformation Layer

## Overview

The **Office Script Transformation Layer** is responsible for converting the rollback object generated in Power Automate into a **Dataverse Web API compatible PATCH payload**.

Dataverse requires specific formats for certain field types (such as lookup fields). This script ensures that the payload conforms to those requirements before sending the update request.

The script is executed using the **Excel Online – Run Script** action in Power Automate.

---

# Purpose of the Script

The Office Script performs the following transformations:

* Converts **lookup GUID values** into the `@odata.bind` format required by Dataverse
* Converts **boolean string values** into proper boolean types
* Preserves **null values** to clear fields correctly
* Passes through standard fields such as text and numeric values

This transformation ensures the payload is valid for the **Dataverse PATCH API**.

---

# Input Payload

The script receives a **JSON string** from Power Automate.

Example input:

```json
{
  "mdm_articletype": "C3870D3D-8D25-F011-8C4D-6045BDF62B3B",
  "mdm_batchexpirycontrol": "false",
  "mdm_averageweightunit": "101f19fe-d420-f011-9989-6045bdf62b3b",
  "mdm_averageweight": null
}
```

This payload represents the **previous state of the Article record** before modification.

---

# Output Payload

The script returns a **PATCH-ready Dataverse payload**.

Example output:

```json
{
  "mdm_ArticleType@odata.bind": "/mdm_lookuparticletypes(C3870D3D-8D25-F011-8C4D-6045BDF62B3B)",
  "mdm_batchexpirycontrol": false,
  "mdm_AverageWeightUnit@odata.bind": "/mdm_lookupaverageweightunits(101f19fe-d420-f011-9989-6045bdf62b3b)",
  "mdm_averageweight": null
}
```

Lookup fields are converted to `@odata.bind`, which Dataverse requires when updating relationships.

---

# Lookup Mapping Configuration

Lookup fields require two pieces of information:

* Schema Name (Navigation Property)
* Target Entity Set Name

These mappings are defined inside the script.

Example configuration:

```typescript
const lookupMap = {
  mdm_articletype: {
    schema: "mdm_ArticleType",
    entitySet: "mdm_lookuparticletypes"
  },
  mdm_averageweightunit: {
    schema: "mdm_AverageWeightUnit",
    entitySet: "mdm_lookupaverageweightunits"
  }
};
```

This allows the script to dynamically convert lookup values.

---

# Office Script Implementation

```typescript
function main(workbook: ExcelScript.Workbook, inputJson: string): string {

  const input: Record<string, string | null> = JSON.parse(inputJson);
  const output: Record<string, string | boolean | number | null> = {};

  const lookupMap: Record<string, { schema: string; entitySet: string }> = {
    mdm_articletype: {
      schema: "mdm_ArticleType",
      entitySet: "mdm_lookuparticletypes"
    },
    mdm_averageweightunit: {
      schema: "mdm_AverageWeightUnit",
      entitySet: "mdm_lookupaverageweightunits"
    }
  };

  for (const field in input) {

    const value: string | null = input[field];

    // Handle lookup fields first
    if (lookupMap[field]) {

      const schema = lookupMap[field].schema;
      const entitySet = lookupMap[field].entitySet;
      const key = `${schema}@odata.bind`;

      if (value === null) {
        output[key] = null;
      } else {
        output[key] = `/${entitySet}(${value})`;
      }

      continue;
    }

    // Handle null values
    if (value === null) {
      output[field] = null;
      continue;
    }

    // Convert boolean strings
    if (value === "true") {
      output[field] = true;
      continue;
    }

    if (value === "false") {
      output[field] = false;
      continue;
    }

    // Default passthrough
    output[field] = value;
  }

  return JSON.stringify(output);
}
```

---

# Flow Integration

The Office Script is called using the **Run Script** action in Power Automate.

### Parameter

| Parameter | Description                         |
| --------- | ----------------------------------- |
| inputJson | Rollback object converted to string |

Power Automate expression:

```
string(variables('varRollbackObject'))
```

---

# Script Output Handling

The script returns a **JSON string**, which must be converted back to an object before sending to Dataverse.

Power Automate expression:

```
json(outputs('Run_script')?['body/result'])
```

This ensures the HTTP action receives a proper JSON payload.

---

# Dataverse PATCH Example

The transformed payload is sent to the Dataverse Web API.

```
PATCH /api/data/v9.2/mdm_articles(<ArticleID>)
```

Example request body:

```json
{
  "mdm_ArticleType@odata.bind": "/mdm_lookuparticletypes(GUID)",
  "mdm_batchexpirycontrol": false,
  "mdm_averageweight": null
}
```

---

# Advantages of This Approach

* Centralized transformation logic
* Simplified Power Automate flow
* Scalable lookup handling
* Reduced expression complexity
* Clean Dataverse API integration

---

# Future Improvements

Possible enhancements to the script include:

* Automatic detection of GUID values
* Metadata-driven lookup mapping
* Automatic number conversion
* Centralized configuration management

---

# Summary

The Office Script serves as a **payload transformation middleware** between Power Automate and Dataverse.

It ensures the rollback payload conforms to Dataverse API requirements while keeping the Power Automate flow simple and maintainable.
