---
name: bdr-signal-vendor-zoominfo
description: >
  ZoomInfo variant of the Vendor / Tool Footprint signal. Instead of scraping vendor customer
  pages (what the WebFetch sibling bdr-scan-vendor does), it harvests the install base directly
  from ZoomInfo firmographics via techAttributeTagList (tech-product tags) — answering "which
  ICP-fit companies currently use Collibra, Alation, Atlan, Purview, Informatica, Reltio, BigID,
  Immuta, OneTrust, Monte Carlo, Ataccama, Manta, Solidatus, Snowflake, Databricks, Fabric, AxiomSL,
  Workiva, etc." as a structured, ICP-filtered list. Reads the SAME vendor table reference page the
  web scan uses, maps each vendor to its ZoomInfo tech-product id via lookup, filters search_companies
  by tech tag + the narrowed firm-wide ICP, scores each company against Eclipse's ICP, dedupes against
  the shared Company+Vendor Pairs seen log, and returns a newsletter digest. Optional Apollo cross-check
  via currently_using_any_of_technology_uids. This is a company-level signal — it does NOT write to
  Zoho; output is a report for email / team chat. Built to run side-by-side with bdr-scan-vendor for a
  coverage comparison.
  Use when asked which ICP companies use a given data tool, to run the ZoomInfo vendor / tool
  footprint scan, harvest an install base, or compare ZoomInfo vs the web scrape on tool adoption.
  Triggers on: "which banks use Collibra/Snowflake/etc via zoominfo", "run the zoominfo vendor
  footprint scan", "zoominfo tool adoption signal", "harvest the install base",
  "bdr-signal-vendor-zoominfo".
  Every run renders the standardized Notion output template, computes live signal/source coverage
  from the Signal & Source Library, logs the run to the Skill Runs database, and posts a summary
  to Teams.
---

# Eclipse BDR — Signal: Vendor / Tool Footprint — ZoomInfo

Harvest the **install base** of Eclipse-relevant data tools directly from ZoomInfo firmographics —
`techAttributeTagList` — answering *"which ICP-fit companies currently use Collibra / Alation /
Informatica / Snowflake / Databricks / BigID / Monte Carlo / etc."* as a structured, ICP-filtered,
contact-ready list. Output is a **newsletter digest**.

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

This is the ZoomInfo twin of `bdr-scan-vendor` (WebFetch). The web scan finds **net-new public
evidence** (case studies, press, logos) for companies a vendor chooses to name; this scan returns
the **whole detectable footprint** from ZoomInfo's tech graph, filtered to the ICP. Run them
side-by-side: the web scan finds the *new* story, this finds the *full* list.

This is a **company-level** signal skill — it **does not interact with Zoho** (Zoho is Leads-only).
Output is a digest only.

---

## ⚠️ Reference Pages

Always fetch fresh — never use a built-in vendor list or cached values.

**Page A — Vendor Table (Notion): Skill References - Signal Vendor Adoption** — the **same** page
`bdr-scan-vendor` uses, so the two skills stay in lockstep:
`https://app.notion.com/p/b0d4e4751c3a8320b9ef011a9a975642`
(in the **Skill References** database; if the URL fails, locate it with `notion-search` for
`"Skill References - Signal Vendor Adoption"`)

Provides the **Vendor Table** (vendor/product, tool category, notes incl. product-level category
overrides like Collibra DQ / Collibra Lineage) and the **Tool Category Table** (per category: wedge
tier `core`/`adjacent`, practice areas, Eclipse wedge text). Use the vendor/product names here as
the input to the ZoomInfo `tech-vendors` → `tech-products` lookups in Step 2.

If Page A is unreachable or empty, **stop and tell the user**.

