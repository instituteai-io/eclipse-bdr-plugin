---
name: signal-company-hiring-icp-data
description: >
  Detect ICP-fit companies that are actively hiring for Data roles. Fetches company ICP
  criteria and target job role keywords from Notion, queries Apollo for companies matching
  ICP profile, checks their active job postings for data-related openings, then surfaces
  ICP-matching contacts at those companies as leads for the Inbound (Leads) module.
  Output is a JSON payload ready for the zoho-crud-lead skill.
  Use when asked to find companies hiring for data roles, run the hiring signal scan,
  detect data hiring activity at ICP companies, or surface data talent demand signals.
  Triggers on: "run hiring signal scan", "which ICP companies are hiring for data",
  "data hiring signal", "signal-company-hiring-icp-data".
---

# Eclipse BDR — Signal: Company Hiring (Data Practice)

Surface ICP-fit companies with active Data job postings, then find ICP contacts at those
companies. Output is JSON only — hand off to zoho-crud-lead to write to Inbound (Leads).

---

## ⚠️ Criteria Reference Pages

Two SharePoint files drive this skill. Always fetch both fresh via `read_resource` — never use cached values.

**Page A — ICP Company & Contact Criteria:**
`file:///b!ZEjP7U2_7UCmi5bHlWbQqhD0IvI686dAqFFpekklCTTT7yqCdusrTarAzUvLNKPW/015GTRKZBD3MJWJGBVSZA3L4E6BCGSMWQM`

Provides: `titles`, `seniority`, `industries`, `company_size`, `geography`, `include_similar_titles`

**Page B — Hiring Signal Keywords:**
`file:///b!ZEjP7U2_7UCmi5bHlWbQqhD0IvI686dAqFFpekklCTTT7yqCdusrTarAzUvLNKPW/015GTRKZBF6MTZJ7ZO2BHZRVS6DJNDQ4BO`

Provides: `data_role_keywords` — list of job title keywords to match against open postings
(e.g. "Data Governance", "Data Engineer", "Chief Data", "Data Quality", "Data Architecture")

If either file is unreachable or empty, stop and tell the user. Do not proceed with
guessed or hardcoded keywords.

---

## Step 1: Fetch Criteria

**From Page A**, extract:
- `titles` — ICP contact titles to find at hiring companies
- `seniority` — seniority levels
- `industries` — industry filters
- `company_size` — employee range
- `geography` — locations
- `include_similar_titles`

**From Page B**, extract:
- `data_role_keywords` — keywords to match against job posting titles (e.g. `["Data Governance", "Data Engineer", "Chief Data Officer", "Data Quality", "Data Architecture", "Head of Data"]`)
- `min_postings` — minimum number of matching postings to qualify a company as a "hiring signal" (default: `1` if not specified)

---

## Step 2: Find ICP Companies with Data Job Postings

Call `apollo_mixed_companies_search` using company-level criteria from Page A:

| Criteria field | Apollo parameter |
|---|---|
| `industries` | `q_organization_keyword_tags` |
| `company_size` | `organization_num_employees_ranges` |
| `geography` | `organization_locations` |

Fixed parameters:
- `per_page`: 25

For each company returned, call `apollo_organizations_job_postings` with the company domain.

Filter the returned job postings: a posting **matches** if its title contains any keyword
from `data_role_keywords` (case-insensitive substring match).

Keep only companies where the count of matching postings ≥ `min_postings`.

These are your **hiring signal companies**.

---

## Step 3: Find ICP Contacts at Hiring Signal Companies

For each hiring signal company, call `apollo_mixed_people_api_search` to find contacts:

| Criteria field | Apollo parameter |
|---|---|
| `titles` (from Page A) | `person_titles` |
| `seniority` (from Page A) | `person_seniorities` |
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
| Person is C-suite (CDO, CAO, CDAO, CIO, CTO) | +3 |
| Person is VP-level | +2 |
| Person is Director-level | +1 |
| Company has 1,000–10,000 employees | +1 |
| No company size data available | -1 |

**ICP match:** `true` if score ≥ 5.

**ICP role match:**
- C-suite data title → `"Primary ICP"`
- VP or Director data title → `"Secondary ICP"`

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
| `practice_areas` | `Practice_Areas_In_Scope` | Always `["Data Management & Analytics"]` |
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

> "[Company] is actively hiring for Data roles ([list matched posting titles, max 3]).
> [First name] is [title] there — surfaced as a Data Management & Analytics signal on [date]."

---

## Step 7: Return JSON

Output **only** this JSON — no prose before or after.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "signal-company-hiring-icp-data",
  "signal_source": "Apollo",
  "criteria_fetched_from": {
    "icp_criteria": "<Page A URL>",
    "hiring_keywords": "<Page B URL>"
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
    "generated_by": "signal-company-hiring-icp-data",
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
          "Practice_Areas_In_Scope": ["Data Management & Analytics"],
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

**Practice area:** Data Management (including Analytics)

**Signal:** A company that is actively hiring CDOs, Data Governance leads, or Data Architects
is building or rebuilding a data program — high-intent signal that they may need external
advisory support for governance frameworks, data quality, or strategy.

