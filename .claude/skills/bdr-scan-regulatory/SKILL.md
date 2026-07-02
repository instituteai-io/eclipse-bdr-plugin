---
name: bdr-scan-regulatory
description: >
  Fetch any regulatory enforcement page URL and extract structured signal objects for
  TeamEclipse BDR outreach. Accepts a URL from OCC, FDIC, CFPB, SEC, FINRA, Federal Reserve,
  NCUA, or any other regulatory body. Auto-detects the source, extracts enforcement actions,
  scores each one against Eclipse's ICP (data, risk, compliance, financial crime consulting),
  and returns structured JSON signals plus a ready-to-send newsletter digest. This is a
  company-level signal — it does NOT write to Zoho; output is a report for email / team chat.
  Two modes: by default (no URL given) it sweeps the full regulator source list curated on the
  team's Notion reference page ("Signal: Regulatory Enforcement Sources") — this is the weekly
  scheduled cadence; passing one or more URLs scans just those.
  Use when the user pastes a regulatory enforcement URL and asks to scan it, run the enforcement
  scan, sweep all regulators, check for signals, or review enforcement actions. Triggers on:
  "scan this page", "run the regulatory scan", "sweep the regulators", "what signals are here",
  "check this enforcement action", "regulatory enforcement scan", or any URL from a known
  regulator domain (sec.gov, consumerfinance.gov, occ.gov, fdic.gov, finra.org,
  federalreserve.gov, ncua.gov, fincen.gov, ofac.treasury.gov, dfs.ny.gov).
  Every run renders the standardized Notion output template, computes live signal/source coverage
  from the Signal & Source Library, logs the run to the Skill Runs database, and posts a summary
  to Teams.
---

# Eclipse BDR — Regulatory Scan by Source

Fetch a regulatory enforcement page URL, extract enforcement actions, score each one against
Eclipse's ICP, and return structured JSON signals ready for BDR outreach.

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

---

## ⚠️ Source Reference Page

The list of regulators and the URLs to sweep are **not hardcoded** — they are curated by the team on
the **Signal: Regulatory Enforcement Sources** Notion page (in the **Skill References** database):
`https://app.notion.com/p/38e4e4751c3a81cba046cc0735359dea`

Fetch it fresh via the Notion MCP `notion-fetch` tool at the start of every **sweep**. If that URL
fails (page moved/renamed), locate it with `notion-search` (query:
`"Signal: Regulatory Enforcement Sources"`) and read the page it returns instead.

The page provides the **Source Table** (one row per regulator: Sweep URL, **Fetch method**,
domain(s), action types to look for, fetch caveats) plus the Fetch Methods key, the freshness window,
and any extra one-off URLs. The **Fetch method** column tells you how to fetch each source — see
Step 3 (some regulators bot-block or time out a plain WebFetch and need `curl + UA`). This skill
carries only the piping (fetch → extract → score → digest); the team owns *which* sources to watch by
editing the page — no skill change needed.

If the page is unreachable or empty in sweep mode, **stop and tell the user** — do not fall back to a
guessed source list. (Manual mode does not need the page; see Step 1.)

---

## Step 1: Pick the Mode

**Sweep mode (default)** — the skill was invoked with no URL. This is the normal weekly cadence.
Fetch the Source Reference Page, then scan every **Sweep URL** in its Source Table (skip rows whose
URL is `TBD` or marked unfetchable). Apply the page's **freshness window** — by default keep only
actions dated within the **last 7 days** so the weekly digest doesn't repeat itself (widen to 30–90
days for an on-demand catch-up). Merge all sources into one ranked output. Report any URL that fails
to fetch in `summary.exclusion_reason` rather than stopping the sweep. Do not ask the user for a URL.

**Manual mode** — the user passed one or more URLs. Scan only those; process each and merge into one
output. Skip the freshness window (the user asked about these specific pages and wants everything on
them). Only ask for a URL if the user clearly meant a specific page ("scan this page") but none came
through.

---

## Step 2: Detect the Source

