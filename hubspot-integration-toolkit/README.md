# hubspot-integration-toolkit

A Claude Code slash command that scaffolds a complete HubSpot ↔ ERP integration toolkit from a data mapping spreadsheet. Works with any HubSpot portal — nothing is hardcoded.

## What it generates

| Artifact | Description |
|---|---|
| **Postman collection** | One collection with FIND, GET, CREATE, and UPDATE requests for every mapped object |
| **Workflow action files** | HubSpot custom coded action files (CommonJS) that fetch object properties and POST them to an external ERP/middleware endpoint |
| **Webhook schema** | JSON Schema draft-07 describing the payload structure — a handoff contract for the team building the receiving endpoint |

Optionally, if a HubSpot private app token is provided, the skill will also:
- Validate all spreadsheet properties against the live portal
- Create any missing properties
- Fix type-mismatched properties (with confirmation before any destructive action)

## Prerequisites

- [Claude Code](https://claude.ai/code) installed and running
- Node.js 18+ and npm
- A HubSpot data mapping spreadsheet (Excel `.xlsx`) with a **"Ready to Build"** sheet

### HubSpot MCP — required for token mode (portal actions)

When you want Claude to validate, create, or fix properties directly on a live HubSpot portal, the **HubSpot MCP integration must be connected** before invoking the skill. The MCP gives Claude direct read access to the portal (properties, objects, owners, org details) and is used alongside the private app token — MCP for reading portal state, the token for write operations.

**Option A — claude.ai integrations (recommended)**

1. Go to [claude.ai](https://claude.ai) → Settings → Integrations
2. Find **HubSpot** and click Connect
3. Authenticate with your HubSpot account
4. The MCP will be available in all Claude Code sessions that use your claude.ai account

**Option B — Claude Code MCP config (self-hosted / local)**

Add the HubSpot MCP server to your Claude Code MCP configuration (`~/.claude/claude.json` or `.mcp.json` in the project root):

```json
{
  "mcpServers": {
    "hubspot": {
      "command": "npx",
      "args": ["-y", "@hubspot/mcp-server"],
      "env": {
        "HUBSPOT_ACCESS_TOKEN": "pat-na1-your-token-here"
      }
    }
  }
}
```

**What the MCP enables**

| Capability | Without MCP | With MCP |
|---|---|---|
| Read existing portal properties | via script only | directly in conversation |
| Verify object/property existence | via script output | directly in conversation |
| Look up owners, org details | not available | available |
| Create / fix properties | via script (`--create`, `--fix-mismatched`) | script + MCP verification |
| Generate Postman + workflow files | ✓ | ✓ |

Without the MCP, the skill still works in token mode — it runs Node.js scripts to validate and create properties. The MCP makes the process more interactive and lets Claude reason about the portal state without requiring a script run for every lookup.

### Expected spreadsheet columns

| Column | Index | Content |
|--------|-------|---------|
| B | 1 | HubSpot Object type (e.g. `Company`, `Deal`, `Product`) |
| L | 11 | HubSpot Property Label (display name) |
| **M** | **12** | **HubSpot Property Internal Name — required; rows without this are skipped** |
| N | 13 | Field Type (`Single-line text`, `Number`, `Date picker`, `Dropdown select`, etc.) |
| O | 14 | Number format (if field type is Number) |
| P | 15 | Pipe-separated dropdown options (e.g. `OPTION_A\|OPTION_B\|OPTION_C`) |

## Installation

Copy (or symlink) the skill file into your Claude Code global commands directory:

```bash
# Option A — clone this repo directly into the commands directory
git clone <repo-url> ~/.claude/commands/hubspot-integration-toolkit

# Option B — symlink from wherever you keep it
ln -s /path/to/hubspot-integration-toolkit ~/.claude/commands/hubspot-integration-toolkit
```

Claude Code scans `~/.claude/commands/` recursively, so the skill will be available as `/hubspot-integration-toolkit` in any project session immediately after installation.

## Usage

Open any project directory in Claude Code and invoke the skill:

```
/hubspot-integration-toolkit
```

Or pass the spreadsheet path as an inline argument to skip the first question:

```
/hubspot-integration-toolkit path/to/mapping.xlsx
```

The skill will ask 9 clarifying questions in a single message before touching anything:

1. Spreadsheet path
2. HubSpot private app token _(optional — enables live portal validation)_
3. Project / portal name
4. Object types and any custom object slugs
5. Find keys (unique identifier property per object)
6. Workflow generation scope
7. External endpoint style (one env var per object vs. single base URL)
8. Output directory
9. Company tiers _(only if the portal uses a tiered Company structure)_

It then echoes back an object config table for confirmation before writing any files.

## Three modes

**Generate only (no token, no MCP)** — generates all artifacts from the spreadsheet alone. Postman CREATE/UPDATE requests may fail if the properties don't yet exist in HubSpot. Useful for drafting and reviewing before touching the portal.

**Token only** — validates properties against the live portal by running Node.js scripts. Reports missing and type-mismatched properties, then asks for confirmation before creating or fixing them. Recreates the full toolkit once the portal state is clean.

**Token + HubSpot MCP** _(recommended for active portal work)_ — the MCP gives Claude direct read access to the portal inside the conversation. Claude can look up existing properties, verify object schemas, and check org details without waiting for a script run. Combined with the token for write operations (create/fix), this is the most interactive and reliable mode. **The HubSpot MCP must be connected before invoking the skill** — see prerequisites above.

## Output structure

```
{project}/
├── postman/
│   └── {ProjectSlug}_Master_Collection.postman_collection.json
├── workflows/
│   ├── company-to-erp.js
│   ├── deal-to-erp.js
│   └── product-to-erp.js   ← generated with warning if products can't be enrolled
└── schemas/
    └── {ProjectSlug}_Webhook_Schema.json
```

## HubSpot workflow setup (after generation)

For each generated workflow file:

1. Create a new **Custom Coded Action** workflow in HubSpot
2. Paste the contents of the relevant `.js` file
3. Add the following secrets in the workflow editor:
   - `PRIVATE_APP_ACCESS_TOKEN` — your HubSpot Private App token
   - `ERP_{OBJECT}_ENDPOINT` — the external endpoint URL for that object
4. For **Deal**: create **two separate workflows** using the same `deal-to-erp.js` file — one triggered by "Deal is created" and one by "Deal is updated". Set the `event_type` input field to `deal.created` / `deal.updated` respectively.

## Notes

- The webhook schema (`schemas/`) is a **documentation artifact for the receiving team** (the ERP/middleware developers). It is not used by the HubSpot workflows themselves — it describes the payload shape they should expect to receive.
- Products cannot be enrolled in HubSpot workflows. The Postman collection handles the ERP → HubSpot direction for products.
- All generated Node.js toolkit scripts use ES Modules (`import`/`export`). The workflow output files use CommonJS (`require`/`exports.main`) as required by HubSpot's custom coded action runtime.
