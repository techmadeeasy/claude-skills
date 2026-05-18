---
description: Generate a HubSpot integration toolkit from a data mapping spreadsheet — Postman collection, custom coded workflow action files, and webhook JSON schema. Works with any HubSpot portal. Optionally validates and creates/fixes properties against a live portal when a token is provided.
argument-hint: "[path/to/mapping.xlsx]"
---

# HubSpot Integration Toolkit

You are executing the `/hubspot-integration` skill. Your job is to scaffold a Node.js toolkit in the current project directory and drive it to generate all integration artifacts from a HubSpot data mapping spreadsheet.

**Do not hardcode anything client-specific.** Every company name, portal ID, property name, and domain value must come from user input or the spreadsheet.

---

## Step 0 — Load Project History

Before asking any questions, check for a history file at `.hubspot-integration/history.json` in the current working directory.

**If the file does not exist:** proceed silently to Step 1. This is a fresh project.

**If the file exists:** load it and present a session briefing before asking anything else. Format:

```
── Integration History ────────────────────────────────
Project : {project}
Portal  : {portal_name} (ID: {portal_id})
Sessions: {count} prior session(s), last on {last_updated}

Last session summary:
  {last session summary line}

Pending items carried forward:
  ⚠️  {object} — {issue}
  ⚠️  {object} — {issue}

Property changes on record:
  {object}.{property} — {last event} on {date} {note if side effects}
──────────────────────────────────────────────────────
```

Then ask: **"Continue from where this project left off, or start fresh?"**
- Continue → carry the existing config forward; skip any Step 1 questions already answered in history
- Fresh → proceed to Step 1 as normal (history file will be archived, not deleted)

**Consequence check on load:** scan the pending items and property registry. If any pending items reference properties the user is about to act on in this session, surface them proactively before those actions are taken.

---

## Step 1 — Gather Requirements (one message, wait for answers)

Before writing any file, ask the user all of the following in a single numbered list. If `$ARGUMENTS` is non-empty, treat it as the spreadsheet path and skip Q1.

1. **Spreadsheet path** — path to the data mapping Excel file (required)

2. **HubSpot private app token** — optional. Three modes depending on what is available:
   - **No token, no MCP**: generate artifacts from spreadsheet only — Postman CREATE requests may fail if properties don't exist yet
   - **Token only**: validate/create/fix properties via Node.js scripts, then generate artifacts
   - **Token + HubSpot MCP** _(best)_: use MCP tools to read portal state directly in the conversation, scripts for write operations — most interactive and reliable

   If a token is provided, also ask: **Is the HubSpot MCP connected in this session?** (The user should have connected it via claude.ai integrations or MCP config before invoking this skill.)

3. **Project / portal name** — used to name output files (e.g. "Acme Corp" → `Acme_Corp_Master_Collection.postman_collection.json`)

4. **Object types in the spreadsheet** — confirm which objects appear in column B:
   - Standard: `Company`, `Deal`, `Product`
   - Custom objects: provide the display label used in col B **and** the HubSpot API type slug (e.g. label "Order" → slug `p12345_orders`)

5. **Find keys** — which property uniquely identifies each object (used in FIND/search requests)?
   - Company → default suggestion: `customernumber`
   - Deal → default suggestion: `prospect_number`
   - Product → default suggestion: ask
   - Custom objects → must specify
   - Reply with the actual internal name or "default" to accept suggestions

6. **Workflow scope** — which objects need workflow action files?
   - Note: product-type objects (Products, Line Items) **cannot be enrolled in HubSpot workflows** — they are ERP → HubSpot only via Postman. Confirm whether to skip or generate with a warning comment.

7. **External endpoint style** — how should ERP endpoints be referenced in workflow files?
   - (A) One env var per object: `ERP_COMPANY_ENDPOINT`, `ERP_DEAL_ENDPOINT`, etc. ← recommended
   - (B) Single base URL: `ERP_BASE_URL` + path suffix per object

8. **Output directory** — default: current directory with subfolders `postman/`, `workflows/`, `schemas/`

9. **Company tiers** — does this portal use a tiered Company structure (e.g. MasterChain → Chain → Stop → Department)? If yes, list tier names in order. These become the `enum` values for a `source_object_name` routing field injected into Company payloads. If no, omit `source_object_name` entirely.

**Wait for answers before proceeding.**

---