Match each URL's domain to a known regulator. Prefer the **Source Table** from the reference page
(its rows carry the regulator, action types, and fetch caveats); the table below is a fallback if the
page is unavailable in manual mode:

| Domain pattern | Regulator | Action types to look for |
|---|---|---|
| `sec.gov` | SEC | Litigation releases, administrative proceedings, press releases |
| `consumerfinance.gov` | CFPB | Consent orders, supervisory actions |
| `occ.gov` or `apps.occ.gov` | OCC | Formal agreements, consent orders, C&D orders, civil money penalties |
| `fdic.gov` | FDIC | Consent orders, civil money penalties, terminations of insurance |
| `finra.org` | FINRA | Disciplinary actions, firm-level fines and suspensions |
| `federalreserve.gov` | Federal Reserve | Enforcement actions against bank holding companies |
| `fincen.gov` | FinCEN | BSA/AML enforcement, civil money penalties |
| `ofac.treasury.gov` | OFAC | Sanctions enforcement / settlement agreements |
| `dfs.ny.gov` | NYDFS | Consent orders, civil money penalties (NY-regulated FIs) |
| `ncua.gov` | NCUA | Enforcement actions against credit unions |
| Other | Unknown | Attempt generic extraction |

---

## Step 3: Fetch and Extract

Fetch each Sweep URL using the **Fetch method** its row specifies on the reference page (many
regulators bot-block or time out a plain WebFetch — the method column is how the page records the
workaround):

- **`WebFetch`** (default) — retrieve with the WebFetch tool (reads server-rendered HTML and RSS/XML).
  In manual mode, always use WebFetch first.
- **`curl + UA`** — for sources flagged this way (currently SEC, OFAC, FinCEN: SEC returns HTTP 403
  without a declared User-Agent; OFAC/FinCEN time out WebFetch on slow origins). Fetch with a Bash
  `curl` that sets a descriptive User-Agent and follows redirects, then parse the returned HTML:
  `curl -sL -A "EclipseBDR/1.0 (contact: <team email>)" -m 45 "<Sweep URL>"`
  (use `-L` especially for FinCEN, whose URL 301-redirects). If a `WebFetch` source fails in a way
  that looks like a bot-block (403) or timeout, retry it once with this curl method before recording
  it as failed.

