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
  Every run renders the standardized Notion output template, computes live signal/source
  coverage from the Signal & Source Library, logs the run to the Skill Runs database, and
  posts a summary to Teams.
---

# Eclipse BDR — Signal: Job Change (Known Leads)

Detect employer or title changes for leads already in the Zoho Inbound (Leads) module.
Updates existing records — never creates new ones. The JSON summary is the data payload;
the user-facing deliverable is the templated report (see 📐 Output Standard).

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

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

Assemble this JSON payload — it feeds the Skill Runs log and any downstream handoff. The
user-facing deliverable is the templated report (see 📐 Output Standard).

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

## 📐 Output Standard (mandatory — every run)

Every run of this skill ends by rendering a standardized report, logging the run to Notion, and
posting a summary to Microsoft Teams. Nothing in this section is hardcoded: signal coverage,
source scoring, and the report format are all fetched fresh from Notion on every run.

### A. Fetch the output template (format authority)

Fetch this page fresh at the start of every run (Notion MCP `notion-fetch`):

**Output Template — Person-Level Leads** — `https://app.notion.com/p/3914e4751c3a8145acd0e21886c316fe`
(in the **Skill References** database)

The rendered run report must follow that page exactly — its Run Header, Sources Used table,
Findings blocks, Run Summary, Skill Runs logging spec, and Teams summary spec. If the URL fails,
locate the page with `notion-search` (query: `"Output Template — Person-Level Leads"`). If it is
still unavailable, render the report with the same section order (Header → Sources → Findings →
Summary) and flag in the output that the template could not be fetched.

### B. Compute signal coverage (never hardcode)

This skill's detection mechanism: employer or title changes for leads already in Zoho CRM, via
Apollo people-match lookups keyed on each lead's stored Apollo Contact ID.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-job-change`) and these
   coverage terms: `"job change"`, `"role transition"`, `"executive move"`, `"champion change"`,
   `"known lead"`. (SQL via `notion-query-data-sources` is plan-gated on this workspace — do
   not rely on it.)
2. `notion-fetch` each candidate row. A row is **covered** if its Skill Coverage property
   (currently named `TEMP - Skill Coverage`) names this skill, or its `Signal Definition` /
   `Observable Evidence` clearly matches the detection mechanism above.
3. Render the header line `**Coverage:** SIG-0XX — <Signal Name>; …` and keep each covered row's
   **Why It Matters**, **Hidden Hypothesis**, **Eclipse Wedge**, **Recommended Disposition**, and
   **Target Personas** — the report's Findings blocks must be grounded in these fields, not in
   invented framing.

If no row matches, render `**Coverage:** none mapped in Signal Library` and flag it for the team
in the report and the Teams summary.

### C. Compute source & source-type scoring (never hardcode)

For every source actually consulted this run (the Apollo platform: people match; the Zoho CRM
Inbound (Leads) module the changes are compared against), build the template's
**Sources Used** table:

1. Resolve the source in the **Source Catalog**
   (`collection://2485d7c1-f09c-46a4-abed-e6991c6932d8`) by `Source URL` or `Source Name`
   (scoped `notion-search`, then `notion-fetch` the row) → get its `SRC-0XX` Source ID.
2. Follow the row's **Source Type** relation into **Source Type Scoring**
   (`collection://3904e475-1c3a-809b-92f8-000b78de539f`) and report the Source Type plus its
   **Default Confidence Score**, **Default Source Reliability**, and **Default Signal Strength**.
   If the Source Catalog row carries its own `Source Reliability` / `Default Source Strength`,
   those per-source values override the type defaults.
3. A consulted source with no Source Catalog row is still listed, flagged `⚠️ uncataloged`, with
   best-judgment scores and a note asking the team to add it to the Source Catalog.

Cache these lookups within a run (one lookup per distinct source) — never across runs.

### D. Log the run to Skill Runs (Notion — mandatory, test runs included)

Create one page per run in the **Skill Runs** database
(`https://app.notion.com/p/3874e4751c3a8084a89be17b28e4c6a1`, data source
`collection://3874e475-1c3a-80a1-8ed7-000ba308ec09`) via `notion-create-pages`:

- **Name:** `bdr-signal-job-change — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-job-change`
- **Multi-select:** provider(s) actually used this run (`Apollo`)
- **Type:** `live run` (or `test run` for dry runs / tests)
- **Page body:** the full rendered report from section A, followed by a toggle block containing
  the run's raw JSON output.

Keep the created page's URL — the Teams summary links to it.

### E. Post the run summary to Teams (mandatory)

Compose the short summary exactly per the template's Teams section (bold headline, 1-sentence
outcome, top-3 bullets, link to the Skill Runs page) and deliver it by invoking the
**bdr-post-to-teams** skill. Post even when the run found nothing — a clean-scan note with the
coverage line is still a result.

---

## Eclipse Context

**Signal:** ICP Role Transition — per Buy Trigger Definitions, role changes for known prospects
are a primary buying trigger. New leaders assess programs in the first 90 days.

**Action after signal:** Outreach acknowledging the move, not referencing prior role.