**Page B — Seen Log (Notion): Company + Vendor Pairs** — the **shared** seen log, reused so the two
vendor skills don't double-report:
- Database: `https://app.notion.com/p/37b9b05a813f802ba6dfd914a15ae570`
- Data source for writes: `collection://37b9b05a-813f-8047-81f7-000b6a9eeb45`
- Columns: `Vendor` (title), `Customer` (text), `Found on` (date), `URL`, `Notes`

Read it in Step 5, write back in Step 7. If Page B is unreachable, continue the scan but say so,
treat nothing as seen, and skip the write-back (never risk double-logging).

**ICP firmographics** — fetch fresh from the **ICP Analysis** page
(`https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0`), Section 1. Current expected band
(sanity check only — trust the page): **500–2,000 employees, $250M–$1B**, FS-core, US primary.

---

## 💳 Credits Note

`lookup` and `search_companies` count toward ZoomInfo request/record limits but are not bulk-credit
enrichments. This skill does **not** call `enrich_*` and does not enrich contacts (so no emails /
phones). Note this in the run summary. The optional Apollo cross-check (Step 4) calls
`apollo_mixed_companies_search`, which **costs 1 Apollo credit per request** — only run it on
explicit user approval (see that step).

---

## Step 1: Pick Vendors & Resolve ICP

1. Fetch Page A; take the vendor/product list from the Vendor Table (or, if the user named specific
   vendors, just those).
2. Fetch the ICP Analysis page; extract size, revenue, industries, geography (Section 1).
3. Resolve ICP to ZoomInfo IDs with `lookup`:

| ICP field | `lookup` fieldName | Used in |
|---|---|---|
| industries | `industries` (fuzzyMatch) | `industryCodes` |
| geography | `states` / `country` | `state` / `country` |
| size / revenue | — | `employeeRangeMin/Max`, `revenueMin/Max` |

---

## Step 2: Resolve Each Vendor to a Tech-Product ID

ZoomInfo's tech filter keys on **tech-product IDs**, not names. For each vendor:

1. `lookup` `tech-vendors` with `fuzzyMatch: "<vendor>"` → get the canonical vendor name.
2. `lookup` `tech-products` with `vendor: "<canonical vendor name>"` → get the product `id`(s).

Capture the `id` (e.g. **Collibra → `110232`**, verified 2026-06-24). Apply any product-level
category override from the Vendor Table notes (e.g. a Collibra DQ product maps to Data Quality /
Observability, not Data Catalog / Governance). If a vendor has no tech-product entry, note it as
**not detectable in ZoomInfo's tech graph** and skip (the web scan may still catch it).

---

## Step 3: Search the Install Base

For each tech-product id, call `search_companies`:

| Filter | Value |
|---|---|
| `techAttributeTagList` | the resolved tech-product id(s), comma-separated |
| `industryCodes` | resolved industry IDs |
| `employeeRangeMin` / `employeeRangeMax` | `500` / `2000` (numeric range — band straddles ZoomInfo buckets) |
| `revenueMin` / `revenueMax` | `250000` / `1000000` (thousands → $250M–$1B) |
| `country` / `state` | geography |
| `pageSize` | 25, paginate if needed |

Capture per company: `name`, `zoominfoCompanyId`, `website`, `employeeCount`, `revenue`, `city`,
`state`, plus which vendor/tool matched (and its tool category from Page A).

---

## Step 4 (optional): Apollo Cross-Check

If the user wants a second-source comparison, also run `apollo_mixed_companies_search` with
`currently_using_any_of_technology_uids: ["<vendor_uid>"]` (lowercase, underscores — e.g.
`collibra`, `snowflake`) plus matching size/revenue/industry filters.

⚠️ This costs **1 Apollo credit per request.** Before calling, say exactly:
**"This will consume 1 credit. Do you want to proceed?"** and wait for approval. Skip silently if
not approved; note the cross-check was not run.

---

## Step 5: Check the Seen Log

