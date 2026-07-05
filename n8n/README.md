# Daily Taiwan Eligible Async Remote Job Hunter (n8n workflow)

Import `daily-taiwan-async-remote-job-hunter.json` into n8n. It runs every day
at 8:30 AM `Asia/Taipei` and enforces one rule above all others:

> A job only survives if Jerry can do it while physically based in Taiwan —
> no relocation, no US/EU/UK/Canada/Australia work authorization, and no
> fixed midnight hours.

## Pipeline

```
Schedule Trigger (08:30 Asia/Taipei)
  -> Config (keywords, filter patterns, sheet/telegram/email settings)
  -> Fetch sources (Remote OK, Remotive, We Work Remotely + optional others)
  -> Normalize Jobs -> Keyword Match Filter
  -> Dedupe against "Raw Jobs" sheet (job_url, then company+title)
  -> Append Raw Jobs
  -> Hard Filter (reject US/EU/UK/Canada/Australia-only, hybrid, onsite language)
       -> hard reject -> Append Rejected Jobs
       -> pass -> AI Taiwan Eligibility Check (Claude) -> Async Filter
          -> Role Fit Filter -> Scoring -> Finalize & Split
             -> reject bucket -> Append Rejected Jobs
             -> filtered bucket -> Append Filtered Jobs
                                -> instant alert if score >= 85
                                -> cover letter draft if score >= 85
  -> Build Daily Digest -> Telegram and/or Gmail
```

## Before you activate it

1. **Google Sheet** — create a sheet named `Taiwan Eligible Async Remote Jobs`
   with these tabs and header rows (exact column names, first row):

   - **Raw Jobs**: `date_found, source, job_title, company, job_url, location_text, salary, description, posted_date, tags, job_id`
   - **Filtered Jobs**: `date_found, score, job_title, company, job_url, salary, taiwan_eligible, taiwan_reason, location_risk, timezone_risk, contractor_possible, async_level, role_category, why_good_fit, decision, application_status, notes`
   - **Applied**: `date_applied, job_title, company, job_url, cover_letter_used, status, follow_up_date, notes` — filled in manually as you apply; the workflow does not write here.
   - **Rejected**: `date_rejected, job_title, company, job_url, reason`
   - **Cover Letters** *(added beyond the original spec, to hold generated drafts)*: `date_found, job_title, company, job_url, score, cover_letter_draft`

2. **Credentials** to create in n8n and wire into every node that currently
   has a `PUT_CREDENTIAL_ID` placeholder:
   - Google Sheets OAuth2 (`googleSheetsOAuth2Api`)
   - Telegram Bot API (`telegramApi`) and/or Gmail OAuth2 (`gmailOAuth2`) — the workflow wires both; disable whichever channel you don't want.
   - HTTP Header Auth credential for Claude: header name `x-api-key`, value = your Anthropic API key. Used by the two HTTP Request nodes that call `https://api.anthropic.com/v1/messages` (eligibility check + cover letter draft).

3. **Config node** — open it and fill in:
   - `spreadsheetId` — the Google Sheet's ID from its URL
   - `telegramChatId` — your Telegram chat ID
   - `digestEmailTo` — where the Gmail digest should go

## Source coverage

Enabled out of the box (public, no auth needed):
- Remote OK (`https://remoteok.com/api`)
- Remotive (`https://remotive.com/api/remote-jobs`)
- We Work Remotely (`https://weworkremotely.com/remote-jobs.rss`)

Disabled placeholders — these boards don't have a stable public API/RSS as of
when this workflow was built, so they ship disabled rather than pointed at a
guessed URL that could break silently:
- Himalayas, Wellfound, Dynamite Jobs, Y Combinator Work at a Startup, Otta

To bring one online: point its HTTP Request node at a real API/RSS endpoint
(or a self-hosted [RSS-Bridge](https://github.com/RSS-Bridge/rss-bridge)
instance targeting that board), enable the node, and wire its output into
**Merge All Sources**. If its output already looks like
`{ job_title, company, job_url, source, location_text, salary, description, posted_date, tags }`,
**Normalize Jobs** picks it up automatically via its passthrough branch — no
other changes needed.

## Scoring (0–100, kept only if >= 70)

- Taiwan eligibility (0–40): 40 if AI says "yes" + low location risk, 30 if "yes" with some risk, 20 if "unclear" without high risk, 0 otherwise.
- Async fit (0–25): 25 if clearly async, 10 if unclear-but-flexible, plus a bonus for "distributed team" / "remote-first" language.
- Role fit (0–25): 25 for a direct match against the target role list (data analyst, data scientist, BI, marketing/growth analyst, AI marketing/automation/workflow consultant), 15 for an adjacent remote role.
- Salary fit (0–25): 10 if listed, 15 if >= $3,000/month, 25 if >= $5,000/month.

Score >= 85 triggers both an instant Telegram alert and a cover-letter draft
(via Claude) appended to the **Cover Letters** tab.

## Final decision logic

- `apply_immediately` — taiwan_eligible = yes, score >= 85, async ok, location risk low, timezone risk low/medium
- `manual_check` — taiwan_eligible = unclear, score >= 85, location risk medium
- `keep` — everything else that scored >= 70 and wasn't hard-rejected
- `reject` — taiwan_eligible = no, OR location/timezone risk high, OR rigid hours, OR score < 70

## Notes on the AI eligibility prompt

The exact prompt from the spec is sent as the `system` message to Claude, with
the job description as the user message, per job. The HTTP Request node builds
the request body via `JSON.stringify(...)` inside the n8n expression (rather
than hand-templating JSON text) so job descriptions containing quotes,
backslashes, or newlines can't corrupt the request.
