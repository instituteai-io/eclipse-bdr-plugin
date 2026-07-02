---
name: bdr-signal-new-exec-apollo
description: >
  Detect a newly appointed executive sponsor at an ICP company (net-new prospect, not an existing CRM
  lead) — a new CDO/CDAO/CRO/CCO/COO/Chief Transformation Officer/Chief AI Officer/CIO or function head.
  A new leader has a mandate and budget and typically reshapes their function in the first 90–180 days,
  making this a top-of-catalog buy trigger (Signal Library SIG-003). Apollo-only: it searches
  apollo_mixed_people_api_search for the sponsor titles at ICP companies, then infers recency from each
  candidate's current-role START DATE (read via enrichment), keeping only those inside the recency
  window. Fetches ICP titles, firmographics, and the recency window fresh from the team's Notion
  reference page, routes each hit to the right Eclipse practice by function, and returns a JSON payload
  ready for the zoho-crud-lead skill.
  CONFIDENCE CAVEAT: Apollo has no native "appointed in the last N days" event, so recency is inferred
  from role start date (noisier, and costs an enrichment credit per candidate). Every hit should be
  confirmed against a press release / leadership page before outreach.
  Use when asked to find newly appointed executives, new CDOs/CROs/etc at ICP companies, run the new
  executive sponsor scan, or find leadership-change signals via Apollo.
  Triggers on: "run the new exec scan", "who just became CDO/CRO at an ICP company", "new executive
  sponsor signal", "leadership change signal via apollo", "bdr-signal-new-exec-apollo".
---

# Eclipse BDR — Signal: New Executive Sponsor (Apollo)

Surface a **newly appointed executive sponsor** at an ICP company — a new CDO / CDAO / CRO / CCO / COO /
Chief Transformation Officer / Chief AI Officer / CIO or function head. A new leader runs a current-state
assessment, sets a 90-day agenda, and brings in outside help to move fast — the prime outreach window
before incumbent advisors entrench. Output is JSON only — hand off to `zoho-crud-lead` to write to
Inbound (Leads).

This is a **person-level** signal (a specific new executive is the lead), so qualified hits are written
to Zoho Leads. It is the net-new counterpart to `bdr-signal-job-change`, which only tracks executives
**already** in the CRM.

> ⚠️ **Confidence caveat — read before running.** Apollo has **no native "appointed in the last N days"
> event**. This skill *infers* recency from each candidate's current-role **start date** (read via
> enrichment) and keeps only those inside the window. That inference is noisier than a true appointment
> event and costs an enrichment credit per candidate checked. **Pair every hit with a quick web/press
> confirmation** (company newsroom / leadership page) before outreach, and run it narrowly (specific
> industries + size) to keep credit spend bounded. This is why the signal is rated medium-confidence on
> the diagnosis.

---

## ⚠️ Criteria Reference Page

One reference page drives this skill. Always fetch it fresh — never use cached values.

**Reference (Notion) — Skill References - Signal: New Executive Sponsor:**
`https://app.notion.com/p/3914e4751c3a81c380e9c32aef69cb53`
(in the **Skill References** database; if the URL fails, locate it with `notion-search` for
`"Signal: New Executive Sponsor"`)

Fetch fresh via the Notion MCP `notion-fetch` tool on every run.

Provides:
- **Section A — ICP Company & Contact Criteria:** the executive sponsor `titles` to detect, `seniority`,
  `industries`, `company_size`, `geography`, `include_similar_titles`
- **Section B — Recency Window & Practice Routing:** `new_role_window_days`, `require_web_confirmation`,
  and the function → `Practice_Areas_In_Scope` routing table

If the reference page is unreachable or empty, stop and tell the user.

---

## Step 1: Fetch Criteria

From Section A extract `titles`, `seniority`, `industries`, `company_size`, `geography`,
`include_similar_titles`. From Section B extract `new_role_window_days` (default `120`),
`require_web_confirmation` (default `true`), and the practice-routing table.

---

## Step 2: Find Candidate Executives at ICP Companies

Call `apollo_mixed_people_api_search`:

| Criteria field | Apollo parameter |
|---|---|
| `titles` (Section A) | `person_titles` |
| `seniority` (Section A) | `person_seniorities` |
| `industries` | `q_organization_keyword_tags` |
| `company_size` | `organization_num_employees_ranges` |
| `geography` | `organization_locations` |
| `include_similar_titles` | `include_similar_titles` |

