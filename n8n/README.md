# Daily Taiwan Eligible Async Remote Job Hunter (n8n)

Imports as a single n8n workflow: `daily-taiwan-eligible-async-remote-job-hunter.json`.

It runs every day at **8:30 AM Asia/Taipei**, pulls jobs from several remote-job
sources, hard-filters and AI-checks each one for genuine Taiwan eligibility,
scores the survivors, writes everything to Google Sheets, sends a daily
digest, fires an instant alert for standout matches, and drafts a cover
letter for anything scoring 85+.

The one rule everything else serves: **a job only survives if Jerry can do
it while physically living in Taiwan** — no relocation, no US/EU/UK/Canada/
Australia work authorization, no fixed midnight hours.

## How to import

1. In n8n: **Workflows → Import from File** → select
   `daily-taiwan-eligible-async-remote-job-hunter.json`.
2. The workflow imports **inactive** and with placeholder values — wire up
   credentials and IDs below before activating it.

## Credentials to create/attach

| Credential name (as referenced in the workflow) | Type | Used by |
|---|---|---|
| `Google Sheets account` | Google Sheets OAuth2 | Get Existing Job URLs, Append Raw Jobs, Append Rejected, Append Filtered Jobs, Save Cover Letter To Notes |
| `Anthropic API` | Header/Anthropic API key | AI Taiwan Eligibility Check, Generate Cover Letter |
| `Telegram account` | Telegram Bot API | Send Instant Alert, Send Daily Digest (Telegram) |
| `Gmail account` | Gmail OAuth2 | Send Daily Digest (Gmail) |

n8n workflow exports never contain credential secrets — after import, open
each node listed above and pick (or create) the matching credential.

## Values to replace after import

- Every `YOUR_GOOGLE_SHEET_ID` (5 nodes) → the spreadsheet ID from your
  Google Sheet's URL.
- `YOUR_TELEGRAM_CHAT_ID` (2 nodes) → your Telegram chat ID.
- `YOUR_EMAIL@example.com` (Gmail digest node) → the inbox you want the
  daily digest sent to.

## Google Sheet setup

Create a spreadsheet named **Taiwan Eligible Async Remote Jobs** with four
tabs and these exact header rows (the workflow auto-maps columns by name):

**Raw Jobs**
`date_found | source | job_title | company | job_url | location_text | salary | description | posted_date | tags | job_id`

**Filtered Jobs**
`date_found | score | job_title | company | job_url | salary | taiwan_eligible | taiwan_reason | location_risk | timezone_risk | contractor_possible | async_level | role_category | why_good_fit | decision | application_status | notes`

**Applied** *(user-maintained — the workflow never writes here; see note below)*
`date_applied | job_title | company | job_url | cover_letter_used | status | follow_up_date | notes`

**Rejected**
`date_rejected | job_title | company | job_url | reason`

## Pipeline

1. **Schedule Trigger** — cron `30 8 * * *`, workflow timezone `Asia/Taipei`.
2. **Config** — holds the keyword/reject/keep phrase lists in one place so
   downstream Code nodes stay short.
3. **Fetch sources**
   - Remote OK (`remoteok.com/api`) and Remotive (`remotive.com/api/remote-jobs`) — JSON APIs.
   - We Work Remotely (4 category RSS feeds) and Himalayas — RSS.
   - Wellfound, Dynamite Jobs, YC Work at a Startup, Otta — **disabled
     placeholder nodes**. None of these expose a stable public API or RSS
     feed; their listing pages are JS-rendered and/or bot-protected. To
     enable one, point its URL at a scraping proxy (ScraperAPI, Browserless,
     Firecrawl, etc.) that returns rendered HTML, add an HTML Extract node,
     re-enable the node, and wire its output into "Merge All Sources".
4. **Normalize + Keyword Match** — maps every source's shape onto the
   common job schema and drops anything that doesn't match the keyword list.
5. **Dedupe** — reads existing `Raw Jobs` rows, skips anything whose
   `job_url` (primary) or `company + job_title` (backup) already exists.
6. **Append Raw Jobs** — every deduped job is logged, filtered or not.
7. **Hard Filter (Keyword Rules)** — regex reject/keep phrase pass. Clear
   rejections (e.g. "US only", "hybrid", "must be based in") go straight to
   the Rejected sheet without spending an AI call.
8. **AI Taiwan Eligibility Check** — calls Claude with the exact prompt from
   the spec, returns `taiwan_eligible / reason / location_risk /
   timezone_risk / contractor_possible / decision` as JSON.
9. **Async Filter** — flags `async_level` (high/medium/low/unclear) from
   the async/anti-async phrase lists.
10. **Role Fit + Score** — computes the 0–100 score (Taiwan eligibility +
    async fit + role fit + salary) exactly per the point table in the spec,
    and derives `decision` (`apply_now` / `manual_check` / `reject`) using
    the Final Decision Logic and the one-sentence hard gate.
11. **Score ≥ 70?** — passes go to Filtered Jobs; everything else is logged
    to Rejected with its reason.
12. **Score ≥ 85?** — triggers an instant Telegram alert and a Claude-drafted
    cover letter (saved into the `notes` column of the matching Filtered
    Jobs row).
13. **Build Daily Digest** — aggregates the day's filtered + rejected jobs
    into the digest format from the spec (top jobs, manual-check list,
    rejection-reason counts) and sends it via Telegram and Gmail.

## Known limitations (be upfront about these)

- **Salary parsing is heuristic.** It pulls the largest number out of the
  `salary` field/text; it doesn't distinguish annual vs. monthly figures
  reliably across sources. Spot-check before trusting the salary score.
- **Taiwan eligibility ultimately depends on the AI check.** The hard
  keyword filter only catches explicit phrases; ambiguous postings are
  passed to Claude, and the prompt asks it to default to `unclear` rather
  than guess — those land in `manual_check`, not auto-reject or auto-apply.
- **The workflow does not submit applications.** `decision = apply_now`
  means "Jerry should apply" — it's a recommendation surfaced in the digest
  and the sheet, not an automated submission. The `Applied` sheet is for
  Jerry to log manually once he's actually applied.
- **Wellfound / Dynamite Jobs / YC Work at a Startup / Otta are disabled by
  default** for the reasons above — enable them once a scraping source is
  wired in.