## Step 2 — Confirm Object Config

Build the object config table from the user's answers and echo it back for confirmation:

```
| Object Label | API Slug        | Find Key          | Workflow? |
|--------------|-----------------|-------------------|-----------|
| Company      | companies       | customernumber    | yes       |
| Deal         | deals           | prospect_number   | yes       |
| Product      | products        | item_number       | no (note) |
| ...custom... | p12345_xxxxx    | ...               | yes/no    |
```

Also derive the **project slug**: lowercase the project name, replace spaces and special characters with underscores (e.g. "Acme Foods" → `acme_foods`). Show it to the user.

Wait for the user to confirm before writing any code.

---

## Step 2b — Portal Recon (token + MCP mode only)

If the user confirmed both a token and an active HubSpot MCP connection, use the MCP tools to inspect the portal **before** scaffolding any scripts. This gives you ground truth about what already exists and avoids unnecessary script runs.

**Run these MCP lookups:**

1. `mcp__*_HubSpot__get_organization_details` — confirm you are connected to the correct portal (name, portal ID). Show the portal name to the user and ask them to confirm before proceeding.

2. For each object type in the config table, call `mcp__*_HubSpot__get_properties` with the API slug — get the full list of existing properties.

3. Cross-reference the spreadsheet mappings against the MCP results:
   - **Exists and type matches** → MATCH ✓
   - **Exists but wrong type** → TYPE_MISMATCH ⚠️
   - **Does not exist** → MISSING ✗

4. Present the results as a per-object summary table before touching anything:
   ```
   Company — 19 spreadsheet properties
     ✓ 16 matched
     ⚠️  2 type mismatch  (will delete + recreate — existing data will be lost)
     ✗  1 missing         (will be created)
   ```

5. For TYPE_MISMATCH entries: warn the user explicitly that fixing them **permanently deletes the existing property data** in the portal. Require a `yes` before proceeding.

6. After the user confirms, use `mcp__*_HubSpot__search_properties` to verify individual property details where needed, then proceed to scaffolding and running the validate-properties.js scripts for write operations.

**If MCP is not connected** (token only): skip this step entirely and proceed to Step 3. The validate-properties.js script will handle portal inspection via the API.

---

## Step 3 — Scaffold the Toolkit

Check for an existing `src/` directory. If scripts from a previous run already exist, read them before deciding what to rewrite vs reuse. Otherwise create the following from scratch.

### package.json

```json
{
  "name": "{project-slug}-hubspot-integration",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "generate:postman":   "node src/generate-postman.js -s \"{spreadsheetFilename}\" -o postman",
    "generate:workflows": "node src/generate-workflows.js -s \"{spreadsheetFilename}\" -o workflows",
    "generate:schema":    "node src/generate-webhook-schema.js -s \"{spreadsheetFilename}\" -o schemas",
    "generate:all":       "npm run generate:postman && npm run generate:schema && npm run generate:workflows",
    "validate":           "node src/validate-properties.js -s \"{spreadsheetFilename}\"",
    "validate:create":    "node src/validate-properties.js -s \"{spreadsheetFilename}\" --create",
    "validate:fix":       "node src/validate-properties.js -s \"{spreadsheetFilename}\" --fix-mismatched"
  },
  "dependencies": {
    "chalk":     "^5.3.0",
    "commander": "^12.0.0",
    "uuid":      "^9.0.0",
    "xlsx":      "^0.18.5"
  },
  "optionalDependencies": {
    "@hubspot/api-client": "^12.0.0"
  }
}
```

Install dependencies (run `npm install`; add `@hubspot/api-client` only if a token was provided).

### .env.example

```
HUBSPOT_ACCESS_TOKEN=pat-na1-xxxxx
```

---

### src/lib/constants.js

