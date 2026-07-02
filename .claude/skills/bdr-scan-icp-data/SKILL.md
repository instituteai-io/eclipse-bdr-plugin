---
name: bdr-scan-icp-data
description: >
  Scan Apollo for ICP contacts in the Data Management practice area. Fetches live search
  criteria from the team's reference page, queries Apollo for matching people, scores each
  result for Eclipse relevance, and returns a JSON payload ready for the zoho-crud-lead skill.
  Use when asked to run the Data ICP scan, find data governance contacts in Apollo,
  scan for CDO or data leadership signals, or surface Data Management opportunities.
  Triggers on: "run the data ICP scan", "check Apollo for data contacts",
  "find CDOs in Apollo", "data governance signals", "ICP Signal Data".
  Every run renders the standardized Notion output template, computes live signal/source
  coverage from the Signal & Source Library, logs the run to the Skill Runs database, and
  posts a summary to Teams.
---

# Eclipse BDR — ICP Signal: Data Management

Scan Apollo for contacts matching Eclipse's Data Management ICP. The JSON payload is the data
handoff to `zoho-crud-lead`; the user-facing deliverable is the templated report (see
📐 Output Standard).

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

---

## ⚠️ Criteria Reference Page

**Source (Notion):** `https://app.notion.com/p/23e4e4751c3a827480e10109d6538c63`
(page *Signal: Company Hiring (Data)*, in the **Skill References** database — read **Section A —
ICP Company & Contact Criteria**, which this skill shares with `bdr-signal-hiring-data`; Section C
on the same page carries the fuller Data practice ICP detail)

Fetch via the Notion MCP `notion-fetch` tool with the URL above. This is the single source of
truth for search criteria. Always fetch it fresh — never use remembered or cached criteria from a prior run.
If unreachable, stop and tell the user.

---

## Step 1: Fetch Criteria

Fetch the criteria reference page. Extract:

- `titles` — list of job titles
- `seniority` — seniority levels
- `industries` — industries / keyword tags
- `company_size` — employee range
- `geography` — locations
- `include_similar_titles` — true/false

If the page is unreachable or empty, stop and ask the user to provide criteria manually
or check the URL. Do not proceed with guessed or cached criteria.

---

## Step 2: Run Apollo Search

Call `apollo_mixed_people_api_search` using exactly the criteria from Step 1.

| Criteria field | Apollo parameter |
|---|---|
| `titles` | `person_titles` |
| `seniority` | `person_seniorities` (map: C-Suite → `c_suite`, VP → `vp`, Director → `director`) |
| `industries` | `q_organization_keyword_tags` |
| `company_size` | `organization_num_employees_ranges` (e.g. "500,2000") |
| `geography` | `person_locations` |
| `include_similar_titles` | `include_similar_titles` |

Fixed parameters:
- `per_page`: 25 (or as specified by the user)
- `page`: 1

---

## Step 3: Score Results

For each person returned, score Eclipse relevance 1–10.

| Condition | Points |
|---|---|
| Company is a bank or credit union | +3 |
| Company is an asset/investment manager or PE firm | +2 |
| Company is a fintech, payments, or insurance firm | +1 |
| Person is C-suite (CDO, CAO, CDAO, CIO, CTO) | +3 |
| Person is VP-level | +2 |
| Person is Director-level | +1 |
| Company has 1,000–10,000 employees | +1 |
| No company size data available | -1 |

**ICP match:** `true` if score ≥ 5.

**ICP role match:**
- C-suite data title → `"Primary ICP"`
- VP or Director data title → `"Secondary ICP"`

---

## Step 4: Build CRM Payload

For every result with `icp_match: true`, build one lead record for the Inbound (Leads) module.

Field mapping:

