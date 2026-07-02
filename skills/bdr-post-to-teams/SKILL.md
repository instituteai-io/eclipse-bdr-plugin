---
name: bdr-post-to-teams
description: >
  Post a message to the Eclipse BDR team's Microsoft Teams channel via Power Automate webhook.
  Accepts a plain-text or markdown string and delivers it to the configured channel.
  Used internally by other BDR skills (vendor scan, hiring signals, regulatory scan, etc.) to
  surface results and digests without leaving the agent workflow.
  Triggers on: "post to teams", "send to teams", "notify the team", "send this to the channel",
  "teams notification", or when another skill explicitly calls post-to-teams to deliver its output.
---

# Eclipse BDR — Post to Teams

Send a message to the Eclipse BDR Microsoft Teams channel via the team's Power Automate webhook.

**This skill is format-agnostic: it posts whatever message it is given, verbatim.** It does not
reformat, summarise, truncate, or restyle the input. Whatever string arrives in `text` is what
lands in the channel. Formatting the message for Teams is the **caller's** responsibility — the
calling skill (or user) should hand over a message that already reads well in Teams.

---

## Webhook

```
POST https://default7a3d1e6a4dea4b20b554105b92aa4a.82.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/294e849aa619469ab093a7eb7da2472b/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=Ike0HBBMvjoOmCmV_52EWfhqmpK3zxg1ZjyUD0eQPC8
Content-Type: application/json

{ "text": "<message>" }
```

A `202 Accepted` response means delivery was handed off to Power Automate successfully.

---

## How to use this skill from another skill

When another BDR skill produces a digest or summary it wants delivered to Teams, it should end with:

> "Now post this to Teams using the post-to-teams skill."

Then invoke this skill with the **already-formatted** message as the `text` payload. This skill
takes that string as-is — the calling skill owns the formatting and decides what the message says.

---

## Step-by-step

1. **Receive the message** — either directly from the user ("post X to Teams") or from another skill passing its output digest. Take it as-is; do not reformat it.

2. **POST to the webhook** using the Bash tool. Build the JSON safely (e.g. pipe the raw message through `jq -Rs '{text: .}'`) so line breaks and special characters are escaped correctly:

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST \
  "https://default7a3d1e6a4dea4b20b554105b92aa4a.82.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/294e849aa619469ab093a7eb7da2472b/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=Ike0HBBMvjoOmCmV_52EWfhqmpK3zxg1ZjyUD0eQPC8" \
  -H "Content-Type: application/json" \
  -d "{\"text\": \"<escaped message here>\"}"
```

3. **Check the response**:
   - `202` → success, confirm to the user that the message was sent to Teams.
   - Any other status → report the HTTP code and response body to the user; do not silently fail.

4. **Confirm** — tell the user: "Posted to Teams ✓" (or report the error clearly).

---

## 🏃 Run Logging

This skill is a building block: when invoked by another Eclipse BDR skill, the **calling** skill
owns the Output Standard (templated report, Skill Runs log, Teams summary) — do not log or post
separately from here. Never log the routine posts this skill delivers on behalf of other skills —
only standalone user-requested posts.

When invoked **standalone by a user**, log the invocation to the **Skill Runs** database
(`https://app.notion.com/p/3874e4751c3a8084a89be17b28e4c6a1`, data source
`collection://3874e475-1c3a-80a1-8ed7-000ba308ec09`) via `notion-create-pages`: Name =
`bdr-post-to-teams — <YYYY-MM-DD> — <what was done>`, Select = `bdr-post-to-teams`, Type =
`live run` or `test run`, body = a short note of the request and result. No Teams post is
required for standalone utility runs.

---

## Notes

- The webhook delivers to a specific channel configured in Power Automate — no routing parameter needed.
- This skill does NOT read from Teams or check delivery status beyond the 202 acknowledgement.

**Reference for callers (this skill does not enforce any of it):**
- Teams renders `**bold**` and `•` bullets; it does NOT render Markdown tables or headings well.
- Power Automate collapses single newlines — use a **blank line (`\n\n`)** between blocks to get visible spacing, or the message reads as a wall of text.
- Keep messages short. A tight summary plus a link to the full write-up (e.g. the Notion Skill Runs page) reads far better than dumping a whole report. Split very long content into multiple calls.