**RSS sources** (FRB, OCC, FDIC) return a feed of headlines: identify the relevant item (e.g. OCC's
monthly "OCC Announces Enforcement Actions for <Month>", FDIC's "FDIC Publishes <Month> Enforcement
Actions"), then fetch that item's linked news-release page for the per-institution detail.

Extract enforcement actions using source-specific logic.

**SEC (`sec.gov`)** — Look for litigation releases, administrative proceedings, press releases. Extract per item: respondent/firm name, case number, date filed, violation type, penalty amount, charge summary. Prefer firm-level actions over named individuals.

**CFPB (`consumerfinance.gov`)** — Look for enforcement action entries. Extract: institution name, action type, date, description, penalty amount, remediation requirements. Common institution types: banks, fintechs, servicers, debt collectors, payment companies.

**OCC (`occ.gov`, `apps.occ.gov`)** — Look for enforcement actions table or search results. Extract: institution name, charter type, action type, effective date, summary. OCC actions are bank-specific — high Eclipse relevance. Key types: Formal Agreement, Consent Order, C&D, CMP.

**FDIC (`fdic.gov`)** — Look for enforcement decisions list. Extract: institution name, city/state, action type, effective date, description. Key types: Consent Order, Order to Pay, Removal/Prohibition Order, Termination of Insurance.

**FINRA (`finra.org`)** — Look for disciplinary actions or monthly reports. Focus on **firm-level** actions. Extract: firm name, violation type, sanction amount, brief description. If URL links to a PDF monthly report, flag the PDF URL and extract any firm names visible in HTML.

**Federal Reserve (`federalreserve.gov`)** — Look for enforcement actions press releases or order PDFs. Extract: institution/holding company name, action type, date, findings summary.

**Unknown source** — Attempt to identify any table/list with institution name, date, action type. Flag source as Unknown. If page is not an enforcement listing, return `signals: []` with a note.

---

## Step 4: Score Eclipse ICP Relevance

Score every extracted enforcement action from 1–10.

| Condition | Points |
|---|---|
| Institution is a bank, credit union, or trust company | +3 |
| Institution is a broker-dealer, investment advisor, or asset manager | +2 |
| Institution is a fintech, payments company, or money services business | +1 |
| Action involves BSA/AML, sanctions, KYC/CDD, or financial crime program deficiencies | +3 |
| Action involves regulatory reporting failures, model risk, or stress testing | +3 |
| Action involves data governance, data quality, IT controls, or reporting architecture | +3 |
| Action involves board/governance deficiencies, audit findings, or management controls | +2 |
| Action creates a mandatory remediation plan or board-level oversight requirement | +1 |
| Action is only against named individuals — firm itself not charged | -2 |
| Action is a small fine under $100K with no remediation requirement | -1 |

**Practice area mapping:**

| Practice Area | Keywords |
|---|---|
| Financial Crime | BSA, AML, anti-money laundering, KYC, CDD, EDD, sanctions, OFAC, fraud, financial crimes program |
| Risk & Compliance | regulatory reporting, MRA, MRIA, model risk, stress testing, risk framework, compliance program deficiencies |
| Data Management | data governance, data quality, reporting controls, data architecture, IT controls, reporting systems failures |
| Governance | board oversight, audit committee, internal controls, management deficiencies, audit findings, corporate governance |

An action can map to multiple practice areas — include all that apply.

**Urgency:**
- `high`: Consent order with board-level reporting requirements, OR penalty over $1M, OR systemic program-level findings
- `medium`: Formal agreement, MOU, penalty $100K–$1M, or targeted remediation
- `low`: Notice of charges, fine under $100K, informal action, individual-only

**ICP match:** `true` if the institution is a financial services firm AND at least one Eclipse practice area applies.

---

## Step 5: Generate Eclipse Assessment

For each signal with `relevance_score >= 5`, generate:

**`wedge`** — Eclipse service angle (1–2 sentences, specific to this action):
- Financial Crime: "BSA/AML remediation roadmap, KYC/CDD program redesign, and compliance program uplift."
- Risk & Compliance: "Regulatory remediation roadmap and risk reporting uplift in response to [specific finding]."
- Data Management: "Data governance framework, reporting controls remediation, and data quality program."
- Governance: "Board-level remediation oversight, audit response support, and management controls redesign."
- Multi-practice: Combine — Eclipse's cross-functional model (data + risk + financial crime) is a key differentiator.

**`recommended_approach`** — Tom's opening outreach angle (1–2 sentences, specific to this action). Reference the specific action type and regulator, lead with Eclipse's relevant experience, frame urgency.

---

## Step 6: Build the Newsletter Digest

This skill does **not** write to Zoho and produces **no** `crm_payload` (no Accounts, no Deals —
Zoho is the Leads-only Inbound module, written exclusively by person-level skills). Instead,
assemble a `digest` object for an email or team-chat post. Include only signals with
`relevance_score >= 5` AND `icp_match: true`, sorted by `relevance_score` descending.

Render `digest.markdown` like this (one block per signal):

```
## Regulatory Enforcement Signals — <scan_date>
Source: <regulator> — <source_url>

### 🔹 <Company> — <action_type> (score <relevance_score>/10, <urgency>)
**What happened:** <signal.summary>
**Practice areas:** <comma-separated practice_areas>
**Eclipse wedge:** <wedge>
**Suggested approach:** <recommended_approach>
[Source](<source_url>)
```

> **Delivery is resolved:** the run summary goes to Microsoft Teams via **bdr-post-to-teams**, and
> the full report lives on the run's Skill Runs page (see 📐 Output Standard). The rendered digest
> format is governed by the output template page fetched per section A of the Output Standard; the
> block above is the fallback when the template page is unreachable.

---

## Step 7: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log (embedded in a toggle block on the run's
page) and any downstream handoff. The user-facing deliverable is the templated report plus the
Skill Runs log entry and the Microsoft Teams summary (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-scan-regulatory",
  "mode": "sweep | manual",
  "output_type": "digest",
  "source_url": "<the URL scanned in manual mode, or 'sweep' for a full-list run>",
  "sources_swept": ["<Sweep URL 1>", "<Sweep URL 2>"],
  "regulator": "<single regulator abbreviation, or 'multiple' for a sweep>",
  "signals": [
    {
      "id": "signal_001",
      "company": "<institution or firm name>",
      "regulator": "<regulator abbreviation>",
      "action_type": "<e.g. Consent Order, Civil Money Penalty, Formal Agreement, Cease and Desist>",
      "date": "<YYYY-MM-DD or best estimate>",
      "summary": "<2–3 sentences: what happened, what was found, what is now required>",
      "source_url": "<direct URL to this specific action if available, otherwise the page URL>",
      "eclipse_assessment": {
        "practice_areas": ["<practice area 1>", "<practice area 2>"],
        "wedge": "<Eclipse service angle specific to this action>",
        "urgency": "high|medium|low",
        "budget_signal": true,
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's outreach angle specific to this action>"
      }
    }
  ],
  "summary": {
    "total_extracted": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "signals_excluded": 0,
    "exclusion_reason": "<if signals were excluded, explain why>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "Microsoft Teams (via bdr-post-to-teams)",
    "title": "Regulatory Enforcement Signals — <scan_date>",
    "markdown": "<rendered digest per Step 6 — one block per included signal, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5` AND `icp_match: true`
- IDs sequential: `signal_001`, `signal_002`, etc.
- In **sweep mode**: set `source_url` to `"sweep"`, list every fetched URL in `sources_swept`, set
  `regulator` to `"multiple"` (each signal still carries its own `regulator`), apply the freshness
  window, and roll any failed-to-fetch URLs into `exclusion_reason` instead of stopping
- Deduplicate companies across sources: if the same institution appears in multiple actions, keep one
  signal entry and note the others in its `summary`
- This skill never writes to Zoho and never hands off to `zoho-crud-lead`
- If page has login wall, is empty, or has no enforcement data: return `signals: []`, an empty `digest.markdown`, and explain in `exclusion_reason`

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

This skill's detection mechanism: regulatory enforcement actions (consent orders, fines, formal
agreements) against named financial institutions, via WebFetch / curl sweeps of the regulator
enforcement pages curated on the team's Notion source list.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-scan-regulatory`) and these coverage terms:
   "regulatory enforcement", "consent order", "civil money penalty", "remediation", "enforcement
   action". (SQL via `notion-query-data-sources` is plan-gated on this workspace — do not rely on
   it.)
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

For every source actually consulted this run (each regulator enforcement page swept this run),
build the template's **Sources Used** table:

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

- **Name:** `bdr-scan-regulatory — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-scan-regulatory`
- **Multi-select:** provider(s) actually used this run (`Web`)
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

**Client:** TeamEclipse — professional services firm specializing in data, risk, compliance, and financial crime consulting for financial institutions.

**Seller-doer:** Tom — uses enforcement signals to identify accounts with active remediation obligations where Eclipse can be positioned as the delivery partner.

**ICP:** Mid-to-large banks, broker-dealers, investment managers, fintechs in regulated markets — especially those with active consent orders, formal agreements, or civil money penalties requiring structured remediation plans.

**Practice areas → wedge:**
- **Financial Crime** → BSA/AML program build or remediation, KYC/CDD redesign, transaction monitoring tuning
- **Risk & Compliance** → Regulatory remediation roadmap, risk reporting uplift, MRA/MRIA response
- **Data Management** → Data governance framework, reporting controls, data quality program
- **Governance** → Board-level remediation oversight, audit support, management control redesign

**Best signal:** A consent order at a mid-to-large bank with specific remediation requirements in one or more Eclipse practice areas — creates urgency, allocates budget, mandates external expertise.
