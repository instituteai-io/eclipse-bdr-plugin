---
name: bdr-signal-hiring-data-zoominfo
description: >
  ZoomInfo variant of the Data Hiring signal. Detects ICP-fit companies actively hiring for
  Data roles — broad data leadership/engineering roles (Chief Data Officer, Data Engineer,
  Data Architect) through governance operating-model roles (Data Governance Lead, Data Steward,
  Metadata Manager, Data Quality Lead, Data Catalog, Master Data). Fetches the same company ICP
  criteria and hiring keyword list from the team's Notion reference page that the Apollo
  version uses, finds ICP companies in ZoomInfo, detects hiring via ZoomInfo Scoops (Open
  Position / Hiring Plans scoop types) instead of a job-postings endpoint, captures the scoop's
  publish date so signal freshness is known, surfaces the ICP decision-maker(s) at those
  companies, then enriches the qualified leads for real contact info. Output is a JSON payload
  ready for the zoho-crud-lead skill. Built to run side-by-side with bdr-signal-hiring-data
  (Apollo) for a data-provider comparison.
  Use when asked to find companies hiring for data roles via ZoomInfo, run the ZoomInfo hiring
  signal scan, or compare ZoomInfo vs Apollo on hiring signals.
  Triggers on: "run zoominfo hiring signal scan", "which ICP companies are hiring for data per
  zoominfo", "zoominfo data hiring signal", "bdr-signal-hiring-data-zoominfo".
---

# Eclipse BDR — Signal: Company Hiring (Data) — ZoomInfo

Surface ICP-fit companies with active Data hiring — leadership, engineering, and governance
operating-model roles alike — using **ZoomInfo** as the data provider, then find the ICP
decision-maker at those companies and enrich them for real contact info. Output is JSON only —
hand off to `zoho-crud-lead` to write to Inbound (Leads).

This is the ZoomInfo twin of `bdr-signal-hiring-data` (Apollo). Same criteria page, same scoring
rubric, same output shape — only the data-provider layer differs, so the two can be run on the
same criteria and diffed.

Data management and data governance hiring are **one consolidated signal**. The precision
distinction lives in the keyword list, which the client manages in Notion (Section B of the
reference page) — not in separate skills and not hardcoded here.

---

## ⚠️ Criteria Reference Page

This is the **same** reference page the Apollo data-hiring skill uses — targeting criteria are
provider-agnostic. Always fetch it fresh — never use cached values.

**Reference (Notion) — Skill References - Signal: Company Hiring (Data):**
`https://app.notion.com/p/23e4e4751c3a827480e10109d6538c63`
(in the **Skill References** database)

Fetch fresh via the Notion MCP `notion-fetch` tool on every run.

Provides:
- **Section A — ICP Company & Contact Criteria:** `titles`, `seniority`, `industries`,
  `company_size`, `geography`, `include_similar_titles`
- **Section B — Data Role Keywords:** `data_role_keywords` — the full list of job title keywords
  to match against hiring scoops (covers both broad data roles and governance operating-model
  roles), and `min_postings`

If the reference page is unreachable or empty, stop and tell the user. Do not proceed with
guessed or hardcoded criteria or keywords.

---

## 💳 Credits Note

`search_companies`, `search_scoops`, and `search_contacts` count toward ZoomInfo request/record
limits but are **not** bulk-credit enrichments — the scan is cheap. This skill **does** run one
credit-consuming enrichment pass, but **only on the qualified leads** (score ≥ 5) in Step 5, to
reveal email/phone. It never enriches the full scanned set. Report the number of records enriched
in the run summary so credit usage is visible. (If the team wants a strictly no-cost dry run,
Step 5 can be skipped — say so in the summary and leave email/phone null.)

---

## Step 1: Fetch Criteria

**From Section A**, extract: `titles`, `seniority`, `industries`, `company_size`, `geography`,
`include_similar_titles`.

**From Section B**, extract:
- `data_role_keywords` — keywords to match against hiring-scoop descriptions / job titles
  (case-insensitive substring match)
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

⚠️ **Industry mapping is lossy.** ZoomInfo's taxonomy does not have distinct "Asset Management"
or "Capital Markets" codes — they collapse into `finance` / `finance.investment`. If `fuzzyMatch`
returns nothing for a Section A industry term, pull the full industry list and map manually. Known
finance/insurance codes: `finance`, `finance.banking`, `finance.investment` (Investment Banking),
`finance.brokerage` (Lending & Brokerage), `finance.venturecapital` (VC & Private Equity),
`insurance`.

