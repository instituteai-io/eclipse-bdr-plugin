---
name: bdr-signal-hiring-data
description: >
  Detect ICP-fit companies that are actively hiring for Data roles — from broad data
  leadership/engineering roles (Chief Data Officer, Data Engineer, Data Architect) to
  governance operating-model roles (Data Governance Lead, Data Steward, Metadata Manager,
  Data Quality Lead, Data Catalog, Master Data). Data management and data governance are
  treated as one signal; the full keyword list is client-managed in Notion, never
  hardcoded here. Fetches company ICP criteria and hiring keyword list fresh from the
  team's Notion reference page, queries Apollo for companies matching the ICP profile,
  checks their active job postings for matching openings (capturing the posting date so
  signal freshness is known), surfaces the ICP decision-maker(s) at those companies, then
  enriches the qualified leads for real contact info. Output is a JSON payload ready for
  the zoho-crud-lead skill.
  Use when asked to find companies hiring for data or data governance roles, run the
  hiring signal scan, detect data hiring activity at ICP companies, find who is building
  out a data or governance team, or surface data talent demand signals.
  Triggers on: "run hiring signal scan", "which ICP companies are hiring for data",
  "data hiring signal", "who's hiring data stewards", "governance hiring signal",
  "run the governance hiring scan", "bdr-signal-hiring-data".
  Every run renders the standardized Notion output template, computes live signal/source
  coverage from the Signal & Source Library, logs the run to the Skill Runs database, and
  posts a summary to Teams.
---

# Eclipse BDR — Signal: Company Hiring (Data)

Surface ICP-fit companies with active Data job postings — leadership, engineering, and
governance operating-model roles alike — then find the ICP decision-maker at those
companies and enrich them for real contact info. The structured JSON payload hands off to
`zoho-crud-lead` to write to Inbound (Leads); the user-facing deliverable is the templated
report (see 📐 Output Standard below).

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

Data management and data governance hiring are **one consolidated signal**. The precision
distinction lives in the keyword list, which the client manages in Notion (Section B of the reference page) —
not in separate skills and not hardcoded here.

---

## ⚠️ Criteria Reference Page

One reference page drives this skill, carrying both sections. Always fetch it fresh —
never use cached values.

**Reference (Notion) — Skill References - Signal: Company Hiring (Data):**
`https://app.notion.com/p/23e4e4751c3a827480e10109d6538c63`
(in the **Skill References** database)

Fetch fresh via the Notion MCP `notion-fetch` tool on every run.

Provides:
- **Section A — ICP Company & Contact Criteria:** `titles`, `seniority`, `industries`,
  `company_size`, `geography`, `include_similar_titles`
- **Section B — Data Role Keywords:** `data_role_keywords` — the full list of job title
  keywords to match against open postings (covers both broad data roles and governance
  operating-model roles), and `min_postings`

If the reference page is unreachable or empty, stop and tell the user. Do not proceed
with guessed or hardcoded criteria or keywords.

---

## Step 1: Fetch Criteria

**From Section A**, extract:
- `titles` — ICP contact titles to find at hiring companies
- `seniority` — seniority levels
- `industries` — industry filters
- `company_size` — employee range
- `geography` — locations
- `include_similar_titles`

**From Section B**, extract:
- `data_role_keywords` — keywords to match against job posting titles (case-insensitive
  substring match)
- `min_postings` — minimum number of matching postings to qualify a company as a
  "hiring signal" (default: `1` if not specified)

---

## Step 2: Find ICP Companies with Data Job Postings

Call `apollo_mixed_companies_search` using company-level criteria from Section A:

| Criteria field | Apollo parameter |
|---|---|
| `industries` | `q_organization_keyword_tags` |
| `company_size` | `organization_num_employees_ranges` |
| `geography` | `organization_locations` |

Fixed parameters:
- `per_page`: 25

For each company returned, call `apollo_organizations_job_postings` with the company's
Apollo organization `id` (returned by the company search — the endpoint does not accept domains).

Filter the returned job postings: a posting **matches** if its title contains any keyword
from `data_role_keywords` (case-insensitive substring match — match on **any one** keyword,
never require all of them).

