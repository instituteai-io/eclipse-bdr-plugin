---
name: bdr-scan-regulatory
description: >
  Fetch any regulatory enforcement page URL and extract structured signal objects for
  TeamEclipse BDR outreach. Accepts a URL from OCC, FDIC, CFPB, SEC, FINRA, Federal Reserve,
  NCUA, or any other regulatory body. Auto-detects the source, extracts enforcement actions,
  scores each one against Eclipse's ICP (data, risk, compliance, financial crime consulting),
  and returns structured JSON signals ready for Zoho CRM input.
  Use when the user pastes a regulatory enforcement URL and asks to scan it, run the enforcement
  scan, check for signals, or review enforcement actions. Triggers on: "scan this page",
  "run the regulatory scan", "what signals are here", "check this enforcement action",
  "regulatory enforcement scan", or any URL from a known regulator domain (sec.gov,
  consumerfinance.gov, occ.gov, fdic.gov, finra.org, federalreserve.gov, ncua.gov).
---

# Eclipse BDR — Regulatory Scan by Source

Fetch a regulatory enforcement page URL, extract enforcement actions, score each one against
Eclipse's ICP, and return structured JSON signals ready for BDR outreach.

---

## Step 1: Confirm the URL

The user must provide a URL. If none was given, ask:

> "Please paste the URL of the regulatory enforcement page you'd like me to scan."

Accept one or more URLs. If multiple, process each and merge into one output.

---

## Step 2: Detect the Source

Match the URL domain to a known regulator:

| Domain pattern | Regulator | Action types to look for |
|---|---|---|
| `sec.gov` | SEC | Litigation releases, administrative proceedings, press releases |
| `consumerfinance.gov` | CFPB | Consent orders, supervisory actions |
| `occ.gov` or `apps.occ.gov` | OCC | Formal agreements, consent orders, C&D orders, civil money penalties |
| `fdic.gov` | FDIC | Consent orders, civil money penalties, terminations of insurance |
| `finra.org` | FINRA | Disciplinary actions, firm-level fines and suspensions |
| `federalreserve.gov` | Federal Reserve | Enforcement actions against bank holding companies |
| `ncua.gov` | NCUA | Enforcement actions against credit unions |
| Other | Unknown | Attempt generic extraction |

---

## Step 3: Fetch and Extract

Use WebFetch to retrieve the page. Extract enforcement actions using source-specific logic.

**SEC (`sec.gov`)** — Look for litigation releases, administrative proceedings, press releases. Extract per item: respondent/firm name, case number, date filed, violation type, penalty amount, charge summary. Prefer firm-level actions over named individuals.

**CFPB (`consumerfinance.gov`)** — Look for enforcement action entries. Extract: institution name, action type, date, description, penalty amount, remediation requirements. Common institution types: banks, fintechs, servicers, debt collectors, payment companies.

**OCC (`occ.gov`, `apps.occ.gov`)** — Look for enforcement actions table or search results. Extract: institution name, charter type, action type, effective date, summary. OCC actions are bank-specific — high Eclipse relevance. Key types: Formal Agreement, Consent Order, C&D, CMP.

**FDIC (`fdic.gov`)** — Look for enforcement decisions list. Extract: institution name, city/state, action type, effective date, description. Key types: Consent Order, Order to Pay, Removal/Prohibition Order, Termination of Insurance.

**FINRA (`finra.org`)** — Look for disciplinary actions or monthly reports. Focus on **firm-level** actions. Extract: firm name, violation type, sanction amount, brief description. If URL links to a PDF monthly report, flag the PDF URL and extract any firm names visible in HTML.

**Federal Reserve (`federalreserve.gov`)** — Look for enforcement actions press releases or order PDFs. Extract: institution/holding company name, action type, date, findings summary.

**Unknown source** — Attempt to identify any table/list with institution name, date, action type. Flag source as Unknown. If page is not an enforcement listing, return `signals: []` with a note.

---

## Step 4: Score Eclipse ICP Relevance

Score every extracted enforcement action from 1–10.

