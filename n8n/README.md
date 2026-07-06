# Daily Taiwan Eligible Async Remote Job Hunter

An n8n workflow that runs every day at **8:30 AM Asia/Taipei** and pulls
remote jobs from multiple job boards, hard-filters them for genuine
Taiwan/worldwide eligibility, runs an AI (Claude) eligibility + fit check,
scores each job 0–100, logs everything to Google Sheets, and sends you a
Telegram digest — with an instant alert and AI-drafted cover letter for any
job scoring 85+.

Import `taiwan-async-remote-job-hunter.json` into n8n (**Workflows → Import
from File**), then follow the setup steps below before activating it.

## 1. Google Sheet

Create a Google Sheet named **"Taiwan Eligible Async Remote Jobs"** with
four tabs and these exact header rows (row 1):

**Raw Jobs**
`date_found | source | job_title | company | job_url | location_text | salary | description | posted_date | tags | job_id`

**Filtered Jobs**
`date_found | score | job_title | company | job_url | salary | taiwan_eligible | taiwan_reason | location_risk | timezone_risk | contractor_possible | async_level | role_category | why_good_fit | decision | application_status | notes | cover_letter_draft`

(`cover_letter_draft` is one extra column beyond the original spec, added so
step 13's AI-drafted cover letters have somewhere to land.)

**Applied**
`date_applied | job_title | company | job_url | cover_letter_used | status | follow_up_date | notes`

**Rejected**
`date_rejected | job_title | company | job_url | reason`

Copy the Sheet's ID (the long string in its URL) and replace every
`YOUR_GOOGLE_SHEET_ID` placeholder in the workflow — easiest way is to open
each Google Sheets node once in the n8n UI and re-pick the document/sheet
from the resource-locator dropdown, which will overwrite the placeholder
for you.

## 2. Credentials to configure in n8n

| Credential | Used by | Notes |
|---|---|---|
| **Google Sheets OAuth2** | all Google Sheets nodes | Standard n8n Google Sheets OAuth credential. |
| **Anthropic API Key** (generic "HTTP Header Auth", header name `x-api-key`) | `Claude: Taiwan Eligibility Check`, `Claude: Draft Cover Letter` | Get a key from the Anthropic console. The workflow calls the Messages API directly via HTTP Request so it works with any n8n version without needing the LangChain nodes installed. |
| **Telegram Bot API** | `Instant Alert (score >= 85)`, `Send Daily Digest (Telegram)` | Create a bot via @BotFather, then replace `YOUR_TELEGRAM_CHAT_ID` in both Telegram nodes with your chat ID. |
| **Gmail OAuth2** *(optional, disabled by default)* | `Send Daily Digest (Gmail — alternate)` | The spec allows Telegram *or* Gmail for the daily digest. The Gmail node is wired up but left disabled — enable it and set your email address in place of `YOUR_EMAIL@example.com` if you'd rather get the digest by email instead of (or alongside) Telegram. |

## 3. What the workflow does, step by step

1. **Trigger** — Schedule Trigger, cron `30 8 * * *`, workflow timezone set
   to `Asia/Taipei` in the workflow settings.
2. **Fetch sources** — parallel branches per job board:
   - **Remote OK, Remotive, We Work Remotely (RSS), Himalayas** use real
     public APIs/RSS feeds and should work out of the box.
   - **Wellfound, Dynamite Jobs, YC Work at a Startup, Otta** have no
     public API or RSS feed — they're JS-rendered single-page apps. The
     included nodes make a best-effort attempt to pull an embedded
     `__NEXT_DATA__` JSON blob out of the raw HTML, but these sites change
     often and may require JS rendering to scrape reliably. If a source
     returns zero jobs, that's expected — consider swapping its `HTTP
     Request` node for an Apify actor, Browserless, or ScrapingBee call for
     more reliable results. The rest of the pipeline (dedupe, filtering,
     scoring, sheets, digest) works identically regardless of which
     sources actually return data.
3. **Normalize** — each source's Code node maps its raw response to the
   common schema (`job_title`, `company`, `job_url`, `source`,
   `location_text`, `salary`, `description`, `posted_date`, `tags`).
4. **Merge All Sources** — unions all 8 branches into one list.
5. **Keyword Match Filter** — keeps only jobs whose title/description/tags
   match one of the spec's search keywords (async remote, worldwide
   remote, remote data analyst, etc.).
6. **Dedupe Jobs** — reads the existing `Raw Jobs` sheet and drops any job
   whose `job_url` (or, failing that, `company + job_title`) already
   exists.
7. **Append to Raw Jobs** — every new, deduped job is logged here
   regardless of what happens next.
8. **Taiwan Hard Filter** — regex/keyword pass on the reject/keep phrase
   lists from the spec. Clear rejects (e.g. "US only", "hybrid") skip
   straight to the `Rejected` sheet and never reach the AI step.
9. **Loop Jobs for AI Check** (`Split In Batches`, size 1) → **Claude:
   Taiwan Eligibility Check** — sends the exact prompt from the spec to
   Claude and parses its JSON verdict (`taiwan_eligible`, `location_risk`,
   `timezone_risk`, `contractor_possible`, `decision`).
10. **Async Filter** and **Role Fit Classification** — keyword-based, per
    the spec's async and role-fit keyword lists.
11. **Scoring Engine** — implements the point system from the spec
    (Taiwan eligibility up to 40, async fit up to 25, role fit up to 25,
    salary fit up to 25, capped at 100) and also enforces the **one-sentence
    rule** as a hard gate: if the job fails Taiwan eligibility, has high
    location/timezone risk, or fails the async filter, its score is forced
    to 0 regardless of other points. It also applies the step-14 final
    decision logic (`ready_to_apply` / `review` / `manual_check` /
    reject).
12. **Score >= 70?** — below 70 goes to `Rejected`; 70+ is appended to
    `Filtered Jobs`.
13. **Score >= 85?** — fires an instant Telegram alert and generates an
    AI cover-letter draft (Claude, spec's exact prompt/background/tone),
    writing the draft back into the `Filtered Jobs` row.
14. Loop continues until all jobs in the batch are processed, then:
15. **Daily Digest** — reads today's rows from `Filtered Jobs` and
    `Rejected`, builds the exact message format from the spec (top 5 jobs
    by score, manual-check list, rejected-reason counts), and sends it via
    Telegram (and optionally Gmail).

## 4. Tuning

- Score weights and keyword lists live in plain JavaScript inside the
  `Async Filter`, `Role Fit Classification`, and `Scoring Engine` Code
  nodes — edit them directly rather than rebuilding the workflow.
- The `Loop Jobs for AI Check` batch size is 1 to stay well under
  Anthropic's rate limits; raise it if you fetch a lot of jobs per day and
  want to speed up the run (n8n will call the AI node once per item in the
  batch regardless, so this mainly affects loop overhead, not API call
  count).
- The workflow does **not** auto-apply to anything — `application_status`
  values (`ready_to_apply`, `review`, `manual_check`) are recommendations
  for you to act on manually; use the `Applied` sheet to track what you've
  actually submitted.
