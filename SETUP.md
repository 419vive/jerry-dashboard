# Daily Taiwan Eligible Async Remote Job Hunter — Setup Guide

## 1. Import the workflow

1. Open n8n
2. Click **+ New Workflow** → **Import from file**
3. Select `taiwan_job_hunter.json`

---

## 2. Create credentials (do this first)

### Google Sheets OAuth2
1. In n8n → **Credentials** → **New** → search "Google Sheets OAuth2 API"
2. Follow the OAuth flow with your Google account
3. Note the credential ID — replace `GOOGLE_SHEETS_CREDENTIAL_ID` in the workflow

### Gmail OAuth2
1. In n8n → **Credentials** → **New** → search "Gmail OAuth2"
2. Follow the OAuth flow (same or different Google account)
3. Note the credential ID — replace `GMAIL_CREDENTIAL_ID` in the workflow

### Anthropic API (HTTP Header Auth)
1. In n8n → **Credentials** → **New** → search "Header Auth"
2. Name: `Anthropic API Key`
3. Name field: `x-api-key`
4. Value field: your Anthropic API key (`sk-ant-...`)
5. Note the credential ID — replace `ANTHROPIC_CREDENTIAL_ID` in the workflow

---

## 3. Create the Google Sheet

1. Create a new Google Sheet named: **Taiwan Eligible Async Remote Jobs**
2. Create 4 sheets (tabs) with these exact names:
   - `Raw Jobs`
   - `Filtered Jobs`
   - `Applied`
   - `Rejected`

### Raw Jobs — Column headers (Row 1)
`date_found | source | job_title | company | job_url | location_text | salary | description | posted_date | tags | job_id`

### Filtered Jobs — Column headers (Row 1)
`date_found | score | job_title | company | job_url | salary | taiwan_eligible | taiwan_reason | location_risk | timezone_risk | contractor_possible | async_level | role_category | why_good_fit | decision | application_status | notes`

### Applied — Column headers (Row 1)
`date_applied | job_title | company | job_url | cover_letter_used | status | follow_up_date | notes`

### Rejected — Column headers (Row 1)
`date_rejected | job_title | company | job_url | reason`

3. Copy the Sheet ID from the URL:
   `https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID_HERE/edit`
4. Replace every `YOUR_GOOGLE_SHEET_ID_HERE` in the workflow JSON with your actual Sheet ID

---

## 4. Update your email address

In the **Send Gmail Digest** node, replace `jerry@example.com` with your actual email.

---

## 5. Verify the schedule

The trigger runs at `30 8 * * *` in the `Asia/Taipei` timezone = 8:30 AM Taiwan time every day.

The workflow-level timezone is also set to `Asia/Taipei` in settings.

---

## 6. What each node does

| Node | Purpose |
|------|---------|
| Daily 8:30 AM Taiwan | Cron trigger, fires daily at 8:30 AM |
| Fetch * RSS / API | Pull jobs from each source |
| Parse * XML | Convert RSS XML → JSON |
| Normalize * | Extract: title, company, URL, salary, description, tags |
| Merge All Job Sources | Combine all 8 sources into one stream |
| Get Existing Raw Jobs | Read sheet to get already-seen URLs |
| Deduplicate Jobs | Skip jobs already in the sheet; keyword pre-filter |
| Hard Taiwan Filter | Regex reject: US only, hybrid, fixed hours, etc. |
| AI Taiwan Eligibility Check | Claude API — per-job eligibility judgment |
| Parse AI + Score + Filter | Parse Claude response, apply async/role/score logic |
| Score 70 Or Above | Branch: 70+ continues, below 70 → Rejected sheet |
| Append to Raw / Filtered Sheet | Write passing jobs to Google Sheets |
| Score 85 Or Above | Branch: 85+ gets a cover letter generated |
| AI Cover Letter | Claude drafts a 220-word cover letter for top jobs |
| Merge After Cover Letter | Reunite 85+ and sub-85 streams |
| Aggregate All Filtered | Collect all passing jobs into one list |
| Build Digest Email | Format the daily summary email |
| Send Gmail Digest | Email the digest to your address |
| Append to Rejected Sheet | Log sub-70 jobs to Rejected tab |

---

## 7. Scoring breakdown

| Category | Points |
|----------|--------|
| Taiwan eligible (worldwide / contractor) | 40 |
| Taiwan eligible (APAC or keep decision) | 30 |
| Taiwan unclear, no hard restriction | 20 |
| Async confirmed (async/asynchronous keyword) | 25 |
| Async likely (flexible hours) | 20 |
| Distributed team | 15 |
| Role: data analyst / scientist / BI / AI | 25 |
| Role: adjacent (analyst, marketing, etc.) | 15 |
| Salary listed above $5,000/mo | 25 |
| Salary listed above $3,000/mo | 15 |
| Salary listed (any amount) | 10 |

**Threshold:** 70+ → saved to Filtered Jobs sheet  
**Alert threshold:** 85+ → instant flag in digest + cover letter drafted

---

## 8. One-sentence rule (enforced in AI check)

> I can do this job while living in Taiwan, without moving, without foreign work authorization, and without destroying my sleep schedule.

If the job cannot pass this sentence, Claude rejects it.

---

## 9. Sources covered

| Source | Method |
|--------|--------|
| Remote OK | RSS feed |
| We Work Remotely | RSS feed |
| Himalayas | RSS feed |
| Remotive | JSON API |
| Dynamite Jobs | RSS feed |
| Wellfound | HTML scrape (JSON-LD) |
| Y Combinator Work at a Startup | HTML scrape (JSON-LD + Next.js data) |
| Otta | HTML scrape (JSON-LD) |

**Note:** Wellfound, YC, and Otta use JavaScript-heavy frontends. If scraping returns empty results, consider adding a Browserless/Puppeteer node or a third-party scraping API (ScraperAPI, Apify) in front of those fetch nodes.

---

## 10. Cost estimate (Anthropic API)

- AI eligibility check: ~300 tokens input + 100 tokens output per job
- Cover letter: ~800 tokens input + 300 tokens output per job
- Typical day: 20–80 jobs reach AI check → ~$0.05–$0.25/day with claude-sonnet-4-6