| Condition | Points |
|---|---|
| Institution is a bank, credit union, or trust company | +3 |
| Institution is a broker-dealer, investment advisor, or asset manager | +2 |
| Institution is a fintech, payments company, or money services business | +1 |
| Action involves BSA/AML, sanctions, KYC/CDD, or financial crime program deficiencies | +3 |
| Action involves regulatory reporting failures, model risk, or stress testing | +3 |
| Action involves data governance, data quality, IT controls, or reporting architecture | +3 |
| Action involves board/governance deficiencies, audit findings, or management controls | +2 |
| Action creates a mandatory remediation plan or board-level oversight requirement | +1 |
| Action is only against named individuals — firm itself not charged | -2 |
| Action is a small fine under $100K with no remediation requirement | -1 |

**Practice area mapping:**

| Practice Area | Keywords |
|---|---|
| Financial Crime | BSA, AML, anti-money laundering, KYC, CDD, EDD, sanctions, OFAC, fraud, financial crimes program |
| Risk & Compliance | regulatory reporting, MRA, MRIA, model risk, stress testing, risk framework, compliance program deficiencies |
| Data Management | data governance, data quality, reporting controls, data architecture, IT controls, reporting systems failures |
| Governance | board oversight, audit committee, internal controls, management deficiencies, audit findings, corporate governance |

An action can map to multiple practice areas — include all that apply.

**Urgency:**
- `high`: Consent order with board-level reporting requirements, OR penalty over $1M, OR systemic program-level findings
- `medium`: Formal agreement, MOU, penalty $100K–$1M, or targeted remediation
- `low`: Notice of charges, fine under $100K, informal action, individual-only

**ICP match:** `true` if the institution is a financial services firm AND at least one Eclipse practice area applies.

---

## Step 5: Generate Eclipse Assessment

For each signal with `relevance_score >= 5`, generate:

**`wedge`** — Eclipse service angle (1–2 sentences, specific to this action):
- Financial Crime: "BSA/AML remediation roadmap, KYC/CDD program redesign, and compliance program uplift."
- Risk & Compliance: "Regulatory remediation roadmap and risk reporting uplift in response to [specific finding]."
- Data Management: "Data governance framework, reporting controls remediation, and data quality program."
- Governance: "Board-level remediation oversight, audit response support, and management controls redesign."
- Multi-practice: Combine — Eclipse's cross-functional model (data + risk + financial crime) is a key differentiator.

**`recommended_approach`** — Tom's opening outreach angle (1–2 sentences, specific to this action). Reference the specific action type and regulator, lead with Eclipse's relevant experience, frame urgency.

---

## Step 6: Generate CRM Payload

For every signal included in the output (i.e. `relevance_score >= 5` AND `icp_match: true`),
generate one account entry and one deal entry in `crm_payload`.

**Accounts** — one per unique company name across all included signals:
- `action`: `upsert`
- `dedup_field`: `company_name`
- `industry`: infer from institution type — bank/credit union → `"Banking"`, broker-dealer/investment advisor → `"Financial Services"`, fintech/payments → `"Financial Technology"`, other → `"Financial Services"`
- `icp_tier`: `null` (not determinable from enforcement data alone — set manually or by enrichment)
- `apollo_account_id`: `null` (set by `apollo_enrich_account`)
- `zoominfo_account_id`: `null` (set by `zoominfo_enrich_account`)
- All other account fields: `null`

**Contacts** — always empty `[]`. Regulatory scan does not create or modify contacts.
Contacts are surfaced by plays after deal creation.

**Deals** — one per included signal:
- `action`: `skip_if_exists`
- `dedup_query`: `"company_name = '<company>' AND signal_source = '<regulator>' AND signal_date = '<date>'"`
- `deal_name`: `"<company> — Regulatory Action — <YYYY-MM-DD>"`
- `stage`: `"Signal Detected"`
- `deal_type`: `"New Logo"`
- `amount`: `null`
- `close_date`: 90 days from `scan_date`
- `lead_source`: `"Regulatory Scan"`
- `practice_areas`: from `eclipse_assessment.practice_areas`
- `signal_type`: `"Regulatory Action"`
- `signal_source`: regulator abbreviation (e.g. `"OCC"`, `"FDIC"`)
- `signal_date`: enforcement action date
- `signal_summary`: from `signal.summary`
- `suggested_approach`: from `eclipse_assessment.recommended_approach`
- `priority_score`: `relevance_score × 10` (scales 1–10 → 10–100)
- `play_source`: `"New Opp Discovery"`

---

## Step 7: Return JSON

Output **only** this JSON — no prose before or after unless the user asks for a summary.

