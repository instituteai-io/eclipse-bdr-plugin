---
name: bdr-scan-vendor
description: >
  Fetch any data-tooling vendor's customer page, case study, press release, or event page and
  extract the companies named in it as Vendor Adoption signals for TeamEclipse BDR outreach.
  Covers the vendors in the team-maintained SharePoint reference table — data catalogs (Collibra,
  Alation, Atlan, Purview), privacy (BigID, Immuta, OneTrust), MDM (Informatica, Reltio,
  Profisee), data quality/observability (Monte Carlo, Ataccama), lineage (Manta, Solidatus),
  cloud data platforms (Snowflake, Databricks, Fabric), and regulatory reporting (AxiomSL,
  Workiva), among others. Auto-detects the vendor and tool category, extracts the customer
  companies, scores each against Eclipse's ICP, and returns structured JSON signals plus a
  ready-to-send newsletter digest. Two modes: by default (no URL given) it sweeps the full
  maintained list of vendor evidence URLs — this is the weekly scheduled cadence; passing one or
  more URLs scans just those. Reported signals are tracked in a Notion seen log
  (Company + Vendor Pairs) so weekly sweeps only surface NEW companies. This is a company-level
  signal — it does NOT write to Zoho; output is a report meant for email / team chat.
  Use when the user asks to run the vendor scan / sweep all vendors, or pastes a vendor
  customer/case-study/press URL to scan, find tool-adoption signals in, or check who is using a
  data tool. Triggers on:
  "scan this vendor page", "run the vendor evidence scan", "who's using Collibra/Snowflake/etc",
  "vendor adoption signal", "tool footprint scan", "bdr-scan-vendor", or any URL from a
  data-tooling vendor domain (collibra.com, alation.com, informatica.com, atlan.com, bigid.com,
  immuta.com, snowflake.com, databricks.com, montecarlo.ai, solidatus.com, etc.).
---

# Eclipse BDR — Vendor / Tool Footprint Scan

Fetch a data-tooling vendor's customer page, case study, press release, webinar, or event page,
extract the customer companies named, score each against Eclipse's ICP, and return structured
JSON signals plus a **newsletter digest** ready to send by email or team chat.

This is a **company-level** signal skill. Per the Eclipse BDR output-routing rule, company-level
signals **do not interact with Zoho** (no Accounts, no Deals — Zoho is Leads-only, and only
person-level skills write Leads). Instead, this skill produces a human-readable digest for review
and outreach prioritization.

It covers ~10 signals from the Signal Library in one skill: data catalog, MDM, data quality /
observability, lineage, privacy / sensitive-data discovery, cloud data platform, regulatory
reporting modernization, and ETL/cloud migration — all detected through vendor evidence.

---

## ⚠️ Reference Pages

Two external references drive this skill. Always fetch both fresh — never use cached values and
never fall back to a built-in vendor list.

**Page A — Vendor Table (SharePoint): Skill References - Signal Vendor Adoption** — fetch via
`read_resource`:
`file:///b!ZEjP7U2_7UCmi5bHlWbQqhD0IvI686dAqFFpekklCTTT7yqCdusrTarAzUvLNKPW/015GTRKZE244FO7ZTJTFGJQOECUI4RB2M5`

If that URI fails (file moved or re-uploaded), locate it with `sharepoint_search` (query:
`"Skill References - Signal Vendor Adoption"`, folderName: `"Skill References"`) and read the
URI it returns instead.

Provides three sections:

1. **Vendor Table** — vendor/product, domain(s) to match URLs against, tool category, customer
   evidence URL (for scheduled sweeps), notes (acquisitions, bot-blocked pages, product-level
   category overrides like Collibra DQ / Collibra Lineage)
2. **Tool Category Table** — per category: wedge tier (`core` / `adjacent`), practice areas, and
   the Eclipse wedge text