```js
export const FIELD_TYPE_MAP = {
  'Single-line text':    { type: 'string',      fieldType: 'text' },
  'Multi-line text':     { type: 'string',      fieldType: 'textarea' },
  'Number':              { type: 'number',      fieldType: 'number' },
  'Date picker':         { type: 'date',        fieldType: 'date' },
  'Dropdown select':     { type: 'enumeration', fieldType: 'select' },
  'Phone number':        { type: 'string',      fieldType: 'phonenumber' },
  'HubSpot user':        { type: 'enumeration', fieldType: 'select' },
  'Single checkbox':     { type: 'bool',        fieldType: 'booleancheckbox' },
  'Multiple checkboxes': { type: 'enumeration', fieldType: 'checkbox' },
};

// Used for type comparison against the live portal — NOT the same as FIELD_TYPE_MAP.
// Critical: HubSpot stores phone number properties with type 'string', not 'phone_number'.
// Using 'phone_number' here causes false type-mismatch errors on every phone field.
export const TYPE_COMPARISON_MAP = {
  'Single-line text':    'string',
  'Multi-line text':     'string',
  'Number':              'number',
  'Date picker':         'date',
  'Dropdown select':     'enumeration',
  'Phone number':        'string',
  'HubSpot user':        'enumeration',
  'Single checkbox':     'bool',
  'Multiple checkboxes': 'enumeration',
};

export const DEFAULT_GROUP = {
  companies: 'companyinformation',
  deals:     'dealinformation',
  products:  'productinformation',
};
```

---

### src/lib/parse-spreadsheet.js

Reads the "Ready to Build" sheet (ask the user if the sheet name is different). Row 0 = headers, data from row 1.

**Column mapping:**

| Index | Content |
|-------|---------|
| 1 (B) | HubSpot Object label |
| 11 (L) | HubSpot Property Label (display name) |
| **12 (M)** | **HubSpot Property Internal Name — REQUIRED. Skip entire row if blank.** |
| 13 (N) | Field Type |
| 14 (O) | Number format |
| 15 (P) | Pipe-separated dropdown options (e.g. `OPTION_A|OPTION_B`) |

The parser accepts a `filePath` and an `objectLabelMap` (e.g. `{ 'Company': 'companies', 'Deal': 'deals' }`). This is what makes it generic — no hardcoded object names.

Skip a row if:
- Col B is blank or not in `objectLabelMap`
- Col M (internal name) is blank ← most important skip rule

Return `{ [apiSlug]: PropertyMapping[] }`.

---

### src/generate-postman.js

Produces **one collection file** with one folder per object type, each with 4 requests: FIND, GET, CREATE, UPDATE.

**Collection structure:**
- `info.name`: `{Project Name} — HubSpot Integration Master Collection`
- `auth`: bearer using `{{hubspot_access_token}}`
- `variable`: `hubspot_access_token` (empty), `base_url` (`https://api.hubapi.com`), `object_id` (example placeholder)

**4 requests per folder:**

| Request | Method | URL |
|---------|--------|-----|
| FIND | POST | `{{base_url}}/crm/v3/objects/{apiSlug}/search` |
| GET | GET | `{{base_url}}/crm/v3/objects/{apiSlug}/{{object_id}}?properties={all}` |
| CREATE | POST | `{{base_url}}/crm/v3/objects/{apiSlug}` |
| UPDATE | PATCH | `{{base_url}}/crm/v3/objects/{apiSlug}/{{object_id}}` |

FIND body: `{ filterGroups: [{ filters: [{ propertyName: findKey, operator: 'EQ', value: <dummy> }] }], properties: [all prop names] }`

CREATE/UPDATE body: `{ properties: { ...all props with dummy values } }`

**`hubspot_owner_id` must never appear in CREATE or UPDATE bodies** — it is read-only in sync contexts and will cause API rejections.

**Dummy value generation** — derive from field type and label, never hardcode client-specific values:
- `Number` + label contains "price"/"cost"/"amount" → `10.99`
- `Number` + label contains "qty"/"quantity"/"count" → `100`
- `Number` (other) → `0`
- `Date picker` → `'2026-01-15'`
- `Single checkbox` → `'true'`
- `Phone number` → `'555-000-0100'`
- `HubSpot user` → `'12345678'`
- `Dropdown select` / `Multiple checkboxes` → `prop.options?.[0] ?? 'OPTION_1'`
- Text → `'Sample {label}'`

**Company only:** If tiers were provided, inject `source_object_name: firstTierValue` into CREATE/UPDATE bodies. This routing field is not in the spreadsheet.

**Product folder:** Add to the folder description:
```
NOTE: Products are managed ERP → HubSpot direction only via this collection.
HubSpot does not support enrolling Products in workflows.
```

**Pending dropdowns warning** (for any dropdown with no options defined):
```
⚠️ PENDING CLIENT INPUT — the following dropdown properties have no option values defined yet:
  - property_name
These must be populated before middleware can validate values.
```

