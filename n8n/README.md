# Daily Taiwan Eligible Async Remote Job Hunter

An n8n workflow that pulls remote job listings, filters them down to roles Jerry can
actually do while physically based in Taiwan (no relocation, no foreign work
authorization, no overnight hours), scores what's left, and sends a daily digest.

Import `taiwan-async-remote-job-hunter.workflow.json` into n8n (Workflows → Import
from File) and follow the setup steps below.

## What it does

1. **Trigger** — Schedule Trigger, cron `30 8 * * *`, workflow timezone set to
   `Asia/Taipei` (08:30 daily).
2. **Fetch** — RemoteOK (`/api`) and Remotive (`/api/remote-jobs`) via their public
   JSON APIs, plus We Work Remotely via RSS (Programming, Sales & Marketing, All
   Other Remote categories). Five more sources — Wellfound, Himalayas, Dynamite
   Jobs, Y Combinator Work at a Startup, Otta — are wired in as **disabled**
   placeholder HTTP Request nodes (see "Best-effort sources" below).
3. **Normalize** — one Code node per source maps raw fields to a common shape:
   `job_title, company, job_url, source, location_text, salary, description,
   posted_date, tags`.
4. **Keyword relevance filter** — keeps jobs matching the brief's async/location
   phrases or target-role phrases (data analyst, data scientist, BI, marketing
   analyst, growth analyst, AI marketing, automation, etc).
5. **Dedupe** — reads existing rows from the `Raw Jobs` sheet, drops anything whose
   `job_url` (or `company + job_title` as a backup key) already exists, then appends
   the new unique rows to `Raw Jobs`.
6. **Taiwan hard filter** — deterministic keyword gate. Rejects immediately on
   phrases like "US only", "must be based in", "hybrid", "onsite", "fixed EST
   hours", etc. — *unless* a strong worldwide/contractor phrase is also present, in
   which case the job is passed on to the AI check instead of auto-rejected
   (ambiguous language shouldn't be killed by a keyword collision).
7. **AI Taiwan eligibility check** — calls Claude with the exact eligibility prompt
   from the brief and parses the returned JSON (`taiwan_eligible`, `reason`,
   `location_risk`, `timezone_risk`, `contractor_possible`, `decision`). Rejects if
   `decision == "reject"`.
8. **Async & timezone filter** — keyword gate for async/flexible-schedule language
   vs. "must overlap full business day" / fixed-hours language.
9. **Role fit scoring** — flags primary vs. adjacent role matches.
10. **Scoring** — 0–100, combining Taiwan-eligibility points (40/30/20/0), async-fit
    points (25/20/15/10), role-fit points (25/15/0), and salary points (25/15/10/0).
    Jobs below 70 are rejected.
11. **Final decision logic** — implements the brief's apply-now / manual-check /
    reject rules, plus a final "one-sentence rule" safety net (auto-rejects if
    `location_risk` or `timezone_risk` is `high`, or `taiwan_eligible == "no"`,
    regardless of score).
12. **Sheets** — appends to `Filtered Jobs` (kept jobs) or `Rejected` (dropped
    jobs, with a reason).
13. **Instant alert + cover letter** — for score ≥ 85, drafts a cover letter with
    Claude (using Jerry's background from the brief) and sends an instant Telegram
    message with the job details + draft.
14. **Daily digest** — aggregates everything processed that run into the exact
    message format from the brief: top jobs, manual-check list, and rejection
    counts by category (US only / EU only / hybrid / timezone risk).

## Setup

1. **Google Sheet** — create a spreadsheet named `Taiwan Eligible Async Remote
   Jobs` with 4 tabs and these exact columns:

   | Tab | Columns |
   |---|---|
   | `Raw Jobs` | `date_found, source, job_title, company, job_url, location_text, salary, description, posted_date, tags, job_id` |
   | `Filtered Jobs` | `date_found, score, job_title, company, job_url, salary, taiwan_eligible, taiwan_reason, location_risk, timezone_risk, contractor_possible, async_level, role_category, why_good_fit, decision, application_status, notes` |
   | `Applied` | `date_applied, job_title, company, job_url, cover_letter_used, status, follow_up_date, notes` |
   | `Rejected` | `date_rejected, job_title, company, job_url, reason` |

   `Applied` is intentionally not written by the workflow — moving a row there
   means Jerry decided to apply, which is a manual action.

2. **Config node** — open the **Config** node (right after the trigger) and set:
   - `google_sheet_id` — your spreadsheet ID (the long string in its URL).
   - `telegram_chat_id` — your Telegram chat ID (skip if using Gmail instead; see
     below).
   - `anthropic_model` — defaults to `claude-sonnet-5`; change if you prefer a
     different Claude model.

3. **Credentials to attach:**
   - All **Google Sheets** nodes (`Get Existing Raw Jobs`, `Append To Raw Jobs
     Sheet`, `Append To Filtered Jobs Sheet`, `Append To Rejected Sheet`) — Google
     Sheets OAuth2, and re-pick the spreadsheet/tab in each node's dropdown (n8n
     needs this once per node after import even though the ID is templated).
   - **Claude: Taiwan Eligibility Check** and **Claude: Draft Cover Letter** — HTTP
     Header Auth credential with header name `x-api-key`, value = your Anthropic
     API key. (`anthropic-version` and `content-type` headers are already set on
     the nodes.)
   - **Send Instant Telegram Alert** / **Send Daily Digest (Telegram)** — Telegram
     API credential (bot token). If you'd rather use email, enable the disabled
     **Gmail** alternative nodes instead (attach Gmail OAuth2, set `sendTo`), and
     disable the Telegram nodes.

4. Turn the workflow **Active**.

## Best-effort sources (disabled by default)

Wellfound, Himalayas, Dynamite Jobs, Y Combinator Work at a Startup, and Otta don't
expose a stable public API or RSS feed — Wellfound and Work at a Startup require a
logged-in session, and the others are JS-rendered listing pages that a plain HTTP
GET won't render. Rather than ship something that silently returns nothing (or
breaks on their next markup change), these five are wired in as disabled HTTP
Request placeholders feeding a normalizer (`Normalize Best-Effort Sources`) that
already expects the standard job shape. To use one, swap the placeholder for a real
scraper (an Apify actor, ScrapingBee/Browserless, or a session-cookie-authenticated
request), map its output to `{job_title, company, job_url, source, location_text,
salary, description, posted_date, tags}`, and re-enable the node.

## Design notes / simplifications

- **Search keywords**: implemented as an OR-match against the brief's async/location
  phrases and role phrases, rather than the literal concatenated phrases (e.g. "remote
  data analyst") — job posts rarely contain that exact string, and every source here
  is a remote-only board to begin with.
- **Salary parsing** is regex-based best-effort extraction from free text; treat the
  monthly-equivalent estimate as approximate.
- **"Apply immediately"** does not submit an application automatically — no workflow
  should do that unsupervised. Score ≥ 85 jobs get `application_status =
  "ready_to_apply"`, an instant alert, and a cover letter draft; Jerry still applies
  by hand and then logs it in the `Applied` sheet.
