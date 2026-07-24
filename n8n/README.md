# Daily Taiwan Eligible Async Remote Job Hunter

An n8n workflow that finds async, global-remote jobs Jerry can do while living
in Taiwan. Taiwan eligibility is the hard gate — a job only survives the
pipeline if it can be done from Taiwan with no relocation, no US/EU/UK/CA/AU
work authorization, and no fixed midnight hours.

Import `taiwan-remote-job-hunter.json` into n8n (Workflows → Import from File).

## What it does

1. **Trigger** — runs daily at 08:30 `Asia/Taipei`.
2. **Fetch sources** — RemoteOK, Remotive (public JSON APIs) and We Work
   Remotely (RSS) are wired for real. Himalayas is best-effort JSON. Wellfound,
   Dynamite Jobs, Work at a Startup, and Otta have no stable public API — their
   branches are stubbed to fail soft (return zero jobs) instead of breaking the
   run. See "Known limitations" below before relying on them.
3. **Keyword match** — keeps jobs whose title/description/tags mention any of
   the async/global/role keywords from the spec.
4. **Dedupe** — reads `job_url` (and `company + job_title` as a backup key)
   from the *Raw Jobs* sheet and drops anything already logged.
5. **Log to Raw Jobs** — every new, not-yet-seen job is appended here
   regardless of what happens next, so nothing is silently lost.
6. **Taiwan hard filter** — keyword reject/keep list from the spec. A
   keep-phrase match (e.g. "worldwide remote") always overrides a reject-phrase
   match (e.g. a stray mention of "office").
7. **AI Taiwan eligibility check** — calls Claude (Anthropic Messages API)
   with the exact eligibility prompt from the spec and parses its JSON verdict.
8. **Async filter** — keeps async/flexible-hours language, rejects
   fixed-hours/live-call language.
9. **Role fit** — tags the job's role category (data analyst, growth analyst,
   AI marketing specialist, etc.).
10. **Scoring** — 0–100 across Taiwan eligibility (40), async fit (25), role
    fit (25), salary (25 max, cumulative categories capped). Anything under 70
    is rejected with a reason; everything else continues.
11. **Google Sheets** — writes to *Raw Jobs*, *Filtered Jobs*, and *Rejected*
    (see sheet layout below). *Applied* is intentionally left for Jerry / a
    follow-up workflow to update by hand — nothing here auto-marks a job
    "applied."
12. **Daily digest** — reads back today's *Filtered Jobs* and *Rejected* rows,
    builds the exact message format from the spec (top jobs, manual-check
    list, rejection counts by category), and sends it via Telegram and Gmail.
13. **Cover letter + instant alert** — for score ≥ 85, drafts a sub-220-word
    cover letter with Claude and sends an instant Telegram + Gmail alert.
14. **Final decision** — `apply_immediately` / `manual_check` per the spec's
    rules; anything that cleared every gate but didn't hit the 85+ auto-apply
    bar defaults to `manual_check` rather than being dropped.

## Before you run it

### 1. Create the Google Sheet
Create a sheet named **Taiwan Eligible Async Remote Jobs** with 4 tabs and
these exact header rows (order matters — the workflow writes by column name):

**Raw Jobs**
`date_found, source, job_title, company, job_url, location_text, salary, description, posted_date, tags, job_id`

**Filtered Jobs**
`date_found, score, job_title, company, job_url, salary, taiwan_eligible, taiwan_reason, location_risk, timezone_risk, contractor_possible, async_level, role_category, why_good_fit, decision, application_status, notes`

**Applied**
`date_applied, job_title, company, job_url, cover_letter_used, status, follow_up_date, notes`

**Rejected**
`date_rejected, job_title, company, job_url, reason`

Copy the sheet's ID from its URL and paste it over every
`REPLACE_WITH_YOUR_GOOGLE_SHEET_ID` placeholder — easiest done via n8n's
resource picker on each Google Sheets node (Get Existing Raw Job URLs, Append
Raw Jobs, Append Filtered Jobs, Append Rejected Jobs, Read Today's Filtered
Jobs, Read Today's Rejected Jobs).

### 2. Set credentials in n8n
- **Google Sheets** — OAuth2 credential, attach to all 6 Google Sheets nodes.
- **Anthropic** — the workflow calls `https://api.anthropic.com/v1/messages`
  directly via HTTP Request nodes (no special node needed). Set an
  `ANTHROPIC_API_KEY` environment variable on your n8n instance (used by the
  `AI Taiwan Eligibility Check` and `AI Cover Letter Draft` nodes), or replace
  the header expression with a Header Auth credential if you prefer.
- **Telegram** — create a bot via @BotFather, add the Telegram credential in
  n8n, and replace `REPLACE_WITH_YOUR_TELEGRAM_CHAT_ID` on both Telegram nodes.
- **Gmail** — OAuth2 credential, and replace `REPLACE_WITH_YOUR_EMAIL` on both
  Gmail nodes.
- You only need Telegram *or* Gmail — delete whichever you don't want, or
  leave both active.

### 3. Sanity-check the model id
The AI nodes call `claude-sonnet-5`. Swap it for whatever Claude model id you
want to run against if that changes.

## Known limitations

- **Wellfound, Dynamite Jobs, Work at a Startup, Otta** have no documented
  public API today, and Wellfound in particular disallows scraping in its
  Terms of Service. Their `Normalize *` nodes currently return `[]` on
  purpose — they don't break the run, but they also don't contribute jobs.
  If you want real coverage from these, that means either an authenticated
  partner feed, a manual weekly check, or a scraper you've verified is
  compliant with each site's ToS — swap the corresponding `Fetch *` /
  `Normalize *` node pair once you have one.
- **Himalayas'** JSON shape is inferred, not confirmed against current docs —
  verify the first live run's output against the actual response shape and
  adjust `Normalize Himalayas` if fields don't line up.
- **Salary parsing** in the scoring step is a best-effort regex over whatever
  free text each source provides; it won't catch every format (annual vs.
  monthly, non-USD currencies, etc.).
- **The "Applied" sheet** is not written to by this workflow — mark
  applications there yourself, or extend the workflow with a step that moves
  `apply_immediately` rows there automatically once you're comfortable with
  it acting unattended.
