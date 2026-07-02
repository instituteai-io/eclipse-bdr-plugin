---
name: bdr-signal-hiring-risk
description: >
  Detect ICP-fit companies that are actively hiring for Risk & Regulatory roles —
  regulatory change, risk transformation, financial crime / AML / KYC / sanctions,
  operational risk, internal audit, controls testing, regulatory reporting, model risk,
  and similar. Companies staffing risk & compliance functions are responding to
  regulatory pressure, audit findings, remediation timelines, or funded program
  build-outs. The keyword list and ICP criteria are client-managed in the team's
  reference page, never hardcoded here. Queries Apollo for companies matching the ICP
  profile, checks their active job postings for risk-role openings, then surfaces
  ICP-matching contacts (CRO, CCO, Head of Financial Crime, Head of Regulatory Change,
  etc.) at those companies as leads for the Inbound (Leads) module. Output is a JSON
  payload ready for the zoho-crud-lead skill.
  Use when asked to find companies hiring for risk, compliance, regulatory, or financial
  crime roles, run the risk hiring signal scan, detect risk & regulatory hiring activity
  at ICP companies, or find who is building out a risk or compliance team.
  Triggers on: "run risk hiring signal scan", "signal hiring risk", "which ICP companies
  are hiring for risk", "risk hiring signal", "who's hiring AML/KYC", "compliance hiring
  signal", "regulatory hiring scan", "bdr-signal-hiring-risk".
  Every run renders the standardized Notion output template, computes live signal/source
  coverage from the Signal & Source Library, logs the run to the Skill Runs database, and
  posts a summary to Teams.
---

# Eclipse BDR — Signal: Company Hiring (Risk & Regulatory)

Surface ICP-fit companies with active **Risk & Regulatory** job postings — regulatory
change, financial crime / AML / KYC, operational risk, audit, controls, regulatory
reporting, model risk — then find ICP contacts at those companies. The structured JSON
payload hands off to `zoho-crud-lead` to write to Inbound (Leads); the user-facing
deliverable is the templated report (see 📐 Output Standard below).

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

This is the Risk & Regulatory sibling of `bdr-signal-hiring-data`. Same mechanism, same
output shape; practice area, ICP criteria, and keyword list differ and are entirely
client-managed in the reference page.

---

## ⚠️ Criteria Reference Page

One reference page drives this skill, carrying both sections. Always fetch it fresh —
never use cached values.

**Reference (Notion) — Skill References - Signal: Company Hiring (Risk):**
`https://app.notion.com/p/6ce4e4751c3a8331828b818f9efe37b4`
(in the **Skill References** database)

Fetch fresh via the Notion MCP `notion-fetch` tool on every run.

Provides:
- **Section A — ICP Company & Contact Criteria:** `titles`, `seniority`, `industries`,
  `company_size`, `geography`, `include_similar_titles`
- **Section B — Risk Role Keywords:** `risk_role_keywords` (job title keywords to match
  against open postings), optional financial-risk keyword list (only if enabled on the
  page), and `min_postings`

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
- `risk_role_keywords` — keywords to match against job posting titles (case-insensitive
  substring match). Include the optional financial-risk list only if the page says it is
  enabled.
- `min_postings` — minimum number of matching postings to qualify a company as a
  "hiring signal" (default: `1` if not specified)

---

## Step 2: Find ICP Companies with Risk Job Postings

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
from `risk_role_keywords` (case-insensitive substring match).

Keep only companies where the count of matching postings ≥ `min_postings`.

These are your **hiring signal companies**.

---

## Step 3: Find ICP Contacts at Hiring Signal Companies

For each hiring signal company, call `apollo_mixed_people_api_search` to find contacts:

| Criteria field | Apollo parameter |
|---|---|
| `titles` (from Section A) | `person_titles` |
| `seniority` (from Section A) | `person_seniorities` |
| company domain | `q_organization_domains` |
| `include_similar_titles` | `include_similar_titles` |

Fixed parameters:
- `per_page`: 10 per company (surface the best matches, not an exhaustive list)

---

## Step 4: Score Results

For each person returned, score Eclipse relevance 1–10.

| Condition | Points |
|---|---|
| Company is a bank or credit union | +3 |
| Company is an asset/investment manager or PE firm | +2 |
| Company is a fintech, payments, or insurance firm | +1 |
| Person is C-suite (CRO, CCO, CDO, CIO, CTO) | +3 |
| Person is VP-level | +2 |
| Person is Director-level | +1 |
| Company has ≥ 2 matching postings (program build, not a single backfill) | +1 |
| Company has 1,000–10,000 employees | +1 |
| No company size data available | -1 |

**ICP match:** `true` if score ≥ 5.

**ICP role match:**
- C-suite risk/compliance title → `"Primary ICP"`
- VP or Director risk/compliance title → `"Secondary ICP"`

---