3. **Scheduled Scan URLs** — one-off URLs to include in scheduled sweeps beyond the per-vendor
   evidence URLs

If Page A is unreachable or empty, **stop and tell the user**. Do not proceed with a guessed
vendor table.

**Page B — Seen Log (Notion): Company + Vendor Pairs** — database of company + vendor pairs
already reported in a digest, so weekly sweeps surface only *new* signals.

- Database: `https://app.notion.com/p/37b9b05a813f802ba6dfd914a15ae570` (lives under the page
  *Skill References - Signal Vendor Adoption (Seen Log)*,
  `https://app.notion.com/p/37b9b05a813f80348d1ad1dad42398e0`)
- Data source for writes: `collection://37b9b05a-813f-8047-81f7-000b6a9eeb45`
- Columns: `Vendor` (title), `Customer` (text), `Found on` (date), `URL`, `Notes`

Read it in Step 6, write back in Step 8. If Page B is unreachable, **continue the scan** (the
output is still useful) but say so prominently, treat nothing as seen, and skip the write-back —
never risk double-logging.

---

## Step 1: Pick the Mode

**Sweep mode (default)** — the skill was invoked with no URL. This is the normal case (it will
run weekly as a scheduled routine). Scan every **Customer Evidence URL** in the Vendor Table plus
every entry under **Scheduled Scan URLs**, skipping rows marked TBD. Report any URL that fails to
fetch in `exclusion_reason` rather than stopping the sweep. Do not ask the user for a URL.

**Manual mode** — the user passed one or more URLs as arguments. Scan only those; process each
and merge into one output. Only ask for a URL if the user clearly meant a specific page (e.g.
"scan this page") but no URL actually came through.

---

## Step 2: Detect the Vendor and Tool Category

Match the URL domain (or page content if the domain is generic, e.g. a press wire or PR Newswire
release) against the **Vendor Table** from the reference page. The matched row gives the vendor,
tool category, and any notes; the **Tool Category Table** gives that category's Eclipse wedge,
practice areas, and wedge tier.

Apply any product-level overrides from the row's notes (e.g. a Collibra DQ story is
Data Quality / Observability, not Data Catalog / Governance).

If the domain matches nothing in the table, set vendor and category to **Unknown** and use a
generic data governance & operating-model uplift wedge — and say so in the summary, so the team
can decide whether to add the vendor to the reference page.

If the page references **cloud migration + a catalog** (e.g. Snowflake/Databricks plus a
governance tool), or **PowerCenter/Informatica cloud migration**, treat it as a migration-driven
governance signal and note that in the summary (Signal Library: catalogs and ETL modernization
often expose discoverability, lineage, and ownership gaps).

---

## Step 3: Fetch and Extract

Use WebFetch to retrieve the page. Extract every **customer company** named as using, deploying,
or evaluating the vendor's product. For each, capture:

- `company` — the customer organization name (NOT the vendor)
- `evidence_type` — one of: `Case Study`, `Customer Story`, `Press Release`, `Webinar / Event`, `Customer Logo / List`, `Partner / SI Announcement`, `Marketplace Listing`
- `evidence_detail` — 1–2 sentences on what the page says this company did with the tool
- `source_url` — direct URL to the specific story if available, otherwise the page URL

**Exclude** the vendor itself, the vendor's partners/SIs (unless the SI announcement names an end
customer — then capture the end customer), and generic mentions with no named company.

If the page is a login wall, is empty, or names no customer companies: return `signals: []` with
a note in `exclusion_reason`.

---

## Step 4: Score Eclipse ICP Relevance

Score every extracted company from 1–10.

