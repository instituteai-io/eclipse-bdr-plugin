---
name: bdr-scan-corp-events
description: >
  WebFetch sibling of the Corporate Events signal. Fetch a pasted URL — press release, M&A news,
  system-integrator partnership announcement, AI/GenAI platform announcement, public RFP / procurement
  notice, or investor deck / 10-K / annual-report excerpt — and extract the corporate event as a
  Corporate-Event signal for TeamEclipse BDR. Covers ~5 Signal-Library signals in one skill: M&A /
  post-merger integration, SI & strategic partnerships, AI/GenAI announcements, public RFPs for
  data/analytics/risk-reporting/AI-governance, and investor-deck / annual-report data-or-AI narrative.
  Auto-classifies the event, extracts company + counterparty + date + what was announced, maps the
  Eclipse wedge per event type, scores ICP relevance, and returns structured JSON signals plus a
  ready-to-send newsletter digest. This is the URL-paste twin of bdr-signal-corp-events-zoominfo
  (which sweeps ZoomInfo Scoops across the whole ICP in one pass) — run the two side-by-side for a
  coverage comparison. This is a COMPANY-LEVEL signal — it does NOT write to Zoho; output is a report
  for email / team chat.
  Use when the user pastes a corporate-event URL and asks to scan it, or names a company + event type
  ("any corporate-event signals for <company>") and wants the source located and scanned.
  Triggers on: "scan this acquisition / merger / partnership / RFP / investor deck", "run the corp
  events scan", "who's acquiring / partnering / announced AI", "corporate event signal",
  "bdr-scan-corp-events", or a press-release / M&A / procurement / investor-relations URL.
---

# Eclipse BDR — Corporate Events Scan (WebFetch)

Fetch a pasted URL (or locate one from a company + event-type prompt), extract the corporate event,
score it against Eclipse's ICP, and return structured signals plus a **newsletter digest** for email
/ team chat.

This is the **WebFetch twin of `bdr-signal-corp-events-zoominfo`**. The ZoomInfo version sweeps
curated Scoops across the entire ICP in one call; this version reads **one pasted source at a time**
(or finds the primary source for a named company + event). Run both and diff them for coverage —
see "ZoomInfo vs WebFetch" at the end.

This is a **company-level** signal skill. Per the Eclipse BDR output-routing rule, company-level
signals **do not interact with Zoho** (Zoho is the Leads-only Inbound module, written exclusively by
person-level skills via `zoho-crud-lead`). This skill produces a human-readable digest only —
**never a `crm_payload`**.

---

## ⚠️ ICP Reference Page

ICP firmographics are **not hardcoded** — fetch them fresh on every run from the **ICP Analysis**
Notion page: `https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0`

Fetch via the Notion MCP `notion-fetch` tool. From **Section 1 (Firm-Wide ICP)** extract company
**size** (employee range), **revenue** range, **primary industries**, and **geography** — used in
Step 4 scoring.

> Current sanity-check values (always trust the page over this note): Financial Services core (Banks,
> Asset/Investment Management, Insurance), US primary, mid-to-large institutions. If the page
> disagrees, trust the page and note the difference in the summary.

If the page is unreachable, **still score** using the fallback FS-core definition above, but flag in
`summary.notes` that firmographics came from the fallback rather than the live page.

---

## Step 1: Confirm the Input

Two modes:

1. **URL paste (primary).** The user gives one or more URLs. If multiple, process each and merge into
   one output.
