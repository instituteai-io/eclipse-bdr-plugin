---
name: bdr-signal-vendor-apollo
description: >
  Apollo variant of the Vendor / Tool Footprint signal. Instead of scraping vendor customer pages
  (what the WebFetch sibling bdr-scan-vendor does) or querying ZoomInfo tech tags (what
  bdr-signal-vendor-zoominfo does), it harvests the install base directly from Apollo firmographics via
  currently_using_any_of_technology_uids — answering "which ICP-fit companies currently use Collibra,
  Alation, Atlan, Purview, Informatica, Reltio, BigID, Immuta, OneTrust, Monte Carlo, Ataccama, Manta,
  Solidatus, Snowflake, Databricks, AxiomSL, Workiva, etc." as a structured, ICP-filtered list. Reads
  the SAME Vendor Table reference page the web and ZoomInfo scans use, resolves each vendor to its
  Apollo technology uid (lowercase/underscore convention), filters apollo_mixed_companies_search by tech
  uid + the narrowed firm-wide ICP, scores each company against Eclipse's ICP, dedupes against the
  shared Company+Vendor Pairs seen log, and returns a newsletter digest. This is a company-level signal
  — it does NOT write to Zoho; output is a report for email / team chat. Built to run side-by-side with
  bdr-scan-vendor (web) and bdr-signal-vendor-zoominfo (ZoomInfo) for a coverage comparison; on accounts
  without a ZoomInfo license this is the structured-footprint path.
  Use when asked which ICP companies use a given data tool via Apollo, to run the Apollo vendor / tool
  footprint scan, or harvest an install base without ZoomInfo.
  Triggers on: "which banks use Collibra/Snowflake/etc via apollo", "run the apollo vendor footprint
  scan", "apollo tool adoption signal", "harvest the install base with apollo",
  "bdr-signal-vendor-apollo".
---

# Eclipse BDR — Signal: Vendor / Tool Footprint — Apollo

Harvest the **install base** of Eclipse-relevant data tools directly from Apollo firmographics —
`currently_using_any_of_technology_uids` — answering *"which ICP-fit companies currently use Collibra /
Alation / Informatica / Snowflake / Databricks / BigID / Monte Carlo / etc."* as a structured,
ICP-filtered list. Output is a **newsletter digest**.

This is the Apollo twin of `bdr-scan-vendor` (WebFetch) and `bdr-signal-vendor-zoominfo` (ZoomInfo). The
web scan finds **net-new public evidence** (case studies, press, logos); the tech-graph scans return the
**whole detectable footprint**, filtered to the ICP. On accounts **without a ZoomInfo license, this
Apollo scan is the structured-footprint path.** Run alongside the web scan: the web scan finds the *new*
story, this finds the *full* list.

This is a **company-level** signal skill — it **does not interact with Zoho** (Zoho is Leads-only).
Output is a digest only.

---

## ⚠️ Reference Pages

Always fetch fresh — never use a built-in vendor list or cached values.

**Page A — Vendor Table (Notion): Skill References - Signal Vendor Adoption** — the **same** page
`bdr-scan-vendor` and `bdr-signal-vendor-zoominfo` use, so all three stay in lockstep:
`https://app.notion.com/p/b0d4e4751c3a8320b9ef011a9a975642`
(in the **Skill References** database; if the URL fails, locate it with `notion-search` for
`"Skill References - Signal Vendor Adoption"`)

Provides the **Vendor Table** (vendor/product, domain(s), tool category, notes incl. product-level
category overrides like Collibra DQ / Collibra Lineage) and the **Tool Category Table** (per category:
wedge tier `core`/`adjacent`, practice areas, Eclipse wedge text). Use the vendor/product names here as
the input to the Apollo technology-uid resolution in Step 2.

If Page A is unreachable or empty, **stop and tell the user**.

**Page B — Seen Log (Notion): Company + Vendor Pairs** — the **shared** seen log, reused so the vendor
skills don't double-report:
- Database: `https://app.notion.com/p/37b9b05a813f802ba6dfd914a15ae570`
- Data source for writes: `collection://37b9b05a-813f-8047-81f7-000b6a9eeb45`
- Columns: `Vendor` (title), `Customer` (text), `Found on` (date), `URL`, `Notes`

Read it in Step 5, write back in Step 7. If Page B is unreachable, continue the scan but say so, treat
nothing as seen, and skip the write-back (never risk double-logging).

