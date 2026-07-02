---
name: bdr-signal-corp-events-zoominfo
description: >
  ZoomInfo variant of the Corporate Events signal. Detects ICP-fit companies experiencing a
  corporate event that creates data / governance / risk demand — M&A and post-merger integration,
  system-integrator & strategic partnership announcements, AI / GenAI announcements, and funding —
  by sweeping ZoomInfo Scoops across the firm-wide ICP instead of fetching press/vendor URLs one at
  a time (that is what the WebFetch sibling bdr-scan-corp-events does). Fetches the firm-wide ICP
  firmographics fresh from the team's 01 ICP & Buyer Persona Analysis Notion page, resolves them to ZoomInfo IDs, then
  queries search_scoops for the M&A / Partnership / Product Launch / Project / Funding scoop types,
  scores each event against Eclipse's ICP, and returns a ready-to-send newsletter digest. This is a
  company-level signal — it does NOT write to Zoho; output is a report for email / team chat. Built
  to run side-by-side with bdr-scan-corp-events (WebFetch) for a coverage comparison.
  Use when asked to find recent M&A, acquisitions, partnerships, AI announcements, or funding at
  ICP companies via ZoomInfo, run the ZoomInfo corporate-events scan, or sweep ZoomInfo Scoops for
  corporate-event signals.
  Triggers on: "run the zoominfo corp events scan", "any M&A signals via zoominfo", "who's
  partnering / got acquired / announced AI", "zoominfo corporate event signal",
  "bdr-signal-corp-events-zoominfo".
  Every run renders the standardized Notion output template, computes live signal/source coverage
  from the Signal & Source Library, logs the run to the Skill Runs database, and posts a summary
  to Teams.
---

# Eclipse BDR — Signal: Corporate Events — ZoomInfo

Detect ICP-fit companies hit by a corporate event that creates data, governance, or risk demand —
**M&A / post-merger integration, system-integrator & strategic partnerships, AI / GenAI
announcements, and funding** — by sweeping **ZoomInfo Scoops**. Output is a **newsletter digest**
for email / team chat.

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

This is the ZoomInfo twin of `bdr-scan-corp-events` (WebFetch). The web version reads one pasted
URL at a time; this version queries ZoomInfo's curated event feed and returns every matching event
across the ICP in one pass — so the two can be diffed for coverage.

This is a **company-level** signal skill. Per the Eclipse BDR output-routing rule, company-level
signals **do not interact with Zoho** (Zoho is Leads-only, written exclusively by person-level
skills via `zoho-crud-lead`). This skill produces a human-readable digest only.

---

## ⚠️ Criteria Reference Page

Targeting firmographics are **not hardcoded** — fetch them fresh on every run. The firm-wide ICP
lives on the **01 ICP & Buyer Persona Analysis** Notion page:
`https://app.notion.com/p/3904e4751c3a81dca87af64d0ea85822`

Fetch via the Notion MCP `notion-fetch` tool. From **§4 (Firm-Wide ICP — §4.1 baseline profile, §4.2 primary industries)** extract: company
**size** (employee range), **revenue** range, **primary industries**, and **geography**.

> Current expected values (sanity check only — always use whatever the page says): **500–2,000
> employees, $250M–$1B revenue**, Financial Services core (Banks, Asset/Investment Management,
> Insurance), US primary. If the page disagrees, trust the page and note the difference in the
> summary.

If the page is unreachable or empty, **stop and tell the user**. Do not proceed with guessed
firmographics.

---

## 💳 Credits Note

`lookup` and `search_scoops` count toward ZoomInfo request/record limits but are not bulk-credit
enrichments. This skill does **not** call `enrich_*` (no per-company bulk credits) and does not
enrich contacts. Note this in the run summary so nobody assumes enrichment ran.

---

## Step 1: Fetch & Resolve ICP

1. Fetch the 01 ICP & Buyer Persona Analysis page; extract size, revenue, industries, geography (§4).
2. Resolve to canonical ZoomInfo IDs with `lookup` (search fails on free-text values):

| ICP field | `lookup` fieldName | Used in |
|---|---|---|
| industries | `industries` (use `fuzzyMatch`, e.g. "banking", "investment", "insurance") | `industryCodes` |
| geography | `states` / `metro-regions` (use `fuzzyMatch`) or `country` directly | `state` / `metroRegions` / `country` |
| size / revenue | — (see Step 2; use the numeric range params) | `employeeRangeMin/Max`, `revenueMin/Max` |

Use returned `id` values, not `attributes.name`.

---

## Step 2: Sweep Scoops

Call `search_scoops` once per event family (or batch the scoop types in one call and split on the
returned `types`). Scoop types to include:

