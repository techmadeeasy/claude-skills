# HubSpot Property Mapper Toolkit

Use this skill when writing or reviewing **Custom code** actions in HubSpot workflows (Operations Hub / Data Hub programmable automation).

## Goals (non-negotiable)

1. **CRM fields come from the record**, not from arbitrary workflow inputs that duplicate HubSpot data.
   - Prefer **`event.object.objectId`** (and `event.object.objectType` when validating) for the enrolled record's id.
   - Prefer **Properties to include in code** with variable names equal to HubSpot **internal property names**, each mapped from the **enrolled object** (same pattern as `event.inputFields` in HubSpot docs—values originate from the record, not free-typed secrets).
   - Use **associations** (v4 Associations API or workflow-supported association inputs) when the needed data lives on related objects—do not ask the user to paste associated record ids unless HubSpot cannot expose them.

2. **Do not require a HubSpot API token** to read CRM properties that the workflow can already inject via **Properties to include in code**. Reserve `process.env` / secrets for **external** systems (e.g. webhook URL, HMAC secret for a non-HubSpot receiver).

3. **`event.inputFields` is for HubSpot-injected record data**, not a second copy of the CRM. If a key in `inputFields` is not mappable from the enrolled object (or its associations / formatted properties workflow), treat that as a design smell unless explicitly justified.

## Before writing or approving code: ask these questions

Ask the user (or answer from context). If any answer is wrong, revise the design before pasting code into HubSpot.

### A. Enrolled object and identity

- **Which object type enrolls this workflow?** (contact / company / deal / custom object) Does the code assume the same type?
- **Record id:** Will **`event.object.objectId`** always be present at runtime? If yes, **do not require** a separate input named `hs_object_id` unless it is optional redundancy; if you still output `hs_object_id` in a webhook payload, derive it from the injected `hs_object_id` property when mapped, or from `event.object.objectId` when not mapped.

### B. Data sources (catch "unretrievable" inputs)

For **each** piece of data the action uses:

- **Where does it live in HubSpot?** (property on enrolled object, association, calculated property, owner, etc.)
- **Can it be passed via "Properties to include in code"** (or associations) so the runtime value is always tied to the enrolled record?
- If the answer is **no** (e.g. external system id not stored on HubSpot, arbitrary user text, cross-portal constant): is this **intentionally** external configuration? If it is CRM data, stop and redesign (sync field into HubSpot, association, or separate automation).

### C. Secrets and env vars

List every `process.env.*` usage:

- **Allowed without extra scrutiny:** URLs and secrets for **non-HubSpot** destinations (e.g. `YOUR_SERVICE_WEBHOOK_URL`, `YOUR_SERVICE_WEBHOOK_SECRET`), or third-party API keys for services HubSpot cannot call natively.
- **Red flag:** `HUBSPOT_ACCESS_TOKEN` / private app token **only** to read properties of the **same object type that is already enrolled**. Replace with Properties to include in code (and associations) unless there is a documented HubSpot limitation that cannot be worked around.

### D. Associations and read-only fields

- Does the action need **labels or ids from associated companies/contacts/deals**? Confirm whether workflow "Properties to include in code" or association-based inputs can supply them; otherwise plan an Associations API call **only if** a token is acceptable and scoped—and document why Properties-in-code is insufficient.

### E. Payload and empties

- If posting a JSON webhook: should **every expected key** always be present (empty string when missing), for downstream stability? Align with team convention.

## Implementation checklist

When producing `exports.main = async (event, callback) => { ... }` style code:

- [ ] Resolve record id from **`event.object.objectId`**, with optional fallback to mapped `hs_object_id` / `record_id` only if product requires backward compatibility.
- [ ] Build `properties` from **`event.inputFields`** keys that correspond to HubSpot property names (plus `orderedInputKeys`-style merge if extra mapped keys are allowed).
- [ ] **No** `axios.get` to `api.hubapi.com` for the enrolled object's own properties unless the user explicitly accepts a private app token and documents why Properties-in-code is insufficient.
- [ ] Webhook POST (or other external call) may use **`axios`**; keep timeouts and error handling consistent with existing repo patterns.
- [ ] Document in comments or repo README: required **secrets**, optional secrets, and **which Properties to include in code** (internal names) for each object type.

## Reference (HubSpot)

- Custom code **`event`** shape: `object`, `inputFields`, `callbackId`, `origin` — see HubSpot docs *Workflows | Custom code actions* (`developers.hubspot.com/docs/api/workflows/custom-code-actions`).
- **Properties to include in code** are read from `event.inputFields['propertyName']` after mapping from the enrolled record in the action UI.

## Repo examples

If the current repo has existing custom code workflow scripts, locate them (e.g. under `scripts/workflows/`) and align with their patterns before producing new code. Look for:

- How they read the enrolled record id (`event.object.objectId` vs. a mapped input).
- Which properties are injected via "Properties to include in code" (`event.inputFields`).
- How secrets / env vars are named and used.
- Error handling and `callback` invocation style.

If no examples exist yet, treat the first script you write as the canonical pattern and document it here for future reference.