**Capture the signal mechanism and freshness for every matched posting** (the team wants to
know how old each signal is — fresher postings are higher-intent):
- `signal_mechanism`: `"Apollo job posting"`
- `posted_date`: the posting's `posted_at` date (YYYY-MM-DD). If Apollo does not return a
  date for that posting, set `null`.
- `age_days`: whole days between `posted_date` and the scan date, or `null` if no date.
- `location`: the posting location if returned (helps catch foreign reqs at a US-HQ'd company).

Keep only companies where the count of matching postings ≥ `min_postings`. For each such
company, record the **freshest** matched posting date as `latest_signal_date` and its
`signal_age_days`.

These are your **hiring signal companies**.

---

## Step 3: Find ICP Contacts at Hiring Signal Companies

The hiring activity is a **company-level** signal. This step finds the **person Eclipse would
actually pitch** — the data decision-maker / leader at the signal company (e.g. the CDO, VP of
Data, Head of Data Governance). This is **not** the recruiter or whoever posted the req; it is
the ICP buyer whose team the new headcount rolls up into. A company staffing a governance
program needs an owner for it, and that owner is the lead.

For each hiring signal company, call `apollo_mixed_people_api_search` to find contacts:

| Criteria field | Apollo parameter |
|---|---|
| `titles` (from Section A) | `person_titles` |
| `seniority` (from Section A) | `person_seniorities` |
| company domain | `q_organization_domains` |
| `include_similar_titles` | `include_similar_titles` |

Fixed parameters:
- `per_page`: 10 per company (surface the best matches, not an exhaustive list)

Note: the people **search** response masks last names and omits email/phone — that is expected.
Real contact info is revealed in Step 5 (enrichment), and only for leads that qualify.

---

## Step 4: Score Results

For each person returned, score Eclipse relevance 1–10.

| Condition | Points |
|---|---|
| Company is a bank or credit union | +3 |
| Company is an asset/investment manager or PE firm | +2 |
| Company is a fintech, payments, or insurance firm | +1 |
| Person is C-suite (CDO, CAO, CDAO, CIO, CTO) | +3 |
| Person is VP-level | +2 |
| Person is Director-level | +1 |
| Company has ≥ 2 matching postings (program build, not a single backfill) | +1 |
| Company has 1,000–10,000 employees | +1 |
| No company size data available | -1 |

**ICP match:** `true` if score ≥ 5.

**ICP role match:**
- C-suite data title → `"Primary ICP"`
- VP or Director data title → `"Secondary ICP"`

---

## Step 5: Enrich Qualified Leads 💳

Only **after** scoring, enrich the contacts with `icp_match: true` (score ≥ 5) to reveal real
contact data. **Enrich qualified leads only** — never the full scanned set — to keep credit
spend tight.

Call `apollo_people_bulk_match` with the qualified contacts' Apollo person `id`s (one bulk call
for all of them), setting `reveal_personal_emails: true` and `reveal_phone_number: true`.

⚠️ **Credit cost:** Apollo people enrichment consumes ≈ 1 enrichment credit per record revealed
(phone reveal may be metered separately on some plans). This is the only credit-consuming step
in the skill; the scan steps above are not bulk-credit enrichments. Always report the number of
records enriched in the run summary so credit usage is visible.

From each enriched record, populate: full `last_name` (unmasked), `email`, `phone`,
`linkedin_url`, and numeric `employee_count` / `industry` where Apollo returns them. If a
specific reveal fails (no email on file, etc.), leave that field `null` and note it — do not
re-enrich.

If the team explicitly wants a no-cost dry run, this step may be skipped; in that case last
names stay masked and email/phone stay `null`. State clearly in the summary that enrichment was
skipped.

---

## Step 6: Build CRM Payload

For every result with `icp_match: true`, build one lead record for the Inbound (Leads) module.

Field mapping:

