---
name: bdr-signal-hiring-risk-zoominfo
description: >
  ZoomInfo variant of the Risk & Regulatory Hiring signal. Detects ICP-fit companies actively
  hiring for Risk & Regulatory roles — regulatory change, risk transformation, financial crime /
  AML / KYC / sanctions, operational risk, internal audit, controls testing, regulatory reporting,
  model risk, and similar. Fetches the same company ICP criteria and risk keyword list from the
  team's Notion reference page that the Apollo version uses, finds ICP companies in ZoomInfo,
  detects hiring via ZoomInfo Scoops (Open Position / Hiring Plans scoop types) instead of a
  job-postings endpoint, then surfaces ICP-matching contacts (CRO, CCO, Head of Financial Crime,
  Head of Regulatory Change, etc.) at those companies as leads for the Inbound (Leads) module.
  Output is a JSON payload ready for the zoho-crud-lead skill. Built to run side-by-side with
  bdr-signal-hiring-risk (Apollo) for a data-provider comparison.
  Use when asked to find companies hiring for risk, compliance, regulatory, or financial crime
  roles via ZoomInfo, run the ZoomInfo risk hiring signal scan, or compare ZoomInfo vs Apollo on
  risk hiring signals.
  Triggers on: "run zoominfo risk hiring signal scan", "which ICP companies are hiring for risk
  per zoominfo", "zoominfo risk hiring signal", "zoominfo who's hiring AML/KYC",
  "bdr-signal-hiring-risk-zoominfo".
---

# Eclipse BDR — Signal: Company Hiring (Risk & Regulatory) — ZoomInfo

Surface ICP-fit companies with active **Risk & Regulatory** hiring — regulatory change, financial
crime / AML / KYC, operational risk, audit, controls, regulatory reporting, model risk — using
**ZoomInfo** as the data provider, then find ICP contacts at those companies. Output is JSON only —
hand off to `zoho-crud-lead` to write to Inbound (Leads).

This is the ZoomInfo twin of `bdr-signal-hiring-risk` (Apollo), and the Risk & Regulatory sibling
of `bdr-signal-hiring-data-zoominfo`. Same criteria page, same scoring rubric, same output shape —
only the data-provider layer differs, so the two can be run on the same criteria and diffed.

---

## ⚠️ Criteria Reference Page

This is the **same** reference page the Apollo risk-hiring skill uses — targeting criteria are
provider-agnostic. Always fetch it fresh — never use cached values.

**Reference (Notion) — Skill References - Signal: Company Hiring (Risk):**
`https://app.notion.com/p/6ce4e4751c3a8331828b818f9efe37b4`
(in the **Skill References** database)

Fetch fresh via the Notion MCP `notion-fetch` tool on every run.

Provides:
- **Section A — ICP Company & Contact Criteria:** `titles`, `seniority`, `industries`,
  `company_size`, `geography`, `include_similar_titles`
- **Section B — Risk Role Keywords:** `risk_role_keywords` (job title keywords to match against
  hiring scoops), optional financial-risk keyword list (only if enabled on the page), and
  `min_postings`

If the reference page is unreachable or empty, stop and tell the user. Do not proceed with
guessed or hardcoded criteria or keywords.

---

## 💳 Credits Note

`search_companies`, `search_scoops`, and `search_contacts` count toward ZoomInfo request/record
limits but are not bulk-credit enrichments. Like the Apollo version, this skill leaves `Email`
and `Phone` **null** — it does not call `enrich_contacts` (which would burn bulk credits). If the
team wants emails/phones populated, that's a separate, credit-consuming enrichment pass to add
later. Flag this in the run summary so nobody assumes contactability data is present.

---

## Step 1: Fetch Criteria

**From Section A**, extract: `titles`, `seniority`, `industries`, `company_size`, `geography`,
`include_similar_titles`.