---

## Step 2: Find ICP Companies Hiring for Data Roles

ZoomInfo has **no job-postings endpoint**. Its native hiring surface is **Scoops** — specifically
the `Open Position` and `Hiring Plans` scoop types. Use `search_scoops`, which accepts the ICP
company filters and the hiring filters in one call:

| Source | `search_scoops` parameter |
|---|---|
| `scoopTypes` | `["Open Position", "Hiring Plans"]` |
| `industryCodes` (resolved in 1b) | `industryCodes` |
| `employeeCount` (resolved in 1b) | `employeeCount` |
| `geography` (resolved in 1b) | `metroRegions` / `state` |
| anchor keyword (see below) | `description` |

Fixed parameters:
- `pageSize`: 25, `sort`: `-originalPublishedDate` (freshest hiring activity first)

### ⚠️ Keyword matching — do NOT AND the keyword list

ZoomInfo's `description` filter treats multiple space-separated tokens as a **conjunction /
phrase** (it requires them to co-occur), **not** as an OR across keywords. Passing the whole
`data_role_keywords` list into `description` therefore returns ~zero results, because no single
scoop contains every keyword at once. The matching rule we want is "**a scoop matches if it
contains ANY one keyword**" — so do the OR ourselves, not via the API:

- **Preferred (one cheap query):** pass a single broad anchor token — `description: "Data"` —
  then apply the `data_role_keywords` list as a **case-insensitive substring filter in code**
  against each returned scoop's job title / description. This is the OR match, done client-side.
- **Alternative (higher recall, more requests):** issue one `search_scoops` per keyword (or per
  small keyword group) and **union** the results, deduping by scoop id.

Do **not** put the full keyword list in `description`, and do **not** also constrain by
`department` — combining the two over-filters to nothing.

> ❌ `jobTitle` is **not** a valid parameter on `search_scoops` (it errors). `jobTitle` is valid
> only on `search_contacts` (Step 3). Match against the scoop's returned title/description text
> in code instead.

A scoop **matches** if its description / job title contains any keyword from `data_role_keywords`
(case-insensitive substring). Group matching scoops by company. Keep only companies where the
count of matching scoops ≥ `min_postings`. These are your **hiring signal companies**.

