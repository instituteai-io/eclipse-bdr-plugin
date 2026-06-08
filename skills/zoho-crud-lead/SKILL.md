---
name: zoho-crud-lead
description: >
  Execute CRUD operations on the Zoho CRM Inbound (Leads) module using a crm_payload
  from any ICP signal skill. Translates generic field names to Zoho API names, runs dedup
  checks, and writes via Zoho MCP. This is the only CRM write skill for people signals —
  all ICP signal skills (icp-signal-data, icp-signal-risk, icp-signal-transformation,
  icp-signal-ai) route their output through here.
  Triggers on: "write the leads to CRM", "push to Zoho", "log these contacts to Inbound",
  "commit to CRM", or when a crm_payload with a leads section is present and the user
  asks to execute, push, or write.
---

# Eclipse BDR — Zoho CRM: CRUD Inbound (Leads)

Accept a `crm_payload` from any ICP signal skill, translate field names to Zoho API names,
run dedup checks, and write to the Inbound (Leads) module via Zoho MCP.

---

## Field Mapping: Generic → Zoho API

| Generic Field | Zoho API Name | Type | Notes |
|---|---|---|---|
| `First_Name` | `First_Name` | Text | |
| `Last_Name` | `Last_Name` | Text | |
| `Designation` | `Designation` | Text | Job title |
| `Company` | `Company` | Text | Required |
| `Email` | `Email` | Email | Dedup fallback |
| `Phone` | `Phone` | Phone | |
| `LinkedIn_Page_URL` | `LinkedIn_Page_URL` | URL | |
| `No_of_Employees` | `No_of_Employees` | Integer | |
| `Industry` | `Industry` | Picklist | See valid values below |
| `Practice_Areas_In_Scope` | `Practice_Areas_In_Scope` | Multi-select | See valid values below |
| `Lead_Source` | `Lead_Source` | Picklist | See valid values below |
| `Apollo_Contact_ID_c` | `Apollo_Contact_ID_c` | Text | ⚡ Pending team creation — skip if not in Zoho |
| `Signal_Type_c` | `Signal_Type_c` | Picklist | ⚡ Pending — values: Job Change / New Hire / Manual |
| `Signal_Date_c` | `Signal_Date_c` | Date | ⚡ Pending team creation |
| `Signal_Summary_c` | `Signal_Summary_c` | Textarea | ⚡ Pending team creation |

⚡ = custom field pending team creation. If field is not yet in Zoho, skip it silently and
include it in the result summary. Never fail on a missing ⚡ field.

**Valid `Industry` values (Inbound module):** Finance, Banking, Technology, Consulting,
Healthcare, Other. Map non-matching values to the closest option or `Other`.

**Valid `Practice_Areas_In_Scope` values:** Data Management & Analytics / Risk & Regulatory /
Financial Crimes & Compliance / Process & Operational Excellence / Federal / Payments /
Banking / Artificial Intelligence ⚡ (pending addition by team)

**Valid `Lead_Source` values (existing):** LinkedIn, Web Research, Cold Call, Referral - Client,
Referral - Employee, Referral - Managing Partner, Other.
`Apollo ICP Scan` ⚡ pending addition — if not yet in Zoho picklist, use `Other` and flag it.

---

## Step 1: Receive the Payload

Accept either:
- A full `crm_payload` object — extract the `leads` array
- A raw `leads` array directly

If no payload is provided, ask:
> "Please paste the `crm_payload` section from the ICP signal skill output."

---

## Step 2: Validate

For each lead record:
- `Company` must be present — reject the record if missing
- At least one of `First_Name` or `Last_Name` must be present — reject if both missing
- `action` defaults to `upsert` if not specified
- ⚡ fields: skip gracefully, log in result summary — do not fail

---

## Step 3: Dedup Check

For each operation where `action` is `upsert`, `update`, or `skip_if_exists`:

1. If `Apollo_Contact_ID_c` is present and the field exists in Zoho → search by `Apollo_Contact_ID_c`
2. Else if `Email` is present → search by `Email`
3. Else → search by `First_Name + Last_Name + Company` (composite)

Behaviour by action:
- `upsert` → if found, update; if not found, create
- `skip_if_exists` → if found, skip; if not found, create
- `update` → if found, update; if not found, report as error
- `create` → skip dedup, always create

---

## Step 4: Write to Zoho

For each lead to write:
1. Map fields using the table above
2. Remove null and empty fields — do not send nulls to Zoho
3. Skip any ⚡ field that doesn't exist yet in Zoho — log it, don't fail
4. Execute via Zoho MCP:
   - New records: `createRecords` on module `Leads`
   - Updates: `updateRecord` on module `Leads` with the Zoho record ID from dedup step

---

## Step 5: Report Results

```
Inbound (Leads) processed: N
  Created: N
  Updated: N
  Skipped (already exists): N
  Errors: N
  Fields skipped (⚡ not yet in Zoho): [Apollo_Contact_ID_c, Signal_Type_c, ...]
  Lead_Source fallback used: N records (Apollo ICP Scan → Other)
```