| Generic | Zoho API Name | Notes |
|---|---|---|
| `first_name` | `First_Name` | |
| `last_name` | `Last_Name` | Unmasked from Step 5; flag if reveal failed |
| `title` | `Designation` | |
| `company` | `Company` | |
| `email` | `Email` | From Step 5 enrichment; null if none on file |
| `phone` | `Phone` | From Step 5 enrichment; null if none on file |
| `linkedin_url` | `LinkedIn_Page_URL` | From Step 5 enrichment |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | Always `["Data Management & Analytics"]` |
| `lead_source` | `Lead_Source` | Always `"Apollo ICP Scan"` ✅ (live picklist value) |
| `apollo_contact_id` | `Apollo_Contact_ID` | ✅ Live in Zoho |
| `signal_type` | `Signal_Type` | Always `"Company Hiring"` ✅ Live in Zoho |
| `signal_mechanism` | `Signal_Mechanism` | Always `"Apollo job posting"` No Zoho field — `zoho-crud-lead` folds it into `Signal_Summary` |
| `signal_date` | `Signal_Date` | Date WE detected the signal = scan date ✅ Live in Zoho |
| `signal_source_date` | `Signal_Source_Date` | The posting's own date (`latest_signal_date`); how old the signal actually is No Zoho field — `zoho-crud-lead` folds it into `Signal_Summary` |
| `signal_summary` | `Signal_Summary` | Generated in Step 7 ✅ Live in Zoho |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will
the Signal custom fields are live in Zoho (verified 2026-07-02); `zoho-crud-lead` skips any that are still missing.

**Dedup field:** `Apollo_Contact_ID`. Dedup fallback: `First_Name + Last_Name + Company`.

---

## Step 7: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max). Include the signal's age so the
reader knows how fresh it is:

> "[Company] is actively hiring for Data roles ([list matched posting titles, max 3]; latest posted
> [signal_age_days] days ago). [First name] is [title] there — surfaced as a Data Management &
> Analytics signal on [scan date]."

If the matched postings are predominantly governance operating-model roles (e.g. Data
Governance, Data Steward, Metadata, Data Catalog), note it:

> "[Company] is staffing a data governance program ([list matched posting titles, max 3]; latest
> posted [signal_age_days] days ago). [First name] is [title] there — surfaced as a Data
> Management & Analytics signal on [scan date]."

If no posting date was available, drop the "latest posted …" clause rather than guessing.

---

## Step 8: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log (embedded in the run page's toggle
block) and the downstream `zoho-crud-lead` handoff. The user-facing deliverable is the
templated report plus the Skill Runs log entry and Teams summary (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-hiring-data",
  "signal_source": "Apollo",
  "signal_mechanism": "Apollo job posting",
  "criteria_fetched_from": {
    "reference_page": "<reference page URL/URI>"
  },
  "summary": {
    "companies_scanned": 0,
    "companies_with_hiring_signal": 0,
    "total_contacts_found": 0,
    "icp_matched": 0,
    "primary_icp": 0,
    "secondary_icp": 0,
    "leads_enriched": 0,
    "emails_found": 0,
    "phones_found": 0,
    "masked_last_names_remaining": 0,
    "enrichment_credits_estimated": 0,
    "enrichment_skipped": false
  },
  "hiring_signal_companies": [
    {
      "company_name": "<name>",
      "domain": "<domain>",
      "signal_mechanism": "Apollo job posting",
      "latest_signal_date": "<YYYY-MM-DD or null>",
      "signal_age_days": 0,
      "matched_postings": [
        {"title": "<title1>", "posted_date": "<YYYY-MM-DD or null>", "age_days": 0, "location": "<loc or null>"}
      ]
    }
  ],
  "crm_payload": {
    "generated_by": "bdr-signal-hiring-data",
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
          "Last_Name": "<last — note MASKED if reveal failed>",
          "Designation": "<title>",
          "Company": "<company>",
          "Email": "<email or null>",
          "Phone": "<phone or null>",
          "LinkedIn_Page_URL": "<url or null>",
          "No_of_Employees": 0,
          "Industry": "<mapped value>",
          "Practice_Areas_In_Scope": ["Data Management & Analytics"],
          "Lead_Source": "Apollo ICP Scan",
          "Apollo_Contact_ID": "<apollo person id>",
          "Signal_Type": "Company Hiring",
          "Signal_Mechanism": "Apollo job posting",
          "Signal_Date": "<today YYYY-MM-DD>",
          "Signal_Source_Date": "<latest_signal_date or null>",
          "Signal_Summary": "<generated summary>"
        },
        "meta": {
          "icp_role_match": "Primary ICP | Secondary ICP",
          "relevance_score": 0,
          "enriched": true,
          "hiring_signal_company": "<company name>",
          "signal_mechanism": "Apollo job posting",
          "latest_signal_date": "<YYYY-MM-DD or null>",
          "signal_age_days": 0,
          "matched_postings": [
            {"title": "<title1>", "posted_date": "<YYYY-MM-DD or null>", "age_days": 0}
          ]
        }
      }
    ]
  }
}
```

- IDs sequential: `lead_001`, `lead_002`, etc.
- `meta` block is for human review only — not written to Zoho
- If no hiring signals found: return empty `leads: []` and `hiring_signal_companies: []`

---

## 📐 Output Standard (mandatory — every run)

Every run of this skill ends by rendering a standardized report, logging the run to Notion, and
posting a summary to Microsoft Teams. Nothing in this section is hardcoded: signal coverage,
source scoring, and the report format are all fetched fresh from Notion on every run.

### A. Fetch the output template (format authority)

Fetch this page fresh at the start of every run (Notion MCP `notion-fetch`):

**Output Template — Company Signals & Contacts** —
`https://app.notion.com/p/3914e4751c3a81f3b5c9f379246f54dc`
(in the **Skill References** database)

