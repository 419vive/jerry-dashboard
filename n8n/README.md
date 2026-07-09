# Daily Taiwan Eligible Async Remote Job Hunter (n8n)

An n8n workflow that finds async, global-remote jobs Jerry can actually do
while living in Taiwan — no relocation, no foreign work authorization, no
fixed midnight hours — scores them, logs them to Google Sheets, and sends a
daily digest.

Import `taiwan-async-remote-job-hunter.json` into n8n (Workflows → Import
from File), then follow the setup steps below before activating it.

## What it does

1. **Trigger** — runs every day at 08:30 `Asia/Taipei`.
2. **Sources** — pulls jobs from RemoteOK and Remotive (free JSON APIs), and
   We Work Remotely + Himalayas (RSS). Wellfound, Dynamite Jobs, YC Work at a
   Startup, and Otta have no public API/RSS and actively block plain
   scraping (Cloudflare / login walls) — see "Sources without an API" below.
3. **Normalize** — maps every source into one shape: `job_title`, `company`,
   `job_url`, `source`, `location_text`, `salary`, `description`,
   `posted_date`, `tags`, `job_id`.
4. **Keyword match** — keeps only jobs matching the async/global/data/AI-marketing
   keyword list from the spec.
5. **Dedupe** — checks `job_url` (and `company + job_title` as a backup key)
   against everything already logged in the *Raw Jobs* sheet, plus dupes
   within the same run.
6. **Hard filter** — rejects jobs with restrictive language ("US only",
   "must be based in", "hybrid", "onsite", ...) unless they also contain
   worldwide/anywhere/contractor language.
7. **AI Taiwan-eligibility check** — calls Claude (Anthropic API) with the
   exact eligibility prompt from the spec and gets back
   `taiwan_eligible / reason / location_risk / timezone_risk /
   contractor_possible / decision` as JSON.
8. **Async filter** — rejects "strict 9 to 5" / fixed-region-hours jobs,
   scores async-friendliness otherwise.
9. **Role fit** — tags data/BI/growth/AI-marketing/automation roles as
   primary fits, other marketing/data/analytics roles as adjacent.
10. **Scoring** — 0–100: Taiwan fit (≤40) + async fit (≤25) + role fit (≤25)
    + salary (≤25). Keeps score ≥ 70.
11. **Google Sheets** — appends to `Raw Jobs`, `Filtered Jobs`, and
    `Rejected` tabs (see schema below). `Applied` is a tab you maintain by
    hand — the workflow never writes to it, and it never submits an
    application on your behalf.
12. **Daily digest** — after every branch has finished writing, re-reads
    today's rows from `Filtered Jobs` and `Rejected` and sends one Telegram
    message with the top jobs, a manual-check list, and rejection counts
    (US only / EU only / hybrid / timezone).
13. **Cover letter (score ≥ 85)** — an instant Telegram alert, plus a Claude
    -drafted cover letter saved as a Gmail **draft** (never sent
    automatically — you review and send it yourself).
14. **Final decision logic** — combines eligibility + score + risk into
    `apply_immediately` / `manual_check` / `reject`, written to the
    `decision` column. This sets a status for you to act on; it does not
    auto-apply to anything.
15. **One-sentence rule** — the Final Decision Logic node force-rejects any
    job that fails "I can do this job while living in Taiwan, without
    moving, without foreign work authorization, and without destroying my
    sleep schedule" regardless of score.

## Setup

### 1. Google Sheet

Create a spreadsheet called **Taiwan Eligible Async Remote Jobs** with 4
tabs and these exact header rows (order matters — the workflow appends by
column position):

**Raw Jobs**
`date_found | source | job_title | company | job_url | location_text | salary | description | posted_date | tags | job_id`

**Filtered Jobs**
`date_found | score | job_title | company | job_url | salary | taiwan_eligible | taiwan_reason | location_risk | timezone_risk | contractor_possible | async_level | role_category | why_good_fit | decision | application_status | notes`

**Applied** (maintained by you, not written by this workflow)
`date_applied | job_title | company | job_url | cover_letter_used | status | follow_up_date | notes`

**Rejected**
`date_rejected | job_title | company | job_url | reason`

Copy the spreadsheet ID out of its URL.

### 2. Credentials to create in n8n

| Credential | Used by | Notes |
|---|---|---|
| Google Sheets OAuth2 | all "Google Sheets" nodes | grant access to the sheet above |
| Telegram API | "Send Instant Alert", "Send Daily Digest" | create a bot via @BotFather, get your `chatId` by messaging the bot and hitting `https://api.telegram.org/bot<token>/getUpdates` |
| Gmail OAuth2 | "Create Cover Letter Draft" | only needs draft-creation scope |
| HTTP Header Auth ("Anthropic API Key") | both "Claude ..." HTTP Request nodes | header name `x-api-key`, value = your Anthropic API key |

After import, open each node listed above and point it at your credential —
the JSON ships with placeholder credential IDs (`GOOGLE_SHEETS_CRED_ID`,
`TELEGRAM_CRED_ID`, `GMAIL_CRED_ID`, `ANTHROPIC_CRED_ID`) that won't resolve
until you do.

### 3. Config node

Open the **Config** node (right after the trigger) and fill in:
- `telegramChatId`
- `googleSheetId`
- `anthropicModel` (defaults to `claude-sonnet-5` — change if you use a
  different model)
- `jerryEmail` (defaults to `419vive@gmail.com`)
- `keepThreshold` / `alertThreshold` if you want different cutoffs than
  70 / 85.

### 4. Sources without an API

Wellfound, Dynamite Jobs, YC Work at a Startup, and Otta don't have a public
RSS/API and block naive scraping. The **Other Sources Placeholder** code
node returns `[]` so the workflow runs end-to-end without them. To add one:
point an HTTP Request node at a scraping service (Apify actor, Browserless,
ScrapingBee, etc.) and have that placeholder node return objects shaped
like the other `Normalize *` nodes (`job_title`, `company`, `job_url`,
`source`, `location_text`, `salary`, `description`, `posted_date`, `tags`).

### 5. Activate

Run it once manually from the n8n editor to confirm every credential
resolves and the sheet columns line up, then toggle the workflow **Active**.

## Notes / limitations

- Google Sheets "Get Row(s)" reads the whole tab on every run to dedupe and
  build the digest. Fine for personal-scale volume; if the sheet grows into
  the tens of thousands of rows, consider archiving old rows periodically.
- `Score >= 85` and `apply_immediately` are decision aids, not automatic job
  applications — you still click "Apply" yourself.
- Google Sheets node parameter names can shift slightly between n8n
  versions; if the "Get Existing Job URLs" / "Get Today's ..." nodes error
  on import, re-pick the "Get Row(s)" operation in the node UI and re-save.
