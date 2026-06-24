---
name: bdr-fetch-contacts-from-company
description: >
  Reusable building-block skill: given a company and an Eclipse practice, return the ICP-matching
  contacts at that company. The shared bridge from a company-level signal (pain-point, corp-events,
  vendor adoption) to actual people — so signal skills, agents, or a person can turn "Acme Bank is a
  signal" into "here are the data / risk / transformation / AI decision-makers at Acme Bank." Takes a
  company (name, or a resolved ZoomInfo/Apollo id, or domain) + a practice flag (Data Management,
  Risk & Regulatory, Transformation, or Artificial Intelligence) + a provider flag (zoominfo default,
  or apollo). Reads the practice's ICP buyer titles and seniority fresh from the ICP Analysis Notion
  page (never hardcoded), then queries the provider for matching contacts AT that company and returns
  them as structured JSON. It does NOT write to Zoho and does NOT enrich emails/phones by default —
  the caller decides whether to route the people to zoho-crud-lead or run a separate enrichment pass.
  Use when asked to find / pull / surface the contacts (people, decision-makers, buyers) at a named
  company for a practice, get the data or risk people at a company, or fetch ICP contacts at a company.
  Triggers on: "fetch contacts from <company>", "who are the data/risk/AI people at <company>",
  "get the ICP contacts at <company>", "surface decision-makers at <company>",
  "bdr-fetch-contacts-from-company".
---

# Eclipse BDR — Fetch Contacts from Company

Given a **company** + an Eclipse **practice**, return the **ICP-matching contacts** at that company.
This is a **building-block utility** — the shared contact-surfacing primitive. It **returns people;
it writes nothing**.

Use it to turn a company-level signal into people: a signal skill (pain-point, corp-events, vendor),
an agent, or a person calls this with a company + practice and gets back the right decision-makers.

> **Composability is the point.** This skill **enriches by default** (returns *contactable* people),
> but it **does not write to Zoho** and **imposes no output format on any caller**. It returns
> enriched contacts **per company** and nothing more — it never patches, stitches into, or assumes
> another skill's table or shape. The caller (a signal skill, an agent, or a person) decides how to
> present or route them. That decoupling is exactly what makes it reusable everywhere.

---

## Inputs

| Input | Required | Notes |
|---|---|---|
| `company` | yes | A company **name**, a resolved **`zoominfoCompanyId`** / **Apollo org id**, or a **domain**. Prefer an id — callers that already have one (e.g. a signal skill that just found the company) skip re-resolution. |
| `practice` | yes | `Data Management` \| `Risk & Regulatory` \| `Transformation` \| `Artificial Intelligence` |
| `provider` | no | `zoominfo` (default) \| `apollo` |
| `max_contacts` | no | default `10` (best matches, not exhaustive) — also the enrichment cap |
| `enrich` | no | default **`true`** — the skill returns *contactable* people (email + phone), which is the point. Set `false` only for cheap name-only use (counting, dedup, "who's there"). See Credits Note. |

If `practice` is missing or ambiguous, ask. If `company` is missing, ask.

---

## ⚠️ Titles Reference (fetched fresh, never hardcoded)

The practice → **buyer titles + seniority** mapping is **not hardcoded**. Fetch it fresh from the
**ICP Analysis** Notion page each run (via `notion-fetch`):
`https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0`

Use the **"Target Buyer Titles"** list under the matching practice section (2A Data Management,
2B Risk & Regulatory, 2C Transformation, 2D Artificial Intelligence) and the firm-wide seniority.
If the page is unreachable, **stop and tell the user** — do not guess titles.

> When the hiring skills are refactored to call this skill, their per-practice reference pages and
> this title source can converge on one reference. For now, ICP Analysis is the single source.

---

## 💳 Credits Note

Provider **contact search** (`search_contacts` / Apollo people search) counts toward request/record
limits but is **not** a bulk-credit enrichment. **Enrichment (`enrich:true`, the default) does spend
credits** — ~1 ZoomInfo bulk credit / 1 Apollo credit **per contact**, capped at `max_contacts`.
ZoomInfo caches enrichment for **a year**, so re-pulling the same person within 12 months is free.
Because `max_contacts` (default 10) bounds the spend, default-on enrichment is the intended behavior
— the skill exists to return contactable people. Disclose credits used in the output. For a large
manual run, confirm the count first; set `enrich:false` for a cheap name-only pass.

---

## Step 1: Resolve the Practice's Titles

Fetch ICP Analysis; extract the practice's **Target Buyer Titles** and the firm-wide **seniority**
(C-Suite, VP, Director, Head). Build:
- a title OR-set for the provider's title filter
- a management-level set (map Head/Chief → C Level Exec; VP → VP Level Exec; Director → Director)

---