The rendered run report must follow that page exactly — its Run Header, Sources Used table,
Findings blocks, and Run Summary. The template governs the report format only; the Skill Runs
logging and Teams delivery below are this skill's own piping, defined here. If the URL fails,
locate the page with `notion-search` (query: `"Output Template — Company Signals & Contacts"`).
If it is still unavailable, render the report with the same section order (Header → Sources →
Findings → Summary) and flag in the output that the template could not be fetched.

### B. Compute signal coverage (never hardcode)

This skill's detection mechanism: active Data and data-governance role job postings at ICP-fit
companies, via the Apollo job-postings endpoint.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-hiring-data`) and these
   coverage terms: "data hiring", "data governance hiring", "data steward", "metadata",
   "hiring cluster". (SQL via `notion-query-data-sources` is plan-gated on this workspace — do
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

For every source actually consulted this run (the Apollo platform: company search + job postings
+ people search and enrichment), build the template's **Sources Used** table:

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

- **Name:** `bdr-signal-hiring-data — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-hiring-data`
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

<1-sentence outcome: n signal companies, m ICP leads enriched.>

• <Company 1 — one-clause signal + top contact name/title>

• <Company 2 — …>

• <Company 3 — …>

Full report: <link to the Skill Runs page created in section D>
```

Cap at the top 3 companies. Post even when the run found nothing — a clean-scan note with the
coverage line is still a result.

---

## Eclipse Context

**Practice area:** Data Management & Analytics (data governance is part of this practice —
the terms are used interchangeably in this signal).

**Signal:** A company actively hiring for data roles is building or rebuilding a data
program — high-intent signal that they may need external advisory support for governance
frameworks, data quality, or strategy. Governance operating-model postings (Data Governance
Lead, Data Steward, Metadata Manager, Data Quality Lead) are the strongest form of this
signal: per the Signal Library, they show the program is being **operationalized**, not just
discussed. Hidden hypothesis: the company likely bought or is implementing a tool but lacks
the people/process model to scale it. **Signal freshness matters** — a posting from this week
is a hotter trigger than one from three months ago, so the posting date is captured and carried
through to the lead.

**Who we surface:** the data **decision-maker** at the signal company (CDO / VP Data / Head of
Data Governance) — the buyer for the engagement — not the person who posted the job.

**Eclipse wedge:** Governance frameworks, role design, stewardship playbooks, operating-model
support, data quality programs, and strategy — turning new data headcount into a working
program.

**History:** Consolidated from two earlier skills (`bdr-scan-hiring-data`, broad data roles;
`bdr-scan-hiring-governance`, governance subset). The split proved artificial — the
client-managed Notion keyword list already spans both role families, so precision
is tuned there, not via separate skills.
</content>