Fetch all rows from Page B (Company + Vendor Pairs). A company+vendor pair is **already seen** if it
matches an existing row — match loosely: case-insensitive, ignore legal suffixes (Inc., Corp., Ltd.,
N.A.), treat product variants as the parent vendor ("Collibra DQ" matches "Collibra").

- **Sweep mode (default, scheduled):** drop already-seen pairs from the digest; count them in
  `summary.already_seen`. This keeps the weekly newsletter from repeating itself.
- **On-demand mode (user asked about a specific vendor):** keep them, mark "(previously reported
  <Found on date>)", and don't re-log in Step 7.

---

## Step 6: Score & Build the Digest

Score each company 1–10 (firmographic-only — there's no "evidence type" here, the tech tag *is* the
evidence):

| Condition | Points |
|---|---|
| Company is a bank, credit union, or trust company | +3 |
| Company is a broker-dealer, asset/investment manager, or insurer | +2 |
| Company is a fintech, payments, or money-services business | +1 |
| Tool category wedge tier is `core` (per Tool Category Table) | +3 |
| Tool category wedge tier is `adjacent` | +1 |
| Company uses ≥ 2 Eclipse-relevant tools (multi-tool data estate) | +1 |
| Company outside FS and not a named target vertical | -2 |

**Practice areas / wedge:** take from the matched category's row in the Tool Category Table (add
**Financial Crime** only if the tool ties to BSA/AML/sanctions/KYC). **ICP match:** `true` if FS
(or named vertical) AND `relevance_score >= 5`. **Urgency:** `high` = `core`-tier tool at a
bank/insurer; `medium` = any Eclipse tool at an ICP-fit FS firm; `low` = adjacent tool or non-core
vertical.

Build a `digest` (only `relevance_score >= 5`, sorted desc). Delivery is the Microsoft Teams summary
via **bdr-post-to-teams**, with the full report on the run's Skill Runs page; the rendered format is
governed by the output template page fetched per section A of 📐 Output Standard. Starting shape
(the fallback when the template page is unreachable):

```
## Vendor Footprint Signals (ZoomInfo) — <scan_date>

### 🔹 <Company> — <tool(s)> / <tool_category> (score <relevance_score>/10, <urgency>)
**Detected stack:** <tools detected via ZoomInfo tech tags>
**Why it matters:** <funded data initiative; bought the tool, needs the operating model>
**Eclipse wedge:** <wedge from Tool Category Table, specific to this company>
**Suggested approach:** <Tom's outreach angle — "what comes after the tool">
```

Lead the wedge with the hidden hypothesis (Signal Library): *they bought/implemented the tool but
lack the people/process operating model to scale it — Eclipse sells what comes after the tool.*

---

## Step 7: Write New Signals to the Seen Log

For every company that made the digest and was **not** already seen, add one row to the Company +
Vendor Pairs database (`notion-create-pages`, parent data source
`collection://37b9b05a-813f-8047-81f7-000b6a9eeb45`):

- `Vendor` (title) — vendor name as it appears in the Vendor Table
- `Customer` — the company name
- `date:Found on:start` — today's scan date
- `userDefined:URL` — `""` or the ZoomInfo company URL if available (this signal is firmographic, not a page)
- `Notes` — `ZoomInfo tech footprint — <tool_category>` plus the detected tool(s)

Skip this step entirely if the seen log was unreachable in Step 5. Below-threshold companies are
not logged.

---

