---
name: bdr-scan-reg-change
description: >
  Fetch any financial-services regulator rulemaking / newsroom URL and turn a new rule, proposed
  rule, supervisory guidance, or call for comment into a prioritized outreach brief for TeamEclipse
  BDR. Covers Buy-Trigger #2 (Regulatory Changes or Updates) — DISTINCT from the enforcement scan
  (bdr-scan-regulatory / Buy-Trigger #3), which only handles consent orders, fines, and actions
  against a named institution. This skill is TOPIC-LEVEL: a rule change targets no single company,
  so it summarizes what changed, the compliance/comment deadline, which ICP segments and Eclipse
  practice areas it impacts, and the Eclipse angle — so Tom can reach out proactively BEFORE the
  deadline forces a scramble. Accepts a URL from FRB, OCC, FDIC, SEC, FINRA, OFAC, FinCEN, NYDFS,
  BCBS, CFPB, NCUA, or the Federal Register. Auto-detects source and document type, and returns
  structured JSON signals plus a ready-to-send newsletter digest. Company-level/topic-level signal —
  it does NOT write to Zoho; output is a report for email / team chat.
  Two modes: by default (no URL given) it sweeps the full regulator source list curated on the team's
  Notion reference page ("Signal: Regulatory Change Sources") — this is the weekly scheduled cadence;
  passing one or more URLs scans just those.
  Use when the user pastes a rulemaking / proposed-rule / guidance / request-for-comment URL and asks
  to scan it, run the regulatory-change scan, sweep the regulators for rule changes, or summarize a
  rule change.
  Triggers on: "scan this rule change", "run the reg change scan", "sweep for new rules",
  "new rule / proposed rule", "call for comment", "regulatory change scan", "what does this rule mean
  for our ICP", "bdr-scan-reg-change", or a rulemaking / Federal Register URL from a known regulator domain
  (federalreserve.gov, occ.gov, fdic.gov, sec.gov, finra.org, ofac.treasury.gov, fincen.gov,
  dfs.ny.gov, bis.org/bcbs, consumerfinance.gov, ncua.gov, federalregister.gov).
  Every run renders the standardized Notion output template, computes live signal/source coverage
  from the Signal & Source Library, logs the run to the Skill Runs database, and posts a summary
  to Teams.
---

# Eclipse BDR — Regulatory Change / Rulemaking Scan

Fetch a financial-services regulator's rulemaking or newsroom page, extract the rule change, and
return a prioritized **outreach brief** mapped to Eclipse's ICP segments and practice areas. Output
is a **newsletter digest** for email / team chat.

> **Signal coverage:** computed fresh from the Signal Library on every run — never hardcoded.
> Every run ends with the standardized report, a Skill Runs log entry in Notion, and a Teams
> summary. See **📐 Output Standard** below.

This closes **Buy-Trigger #2 (Regulatory Changes or Updates)** — previously zero coverage. It is the
**proactive twin** of `bdr-scan-regulatory`:

| | `bdr-scan-regulatory` (Buy-Trigger #3) | `bdr-scan-reg-change` (Buy-Trigger #2) — this skill |
|---|---|---|
| Detects | Enforcement actions, consent orders, fines | New / proposed rules, guidance, calls for comment |
| Targets | A **named institution** (company-level) | **No single company** — a **topic** affecting a segment |
| Timing | Reactive (something already went wrong) | Proactive (a deadline is coming) |
| Outreach | The fined institution's remediation owners | ICP segments the rule will burden |

Keep the two distinct — they share source domains but never the same document type. If the page is an
enforcement action against a named firm, **hand off to `bdr-scan-regulatory`** instead.

This is a **topic-level** signal skill. Per the Eclipse BDR output-routing rule, topic- and
company-level signals **do not interact with Zoho** (Zoho is the Leads-only Inbound module, written
exclusively by person-level skills via `zoho-crud-lead`). This skill produces a human-readable digest
only — **never a `crm_payload`**.

---

## ⚠️ ICP Segment Reference Page

The ICP segments a rule maps to are **not hardcoded** — fetch them fresh on every run from the
**ICP Analysis** Notion page:
`https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0`

Fetch via the Notion MCP `notion-fetch` tool. Use **Section 1 (Firm-Wide ICP)** for the addressable
market (industries, size, geography) and **Sections 5.x (per-practice ICPs)** for the four Eclipse
practices and their buyer personas:

| Eclipse practice | Personas to target when a rule lands here |
|---|---|
| Data Management | CDO, Head of Data Governance, Head of Data Quality, Data Office leadership |
| Risk & Regulatory Change | CRO, CCO, Head of Regulatory Change, Head of Reg Reporting |
| Transformation | COO, Head of Transformation, Program / Change leadership |
| Artificial Intelligence | Head of AI / ML, Model Risk, AI Governance leadership |

> Current sanity-check values (always trust the page over this note): Financial Services core
> (Banks, Asset/Investment Management, Insurance), US primary, mid-to-large institutions. If the page
> disagrees, trust the page and note the difference in the summary.

If the page is unreachable or empty, **still produce the brief** but flag in `summary.notes` that ICP
segments were mapped from the skill's fallback persona table rather than the live page. Do not invent
firmographics.

---

## ⚠️ Source Reference Page

The list of regulators and the URLs to sweep are **not hardcoded** — they are curated by the team on
the **Signal: Regulatory Change Sources** Notion page (in the **Skill References** database):
`https://app.notion.com/p/38e4e4751c3a813a8d68f4895ed1b7cf`

Fetch it fresh via the Notion MCP `notion-fetch` tool at the start of every **sweep**. If that URL
fails (page moved/renamed), locate it with `notion-search` (query:
`"Signal: Regulatory Change Sources"`) and read the page it returns instead.

The page provides the **Source Table** (Sweep URL, **Fetch method**, document types, caveats) plus the
Fetch Methods key, the freshness window, and any extra one-off URLs. The **primary source is the
Federal Register API** — one machine-readable, dated call covers all federal rulemaking agencies (OCC,
FRB, FDIC, FinCEN, SEC, NCUA, CFPB); the other rows are supplements (agency guidance, FINRA, BCBS,
NYDFS, OFAC) for what the Federal Register doesn't carry. The **Fetch method** column tells you how to
fetch each — see Step 3 (OFAC needs `curl + UA`). This skill carries only the piping (fetch → extract →
map → score → digest); the team owns *which* sources to watch by editing the page — no skill change needed.

> This is a **separate** reference page from the **ICP Segment Reference Page** above: that one maps a
> rule to *who* it impacts; this one lists *where* to look for rules. Both are fetched fresh.

If this page is unreachable or empty in sweep mode, **stop and tell the user** — do not fall back to a
guessed source list. (Manual mode does not need it; see Step 1.)

---

## Step 1: Pick the Mode

**Sweep mode (default)** — the skill was invoked with no URL. This is the normal weekly cadence.
Fetch the Source Reference Page, then query its **primary source, the Federal Register API**, setting
`conditions[publication_date][gte]` to the start of the freshness window — by default the **last 7
days** (widen to 30–90 days for an on-demand catch-up); the date filter is the dedup. Then fetch the
**supplement** rows (OCC bulletins RSS, FDIC FILs, FINRA, BCBS, NYDFS, OFAC) using each row's Fetch
method, skipping rows whose URL is `TBD`. Merge all sources into one ranked output, de-duplicating a
rule that appears in both the FR API and a supplement. Report any URL that fails to fetch in
`summary.exclusion_reason` rather than stopping the sweep. Do not ask the user for a URL.

**Manual mode** — the user passed one or more URLs. Scan only those; process each and merge into one
output, skipping the freshness window. If the user gives only a topic ("scan the new FinCEN AML rule")
with no URL, use **WebSearch** to locate the primary source on the regulator's domain or the Federal
Register, confirm the URL, then proceed.

---

## Step 2: Detect the Source & Document Type

Match the URL domain to a known regulator (sources from Buy-Trigger #2):

| Domain pattern | Regulator |
|---|---|
| `federalreserve.gov` | Federal Reserve Board (FRB) |
| `occ.treas.gov`, `occ.gov` | Office of the Comptroller of the Currency (OCC) |
| `fdic.gov` | Federal Deposit Insurance Corporation (FDIC) |
| `sec.gov` | Securities and Exchange Commission (SEC) |
| `finra.org` | Financial Industry Regulatory Authority (FINRA) |
| `ofac.treasury.gov`, `treasury.gov/...ofac` | Office of Foreign Assets Control (OFAC) |
| `fincen.gov` | Financial Crimes Enforcement Network (FinCEN) |
| `dfs.ny.gov` | New York Department of Financial Services (NYDFS) |
| `bis.org/bcbs` | Basel Committee on Banking Supervision (BCBS) |
| `consumerfinance.gov` | Consumer Financial Protection Bureau (CFPB) |
| `ncua.gov` | National Credit Union Administration (NCUA) |
| `federalregister.gov` | Federal Register (cross-agency rulemaking of record) |
| Other | Unknown — attempt generic extraction |

Then classify the **document type** — this drives the urgency model:

| Document type | What it means | Typical deadline |
|---|---|---|
| **Final Rule** | Adopted; compliance is now mandatory | Compliance / effective date |
| **Proposed Rule (NPRM)** | Drafted; open for comment before adoption | Comment-due date, then a later effective date |
| **Request for Comment / ANPR** | Early-stage; agency is gathering input | Comment-due date |
| **Supervisory Guidance / Bulletin** | Expectations clarified (not always binding) | Often none / "examiners will assess" |
| **Interim Final Rule** | Effective immediately, comment still open | Effective date (already live) |

---

## Step 3: Fetch & Extract

Fetch each source using the **Fetch method** its row specifies on the reference page:

- **`WebFetch (JSON)`** — the Federal Register API (primary). WebFetch reads the JSON directly; walk
  the `results` array, taking `title`, `type` (Rule / Proposed Rule), `publication_date`, `agencies`,
  `comments_close_on`, `abstract`, and `html_url` per document. For the ones that score well, fetch the
  `html_url` for the full "SUMMARY"/"DATES" detail.
- **`WebFetch`** — default for the RSS/HTML supplement rows (OCC bulletins RSS, FDIC FILs, FINRA, BCBS,
  NYDFS). RSS returns headlines → fetch the linked page for detail.
- **`curl + UA`** — for sources flagged this way (currently OFAC: it times out WebFetch). Fetch with a
  Bash `curl` that sets a descriptive User-Agent and follows redirects:
  `curl -sL -A "EclipseBDR/1.0 (contact: <team email>)" -m 45 "<Sweep URL>"`. If a `WebFetch` source
  fails like a bot-block (403) or timeout, retry once with this curl method before recording it failed.

PDFs and Federal Register long-form documents may need careful extraction — pull the "SUMMARY" and
"DATES" sections first. Extract per rule:

- **Topic** — what the rule is about (e.g. "AML/CFT program modernization", "climate risk
  management", "incident-reporting", "model risk", "data/reporting requirements").
- **Document type** (Step 2).
- **Effective date** and/or **comment-due date** (Federal Register "DATES" section is authoritative).
- **What changes** — the substantive shift, in plain English.
- **Obligations created** — new programs, reports, controls, attestations, or capabilities firms must
  stand up.
- **Who is in scope** — institution types / asset thresholds the rule names.

If the page is not a rulemaking document (e.g. it's an enforcement action), say so and recommend
`bdr-scan-regulatory`; return `signals: []` with the reason in `exclusion_reason`.

---

## Step 4: Map to Eclipse Practice Areas & ICP Segments

A rule change is a topic — map it to **practice areas** (the same four the enforcement scan uses, so
the digests read consistently) and to the **ICP segments** most affected.

| Practice Area | Map a rule here when it touches… |
|---|---|
| Financial Crime | BSA/AML, CFT, sanctions/OFAC, KYC/CDD/EDD, beneficial ownership, fraud, transaction monitoring |
| Risk & Compliance | Regulatory reporting, capital/liquidity, stress testing, model risk, operational/op-resilience, incident reporting, climate risk |
| Data Management | Data governance, data quality, reporting architecture, lineage, recordkeeping, data-retention/privacy requirements |
| Governance | Board oversight, accountability/SMCR-style regimes, internal controls, audit, third-party/vendor risk |

A rule can map to multiple practice areas — include all that apply.

**ICP segment targeting** (derived, not company-named):
- Use the in-scope institution types + asset thresholds from Step 3, intersected with the firm-wide
  ICP from the reference page, to name the **segments** to prioritize (e.g. "mid-to-large national
  banks $10B–$250B", "registered investment advisers", "broker-dealers").
- Name the **personas** to reach in each affected practice (from the reference table above).

> **Do not fabricate a target company.** This signal is topic-level. Outreach targets are segments +
> personas, surfaced later by a person-level skill — not a single account invented here.

---

## Step 5: Score Urgency & ICP Relevance

**Urgency** is driven by deadline proximity × breadth of impact:

| Condition | Effect |
|---|---|
| Final / Interim Final Rule with a compliance date within 12 months | → `high` |
| Proposed Rule or RFC with a comment-due date within 60 days | → `high` (Eclipse can help shape the response *now*) |
| Final Rule with a compliance date 12–24 months out | → `medium` |
| Proposed Rule with a distant/unset effective date | → `medium` |
| Supervisory guidance with no hard deadline | → `low` (nurture / thought-leadership) |
| Broad applicability across the ICP (most segments affected) | bump up one level |
| Narrow applicability (a single niche segment) | bump down one level |

**Relevance score (1–10)** — how much net-new Eclipse demand the rule creates:

| Condition | Points |
|---|---|
| Rule creates a new mandatory program, report, or control firms must build | +3 |
| Rule lands squarely in one of Eclipse's four practice areas | +3 |
| Rule applies broadly across the firm-wide ICP | +2 |
| Hard deadline within 12 months (compliance or comment) | +2 |
| Rule maps to multiple practice areas (cross-functional — Eclipse's differentiator) | +1 |
| Guidance only, no binding obligation | -1 |
| Out of scope for the ICP (non-FS, or institution types Eclipse doesn't serve) | -3 |

**ICP match:** `true` if the rule affects Eclipse's addressable market AND at least one practice area
applies AND `relevance_score >= 5`.

---

## Step 6: Generate the Outreach Brief

For each signal with `relevance_score >= 5`, generate:

**`wedge`** — Eclipse's service angle, specific to the rule (1–2 sentences). Examples:
- Financial Crime rule: "Gap assessment against the new AML/CFT requirements, program redesign, and
  remediation roadmap ahead of the compliance date."
- Risk & Compliance rule: "Regulatory-change impact assessment, new-report build (data sourcing →
  controls → attestation), and readiness testing before the effective date."
- Data Management rule: "Data lineage, recordkeeping, and reporting-control build to evidence
  compliance with the new requirement."
- Governance rule: "Accountability-regime / board-oversight uplift and control framework alignment."
- Multi-practice: combine — Eclipse's cross-functional model (data + risk + financial crime) is the
  differentiator.

**`recommended_approach`** — Tom's proactive outreach angle (1–2 sentences). Lead with the deadline,
name the affected segment + persona, position Eclipse's relevant experience, and frame the value of
acting *before* the deadline (shape the comment response, or get a head start on the build).

---

## Step 7: Build the Newsletter Digest

This skill writes **no** Zoho records and produces **no** `crm_payload`. Assemble a `digest` object
for email / team chat. Include only signals with `relevance_score >= 5` AND `icp_match: true`, sorted
by `relevance_score` descending. Keep `digest.markdown` clean and channel-neutral.

Render one block per signal:

```
## Regulatory Change Signals — <scan_date>
Source: <regulator> — <source_url>

### 🔹 <Topic> — <document_type> (score <relevance_score>/10, <urgency>)
**What's changing:** <summary>
**Deadline:** <comment-due and/or compliance date> <(effective <date>)>
**Who it impacts:** <ICP segments> — <personas>
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

## Step 8: Return JSON

Assemble this JSON payload — it feeds the Skill Runs log (embedded in a toggle block on the run's
page) and any downstream handoff. The user-facing deliverable is the templated report plus the
Skill Runs log entry and the Microsoft Teams summary (see 📐 Output Standard).

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-scan-reg-change",
  "mode": "sweep | manual",
  "output_type": "digest",
  "source_url": "<the URL scanned in manual mode, or 'sweep' for a full-list run>",
  "sources_swept": ["<Sweep URL 1>", "<Sweep URL 2>"],
  "regulator": "<single regulator abbreviation, or 'multiple' for a sweep>",
  "criteria_fetched_from": {
    "icp_segments_page": "https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0",
    "sources_page": "https://app.notion.com/p/38e4e4751c3a813a8d68f4895ed1b7cf"
  },
  "signals": [
    {
      "id": "signal_001",
      "topic": "<short topic title, e.g. 'AML/CFT program modernization rule'>",
      "regulator": "<regulator abbreviation>",
      "document_type": "Final Rule | Proposed Rule | Request for Comment | Supervisory Guidance | Interim Final Rule",
      "comment_due_date": "<YYYY-MM-DD or null>",
      "effective_date": "<YYYY-MM-DD or null>",
      "summary": "<2–3 sentences: what is changing and what firms must now do>",
      "obligations_created": ["<new program / report / control 1>", "<...>"],
      "in_scope": "<institution types / asset thresholds the rule names>",
      "source_url": "<direct URL to this rule if available, otherwise the page URL>",
      "eclipse_assessment": {
        "practice_areas": ["<practice area 1>", "<practice area 2>"],
        "icp_segments": ["<segment 1>", "<segment 2>"],
        "personas": ["<persona 1>", "<persona 2>"],
        "wedge": "<Eclipse service angle specific to this rule>",
        "urgency": "high|medium|low",
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's proactive outreach angle specific to this rule>"
      }
    }
  ],
  "summary": {
    "total_extracted": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "signals_excluded": 0,
    "exclusion_reason": "<if signals were excluded, explain why>",
    "notes": "<ICP segment source used; any discrepancy vs the reference page>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "Microsoft Teams (via bdr-post-to-teams)",
    "title": "Regulatory Change Signals — <scan_date>",
    "markdown": "<rendered digest per Step 7 — one block per included signal, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5` AND `icp_match: true`.
- IDs sequential: `signal_001`, `signal_002`, etc.
- In **sweep mode**: set `source_url` to `"sweep"`, list every fetched URL in `sources_swept`, set
  `regulator` to `"multiple"` (each signal still carries its own `regulator`), apply the freshness
  window, and roll any failed-to-fetch URLs into `exclusion_reason` instead of stopping.
- Deduplicate rules across sources: a joint NPRM that appears on several agency pages (e.g. an
  OCC/FDIC/NCUA + FinCEN rule) is **one** signal — note the co-issuers in its `summary`.
- This skill never writes to Zoho and never hands off to `zoho-crud-lead`.
- If the page is an enforcement action (not a rule change): return `signals: []` and recommend
  `bdr-scan-regulatory` in `exclusion_reason`.
- If the page has a login wall, is empty, or has no rulemaking content: return `signals: []`, an empty
  `digest.markdown`, and explain in `exclusion_reason`.

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

This skill's detection mechanism: new and proposed rules, supervisory guidance, and calls for
comment from financial-services regulators, via WebFetch / curl sweeps of the Federal Register API
and the regulator rulemaking pages curated on the team's Notion source list.

Resolve which Signal Library signals this run covers — fresh, on every run:

1. Query the **Signal Library** data source
   (`collection://3904e475-1c3a-8085-8c1e-000b40a34f87`, on the **02_Signal & Source Library**
   page `https://app.notion.com/p/3904e4751c3a80b98da7c6bac9ca34c7`) with `notion-search` scoped
   via `data_source_url`, using this skill's own name (`bdr-scan-reg-change`) and these coverage terms:
   "regulatory change", "rulemaking", "proposed rule", "supervisory guidance", "comment deadline".
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

For every source actually consulted this run (the Federal Register API plus each regulator
rulemaking / newsroom page swept this run), build the template's **Sources Used** table:

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

- **Name:** `bdr-scan-reg-change — <YYYY-MM-DD> — <short descriptor of the outcome>`
- **Select:** `bdr-scan-reg-change`
- **Multi-select:** provider(s) actually used this run (`Web`)
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

**Client:** TeamEclipse — professional services firm specializing in data, risk, compliance, and
financial crime consulting for financial institutions.

**Seller-doer:** Tom — uses regulatory-change signals to reach ICP segments *before* a compliance or
comment deadline, positioning Eclipse to shape the response and own the build.

**ICP:** Mid-to-large banks, broker-dealers, investment managers, insurers, and fintechs in regulated
markets — fetched fresh from the ICP Analysis page.

**Practice areas → wedge:**
- **Financial Crime** → gap assessment vs. new AML/sanctions requirements, program redesign, remediation roadmap
- **Risk & Compliance** → regulatory-change impact assessment, new-report build, readiness testing
- **Data Management** → lineage, recordkeeping, and reporting-control build to evidence compliance
- **Governance** → accountability-regime and board-oversight uplift, control framework alignment

**Best signal:** A **final or proposed rule** that creates a new mandatory program/report in one or
more Eclipse practice areas, with a deadline inside 12 months and broad ICP applicability — maximum
net-new demand, with time for Eclipse to help *before* the scramble.

**Relationship to siblings:** Same output shape (newsletter digest, no CRM writes) as
`bdr-scan-regulatory` / `bdr-scan-vendor` / `bdr-scan-corp-events`. The proactive (rulemaking)
counterpart to the reactive (enforcement) `bdr-scan-regulatory`. Scheduled cadence: weekly sweep
across the Buy-Trigger #2 source list.