| Condition | Points |
|---|---|
| Company is a bank, credit union, or trust company | +3 |
| Company is a broker-dealer, investment/asset manager, or insurer | +2 |
| Company is a fintech, payments company, or money services business | +1 |
| Evidence is a named case study, customer story, or press release (confirmed deployment) | +3 |
| Evidence is a customer logo/list, webinar, or conference appearance | +2 |
| Evidence is a partner/SI announcement or marketplace listing | +1 |
| Tool category's wedge tier is `core` (per the Tool Category Table) | +2 |
| Tool category's wedge tier is `adjacent` (per the Tool Category Table) | +1 |
| Company is not a financial institution and not a named target vertical | -2 |

**Practice area mapping** — take the practice areas from the matched category's row in the
**Tool Category Table** (an account can map to multiple — include all that apply). One addition
the table can't express: add **Financial Crime** only if the evidence explicitly ties the tool to
BSA/AML, sanctions, or KYC data.

**Urgency:**
- `high`: Named case study or press release at a bank/insurer for a `core`-tier tool (confirmed funded initiative in an Eclipse core wedge)
- `medium`: Customer story, webinar, or logo for any tool at an ICP-fit FS firm
- `low`: Partner announcement, marketplace listing, or evidence at a non-core vertical

**ICP match:** `true` if the company is a financial-services firm (or a named target vertical) AND `relevance_score >= 5`.

---

## Step 5: Generate Eclipse Assessment

For each signal with `relevance_score >= 5`, generate:

**`wedge`** — Eclipse service angle (1–2 sentences), built from the category's wedge text in the
**Tool Category Table** and made specific to this company and evidence. The hidden hypothesis
(Signal Library): *the tool purchase is a symptom of a larger data-trust, regulatory,
self-service, or AI-readiness push — they likely bought the tool but lack the people/process
operating model to scale it.* Lead the wedge with that gap.

**`recommended_approach`** — Tom's opening outreach angle (1–2 sentences). Reference the specific
tool and what the evidence shows, then position Eclipse on "what comes after the tool": operating
model, adoption, stewardship, controls, and value realization.

---

## Step 6: Check the Seen Log

Fetch all rows from the **Company + Vendor Pairs** database (Page B). A signal is **already
seen** if its company + vendor matches an existing row. Match loosely: case-insensitive, ignore
legal suffixes (Inc., Corp., Ltd., N.A.), and treat product variants as the parent vendor
("Collibra DQ" matches a "Collibra" row).

- **Sweep mode:** drop already-seen signals from the digest entirely; count them in
  `summary.already_seen`. This is what keeps the weekly newsletter from repeating itself.
- **Manual mode:** keep them in the digest but mark the block "(previously reported
  \<Found on date\>)" — the user asked about this specific page and wants the full picture.
  Don't re-log them in Step 8.

---

## Step 7: Build the Newsletter Digest

This skill does **not** write to Zoho and produces **no** `crm_payload`. Instead, assemble a
`digest` object that can be dropped into an email or team-chat post. Include only signals with
`relevance_score >= 5`, sorted by `relevance_score` descending (high-priority first).

The eventual delivery will be a **newsletter** (email or team chat — channel not yet decided), so
keep `digest.markdown` clean and channel-neutral. The exact format is intentionally open for now
— use judgment — as long as each signal block covers: company, tool category, score and urgency,
the evidence (type + one line), why it matters, the Eclipse wedge, and the suggested approach.
A reasonable starting shape, not a contract:

```
## Vendor Adoption Signals — <scan_date>

### 🔹 <Company> — <tool_category> (score <relevance_score>/10, <urgency>)
**Evidence:** <evidence_type> — <one-line evidence_detail>
**Why it matters:** <why this is a funded-initiative signal>
**Eclipse wedge:** <wedge>
**Suggested approach:** <recommended_approach>
[Source](<source_url>)
```

---

## Step 8: Write New Signals to the Seen Log

For every signal that made the digest and was **not** already seen, add one row to the
Company + Vendor Pairs database (`notion-create-pages`, parent data source
`collection://37b9b05a-813f-8047-81f7-000b6a9eeb45`):

