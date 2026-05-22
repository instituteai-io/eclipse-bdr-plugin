---
name: zoho-crud-contact
description: >
  Execute CRUD operations on Zoho CRM Contacts using a crm_payload.contacts array from
  any signal skill, play, or enrichment skill. Translates generic field names to Zoho
  API names using the mapping table in this file and writes via Zoho MCP.
  Use when a skill has produced a crm_payload with a contacts section and the user
  wants to write it to Zoho CRM. Triggers on: "write the contacts to CRM",
  "update this person in Zoho", "log this contact to CRM", "update their job change
  in Zoho", or when a crm_payload.contacts block is present and the user asks to
  execute, push, or commit to CRM.
---

# Eclipse BDR — Zoho CRM: CRUD Contacts

Accept a `crm_payload.contacts` array from any skill, translate generic field names to
Zoho API names, run dedup checks, and execute writes via Zoho MCP.

---

## Field Mapping: Generic → Zoho API

| Generic Field | Zoho API Name | Type | Notes |
|---|---|---|---|
| `first_name` | `First_Name` | Text | |
| `last_name` | `Last_Name` | Text | |
| `title` | `Title` | Text | |
| `email` | `Email` | Email | Key dedup fallback |
| `phone` | `Phone` | Phone | |
| `linkedin_url` | `LinkedIn_Page_c` | URL | Custom field |
| `company_name` | `Account_Name` | Lookup → Accounts | Links contact to company |
| `seniority_level` | `Level_c` | Picklist | Entry Level / Mid-Level / Senior Management / Executive |
| `priority` | `Priority_c` | Picklist | 1 High – Potential Buyer / ICP / 2 Med – Influencer / 3 Low – Passive |
| `relationship_strength` | `Relationship_Strength_c` | Picklist | 1 – No Relationship … 5 – Excellent |
| `work_status` | `Work_Status_c` | Picklist | Employee / In Transition / Retired |
| `is_eclipse_alum` | `Eclipse_Alum_c` | Boolean | |
| `practice_areas` | `Practice_Areas_In_Scope` | Multi-select | |
| `icp_role_match` | `ICP_Role_Match` | Picklist | ⚡ New field — Primary ICP / Secondary ICP / Not ICP |
| `apollo_contact_id` | `Apollo_Contact_ID` | Text | ⚡ New field. Set by apollo_enrich_contact |
| `zoominfo_contact_id` | `ZoomInfo_Contact_ID` | Text | ⚡ New field (future). Set by zoominfo_enrich_contact |

⚡ = field not yet in Zoho. If present in payload, skip that field and note it in the result summary — do not fail.

---

## Step 1: Receive the Payload

The user must provide a `crm_payload.contacts` array. Accept either:
- A full `crm_payload` object (extract the `contacts` key), or
- A raw `contacts` array directly

If no payload is provided, ask:
> "Please paste the `crm_payload.contacts` section from the skill output."

---

## Step 2: Validate

For each contact operation:
- At minimum `first_name` or `last_name` must be present — reject the record if both are missing
- `company_name` must be present — needed to link the contact to an Account
- `action` must be one of: `create`, `update`, `upsert`, `skip_if_exists` — default to `upsert` if absent
- Warn (do not fail) on any ⚡ fields — skip those fields in the write

---

## Step 3: Dedup Check

For each operation where `action` is `upsert`, `update`, or `skip_if_exists`:

1. If `apollo_contact_id` is present and non-null → search Zoho by `Apollo_Contact_ID`
2. If `email` is present → search by `Email` (exact match)
3. Otherwise → search by `First_Name + Last_Name + Account_Name` (composite match)

Behaviour by action:
- `skip_if_exists` → if found, skip; if not found, create
- `upsert` → if found, update; if not found, create
- `update` → if found, update; if not found, report as error
- `create` → skip dedup check, always create

---

## Step 4: Map Fields and Write

For each contact to write:
1. Translate every key in `fields` using the mapping table above
2. Remove null and empty fields — do not send nulls to Zoho
3. Execute via Zoho MCP:
   - New records: `createRecords` on `Contacts` module
   - Updates: `updateRecord` on `Contacts` module with the Zoho record ID from dedup step

---

## Step 5: Report Results

```
Contacts processed: N
  Created: N
  Updated: N
  Skipped (already exists): N
  Errors: N
  Fields skipped (⚡ not yet in Zoho): [ICP_Role_Match, Apollo_Contact_ID, ...]
```