2. **Company + event prompt.** The user names a company and an event type ("any corporate-event
   signals for Acme Bank", "did Acme announce an AI initiative?"). Use **WebSearch** to locate the
   primary source (the company's press release, the M&A news item, the procurement portal, the
   investor-relations 10-K/deck), confirm the URL, then proceed as in mode 1.

If neither is given, ask:

> "Paste a corporate-event URL (M&A, partnership, AI announcement, RFP, or investor deck), or name a
> company and the event you want me to look for."

---

## Step 2: Classify the Event

Identify which of the five corporate-event families the source describes:

| Event family | What the source looks like |
|---|---|
| **M&A / post-merger integration** | Acquisition / merger press release, deal news, "completes acquisition of…", integration announcement |
| **SI / strategic partnership** | Partnership with a system integrator (Accenture, Deloitte, KPMG, EY, PwC, Capgemini, IBM, etc.) or a strategic alliance announcement |
| **AI / GenAI announcement** | "launches AI platform", GenAI initiative, AI center of excellence, model deployment news |
| **Public RFP / procurement** | RFP or procurement notice for data catalog / MDM / analytics / risk reporting / AI governance / data platform |
| **Investor-deck / annual-report narrative** | 10-K / 10-Q / annual report / investor deck with a "data-driven", "AI", or "digital transformation" mandate |

If the source is none of these (e.g. an earnings beat with no data/AI/transformation angle), return
`signals: []` with the reason in `exclusion_reason`.

---

## Step 3: Fetch & Extract

Use **WebFetch** to retrieve the page (PDFs — investor decks, 10-Ks — may need careful extraction;
pull the relevant section). Extract per event:

- **Company** — the ICP-side company (the buyer/acquirer in M&A, the firm issuing the RFP, the
  reporting entity for an investor narrative).
- **Counterparty** — acquired target, SI partner, vendor, or null.
- **Date** — announcement / filing / RFP-close date.
- **What was announced** — the substantive event, in plain English.
- **Mandate / driver** — the underlying data / governance / risk / AI push the event signals (the
  hidden hypothesis: the event is a *symptom* of a larger data-trust or AI-readiness need).

---

## Step 4: Classify the Wedge & Score ICP Relevance

Map the Eclipse wedge per event type:

| event_type | Eclipse wedge to lead with |
|---|---|
| M&A / post-merger | Integration data assessment, taxonomy & lineage reconciliation, single-source-of-truth, controls alignment across the merged entities. |
| SI / strategic partnership | Independent advisory / QA over the SI-led rollout — protect the data quality, governance, and value realization the SI won't own. |
| AI / GenAI announcement | AI-readiness: the data foundation, governance, and lineage the AI story needs but usually lacks. |
| Public RFP | Tool-agnostic requirements definition, evaluation support, and adoption/governance the RFP itself won't specify. |
| Investor narrative | Translate the aspiration ("data-driven", "AI-led") into an executable data & governance strategy. |

Score each event 1–10:

| Condition | Points |
|---|---|
| Company is a bank, credit union, or trust company | +3 |
| Company is a broker-dealer, asset/investment manager, or insurer | +2 |
| Company is a fintech, payments, or money-services business | +1 |
| Event is M&A / post-merger integration (highest data-fragmentation demand) | +3 |
| Event is a public RFP for data/analytics/risk-reporting/AI-governance (active budget) | +3 |
| Event is an AI/GenAI announcement | +2 |
| Event is an SI / strategic partnership | +1 |
| Event is an investor-narrative signal only (story may be ahead of execution) | -1 |
| Company outside FS and not a named target vertical | -3 |

**Urgency:**
- `high` = M&A at an in-band bank/insurer, OR a public RFP in an Eclipse practice area
- `medium` = AI announcement or SI partnership at an ICP-fit FS firm
- `low` = investor-narrative-only, or any event at a non-core vertical

**ICP match:** `true` if the company is FS (or a named target vertical) AND `relevance_score >= 5`.

> Investor-narrative signals are **soft** — score them lower and flag them as nurture in the
> `recommended_approach` (the story may be ahead of the execution).

---

## Step 5: Generate the Eclipse Assessment

For each signal with `relevance_score >= 5`, generate:

**`wedge`** — the Eclipse service angle, specific to this event (from the table above, made concrete
to the company and counterparty).

**`recommended_approach`** — Tom's outreach angle (1–2 sentences): reference the specific event and
counterparty, lead with the wedge, frame the timing window the event opens.

---

## Step 6: Build the Newsletter Digest

This skill writes **no** Zoho records and produces **no** `crm_payload`. Assemble a `digest` object
for email / team chat. Include only signals with `relevance_score >= 5` AND `icp_match: true`, sorted
by `relevance_score` descending. Keep `digest.markdown` clean and channel-neutral.

Render one block per signal:

```
## Corporate Event Signals — <scan_date>

### 🔹 <Company> — <event_type> (score <relevance_score>/10, <urgency>)
**Event:** <what happened — counterparty, amount, etc.> (<event date>)
**Why it matters:** <data/governance/risk/AI demand this creates>
**Eclipse wedge:** <wedge>
**Suggested approach:** <recommended_approach>
[Source](<source_url>)
```

> **Delivery format is TBD** (email vs. team chat). Keep `digest.markdown` clean and channel-neutral;
> `digest.channel` is a placeholder until the team picks the channel. To post it, hand
> `digest.markdown` to `bdr-post-to-teams`.

---

## Step 7: Return JSON

Output **only** this JSON — no prose before or after unless the user asks for a summary.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-scan-corp-events",
  "signal_source": "WebFetch",
  "output_type": "digest",
  "source_url": "<the URL that was scanned>",
  "criteria_fetched_from": { "reference_page": "https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0" },
  "signals": [
    {
      "id": "signal_001",
      "company": "<ICP-side company name>",
      "event_type": "M&A | Partnership | AI Announcement | RFP | Investor Narrative",
      "counterparty": "<acquired target / SI partner / vendor, or null>",
      "date": "<YYYY-MM-DD or best estimate>",
      "summary": "<2–3 sentences: what happened and the underlying data/AI/risk mandate>",
      "source_url": "<direct URL to this event if available, otherwise the page URL>",
      "eclipse_assessment": {
        "practice_areas": ["<practice area 1>", "<practice area 2>"],
        "wedge": "<Eclipse service angle specific to this event>",
        "urgency": "high|medium|low",
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's outreach angle specific to this event>"
      }
    }
  ],
  "summary": {
    "total_extracted": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "by_event_type": { "M&A": 0, "Partnership": 0, "AI Announcement": 0, "RFP": 0, "Investor Narrative": 0 },
    "signals_excluded": 0,
    "exclusion_reason": "<if signals were excluded, explain why>",
    "notes": "<ICP firmographics source used; any discrepancy vs the reference page>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "TBD (email | team chat)",
    "title": "Corporate Event Signals — <scan_date>",
    "markdown": "<rendered digest per Step 6 — one block per included signal, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5` AND `icp_match: true`.
- IDs sequential: `signal_001`, `signal_002`, etc.
- This skill never writes to Zoho and never hands off to `zoho-crud-lead`.
- If the page is not a corporate event, has a login wall, or is empty: return `signals: []`, an empty
  `digest.markdown`, and explain in `exclusion_reason`.

---

## ZoomInfo vs WebFetch (`bdr-signal-corp-events-zoominfo`) — what to compare

This skill is the **WebFetch arm** of a deliberate A/B with the ZoomInfo arm. When comparing:

- **Coverage & recall** — the ZoomInfo version returns the full set of recent events across the ICP
  in one call; this version only sees the pasted URL (or one located source). Use this version to
  go *deep* on a specific event the ZoomInfo sweep flagged, or to catch event types ZoomInfo Scoops
  underweight (public **RFPs** and **investor-narrative** signals are weaker in Scoops — this skill's
  edge).
- **Freshness** — pointed straight at a same-day press release, WebFetch can beat research-curated
  Scoops by a day or two.
- **Precision** — both struggle most with the AI-announcement family; compare how cleanly each
  isolates a genuine AI initiative from generic product news.

---

## Eclipse Context

**Client:** TeamEclipse — data, risk, compliance, and financial crime consulting for financial
institutions. **Seller-doer:** Tom.

**Signal (Signal Library):** A corporate event — M&A, SI/strategic partnership, AI announcement,
public RFP, or an investor-deck data/AI narrative — opens a window of data, governance, and risk
demand. M&A fragments data estates; partnerships hand execution to an SI who won't own data quality;
AI announcements outrun the data foundation; RFPs allocate budget; investor narratives commit a
mandate ahead of the capability to deliver it.

**Eclipse wedge:** "What comes after the event" — integration data assessments and taxonomy/lineage
reconciliation (M&A), independent advisory/QA over SI rollouts, AI-readiness and data-foundation
work, tool-agnostic requirements & adoption support (RFPs), and aspiration→executable-strategy
translation (investor narrative).

**Output routing:** Company-level signal → **newsletter digest only**, no CRM writes.

**Relationship to siblings:** Same output shape as `bdr-scan-regulatory` / `bdr-scan-reg-change` /
`bdr-scan-vendor`; the WebFetch twin of `bdr-signal-corp-events-zoominfo`. Fetches firmographics
fresh from the ICP Analysis page rather than a dedicated keyword reference page.