Fixed parameters:
- `per_page`: 25

These are **candidate** executives — people currently holding a sponsor title at an ICP company. Recency
is not yet known (the people search does not reliably expose role-start date). Last names are masked and
email/phone omitted at this stage — expected.

---

## Step 3: Infer Recency (Enrichment) 💳

For each candidate, call `apollo_people_match` (or `apollo_people_bulk_match`) to read the current-role
**start date** (Apollo returns employment history / current-position start where available), and reveal
contact data.

⚠️ **Credit cost:** ≈ 1 enrichment credit per candidate checked. Because recency can't be pre-filtered,
this is the main credit cost of the skill — **run narrowly** (tight industry + size) and report the
count in the run summary. If the team wants a bounded dry run, cap the number of candidates enriched and
say so.

Compute `role_age_days` = days between the current-role start date and the scan date. **Keep only
candidates with `role_age_days <= new_role_window_days`.** If Apollo returns no start date for a
candidate, set `role_age_days: null` and **exclude** them from the qualified set (do not guess a new
appointment) — list them under `summary.no_start_date` so a human can check manually if desired.

---

## Step 4: Score & Route to Practice

For each executive inside the recency window, score Eclipse relevance 1–10:

| Condition | Points |
|---|---|
| Company is a bank or credit union | +3 |
| Company is an asset/investment manager, insurer, or PE firm | +2 |
| Company is a fintech, payments, or money-services business | +1 |
| Title is a C-suite sponsor (CDO, CDAO, CRO, CCO, COO, Chief Transformation Officer, Chief AI Officer, CIO) | +3 |
| Title is a function head (Head of Data Governance, Head of Financial Crime, Head of Transformation, etc.) | +2 |
| `role_age_days` ≤ 90 (very fresh appointment) | +1 |
| Company has 1,000–10,000 employees | +1 |
| No company size data available | -1 |

**ICP match:** `true` if score ≥ 5.

**Practice routing:** map each executive's function to `Practice_Areas_In_Scope` using the Section B
table (CDO/CDAO/CAO/Head of Data Governance → Data Management & Analytics; CRO/CCO/Financial Crime/
Regulatory Change → Risk & Regulatory Change; COO/Chief Transformation Officer/Head of Transformation →
Transformation; Chief AI Officer → Artificial Intelligence; CIO → route by context).

---

## Step 5: Enrich Contact Data

Enrichment already ran in Step 3 (it doubled as the recency read). From each enriched record populate
full `last_name`, `email`, `phone`, `linkedin_url`, and numeric `employee_count` / `industry` where
Apollo returns them. If a reveal failed, leave that field `null` and note it — do not re-enrich.

---

## Step 6: Build CRM Payload

For every executive with `icp_match: true` **and** `role_age_days <= new_role_window_days`, build one
lead record for the Inbound (Leads) module.

| Generic | Zoho API Name | Notes |
|---|---|---|
| `first_name` | `First_Name` | |
| `last_name` | `Last_Name` | Unmasked from Step 3/5; flag if reveal failed |
| `title` | `Designation` | The new (current) executive title |
| `company` | `Company` | |
| `email` | `Email` | null if none on file |
| `phone` | `Phone` | null if none on file |
| `linkedin_url` | `LinkedIn_Page_URL` | |
| `employee_count` | `No_of_Employees` | |
| `industry` | `Industry` | Map to closest Zoho picklist value |
| `practice_areas` | `Practice_Areas_In_Scope` | Routed by function per Section B |
| `lead_source` | `Lead_Source` | Always `"Apollo ICP Scan"` ⚡ (value pending team addition) |
| `apollo_contact_id` | `Apollo_Contact_ID_c` | ⚡ Field pending team creation |
| `signal_type` | `Signal_Type_c` | Always `"New Executive Sponsor"` ⚡ Field pending team creation |
| `signal_mechanism` | `Signal_Mechanism_c` | Always `"Apollo role-start inference"` ⚡ Field pending team creation |
| `signal_date` | `Signal_Date_c` | Date WE detected the signal = scan date ⚡ Field pending team creation |
| `signal_source_date` | `Signal_Source_Date_c` | The exec's current-role start date; how old the appointment is ⚡ Field pending team creation |
| `signal_summary` | `Signal_Summary_c` | Generated in Step 7 ⚡ Field pending team creation |
| `needs_confirmation` | `Signal_Needs_Confirmation_c` | `true` when `require_web_confirmation` — human confirms via press/leadership page before outreach ⚡ Field pending team creation |