**sanitizeName** — apply to every `internalName` before use:
```js
function sanitizeName(name) {
  return name.toLowerCase()
    .replace(/[^a-z0-9_]/g, '_')
    .replace(/_+/g, '_')
    .replace(/^_|_$/g, '');
}
```

Output: `{outputDir}/postman/{ProjectSlug}_Master_Collection.postman_collection.json`

---

### src/generate-workflows.js

Generates one `.js` file per workflow-enabled object type.

**Critical:** Workflow files must use **CommonJS syntax** — HubSpot's custom coded action runtime requires `require()` and `exports.main`. Do not use `import`/`export` in these output files (the generator itself uses ES modules; the generated files do not).

**Template for each workflow file:**

```js
// HubSpot Custom Coded Workflow Action — {ObjectLabel} → ERP
//
// Trigger  : {see per-object rules}
// Action   : Fetches all mapped properties and POSTs to ERP/middleware
//
// Secrets to configure in HubSpot workflow editor:
//   PRIVATE_APP_ACCESS_TOKEN  — HubSpot Private App token (CRM read access)
//   ERP_{OBJECT_UPPER}_ENDPOINT — Endpoint URL for {ObjectLabel} events
//
// Input fields: {see per-object rules}

const hubspot = require('@hubspot/api-client');
const axios   = require('axios');

const PROPERTIES = {JSON.stringify(props, null, 2)};

exports.main = async (event, callback) => {
  const objectId = event.object.objectId;
  // For Deal only: const eventType = event.inputFields['event_type'] || 'deal.updated';

  const hubspotClient = new hubspot.Client({
    accessToken: process.env.PRIVATE_APP_ACCESS_TOKEN,
  });

  const record = await hubspotClient.crm.{apiMethod}.basicApi.getById(objectId, PROPERTIES);

  const payload = {
    event_type:        '{event_type_value}',
    hubspot_object_id: String(objectId),
    timestamp:         new Date().toISOString(),
    properties:        record.properties,
  };

  const response = await axios.post(
    process.env.ERP_{OBJECT_UPPER}_ENDPOINT,
    payload,
    { headers: { 'Content-Type': 'application/json' }, timeout: 15000 }
  );

  callback({
    outputFields: {
      hs_execution_state: 'SUCCESS',
      status_code:        String(response.status),
    },
  });
};
```

**Per-object specifics:**

**Company:**
- Trigger comment: `Company-based workflow (e.g. "Company is created" or property change)`
- `event_type`: `'customer.created'`
- If tiers configured: add `source_object_name: record.properties.source_object_name || null` as a top-level payload field; inject `'source_object_name'` into `PROPERTIES` if not already present
- API method: `hubspotClient.crm.companies.basicApi.getById`

**Deal:**
- **This file powers TWO workflows.** Header comment must say:
  ```
  // Trigger  : Create TWO workflows in HubSpot using this same code:
  //              1. "Deal is created"  → set input field event_type = deal.created
  //              2. "Deal is updated"  → set input field event_type = deal.updated
  // Input fields to configure:
  //   event_type (string) — "deal.created" or "deal.updated"
  ```
- `event_type`: `event.inputFields['event_type'] || 'deal.updated'`
- API method: `hubspotClient.crm.deals.basicApi.getById`

**Product (if generated):**
- Add prominent warning:
  ```
  // ⚠️  NOTE: HubSpot Products cannot be enrolled in standard workflows.
  // This file is for reference only. Sync is ERP → HubSpot via Postman.
  ```
- `event_type`: `'product.upserted'`
- API method: `hubspotClient.crm.products.basicApi.getById`

**Custom objects:**
- `event_type`: `'{objectLabel}.upserted'` (lowercase)
- API method: `hubspotClient.crm.objects.basicApi.getById(objectTypeId, objectId, PROPERTIES)` — generic CRM objects API

Output: `{outputDir}/workflows/{objectLabel}-to-erp.js`

---

### src/generate-webhook-schema.js

Generates a single JSON Schema draft-07 file for all workflow-enabled object payloads.

**Field type → JSON Schema type:**

