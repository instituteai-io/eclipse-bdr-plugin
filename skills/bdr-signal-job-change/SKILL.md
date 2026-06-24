---
name: bdr-signal-job-change
description: >
  Detect role changes for known leads already in Zoho CRM. Pulls all leads with a stored
  Apollo Contact ID, checks their current profile in Apollo, and flags any who have changed
  employer or title since the record was last updated. Updates the matching Zoho Lead record
  with the new signal and returns a JSON summary.
  Use when asked to check for job changes, detect role transitions in known leads,
  run the job change signal scan, or find prospects who changed companies.
  Triggers on: "check for job changes", "run job change signal", "who changed roles",
  "role transition scan", "bdr-signal-job-change".
---

# Eclipse BDR — Signal: Job Change (Known Leads)

Detect employer or title changes for leads already in the Zoho Inbound (Leads) module.
Updates existing records — never creates new ones. Output is JSON only.

---

## ⚠️ Criteria Reference Page

**Source (Notion):** `https://app.notion.com/p/7de4e4751c3a83da9ecf8121e82532a7`
(page *Skill References - Signal: Job Change (Known Leads)*, in the **Skill References** database)

Fetch via the Notion MCP `notion-fetch` tool with the URL above. This page defines:
- Which Zoho Lead fields to compare against Apollo (e.g. `Company`, `Designation`)
- What qualifies as a "significant" change worth flagging
- Any lead filters to apply before running (e.g. only check leads created in the last N days,
  or only leads in a specific practice area)

Always fetch it fresh — never use remembered criteria. If unreachable, stop and tell the user.

---

## Step 1: Fetch Criteria

Fetch the criteria reference page. Extract:

- `fields_to_compare` — list of Zoho field names to compare against Apollo data (e.g. `["Company", "Designation"]`)
- `lead_filter` — optional filter for which leads to check (e.g. `practice_area = "Data Management & Analytics"`, or `all`)
- `lookback_days` — only check leads whose `Signal_Date_c` is older than this many days (avoids re-triggering too soon)
- `require_apollo_id` — `true` / `false`: whether to skip leads missing `Apollo_Contact_ID_c`

If the page is unreachable or empty, stop and ask the user. Do not proceed with guessed criteria.

---

## Step 2: Pull Known Leads from Zoho

Query the Zoho Inbound (Leads) module via Zoho MCP (`getRecords` or `executeCOQLQuery`).

Apply the `lead_filter` from Step 1. At minimum, include these fields in the response:
`id`, `First_Name`, `Last_Name`, `Company`, `Designation`, `Apollo_Contact_ID_c`, `Signal_Date_c`

If `require_apollo_id` is `true` (default): skip any lead where `Apollo_Contact_ID_c` is blank.
Log the count of skipped records in the final summary.

Batch in pages of 50 if the lead count is large.

---

## Step 3: Match Against Apollo

For each lead from Step 2, call `apollo_people_match` with:

| Lead field | Apollo parameter |
|---|---|
| `Apollo_Contact_ID_c` | `id` (direct lookup if present) |
| `First_Name` + `Last_Name` | `first_name` + `last_name` (fallback if no Apollo ID) |
| `Company` | `organization_name` (fallback only) |

For each Apollo response, capture:
- `current_employer` → maps to `Company`
- `current_title` → maps to `Designation`
- `linkedin_url` (update if changed)

If Apollo returns no match for a lead: log as `no_apollo_match`, skip, continue.

---

## Step 4: Detect Changes

Compare each field listed in `fields_to_compare` (from Step 1) between the Zoho record and the Apollo response.

A **job change** is detected when:
- `Company` differs (employer change), OR
- `Designation` differs (title change)

String comparison: case-insensitive, trim whitespace. Ignore minor formatting differences
(e.g. "JPMorgan Chase & Co." vs "JPMorgan Chase"). Use contains/fuzzy only if the criteria
page explicitly instructs it.

For each change detected, build a change record:

```
{
  "zoho_lead_id": "<id>",
  "apollo_contact_id": "<id>",
  "first_name": "<first>",
  "last_name": "<last>",
  "changes": {
    "Company": { "was": "<old>", "now": "<new>" },
    "Designation": { "was": "<old>", "now": "<new>" }
  }
}
```

---

## Step 5: Generate Signal Summary

For each changed lead, write a `signal_summary` (2 sentences max):

> "[First name] was previously [old title] at [old company].
> Apollo shows they are now [new title] at [new company] as of [today's date]."

If only the title changed (same company):
> "[First name] was [old title] at [company].
> Apollo shows a title change to [new title] as of [today's date]."

---

## Step 6: Update Zoho Lead

For each changed lead, call `updateRecord` on module `Leads` with the Zoho record ID.

Fields to update:

| Field | Value |
|---|---|
| `Company` | Apollo `current_employer` (if changed) |
| `Designation` | Apollo `current_title` (if changed) |
| `Signal_Type_c` | `"Job Change"` ⚡ |
| `Signal_Date_c` | Today's date ⚡ |
| `Signal_Summary_c` | Generated in Step 5 ⚡ |

⚡ = custom field pending team creation. Skip gracefully if not yet in Zoho — log in result.

Do **not** update any other fields. Do **not** create new records. Do **not** touch any module
other than Leads.

---

## Step 7: Return JSON

Output **only** this JSON — no prose before or after.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-job-change",
  "signal_source": "Apollo",
  "criteria_fetched_from": "<URL fetched in Step 1>",
  "summary": {
    "leads_checked": 0,
    "skipped_no_apollo_id": 0,
    "skipped_no_apollo_match": 0,
    "job_changes_detected": 0,
    "zoho_records_updated": 0,
    "zoho_update_errors": 0,
    "fields_skipped_pending": []
  },
  "changes": [
    {
      "zoho_lead_id": "<id>",
      "apollo_contact_id": "<id>",
      "first_name": "<first>",
      "last_name": "<last>",
      "changes": {
        "Company": { "was": "<old>", "now": "<new>" },
        "Designation": { "was": "<old>", "now": "<new>" }
      },
      "signal_summary": "<generated summary>",
      "zoho_update_status": "updated | error | skipped"
    }
  ]
}
```

- `changes` array contains only leads where at least one field changed
- If no changes found: return empty `changes: []` with a note in `summary`

---

## Eclipse Context

**Signal:** ICP Role Transition — per Buy Trigger Definitions, role changes for known prospects
are a primary buying trigger. New leaders assess programs in the first 90 days.

**Action after signal:** Outreach acknowledging the move, not referencing prior role.