**Capture the signal mechanism and freshness for every matched scoop** (the team wants to know
how old each signal is — fresher scoops are higher-intent):
- `signal_mechanism`: `"ZoomInfo Scoop — Open Position"` or `"ZoomInfo Scoop — Hiring Plans"`
  (use the scoop's own `scoopType`).
- `published_date`: the scoop's `originalPublishedDate` (YYYY-MM-DD), or `null` if absent.
- `age_days`: whole days between `published_date` and the scan date, or `null`.

For each hiring signal company, record the **freshest** matched scoop date as `latest_signal_date`
and its `signal_age_days`. Capture each company's `zoominfoCompanyId` for Step 3.

> Note on `Hiring Plans` scoops: these are department-trend summaries (skills/technologies, not
> specific titles), so they rarely substring-match a role keyword and usually contribute little.
> Most real matches come from `Open Position` scoops, which carry concrete job titles.

> Alternative (higher precision, higher cost): for a known target-account list, call
> `enrich_scoops` per company with the same `scoopTypes` filter instead of a broad
> `search_scoops`. Note it consumes a credit per company.

---

## Step 3: Find ICP Contacts at Hiring Signal Companies

The hiring activity is a **company-level** signal. This step finds the **person Eclipse would
actually pitch** — the data decision-maker / leader at the signal company (e.g. the CDO, VP of
Data, Head of Data Governance). This is **not** the recruiter or whoever the scoop is about; it
is the ICP buyer whose team the new headcount rolls up into.

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
`companyName`, `externalUrls` (LinkedIn), `contactAccuracyScore`. Email/phone come from Step 5.

---

## Step 4: Score Results

For each person returned, score Eclipse relevance 1–10. **Identical rubric to the Apollo version**
so scores are directly comparable.

| Condition | Points |
|---|---|
| Company is a bank or credit union | +3 |
| Company is an asset/investment manager or PE firm | +2 |
| Company is a fintech, payments, or insurance firm | +1 |
| Person is C-suite (CDO, CAO, CDAO, CIO, CTO) | +3 |
| Person is VP-level | +2 |
| Person is Director-level | +1 |
| Company has ≥ 2 matching hiring scoops (program build, not a single backfill) | +1 |
| Company has 1,000–10,000 employees | +1 |
| No company size data available | -1 |

**ICP match:** `true` if score ≥ 5.

**ICP role match:**
- C-suite data title → `"Primary ICP"`
- VP or Director data title → `"Secondary ICP"`

---

## Step 5: Enrich Qualified Leads 💳

Only **after** scoring, enrich the contacts with `icp_match: true` (score ≥ 5) to reveal real
contact data. **Enrich qualified leads only** — never the full scanned set — to keep credit
spend tight.

Call `enrich_contacts` with the qualified contacts' `personId`s (one call for all of them),
requesting email and phone output fields.

⚠️ **Credit cost:** `enrich_contacts` is a bulk-credit enrichment — it consumes ≈ 1 bulk credit
per record. This is the only credit-consuming step in the skill. Always report the number of
records enriched in the run summary. The contact records from Step 3 typically show
`hasEmail: true` / `hasMobilePhone: true`, so the data exists; this step is what actually unlocks
it.

From each enriched record, populate `email` and `phone` (names are already full from Step 3). If a
specific reveal fails, leave that field `null` and note it — do not re-enrich.

If the team explicitly wants a no-cost dry run, this step may be skipped; email/phone stay `null`.
State clearly in the summary that enrichment was skipped.

---

## Step 6: Build CRM Payload

For every result with `icp_match: true`, build one lead record for the Inbound (Leads) module.

Field mapping:

| Generic | Zoho API Name | Notes |
|---|---|---|
| `first_name` | `First_Name` | |
| `last_name` | `Last_Name` | Full from `search_contacts` |
| `title` | `Designation` | |
| `company` | `Company` | |
| `email` | `Email` | From Step 5 enrichment; null if none on file |
| `phone` | `Phone` | From Step 5 enrichment; null if none on file |
| `linkedin_url` | `LinkedIn_Page_URL` | From `externalUrls` |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | Always `["Data Management & Analytics"]` |
| `lead_source` | `Lead_Source` | Always `"ZoomInfo ICP Scan"` ⚡ (value pending team addition) |
| `zoominfo_contact_id` | `ZoomInfo_Contact_ID_c` | ⚡ Field pending team creation (ZoomInfo `personId`). NOTE: can be a **negative** integer — Zoho field must be text/long and tolerate the sign |
| `signal_type` | `Signal_Type_c` | Always `"Company Hiring"` ⚡ Field pending team creation |
| `signal_mechanism` | `Signal_Mechanism_c` | `"ZoomInfo Scoop — Open Position"` / `"… Hiring Plans"` ⚡ Field pending team creation |
| `signal_date` | `Signal_Date_c` | Date WE detected the signal = scan date ⚡ Field pending team creation |
| `signal_source_date` | `Signal_Source_Date_c` | The scoop's own publish date (`latest_signal_date`); how old the signal actually is ⚡ Field pending team creation |
| `signal_summary` | `Signal_Summary_c` | Generated in Step 7 ⚡ Field pending team creation |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will skip ⚡
fields gracefully until they exist in Zoho.

**Dedup field:** `ZoomInfo_Contact_ID_c`. Dedup fallback: `First_Name + Last_Name + Company`.

> ⚠️ **Integration touchpoint:** `zoho-crud-lead` currently keys dedup on `Apollo_Contact_ID_c`.
> This payload uses `ZoomInfo_Contact_ID_c`. The CRM writer must be taught the new dedup key (or
> dedup on the name+company fallback) before these leads write cleanly alongside Apollo-sourced
> ones. Surface this in the run summary; do not silently assume it round-trips.

---

## Step 7: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max). Include the signal's age so the
reader knows how fresh it is:

> "[Company] is actively hiring for Data roles ([list matched scoop titles, max 3]; latest scoop
> published [signal_age_days] days ago). [First name] is [title] there — surfaced via ZoomInfo as a
> Data Management & Analytics signal on [scan date]."

If the matched scoops are predominantly governance operating-model roles (e.g. Data Governance,
Data Steward, Metadata, Data Catalog), note it:

> "[Company] is staffing a data governance program ([list matched scoop titles, max 3]; latest
> scoop published [signal_age_days] days ago). [First name] is [title] there — surfaced via
> ZoomInfo as a Data Management & Analytics signal on [scan date]."