| Field Type | JSON Schema |
|------------|-------------|
| Single-line text / Multi-line text / Phone number / HubSpot user | `{ "type": "string" }` |
| Number | `{ "type": "number" }` |
| Date picker | `{ "type": "string", "format": "date" }` |
| Dropdown select (with options) | `{ "type": "string", "enum": [...] }` |
| Dropdown select (no options) | `{ "type": "string" }` |
| Single checkbox | `{ "type": "boolean" }` |
| Multiple checkboxes | `{ "type": "string" }` (HubSpot returns semicolon-separated string) |

Output: `{outputDir}/schemas/{ProjectSlug}_Webhook_Schema.json`

---

### src/validate-properties.js (only if token mode)

Flags: `--create` (create missing), `--fix-mismatched` (delete then recreate type-mismatched)

Guard at startup:
```js
if (!process.env.HUBSPOT_ACCESS_TOKEN) {
  console.error('HUBSPOT_ACCESS_TOKEN not set. Run: export HUBSPOT_ACCESS_TOKEN="pat-na1-..."');
  process.exit(1);
}
```

**Type comparison:** Use `TYPE_COMPARISON_MAP`, not `FIELD_TYPE_MAP`. Critical for phone number properties — HubSpot stores them with `type: 'string'`, not `type: 'phone_number'`.

**HubSpot SDK error parsing** — the SDK packs errors into `e.message` as a multi-line string, not a structured object:
```js
function parseHubSpotError(e) {
  if (e.response?.body && typeof e.response.body === 'object') return e.response.body;
  try {
    const msg = e.message || '';
    const bodyStart = msg.indexOf('Body: ');
    if (bodyStart === -1) return null;
    const afterBody = msg.slice(bodyStart + 6);
    const headersEnd = afterBody.indexOf('\nHeaders:');
    const jsonStr = headersEnd !== -1 ? afterBody.slice(0, headersEnd) : afterBody;
    return JSON.parse(jsonStr.trim());
  } catch {}
  return null;
}
```

**Calculated property blocking** — when deleting a property that feeds a calculated property, HubSpot returns subCategory `PropertyValidationError.CANNOT_DELETE_PROPERTY_IN_USE`. The blocking calculated property name is in `body.errors[0].context.parentName[0]` (format: `"0-2/calc_prop_name"`). Archive the calculated property first, then re-archive the source property, then recreate with the correct type. Log a warning that the calculated property has been permanently removed.

**sanitizeName must be applied** to every property internal name before creation or lookup. HubSpot rejects names with uppercase letters or characters outside `[a-z0-9_]`.

---

## Step 4 — Run Generators

```bash
node src/generate-postman.js -s "{spreadsheetPath}" -o "{outputDir}/postman"
node src/generate-webhook-schema.js -s "{spreadsheetPath}" -o "{outputDir}/schemas"
node src/generate-workflows.js -s "{spreadsheetPath}" -o "{outputDir}/workflows"
```

Show output after each step. Common failure causes:
- Sheet name is not "Ready to Build" → ask the user for the correct sheet name
- All rows have blank col M → the object has no internal names mapped yet; generate placeholder files and note what needs filling in
- Package not found → run `npm install` again

---

## Step 5 — Validate Against Portal (token mode only)

```bash
export HUBSPOT_ACCESS_TOKEN="{token}"
node src/validate-properties.js -s "{spreadsheetPath}"
```

Show a per-object summary: matched / mismatched / missing counts.

**Before running --create:** confirm with user. Missing properties will be created.

**Before running --fix-mismatched:** cross-check each mismatched property against the history file's `property_registry` **before** warning the user. If the property has a prior `fixed` event in history, escalate the warning:

```
⚠️  HISTORY ALERT — {object}.{property_name}
    This property was already fixed on {date} (changed from {fromType} → {toType}).
    A calculated property '{name}' was permanently deleted as a side effect at that time.
    Fixing it again will delete all data stored in this property since {date}.
    Are you sure you want to proceed?
```

If no prior history: use the standard warning that fixing deletes existing property data. Require explicit confirmation either way. Then run:
```bash
node src/validate-properties.js -s "{spreadsheetPath}" --fix-mismatched
```

**Before adding a property that appears in pending items:** confirm with the user that it's intentional and matches what was deferred in the last session. Example:
```
ℹ️  HISTORY NOTE — {object}.{property_name} was flagged as pending in the last session
    ({reason}). This action will resolve that pending item — confirm?
```

After any create/fix pass, re-run validation to confirm.

---

## Step 6 — Final Report

Report:

