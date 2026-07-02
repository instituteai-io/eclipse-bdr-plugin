---
name: bdr-signal-funding-growth
description: >
  Detect ICP-fit companies that just raised capital or are scaling fast — a "growth has outrun the
  operating model" trigger that reliably precedes spend on data foundations, governance, risk/controls,
  and transformation. This is Apollo's distinct edge (no ZoomInfo required): it filters
  apollo_mixed_companies_search by recent funding (latest_funding_stage / amount / date), rapid headcount
  growth (organization_headcount_growth_range), and hiring-surge recency (organization_job_posted_at_range),
  all within the firm-wide ICP band. Fetches ICP firmographics and the growth thresholds fresh from the
  team's Notion reference page, scores each company against Eclipse's ICP, and returns a newsletter
  digest. This is a company-level signal — it does NOT write to Zoho; output is a report for email /
  team chat (contacts can be pulled in a follow-on pass via bdr-fetch-contacts-from-company).
  Use when asked which ICP companies recently raised funding or are scaling fast, to run the funding /
  growth scan, or find newly funded financial-services companies.
  Triggers on: "run the funding scan", "which ICP companies just raised", "who's scaling fast",
  "funding / rapid growth signal", "recently funded banks / fintechs", "bdr-signal-funding-growth".
  Every run renders the standardized Notion output template, computes live signal/source coverage
  from the Signal & Source Library, logs the run to the Skill Runs database, and posts a summary
  to Teams.
---

# Eclipse BDR — Signal: Funding & Rapid Growth (Apollo)

Surface ICP-fit companies that **just raised capital or are scaling headcount fast**. New capital and
rapid growth reliably precede investment in data foundations, governance, risk/controls maturity, and
transformation — exactly Eclipse's wedges — so this is a distinctive net-new pipeline source. Output is
a **newsletter digest**.

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

This is **Apollo's distinct edge**: funding + headcount-growth + hiring-recency are firmographic deltas
Apollo exposes that a curated event feed does not. No ZoomInfo required.

This is a **company-level** signal skill — it **does not interact with Zoho** (Zoho is Leads-only).
Output is a digest only. If the team wants to route a specific account to outreach, pull contacts in a
follow-on pass with `bdr-fetch-contacts-from-company`.

---

## ⚠️ Reference Page

Always fetch fresh — never use cached values.

**Reference (Notion) — Skill References - Signal: Funding & Rapid Growth:**
`https://app.notion.com/p/3914e4751c3a81c68a74cf53a209fe2b`
(in the **Skill References** database; if the URL fails, locate it with `notion-search` for
`"Signal: Funding & Rapid Growth"`)

Provides:
- **Section A — ICP Company Criteria:** industries, `company_size`, revenue, geography
- **Section B — Growth Triggers & Thresholds:** `funding_recency_months`, `min_headcount_growth_pct`,
  `job_recency_days`, `min_active_postings`, and the funding stages of interest

Also cross-check the live **ICP Analysis** page
(`https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0`, Section 1) for the authoritative
firmographic band. If the reference page is unreachable or empty, stop and tell the user.

---

## 💳 Credits Note

`apollo_mixed_companies_search` **costs 1 Apollo credit per request.** This skill runs a small number of
requests (one per trigger, or a combined request where filters allow). Report the number of requests /
credits used in the run summary. This skill does **not** enrich contacts.

---

## Step 1: Fetch Criteria

From the reference page, extract Section A (industries, `company_size`, revenue, geography) and Section B
(the four thresholds + funding stages). Defaults if a threshold is missing:
`funding_recency_months: 12`, `min_headcount_growth_pct: 20`, `job_recency_days: 90`,
`min_active_postings: 5`.

---

## Step 2: Query Apollo for Growth Signals

Run `apollo_mixed_companies_search` with the ICP firmographic filters plus **each** growth trigger. A
company qualifies if it meets **any one** trigger.

Base ICP filters (all requests):