**From Section B**, extract:
- `risk_role_keywords` — keywords to match against hiring-scoop descriptions / job titles
  (case-insensitive substring match). Include the optional financial-risk list only if the page
  says it is enabled.
- `min_postings` — minimum number of matching hiring scoops to qualify a company as a "hiring
  signal" (default: `1` if not specified)

---

## Step 1b: Resolve ZoomInfo Lookup Values

ZoomInfo search fails on free-text filter values — it needs canonical IDs. Before searching,
call `lookup` to translate Section A criteria into ZoomInfo values:

| Section A field | `lookup` fieldName | Used in |
|---|---|---|
| `industries` | `industries` (use `fuzzyMatch`) | `industryCodes` |
| `geography` | `metro-regions` or `states` (use `fuzzyMatch`) | `metroRegion` / `state` |
| `company_size` | `employee-count` | `employeeCount` |

Use the returned `id` values (not `attributes.name`) in the searches below.

---

## Step 2: Find ICP Companies Hiring for Risk Roles

ZoomInfo has **no job-postings endpoint**. Its native hiring surface is **Scoops** — specifically
the `Open Position` and `Hiring Plans` scoop types. Use `search_scoops`, which accepts the ICP
company filters and the hiring filters in one call:

| Source | `search_scoops` parameter |
|---|---|
| `scoopTypes` | `["Open Position", "Hiring Plans"]` |
| `industryCodes` (resolved in 1b) | `industryCodes` |
| `employeeCount` (resolved in 1b) | `employeeCount` |
| `geography` (resolved in 1b) | `metroRegions` / `state` |
| `risk_role_keywords` (Section B) | `description` (space-separated keywords) and/or `jobTitle` |
| department | `department: ["Finance", "Legal", "Operations"]` |

Fixed parameters:
- `pageSize`: 25, `sort`: `-originalPublishedDate` (freshest hiring activity first)

> Note on `department`: ZoomInfo's scoop departments don't carry a dedicated "Risk/Compliance"
> value — risk, compliance, and financial-crime roles scatter across Finance, Legal, and
> Operations. Treat `department` as a soft hint only; **keyword matching against
> `risk_role_keywords` is the primary filter.** If the department filter suppresses obvious
> matches, drop it and rely on keywords alone.

A scoop **matches** if its description / job title contains any keyword from `risk_role_keywords`
(case-insensitive substring). Group matching scoops by company. Keep only companies where the
count of matching scoops ≥ `min_postings`. These are your **hiring signal companies**.

> Alternative (higher precision, higher cost): for a known target-account list, call
> `enrich_scoops` per company with the same `scoopTypes` + `description` filters instead of a
> broad `search_scoops`. Note it consumes a credit per company.

Capture each hiring signal company's `zoominfoCompanyId` for Step 3.

---

## Step 3: Find ICP Contacts at Hiring Signal Companies

For each hiring signal company, call `search_contacts`:

| Section A field | `search_contacts` parameter |
|---|---|
| company | `companyId` (the `zoominfoCompanyId` from Step 2) |
| `titles` | `jobTitle` (OR-joined) if `include_similar_titles` is true; else `exactJobTitle` |
| `seniority` → management level | `managementLevel` (e.g. `"C Level Exec,VP Level Exec,Director"`) |

Fixed parameters:
- `pageSize`: 10 per company (surface the best matches, not an exhaustive list)
- `sort`: `-contactAccuracyScore` (most reachable first)

Capture per contact: `personId`, `firstName`, `lastName`, `jobTitle`, `managementLevel`,
`companyName`, `externalUrls` (LinkedIn), `contactAccuracyScore`. Leave email/phone null (see
Credits Note).

---

## Step 4: Score Results

For each person returned, score Eclipse relevance 1–10. **Identical rubric to the Apollo version**
so scores are directly comparable.

