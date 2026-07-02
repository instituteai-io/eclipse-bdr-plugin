---
name: bdr-signal-new-exec-apollo
description: >
  Detect a newly appointed executive sponsor at an ICP company (net-new prospect, not an existing CRM
  lead) — a new CDO/CDAO/CRO/CCO/COO/Chief Transformation Officer/Chief AI Officer/CIO or function head.
  A new leader has a mandate and budget and typically reshapes their function in the first 90–180 days,
  making this a top-of-catalog buy trigger (Signal Library SIG-003). Apollo-only: it searches
  apollo_mixed_people_api_search for the sponsor titles at ICP companies, then infers recency from each
  candidate's current-role START DATE (read via enrichment), keeping only those inside the recency
  window. Fetches ICP titles, firmographics, and the recency window fresh from the team's Notion
  reference page, routes each hit to the right Eclipse practice by function, and returns a JSON payload
  ready for the zoho-crud-lead skill.
  CONFIDENCE CAVEAT: Apollo has no native "appointed in the last N days" event, so recency is inferred
  from role start date (noisier, and costs an enrichment credit per candidate). Every hit should be
  confirmed against a press release / leadership page before outreach.
  Use when asked to find newly appointed executives, new CDOs/CROs/etc at ICP companies, run the new
  executive sponsor scan, or find leadership-change signals via Apollo.
  Triggers on: "run the new exec scan", "who just became CDO/CRO at an ICP company", "new executive
  sponsor signal", "leadership change signal via apollo", "bdr-signal-new-exec-apollo".
  Every run renders the standardized Notion output template, computes live signal/source coverage
  from the Signal & Source Library, logs the run to the Skill Runs database, and posts a summary
  to Teams.
---

# Eclipse BDR — Signal: New Executive Sponsor (Apollo)

Surface a **newly appointed executive sponsor** at an ICP company — a new CDO / CDAO / CRO / CCO / COO /
Chief Transformation Officer / Chief AI Officer / CIO or function head. A new leader runs a current-state
assessment, sets a 90-day agenda, and brings in outside help to move fast — the prime outreach window
before incumbent advisors entrench. The JSON payload is the data handoff to `zoho-crud-lead` (which
writes to Inbound (Leads)); the user-facing deliverable is the templated report
(see 📐 Output Standard).

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

This is a **person-level** signal (a specific new executive is the lead), so qualified hits are written
to Zoho Leads. It is the net-new counterpart to `bdr-signal-job-change`, which only tracks executives
**already** in the CRM.

> ⚠️ **Confidence caveat — read before running.** Apollo has **no native "appointed in the last N days"
> event**. This skill *infers* recency from each candidate's current-role **start date** (read via
> enrichment) and keeps only those inside the window. That inference is noisier than a true appointment
> event and costs an enrichment credit per candidate checked. **Pair every hit with a quick web/press
> confirmation** (company newsroom / leadership page) before outreach, and run it narrowly (specific
> industries + size) to keep credit spend bounded. This is why the signal is rated medium-confidence on
> the diagnosis.

---

## ⚠️ Criteria Reference Page

One reference page drives this skill. Always fetch it fresh — never use cached values.

**Reference (Notion) — Skill References - Signal: New Executive Sponsor:**
`https://app.notion.com/p/3914e4751c3a81c380e9c32aef69cb53`
(in the **Skill References** database; if the URL fails, locate it with `notion-search` for
`"Signal: New Executive Sponsor"`)

Fetch fresh via the Notion MCP `notion-fetch` tool on every run.

Provides:
- **Section A — ICP Company & Contact Criteria:** the executive sponsor `titles` to detect, `seniority`,
  `industries`, `company_size`, `geography`, `include_similar_titles`
- **Section B — Recency Window & Practice Routing:** `new_role_window_days`, `require_web_confirmation`,
  and the function → `Practice_Areas_In_Scope` routing table

If the reference page is unreachable or empty, stop and tell the user.

---

## Step 1: Fetch Criteria

From Section A extract `titles`, `seniority`, `industries`, `company_size`, `geography`,
`include_similar_titles`. From Section B extract `new_role_window_days` (default `120`),
`require_web_confirmation` (default `true`), and the practice-routing table.

---

## Step 2: Find Candidate Executives at ICP Companies

Call `apollo_mixed_people_api_search`:

| Criteria field | Apollo parameter |
|---|---|
| `titles` (Section A) | `person_titles` |
| `seniority` (Section A) | `person_seniorities` |
| `industries` | `q_organization_keyword_tags` |
| `company_size` | `organization_num_employees_ranges` |
| `geography` | `organization_locations` |
| `include_similar_titles` | `include_similar_titles` |

