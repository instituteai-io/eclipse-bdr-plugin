---
name: bdr-signal-painpoint-zoominfo
description: >
  Detect ICP-fit companies signalling demand for a given Eclipse practice via ZoomInfo Scoops of
  type Pain Point and Project. One consolidated skill, parameterized by practice (Data Management,
  Risk & Regulatory, Transformation, or Artificial Intelligence) — the only thing that changes per
  practice is which client-curated reference file it reads. Each reference file (in the Skill
  References Notion DB) holds that practice's scoop topics, description keywords, Eclipse wedge
  mapping, personas, and thresholds; the skill NEVER hardcodes the pains. Acts as a pseudo-intent
  feed (works without a ZoomInfo Intent license — the intent-topic catalog is empty on this
  account). Fetches the practice reference file fresh, resolves ZoomInfo IDs via lookup, queries
  search_scoops, scores against the ICP, and returns a newsletter / outreach-brief digest. This is a
  company-level signal — it does NOT write to Zoho; output is a report for email / team chat.
  Use when asked to find companies showing pain points or planned projects for data, risk /
  compliance, transformation, or AI via ZoomInfo, run the pain-point scan for a practice, or find
  pseudo-intent signals.
  Triggers on: "run the pain-point scan", "data/risk/transformation/AI pain points via zoominfo",
  "who has compliance pain points", "pain point scoops for <practice>", "pseudo-intent signal",
  "bdr-signal-painpoint-zoominfo".
  Every run renders the standardized Notion output template, computes live signal/source coverage
  from the Signal & Source Library, logs the run to the Skill Runs database, and posts a summary
  to Teams.
---

# Eclipse BDR — Signal: Pain Point — ZoomInfo (per-practice)

Surface ICP-fit companies signalling demand via ZoomInfo Scoops of type **`Pain Point`** and
**`Project`**, for **one Eclipse practice per run**. The detection mechanism is identical across
practices — only the **reference file** changes. Output is a **newsletter / outreach-brief digest**.

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

This is a **company-level** signal skill — it **does not write to Zoho** (Zoho is Leads-only). After
qualifying companies, it surfaces the **enriched ICP contacts** at each by calling
`bdr-fetch-contacts-from-company`, so a single run yields both the pain signals *and* the people to
call. Output is still a digest/brief (no CRM writes), now with a contacts column.

> **Pseudo-intent:** the ZoomInfo Intent product appears **unlicensed** on this account (intent-topic
> catalog lookup returned 0 items, 2026-06-24). Scoops `Pain Point` / `Project` recover much of that
> value. **Flag to the team:** confirm the Intent entitlement; if live, a true intent variant can be
> added per practice.

---

## Step 0: Pick the Practice

The skill runs for **one practice per invocation**. Determine it from the user's request
(default: ask if ambiguous). Each practice maps to its own client-curated reference page — **the
only thing that varies per practice**:

| Practice | Reference page (Skill References DB) |
|---|---|
| Data Management | `https://app.notion.com/p/3894e4751c3a81e48325e20d284fb29b` |
| Risk & Regulatory | `https://app.notion.com/p/3894e4751c3a8100816afcc1b33d49ff` |
| Transformation | `https://app.notion.com/p/3894e4751c3a811dba53eea207515f14` |
| Artificial Intelligence | `https://app.notion.com/p/3894e4751c3a817ab015e9ccc99dfb51` |

If the URL fails (page moved/renamed), locate it with `notion-search` (query: `"Skill References -
Signal: Pain Point (<practice>)"`).

> To sweep multiple practices, run the skill once per practice and merge the digests. Do not blend
> practices in a single run — each has its own keywords, wedge, and personas.

---

## ⚠️ Criteria Reference Page

Targeting is **not hardcoded** — fetch the practice's reference page fresh on every run via the
Notion MCP `notion-fetch`. From it, extract:

- **Section A — ICP Company Criteria:** industries, company size (employee range), revenue range,
  geography.
- **Section B — Scoop Targeting:** scoop types (`Pain Point`, `Project`), scoop topics (IDs + names),
  department hint, and the **description keywords** (the primary matcher).
