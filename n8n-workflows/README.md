# Daily Taiwan Eligible Async Remote Job Hunter

n8n workflow that pulls remote job postings daily, hard-filters out anything that
isn't workable from Taiwan, runs an AI (Claude) eligibility + scoring pass on the
rest, logs everything to Google Sheets, and sends a Telegram digest every morning
at 8:30 AM Asia/Taipei.

Import `taiwan-eligible-async-remote-job-hunter.json` into n8n (Workflows → Import
from File) to get started.

## What's wired up automatically

- **Sources with a public API/RSS feed** (fully implemented): Remote OK, Remotive,
  We Work Remotely (RSS), Himalayas.
- **Sources with no public API** (Wellfound, Dynamite Jobs, Work at a Startup,
  Otta): added as **disabled** HTTP Request stub nodes so they're visible in the
  graph. None of them expose a public jobs API or RSS feed, so pulling from them
  requires either a scraping service (e.g. an Apify actor) or manual weekly
  review. Enable and wire up the ones you care about once you've picked a
  scraping approach.
- Keyword search filter, dedupe against previously-seen `job_url`s, the Taiwan
  eligibility hard filter (reject/keep phrase lists), an AI eligibility check via
  the Anthropic API, async/role-fit/salary scoring, the keep/reject/manual-check
  decision logic, cover-letter drafting for scores ≥ 85, and the daily digest
  message.

## One-time setup

1. **Google Sheet** — create a spreadsheet called `Taiwan Eligible Async Remote
   Jobs` with four tabs and these header rows (row 1):
   - `Raw Jobs`: `date_found, source, job_title, company, job_url, location_text, salary, description, posted_date, tags, job_id`
   - `Filtered Jobs`: `date_found, score, job_title, company, job_url, salary, taiwan_eligible, taiwan_reason, location_risk, timezone_risk, contractor_possible, async_level, role_category, why_good_fit, decision, application_status, notes`
   - `Applied`: `date_applied, job_title, company, job_url, cover_letter_used, status, follow_up_date, notes`
   - `Rejected`: `date_rejected, job_title, company, job_url, reason`

   The workflow only writes to `Raw Jobs`, `Filtered Jobs`, and `Rejected`.
   `Applied` is for you to fill in by hand as you actually apply.

2. **Open the `Config` node** (right after the trigger) and set:
   - `GOOGLE_SHEET_ID` — the ID from your sheet's URL.
   - `TELEGRAM_CHAT_ID` — your Telegram chat/user ID (message
     [@userinfobot](https://t.me/userinfobot) to get it).
   - `ANTHROPIC_MODEL` — defaults to `claude-sonnet-5`; change if you want a
     different model.

3. **Credentials** (assign these in the n8n UI after import):
   - Every Google Sheets node → a `Google Sheets` OAuth2 credential.
   - `AI Taiwan Eligibility Check` and `AI Generate Cover Letter` (both HTTP
     Request nodes) → an **HTTP Header Auth** credential with header name
     `x-api-key` and your Anthropic API key as the value.
   - `Send Instant Alert Telegram` and `Send Daily Digest Telegram` → a
     `Telegram` API credential (bot token from [@BotFather](https://t.me/BotFather)).
   - Optional: enable the disabled `Send Instant Alert Gmail (alt)` / `Send
     Daily Digest Gmail (alt)` nodes instead of/alongside Telegram, and assign a
     Gmail OAuth2 credential.

4. Turn the workflow **Active**. It's scheduled for 8:30 AM `Asia/Taipei` daily
   (set in the trigger node and workflow settings).

## How a job flows through the pipeline

1. Fetch each source → normalize into a common shape (`job_title`, `company`,
   `job_url`, `source`, `location_text`, `salary`, `description`, `posted_date`,
   `tags`).
2. **Keyword filter** — keep only postings matching the async/remote/role
   keyword list.
3. **Dedupe** — drop anything whose `job_url` (or `company`+`job_title` as a
   backup key) is already in the `Raw Jobs` sheet.
4. **Append to Raw Jobs** — every new, deduped posting is logged here
   regardless of what happens next.
5. **Hard filter** — regex reject list (`US only`, `hybrid`, `must be based
   in`, ...) vs. keep list (`worldwide remote`, `EOR supported`, ...). Jobs that
   hit a reject phrase skip the AI call entirely and go straight to `Rejected`
   (saves API cost).
6. **AI Taiwan eligibility check** — the exact prompt from the spec, sent to
   Claude via the Anthropic Messages API, returns
   `taiwan_eligible / reason / location_risk / timezone_risk /
   contractor_possible / decision` as JSON.
7. **Score And Decide** — one Code node computes:
   - Taiwan eligibility points (40 / 30 / 20 / 0)
   - Async fit points (25 / 20 / 15 / 10, highest matching tier)
   - Role fit points (25 core / 15 adjacent / 0)
   - Salary points (25 / 15 / 10 / 0, tries to detect monthly vs. annual
     figures)
   - Final `decision`: `apply`, `manual_check`, `keep_review`, or `reject`,
     following the rules in the spec (score ≥ 70 gate, ≥ 85 + eligible + low
     risk → auto-apply candidate, etc.)
8. Jobs scoring **< 70 or otherwise rejected** → `Rejected` sheet.
   Jobs scoring **≥ 70** → `Filtered Jobs` sheet with `application_status` set
   from the decision.
9. Jobs scoring **≥ 85** additionally get a tailored cover letter draft
   (Claude) and an instant Telegram alert with the job details + draft.
10. At the end of the run, every job processed today is aggregated into one
    **daily digest** message (top jobs by score, manual-check list, rejected
    counts by category) and sent via Telegram at 8:30 AM.

## Notes and tuning knobs

- The keyword filter includes a few generic fallback terms (`data analyst`,
  `remote`, etc.) beyond the exact phrases in the spec, so postings titled
  simply "Data Analyst" at a remote-friendly company aren't missed. Trim the
  `KEYWORDS` array in the **Keyword Filter** node if it's too broad.
- Reject-reason categorization in the digest (US only / EU only / hybrid /
  timezone) is derived from the hard-filter match text; AI-based rejections
  (wrong eligibility, high risk, low score) still count toward "Total
  rejected" but aren't split into those four buckets.
- Remotive/Himalayas are fetched without a `search` query param (broad pull),
  relying on the keyword filter downstream. Add a `search`/`category` query
  parameter to the `Fetch Remotive` / `Fetch Himalayas` HTTP Request nodes if
  you want to narrow the pull itself.