| ICP field | Apollo parameter |
|---|---|
| industries | `q_organization_keyword_tags` |
| size | `organization_num_employees_ranges` (e.g. `"500,2000"`) |
| geography | `organization_locations` |

Trigger filters:

| Trigger | Apollo parameter(s) | Keep if |
|---|---|---|
| Recent funding | `latest_funding_stage`, and read `latest_funding_round_date` / `latest_funding_amount` on results | funding date within `funding_recency_months` |
| Rapid headcount growth | `organization_headcount_growth_range` | growth ≥ `min_headcount_growth_pct` |
| Hiring surge | `organization_job_posted_at_range` (last `job_recency_days`) + active posting count | active postings ≥ `min_active_postings` |

`per_page`: 25, paginate if needed. Capture per company: `name`, Apollo `organization id`, `domain`,
`employee_count`, `estimated_num_employees`, `latest_funding_stage`, `latest_funding_round_date`,
`latest_funding_amount`, `headcount_growth` (if returned), active-posting count, `city`, `state`, and
**which trigger(s)** fired.

---

## Step 3: Score & Assess

Score each company 1–10 (firmographic-only):

| Condition | Points |
|---|---|
| Company is a bank, credit union, or trust company | +3 |
| Company is a broker-dealer, asset/investment manager, or insurer | +2 |
| Company is a fintech, payments, or money-services business | +1 |
| Recent funding round within the window (fresh capital = fresh budget) | +2 |
| Headcount growth ≥ threshold | +2 |
| Hiring surge (≥ min active postings) | +1 |
| Meets ≥ 2 triggers (funded **and** scaling) | +1 |
| Company outside FS and not a named target vertical | -2 |

**ICP match:** `true` if FS (or named vertical) AND `relevance_score >= 5`.

**Urgency:** `high` = a recent funding round at a bank/insurer/FS firm; `medium` = headcount-growth or
hiring-surge at an ICP-fit FS firm; `low` = single soft trigger or non-core vertical.

**Practice routing:** funding/growth is cross-practice — note the most likely Eclipse entry point per
company (a scaling fintech → data foundations + governance; a PE-backed platform → transformation +
operating model; a growing bank → risk/controls maturity).

---

## Step 4: Build the Digest

Build a `digest` (only `relevance_score >= 5`, sorted desc). Delivery is the Microsoft Teams summary
via **bdr-post-to-teams**, with the full report on the run's Skill Runs page; the rendered format is
governed by the output template page fetched per section A of 📐 Output Standard. Starting shape
(the fallback when the template page is unreachable):

```
## Funding & Growth Signals (Apollo) — <scan_date>

### 🔹 <Company> — <trigger(s)> (score <relevance_score>/10, <urgency>)
**What fired:** <e.g. Series C, $120M, 3 months ago; +35% headcount YoY; 12 open reqs>
**Why it matters:** growth has outrun the operating model — data, controls, and process haven't scaled with capital/headcount
**Eclipse entry point:** <practice-specific angle for this company>
**Suggested approach:** <Tom's outreach angle tied to the growth event>
```

Lead with the hidden hypothesis: *growth has outrun the operating model — Eclipse puts the foundation
under the growth.*

---

## Step 5: Dedupe Within the Run

This signal does not currently use the shared Company + Vendor Pairs seen log (that log is keyed on
company+vendor). Dedupe **within the run** (one entry per company, listing all triggers that fired) and
note any company you recognize as previously surfaced. If weekly cadence later needs cross-run dedupe, a
dedicated funding seen log can be added.

---