| Condition | Points |
|---|---|
| Company is a bank or credit union | +3 |
| Company is an asset/investment manager or PE firm | +2 |
| Company is a fintech, payments, or insurance firm | +1 |
| Person is C-suite (CRO, CCO, CDO, CIO, CTO) | +3 |
| Person is VP-level | +2 |
| Person is Director-level | +1 |
| Company has ≥ 2 matching hiring scoops (program build, not a single backfill) | +1 |
| Company has 1,000–10,000 employees | +1 |
| No company size data available | -1 |

**ICP match:** `true` if score ≥ 5.

**ICP role match:**
- C-suite risk/compliance title → `"Primary ICP"`
- VP or Director risk/compliance title → `"Secondary ICP"`

---

## Step 5: Build CRM Payload

For every result with `icp_match: true`, build one lead record for the Inbound (Leads) module.

Field mapping:

| Generic | Zoho API Name | Notes |
|---|---|---|
| `first_name` | `First_Name` | |
| `last_name` | `Last_Name` | |
| `title` | `Designation` | |
| `company` | `Company` | |
| `email` | `Email` | Null — not enriched (see Credits Note) |
| `phone` | `Phone` | Null — not enriched (see Credits Note) |
| `linkedin_url` | `LinkedIn_Page_URL` | From `externalUrls` |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | `["Risk & Regulatory"]` — additionally include `"Financial Crimes & Compliance"` when the company's matched scoops include financial crime keywords (AML, BSA, KYC, Sanctions, Financial Crime, Transaction Monitoring) |
| `lead_source` | `Lead_Source` | Always `"ZoomInfo ICP Scan"` ⚡ (value pending team addition) |
| `zoominfo_contact_id` | `ZoomInfo_Contact_ID_c` | ⚡ Field pending team creation (ZoomInfo `personId`) |
| `signal_type` | `Signal_Type_c` | Always `"Company Hiring"` ⚡ Field pending team creation |
| `signal_date` | `Signal_Date_c` | Today's date ⚡ Field pending team creation |
| `signal_summary` | `Signal_Summary_c` | Generated in Step 6 ⚡ Field pending team creation |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will skip ⚡
fields gracefully until they exist in Zoho.

**Dedup field:** `ZoomInfo_Contact_ID_c`. Dedup fallback: `First_Name + Last_Name + Company`.

> ⚠️ **Integration touchpoint:** `zoho-crud-lead` currently keys dedup on `Apollo_Contact_ID_c`.
> This payload uses `ZoomInfo_Contact_ID_c`. The CRM writer must be taught the new dedup key (or
> dedup on the name+company fallback) before these leads write cleanly alongside Apollo-sourced
> ones. Surface this in the run summary; do not silently assume it round-trips.

---

## Step 6: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max):

> "[Company] is actively hiring for Risk & Regulatory roles ([list matched scoop titles, max 3]).
> [First name] is [title] there — surfaced via ZoomInfo as a Risk & Regulatory signal on [date]."

If the matched scoops are predominantly financial crime roles (AML, BSA, KYC, Sanctions,
Financial Crime, Transaction Monitoring), note it:

> "[Company] is staffing a financial crime compliance program ([list matched scoop titles, max 3]).
> [First name] is [title] there — surfaced via ZoomInfo as a Risk & Regulatory signal on [date]."

---

## Step 7: Return JSON

