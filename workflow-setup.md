# Daily Taiwan Eligible Async Remote Job Hunter — Setup Guide

## 1. Import the Workflow

1. Open n8n
2. Go to **Workflows** → **Import from File**
3. Upload `taiwan-job-hunter-workflow.json`

---

## 2. Create the Google Sheet

Create a new Google Sheet named **"Taiwan Eligible Async Remote Jobs"**

Add these four sheets (tabs):

### Sheet 1: Raw Jobs
| date_found | source | job_title | company | job_url | location_text | salary | description | posted_date | tags | job_id |

### Sheet 2: Filtered Jobs
| date_found | score | job_title | company | job_url | salary | taiwan_eligible | taiwan_reason | location_risk | timezone_risk | contractor_possible | async_level | role_category | why_good_fit | decision | application_status | notes |

### Sheet 3: Applied
| date_applied | job_title | company | job_url | cover_letter_used | status | follow_up_date | notes |

### Sheet 4: Rejected
| date_rejected | job_title | company | job_url | reason |

Copy the **Sheet ID** from the URL:
`https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID_HERE/edit`

---

## 3. Set Workflow Variables

In n8n, go to **Variables** and add:

| Variable | Value |
|---|---|
| `GOOGLE_SHEET_ID` | Your Google Sheet ID |
| `TELEGRAM_CHAT_ID` | Your Telegram chat ID (optional) |

---

## 4. Set Up Credentials

### Google Sheets
1. n8n → **Credentials** → **New** → **Google Sheets OAuth2**
2. Connect your Google account
3. Name it: `Google Sheets`

### OpenAI API
1. n8n → **Credentials** → **New** → **HTTP Header Auth**
2. Name: `OpenAI API`
3. Header Name: `Authorization`
4. Header Value: `Bearer sk-your-openai-key-here`

### Gmail
1. n8n → **Credentials** → **New** → **Gmail OAuth2**
2. Connect your Google account (419vive@gmail.com)
3. Name it: `Gmail`

### Telegram (optional — skip if not using)
1. Message @BotFather on Telegram → `/newbot`
2. Copy the bot token
3. n8n → **Credentials** → **New** → **Telegram API**
4. Name: `Telegram Bot`
5. Paste the token
6. Get your chat ID by messaging your bot and visiting:
   `https://api.telegram.org/botYOUR_TOKEN/getUpdates`

---

## 5. Map Credentials to Nodes

After importing, open each node and select the matching credential:
- All `Google Sheets` nodes → select `Google Sheets`
- `AI Taiwan Eligibility Check` → select `OpenAI API`
- `Generate Cover Letter` → select `OpenAI API`
- `Send Gmail Digest` → select `Gmail`
- `Send Telegram Alert` → select `Telegram Bot`

---

## 6. Schedule

The workflow runs daily at **8:30 AM Taiwan time (Asia/Taipei)**.

Already configured in the Schedule Trigger node.

---

## 7. One-Time Manual Test

Before the first scheduled run:
1. Click **Execute Workflow** to test manually
2. Check the **Raw Jobs** sheet gets populated
3. Check the **Filtered Jobs** sheet
4. Verify Gmail receives the digest

---

## Scoring Reference

| Factor | Points |
|---|---|
| Worldwide / work from anywhere / global | 40 |
| APAC friendly | 30 |
| Unclear, no country restriction | 20 |
| Country restricted | 0 |
| Clearly async | 25 |
| Flexible hours | 20 |
| Distributed team | 15 |
| Remote first | 10 |
| Primary role match (data, AI, BI, growth) | 25 |
| Adjacent role | 15 |
| Salary > $5K/month | 25 |
| Salary > $3K/month | 15 |
| Salary listed | 10 |

**Keep if score ≥ 70. Instant alert if score ≥ 85.**

---

## Job Sources

| Source | Method |
|---|---|
| Remote OK | JSON API |
| Remotive | REST API |
| We Work Remotely | RSS |
| Himalayas | RSS |
| Dynamite Jobs | RSS |
| YC Work at Startup | HTML parse |
