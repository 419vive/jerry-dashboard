# Daily Taiwan Eligible Async Remote Job Hunter — Setup Guide

## What this does

Runs every day at 8:30 AM Taiwan time. Pulls jobs from 8 sources, filters hard for Taiwan eligibility, scores each job 0–100, emails you a digest, and drafts cover letters for jobs scoring ≥ 85.

---

## Import the workflow

1. Open n8n → **Workflows** → **Import from file**
2. Select `taiwan-job-hunter.json`
3. The workflow imports inactive. Do the setup steps below before activating.

---

## Required credentials (set these first)

### 1. Google Sheets OAuth2
- In n8n: **Credentials** → **New** → Google Sheets OAuth2
- Name it exactly: `Google Sheets account`
- Follow the Google Cloud OAuth2 setup flow

### 2. Gmail OAuth2
- In n8n: **Credentials** → **New** → Gmail OAuth2
- Name it exactly: `Gmail account`
- Same Google Cloud project works

### 3. Telegram (optional but recommended for 85+ alerts)
- Create a bot via [@BotFather](https://t.me/BotFather) → get bot token
- In n8n: **Credentials** → **New** → Telegram API
- Name it: `Telegram account`
- Get your chat ID: message your bot, then visit `https://api.telegram.org/bot<TOKEN>/getUpdates`

---

## Required variables (set in n8n Settings → Variables)

| Variable | Value |
|---|---|
| `ANTHROPIC_API_KEY` | Your key from console.anthropic.com |
| `TELEGRAM_CHAT_ID` | Your Telegram chat/user ID (numeric) |

---

## Google Sheets setup

1. Create a new Google Sheet called: **Taiwan Eligible Async Remote Jobs**
2. Copy the Sheet ID from the URL: `https://docs.google.com/spreadsheets/d/SHEET_ID_HERE/edit`
3. Create these 4 tabs (exact names):

**Tab 1: Raw Jobs**
```
date_found | source | job_title | company | job_url | location_text | salary | description | posted_date | tags | job_id
```

**Tab 2: Filtered Jobs**
```
date_found | score | job_title | company | job_url | salary | taiwan_eligible | taiwan_reason | location_risk | timezone_risk | contractor_possible | async_level | role_category | why_good_fit | decision | application_status | notes
```

**Tab 3: Applied**
```
date_applied | job_title | company | job_url | cover_letter_used | status | follow_up_date | notes
```

**Tab 4: Rejected**
```
date_rejected | job_title | company | job_url | reason
```

---

## Replace placeholders in the workflow

In the imported workflow, search for and replace these placeholder values:

| Placeholder | Replace with |
|---|---|
| `GOOGLE_SHEET_ID_PLACEHOLDER` | Your actual Sheet ID |
| `GOOGLE_SHEETS_CREDENTIAL_ID` | ID of your Google Sheets credential |
| `GMAIL_CREDENTIAL_ID` | ID of your Gmail credential |
| `TELEGRAM_CREDENTIAL_ID` | ID of your Telegram credential |

You can edit these directly in each node's settings after import.

---

## Scoring system

| Category | Points |
|---|---|
| Taiwan eligible (worldwide/contractor/APAC) | 20–40 |
| Async fit (high/medium/low) | 10–25 |
| Role fit (data/AI/marketing analytics) | 15–25 |
| Salary listed / ≥$3k / ≥$5k per month | 10–25 |
| **Max score** | **115** (capped at 100 in display) |

**Score thresholds:**
- ≥ 85 → Immediate apply: cover letter drafted + Telegram alert
- 70–84 → Saved for manual review
- < 70 → Auto-rejected

---

## Job sources

| Source | Method |
|---|---|
| RemoteOK | Official JSON API |
| Remotive | Official REST API |
| We Work Remotely | RSS feed |
| Himalayas | RSS feed |
| Dynamite Jobs | RSS feed |
| Wellfound | HTML scrape (may need updating) |
| YC Work at a Startup | HTML scrape (may need updating) |
| Otta | GraphQL API (may require auth) |

> Wellfound, YC, and Otta use scraping which can break if the site changes. The workflow uses `continueOnFail: true` on all fetch nodes so a broken scraper won't stop the rest.

---

## AI model used

- **Taiwan eligibility check:** `claude-haiku-4-5-20251001` (fast, cheap, ~$0.001/job)
- **Cover letter generation:** `claude-sonnet-4-6` (higher quality)

Estimated daily cost: < $0.10 for 50 jobs checked.

---

## Test before activating

1. Open the workflow
2. Click **Execute Workflow** manually
3. Check each node's output to verify data is flowing
4. Confirm the digest email arrives
5. If everything looks good → toggle **Active** to ON

---

## One-sentence rule (built in)

Every job in your Filtered Jobs sheet passed this test:

> *I can do this job while living in Taiwan, without moving, without foreign work authorization, and without destroying my sleep schedule.*

If a job doesn't pass this in both the hard filter AND the AI check, it's rejected automatically.
