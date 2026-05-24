# Taiwan Job Hunter — Setup Guide

## Step 1: Import the workflow

1. Open your n8n instance
2. Go to **Workflows → Import from file**
3. Upload `taiwan-job-hunter-workflow.json`

---

## Step 2: Create the Google Sheet

Create a new Google Sheet named **Taiwan Eligible Async Remote Jobs** with 4 tabs:

### Sheet 1: Raw Jobs
| date_found | source | job_title | company | job_url | location_text | salary | description | posted_date | tags | job_id |

### Sheet 2: Filtered Jobs
| date_found | score | job_title | company | job_url | salary | taiwan_eligible | taiwan_reason | location_risk | timezone_risk | contractor_possible | async_level | role_category | why_good_fit | decision | application_status | notes |

### Sheet 3: Applied
| date_applied | job_title | company | job_url | cover_letter_used | status | follow_up_date | notes |

### Sheet 4: Rejected
| date_rejected | job_title | company | job_url | reason |

After creating the sheet, copy the Sheet ID from the URL:
`https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID_HERE/edit`

---

## Step 3: Configure credentials in n8n

In n8n **Settings → Credentials**, add:

### Google Sheets OAuth2
- Name: `Google Sheets account`
- Authenticate with your Google account

### OpenAI API
- Name: `OpenAI account`
- Add your API key from platform.openai.com

### Gmail OAuth2
- Name: `Gmail account`
- Authenticate with your Google account (same or different from Sheets)

---

## Step 4: Update workflow placeholders

After importing, update these values in the workflow nodes:

| Node | Field | Replace with |
|------|-------|-------------|
| Read Existing Jobs | Document ID | Your Google Sheet ID |
| Write Raw Jobs | Document ID | Your Google Sheet ID |
| Write Filtered Jobs | Document ID | Your Google Sheet ID |
| Write Rejected Jobs | Document ID | Your Google Sheet ID |
| All credential fields | Credential ID | Your actual credential IDs |

---

## Step 5: Activate

1. Click **Activate** on the workflow
2. The workflow runs daily at **8:30 AM Taiwan time (Asia/Taipei)**
3. You'll receive a daily digest email at `419vive@gmail.com`
4. Jobs scoring 85+ trigger an instant alert email

---

## Score breakdown

| Category | Points |
|----------|--------|
| Taiwan-eligible (worldwide/global/contractor) | 40 |
| APAC remote | 30 |
| Unclear but no restriction | 20 |
| Async-first culture | 25 |
| Flexible hours / distributed | 15–20 |
| High role fit (data, BI, AI marketing) | 25 |
| Adjacent role | 15 |
| Salary ≥ $5k/mo | 25 |
| Salary ≥ $3k/mo | 15 |

**Min to keep: 70 | Instant alert + cover letter: 85**

---

## One-sentence test

> *I can do this job while living in Taiwan, without moving, without foreign work authorization, and without destroying my sleep schedule.*

Every job must pass this. If it doesn't, it's rejected.

---

## Job sources

| Source | Method |
|--------|--------|
| Remote OK | JSON API (`remoteok.com/api`) |
| Remotive | JSON API (`remotive.com/api/remote-jobs`) |
| We Work Remotely | RSS feed |
| Himalayas | JSON API |
| Dynamite Jobs | RSS feed |
| Wellfound | HTML scrape (JSON-LD) |
| YC Work at Startup | HTML scrape (JSON-LD) |
| Otta | JSON API / HTML scrape |

> Note: Wellfound and Otta may require login for full job data. The workflow extracts what's publicly available. Consider adding your session cookies via HTTP Request node headers if you have accounts.
