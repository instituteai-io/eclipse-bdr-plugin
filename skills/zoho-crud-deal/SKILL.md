---
name: zoho-crud-deal
description: >
  Execute CRUD operations on Zoho CRM Deals using a crm_payload.deals array from any
  signal skill or play. Translates generic field names to Zoho API names using the
  mapping table in this file, runs composite dedup checks, and writes via Zoho MCP.
  Use when a skill has produced a crm_payload with a deals section and the user wants
  to write it to Zoho CRM. Triggers on: "write the deals to CRM", "create the deal
  in Zoho", "log this opportunity to CRM", "push this signal to Zoho", or when a
  crm_payload.deals block is present and the user asks to execute, push, or commit to CRM.
---

# Eclipse BDR — Zoho CRM: CRUD Deals

Accept a `crm_payload.deals` array from any signal skill or play, translate generic
field names to Zoho API names, run composite dedup checks, and execute writes via Zoho MCP.

---

## Field Mapping: Generic → Zoho API

| Generic Field | Zoho API Name | Type | Notes |
|---|---|---|---|
| `deal_name` | `Deal_Name` | Text | Required |
| `company_name` | `Account_Name` | Lookup → Accounts | Required |
| `primary_contact` | `Contact_Name` | Lookup → Contacts | Optional — left null by signal skills |
| `stage` | `Stage` | Picklist | Must match BDR pipeline stages |
| `deal_type` | `Type` | Picklist | |
| `amount` | `Amount` | Currency | Left null — filled during qualification |
| `close_date` | `Closing_Date` | Date | |
| `lead_source` | `Lead_Source` | Picklist | |
| `practice_areas` | `Practice_Areas_In_Scope` | Multi-select | |
| `signal_type` | `Signal_Type` | Picklist | ⚡ New field |
| `signal_source` | `Signal_Source` | Text | ⚡ New field |
| `signal_date` | `Signal_Date` | Date | ⚡ New field |
| `signal_summary` | `Signal_Summary` | Textarea | ⚡ New field |
| `suggested_approach` | `Suggested_Approach` | Textarea | ⚡ New field |
| `priority_score` | `Priority_Score` | Integer | ⚡ New field — range 1–100 |
| `play_source` | `Play_Source` | Picklist | ⚡ New field |

⚡ = field not yet in Zoho. If present in payload, skip that field and note it in the result summary — do not fail.

**Valid BDR pipeline stages:** Signal Detected → Outreach Sent → Response Received → Discovery Booked → Qualified
Do not use delivery pipeline stages (Scoping, Proposal Development, etc.) in BDR deal output.

---

## Step 1: Receive the Payload

The user must provide a `crm_payload.deals` array. Accept either:
- A full `crm_payload` object (extract the `deals` key), or
- A raw `deals` array directly

If no payload is provided, ask:
> "Please paste the `crm_payload.deals` section from the skill output."

---

## Step 2: Validate

For each deal operation:
- `deal_name` must be present — reject the record if missing
- `company_name` must be present — needed to link to Account
- `stage` must be a valid BDR pipeline stage — reject if not
- `action` must be one of: `create`, `update`, `upsert`, `skip_if_exists` — default to `skip_if_exists` if absent
- Warn (do not fail) on any ⚡ fields — skip those fields in the write

---

## Step 3: Dedup Check

Deals do not dedup on a single field. Use `dedup_query` from the payload if provided.
If not provided, construct the default query:

```
Account_Name = '<company_name>' AND Signal_Source = '<signal_source>' AND Signal_Date = '<signal_date>'
```

Translate `company_name` → `Account_Name` in the query before execution.

Run this as a COQL query via Zoho MCP (`executeCOQLQuery`).

Behaviour by action:
- `skip_if_exists` → if match found, skip; if not found, create
- `upsert` → if match found, update that record; if not found, create
- `update` → if match found, update; if not found, report as error
- `create` → skip dedup check, always create

**Why composite dedup:** Multiple deals for the same account are expected and correct
(e.g. a company can have both a regulatory action deal and a hiring signal deal).
Same company + same signal source + same signal date = same event, not a new opportunity.

---

## Step 4: Map Fields and Write

For each deal to write:
1. Translate every key in `fields` using the mapping table above
2. Remove null and empty fields — do not send nulls to Zoho
3. Execute via Zoho MCP:
   - New records: `createRecords` on `Deals` module
   - Updates: `updateRecord` on `Deals` module with the Zoho record ID from dedup step

---

## Step 5: Report Results

```
Deals processed: N
  Created: N
  Updated: N
  Skipped (already exists): N
  Errors: N
  Fields skipped (⚡ not yet in Zoho): [Signal_Type, Signal_Source, Signal_Date, ...]
```
