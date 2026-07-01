# Daily Taiwan Eligible Async Remote Job Hunter

An n8n workflow that finds async, global-remote jobs Jerry can do while living
in Taiwan — no relocation, no US/EU/UK/Canada/Australia work authorization,
no fixed midnight-Taiwan-time hours.

File: [`daily-taiwan-async-remote-job-hunter.json`](./daily-taiwan-async-remote-job-hunter.json)

## What it does

1. **Trigger** — runs daily at 8:30 AM `Asia/Taipei`.
2. **Sources** — pulls jobs from RemoteOK (JSON API) and Remotive (JSON API)
   directly, and We Work Remotely + Himalayas via RSS. Wellfound, Dynamite
   Jobs, YC Work at a Startup, and Otta have no reliable public API/RSS (see
   [Manual-only sources](#manual-only-sources) below) — they ship as
   **disabled, disconnected** placeholder nodes so you can wire up an
   authorized integration later, or just check them by hand.
3. **Keyword match** — keeps jobs whose title/description/tags match the
   requested async/remote/role keyword list.
4. **Dedupe** — checks `job_url` (and `company + job_title` as a fallback)
   against the `Raw Jobs` sheet before doing any further work.
5. **Hard location filter** — rejects "US only", "hybrid", "onsite", "must
   live in", etc. unless a worldwide/APAC/contractor-friendly phrase is also
   present.
6. **AI Taiwan-eligibility check** — calls Claude (Anthropic Messages API)
   with the exact eligibility prompt you specified, per job, and parses the
   `taiwan_eligible / reason / location_risk / timezone_risk /
   contractor_possible / decision` JSON it returns.
7. **Async & timezone filter** — flags fixed-hours/"must overlap full
   business day" postings as high timezone risk.
8. **Role fit** — tags data/BI/growth/AI-marketing/automation roles as core
   fit, adjacent roles as adjacent fit.
9. **Scoring** — 0–100 per the rubric below; jobs under 70 are dropped
   (and logged to `Rejected`); jobs ≥ 85 get an **instant Telegram alert**
   plus an AI-drafted cover letter.
10. **Google Sheets** — writes to `Raw Jobs`, `Filtered Jobs`, and
    `Rejected`. `Applied` is intentionally left for you to fill in by hand
    (see the sticky note in the workflow).
11. **Daily digest** — one Telegram message (Gmail node included but
    disabled as an alternative channel) with the top jobs, a manual-check
    list, and a rejection-reason summary.

## Before you import

### 1. Create the Google Sheet

Create a spreadsheet named **"Taiwan Eligible Async Remote Jobs"** with four
tabs and these exact header rows (the workflow auto-maps columns by name):

**Raw Jobs**
```
date_found	source	job_title	company	job_url	location_text	salary	description	posted_date	tags	job_id
```

**Filtered Jobs**
```
date_found	score	job_title	company	job_url	salary	taiwan_eligible	taiwan_reason	location_risk	timezone_risk	contractor_possible	async_level	role_category	why_good_fit	decision	application_status	notes
```

**Applied** *(you fill this in manually as you apply — the workflow never writes to it)*
```
date_applied	job_title	company	job_url	cover_letter_used	status	follow_up_date	notes
```

**Rejected**
```
date_rejected	job_title	company	job_url	reason
```

### 2. Credentials to set up in n8n

| Credential | Type | Used for |
|---|---|---|
| Google Sheets account | Google Sheets OAuth2 | Raw Jobs / Filtered Jobs / Rejected read+append |
| Anthropic API Key | Generic **HTTP Header Auth** — header name `x-api-key`, value = your Anthropic API key | Taiwan eligibility check + cover letter draft |
| Telegram account | Telegram API (bot token) | Instant alerts + daily digest |
| Gmail account *(optional)* | Gmail OAuth2 | Alternative to Telegram for the daily digest (disabled by default) |

### 3. After importing the JSON

- Open every **Google Sheets** node and pick your spreadsheet from the
  `documentId` field (it's currently a placeholder: `YOUR_GOOGLE_SHEET_ID`).
- Attach the credentials above to the Google Sheets, HTTP Request (Claude),
  Telegram, and (if used) Gmail nodes — n8n will flag them as missing on
  import.
- Set an n8n environment variable `TELEGRAM_CHAT_ID` (or hardcode it into
  the two Telegram nodes' `chatId` field).
- **Verify the RSS/API URLs still work.** Job boards change their endpoints
  without notice. In particular:
  - `https://remoteok.com/api` and `https://remotive.com/api/remote-jobs`
    are stable, long-standing public APIs.
  - The We Work Remotely category RSS feeds
    (`weworkremotely.com/categories/remote-*-jobs.rss`) are documented and
    stable.
  - The Himalayas RSS URL (`himalayas.app/jobs/rss`) is marked with a note
    in the workflow — confirm it resolves after import; if not, swap in
    their current jobs feed/API URL.
- Turn the workflow **Active** once everything above is verified end-to-end
  with a manual test run.

## Manual-only sources

Wellfound, Dynamite Jobs, YC's Work at a Startup, and Otta don't expose a
public API or RSS feed as of this writing — Wellfound and YC sit behind
login/Cloudflare bot protection, and none of them publish a stable feed.
Automating them reliably would mean scraping fragile HTML or storing
personal login credentials in n8n, which isn't something this workflow does
by default. Each has a **disabled, disconnected** `HTTP Request` placeholder
node (grouped under the sticky note "Manual/Scrape-Only Sources") so you can
wire up a compliant integration later if one becomes available — until then,
check those four boards by hand.

## Scoring rubric

| Category | Points |
|---|---|
| Taiwan eligibility | 40 worldwide/anywhere/contractor-friendly · 30 APAC · 20 unclear-no-restriction · 0 country-restricted |
| Async fit | 25 clearly async · 20 flexible hours · +15 distributed team · +10 remote-first |
| Role fit | 25 core role (data/BI/growth/AI-marketing/automation) · 15 adjacent |
| Salary fit | 10 listed · +15 above $3,000/mo · +25 above $5,000/mo |

Kept only if total ≥ 70 (capped at 100). Instant Telegram alert + AI cover
letter draft for anything ≥ 85.

## Final decision logic

- **apply_immediately** — `taiwan_eligible = yes`, score ≥ 85, async level
  high/medium, location risk low, timezone risk low/medium.
- **manual_check** — `taiwan_eligible = unclear`, score ≥ 85, location risk
  medium.
- **reject** — `taiwan_eligible = no`, or location risk high, or timezone
  risk high, or score < 70 (all handled by earlier filter stages, which also
  write the reason to the `Rejected` sheet).
- Everything else that clears the ≥70 bar lands in the daily digest as
  `review` for Jerry to look over manually.

Every surviving job is meant to pass the one-sentence rule: *"I can do this
job while living in Taiwan, without moving, without foreign work
authorization, and without destroying my sleep schedule."*