| Event family | `scoopTypes` value(s) | Notes |
|---|---|---|
| M&A / post-merger integration | `Mergers & Acquisitions (M&A)` | also `Divestiture` if useful |
| SI / strategic partnership | `Partnership` | |
| AI / GenAI announcement | `Product Launch`, `Project` | filter to AI via topic / description (see below) |
| Funding | `Funding` | a "scaling, immature operating model" signal |

Apply the ICP firmographic filters on the same call:
- `industryCodes` — resolved industry IDs
- `employeeRangeMin`: `500`, `employeeRangeMax`: `2000` *(use the numeric range, not the coarse
  `employeeCount` buckets — the ICP band straddles ZoomInfo's `1000to4999` bucket)*
- `revenueMin`: `250000`, `revenueMax`: `1000000` *(revenue is in **thousands** → $250M–$1B)*
- `country` / `state` — geography
- `publishedStartDate` — freshness window. Default to the **last 7 days** for the scheduled weekly
  cadence (prevents repeats without a seen-log); widen to 30–90 days for an on-demand catch-up run.
- `sort`: `-originalPublishedDate`, `pageSize`: 25

**AI filter:** `Product Launch` / `Project` scoops are noisy — keep one only if its `topics`
include Artificial Intelligence (or the description mentions AI / GenAI / machine learning /
generative). Resolve the AI scoop-topic id via `lookup` (`scoop-topics`, fuzzyMatch "artificial")
and pass it as `scoopTopics`, or filter in post-processing on the returned `topics`.

> Higher-precision alternative for a known target-account list: call `enrich_scoops` per company
> with the same `scoopTypes`. Costs a credit per company — only for named accounts.

Group results by company; capture `zoominfoCompanyId`, company name, scoop `description`,
`originalPublishedDate`, `types`, `topics`, and `link`.

---

## Step 3: Classify & Map the Wedge

For each event, set `event_type` (M&A / Partnership / AI Announcement / Funding) and map the
Eclipse wedge:

| event_type | Eclipse wedge to lead with |
|---|---|
| M&A / post-merger | Integration data assessment, taxonomy & lineage reconciliation, single-source-of-truth, controls alignment across the merged entities. |
| SI / strategic partnership | Independent advisory / QA over the SI-led rollout; protect data quality, governance, and value realization the SI won't own. |
| AI / GenAI announcement | AI-readiness: the data foundation, governance, and lineage the AI story needs but usually lacks. |
| Funding | A scaling org typically outruns its data/governance operating model — strategy, stewardship, and controls to scale safely. |

Hidden hypothesis (Signal Library): the event is a *symptom* of a larger data-trust / regulatory /
AI-readiness push — lead with the gap, not the tool.

---

## Step 4: Score Eclipse ICP Relevance

Score each event 1–10:

| Condition | Points |
|---|---|
| Company is a bank, credit union, or trust company | +3 |
| Company is a broker-dealer, asset/investment manager, or insurer | +2 |
| Company is a fintech, payments, or money-services business | +1 |
| Event is M&A / post-merger integration (highest data-fragmentation demand) | +3 |
| Event is an AI/GenAI announcement | +2 |
| Event is an SI/strategic partnership or funding round | +1 |
| Event published within the last 14 days (fresh trigger) | +1 |
| Company outside FS and not a named target vertical | -2 |

**Urgency:** `high` = M&A at a bank/insurer in-band; `medium` = AI/partnership at an ICP-fit FS
firm; `low` = funding, or any event at a non-core vertical.

**ICP match:** `true` if the company is FS (or a named target vertical) AND `relevance_score >= 5`.

---

## Step 5: Build the Newsletter Digest

This skill writes **no** Zoho records and produces **no** `crm_payload`. Assemble a `digest`
object for email / team chat. Include only events with `relevance_score >= 5`, sorted by score
descending. Keep `digest.markdown` clean and channel-neutral. Delivery is the Microsoft Teams
summary via **bdr-post-to-teams**, with the full report on the run's Skill Runs page; the rendered
format is governed by the output template page fetched per section A of 📐 Output Standard. A
reasonable fallback shape when the template page is unreachable:

```
## Corporate Event Signals (ZoomInfo) — <scan_date>

### 🔹 <Company> — <event_type> (score <relevance_score>/10, <urgency>)
**Event:** <what happened — counterparty, amount, etc.> (<event date>)
**Why it matters:** <data/governance/risk demand this creates>
**Eclipse wedge:** <wedge>
**Suggested approach:** <Tom's outreach angle>
[Source](<link>)
```

---

## Step 6: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log (embedded in a toggle block on the run's
page) and any downstream handoff. The user-facing deliverable is the templated report plus the
Skill Runs log entry and the Microsoft Teams summary (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-corp-events-zoominfo",
  "signal_source": "ZoomInfo",
  "output_type": "digest",
  "criteria_fetched_from": { "reference_page": "https://app.notion.com/p/3904e4751c3a81dca87af64d0ea85822" },
  "window": { "published_start_date": "<YYYY-MM-DD>", "published_end_date": "<YYYY-MM-DD or null>" },
  "signals": [
    {
      "id": "signal_001",
      "company": "<name>",
      "zoominfo_company_id": "<id>",
      "event_type": "M&A | Partnership | AI Announcement | Funding",
      "counterparty": "<other party or null>",
      "date": "<YYYY-MM-DD>",
      "summary": "<2 sentences: what happened>",
      "source_url": "<scoop link or null>",
      "eclipse_assessment": {
        "practice_areas": ["Data Management"],
        "wedge": "<service angle specific to this event>",
        "urgency": "high|medium|low",
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's outreach angle>"
      }
    }
  ],
  "summary": {
    "scoops_scanned": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "by_event_type": { "M&A": 0, "Partnership": 0, "AI Announcement": 0, "Funding": 0 },
    "enriched": false,
    "notes": "<e.g. ICP firmographics used; any discrepancy vs the reference page>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "Microsoft Teams (via bdr-post-to-teams)",
    "title": "Corporate Event Signals (ZoomInfo) — <scan_date>",
    "markdown": "<rendered digest per Step 5 — one block per included signal, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5`; IDs sequential `signal_001`, `signal_002`, …
- Deduplicate companies: if a company has multiple events, keep one signal entry and note the
  others in its `summary`.
- This skill never writes to Zoho and never hands off to `zoho-crud-lead`.
- If no qualifying events: return `signals: []`, empty `digest.markdown`, and explain in
  `summary.notes`.

---

## ZoomInfo vs WebFetch (`bdr-scan-corp-events`) — what to compare

- **Coverage & recall** — ZoomInfo Scoops returns the full set of recent events across the ICP in
  one call (live test 2026-06-24: 49 M&A + Partnership scoops at banks ≥1,000). The WebFetch
  version only sees a pasted URL. Compare how many in-ICP events each surfaces.
- **Freshness** — Scoops are research-curated and dated; some may lag a same-day press release the
  WebFetch version catches when pointed straight at it.
- **Precision** — the AI-announcement family is the noisiest; compare how cleanly the topic filter
  isolates genuine AI initiatives from generic product news.

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
Findings blocks, and Run Summary. The template governs the report format only; the Skill Runs
logging and Teams delivery below are this skill's own piping, defined here. If the URL fails,
locate the page with `notion-search` (query: `"Output Template — Broad Scan Digest"`). If it is
still unavailable, render the report with the same section order (Header → Sources → Findings →
Summary) and flag in the output that the template could not be fetched.

### B. Compute signal coverage (never hardcode)

This skill's detection mechanism: corporate events (mergers & acquisitions, partnerships, AI /
GenAI announcements, funding) at ICP-fit companies, via a ZoomInfo Scoops sweep (`search_scoops`)
across the firm-wide ICP.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-signal-corp-events-zoominfo`) and these coverage terms:
   "merger", "acquisition", "partnership", "AI announcement", "funding", "corporate event". (SQL
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
calls), build the template's **Sources Used** table:

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

- **Name:** `bdr-signal-corp-events-zoominfo — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-signal-corp-events-zoominfo`
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

<1-sentence outcome: n findings, m high-priority.>

• <Top finding 1 — company/topic + one clause>

• <Top finding 2>

• <Top finding 3>

Full report: <link to the Skill Runs page created in section D>
```

Cap at the top 3 findings. Post even when the run found nothing — a clean-scan note with the
coverage line is still a result.

---

## Eclipse Context

**Client:** TeamEclipse — data, risk, compliance, and financial crime consulting for financial
institutions. **Seller-doer:** Tom.

**Signal (Signal Library):** A corporate event — M&A, SI/strategic partnership, AI announcement, or
funding — creates a window of data, governance, and risk demand. M&A fragments data estates;
partnerships hand execution to an SI who won't own data quality; AI announcements outrun the data
foundation; funding outruns the operating model.

**Eclipse wedge:** "What comes after the event" — integration data assessments and taxonomy/lineage
reconciliation (M&A), independent advisory/QA over SI rollouts, AI-readiness and data-foundation
work, and operating-model/stewardship/controls uplift for fast-scaling firms.

**Output routing:** Company-level signal → **newsletter digest only**, no CRM writes.

**Relationship to siblings:** Same output shape as `bdr-scan-vendor` / `bdr-scan-regulatory`;
ZoomInfo-Scoops twin of the WebFetch `bdr-scan-corp-events`. Fetches firmographics fresh from the
01 ICP & Buyer Persona Analysis page rather than a dedicated keyword reference page.
