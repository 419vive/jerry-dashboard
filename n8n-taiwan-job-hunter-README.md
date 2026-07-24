# Daily Taiwan Eligible Async Remote Job Hunter (n8n workflow)

Importable n8n workflow: **`n8n-taiwan-job-hunter-workflow.json`**

Finds async, global-remote jobs Jerry can legally and practically do while
living in Taiwan — no relocation, no US/EU/UK/CA/AU work authorization, no
fixed midnight hours — and reports the best matches every morning.

## What it does

Runs daily at **08:30 Asia/Taipei**:

1. **Fetch** — pulls jobs from RemoteOK, Remotive, Himalayas, and We Work
   Remotely via their public API/RSS feeds, plus a best-effort HTML scrape of
   Wellfound, Dynamite Jobs, YC Work at a Startup, and Otta (these four have
   no public search API — see *Known limitations* below).
2. **Normalize** — maps every job to `job_title, company, job_url, source,
   location_text, salary, description, posted_date, tags`.
3. **Dedupe** — against the `Raw Jobs` sheet by `job_url`, falling back to
   `company + job_title`.
4. **Hard filter** — rejects on sight ("US only", "hybrid", "onsite", "must
   be based in", ...) unless a worldwide/APAC/contractor override phrase is
   also present.
5. **AI Taiwan-eligibility check** — Claude (Haiku) is prompted with the
   exact eligibility rubric from the spec and returns
   `taiwan_eligible / reason / location_risk / timezone_risk /
   contractor_possible / decision` as JSON.
6. **Async filter** — rejects strict 9-5 / fixed-region-hours postings.
7. **Role fit + scoring** — tags role category (data / BI / AI marketing /
   growth / automation) and scores 0-100 (Taiwan eligibility 40, async fit
   25, role fit 25, salary 25, capped at 100). Only score ≥ 70 survives.
8. **Sheets** — every fetched job lands in `Raw Jobs`; survivors land in
   `Filtered Jobs`; anything rejected at any stage lands in `Rejected` with
   the specific reason.
9. **Instant alert + cover letter** — score ≥ 85 triggers a Telegram alert
   immediately and a Claude (Sonnet) cover-letter draft, written back into
   the `notes` column of that job's `Filtered Jobs` row.
10. **Daily digest** — one Telegram message and one Gmail email every
    morning: top jobs (sorted by score), a manual-check list (Taiwan
    eligibility unclear but score ≥ 85), and a rejection-reason breakdown
    (US-only / EU-only / hybrid / timezone / other counts).

## Setup

### 1. Google Sheet

Create a spreadsheet named **"Taiwan Eligible Async Remote Jobs"** with four
tabs and these exact header rows (row 1):

- **Raw Jobs** — `date_found, source, job_title, company, job_url,
  location_text, salary, description, posted_date, tags, job_id`
- **Filtered Jobs** — `date_found, score, job_title, company, job_url,
  salary, taiwan_eligible, taiwan_reason, location_risk, timezone_risk,
  contractor_possible, async_level, role_category, why_good_fit, decision,
  application_status, notes`
- **Applied** — `date_applied, job_title, company, job_url, cover_letter_used,
  status, follow_up_date, notes`
- **Rejected** — `date_rejected, job_title, company, job_url, reason`

Copy the spreadsheet ID into every `documentId.value` field in the workflow
JSON (search-replace `REPLACE_WITH_GOOGLE_SHEET_ID`), or re-point each
Google Sheets node at your sheet from the n8n editor after import.

> The `Parse AI Eligibility JSON` node outputs `reason`, which is written to
> the `taiwan_reason` column via `columns.mappingMode: autoMapInputData`.
> Google Sheets auto-mapping matches by column *name* — since the item field
> is `reason` and the sheet column is `taiwan_reason`, either rename the
> sheet column to `reason`, or open the node in n8n and manually map
> `reason → taiwan_reason` once imported.

### 2. Credentials (n8n → Credentials)

| Credential | Type | Used by |
|---|---|---|
| Google Sheets account | Google Sheets OAuth2 | all 8 Google Sheets nodes |
| Anthropic API Key | Generic **Header Auth**, header name `x-api-key`, value = your Anthropic API key | `AI Taiwan Eligibility Check`, `Generate Cover Letter` |
| Telegram Bot | Telegram API (bot token) | `Send Instant Alert`, `Send Daily Digest (Telegram)` |
| Gmail account | Gmail OAuth2 | `Send Daily Digest (Gmail)` |

After import, open each node above and attach the matching credential
(the JSON ships with placeholder credential IDs).

### 3. Replace placeholders

- `REPLACE_WITH_GOOGLE_SHEET_ID` → your Sheet ID (8 nodes)
- `REPLACE_WITH_TELEGRAM_CHAT_ID` → your Telegram chat ID (2 nodes) — or
  delete the Telegram nodes if you only want Gmail
- Gmail `sendTo` defaults to `419vive@gmail.com`

### 4. Activate

Workflow timezone is set to `Asia/Taipei` and the schedule is
`30 8 * * *` (08:30 daily). Toggle the workflow **Active** once credentials
and sheet IDs are in place.

## Known limitations (read before relying on this)

- **Wellfound, Dynamite Jobs, YC Work at a Startup, Otta** have no public
  job-search API. Their fetch nodes do a raw HTTP GET + regex scrape of the
  page HTML, which will likely return **0 results** if the listing is
  client-side rendered (common for all four). Swap those `Fetch *` HTTP
  Request nodes for a scraping service (Firecrawl, ScrapingBee, Browserless)
  if you need real coverage from these sources — keep the output in a `data`
  field (raw HTML string) so the paired `Normalize *` node keeps working.
- The AI eligibility check and scoring are heuristics, not guarantees —
  spot-check a few `Filtered Jobs` and `Rejected` rows against the actual
  job posts after the first few runs and tune the keyword lists inside the
  `Hard Filter Keywords`, `Async Filter`, and `Role Fit Tagging` Code nodes.
- Salary scoring assumes listed figures are USD/month; if a source lists an
  annual figure, that job will likely under-score — adjust the parsing in
  the `Scoring` node if you see this happen often.
- Job-board response shapes (RemoteOK/Remotive/Himalayas JSON, WWR RSS)
  change occasionally; if a `Normalize *` node starts returning 0 items,
  check that source's current response shape against the field names read
  in that node.