```json
{
  "scan_date": "<today YYYY-MM-DD>",
  "source_url": "<the URL that was scanned>",
  "regulator": "<OCC|FDIC|CFPB|SEC|FINRA|Federal Reserve|NCUA|Unknown>",
  "signals": [
    {
      "id": "signal_001",
      "company": "<institution or firm name>",
      "regulator": "<regulator abbreviation>",
      "action_type": "<e.g. Consent Order, Civil Money Penalty, Formal Agreement, Cease and Desist>",
      "date": "<YYYY-MM-DD or best estimate>",
      "summary": "<2–3 sentences: what happened, what was found, what is now required>",
      "source_url": "<direct URL to this specific action if available, otherwise the page URL>",
      "eclipse_assessment": {
        "practice_areas": ["<practice area 1>", "<practice area 2>"],
        "wedge": "<Eclipse service angle specific to this action>",
        "urgency": "high|medium|low",
        "budget_signal": true,
        "icp_match": true,
        "relevance_score": 0,
        "recommended_approach": "<Tom's outreach angle specific to this action>"
      }
    }
  ],
  "summary": {
    "total_extracted": 0,
    "icp_matched": 0,
    "high_priority": 0,
    "signals_excluded": 0,
    "exclusion_reason": "<if signals were excluded, explain why>"
  },
  "crm_payload": {
    "generated_by": "bdr-scan-regulatory",
    "generated_at": "<today YYYY-MM-DD>",
    "accounts": [
      {
        "id": "acc_001",
        "action": "upsert",
        "dedup_field": "company_name",
        "fields": {
          "company_name": "<institution name>",
          "industry": "<inferred from institution type>",
          "icp_tier": null,
          "apollo_account_id": null,
          "zoominfo_account_id": null
        }
      }
    ],
    "contacts": [],
    "deals": [
      {
        "id": "deal_001",
        "action": "skip_if_exists",
        "dedup_query": "company_name = '<company>' AND signal_source = '<regulator>' AND signal_date = '<date>'",
        "fields": {
          "deal_name": "<company> — Regulatory Action — <YYYY-MM-DD>",
          "company_name": "<institution name>",
          "primary_contact": null,
          "stage": "Signal Detected",
          "deal_type": "New Logo",
          "amount": null,
          "close_date": "<90 days from scan_date>",
          "lead_source": "Regulatory Scan",
          "practice_areas": ["<from eclipse_assessment.practice_areas>"],
          "signal_type": "Regulatory Action",
          "signal_source": "<regulator abbreviation>",
          "signal_date": "<enforcement action date>",
          "signal_summary": "<from signal.summary>",
          "suggested_approach": "<from eclipse_assessment.recommended_approach>",
          "priority_score": 0,
          "play_source": "New Opp Discovery"
        }
      }
    ]
  }
}
```

- Include only signals with `relevance_score >= 5`
- IDs sequential: `signal_001`, `signal_002`, etc. — `acc_001`, `acc_002`, etc. — `deal_001`, `deal_002`, etc.
- One account entry per unique company (deduplicate if same company appears in multiple signals)
- If page has login wall, is empty, or has no enforcement data: return `signals: []`, `crm_payload.accounts: []`, `crm_payload.deals: []`, and explain in `exclusion_reason`

---

## Eclipse Context

**Client:** TeamEclipse — professional services firm specializing in data, risk, compliance, and financial crime consulting for financial institutions.

**Seller-doer:** Tom — uses enforcement signals to identify accounts with active remediation obligations where Eclipse can be positioned as the delivery partner.

**ICP:** Mid-to-large banks, broker-dealers, investment managers, fintechs in regulated markets — especially those with active consent orders, formal agreements, or civil money penalties requiring structured remediation plans.

**Practice areas → wedge:**
- **Financial Crime** → BSA/AML program build or remediation, KYC/CDD redesign, transaction monitoring tuning
- **Risk & Compliance** → Regulatory remediation roadmap, risk reporting uplift, MRA/MRIA response
- **Data Management** → Data governance framework, reporting controls, data quality program
- **Governance** → Board-level remediation oversight, audit support, management control redesign

**Best signal:** A consent order at a mid-to-large bank with specific remediation requirements in one or more Eclipse practice areas — creates urgency, allocates budget, mandates external expertise.
