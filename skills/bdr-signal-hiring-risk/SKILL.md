---
name: bdr-signal-hiring-risk
description: >
  Detect ICP-fit companies that are actively hiring for Risk & Regulatory roles —
  regulatory change, risk transformation, financial crime / AML / KYC / sanctions,
  operational risk, internal audit, controls testing, regulatory reporting, model risk,
  and similar. Companies staffing risk & compliance functions are responding to
  regulatory pressure, audit findings, remediation timelines, or funded program
  build-outs. The keyword list and ICP criteria are client-managed in the team's
  reference page, never hardcoded here. Queries Apollo for companies matching the ICP
  profile, checks their active job postings for risk-role openings, then surfaces
  ICP-matching contacts (CRO, CCO, Head of Financial Crime, Head of Regulatory Change,
  etc.) at those companies as leads for the Inbound (Leads) module. Output is a JSON
  payload ready for the zoho-crud-lead skill.
  Use when asked to find companies hiring for risk, compliance, regulatory, or financial
  crime roles, run the risk hiring signal scan, detect risk & regulatory hiring activity
  at ICP companies, or find who is building out a risk or compliance team.
  Triggers on: "run risk hiring signal scan", "signal hiring risk", "which ICP companies
  are hiring for risk", "risk hiring signal", "who's hiring AML/KYC", "compliance hiring
  signal", "regulatory hiring scan", "bdr-signal-hiring-risk".
---

# Eclipse BDR — Signal: Company Hiring (Risk & Regulatory)

Surface ICP-fit companies with active **Risk & Regulatory** job postings — regulatory
change, financial crime / AML / KYC, operational risk, audit, controls, regulatory
reporting, model risk — then find ICP contacts at those companies. Output is JSON only —
hand off to `zoho-crud-lead` to write to Inbound (Leads).

This is the Risk & Regulatory sibling of `bdr-signal-hiring-data`. Same mechanism, same
output shape; practice area, ICP criteria, and keyword list differ and are entirely
client-managed in the reference page.

---

## ⚠️ Criteria Reference Page

One reference page drives this skill, carrying both sections. Always fetch it fresh —
never use cached values.

**Reference — Skill References - Signal: Company Hiring (Risk):**
`file:///b!ZEjP7U2_7UCmi5bHlWbQqhD0IvI686dAqFFpekklCTTT7yqCdusrTarAzUvLNKPW/015GTRKZENHSJMWPS43NBIHBXYDMDOEMHE`

Fetch fresh via `read_resource` on every run.

Provides:
- **Section A — ICP Company & Contact Criteria:** `titles`, `seniority`, `industries`,
  `company_size`, `geography`, `include_similar_titles`
- **Section B — Risk Role Keywords:** `risk_role_keywords` (job title keywords to match
  against open postings), optional financial-risk keyword list (only if enabled on the
  page), and `min_postings`

If the reference page is unreachable or empty, stop and tell the user. Do not proceed
with guessed or hardcoded criteria or keywords.

---

## Step 1: Fetch Criteria

**From Section A**, extract:
- `titles` — ICP contact titles to find at hiring companies
- `seniority` — seniority levels
- `industries` — industry filters
- `company_size` — employee range
- `geography` — locations
- `include_similar_titles`

**From Section B**, extract:
- `risk_role_keywords` — keywords to match against job posting titles (case-insensitive
  substring match). Include the optional financial-risk list only if the page says it is
  enabled.
- `min_postings` — minimum number of matching postings to qualify a company as a
  "hiring signal" (default: `1` if not specified)

---

## Step 2: Find ICP Companies with Risk Job Postings

Call `apollo_mixed_companies_search` using company-level criteria from Section A:

| Criteria field | Apollo parameter |
|---|---|
| `industries` | `q_organization_keyword_tags` |
| `company_size` | `organization_num_employees_ranges` |
| `geography` | `organization_locations` |

Fixed parameters:
- `per_page`: 25

For each company returned, call `apollo_organizations_job_postings` with the company domain.

Filter the returned job postings: a posting **matches** if its title contains any keyword
from `risk_role_keywords` (case-insensitive substring match).

Keep only companies where the count of matching postings ≥ `min_postings`.

These are your **hiring signal companies**.

---

## Step 3: Find ICP Contacts at Hiring Signal Companies

For each hiring signal company, call `apollo_mixed_people_api_search` to find contacts:

| Criteria field | Apollo parameter |
|---|---|
| `titles` (from Section A) | `person_titles` |
| `seniority` (from Section A) | `person_seniorities` |
| company domain | `q_organization_domains` |
| `include_similar_titles` | `include_similar_titles` |

Fixed parameters:
- `per_page`: 10 per company (surface the best matches, not an exhaustive list)

---

## Step 4: Score Results

For each person returned, score Eclipse relevance 1–10.