Fixed parameters:
- `per_page`: 25

These are **candidate** executives — people currently holding a sponsor title at an ICP company. Recency
is not yet known (the people search does not reliably expose role-start date). Last names are masked and
email/phone omitted at this stage — expected.

---

## Step 3: Infer Recency (Enrichment) 💳

For each candidate, call `apollo_people_match` (or `apollo_people_bulk_match`) to read the current-role
**start date** (Apollo returns employment history / current-position start where available), and reveal
contact data.

⚠️ **Credit cost:** ≈ 1 enrichment credit per candidate checked. Because recency can't be pre-filtered,
this is the main credit cost of the skill — **run narrowly** (tight industry + size) and report the
count in the run summary. If the team wants a bounded dry run, cap the number of candidates enriched and
say so.

Compute `role_age_days` = days between the current-role start date and the scan date. **Keep only
candidates with `role_age_days <= new_role_window_days`.** If Apollo returns no start date for a
candidate, set `role_age_days: null` and **exclude** them from the qualified set (do not guess a new
appointment) — list them under `summary.no_start_date` so a human can check manually if desired.

---

## Step 4: Score & Route to Practice

For each executive inside the recency window, score Eclipse relevance 1–10:

| Condition | Points |
|---|---|
| Company is a bank or credit union | +3 |
| Company is an asset/investment manager, insurer, or PE firm | +2 |
| Company is a fintech, payments, or money-services business | +1 |
| Title is a C-suite sponsor (CDO, CDAO, CRO, CCO, COO, Chief Transformation Officer, Chief AI Officer, CIO) | +3 |
| Title is a function head (Head of Data Governance, Head of Financial Crime, Head of Transformation, etc.) | +2 |
| `role_age_days` ≤ 90 (very fresh appointment) | +1 |
| Company has 1,000–10,000 employees | +1 |
| No company size data available | -1 |

**ICP match:** `true` if score ≥ 5.

**Practice routing:** map each executive's function to `Practice_Areas_In_Scope` using the Section B
table (CDO/CDAO/CAO/Head of Data Governance → Data Management & Analytics; CRO/CCO/Financial Crime/
Regulatory Change → Risk & Regulatory Change; COO/Chief Transformation Officer/Head of Transformation →
Transformation; Chief AI Officer → Artificial Intelligence; CIO → route by context).

---

## Step 5: Enrich Contact Data

Enrichment already ran in Step 3 (it doubled as the recency read). From each enriched record populate
full `last_name`, `email`, `phone`, `linkedin_url`, and numeric `employee_count` / `industry` where
Apollo returns them. If a reveal failed, leave that field `null` and note it — do not re-enrich.

---

## Step 6: Build CRM Payload

For every executive with `icp_match: true` **and** `role_age_days <= new_role_window_days`, build one
lead record for the Inbound (Leads) module.

| Generic | Zoho API Name | Notes |
|---|---|---|
| `first_name` | `First_Name` | |
| `last_name` | `Last_Name` | Unmasked from Step 3/5; flag if reveal failed |
| `title` | `Designation` | The new (current) executive title |
| `company` | `Company` | |
| `email` | `Email` | null if none on file |
| `phone` | `Phone` | null if none on file |
| `linkedin_url` | `LinkedIn_Page_URL` | |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | Routed by function per Section B |
| `lead_source` | `Lead_Source` | Always `"Apollo ICP Scan"` ⚡ (value pending team addition) |
| `apollo_contact_id` | `Apollo_Contact_ID_c` | ⚡ Field pending team creation |
| `signal_type` | `Signal_Type_c` | Always `"New Executive Sponsor"` ⚡ Field pending team creation |
| `signal_mechanism` | `Signal_Mechanism_c` | Always `"Apollo role-start inference"` ⚡ Field pending team creation |
| `signal_date` | `Signal_Date_c` | Date WE detected the signal = scan date ⚡ Field pending team creation |
| `signal_source_date` | `Signal_Source_Date_c` | The exec's current-role start date; how old the appointment is ⚡ Field pending team creation |
| `signal_summary` | `Signal_Summary_c` | Generated in Step 7 ⚡ Field pending team creation |
| `needs_confirmation` | `Signal_Needs_Confirmation_c` | `true` when `require_web_confirmation` — human confirms via press/leadership page before outreach ⚡ Field pending team creation |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will skip ⚡ fields
gracefully until they exist in Zoho.

**Dedup field:** `Apollo_Contact_ID_c`. Dedup fallback: `First_Name + Last_Name + Company`.