1. All generated files with relative paths
2. Per-object summary: property count, find key, workflow generated
3. Pending items: dropdowns with no options (need client input before middleware can validate)
4. No-token caveat if applicable
5. Workflow setup checklist per object:

```
{ObjectLabel} Workflow Setup:
  Secrets: PRIVATE_APP_ACCESS_TOKEN, ERP_{OBJECT_UPPER}_ENDPOINT
  Trigger: {from file header}
  Output fields: hs_execution_state, status_code
```

For Deal: remind the user to create **two separate workflows** in HubSpot using the single deal-to-erp.js file — one triggered by "Deal is created" and one by "Deal is updated", each with the `event_type` input field set accordingly.

---

## Step 7 — Update Project History

After every session — whether artifacts were generated, properties were changed, or just a validation was run — write or update `.hubspot-integration/history.json`.

Create the `.hubspot-integration/` directory if it does not exist. Never delete the history file; only append and update.

### History file schema

```json
{
  "project": "project_slug",
  "portal_id": "12345678",
  "portal_name": "Acme Corp HubSpot",
  "created_at": "ISO-8601 timestamp of first session",
  "last_updated": "ISO-8601 timestamp of this session",
  "sessions": [
    {
      "date": "ISO-8601 timestamp",
      "summary": "One-line human description of what happened",
      "actions": [
        {
          "type": "property_created",
          "object": "companies",
          "name": "source_object_name",
          "fieldType": "Dropdown select",
          "options": ["MASTERCHAIN", "CHAIN", "STOP", "DEPARTMENT"]
        },
        {
          "type": "property_fixed",
          "object": "companies",
          "name": "stop_no",
          "fromType": "string",
          "toType": "number",
          "sideEffects": ["Calculated property 'stop' permanently deleted — depended on this property"]
        },
        {
          "type": "artifacts_generated",
          "artifacts": ["postman", "workflows", "schema"],
          "spreadsheet": "mapping.xlsx",
          "propertyCount": { "companies": 19, "deals": 12, "products": 21 }
        },
        {
          "type": "validation_run",
          "results": { "companies": { "matched": 16, "mismatched": 2, "missing": 1 } }
        }
      ]
    }
  ],
  "pending": [
    {
      "object": "deals",
      "type": "missing_internal_names",
      "properties": ["region", "territory", "shipping_country"],
      "note": "No HubSpot internal name in col M — properties cannot be created until spreadsheet is updated"
    },
    {
      "object": "companies",
      "type": "pending_options",
      "properties": ["erp_active"],
      "note": "Dropdown options not yet defined — client input required before middleware can validate values"
    }
  ],
  "property_registry": {
    "companies": {
      "stop_no": {
        "currentType": "number",
        "history": [
          {
            "date": "ISO-8601",
            "event": "fixed",
            "fromType": "string",
            "toType": "number",
            "sideEffects": ["Calculated property 'stop' permanently deleted"]
          }
        ]
      },
      "source_object_name": {
        "currentType": "enumeration",
        "history": [
          { "date": "ISO-8601", "event": "created", "fieldType": "Dropdown select" }
        ]
      }
    }
  }
}
```

### Compaction policy

After writing the new session entry, check whether the history file needs compacting.

**Trigger compaction when either condition is true:**
- `sessions` array has more than **5 entries**, OR
- The JSON string length of the file exceeds **8 000 characters** (≈ 2 000 tokens)

**Compaction algorithm:**

1. Sort `sessions` by date ascending. Keep the **3 most recent** sessions in full detail. Collapse everything older into a single `compacted_history` block:

```json
"compacted_history": {
  "up_to": "ISO-8601 date of the last compacted session",
  "sessions_count": 7,
  "summary": "Prose summary of all compacted sessions — what was created, fixed, generated, and any permanent side effects (e.g. calculated properties deleted). Be specific about side effects; be brief about routine runs."
}
```

2. **Property registry** — for each property, keep:
   - `currentType` — always
   - The **most recent history entry** — always
   - Any entry that has a non-empty `sideEffects` array — always (side effects are permanent facts, never discard them)
   - Discard all other older history entries

3. **Pending items** — remove any item that has been resolved. Keep only genuinely open items.

4. Never shrink the file below a viable state. If after compaction the `sessions` array is empty, that means all sessions were compacted — that is fine as long as `compacted_history` exists.