Output **only** this JSON — no prose before or after.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-hiring-risk-zoominfo",
  "signal_source": "ZoomInfo",
  "criteria_fetched_from": {
    "reference_page": "<reference page URL/URI>"
  },
  "summary": {
    "companies_scanned": 0,
    "companies_with_hiring_signal": 0,
    "total_contacts_found": 0,
    "icp_matched": 0,
    "primary_icp": 0,
    "secondary_icp": 0,
    "email_phone_enriched": false,
    "zoho_crud_dedup_note": "payload keys on ZoomInfo_Contact_ID_c — zoho-crud-lead must support it"
  },
  "hiring_signal_companies": [
    {
      "company_name": "<name>",
      "zoominfo_company_id": "<id>",
      "domain": "<domain>",
      "matched_scoops": ["<title1>", "<title2>"]
    }
  ],
  "crm_payload": {
    "generated_by": "bdr-signal-hiring-risk-zoominfo",
    "generated_at": "<today YYYY-MM-DD>",
    "module": "Leads",
    "action": "upsert",
    "dedup_field": "ZoomInfo_Contact_ID_c",
    "dedup_fallback": "First_Name + Last_Name + Company",
    "leads": [
      {
        "id": "lead_001",
        "fields": {
          "First_Name": "<first>",
          "Last_Name": "<last>",
          "Designation": "<title>",
          "Company": "<company>",
          "Email": null,
          "Phone": null,
          "LinkedIn_Page_URL": "<url or null>",
          "No_of_Employees": 0,
          "Industry": "<mapped value>",
          "Practice_Areas_In_Scope": ["Risk & Regulatory"],
          "Lead_Source": "ZoomInfo ICP Scan",
          "ZoomInfo_Contact_ID_c": "<zoominfo person id>",
          "Signal_Type_c": "Company Hiring",
          "Signal_Date_c": "<today YYYY-MM-DD>",
          "Signal_Summary_c": "<generated summary>"
        },
        "meta": {
          "icp_role_match": "Primary ICP | Secondary ICP",
          "relevance_score": 0,
          "contact_accuracy_score": 0,
          "hiring_signal_company": "<company name>",
          "matched_scoops": ["<title1>", "<title2>"]
        }
      }
    ]
  }
}
```

- IDs sequential: `lead_001`, `lead_002`, etc.
- `meta` block is for human review only — not written to Zoho
- If no hiring signals found: return empty `leads: []` and `hiring_signal_companies: []`

---

## ZoomInfo vs Apollo — what to compare

When run against the same criteria as `bdr-signal-hiring-risk`, compare:
- **Company coverage** — does ZoomInfo surface risk/compliance hiring at banks and insurers the
  Apollo postings endpoint misses? (Per Nico's research, ZoomInfo is expected to win on
  financial-industry and large-enterprise coverage — the core market for this signal.)
- **Signal mechanism** — Apollo reads live job postings; ZoomInfo reads research-curated Scoops
  (`Open Position` / `Hiring Plans`). Scoops may lag live postings but carry ZoomInfo's
  human-verified context.
- **Financial-crime precision** — compare how cleanly each provider isolates AML/KYC/sanctions
  hiring (the highest-value sub-signal) from general risk roles.
- **Contact match + accuracy** — ZoomInfo returns a `contactAccuracyScore` per contact; Apollo
  masks some last names. Compare fill quality.

---

## Eclipse Context

**Practice area:** Risk & Regulatory Change (including Financial Crime Compliance).

**Signal:** A company actively hiring for risk, compliance, regulatory change, or financial crime
roles is responding to pressure — regulatory expectations, audit findings, remediation timelines,
or a funded program build-out. Per the ICP & Buyer Persona Analysis, strong-fit indicators include
active regulatory/audit/compliance pressure, large-scale risk or compliance change initiatives,
AML/KYC modernization, and manual controls, testing, or reporting processes.

**Eclipse wedge:** Regulatory remediation planning, financial crime compliance support (AML, KYC,
transaction monitoring), controls design & testing, risk data & reporting alignment, audit
readiness & issue remediation, and operating model design for risk & compliance — plus financial
risk (credit, liquidity, market, valuation) and non-financial risk (operational, technology,
third-party, model risk) offerings.

**Relationship to siblings:** Same detection mechanism as `bdr-signal-hiring-data-zoominfo`,
different practice. Same output shape and `zoho-crud-lead` handoff as the Apollo
`bdr-signal-hiring-risk`; each reads its own client-managed Notion reference page.
