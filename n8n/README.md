# Daily Taiwan Eligible Async Remote Job Hunter (n8n workflow)

An importable n8n workflow that pulls remote job listings, hard-filters them for
Taiwan eligibility, runs an AI (Claude) eligibility + risk check, scores each job,
and sends Jerry a daily digest (plus an instant alert for anything scoring 85+).

File: [`taiwan-eligible-async-remote-job-hunter.json`](./taiwan-eligible-async-remote-job-hunter.json)

## The one rule everything is built around

> I can do this job while living in Taiwan, without moving, without foreign work
> authorization, and without destroying my sleep schedule.

A job only reaches "apply" or "manual_check" if it survives every stage below.
Anything that fails is logged to the **Rejected** sheet with a reason — nothing is
silently dropped.

## How to import

1. In n8n: **Workflows → Import from File** → select the JSON file. (Or paste its
   contents into **Import from URL/Clipboard**.)
2. Open every node once after import — n8n re-validates node parameters the first
   time you open them, and the Google Sheets / HTTP nodes need your own resource
   selected from their dropdowns.

## Required setup before activating

1. **Google Sheet** — create a spreadsheet named `Taiwan Eligible Async Remote Jobs`
   with 4 tabs and these exact header rows (auto-mapping relies on the field names
   matching):

   - **Raw Jobs**: `date_found, source, job_title, company, job_url, location_text, salary, description, posted_date, tags, job_id`
   - **Filtered Jobs**: `date_found, score, job_title, company, job_url, salary, taiwan_eligible, taiwan_reason, location_risk, timezone_risk, contractor_possible, async_level, role_category, why_good_fit, decision, application_status, notes`
   - **Applied**: `date_applied, job_title, company, job_url, cover_letter_used, status, follow_up_date, notes`
   - **Rejected**: `date_rejected, job_title, company, job_url, reason`

   Copy the sheet's ID (the long string in its URL) into every Google Sheets node's
   `documentId` field (currently `REPLACE_WITH_GOOGLE_SHEET_ID`).

   The **Applied** sheet is intentionally *not* written by this workflow — actually
   submitting an application should stay a human decision, so update that tab
   yourself (or build a small companion workflow) once you've actually applied.

2. **Credentials**
   - **Google Sheets OAuth2** — attach to the 4 Google Sheets nodes.
   - **HTTP Header Auth** credential named e.g. "Anthropic API Key" with header
     `x-api-key` = your Anthropic API key. Attach to both `AI Taiwan Eligibility
     Check` and `Generate Cover Letter Draft` HTTP Request nodes.
   - **Telegram** bot token (if you want Telegram alerts) — attach to the two
     Telegram nodes, and set your `chatId`.
   - **Gmail OAuth2** (if you want email alerts) — attach to the two Gmail nodes,
     and set the destination email address. You can disable whichever channel
     (Telegram or Gmail) you don't want.

3. **Timezone** — workflow Settings already has `timezone: Asia/Taipei`, so the
   `30 8 * * *` cron on the Schedule Trigger fires at 08:30 Taiwan time daily.

## Pipeline overview

1. **Schedule Trigger** — 08:30 Asia/Taipei, daily.
2. **Sources** — RemoteOK (API), Remotive (API), We Work Remotely (RSS,
   Programming + Sales/Marketing categories), Himalayas (API). Each is tagged with
   `_source` and merged into one stream.
3. **Wellfound, Dynamite Jobs, Y Combinator Work at a Startup, and Otta** have no
   public API or RSS and are heavily JS-rendered / anti-bot protected, so they
   are **not** wired in — see the sticky note in the workflow canvas for
   recommended alternatives (email-digest parsing or a dedicated scraper service
   posting into a webhook).
4. **Normalize** — maps every source's raw fields into a common schema
   (`job_title, company, job_url, source, location_text, salary, description,
   posted_date, tags`).
5. **Keyword filter** — keeps only postings matching the target search phrases
   (async remote, worldwide remote, remote data analyst, etc.).
6. **Dedupe** — compares against existing `job_url` (and `company::job_title`
   fallback) already in the Raw Jobs sheet; only new jobs continue.
7. **Append Raw Jobs** — every new unique job is logged here regardless of what
   happens next.
8. **Hard filter** — instantly rejects "US only / EU only / hybrid / must be
   authorized to work in ... / fixed US or EU hours" etc., and flags jobs that
   explicitly say worldwide remote / APAC remote / contractor-friendly / EOR /
   Deel / Remote.com / Oyster. Hard-rejected jobs skip the AI call entirely (saves
   API cost) and go straight to the Rejected sheet.
9. **AI Taiwan eligibility check** — calls Claude (`claude-sonnet-5`) with the
   exact eligibility prompt from the spec, returns
   `taiwan_eligible / reason / location_risk / timezone_risk /
   contractor_possible / decision`.
10. **Async filter** — flags high/medium/low async-friendliness, or hard-rejects
    rigid-hours postings ("strict 9 to 5", "daily live calls required", etc.).
11. **Role fit filter** — flags core vs. adjacent role match (data analyst, BI
    analyst, growth/marketing analyst, AI automation/workflow roles, ...).
12. **Scoring (0–100)** — Taiwan eligibility (0/20/30/40), async fit (up to 25),
    role fit (15/25), salary (10/15/25 by tier), capped at 100.
13. **Final decision logic** — the "one sentence rule" gate first (reject
    anything that fails it, no matter the score), then applies the exact
    apply / manual_check / reject rules from the spec. Only score ≥ 70 survives
    as apply/manual_check.
14. **Append Filtered Jobs** / **Append Rejected Jobs** — everything lands in one
    of these two sheets with full reasoning attached.
15. **Instant alert** — score ≥ 85 *and* decision = apply triggers a Claude-
    generated cover letter draft plus a Telegram/Gmail alert immediately.
16. **Daily digest** — after all jobs are processed, builds one message with the
    top 5 jobs, the manual-check list, and a rejected-reason breakdown (US only /
    EU only / hybrid / timezone risk / other region-restricted), sent via
    Telegram and/or Gmail.

## Notes on fidelity to the spec

- Async and salary scoring use the **highest matching tier**, not a sum of every
  tier, since the spec lists them as an ascending scale.
- The AI eligibility prompt and cover-letter prompt are sent to Claude verbatim
  (word-for-word from the spec, just interpolated with the job's own title,
  company, and description).
- This workflow never auto-submits an application. "Apply immediately" only
  means: mark `application_status = Ready to Apply` in Filtered Jobs and draft a
  cover letter for your review — you still send it yourself.
- Third-party job board APIs/RSS feeds can change their response shape without
  notice; if a source stops returning expected fields, check the `Normalize Job
  Data` node first.