**ICP firmographics** — fetch fresh from the **ICP Analysis** page
(`https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0`), Section 1. Current expected band (sanity
check only — trust the page): **500–2,000 employees, $250M–$1B**, FS-core, US primary.

---

## 💳 Credits Note

`apollo_mixed_companies_search` **costs 1 Apollo credit per request.** This skill batches vendors into
as few requests as possible (see Step 3) and does **not** enrich contacts (no emails/phones). Always
report the number of Apollo requests / credits used in the run summary. If the user wants a bounded
run, scan only the vendors they name.

---

## Step 1: Pick Vendors & Resolve ICP

1. Fetch Page A; take the vendor/product list from the Vendor Table (or, if the user named specific
   vendors, just those).
2. Fetch the ICP Analysis page; extract industries, size, revenue, geography (Section 1). Translate to
   Apollo company filters:

| ICP field | Apollo parameter |
|---|---|
| industries | `q_organization_keyword_tags` |
| size | `organization_num_employees_ranges` (e.g. `"500,2000"`) |
| geography | `organization_locations` |

(Apollo does not filter on revenue as cleanly as headcount — rely on the employee band and note revenue
as a manual sanity check in the digest.)

---

## Step 2: Resolve Each Vendor to an Apollo Technology UID

Apollo's tech filter keys on **technology uids** — lowercase, underscore-separated slugs of the product
name (e.g. `collibra`, `alation`, `snowflake`, `databricks`, `informatica`, `microsoft_purview`,
`monte_carlo`, `bigid`, `immuta`). For each vendor/product:

1. Derive the candidate uid from the product name using that convention.
2. Confirm it by running a minimal `apollo_mixed_companies_search` with only that uid (or check Apollo's
   technology reference if available) — a uid that returns zero companies across the whole index is
   likely wrong; try an alternate slug (e.g. `microsoft_purview` vs `azure_purview`).

Apply any product-level category override from the Vendor Table notes (e.g. a Collibra DQ product maps
to Data Quality / Observability, not Data Catalog / Governance). If a vendor has no resolvable Apollo
technology uid, note it as **not detectable in Apollo's tech graph** and skip (the web scan may still
catch it).

Record the uid you used for each vendor so the run is reproducible.

---

## Step 3: Search the Install Base

Query `apollo_mixed_companies_search`. To conserve credits, pass **multiple** vendor uids in one call
where you want the union (any-of), then attribute the matched tool(s) per company from the response:

| Filter | Value |
|---|---|
| `currently_using_any_of_technology_uids` | the resolved vendor uid(s) |
| `q_organization_keyword_tags` | resolved industries |
| `organization_num_employees_ranges` | `["500,2000"]` |
| `organization_locations` | geography |
| `per_page` | 25, paginate if needed |

Capture per company: `name`, Apollo `organization id`, `domain`/`website`, `employee_count`,
`estimated_num_employees`, `city`, `state`, plus which vendor/tool matched (and its tool category from
Page A). If Apollo returns the org's technology list, use it to attribute **all** Eclipse-relevant tools
each company runs (a multi-tool estate scores higher in Step 6).

---

## Step 4: (Reserved) Second-Source Cross-Check

If the team also has ZoomInfo and wants a second source, run `bdr-signal-vendor-zoominfo` on the same
vendors and compare footprints — do not duplicate that logic here. On Apollo-only accounts, skip.

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

Score each company 1–10 (firmographic-only — there's no "evidence type" here, the tech uid *is* the
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
**Financial Crime** only if the tool ties to BSA/AML/sanctions/KYC). **ICP match:** `true` if FS (or
named vertical) AND `relevance_score >= 5`. **Urgency:** `high` = `core`-tier tool at a bank/insurer;
`medium` = any Eclipse tool at an ICP-fit FS firm; `low` = adjacent tool or non-core vertical.

Build a `digest` (only `relevance_score >= 5`, sorted desc). Starting shape (not a contract):

```
## Vendor Footprint Signals (Apollo) — <scan_date>

### 🔹 <Company> — <tool(s)> / <tool_category> (score <relevance_score>/10, <urgency>)
**Detected stack:** <tools detected via Apollo technographics>
**Why it matters:** <funded data initiative; bought the tool, needs the operating model>
**Eclipse wedge:** <wedge from Tool Category Table, specific to this company>
**Suggested approach:** <Tom's outreach angle — "what comes after the tool">
```

