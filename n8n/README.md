# Daily Taiwan Eligible Async Remote Job Hunter

An n8n workflow that finds async, global remote **AI Agent** jobs Jerry can do
while physically based in Taiwan — no relocation, no US/EU/UK/Canada/Australia
work authorization, no fixed midnight hours.

**Job field focus: AI Agent roles** — AI Agent Engineer/Developer, Agentic AI
Engineer, LLM agent building, multi-agent systems, AI automation/workflow
consulting, applied AI engineer, prompt/RAG engineer, chatbot developer. All
other requirements (Taiwan eligibility, async-first, scoring, sheets, alerts,
digest) are unchanged from the general version of this workflow.

Every job that survives the pipeline must pass this sentence:

> I can do this job while living in Taiwan, without moving, without foreign
> work authorization, and without destroying my sleep schedule.

## Import

1. In n8n: **Workflows → Import from File** → select `taiwan-async-remote-job-hunter.json`.
2. Open the **Config - Fill These In** node (right after the trigger) and set:
   - `googleSheetId` — the ID of your Google Sheet (from its URL)
   - `telegramChatId` — your Telegram chat ID (message `@userinfobot` to get it)
   - `gmailTo` — defaults to `419vive@gmail.com`
   - `anthropicModel` — defaults to `claude-sonnet-5`; change if you use a
     different Claude model
3. Add credentials (n8n → Credentials):
   - **Google Sheets** (OAuth2)
   - **Telegram API** (bot token from `@BotFather`)
   - **Gmail** (OAuth2)
   - **Anthropic API** — used by the two `HTTP Request` nodes that call
     `https://api.anthropic.com/v1/messages` (`AI - Taiwan Eligibility Check`
     and `AI - Generate Cover Letter`). If your n8n version doesn't expose a
     predefined "Anthropic" credential type for HTTP Request nodes, switch
     those two nodes' Authentication to **Header Auth** and set header
     `x-api-key: <your key>` instead.
4. Attach the credentials to every Google Sheets / Telegram / Gmail / HTTP
   Request node (n8n won't carry them over from a JSON import automatically).

## Google Sheet setup

Create a sheet named **Taiwan Eligible Async Remote Jobs** with 4 tabs and
these exact header rows (row 1):

**Raw Jobs**
`date_found | source | job_title | company | job_url | location_text | salary | description | posted_date | tags | job_id`

**Filtered Jobs**
`date_found | score | job_title | company | job_url | salary | taiwan_eligible | taiwan_reason | location_risk | timezone_risk | contractor_possible | async_level | role_category | why_good_fit | decision | application_status | notes | cover_letter_draft`

**Applied** (filled in manually by Jerry when he actually applies — the
workflow does not auto-apply)
`date_applied | job_title | company | job_url | cover_letter_used | status | follow_up_date | notes`

**Rejected**
`date_rejected | job_title | company | job_url | reason`

## Pipeline

1. **Schedule Trigger** — daily 8:30 AM `Asia/Taipei`.
2. **Source fetch** — RemoteOK and Remotive (public JSON APIs), We Work
   Remotely and Himalayas (RSS) are live. Wellfound, Dynamite Jobs, YC Work
   at a Startup and Otta have no public API/RSS and are shipped as
   **disabled stub nodes** — see "Scraper stubs" below.
3. **Normalize** — maps every source to a common schema (`job_title`,
   `company`, `job_url`, `source`, `location_text`, `salary`, `description`,
   `posted_date`, `tags`).
4. **Keyword match** — keeps only postings mentioning async/global/remote
   signals (worldwide remote, work from anywhere, ...) or AI Agent field
   keywords (ai agent engineer, agentic ai, llm agent, multi-agent,
   ai automation engineer, ai workflow consultant, prompt engineer, rag
   engineer, chatbot developer, langchain, crewai, autogen, ...).
5. **Dedupe** — checks `job_url` (primary key) and `company + job_title`
   (fallback) against the existing Raw Jobs sheet; new jobs get appended
   to Raw Jobs regardless of what happens next.
6. **Taiwan hard filter** — keyword rejects (US/Canada/UK/EU/Australia-only,
   hybrid/onsite, "must be based in", fixed US/EU hours) before spending an
   AI call.
7. **AI Taiwan eligibility check** — calls Claude with the exact eligibility
   prompt from the brief, per job, and parses the JSON verdict
   (`taiwan_eligible`, `location_risk`, `timezone_risk`, `decision`, ...).
8. **Async & timezone filter** — tags `async_level` and rejects strict-hours
   postings.
9. **AI Agent role fit + scoring** — 0–40 Taiwan-eligibility points, 0–25
   async-fit points, 0–25 role-fit points (core AI Agent titles score higher
   than adjacent automation/AI-adjacent roles), 0–25 salary points; computes
   the final `decision` (`apply_immediately` / `manual_check` /
   `keep_review` / `reject`) using the brief's step-14 rules.
10. **Filtered Jobs** — anything not rejected (score ≥ 70) is appended.
    Score ≥ 85 also triggers an instant Telegram alert and an AI-drafted
    cover letter (≤ 220 words) saved back into the row.
11. **Daily digest** — top jobs (score sorted), the manual-check list, and a
    rejection breakdown (US only / EU only / hybrid / timezone risk counts)
    sent via Telegram and Gmail.

Every reject branch writes to the **Rejected** sheet with a reason, so
nothing disappears silently.

## Scraper stubs (Wellfound, Dynamite Jobs, YC Work at a Startup, Otta)

These sites don't offer a public API or RSS feed and require either login
or JS-rendered scraping, which a plain `HTTP Request` node can't do
reliably. Each has a disabled `HTTP Request` node plus a disabled
`Normalize` code node already wired into the pipeline as a placeholder. To
enable one:

1. Point the fetch node at a scraping service (Apify actor, ScrapingBee,
   Browserless, etc.) instead of the raw URL, or replace it with that
   service's dedicated n8n node.
2. Add an **HTML Extract** node (or use the scraping service's structured
   JSON output) to pull out job title, company, URL, location, description.
3. Fill in the matching `Normalize - *` code node to map those fields into
   the common schema (see any of the working Normalize nodes for the
   pattern).
4. Re-enable both nodes.

## Notes / things to double-check after import

- n8n node parameter dropdowns (e.g. Google Sheets "operation") can shift
  slightly between n8n versions — if a node shows a configuration warning
  after import, just reselect the operation from the dropdown; the
  underlying field mappings are already correct.
- The Anthropic model ID in `Config - Fill These In` should match whatever
  model slug your Anthropic account actually has access to.
- `instantAlertThreshold` (85) and `minScoreToKeep` (70) are exposed in the
  Config node so you can tune them without touching code nodes.
