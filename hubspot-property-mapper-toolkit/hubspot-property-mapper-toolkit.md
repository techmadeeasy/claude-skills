# HubSpot Property Mapper Toolkit

Given a structured mapping file, this skill provisions custom HubSpot properties across one or more object types. It checks for existing properties before creating, making every run safe to re-run against any portal.

## Prerequisites

Before starting, confirm both of the following are in place:

1. **HubSpot MCP is configured and authenticated** for the target portal. All read operations (checking existing properties) go through the MCP. If `get_properties` returns an auth error or the MCP is not connected, stop and ask the user to authenticate before proceeding.
2. **A mapping file** has been provided (path or content). If not, ask the user for it before doing anything else.

Verify MCP connectivity by calling `get_organization_details` — it confirms which portal you are operating against and surfaces the portal name and ID so the user can confirm it is the right one.

## Mapping file format

The mapping file is a JSON file with HubSpot object types as top-level keys:

```json
{
  "contacts": [ ... ],
  "companies": [ ... ],
  "deals": [ ... ],
  "tickets": [ ... ],
  "line_items": [ ... ]
}
```

Each property entry in an array follows HubSpot's property definition schema:

```json
{
  "name": "internal_property_name",
  "label": "Display Label",
  "type": "string",
  "fieldType": "text",
  "groupName": "contactinformation",
  "description": "Optional description",
  "options": [
    { "label": "Option A", "value": "option_a", "displayOrder": 0 }
  ]
}
```

**Required fields:** `name`, `label`, `type`, `fieldType`, `groupName`.  
**`options`** is required only when `type` is `enumeration`.

Valid `type` values: `string`, `number`, `date`, `datetime`, `enumeration`, `bool`, `phone_number`.  
Valid `fieldType` values: `text`, `textarea`, `number`, `date`, `file`, `checkbox`, `booleancheckbox`, `radio`, `select`, `phonenumber`.

## Workflow

### Step 1 — Validate the mapping file
- Parse and validate the JSON.
- For each property entry, confirm all required fields are present and `type`/`fieldType` combinations are valid.
- If `type` is `enumeration` and `options` is missing or empty, flag it as an error.
- Report all validation errors upfront and stop — do not proceed with a broken mapping file.

### Step 2 — Confirm portal identity
- Call `get_organization_details` via MCP.
- Output the portal name and ID and ask the user to confirm this is the correct portal before making any changes.

### Step 3 — Check existing properties (via HubSpot MCP)
For each object type in the mapping file:
- Call `get_properties` via MCP to retrieve all existing properties for that object type.
- Build a set of existing internal property `name` values.
- Diff against the mapping file entries to produce two lists:
  - **To create** — properties in the mapping file not found on the portal.
  - **Already exists** — properties already present (will be skipped).

Show the user this diff and confirm before proceeding to creation.

### Step 4 — Create missing properties
For each property in the **to create** list:
- Use the MCP or the HubSpot Properties API (`POST /crm/v3/properties/{objectType}`) to create the property.
- Treat a `409 Conflict` response as a no-op (property exists but was not in the MCP results) — log it as skipped.
- On any other error, log it and continue with the remaining properties — do not abort the entire run.

### Step 5 — Report results
Output a summary table when done:

| Object | Property name | Status |
|--------|--------------|--------|
| contacts | my_custom_field | created |
| contacts | existing_field | skipped (already exists) |
| deals | deal_stage_reason | created |
| deals | bad_fieldtype | failed — invalid fieldType |

If any properties failed, list them at the end with the error message so the user can fix and retry.

## Property naming conventions
- Use `snake_case` for internal `name` values.
- Never use the `hs_` prefix — it is reserved for HubSpot system properties.
- `groupName` must reference a property group that **already exists** on the portal. Do not assume groups are auto-created; if a group is missing, flag it before attempting creation.

## Re-runs and idempotency
Running the skill more than once against the same portal and mapping file is safe. Already-existing properties are always skipped, never overwritten. If you need to update an existing property's label, description, or options, that is out of scope for this skill — use the HubSpot UI or a separate update workflow.