## Step 2: Resolve the Company to an ID

- If `company` is already a `zoominfoCompanyId` (provider=zoominfo) or Apollo org id (provider=apollo),
  use it directly.
- If a **name** or **domain** was given, resolve it:
  - ZoomInfo: `search_companies` (companyName or companyWebsite) → take the best match's `companyId`.
  - Apollo: `apollo_mixed_companies_search` (q_organization_name / domain) → org `id`.
- If multiple plausible matches, pick the closest by name + domain and note the choice; if genuinely
  ambiguous, list the candidates and ask.

---

## Step 3: Search Contacts AT the Company

**ZoomInfo (`search_contacts`):**

| Param | Value |
|---|---|
| `companyId` | the resolved id (accepts a comma-separated list to batch multiple companies) |
| `jobTitle` | the practice title OR-set (letters/spaces + `OR` only — e.g. `"Chief Data Officer OR Data Governance OR Data Quality OR Chief Analytics Officer"`) |
| `managementLevel` | mapped levels, e.g. `"C Level Exec,VP Level Exec,Director"` |
| `sort` | `-contactAccuracyScore` (most reachable first) |
| `pageSize` | `max_contacts` |

**Apollo (`apollo_mixed_people_api_search`):** `organization_ids: [<org id>]`, `person_titles: [...]`,
`person_seniorities: ["c_suite","vp","director","head"]`, `per_page: max_contacts`. (Apollo masks
some last names until enriched — flag this in the output, don't auto-enrich.)

Capture per contact: `first_name`, `last_name`, `title`, `management_level` / seniority,
`linkedin_url` (ZoomInfo `externalUrls`; Apollo `linkedin_url`), accuracy/relevance score, provider
person id. Leave `email`/`phone` null unless `enrich:true`.

---

## Step 4: Enrich (default ON)

By default (`enrich:true`), enrich the surfaced contacts so the output is *contactable*: ZoomInfo
`enrich_contacts` by `personId` (batch up to 10 per call) / Apollo `apollo_people_match` by id,
requesting `email`, `phone`, `mobilePhone`, `jobTitle`, `managementLevel`, `externalUrls`,
`companyName`. Enrich only the `max_contacts` you're returning (bounds credits). Skip this step only
if `enrich:false`. Report credits used in the summary.

---

## Step 5: Return JSON

Output **only** this JSON — no prose before or after unless the user asks for a summary.

```json
{
  "skill": "bdr-fetch-contacts-from-company",
  "fetched_at": "<today YYYY-MM-DD>",
  "provider": "zoominfo | apollo",
  "practice": "<practice>",
  "enriched": true,
  "titles_source": "https://app.notion.com/p/3a24e4751c3a8289823f01be4d08eec0",
  "company": {
    "input": "<what the caller passed>",
    "resolved_name": "<name>",
    "resolved_id": "<provider company/org id>"
  },
  "contacts": [
    {
      "first_name": "<first>",
      "last_name": "<last or masked note>",
      "title": "<title>",
      "seniority": "<C Level Exec | VP Level Exec | Director | ...>",
      "linkedin_url": "<url or null>",
      "provider_person_id": "<id>",
      "accuracy_or_relevance": 0,
      "email": "<business email, when enriched>",
      "phone": "<direct phone, when enriched>",
      "mobile_phone": "<mobile, when enriched>"
    }
  ],
  "summary": {
    "contacts_found": 0,
    "enriched": true,
    "credits_used": 0,
    "notes": "<company match note; Apollo last-name masking if applicable; titles practice used>"
  }
}
```

- Sort by accuracy/relevance desc; cap at `max_contacts`.
- This skill never writes to Zoho. To create leads, the caller hands these contacts to
  `zoho-crud-lead`.
- If the company can't be resolved or has no matching contacts: return `contacts: []` and explain in
  `summary.notes`.
- **Batch use:** for several companies of the same practice, pass a comma-separated `companyId` list
  to one ZoomInfo `search_contacts` call and group the results by company in the output.

---

## Eclipse Context

**Client:** TeamEclipse — data, risk, compliance, financial crime consulting for FIs.
**Seller-doer:** Tom.

**Role in the system:** the shared people-surfacing primitive. Company-level signal skills
(`bdr-signal-painpoint-zoominfo`, `bdr-signal-corp-events-zoominfo`, `bdr-signal-vendor-zoominfo`)
identify the *account* and stay digest-only; this skill turns any such account into the right
*people* for a given practice, on demand. It centralizes the contact logic currently duplicated in
the hiring skills' "Find ICP Contacts" step — those will be refactored to call this in a later pass.

**Output routing:** returns people; writes nothing. Composition (Zoho via `zoho-crud-lead`,
enrichment) is the caller's decision — which is exactly what keeps this skill reusable.