| Generic | Zoho API Name | Notes |
|---|---|---|
| `first_name` | `First_Name` | |
| `last_name` | `Last_Name` | Flag if masked by Apollo |
| `title` | `Designation` | |
| `company` | `Company` | |
| `email` | `Email` | Null if not available — enrichment required |
| `phone` | `Phone` | Null if not available |
| `linkedin_url` | `LinkedIn_Page_URL` | |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | Always `["Data Management & Analytics"]` for this skill |
| `lead_source` | `Lead_Source` | Always `"Apollo ICP Scan"` ✅ (live picklist value) |
| `apollo_contact_id` | `Apollo_Contact_ID` | ✅ Live in Zoho |
| `signal_type` | `Signal_Type` | Always `"New Hire"` — ✅ Live in Zoho |
| `signal_date` | `Signal_Date` | Today's date — ✅ Live in Zoho |
| `signal_summary` | `Signal_Summary` | Generated in Step 5 — ✅ Live in Zoho |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will
the Signal custom fields are live in Zoho (verified 2026-07-02); `zoho-crud-lead` skips any that are still missing.

**Dedup field:** `Apollo_Contact_ID`. Dedup fallback: `First_Name + Last_Name + Company`.

---

## Step 5: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max):

> "[First name] is [title] at [company], a [company type] with ~[size] employees.
> Surfaced by Apollo ICP scan for the Data Management practice area on [date]."

---

## Step 6: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log and the `zoho-crud-lead` handoff. The
user-facing deliverable is the templated report (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-scan-icp-data",
  "signal_source": "Apollo",
  "criteria_fetched_from": "<URL fetched in Step 1>",
  "summary": {
    "total_returned_by_apollo": 0,
    "icp_matched": 0,
    "primary_icp": 0,
    "secondary_icp": 0,
    "masked_last_names": 0
  },
  "crm_payload": {
    "generated_by": "bdr-scan-icp-data",
    "generated_at": "<today YYYY-MM-DD>",
    "module": "Leads",
    "action": "upsert",
    "dedup_field": "Apollo_Contact_ID",
    "dedup_fallback": "First_Name + Last_Name + Company",
    "leads": [
      {
        "id": "lead_001",
        "fields": {
          "First_Name": "<first>",
          "Last_Name": "<last — note MASKED if applicable>",
          "Designation": "<title>",
          "Company": "<company>",
          "Email": null,
          "Phone": null,
          "LinkedIn_Page_URL": "<url or null>",
          "No_of_Employees": 0,
          "Industry": "<mapped value>",
          "Practice_Areas_In_Scope": ["Data Management & Analytics"],
          "Lead_Source": "Apollo ICP Scan",
          "Apollo_Contact_ID": "<apollo person id>",
          "Signal_Type": "New Hire",
          "Signal_Date": "<today YYYY-MM-DD>",
          "Signal_Summary": "<generated summary>"
        },
        "meta": {
          "icp_role_match": "Primary ICP | Secondary ICP",
          "relevance_score": 0
        }
      }
    ]
  }
}
```

- IDs sequential: `lead_001`, `lead_002`, etc.
- `meta` block is for human review only — not written to Zoho
- If Apollo returns no results: return empty `leads: []` with a note in `summary`

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

This skill's detection mechanism: newly surfaced ICP-fit Data Management contacts (Chief Data
Officer and other data leadership titles) at financial-services companies, via the Apollo people
search.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-scan-icp-data`) and these coverage
   terms: `"Chief Data Officer"`, `"data governance"`, `"data leadership"`, `"new hire"`,
   `"ICP scan"`. (SQL via `notion-query-data-sources` is plan-gated on this workspace — do
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

For every source actually consulted this run (the Apollo platform: people search), build the
template's **Sources Used** table:

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

- **Name:** `bdr-scan-icp-data — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-scan-icp-data`
- **Multi-select:** provider(s) actually used this run (`Apollo`)
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

**Practice area:** Data Management (including Analytics)

**What Eclipse does here:** Data governance frameworks, data quality & controls, data strategy
& operating model, metadata/lineage/cataloging, cloud data migration readiness, analytics
enablement.

**Best signal:** A CDO, CDAO, or Head of Data Governance who recently joined a mid-to-large
financial services firm — new leaders assess programs in the first 90 days.