**What must never be discarded regardless of file size:**
- Any `sideEffects` entry (e.g. a calculated property was permanently deleted)
- The `property_registry` current state for every property ever touched
- All currently open `pending` items
- The `compacted_history` block once it exists — never delete it, only update it

**After compaction**, verify the resulting JSON is valid and write it back. Log a one-line note at the end of the session report: `"History compacted — {N} sessions summarised, file reduced to ~{char_count} chars."`

---

### What to log per action type

| Action | Log when |
|--------|----------|
| `property_created` | A missing property was created via `--create` |
| `property_fixed` | A mismatched property was deleted + recreated via `--fix-mismatched` — always include `sideEffects` if a calculated property was deleted |
| `property_skipped` | A property was identified as mismatched or missing but the user chose not to act on it this session |
| `artifacts_generated` | Any generator script ran successfully — include which artifacts and property counts |
| `validation_run` | A validation-only pass ran (no --create or --fix) |
| `spreadsheet_updated` | The user updated the spreadsheet (e.g. added col M values) during the session |

### Pending items maintenance

- **Add** a pending item when: a property is skipped, col M is blank for tracked rows, or dropdown options are missing
- **Remove** a pending item when: the issue is resolved (property created, options defined, etc.)
- Keep the pending list accurate — stale pending items erode trust in the history

### Consequence checks powered by history

When the user requests any of the following in a future session, check history first and surface the relevant context before acting:

| User request | History check | Warning to show |
|---|---|---|
| Fix a type mismatch | `property_registry[object][name].history` has a `fixed` event | "Already fixed on {date} — refixing deletes data accumulated since then" |
| Create a property | `pending` list | "This was flagged as pending — confirm this resolves it" |
| Change find key | `sessions[].actions` where `artifacts_generated` used a different find key | "Postman collection was last generated with find key {old} — update any downstream consumers" |
| Regenerate artifacts | Last `artifacts_generated` action | "Last generated on {date} with {N} properties. {M} properties have been added/changed since then — confirm regeneration" |
| Add options to a dropdown | `pending` list with `type: pending_options` | "This was flagged as pending client input — confirm the options are now finalised before adding them" |

---

## Non-Negotiable Rules

- **Always load history before doing anything** — `.hubspot-integration/history.json` is the source of truth for what has been done, what is pending, and what side effects to expect. Never skip Step 0.
- **Always write history at the end of every session** — even if only a validation was run. An unwritten session is a lost session.
- **Always compact after writing** — check the two thresholds every time. A history file that grows unchecked becomes token-expensive and defeats its own purpose.
- **Side effects are permanent record** — never discard a `sideEffects` entry from the property registry, even during aggressive compaction. They document irreversible changes to the portal.
- **Never delete the history file** — if the user wants a fresh start, archive the existing file (rename with a timestamp suffix) rather than deleting it.
- **Surface history consequences proactively** — don't wait for the user to ask. If a requested action touches something with a history entry, show the relevant context before proceeding.
- **Keep pending items accurate** — resolve them when fixed, add them when discovered. A stale pending list is worse than no list.
- **When HubSpot MCP is connected, always use it for read operations first** — confirm portal identity (`get_organization_details`) before touching anything, then use `get_properties` to check existing state rather than inferring it from the spreadsheet alone. MCP reads are faster and more reliable than running a script and parsing its output.
- **MCP is read-only for property management** — it cannot create or modify properties. Use the Node.js scripts (with `--create` / `--fix-mismatched`) or the HubSpot API directly for write operations.
- **Always confirm portal identity before portal actions** — show the portal name from `get_organization_details` and get a `yes` from the user before creating or modifying any properties.

- **sanitizeName every internal name** before use in any output. HubSpot rejects uppercase letters and characters outside `[a-z0-9_]`.
- **hubspot_owner_id never appears in CREATE or UPDATE bodies** in the Postman collection.
- **Phone number TYPE_COMPARISON_MAP entry must be `'string'`** — not `'phone_number'`. Using the wrong value causes endless false mismatches.
- **Workflow output files use CommonJS.** Generator scripts use ES Modules. Never confuse the two.
- **source_object_name** is only injected (Company workflow + Postman body) when the user confirms company tiers are in use. Otherwise omit it entirely.
- **Never hardcode client names, portal IDs, or domain values.** All such values come from user input or the spreadsheet.