If no scoop date was available, drop the "latest scoop published …" clause rather than guessing.

---

## Step 8: Return JSON

Output **only** this JSON — no prose before or after.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-hiring-data-zoominfo",
  "signal_source": "ZoomInfo",
  "signal_mechanism": "ZoomInfo Scoop (Open Position / Hiring Plans)",
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
    "leads_enriched": 0,
    "emails_found": 0,
    "phones_found": 0,
    "enrichment_credits_estimated": 0,
    "enrichment_skipped": false,
    "zoho_crud_dedup_note": "payload keys on ZoomInfo_Contact_ID_c — zoho-crud-lead must support it"
  },
  "hiring_signal_companies": [
    {
      "company_name": "<name>",
      "zoominfo_company_id": "<id>",
      "domain": "<domain>",
      "latest_signal_date": "<YYYY-MM-DD or null>",
      "signal_age_days": 0,
      "matched_scoops": [
        {"title": "<title1>", "scoop_type": "Open Position", "published_date": "<YYYY-MM-DD or null>", "age_days": 0}
      ]
    }
  ],
  "crm_payload": {
    "generated_by": "bdr-signal-hiring-data-zoominfo",
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
          "Email": "<email or null>",
          "Phone": "<phone or null>",
          "LinkedIn_Page_URL": "<url or null>",
          "No_of_Employees": 0,
          "Industry": "<mapped value>",
          "Practice_Areas_In_Scope": ["Data Management & Analytics"],
          "Lead_Source": "ZoomInfo ICP Scan",
          "ZoomInfo_Contact_ID_c": "<zoominfo person id>",
          "Signal_Type_c": "Company Hiring",
          "Signal_Mechanism_c": "ZoomInfo Scoop — Open Position",
          "Signal_Date_c": "<today YYYY-MM-DD>",
          "Signal_Source_Date_c": "<latest_signal_date or null>",
          "Signal_Summary_c": "<generated summary>"
        },
        "meta": {
          "icp_role_match": "Primary ICP | Secondary ICP",
          "relevance_score": 0,
          "contact_accuracy_score": 0,
          "enriched": true,
          "hiring_signal_company": "<company name>",
          "signal_mechanism": "ZoomInfo Scoop — Open Position",
          "latest_signal_date": "<YYYY-MM-DD or null>",
          "signal_age_days": 0,
          "matched_scoops": [
            {"title": "<title1>", "scoop_type": "Open Position", "published_date": "<YYYY-MM-DD or null>", "age_days": 0}
          ]
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

When run against the same criteria as `bdr-signal-hiring-data`, compare:
- **Company coverage** — does ZoomInfo surface hiring at banks/financial-services firms the
  Apollo postings endpoint misses? (Per Nico's research, ZoomInfo is expected to win on
  financial-industry and large-enterprise coverage.)
- **Signal mechanism + freshness** — Apollo reads live job postings (with a posting date);
  ZoomInfo reads research-curated Scoops (`Open Position` / `Hiring Plans`, with an
  `originalPublishedDate`). Both now carry the source date so you can compare how stale each
  provider's signal is. Scoops may lag live postings but carry ZoomInfo's human-verified context.
- **Contact match + accuracy** — ZoomInfo returns a `contactAccuracyScore` per contact and full
  last names; Apollo masks last names until enrichment. Compare fill quality after Step 5.

---

## Eclipse Context

**Practice area:** Data Management & Analytics (data governance is part of this practice — the
terms are used interchangeably in this signal).

**Signal:** A company actively hiring for data roles is building or rebuilding a data program —
high-intent signal that they may need external advisory support for governance frameworks, data
quality, or strategy. Governance operating-model postings (Data Governance Lead, Data Steward,
Metadata Manager, Data Quality Lead) are the strongest form of this signal: per the Signal
Library, they show the program is being **operationalized**, not just discussed. Hidden
hypothesis: the company likely bought or is implementing a tool but lacks the people/process
model to scale it. **Signal freshness matters** — a scoop published this week is a hotter trigger
than one from three months ago, so the scoop's publish date is captured and carried through to the
lead.

**Who we surface:** the data **decision-maker** at the signal company (CDO / VP Data / Head of
Data Governance) — the buyer for the engagement — not the person the scoop is about.

**Eclipse wedge:** Governance frameworks, role design, stewardship playbooks, operating-model
support, data quality programs, and strategy — turning new data headcount into a working program.
</content>