Lead the wedge with the hidden hypothesis (Signal Library): *they bought/implemented the tool but lack
the people/process operating model to scale it — Eclipse sells what comes after the tool.*

---

## Step 7: Write New Signals to the Seen Log

For every company that made the digest and was **not** already seen, add one row to the Company + Vendor
Pairs database (`notion-create-pages`, parent data source
`collection://37b9b05a-813f-8047-81f7-000b6a9eeb45`):

- `Vendor` (title) — vendor name as it appears in the Vendor Table
- `Customer` — the company name
- `date:Found on:start` — today's scan date
- `userDefined:URL` — `""` or the Apollo company URL if available (this signal is firmographic, not a page)
- `Notes` — `Apollo tech footprint — <tool_category>` plus the detected tool(s)

Skip this step entirely if the seen log was unreachable in Step 5. Below-threshold companies are not
logged.

---

## Step 8: Return JSON

Output **only** this JSON — no prose before or after unless the user asks for a summary.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-vendor-apollo",
  "signal_source": "Apollo",
  "output_type": "digest",
  "criteria_fetched_from": {
    "vendor_table": "https://app.notion.com/p/b0d4e4751c3a8320b9ef011a9a975642",
    "icp_analysis": "https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0"
  },
  "signals": [
    {
      "id": "signal_001",
      "company": "<name>",
      "apollo_organization_id": "<id>",
      "domain": "<website>",
      "tools_detected": ["<tool1>", "<tool2>"],
      "technology_uids": ["<uid1>", "<uid2>"],
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
    "vendors_not_in_apollo_tech_graph": [],
    "apollo_requests_used": 0,
    "companies_found": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "already_seen": 0,
    "logged_to_seen_log": 0,
    "enriched": false,
    "notes": "<ICP band used; any discrepancy vs reference pages; credit usage>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "TBD (email | team chat)",
    "title": "Vendor Footprint Signals (Apollo) — <scan_date>",
    "markdown": "<rendered digest per Step 6 — one block per included signal, high-priority first>"
  }
}
```

- Include only signals with `relevance_score >= 5`; IDs sequential `signal_001`, …
- Deduplicate companies: one entry per company, list all detected tools in `tools_detected`.
- Never writes to Zoho; never hands off to `zoho-crud-lead`.
- If nothing qualifies: `signals: []`, empty `digest.markdown`, explain in `summary.notes`.

---

## Apollo vs ZoomInfo vs WebFetch — what to compare

- **Recall vs novelty** — the tech-graph scans (Apollo, ZoomInfo) return the full ICP-filtered
  footprint; the web scan only surfaces companies a vendor publicly names, but catches *new* case
  studies/press the tech graph hasn't ingested.
- **Apollo vs ZoomInfo** — coverage differs per vendor; on accounts without a ZoomInfo license, Apollo
  is the structured-footprint path. Where both are available, run each and union the results.
- **Recency** — a tech uid reflects detected/declared usage, not necessarily a *recent* purchase; pair
  with the web scan's dated case studies or a hiring signal to confirm the footprint is a live program.
- **Contactability** — Apollo companies carry an `organization id`, so contacts can be pulled in a
  follow-on pass via `bdr-fetch-contacts-from-company`.

---

## Eclipse Context

**Client:** TeamEclipse — data, risk, compliance, financial crime consulting for FIs.
**Seller-doer:** Tom — uses tool footprint to find accounts with a *funded* data initiative (a paid
tool) that still need the operating model, stewardship, controls, and adoption work to realize value.

**Output routing:** Company-level signal → **newsletter digest only**, no CRM writes.

**Why this signal matters (Signal Library):** A company running a governance / catalog / privacy / MDM /
lineage / DQ / cloud-data tool has made a real investment — a symptom of a larger data-trust,
regulatory, self-service, or AI-readiness push.

**Hidden hypothesis to lead with:** they bought or are implementing a tool but lack the people/process
model to scale it. Eclipse sells "what comes after the tool."

**Relationship to siblings:** Apollo-firmographics twin of the WebFetch `bdr-scan-vendor` and the
ZoomInfo `bdr-signal-vendor-zoominfo`; shares their Vendor Table reference page and Company + Vendor
Pairs seen log. Same digest output shape as the other company-level scans.