## Step 5: Build CRM Payload

For every result with `icp_match: true`, build one lead record for the Inbound (Leads) module.

Field mapping:

| Generic | Zoho API Name | Notes |
|---|---|---|
| `first_name` | `First_Name` | |
| `last_name` | `Last_Name` | Flag if masked by Apollo |
| `title` | `Designation` | |
| `company` | `Company` | |
| `email` | `Email` | Null if not available |
| `phone` | `Phone` | Null if not available |
| `linkedin_url` | `LinkedIn_Page_URL` | |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | `["Risk & Regulatory"]` — additionally include `"Financial Crimes & Compliance"` when the company's matched postings include financial crime keywords (AML, BSA, KYC, Sanctions, Financial Crime, Transaction Monitoring) |
| `lead_source` | `Lead_Source` | Always `"Apollo ICP Scan"` ✅ (live picklist value) |
| `apollo_contact_id` | `Apollo_Contact_ID` | ✅ Live in Zoho |
| `signal_type` | `Signal_Type` | Always `"Company Hiring"` ✅ Live in Zoho |
| `signal_date` | `Signal_Date` | Today's date ✅ Live in Zoho |
| `signal_summary` | `Signal_Summary` | Generated in Step 6 ✅ Live in Zoho |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will
the Signal custom fields are live in Zoho (verified 2026-07-02); `zoho-crud-lead` skips any that are still missing.

**Dedup field:** `Apollo_Contact_ID`. Dedup fallback: `First_Name + Last_Name + Company`.

---

## Step 6: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max):

> "[Company] is actively hiring for Risk & Regulatory roles ([list matched posting titles,
> max 3]). [First name] is [title] there — surfaced as a Risk & Regulatory signal on [date]."

If the matched postings are predominantly financial crime roles (AML, BSA, KYC, Sanctions,
Financial Crime, Transaction Monitoring), note it:

> "[Company] is staffing a financial crime compliance program ([list matched posting titles,
> max 3]). [First name] is [title] there — surfaced as a Risk & Regulatory signal on [date]."

---

## Step 7: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log (embedded in the run page's toggle
block) and the downstream `zoho-crud-lead` handoff. The user-facing deliverable is the
templated report plus the Skill Runs log entry and Teams summary (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-hiring-risk",
  "signal_source": "Apollo",
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
    "masked_last_names": 0
  },
  "hiring_signal_companies": [
    {
      "company_name": "<name>",
      "domain": "<domain>",
      "matched_postings": ["<title1>", "<title2>"]
    }
  ],
  "crm_payload": {
    "generated_by": "bdr-signal-hiring-risk",
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
          "Practice_Areas_In_Scope": ["Risk & Regulatory"],
          "Lead_Source": "Apollo ICP Scan",
          "Apollo_Contact_ID": "<apollo person id>",
          "Signal_Type": "Company Hiring",
          "Signal_Date": "<today YYYY-MM-DD>",
          "Signal_Summary": "<generated summary>"
        },
        "meta": {
          "icp_role_match": "Primary ICP | Secondary ICP",
          "relevance_score": 0,
          "hiring_signal_company": "<company name>",
          "matched_postings": ["<title1>", "<title2>"]
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

This skill's detection mechanism: active Risk & Regulatory role job postings (regulatory change,
financial crime / AML / KYC, operational risk, audit, controls, regulatory reporting, model
risk) at ICP-fit companies, via the Apollo job-postings endpoint.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-hiring-risk`) and these
   coverage terms: "risk hiring", "compliance hiring", "financial crime", "AML / KYC",
   "regulatory change", "model risk". (SQL via `notion-query-data-sources` is plan-gated on this
   workspace — do not rely on it.)
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
+ people search), build the template's **Sources Used** table:

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

- **Name:** `bdr-signal-hiring-risk — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-hiring-risk`
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

**Practice area:** Risk & Regulatory Change (including Financial Crime Compliance).

**Signal:** A company actively hiring for risk, compliance, regulatory change, or financial
crime roles is responding to pressure — regulatory expectations, audit findings, remediation
timelines, or a funded program build-out. Per the ICP & Buyer Persona Analysis (§5.2),
strong-fit indicators include active regulatory/audit/compliance pressure, large-scale risk
or compliance change initiatives, AML/KYC modernization, and manual controls, testing, or
reporting processes.

**Eclipse wedge:** Regulatory remediation planning, financial crime compliance support
(AML, KYC, transaction monitoring), controls design & testing, risk data & reporting
alignment, audit readiness & issue remediation, and operating model design for risk &
compliance — plus financial risk (credit, liquidity, market, valuation) and non-financial
risk (operational, technology, third-party, model risk) offerings.

**Relationship to `bdr-signal-hiring-data`:** Same detection mechanism, different practice.
Both skills share the output shape and the zoho-crud-lead handoff; each reads its own
client-managed reference page.