## Step 6: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log (embedded in a toggle block on the run's
page) and any downstream handoff. The user-facing deliverable is the templated report plus the
Skill Runs log entry and the Microsoft Teams summary (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-funding-growth",
  "signal_source": "Apollo",
  "output_type": "digest",
  "criteria_fetched_from": {
    "reference_page": "https://app.notion.com/p/3914e4751c3a81c68a74cf53a209fe2b",
    "icp_analysis": "https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0"
  },
  "signals": [
    {
      "id": "signal_001",
      "company": "<name>",
      "apollo_organization_id": "<id>",
      "domain": "<website>",
      "triggers_fired": ["funding", "headcount_growth"],
      "latest_funding_stage": "<stage or null>",
      "latest_funding_date": "<YYYY-MM-DD or null>",
      "latest_funding_amount": "<amount or null>",
      "headcount_growth_pct": 0,
      "active_postings": 0,
      "employee_count": 0,
      "eclipse_assessment": {
        "practice_entry_point": "<practice + angle>",
        "urgency": "high|medium|low",
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's outreach angle>"
      }
    }
  ],
  "summary": {
    "apollo_requests_used": 0,
    "companies_found": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "by_trigger": {"funding": 0, "headcount_growth": 0, "hiring_surge": 0},
    "notes": "<ICP band + thresholds used; credit usage>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "Microsoft Teams (via bdr-post-to-teams)",
    "title": "Funding & Growth Signals (Apollo) — <scan_date>",
    "markdown": "<rendered digest per Step 4 — one block per included signal, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5`; IDs sequential `signal_001`, …
- Never writes to Zoho; never hands off to `zoho-crud-lead`.
- If nothing qualifies: `signals: []`, empty `digest.markdown`, explain in `summary.notes`.

---

## 📐 Output Standard (mandatory — every run)

Every run of this skill ends by rendering a standardized report, logging the run to Notion, and
posting a summary to Microsoft Teams. Nothing in this section is hardcoded: signal coverage,
source scoring, and the report format are all fetched fresh from Notion on every run.

### A. Fetch the output template (format authority)

Fetch this page fresh at the start of every run (Notion MCP `notion-fetch`):

**Output Template — Broad Scan Digest** — `https://app.notion.com/p/3914e4751c3a81ac96dad69ef0711f8c`
(in the **Skill References** database)

The rendered run report must follow that page exactly — its Run Header, Sources Used table,
Findings blocks, Run Summary, Skill Runs logging spec, and Teams summary spec. If the URL fails,
locate the page with `notion-search` (query: `"Output Template — Broad Scan Digest"`). If it is
still unavailable, render the report with the same section order (Header → Sources → Findings →
Summary) and flag in the output that the template could not be fetched.

### B. Compute signal coverage (never hardcode)

This skill's detection mechanism: ICP-fit companies that recently raised capital or are scaling
headcount fast, via Apollo firmographic growth filters (latest funding stage / amount / date,
headcount-growth range, job-posting recency).

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-funding-growth`) and these coverage terms:
   "funding", "capital raise", "headcount growth", "hiring surge", "rapid growth". (SQL via
   `notion-query-data-sources` is plan-gated on this workspace — do not rely on it.)
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

For every source actually consulted this run (the Apollo platform: company search with funding and
growth filters), build the template's **Sources Used** table:

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

- **Name:** `bdr-signal-funding-growth — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-funding-growth`
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

**Client:** TeamEclipse — data, risk, compliance, financial crime consulting for FIs.
**Seller-doer:** Tom — wants accounts where budget just appeared (fresh raise) or where scale is
outrunning the operating model (rapid headcount growth), before the foundation work becomes a crisis.

**Output routing:** Company-level signal → **newsletter digest only**, no CRM writes.

**Why this signal matters:** Not in the client Signal Library as a standalone entry, but adjacent to
SIG-018 (M&A / funding) and Keyword Library KWD-054 (Funding / Investment). It is the one high-value
trigger Apollo surfaces natively that the event-feed signals do not — a distinctive reason to add it to
the Apollo-only fleet.

**Hidden hypothesis to lead with:** growth has outrun the operating model — data, controls, and process
haven't scaled with headcount/capital. Eclipse helps put the foundation under the growth.

**Relationship to siblings:** Company-level digest scan like `bdr-scan-vendor` and
`bdr-signal-vendor-apollo`; same output shape. Unlike those, it keys on firmographic growth deltas
rather than a tech footprint, so it uses its own reference page and (for now) in-run dedupe.