| Condition | Points |
|---|---|
| Company is a bank or credit union | +3 |
| Company is an asset/investment manager or PE firm | +2 |
| Company is a fintech, payments, or insurance firm | +1 |
| Person is C-suite (CRO, CCO, CDO, CIO, CTO) | +3 |
| Person is VP-level | +2 |
| Person is Director-level | +1 |
| Company has ≥ 2 matching postings (program build, not a single backfill) | +1 |
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
| `last_name` | `Last_Name` | Flag if masked by Apollo |
| `title` | `Designation` | |
| `company` | `Company` | |
| `email` | `Email` | Null if not available |
| `phone` | `Phone` | Null if not available |
| `linkedin_url` | `LinkedIn_Page_URL` | |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | `["Risk & Regulatory"]` — additionally include `"Financial Crimes & Compliance"` when the company's matched postings include financial crime keywords (AML, BSA, KYC, Sanctions, Financial Crime, Transaction Monitoring) |
| `lead_source` | `Lead_Source` | Always `"Apollo ICP Scan"` ⚡ (value pending team addition) |
| `apollo_contact_id` | `Apollo_Contact_ID_c` | ⚡ Field pending team creation |
| `signal_type` | `Signal_Type_c` | Always `"Company Hiring"` ⚡ Field pending team creation |
| `signal_date` | `Signal_Date_c` | Today's date ⚡ Field pending team creation |
| `signal_summary` | `Signal_Summary_c` | Generated in Step 6 ⚡ Field pending team creation |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will
skip ⚡ fields gracefully until they exist in Zoho.

**Dedup field:** `Apollo_Contact_ID_c`. Dedup fallback: `First_Name + Last_Name + Company`.

---

## Step 6: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max):

> "[Company] is actively hiring for Risk & Regulatory roles ([list matched posting titles,
> max 3]). [First name] is [title] there — surfaced as a Risk & Regulatory signal on [date]."

If the matched postings are predominantly financial crime roles (AML, BSA, KYC, Sanctions,
Financial Crime, Transaction Monitoring), note it:

> "[Company] is staffing a financial crime compliance program ([list matched posting titles,
> max 3]). [First name] is [title] there — surfaced as a Risk & Regulatory signal on [date]."

---

## Step 7: Return JSON

Output **only** this JSON — no prose before or after.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-hiring-risk",
  "signal_source": "Apollo",
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
    "masked_last_names": 0
  },
  "hiring_signal_companies": [
    {
      "company_name": "<name>",
      "domain": "<domain>",
      "matched_postings": ["<title1>", "<title2>"]
    }
  ],
  "crm_payload": {
    "generated_by": "bdr-signal-hiring-risk",
    "generated_at": "<today YYYY-MM-DD>",
    "module": "Leads",
    "action": "upsert",
    "dedup_field": "Apollo_Contact_ID_c",
    "dedup_fallback": "First_Name + Last_Name + Company",
    "leads": [
      {
        "id": "lead_001",
        "fields": {
          "First_Name": "<first>",
          "Last_Name": "<last — note MASKED if applicable>",
          "Designation": "<title>",
          "Company": "<company>",
          "Email": null,
          "Phone": null,
          "LinkedIn_Page_URL": "<url or null>",
          "No_of_Employees": 0,
          "Industry": "<mapped value>",
          "Practice_Areas_In_Scope": ["Risk & Regulatory"],
          "Lead_Source": "Apollo ICP Scan",
          "Apollo_Contact_ID_c": "<apollo person id>",
          "Signal_Type_c": "Company Hiring",
          "Signal_Date_c": "<today YYYY-MM-DD>",
          "Signal_Summary_c": "<generated summary>"
        },
        "meta": {
          "icp_role_match": "Primary ICP | Secondary ICP",
          "relevance_score": 0,
          "hiring_signal_company": "<company name>",
          "matched_postings": ["<title1>", "<title2>"]
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

## Eclipse Context

**Practice area:** Risk & Regulatory Change (including Financial Crime Compliance).

**Signal:** A company actively hiring for risk, compliance, regulatory change, or financial
crime roles is responding to pressure — regulatory expectations, audit findings, remediation
timelines, or a funded program build-out. Per the ICP & Buyer Persona Analysis (§5.2),
strong-fit indicators include active regulatory/audit/compliance pressure, large-scale risk
or compliance change initiatives, AML/KYC modernization, and manual controls, testing, or
reporting processes.

**Eclipse wedge:** Regulatory remediation planning, financial crime compliance support
(AML, KYC, transaction monitoring), controls design & testing, risk data & reporting
alignment, audit readiness & issue remediation, and operating model design for risk &
compliance — plus financial risk (credit, liquidity, market, valuation) and non-financial
risk (operational, technology, third-party, model risk) offerings.

**Relationship to `bdr-signal-hiring-data`:** Same detection mechanism, different practice.
Both skills share the output shape and the zoho-crud-lead handoff; each reads its own
client-managed reference page.