---

## Step 7: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max), including how fresh the appointment is
and the confirmation flag:

> "[First name] [last name] appears to have become [title] at [Company] ~[role_age_days] days ago —
> surfaced as a New Executive Sponsor signal ([practice]) on [scan date]. Confirm against a press
> release / leadership page before outreach (Apollo role-start inference)."

---

## Step 8: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log and the `zoho-crud-lead` handoff. The
user-facing deliverable is the templated report (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-new-exec-apollo",
  "signal_source": "Apollo",
  "signal_mechanism": "Apollo role-start inference",
  "criteria_fetched_from": {
    "reference_page": "https://app.notion.com/p/3914e4751c3a81c380e9c32aef69cb53"
  },
  "summary": {
    "candidates_found": 0,
    "candidates_enriched": 0,
    "within_recency_window": 0,
    "icp_matched": 0,
    "enrichment_credits_estimated": 0,
    "no_start_date": [],
    "notes": "<recency window used; all hits flagged for web confirmation; credit usage>"
  },
  "crm_payload": {
    "generated_by": "bdr-signal-new-exec-apollo",
    "generated_at": "<today YYYY-MM-DD>",
    "module": "Leads",
    "action": "upsert",
    "dedup_field": "Apollo_Contact_ID_c",
    "dedup_fallback": "First_Name + Last_Name + Company",
    "leads": [
      {
        "id": "lead_001",
        "fields": {
          "First_Name": "<first>",
          "Last_Name": "<last — note MASKED if reveal failed>",
          "Designation": "<new title>",
          "Company": "<company>",
          "Email": "<email or null>",
          "Phone": "<phone or null>",
          "LinkedIn_Page_URL": "<url or null>",
          "No_of_Employees": 0,
          "Industry": "<mapped value>",
          "Practice_Areas_In_Scope": ["<routed by function>"],
          "Lead_Source": "Apollo ICP Scan",
          "Apollo_Contact_ID_c": "<apollo person id>",
          "Signal_Type_c": "New Executive Sponsor",
          "Signal_Mechanism_c": "Apollo role-start inference",
          "Signal_Date_c": "<today YYYY-MM-DD>",
          "Signal_Source_Date_c": "<role start date or null>",
          "Signal_Summary_c": "<generated summary>",
          "Signal_Needs_Confirmation_c": true
        },
        "meta": {
          "role_age_days": 0,
          "relevance_score": 0,
          "enriched": true,
          "practice_routed": "<practice>",
          "needs_web_confirmation": true
        }
      }
    ]
  }
}
```

- IDs sequential: `lead_001`, `lead_002`, etc.
- `meta` block is for human review only — not written to Zoho
- If no qualifying executives found: return empty `leads: []`
- Never write a lead whose `role_age_days` is `null` or outside the window — those are not confirmed appointments

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

This skill's detection mechanism: newly appointed executive sponsors at ICP companies, inferred
from each candidate's current-role start date via the Apollo people search plus people-match
enrichment.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-new-exec-apollo`) and these
   coverage terms: `"new executive sponsor"`, `"leadership change"`, `"newly appointed"`,
   `"executive appointment"`, `"new leader"`. (SQL via `notion-query-data-sources` is plan-gated
   on this workspace — do not rely on it.)
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

For every source actually consulted this run (the Apollo platform: people search + people-match
enrichment), build the template's **Sources Used** table:

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

- **Name:** `bdr-signal-new-exec-apollo — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-new-exec-apollo`
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

**Practice area:** Multi-practice — routed per executive by function (Data / Risk & Regulatory /
Transformation / AI).

**Signal:** A newly appointed executive sponsor is the highest-intent relationship trigger in the
catalog (Signal Library SIG-003). New leaders reassess, set a 90-day agenda, and buy outside help early.
Catching the appointment inside the first ~120 days is the prime window.

**Who we surface:** the new executive themselves — the buyer — written to Zoho Leads as a net-new
prospect (contrast with `bdr-signal-job-change`, which updates executives already in the CRM).

**Eclipse wedge:** 90-day strategy support, current-state assessment, roadmap, operating-model design,
and executive advisory tuned to the new leader's function.

**Confidence & mechanism:** Medium-confidence by design — Apollo has no true appointment event, so
recency is inferred from role-start date and every hit is flagged for web/press confirmation. Rated a
gap-closer for SIG-003 on the diagnosis, downgraded from the original ZoomInfo-Executive-Move path
because this account is Apollo-only. If a stronger appointment source is added later (press-release sweep
or a licensed event feed), this skill should defer recency detection to it.
