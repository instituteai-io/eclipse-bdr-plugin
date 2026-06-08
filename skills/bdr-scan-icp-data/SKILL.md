---
name: bdr-scan-icp-data
description: >
  Scan Apollo for ICP contacts in the Data Management practice area. Fetches live search
  criteria from the team's reference page, queries Apollo for matching people, scores each
  result for Eclipse relevance, and returns a JSON payload ready for the zoho-crud-lead skill.
  Use when asked to run the Data ICP scan, find data governance contacts in Apollo,
  scan for CDO or data leadership signals, or surface Data Management opportunities.
  Triggers on: "run the data ICP scan", "check Apollo for data contacts",
  "find CDOs in Apollo", "data governance signals", "ICP Signal Data".
---

# Eclipse BDR — ICP Signal: Data Management

Scan Apollo for contacts matching Eclipse's Data Management ICP. Output is JSON only.

---

## ⚠️ Criteria Reference Page

**Source (SharePoint):** `file:///b!ZEjP7U2_7UCmi5bHlWbQqhD0IvI686dAqFFpekklCTTT7yqCdusrTarAzUvLNKPW/015GTRKZBD3MJWJGBVSZA3L4E6BCGSMWQM`

Fetch via `read_resource` with the URI above. This is the single source of truth for search
criteria. Always fetch it fresh — never use remembered or cached criteria from a prior run.
If unreachable, stop and tell the user.

---

## Step 1: Fetch Criteria

Fetch the criteria reference page. Extract:

- `titles` — list of job titles
- `seniority` — seniority levels
- `industries` — industries / keyword tags
- `company_size` — employee range
- `geography` — locations
- `include_similar_titles` — true/false

If the page is unreachable or empty, stop and ask the user to provide criteria manually
or check the URL. Do not proceed with guessed or cached criteria.

---

## Step 2: Run Apollo Search

Call `apollo_mixed_people_api_search` using exactly the criteria from Step 1.

| Criteria field | Apollo parameter |
|---|---|
| `titles` | `person_titles` |
| `seniority` | `person_seniorities` (map: C-Suite → `c_suite`, VP → `vp`, Director → `director`) |
| `industries` | `q_organization_keyword_tags` |
| `company_size` | `organization_num_employees_ranges` (e.g. "500,2000") |
| `geography` | `person_locations` |
| `include_similar_titles` | `include_similar_titles` |

Fixed parameters:
- `per_page`: 25 (or as specified by the user)
- `page`: 1

---

## Step 3: Score Results

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

## Step 4: Build CRM Payload

For every result with `icp_match: true`, build one lead record for the Inbound (Leads) module.

Field mapping:

| Generic | Zoho API Name | Notes |
|---|---|---|
| `first_name` | `First_Name` | |
| `last_name` | `Last_Name` | Flag if masked by Apollo |
| `title` | `Designation` | |
| `company` | `Company` | |
| `email` | `Email` | Null if not available — enrichment required |
| `phone` | `Phone` | Null if not available |
| `linkedin_url` | `LinkedIn_Page_URL` | |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | Always `["Data Management & Analytics"]` for this skill |
| `lead_source` | `Lead_Source` | Always `"Apollo ICP Scan"` — ⚡ value pending team addition |
| `apollo_contact_id` | `Apollo_Contact_ID_c` | ⚡ Field pending team creation |
| `signal_type` | `Signal_Type_c` | Always `"New Hire"` — ⚡ Field pending team creation |
| `signal_date` | `Signal_Date_c` | Today's date — ⚡ Field pending team creation |
| `signal_summary` | `Signal_Summary_c` | Generated in Step 5 — ⚡ Field pending team creation |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will
skip ⚡ fields gracefully until they exist in Zoho.

**Dedup field:** `Apollo_Contact_ID_c`. Dedup fallback: `First_Name + Last_Name + Company`.

---

## Step 5: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max):

> "[First name] is [title] at [company], a [company type] with ~[size] employees.
> Surfaced by Apollo ICP scan for the Data Management practice area on [date]."

---

## Step 6: Return JSON

Output **only** this JSON — no prose before or after.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-scan-icp-data",
  "signal_source": "Apollo",
  "criteria_fetched_from": "<URL fetched in Step 1>",
  "summary": {
    "total_returned_by_apollo": 0,
    "icp_matched": 0,
    "primary_icp": 0,
    "secondary_icp": 0,
    "masked_last_names": 0
  },
  "crm_payload": {
    "generated_by": "bdr-scan-icp-data",
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
          "Signal_Type_c": "New Hire",
          "Signal_Date_c": "<today YYYY-MM-DD>",
          "Signal_Summary_c": "<generated summary>"
        },
        "meta": {
          "icp_role_match": "Primary ICP | Secondary ICP",
          "relevance_score": 0
        }
      }
    ]
  }
}
```

- IDs sequential: `lead_001`, `lead_002`, etc.
- `meta` block is for human review only — not written to Zoho
- If Apollo returns no results: return empty `leads: []` with a note in `summary`

---

## Eclipse Context

**Practice area:** Data Management (including Analytics)

**What Eclipse does here:** Data governance frameworks, data quality & controls, data strategy
& operating model, metadata/lineage/cataloging, cloud data migration readiness, analytics
enablement.

**Best signal:** A CDO, CDAO, or Head of Data Governance who recently joined a mid-to-large
financial services firm — new leaders assess programs in the first 90 days.
