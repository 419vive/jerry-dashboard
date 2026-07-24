# Daily Taiwan Eligible Async Remote Job Hunter (n8n workflow)

`daily-taiwan-async-remote-job-hunter.json` is an importable n8n workflow that
hunts daily for remote jobs Jerry can legally and practically do while living
in Taiwan — no relocation, no foreign work authorization, no midnight shifts.

## Import

n8n → Workflows → **Import from File** → select
`daily-taiwan-async-remote-job-hunter.json`. It imports inactive; review the
placeholders below, then toggle it active.

## Pipeline

1. **Schedule Trigger** — daily at 08:30, `Asia/Taipei` (set in workflow
   Settings → Timezone, plus a cron expression on the trigger node).
2. **Sources** — RemoteOK (API), Remotive (API, looped once per keyword),
   Himalayas (API, looped once per keyword), We Work Remotely (RSS × 3
   categories: Programming, Sales & Marketing, All Other).
   *Wellfound, YC "Work at a Startup", Otta and Dynamite Jobs are **not**
   wired up live* — they require a logged-in / JS-rendered session or have no
   documented public feed, so a plain HTTP Request can't read them. There's a
   disabled placeholder node + sticky note showing where to plug in a
   scraping service (e.g. an Apify actor) if you want them included.
3. **Step 4 — Normalize All**: maps every source into one schema
   (`job_title, company, job_url, source, location_text, salary,
   description, posted_date, tags, job_id`) and drops in-run duplicate URLs.
4. **Step 5 — Dedupe vs Sheet**: reads existing `job_url`s (and a
   `company+job_title` backup key) from the *Raw Jobs* tab and drops anything
   already seen, then appends the rest to *Raw Jobs*.
5. **Step 6 — Taiwan Hard Filter**: free keyword pass. Rejects obvious
   country-restricted/onsite postings before spending an AI call, unless a
   whitelist phrase (worldwide remote, APAC remote, EOR/Deel/Remote.com/
   Oyster, etc.) is also present.
6. **Step 7 — AI Taiwan Eligibility Check**: calls the Claude Messages API
   with the exact eligibility prompt from the spec and parses the JSON
   verdict (`taiwan_eligible`, `location_risk`, `timezone_risk`,
   `contractor_possible`, `decision`).
7. **Step 8 — Async Filter**: keyword pass for async/flexible/distributed
   language vs. rigid-hours language.
8. **Step 9 — Role Fit Filter**: scores role relevance (data/BI/growth/AI
   marketing/automation roles first).
9. **Step 10 — Scoring**: combines Taiwan eligibility, async fit, role fit
   and salary into a single 0–100 score (see "Scoring assumption" below).
10. **Step 14 — Final Decision**: applies the apply / manual_check / reject
    rules from the spec and writes `why_good_fit`.
11. Results are routed to the **Filtered Jobs** or **Rejected** sheet tabs,
    fed into the **daily digest**, and jobs scoring ≥ 85 get a **Claude-drafted
    cover letter** saved as a Gmail draft for Jerry to review before sending.

## Required credentials (create these in n8n first)

| Credential | Used by | Notes |
|---|---|---|
| HTTP Header Auth (`x-api-key: <ANTHROPIC_API_KEY>`) | *Step 7 AI check*, *Step 13 cover letter* HTTP Request nodes | Attach to both `httpRequest` nodes calling `api.anthropic.com` |
| Google Sheets OAuth2 | all `Google Sheets` nodes | Needs access to the sheet below |
| Gmail OAuth2 | *Create Gmail Draft*, *Send Gmail Digest* | Optional if you only want Telegram |
| Telegram API (bot token) | *Send Telegram Digest* | Optional if you only want Gmail |

Every `Google Sheets` node currently has `documentId` set to the placeholder
`YOUR_GOOGLE_SHEET_ID` — replace it (and pick the right tab in `sheetName`)
after you create the sheet below. Set `TELEGRAM_CHAT_ID` as an n8n
environment variable, or hardcode it on the Telegram node. The Gmail nodes
default `sendTo`/draft recipient to `419vive@gmail.com` — change if needed.

## Google Sheet: "Taiwan Eligible Async Remote Jobs"

Create one spreadsheet with four tabs and these header rows:

**Raw Jobs**
`date_found, source, job_title, company, job_url, location_text, salary, description, posted_date, tags, job_id`

**Filtered Jobs**
`date_found, score, job_title, company, job_url, salary, taiwan_eligible, taiwan_reason, location_risk, timezone_risk, contractor_possible, async_level, role_category, why_good_fit, decision, application_status, notes`

**Applied**
`date_applied, job_title, company, job_url, cover_letter_used, status, follow_up_date, notes`
*(This tab is filled in manually as Jerry actually applies — the workflow
only prepares cover letter drafts, it never auto-applies.)*

**Rejected**
`date_rejected, job_title, company, job_url, reason`

The `Get Existing Raw Jobs` and `Append to *` Google Sheets nodes use
`autoMapInputData`, so as long as the sheet header row matches the field
names produced by the Code nodes, columns map automatically.

## Scoring assumption (worth knowing before you trust the numbers)

The original spec lists async-fit, role-fit and salary-fit as several
point tiers (e.g. "25 if async, 20 if flexible hours, 15 if distributed
team, 10 if remote first") without saying whether they stack. Implemented
here as **max-of-tier, not additive** — a job gets the single highest
tier it qualifies for in each category — so the total stays bounded near
100 instead of blowing past it. If you want additive scoring instead,
edit the `Step 8: Async Filter`, `Step 9: Role Fit Filter` and
`Step 10: Scoring` Code nodes.

Salary parsing is a heuristic: it grabs the largest number in the salary
text and guesses annual-vs-monthly by whether it's ≥ 12,000. Free-text
salary fields ("competitive", ranges with unusual formatting) may not
parse — those jobs just get 0 salary points instead of failing.

## Known limitations / TODO before relying on this

- Wellfound, YC Work at a Startup, Otta and Dynamite Jobs are not fetched
  live (see above) — the digest's rejected-source counts will only ever
  reflect RemoteOK/Remotive/Himalayas/WWR.
- The AI eligibility prompt trusts the job description text; postings that
  don't clearly state a location policy will often come back `unclear` and
  land in manual_check rather than being auto-applied.
- Run once manually ("Execute workflow") before enabling the schedule, and
  spot-check the first digest against the Filtered/Rejected sheets.
