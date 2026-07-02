---
name: zoho-crud-lead
description: >
  Execute CRUD operations on the Zoho CRM Inbound (Leads) module using a crm_payload
  from any BDR signal skill that surfaces people. Normalizes field names to the live Zoho
  API names, validates picklist values against live Zoho metadata, runs dedup checks, and
  writes via Zoho MCP. This is the only CRM write skill for people signals — all
  person-surfacing skills (bdr-scan-icp-data, bdr-signal-hiring-data / -risk /
  -transformation / -ai and their ZoomInfo variants, bdr-signal-new-exec-apollo,
  bdr-signal-job-change) route their output through here.
  Triggers on: "write the leads to CRM", "push to Zoho", "log these contacts to Inbound",
  "commit to CRM", or when a crm_payload with a leads section is present and the user
  asks to execute, push, or write.
---

# Eclipse BDR — Zoho CRM: CRUD Inbound (Leads)

Accept a `crm_payload` from any person-surfacing BDR skill, normalize field names to the
live Zoho API names, validate picklist values against live Zoho metadata, run dedup
checks, and write to the Inbound (Leads) module via Zoho MCP.

**Scope guard:** this skill writes to the `Leads` module (labelled **Inbound** in the Zoho
UI) and nothing else. No Accounts, no Contacts, no Deals — company-level signals go to the
Teams digest, never through here.

---

## Canonical Field Contract (live-verified against Zoho, 2026-07-02)

The canonical payload field names ARE the live Zoho API names. All four Signal custom
fields exist in Zoho today — they are live, not pending.

