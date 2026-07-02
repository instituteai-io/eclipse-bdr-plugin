---
name: bdr-signal-job-change-zoominfo
description: >
  ZoomInfo variant of the Job Change signal. Detects role changes for known leads already in
  Zoho CRM by enriching each one against ZoomInfo (by stored ZoomInfo Person ID if present,
  else name + company) and flagging any who have changed employer or title since the record
  was last updated. Uses ZoomInfo's positionStartDate to corroborate the change. Updates the
  matching Zoho Lead record with the new signal and returns a JSON summary. Built to run
  side-by-side with bdr-signal-job-change (Apollo) for a data-provider comparison.
  Use when asked to check for job changes via ZoomInfo, run the ZoomInfo job change scan,
  or compare ZoomInfo vs Apollo on role transitions.
  Triggers on: "check for job changes with zoominfo", "run zoominfo job change signal",
  "zoominfo role transition scan", "bdr-signal-job-change-zoominfo".
  Every run renders the standardized Notion output template, computes live signal/source
  coverage from the Signal & Source Library, logs the run to the Skill Runs database, and
  posts a summary to Teams.
---

# Eclipse BDR — Signal: Job Change (Known Leads) — ZoomInfo

Detect employer or title changes for leads already in the Zoho Inbound (Leads) module,
using **ZoomInfo** as the data provider. Updates existing records — never creates new ones.
The JSON summary is the data payload; the user-facing deliverable is the templated report
(see 📐 Output Standard).

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

This is the ZoomInfo twin of `bdr-signal-job-change` (Apollo). Same criteria page, same Zoho
fields, same output shape — only the data-provider layer differs, so the two can be run on the
same lead set and diffed.

---

## ⚠️ Criteria Reference Page

**Source (Notion):** `https://app.notion.com/p/7de4e4751c3a83da9ecf8121e82532a7`
(page *Skill References - Signal: Job Change (Known Leads)*, in the **Skill References** database)

This is the **same** reference page the Apollo job-change skill uses — targeting criteria are
provider-agnostic. Fetch via the Notion MCP `notion-fetch` tool with the URL above. This page defines:
- Which Zoho Lead fields to compare against ZoomInfo (e.g. `Company`, `Designation`)
- What qualifies as a "significant" change worth flagging
- Any lead filters to apply before running (e.g. only check leads created in the last N days,
  or only leads in a specific practice area)

Always fetch it fresh — never use remembered criteria. If unreachable, stop and tell the user.

---

## 💳 Credits Note

ZoomInfo `enrich_contacts` consumes **ZoomInfo Bulk Credits** (unlike Apollo's match endpoint).
Once a contact is enriched, re-enriching the same contact is free for one year.

Before running the enrichment batch, surface a one-line estimate to the user:
> "This will enrich up to N contacts against ZoomInfo (≈ up to N bulk credits, minus any
> enriched in the last 12 months). Proceed?"

Proceed once the user confirms. Do not prompt per-contact.

---

## Step 1: Fetch Criteria

Fetch the criteria reference page. Extract:

- `fields_to_compare` — list of Zoho field names to compare against ZoomInfo data (e.g. `["Company", "Designation"]`)
- `lead_filter` — optional filter for which leads to check (e.g. `practice_area = "Data Management & Analytics"`, or `all`)
- `lookback_days` — only check leads whose `Signal_Date` is older than this many days (avoids re-triggering too soon)
- `require_provider_id` — `true` / `false`: whether to skip leads missing a stored provider contact ID

If the page is unreachable or empty, stop and ask the user. Do not proceed with guessed criteria.

---

## Step 2: Pull Known Leads from Zoho

Query the Zoho Inbound (Leads) module via Zoho MCP (`getRecords` or `executeCOQLQuery`).

Apply the `lead_filter` from Step 1. At minimum, include these fields in the response:
`id`, `First_Name`, `Last_Name`, `Company`, `Designation`, `ZoomInfo_Contact_ID`,
`Apollo_Contact_ID`, `Signal_Date`