⚡ = custom field not yet created. Include in payload regardless — `zoho-crud-lead` will skip ⚡ fields
gracefully until they exist in Zoho.

**Dedup field:** `Apollo_Contact_ID_c`. Dedup fallback: `First_Name + Last_Name + Company`.

---

## Step 7: Generate Signal Summary

For each lead record, write a `signal_summary` (2 sentences max), including how fresh the appointment is
and the confirmation flag:

> "[First name] [last name] appears to have become [title] at [Company] ~[role_age_days] days ago —
> surfaced as a New Executive Sponsor signal ([practice]) on [scan date]. Confirm against a press
> release / leadership page before outreach (Apollo role-start inference)."

---

## Step 8: Return JSON

Output **only** this JSON — no prose before or after.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "skill": "bdr-signal-new-exec-apollo",
  "signal_source": "Apollo",
  "signal_mechanism": "Apollo role-start inference",
  "criteria_fetched_from": {
    "reference_page": "https://app.notion.com/p/3914e4751c3a81c380e9c32aef69cb53"
  },
  "summary": {
    "candidates_found": 0,
    "candidates_enriched": 0,
    "within_recency_window": 0,
    "icp_matched": 0,
    "enrichment_credits_estimated": 0,
    "no_start_date": [],
    "notes": "<recency window used; all hits flagged for web confirmation; credit usage>"
  },
  "crm_payload": {
    "generated_by": "bdr-signal-new-exec-apollo",
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
          "Last_Name": "<last — note MASKED if reveal failed>",
          "Designation": "<new title>",
          "Company": "<company>",
          "Email": "<email or null>",
          "Phone": "<phone or null>",
          "LinkedIn_Page_URL": "<url or null>",
          "No_of_Employees": 0,
          "Industry": "<mapped value>",
          "Practice_Areas_In_Scope": ["<routed by function>"],
          "Lead_Source": "Apollo ICP Scan",
          "Apollo_Contact_ID_c": "<apollo person id>",
          "Signal_Type_c": "New Executive Sponsor",
          "Signal_Mechanism_c": "Apollo role-start inference",
          "Signal_Date_c": "<today YYYY-MM-DD>",
          "Signal_Source_Date_c": "<role start date or null>",
          "Signal_Summary_c": "<generated summary>",
          "Signal_Needs_Confirmation_c": true
        },
        "meta": {
          "role_age_days": 0,
          "relevance_score": 0,
          "enriched": true,
          "practice_routed": "<practice>",
          "needs_web_confirmation": true
        }
      }
    ]
  }
}
```

- IDs sequential: `lead_001`, `lead_002`, etc.
- `meta` block is for human review only — not written to Zoho
- If no qualifying executives found: return empty `leads: []`
- Never write a lead whose `role_age_days` is `null` or outside the window — those are not confirmed appointments

---

## Eclipse Context

**Practice area:** Multi-practice — routed per executive by function (Data / Risk & Regulatory /
Transformation / AI).

**Signal:** A newly appointed executive sponsor is the highest-intent relationship trigger in the
catalog (Signal Library SIG-003). New leaders reassess, set a 90-day agenda, and buy outside help early.
Catching the appointment inside the first ~120 days is the prime window.

**Who we surface:** the new executive themselves — the buyer — written to Zoho Leads as a net-new
prospect (contrast with `bdr-signal-job-change`, which updates executives already in the CRM).

**Eclipse wedge:** 90-day strategy support, current-state assessment, roadmap, operating-model design,
and executive advisory tuned to the new leader's function.

**Confidence & mechanism:** Medium-confidence by design — Apollo has no true appointment event, so
recency is inferred from role-start date and every hit is flagged for web/press confirmation. Rated a
gap-closer for SIG-003 on the diagnosis, downgraded from the original ZoomInfo-Executive-Move path
because this account is Apollo-only. If a stronger appointment source is added later (press-release sweep
or a licensed event feed), this skill should defer recency detection to it.