- **Section C — Eclipse Wedge Mapping:** pain pattern → wedge table.
- **Section D — Personas:** titles to reference in the suggested approach.
- **Section E — Thresholds & Notes:** `min_scoops`, practice-area label, exclusions, cadence.

If the reference page is unreachable or empty, **stop and tell the user**. Do not proceed with
guessed topics or keywords.

---

## 💳 Credits Note

`lookup` and `search_scoops` count toward ZoomInfo request/record limits but are not bulk-credit
enrichments. The **contact step (Step 5) calls `bdr-fetch-contacts-from-company`, which enriches by
default** — that spends ~1 ZoomInfo bulk credit per contact returned (bounded per company by that
skill's `max_contacts`, e.g. 2–3 for a digest; ZoomInfo caches enrichment for a year). Report
credits used in the run summary. For a no-credit pass, run Step 5 with `enrich:false` or skip it.

---

## Step 1: Resolve ZoomInfo IDs

Resolve the reference-page values to canonical ZoomInfo IDs with `lookup` (search fails on free
text). Use the topic IDs in the reference file as a starting aid, but re-resolve to guard against
drift:

| Reference field | `lookup` fieldName | Used in |
|---|---|---|
| industries | `industries` (fuzzyMatch) | `industryCodes` |
| geography | `states` / `country` | `state` / `country` |
| scoop topics | `scoop-topics` (fuzzyMatch on each topic name) | `scoopTopics` |

Use returned `id` values, not `attributes.name`.

---

## Step 2: Sweep Pain-Point & Project Scoops

Call `search_scoops`:

| Filter | Value |
|---|---|
| `scoopTypes` | `["Pain Point", "Project"]` |
| `scoopTopics` | resolved topic IDs from the reference file (omit if the practice has no good topic match — keywords carry it) |
| `industryCodes` | resolved industry IDs |
| `employeeRangeMin` / `employeeRangeMax` | from Section A (e.g. `500` / `2000` — use the numeric range; the ICP band straddles ZoomInfo's coarse buckets) |
| `country` / `state` | geography |
| `department` | Section B department hint — **soft hint only**; drop it if it suppresses obvious matches |
| `publishedStartDate` | freshness window — default last 14 days for scheduled cadence; widen to 60–90 days if volume is thin (Pain Point / Project scoops are lower-frequency than M&A) |
| `sort` | `-originalPublishedDate`, `pageSize`: 25 |

> ⚠️ **Do NOT pass `revenueMin/Max` (or the `description` param) to `search_scoops`.** Learned in
> live testing (2026-06-24): a `revenueMin/Max` filter silently excludes every company whose
> revenue is null in ZoomInfo — stacking it on top of the employee band zeroed out otherwise-valid
> results. And the `description` param ANDs its words, over-filtering to ~0. Filter by revenue (if
> needed) and by keywords **client-side, on the returned scoops** — see below. Drop `revenueMin/Max`
> from the call entirely; rely on `industryCodes` + `employeeRange` + `scoopTopics` for targeting.

**Do NOT use the `description` API param for keyword matching.** Instead, after the call, **qualify**
a scoop client-side if it is a `Pain Point` or `Project` whose returned `description` (or `topics`)
matches any **description keyword** from Section B (case-insensitive substring). Keywords are the
primary matcher — `scoopTopics` only narrow the pull. Drop generic finance/ops noise (RFPs for
expense tracking, ERP/CRM, board reporting) that lacks the practice's angle. Group by company; keep
companies with ≥ `min_scoops` qualifying scoops.

> Higher-precision alternative for a known target-account list: `enrich_scoops` per company with the
> same filters (costs a credit per company).

Capture per qualifying scoop: company, `zoominfoCompanyId`, `description`, `originalPublishedDate`,
`types`, `topics`, `link`.

---

## Step 3: Map the Eclipse Wedge

For each qualifying company, map the matched pain to the wedge using **Section C** of the reference
file (pain pattern → Eclipse wedge). Lead with the hidden hypothesis: a stated pain or planned
project is a funded (or about-to-be-funded) window — Eclipse helps execute it.

---

## Step 4: Score Eclipse ICP Relevance

Score each company 1–10:

| Condition | Points |
|---|---|
| Company is a bank, credit union, or trust company | +3 |
| Company is a broker-dealer, asset/investment manager, or insurer | +2 |
| Company is a fintech, payments, or money-services business (or a named target vertical for the practice) | +1 |
| Has a `Pain Point` scoop (active, felt pressure) | +3 |
| Has a `Project` scoop (planned/funded initiative) | +2 |
| Matches a high-value sub-signal called out in the reference file (e.g. financial crime for Risk) | +1 |
| Published within the last 30 days | +1 |
| Company outside the practice's target industries | -2 |

**Urgency:** `high` = Pain Point at a core-ICP firm; `medium` = Project at an ICP-fit firm; `low` =
soft/older signal or off-target vertical. **ICP match:** `true` if in target industries AND
`relevance_score >= 5`.

---

## Step 5: Fetch Contacts for Each Qualified Company

For every qualified company (Step 2), call **`bdr-fetch-contacts-from-company`** with:
- `company`: the company's `zoominfoCompanyId`
- `practice`: this run's practice
- `provider`: `zoominfo` (match this run)
- `enrich`: `true` (default — return contactable people); `max_contacts`: 2–3 per company for a digest

Collect the returned enriched contacts per company. **Impose no output shape on that skill** — it
returns contacts per company; *this* skill decides how to present them (Step 6). If a company returns
no ICP-title contacts, note it — don't drop the company.

---

## Step 6: Build the Digest / Outreach Brief

No Zoho writes, no `crm_payload`. Assemble the `digest` as a **single table** — include only signals
with `relevance_score >= 5`, sorted by score desc. Delivery is the Microsoft Teams summary via
**bdr-post-to-teams**, with the full report on the run's Skill Runs page; the rendered format is
governed by the output template page fetched per section A of 📐 Output Standard (the table below is
the fallback shape). Columns:

`| Company | Type | Signal | Eclipse wedge | ICP contacts (enriched) |`

- **Company** — one row per company (not per scoop).
- **Type** — `Pain Point`, `Project`, or `N× Pain Point/Project` when grouped.
- **Signal** — the scoop(s). **When a company has multiple same-theme scoops, group them into one row
  and break them into bullets** (`<br>• …`) so the cell stays readable. Do **not** collapse distinct
  scoops into one vague phrase, and do **not** list the same scoop more than once.
- **Eclipse wedge** — from Section C, specific to the company.
- **ICP contacts (enriched)** — the Step 5 contacts as `**Name — Title** · email · phone`,
  `<br>`-separated (2–3 max). If none, `_No ICP-title contact found_`. Flag obviously off-profile
  enrichment (e.g. a foreign number / no business email) rather than presenting it as clean.

Add a one-line "Strongest plays" takeaway under the table. Reference Section D personas in any
outreach angle.

---

## Step 7: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log (embedded in a toggle block on the run's
page) and any downstream handoff. The user-facing deliverable is the templated report plus the
Skill Runs log entry and the Microsoft Teams summary (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-painpoint-zoominfo",
  "practice": "<Data Management | Risk & Regulatory | Transformation | Artificial Intelligence>",
  "signal_source": "ZoomInfo",
  "output_type": "digest",
  "intent_license_note": "ZoomInfo Intent appeared unlicensed (empty intent-topic catalog 2026-06-24); uses Scoops Pain Point/Project as pseudo-intent",
  "criteria_fetched_from": { "reference_page": "<practice reference page URL>" },
  "window": { "published_start_date": "<YYYY-MM-DD>", "published_end_date": "<YYYY-MM-DD or null>" },
  "signals": [
    {
      "id": "signal_001",
      "company": "<name>",
      "zoominfo_company_id": "<id>",
      "scoop_type": "Pain Point | Project",
      "topics": ["<topic>"],
      "date": "<YYYY-MM-DD>",
      "summary": "<2 sentences: what the scoop says>",
      "source_url": "<scoop link or null>",
      "matched_scoops": ["<distinct scoop 1>", "<distinct scoop 2>"],
      "eclipse_assessment": {
        "practice_areas": ["<practice label from Section E>"],
        "wedge": "<mapped wedge specific to this signal>",
        "urgency": "high|medium|low",
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's outreach angle to the practice persona>"
      },
      "contacts": [
        { "name": "<first last>", "title": "<title>", "email": "<email or null>", "phone": "<phone or null>", "linkedin_url": "<url or null>", "flag": "<off-profile note, or null>" }
      ]
    }
  ],
  "summary": {
    "scoops_scanned": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "by_type": { "Pain Point": 0, "Project": 0 },
    "contacts_found": 0,
    "enriched": true,
    "credits_used": 0,
    "notes": "<ICP band used; contacts via bdr-fetch-contacts-from-company; any discrepancy vs the reference page>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "Microsoft Teams (via bdr-post-to-teams)",
    "title": "Pain-Point Signals — <Practice> (ZoomInfo) — <scan_date>",
    "markdown": "<rendered digest per Step 6 — the single table, one row per company, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5`; IDs sequential `signal_001`, …
- Deduplicate companies: one entry per company, note multiple scoops in `summary`.
- For Risk & Regulatory, add `"Financial Crimes & Compliance"` to `practice_areas` when matched on
  AML/BSA/KYC/sanctions/financial-crime/transaction-monitoring keywords (per that reference file).
- Never writes to Zoho; never hands off to `zoho-crud-lead`.
- If nothing qualifies: `signals: []`, empty `digest.markdown`, explain in `summary.notes`.

---

## Notes & comparisons

- **One mechanism, four reference files.** Adding/retuning a practice = edit (or add) a reference
  page, no code change. To add a 5th practice, add a row to the Step 0 table + a reference page.
- **vs `bdr-scan-regulatory`:** that scans a pasted regulator enforcement URL (the public action);
  this reads ZoomInfo's company-side pain points / planned projects. Complementary.
- **vs `bdr-signal-corp-events-zoominfo`:** the AI practice overlaps the corp-events AI-announcement
  path — corp-events catches the *announcement event*, this catches the *pain / planned project*.
  Dedupe at the digest stage if both run.
- **vs a true Intent signal:** Scoops `Project` ("planning X over the next N months") is the closest
  proxy without the license. Live proof (2026-06-24): Rabobank "planning AI-driven compliance and
  risk management over the next three months" (Project); FCMB regulatory-verification delay (Pain
  Point) — 171 compliance/audit scoops at banks ≥1,000 in one call.

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

This skill's detection mechanism: ICP-fit companies signalling practice-specific demand (pain
points and planned projects), via ZoomInfo Scoops of type `Pain Point` and `Project` swept per
Eclipse practice as a pseudo-intent feed.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-painpoint-zoominfo`) and these coverage terms:
   "pain point", "planned project", "pseudo-intent", "compliance pressure", "data initiative". (SQL
   via `notion-query-data-sources` is plan-gated on this workspace — do not rely on it.)
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

For every source actually consulted this run (the ZoomInfo platform: Scoops search plus `lookup`
calls; contact pulls run through the bdr-fetch-contacts-from-company skill), build the template's
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

- **Name:** `bdr-signal-painpoint-zoominfo — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-painpoint-zoominfo`
- **Multi-select:** provider(s) actually used this run (`Zoominfo`)
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
**Seller-doer:** Tom.

**Signal:** A stated pain point or a planned project is a company telling you it's under pressure or
about to fund work — the moment to offer help. Detected per practice via the client-curated
reference files.

**Practices covered:** Data Management, Risk & Regulatory Change (incl. Financial Crime),
Transformation, Artificial Intelligence — one per run.

**Output routing:** Company-level signal → **newsletter / outreach-brief digest**, no CRM writes
(mirrors `bdr-scan-regulatory`). Optional future extension: surface the practice persona contact and
route to Leads via `zoho-crud-lead` — out of scope per the company-level routing rule.

**Relationship to siblings:** Same digest output shape as `bdr-scan-regulatory` /
`bdr-signal-corp-events-zoominfo`. Replaces the earlier Risk-only `bdr-signal-risk-painpoint-zoominfo`
(consolidated 2026-06-24: one mechanism, parameterized per practice by reference file — the pattern
the team is standardizing toward; hiring will follow in a later pass).