- `Vendor` (title) — vendor name as it appears in the Vendor Table
- `Customer` — the customer company name
- `date:Found on:start` — today's scan date
- `userDefined:URL` — the signal's `source_url`
- `Notes` — `<evidence_type> — <tool_category>`, plus anything worth remembering

The log records what has been **reported**, not everything extracted — below-threshold companies
are not logged (they're cheap to re-score, and their relevance may change).

Skip this step entirely if the seen log was unreachable in Step 6.

---

## Step 9: Return JSON

Output **only** this JSON — no prose before or after unless the user asks for a summary.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-scan-vendor",
  "output_type": "digest",
  "source_url": "<the URL that was scanned, or 'sweep' for a full-list run>",
  "vendor": "<detected vendor name or Unknown>",
  "tool_category": "<a category from the Tool Category Table, or Unknown>",
  "signals": [
    {
      "id": "signal_001",
      "company": "<customer company name>",
      "vendor": "<vendor name>",
      "tool_category": "<tool category>",
      "evidence_type": "<Case Study | Customer Story | Press Release | Webinar / Event | Customer Logo / List | Partner / SI Announcement | Marketplace Listing>",
      "date": "<YYYY-MM-DD or best estimate, else null>",
      "summary": "<2–3 sentences: what the page says this company did with the tool>",
      "source_url": "<direct URL to this specific story if available, otherwise the page URL>",
      "eclipse_assessment": {
        "practice_areas": ["<practice area 1>", "<practice area 2>"],
        "wedge": "<Eclipse service angle specific to this company and tool>",
        "urgency": "high|medium|low",
        "budget_signal": true,
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's outreach angle specific to this signal>"
      }
    }
  ],
  "summary": {
    "total_extracted": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "signals_excluded": 0,
    "already_seen": 0,
    "logged_to_seen_log": 0,
    "exclusion_reason": "<if signals were excluded or URLs failed to fetch, explain why>"
  },
  "digest": {
    "format": "newsletter",
    "channel": "TBD (email | team chat)",
    "title": "Vendor Adoption Signals — <scan_date>",
    "markdown": "<rendered digest per Step 7 — one block per included signal, high-priority first>"
  }
}
```

- For multi-URL runs (paste mode with several URLs, or sweep mode), set `vendor` and
  `tool_category` to `"multiple"` at the top level — each signal carries its own
- Include only signals with `relevance_score >= 5`
- IDs sequential: `signal_001`, `signal_002`, …
- Deduplicate companies: if the same company appears across multiple signals/tools, keep one signal entry and note the multiple tools in its `summary`
- This skill never writes to Zoho and never hands off to `zoho-crud-lead`
- If the page has a login wall, is empty, or names no customer companies: return `signals: []`, an empty `digest.markdown`, and explain in `exclusion_reason`

---

## Eclipse Context

**Client:** TeamEclipse — professional services firm specializing in data, risk, compliance, and financial crime consulting for financial institutions.

**Seller-doer:** Tom — uses vendor evidence to identify accounts with a *funded* data initiative (a paid tool) that still need the operating model, stewardship, controls, and adoption work to realize value.

**Output routing:** Company-level signal → **newsletter digest only**, no CRM writes. Zoho is reserved for the Inbound (Leads) module, written exclusively by person-level signal skills via `zoho-crud-lead`.

**Why this signal matters (Signal Library):** A company appearing in a vendor's customer material has made a real investment in governance, catalog, privacy, metadata, MDM, lineage, DQ, or a cloud data platform. The tool purchase is a symptom of a larger data-trust, regulatory, self-service, or AI-readiness push.

**Hidden hypothesis to lead with:** They bought or are implementing a tool but lack the people/process model to scale it. Eclipse sells "what comes after the tool."

**Best signal:** A named case study or press release at a mid-to-large bank or insurer for a governance, lineage, data-quality, MDM, or regulatory-reporting tool — confirms budget and a live initiative in an Eclipse core practice area.