## Step 8: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log (embedded in a toggle block on the run's
page) and any downstream handoff. The user-facing deliverable is the templated report plus the
Skill Runs log entry and the Microsoft Teams summary (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-vendor-zoominfo",
  "signal_source": "ZoomInfo",
  "output_type": "digest",
  "apollo_cross_check": false,
  "criteria_fetched_from": {
    "vendor_table": "https://app.notion.com/p/b0d4e4751c3a8320b9ef011a9a975642",
    "icp_analysis": "https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0"
  },
  "signals": [
    {
      "id": "signal_001",
      "company": "<name>",
      "zoominfo_company_id": "<id>",
      "domain": "<website>",
      "tools_detected": ["<tool1>", "<tool2>"],
      "tool_category": "<category from Tool Category Table>",
      "employee_count": 0,
      "eclipse_assessment": {
        "practice_areas": ["<area 1>"],
        "wedge": "<service angle specific to this company>",
        "urgency": "high|medium|low",
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's outreach angle>"
      }
    }
  ],
  "summary": {
    "vendors_searched": 0,
    "vendors_not_in_zoominfo_tech_graph": [],
    "companies_found": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "already_seen": 0,
    "logged_to_seen_log": 0,
    "enriched": false,
    "notes": "<ICP band used; any discrepancy vs reference pages; Apollo cross-check status>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "Microsoft Teams (via bdr-post-to-teams)",
    "title": "Vendor Footprint Signals (ZoomInfo) — <scan_date>",
    "markdown": "<rendered digest per Step 6 — one block per included signal, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5`; IDs sequential `signal_001`, …
- Deduplicate companies: one entry per company, list all detected tools in `tools_detected`.
- Never writes to Zoho; never hands off to `zoho-crud-lead`.
- If nothing qualifies: `signals: []`, empty `digest.markdown`, explain in `summary.notes`.

---

## ZoomInfo vs WebFetch (`bdr-scan-vendor`) — what to compare

- **Recall vs novelty** — this scan returns the full ICP-filtered footprint (live test 2026-06-24:
  Collibra → 73 banks incl. JPMorgan, BofA, Citi, Wells Fargo); the web scan only surfaces companies
  a vendor publicly names, but catches *new* case studies/press the tech graph hasn't ingested.
- **Recency** — a tech tag reflects detected/declared usage, not necessarily a *recent* purchase;
  the web scan's dated case studies better indicate an *active* initiative. Use the shared seen log
  + (optionally) a hiring or `Project` scoop to confirm the footprint is a live program.
- **Contactability** — ZoomInfo companies carry `zoominfoCompanyId`, so contacts can be pulled in a
  follow-on pass; web-scraped names often need separate resolution.

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

This skill's detection mechanism: the current install base of Eclipse-relevant data tools at
ICP-fit companies, via ZoomInfo technographics (`search_companies` filtered by
`techAttributeTagList` tech-product tags).

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-vendor-zoominfo`) and these coverage terms:
   "vendor adoption", "tool footprint", "technographics", "data catalog", "cloud data platform".
   (SQL via `notion-query-data-sources` is plan-gated on this workspace — do not rely on it.)
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

For every source actually consulted this run (the ZoomInfo platform: company search with
tech-product tags plus `lookup` calls, and the Apollo platform when the optional cross-check
runs), build the template's **Sources Used** table:

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

- **Name:** `bdr-signal-vendor-zoominfo — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-vendor-zoominfo`
- **Multi-select:** provider(s) actually used this run (`Zoominfo`; add `Apollo` if the optional
  Apollo cross-check (Step 4) ran)
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
**Seller-doer:** Tom — uses tool footprint to find accounts with a *funded* data initiative (a paid
tool) that still need the operating model, stewardship, controls, and adoption work to realize value.

**Output routing:** Company-level signal → **newsletter digest only**, no CRM writes.

**Why this signal matters (Signal Library):** A company running a governance / catalog / privacy /
MDM / lineage / DQ / cloud-data tool has made a real investment — a symptom of a larger data-trust,
regulatory, self-service, or AI-readiness push.

**Hidden hypothesis to lead with:** they bought or are implementing a tool but lack the
people/process model to scale it. Eclipse sells "what comes after the tool."

**Relationship to siblings:** ZoomInfo-firmographics twin of the WebFetch `bdr-scan-vendor`; shares
its Vendor Table reference page and Company + Vendor Pairs seen log. Same digest output shape as the
other company-level scans.