**ID handling:**
- Prefer `ZoomInfo_Contact_ID` (a ZoomInfo `personId`) for the most accurate enrichment. ✖ not stored in Zoho (schema final)
- If `require_provider_id` is `true` and a lead has no `ZoomInfo_Contact_ID`, fall back to
  name + company matching (see Step 3). Most existing leads were created from Apollo and will
  **not** have a ZoomInfo ID yet — this is expected; name + company is the normal path on first run.

Log the count of leads matched by ID vs by name+company in the final summary.

Batch in groups of **10** (the `enrich_contacts` per-call maximum).

---

## Step 3: Enrich Against ZoomInfo

For each batch (max 10 leads), call `enrich_contacts` with one contact object per lead:

| Lead field | ZoomInfo identification (in priority order) |
|---|---|
| `ZoomInfo_Contact_ID` | `personId` (direct lookup — most accurate) |
| `First_Name` + `Last_Name` + `Company` | `firstName` + `lastName` + `companyName` (fallback) |

Request these `requiredFields`:
`jobTitle`, `companyName`, `managementLevel`, `positionStartDate`, `externalUrls`,
`contactAccuracyScore`, `lastUpdatedDate`, `zoominfoCompanyId`

From each enriched result, capture:
- `companyName` → maps to `Company`
- `jobTitle` → maps to `Designation`
- `positionStartDate` → corroboration signal (recent date = strong evidence of a real move)
- `externalUrls` (LinkedIn) → update if changed
- `personId` → store back to `ZoomInfo_Contact_ID` for future runs

If ZoomInfo returns no match for a lead: log as `no_zoominfo_match`, skip, continue.

---

## Step 4: Detect Changes

Compare each field listed in `fields_to_compare` (from Step 1) between the Zoho record and the
ZoomInfo result.

A **job change** is detected when:
- `Company` differs (employer change), OR
- `Designation` differs (title change)

String comparison: case-insensitive, trim whitespace. Ignore minor formatting differences
(e.g. "JPMorgan Chase & Co." vs "JPMorgan Chase"). Use contains/fuzzy only if the criteria
page explicitly instructs it.

**Corroboration:** if `positionStartDate` falls within `lookback_days` of today, mark the change
`corroborated: true`. A title/employer diff *with* a recent start date is high-confidence; a diff
*without* one may be a stale ZoomInfo record — still flag it, but note `corroborated: false`.

For each change detected, build a change record:

```
{
  "zoho_lead_id": "<id>",
  "zoominfo_contact_id": "<personId>",
  "first_name": "<first>",
  "last_name": "<last>",
  "matched_by": "zoominfo_id | name_company",
  "corroborated": true,
  "position_start_date": "<YYYY-MM-DD or null>",
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
> ZoomInfo shows they are now [new title] at [new company] (started [positionStartDate]) as of [today's date]."

If only the title changed (same company):
> "[First name] was [old title] at [company].
> ZoomInfo shows a title change to [new title] as of [today's date]."

Omit the `(started …)` clause if `positionStartDate` is null.

---

## Step 6: Update Zoho Lead

For each changed lead, call `updateRecord` on module `Leads` with the Zoho record ID.

Fields to update:

| Field | Value |
|---|---|
| `Company` | ZoomInfo `companyName` (if changed) |
| `Designation` | ZoomInfo `jobTitle` (if changed) |
| `ZoomInfo_Contact_ID` | ZoomInfo `personId` ✖ not in Zoho (schema final) — `zoho-crud-lead` drops it |
| `Signal_Type` | `"Job Change"` ✅ |
| `Signal_Date` | Today's date ✅ |
| `Signal_Summary` | Generated in Step 5 ✅ |

✅ = live in Zoho (verified 2026-07-02). ✖ = not in Zoho and not coming (schema final) — `zoho-crud-lead` handles the fallback, log in result.

Do **not** update any other fields. Do **not** create new records. Do **not** touch any module
other than Leads.

---

## Step 7: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log and any downstream handoff. The
user-facing deliverable is the templated report (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-job-change-zoominfo",
  "signal_source": "ZoomInfo",
  "criteria_fetched_from": "<URL fetched in Step 1>",
  "summary": {
    "leads_checked": 0,
    "matched_by_zoominfo_id": 0,
    "matched_by_name_company": 0,
    "skipped_no_provider_id": 0,
    "skipped_no_zoominfo_match": 0,
    "job_changes_detected": 0,
    "job_changes_corroborated": 0,
    "zoho_records_updated": 0,
    "zoho_update_errors": 0,
    "fields_skipped_pending": []
  },
  "changes": [
    {
      "zoho_lead_id": "<id>",
      "zoominfo_contact_id": "<personId>",
      "first_name": "<first>",
      "last_name": "<last>",
      "matched_by": "zoominfo_id | name_company",
      "corroborated": true,
      "position_start_date": "<YYYY-MM-DD or null>",
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

## ZoomInfo vs Apollo — what to compare

When run against the same lead set as `bdr-signal-job-change`, compare:
- **Match rate** — `no_zoominfo_match` vs Apollo's `no_apollo_match`
- **Changes detected** — does one provider catch moves the other misses?
- **Corroboration** — ZoomInfo's `positionStartDate` lets it distinguish a real recent move
  from a stale record; Apollo has no equivalent, so its diffs are lower-confidence.
- **Cost** — ZoomInfo burns bulk credits per enrichment; Apollo's match does not.

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
Findings blocks, and Run Summary. The template governs the report format only; the Skill Runs
logging and Teams delivery below are this skill's own piping, defined here. If the URL fails,
locate the page with `notion-search` (query: `"Output Template — Person-Level Leads"`). If it is
still unavailable, render the report with the same section order (Header → Sources → Findings →
Summary) and flag in the output that the template could not be fetched.

### B. Compute signal coverage (never hardcode)

This skill's detection mechanism: employer or title changes for leads already in Zoho CRM, via
ZoomInfo contact enrichment with position-start-date corroboration.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-job-change-zoominfo`) and
   these coverage terms: `"job change"`, `"role transition"`, `"executive move"`,
   `"champion change"`, `"known lead"`. (SQL via `notion-query-data-sources` is plan-gated on
   this workspace — do not rely on it.)
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

For every source actually consulted this run (the ZoomInfo platform: contact enrichment; the
Zoho CRM Inbound (Leads) module the changes are compared against), build the template's
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

- **Name:** `bdr-signal-job-change-zoominfo — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-job-change-zoominfo`
- **Multi-select:** provider(s) actually used this run (`Zoominfo`)
- **Type:** `live run` (or `test run` for dry runs / tests)
- **Page body:** the full rendered report from section A, followed by a toggle block containing
  the run's raw JSON output.

Keep the created page's URL — the Teams summary links to it.

### E. Post the run summary to Teams (mandatory)

Compose the short summary and deliver it by invoking the **bdr-post-to-teams** skill.
Microsoft Teams renders `*single asterisks*` as bold and `_underscores_` as italic — `**double
asterisks**`, Markdown tables, and headings do NOT render. Power Automate collapses single
newlines, so put a blank line between every block AND every bullet, or the message displays as
a wall of text:

```
*<emoji> <Skill headline> — <YYYY-MM-DD>*

<1-sentence outcome: n people surfaced, m enriched.>

• <Person 1 — title, company, one-clause signal>

• <Person 2 — …>

• <Person 3 — …>

Full report: <link to the Skill Runs page created in section D>
```

Cap at the top 3 people. Post even when the run found nothing — a clean-scan note with the
coverage line is still a result.

---

## Eclipse Context

**Signal:** ICP Role Transition — per Buy Trigger Definitions, role changes for known prospects
are a primary buying trigger. New leaders assess programs in the first 90 days.

**Action after signal:** Outreach acknowledging the move, not referencing prior role.
