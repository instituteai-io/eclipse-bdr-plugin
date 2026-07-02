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
---

# Eclipse BDR — Signal: Funding & Rapid Growth (Apollo)

Surface ICP-fit companies that **just raised capital or are scaling headcount fast**. New capital and
rapid growth reliably precede investment in data foundations, governance, risk/controls maturity, and
transformation — exactly Eclipse's wedges — so this is a distinctive net-new pipeline source. Output is
a **newsletter digest**.

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

Build a `digest` (only `relevance_score >= 5`, sorted desc). Starting shape (not a contract):

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

Output **only** this JSON — no prose before or after unless the user asks for a summary.

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
    "channel": "TBD (email | team chat)",
    "title": "Funding & Growth Signals (Apollo) — <scan_date>",
    "markdown": "<rendered digest per Step 4 — one block per included signal, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5`; IDs sequential `signal_001`, …
- Never writes to Zoho; never hands off to `zoho-crud-lead`.
- If nothing qualifies: `signals: []`, empty `digest.markdown`, explain in `summary.notes`.

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