| Payload / Zoho API Name | Type (max length) | Notes |
|---|---|---|
| `Company` | Text (200) | Required by this skill — reject record if missing |
| `First_Name` | Text (40) | |
| `Last_Name` | Text (80) | **Zoho system-mandatory** — reject record if missing |
| `Designation` | Text (100) | Job title — labelled "Title" in UI |
| `Email` | Email (100) | **Unique in Zoho** (case-insensitive) — see dedup + error handling |
| `Phone` | Phone | |
| `Mobile` | Phone | |
| `Website` | URL | |
| `LinkedIn_Page_URL` | URL (450) | |
| `No_of_Employees` | Integer | |
| `Annual_Revenue` | Currency | |
| `Industry` | Picklist | Validate live — see picklist policy |
| `Practice_Areas_In_Scope` | Multi-select | Validate live — see picklist policy |
| `Lead_Source` | Picklist | `Apollo ICP Scan` is a live value ✅ |
| `Lead_Status` | Picklist | **Set to `New` on every create** (so the team's "new AI leads" view catches it). Never write it on update — the team owns status progression |
| `Owner` | User lookup | **Set on every create** from the Default Inbound Owner setting in Notion (see Step 2). Never write it on update. Zoho records always have an owner — empty is impossible (verified 2026-07-02) |
| `Description` | Textarea (32,000) | |
| `Apollo_Contact_ID` | Text (255) | ✅ Live custom field — primary dedup key for Apollo leads |
| `Signal_Type` | Picklist | ✅ Live — values: `Job Change` / `New Hire` / `Manual` |
| `Signal_Date` | Date (YYYY-MM-DD) | ✅ Live custom field |
| `Signal_Summary` | Textarea (32,000) | ✅ Live custom field |
| `ZoomInfo_Contact_ID` | Text (255) | ✖ Not in Zoho — **schema is final**, field is not coming. Always drop + log; dedup falls back to Email / name+company |

Truncate any value exceeding its max length and note the truncation in the result summary.

### Alias Normalization (accept legacy payloads, never emit them)

Before anything else, normalize each record's field names:

1. **Strip a trailing `_c`** — `Apollo_Contact_ID_c` → `Apollo_Contact_ID`,
   `Signal_Type_c` → `Signal_Type`, etc. (Older payload specs used `_c`-suffixed names;
   the fields were created in Zoho without the suffix.)
2. **Map lowercase generic names** (from the CRM Payload — Leads spec):
   `first_name`→`First_Name`, `last_name`→`Last_Name`, `title`→`Designation`,
   `company`→`Company`, `email`→`Email`, `phone`→`Phone`,
   `linkedin_url`→`LinkedIn_Page_URL`, `employee_count`→`No_of_Employees`,
   `industry`→`Industry`, `practice_areas`→`Practice_Areas_In_Scope`,
   `lead_source`→`Lead_Source`, `apollo_contact_id`→`Apollo_Contact_ID`,
   `zoominfo_contact_id`→`ZoomInfo_Contact_ID`, `signal_type`→`Signal_Type`,
   `signal_date`→`Signal_Date`, `signal_summary`→`Signal_Summary`.

### Extension Fields — Fold Into Signal_Summary, Never Write As Fields

Some producer skills emit richer signal metadata that has no Zoho field. Do NOT write
these as fields; fold them into `Signal_Summary` so nothing is lost:

| Payload field (any alias) | Fold rule |
|---|---|
| `Signal_Mechanism` | Append sentence: `Mechanism: <value>.` |
| `Signal_Source_Date` | Append sentence: `Source date: <value>.` |
| `Signal_Needs_Confirmation` (true) | Prefix summary with `⚠ Needs confirmation — ` |

Any other unknown field: skip it and list it in the result summary — never fail on it,
never send it to Zoho.

---

## Step 1: Receive the Payload

Accept either:
- A full `crm_payload` object — extract the `leads` array
- A raw `leads` array directly

If no payload is provided, ask:
> "Please paste the `crm_payload` section from the signal skill output."

The `meta` block on each lead is for human review and routing only — never written to Zoho.

---

## Step 2: Fetch Live Field Metadata & Settings (once per run)

**2a — Owner setting from Notion.** Fetch the **Zoho CRM — Inbound Module Schema** page
(`https://app.notion.com/p/03b4e4751c3a825d830f814cb7920f2b`) and read the
**Default Inbound Owner** email from its ⚙️ Settings table. Resolve the email to a Zoho
user via `getUsers` (type `ActiveUsers`). On every **create**, set
`Owner: {"id": "<resolved user id>"}`. Never set `Owner` on an update. If the setting is
missing or the email matches no active Zoho user: omit `Owner` (Zoho assigns the API
user) and flag it in the run report. Never hardcode an owner.

**2b — Live field metadata.** Call `getFields` on module `Leads` once at the start of the
run. From the response, build:
- The set of existing field API names (this is how you know whether `ZoomInfo_Contact_ID`
  exists yet — do not hardcode its absence).
- The live picklist values for `Industry`, `Practice_Areas_In_Scope`, `Lead_Source`,
  `Lead_Status`, and `Signal_Type`.

Never send a picklist value that is not in the live list — Zoho may reject the whole record.

---

## Step 3: Validate & Map Values

For each lead record (after alias normalization):

**Hard requirements — reject the record (report, don't write) if:**
- `Company` is missing or empty
- `Last_Name` is missing or empty (Zoho system-mandatory; if the provider masked the last
  name, the record is not CRM-ready — report it for manual handling, do not invent a value)

**Picklist policy (validate against live values from Step 2):**
- `Industry` — if the payload value is live, use it; else map to the closest live value
  (e.g. anything financial-services-ish → `Finance` or `Banking`); else `Other`. Log mappings.
- `Practice_Areas_In_Scope` — known mappings: `Transformation` → `Process & Operational
  Excellence`, `Data Management` → `Data Management & Analytics`. `Risk & Regulatory`,
  `Financial Crimes & Compliance`, `Artificial Intelligence` and the other live values pass
  through as-is. Drop (and log) any value with no live equivalent.
- `Lead_Source` — if live (e.g. `Apollo ICP Scan` ✅), use it. If not live (e.g.
  `ZoomInfo ICP Scan` — pending picklist addition), write `Other` and append the real
  source as a sentence in `Signal_Summary`: `Source: ZoomInfo ICP Scan.` Log the fallback.
- `Signal_Type` — live values are `Job Change` / `New Hire` / `Manual`. If the payload
  value is live, use it. If not (e.g. `Company Hiring`, `New Executive Sponsor` — pending
  picklist additions), leave `Signal_Type` unset and prefix `Signal_Summary` with the real
  type in brackets: `[Company Hiring] <summary…>`. Log the fallback.
- `Lead_Status` — set to `New` on every **create** so AI-created leads land in the team's
  triage view (`Inbound Source = Apollo ICP Scan` + `Inbound Status = New`). Never write
  `Lead_Status` on an **update** — the team owns status progression after creation.
- `Owner` — set on every **create** from the resolved Default Inbound Owner (Step 2a).
  Never on update.

**General:**
- `action` defaults to `upsert` if not specified
- Remove null and empty fields — never send nulls to Zoho
- `Signal_Date` must be `YYYY-MM-DD`
- If `ZoomInfo_Contact_ID` doesn't exist in Zoho yet (per Step 2): drop the field, log it,
  and rely on the dedup fallbacks below

---

## Step 4: Dedup Check

For each record where `action` is `upsert`, `update`, or `skip_if_exists`, search `Leads`
in this priority order (stop at first hit). Use `searchRecords` with
`converted: "both"` so a lead that was already converted is found rather than re-created:

1. **Provider ID** — `(Apollo_Contact_ID:equals:<id>)` if present; likewise
   `ZoomInfo_Contact_ID` once that field exists in Zoho
2. **Email** — `(Email:equals:<email>)` (Email is unique in Zoho, so this is authoritative)
3. **Composite** — `((Last_Name:equals:<last>)and(Company:equals:<company>))`, then
   confirm `First_Name` matches before treating it as the same person

Behaviour by action:
- `upsert` → found: update; not found: create
- `skip_if_exists` → found: skip; not found: create
- `update` → found: update; not found: report as error
- `create` → skip dedup, always create

---

## Step 5: Write to Zoho

- **Creates:** `createRecords` on module `Leads` — batch multiple creates in one call.
- **Updates:** `updateRecord` on module `Leads` with the record ID from the dedup step.
  - On update, do not blank existing values: only send fields that have non-null payload
    values (already guaranteed by Step 3).
  - `Practice_Areas_In_Scope` on update: **merge** — read the existing record's values,
    union with the payload values, send the combined list (a person can match multiple
    practices across scans).
- **DUPLICATE_DATA error on create** (unique `Email` collision): re-search by that email,
  then apply the record as an update to the found ID. Count it as an update, note the
  collision in the result summary.
- Report every write with its Zoho record ID.

---

## 🏃 Run Logging

This skill is a building block: when invoked by another Eclipse BDR skill, the **calling**
skill owns the Output Standard (templated report, Skill Runs log, Teams summary) — do not
log or post separately from here.

When invoked **standalone by a user**, log the invocation to the **Skill Runs** database
(`https://app.notion.com/p/3874e4751c3a8084a89be17b28e4c6a1`, data source
`collection://3874e475-1c3a-80a1-8ed7-000ba308ec09`) via `notion-create-pages`: Name =
`zoho-crud-lead — <YYYY-MM-DD> — <what was done>`, Select = `zoho-crud-lead`, Type = `live run`
or `test run`, body = a short note of the request and result. No Teams post is required for
standalone utility runs.

---

## Step 6: Report Results

```
Inbound (Leads) processed: N
  Created: N   [record IDs]
  Updated: N   [record IDs]
  Skipped (already exists): N
  Rejected (validation): N   [reason per record — e.g. Last_Name missing/masked]
  Errors: N
  Picklist fallbacks: [e.g. Lead_Source "ZoomInfo ICP Scan" → Other (3 records);
                       Signal_Type "Company Hiring" → folded into Signal_Summary (5 records)]
  Fields dropped (not in Zoho): [e.g. ZoomInfo_Contact_ID — pending field creation]
  Values truncated: [field: N records]
```

---

## ✖ Known Schema Gaps — live Zoho schema is FINAL (decided 2026-07-02)

No CRM schema changes are coming. These fallbacks are the permanent behaviour, not
interim workarounds:

1. **No `ZoomInfo_Contact_ID` field** — the ZoomInfo person ID is never stored; dedup for
   ZoomInfo-sourced leads is Email → name+company.
2. **`Signal_Type` has only** `Job Change` / `New Hire` / `Manual` — other types
   (`Company Hiring`, `New Executive Sponsor`) live as a `[Type] ` prefix in `Signal_Summary`.
3. **`Lead_Source` has no `ZoomInfo ICP Scan`** — ZoomInfo-sourced leads carry `Other`
   with the real source in `Signal_Summary`.

(Step 2 still validates picklists live each run — if the team ever does add values, the
fallbacks retire themselves with no skill change.)
