---
name: zoho-crud-account
description: >
  Execute CRUD operations on Zoho CRM Accounts using a crm_payload.accounts array from
  any signal skill, play, or enrichment skill. Translates generic field names to Zoho
  API names using the mapping table in this file and writes via Zoho MCP.
  Use when a skill has produced a crm_payload with an accounts section and the user
  wants to write it to Zoho CRM. Triggers on: "write the accounts to CRM",
  "create the account in Zoho", "update this company in CRM", "log this account to Zoho",
  or when a crm_payload.accounts block is present and the user asks to execute, push,
  or commit to CRM.
---

# Eclipse BDR — Zoho CRM: CRUD Accounts

Accept a `crm_payload.accounts` array from any skill, translate generic field names to
Zoho API names, run dedup checks, and execute writes via Zoho MCP.

---

## Field Mapping: Generic → Zoho API

| Generic Field | Zoho API Name | Type | Notes |
|---|---|---|---|
| `company_name` | `Account_Name` | Text | Required. Primary identifier |
| `industry` | `Industry` | Picklist | |
| `company_size` | `Company_Size_c` | Picklist | < 500 / 500-2,500 / 2,500-10,000 / 10,000-50,000 / 50,000+ |
| `annual_revenue` | `Annual_Revenue` | Currency | |
| `employee_count` | `Employees` | Integer | |
| `website` | `Website` | URL | |
| `linkedin_url` | `LinkedIn_Page_c` | URL | Custom field |
| `icp_tier` | `ICP_Tier` | Picklist | ⚡ New field — Tier 1 – Strong Fit / Tier 2 – Moderate Fit / Tier 3 – Watch List |
| `practice_areas` | `Practice_Areas_In_Scope` | Multi-select | |
| `apollo_account_id` | `Apollo_Account_ID` | Text | ⚡ New field. Set by apollo_enrich_account, not signal skills |
| `zoominfo_account_id` | `ZoomInfo_Account_ID` | Text | ⚡ New field (future). Set by zoominfo_enrich_account |

⚡ = field not yet in Zoho. If present in payload, skip that field and note it in the result summary — do not fail.

---

## Step 1: Receive the Payload

The user must provide a `crm_payload.accounts` array. Accept either:
- A full `crm_payload` object (extract the `accounts` key), or
- A raw `accounts` array directly

If no payload is provided, ask:
> "Please paste the `crm_payload.accounts` section from the skill output."

---

## Step 2: Validate

For each account operation:
- `company_name` must be present — reject the record if missing
- `action` must be one of: `create`, `update`, `upsert`, `skip_if_exists` — default to `upsert` if absent
- Warn (do not fail) on any ⚡ fields — skip those fields in the write, include them in the result summary

---

## Step 3: Dedup Check

For each operation where `action` is `upsert`, `update`, or `skip_if_exists`:

1. If `apollo_account_id` is present and non-null → search Zoho by `Apollo_Account_ID` field
2. Otherwise → search Zoho by `Account_Name` (exact match)

Behaviour by action:
- `skip_if_exists` → if found, skip; if not found, create
- `upsert` → if found, update; if not found, create
- `update` → if found, update; if not found, report as error
- `create` → skip dedup check, always create

---

## Step 4: Map Fields and Write

For each account to write:
1. Translate every key in `fields` using the mapping table above
2. Remove null and empty fields — do not send nulls to Zoho
3. Execute via Zoho MCP:
   - New records: `createRecords` on `Accounts` module
   - Updates: `updateRecord` on `Accounts` module with the Zoho record ID from dedup step

---

## Step 5: Report Results

```
Accounts processed: N
  Created: N
  Updated: N
  Skipped (already exists): N
  Errors: N
  Fields skipped (⚡ not yet in Zoho): [ICP_Tier, Apollo_Account_ID, ...]
```

Return the Zoho record ID for each created or updated account — downstream deal writes
need this to link the Account_Name lookup correctly.
