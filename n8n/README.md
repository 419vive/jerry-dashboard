# Daily Taiwan Eligible Async Remote Job Hunter (n8n workflow)

Import `taiwan-async-remote-job-hunter.json` into n8n (Workflows → Import from File).
It runs every day at **8:30 AM Asia/Taipei** and hunts for async, worldwide-remote jobs
Jerry can legally and practically do while living in Taiwan.

## Pipeline

1. **Schedule Trigger** — daily at 8:30 AM (workflow timezone is set to `Asia/Taipei`).
2. **Fetch sources** — RemoteOK and Remotive via their public JSON APIs, We Work Remotely via
   RSS, Himalayas via its jobs API. Wellfound, Dynamite Jobs, YC Work at a Startup, and Otta
   are wired up as **disabled placeholders** (see "Sources that need extra work" below).
3. **Normalize** — every source is mapped to the same schema: `job_title`, `company`,
   `job_url`, `source`, `location_text`, `salary`, `description`, `posted_date`, `tags`.
4. **Keyword match** — keeps postings that mention async/global/worldwide-remote language or
   one of the target roles (data analyst, data scientist, BI analyst, marketing/growth
   analyst, AI automation/marketing/workflow specialist, etc).
5. **Dedupe** — against `job_url` (primary) and `company + job_title` (backup) already present
   in the **Raw Jobs** sheet.
6. **Taiwan hard filter** — rejects "US/UK/EU/Canada/Australia only", "must be based in/live
   in/authorized to work in", "hybrid", "onsite", "local work permit required", etc., unless the
   post also contains a keep-signal ("worldwide remote", "APAC remote", "hires via Deel", "EOR
   supported", "contractor role", ...). Rejects go straight to the **Rejected** sheet.
7. **AI Taiwan eligibility check** — an OpenAI call using the exact eligibility prompt from the
   spec, returning `taiwan_eligible`, `reason`, `location_risk`, `timezone_risk`,
   `contractor_possible`, `decision` as JSON.
8. **Async fit scoring**, **Role fit scoring**, **Total score** (0–100) — Taiwan eligibility
   (0–40) + async fit (0–25) + role fit (0–25) + salary fit (0–25), capped at 100.
9. Jobs scoring **below 70** are appended to **Rejected** with the reason. Jobs **≥ 70** get a
   final apply/manual_check/reject decision and are upserted into **Filtered Jobs**
   (matched/deduped on `job_url`).
10. Jobs scoring **≥ 85** trigger an instant Telegram (and optional Gmail) alert and an
    AI-drafted cover letter, which gets appended to that row's `notes` column.
11. After every job in the run has been processed, a **daily digest** (top jobs, manual-check
    list, rejected-reason counts) is sent over Telegram (and optionally Gmail).

The workflow never submits an application on its own — `application_status = ready_to_apply`
just flags that Jerry should apply today. Move the row into the **Applied** tab by hand once he
does.

## One-time setup

### 1. Google Sheet

Create a spreadsheet called **"Taiwan Eligible Async Remote Jobs"** with 4 tabs and these exact
header rows (the workflow auto-maps JSON fields to headers by name, so the headers must match):

**Raw Jobs**
`date_found, source, job_title, company, job_url, location_text, salary, description, posted_date, tags, job_id`

**Filtered Jobs**
`date_found, score, job_title, company, job_url, salary, taiwan_eligible, taiwan_reason, location_risk, timezone_risk, contractor_possible, async_level, role_category, why_good_fit, decision, application_status, notes`

**Applied**
`date_applied, job_title, company, job_url, cover_letter_used, status, follow_up_date, notes`

**Rejected**
`date_rejected, job_title, company, job_url, reason`

### 2. Credentials

Attach these credentials to the matching nodes after import (all Google Sheets, OpenAI, and
Telegram/Gmail nodes ship without credentials attached):

- **Google Sheets** — every `googleSheets` node. Also replace the placeholder
  `PUT_YOUR_GOOGLE_SHEET_ID_HERE` in each node's Document field with your real spreadsheet ID.
- **OpenAI** — "AI Taiwan Eligibility Check" and "AI Cover Letter Draft" nodes. Swap the
  `gpt-4o-mini` model id for whatever you have access to.
- **Telegram** — "Telegram Instant Alert" and "Telegram Daily Digest". Replace
  `PUT_YOUR_TELEGRAM_CHAT_ID_HERE` with your chat ID.
- **Gmail** (optional) — "Gmail Instant Alert (optional)" and "Gmail Daily Digest (optional)"
  are disabled by default since Telegram is the default channel. Enable whichever nodes you
  actually want to use.

### 3. Sources that need extra work

Wellfound, Dynamite Jobs, YC's Work at a Startup, and Otta don't expose a public API or RSS
feed (Work at a Startup and Otta also require a logged-in session to see full listings), so a
plain `HTTP Request` node can't reliably pull their listings. Those four source branches ship
**disabled** in the workflow. To include them, point the corresponding "Scrape ... (Manual
Setup)" node at a scraping service (e.g. ScraperAPI, Bright Data) or an authenticated
browser-automation tool, then adjust the matching `Normalize` node's field mapping to fit
whatever shape that scraper returns.

## Notes on node versions

Node parameter shapes (especially for the OpenAI, Telegram, and Gmail nodes) can shift slightly
between n8n releases. On import, n8n will flag any node whose version needs updating and offer
to migrate parameters automatically — re-check those three node types first if anything looks
off after import.
