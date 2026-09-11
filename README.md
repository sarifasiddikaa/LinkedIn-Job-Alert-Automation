# LinkedIn Job Alert Automation

An n8n workflow that automatically scrapes fresh LinkedIn job postings, saves new
listings to Google Sheets, and generates an AI-written hiring-trend summary —
triggered on demand via webhook or automatically every day on a schedule.

Built for an n8n/AI-automation coursework assignment (LinkedIn Job Alert
Automation with Apify, HTTP Request nodes, and Google Sheets), extended with a
few production-style touches: input validation, bounded polling with a real
failure path, deduplication against existing sheet rows, and Slack notifications.

## What it does

1. Accepts a job title, location, and result limit (via webhook, or a saved
   default for the daily scheduled run).
2. Starts an Apify actor run to scrape matching LinkedIn job postings.
3. Polls the run status until it finishes (bounded — won't loop forever).
4. Fetches the scraped dataset once the run succeeds.
5. Filters out jobs already saved from a previous run.
6. Appends only the new postings to a "Jobs" tab in Google Sheets.
7. Sends the batch to an AI model, which writes a short summary of the most
   common hiring companies, locations, and any hiring trends.
8. Logs that summary to a "Summaries" tab.
9. Posts a Slack notification with the results (or an alert if the run failed).
10. Returns a JSON response to the webhook caller.

## Workflow structure

```
Webhook ─┐
         ├─→ Validate ─→ Start Run ─→ Init Poll State ─→ Wait ─→ Check Status
Schedule ┘                                                          │
Trigger →                                                    Code in JavaScript
   │                                                                 │
Set Default                                                   Check Succeeded
Search                                                          ├── True ──→ Get Dataset ─→ Transform
                                                                 │                              │
                                                                 False                    Get Existing
                                                                   │                             │
                                                            Check Still Running            Filter New
                                                                 ├── True → Wait (loop)          │
                                                                 └── False ↓               Append Sheet
                                                                     Format Error                │
                                                                       │                     Ai Summary
                                                              ┌────────┴───────┐                 │
                                                        Respond Error   Notify Failure     Format Summary
                                                                                                  │
                                                                                            Log Summary
                                                                                          ┌───────┴───────┐
                                                                                   Respond Success   Notify New Jobs
```

## Setup

### 1. Import the workflow
Open n8n → **Workflows → Import from File** → select the exported `.json`.

### 2. Apify (job scraping)
1. Get an API token from **console.apify.com → Settings → Integrations**.
2. Note: Apify requires a **verified payment method on file** to run
   pay-per-event actors, even when usage is covered by free monthly credit.
   Add a card at `console.apify.com/billing` if you hit a "Payment required" error.
3. In the **Start Run**, **Check Status**, and **Get Dataset** nodes, set the
   `token` query parameter to your Apify token.
4. This build uses the `curious_coder~linkedin-jobs-scraper` actor, which
   expects `keywords` (array) and `location` in its input body — adjust if you
   swap actors, since input field names vary between actors.

### 3. Google Sheets
1. Create a sheet with two tabs:
   - `Jobs`: `jobTitle | companyName | location | postedTime | jobUrl | companyUrl`
   - `Summaries`: `timestamp | jobTitle | location | jobsFound | summary`
2. Connect a Google Sheets OAuth2 credential in n8n and select it in the
   **Get Existing**, **Append Sheet**, and **Log Summary** nodes.
3. Set each node's Spreadsheet ID to your sheet, and Range to `Jobs!A:F` or
   `Summaries!A:E` as appropriate.

### 4. AI summary (Google Gemini)
1. Get a free API key from **aistudio.google.com → Get API key**.
2. In the **Ai Summary** node, set the URL to:
   ```
   https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-lite:generateContent?key=YOUR_KEY
   ```
   (check aistudio.google.com for the current recommended model name if this
   one has since been retired).

### 5. Slack notifications
1. Create a Slack app at **api.slack.com/apps → Create New App → Blank app**.
2. Under **OAuth & Permissions**, add the `chat:write` bot scope, then
   **Install to Workspace** and copy the **Bot User OAuth Token** (`xoxb-...`).
3. Invite the bot to your target channel: `/invite @YourBotName`.
4. In n8n, add a Slack credential with that token, and set the channel ID in
   both the **Notify New Jobs** and **Notify Failure** nodes.

### 6. Daily schedule
Edit the **Set Default Search** node with the `jobTitle`/`location`/`limit`
you want searched automatically each day, and adjust the trigger hour in the
**Schedule Trigger** node if needed.

## Triggering it

Activate the workflow (toggle top-right in the n8n editor), then:

```bash
curl -X POST https://<your-n8n-domain>/webhook/linkedin-job-alert \
  -H "Content-Type: application/json" \
  -d '{"jobTitle": "Python Developer", "location": "Bangladesh", "limit": 20}'
```

Sample success response:
```json
{
  "status": "success",
  "jobsFound": 100,
  "jobsAdded": 100,
  "summary": "The job postings are overwhelmingly dominated by The Home Depot..."
}
```

## Design notes / what's beyond the base requirement

- **Input validation** — empty job titles are rejected, `limit` is capped at 100.
- **Bounded polling** — the Wait → Check Status loop stops after a max attempt
  count instead of running forever on a stuck Apify run.
- **Three-way status routing** — succeeded / still running / failed are handled
  as distinct paths (via two IF nodes) rather than a single if/else.
- **Deduplication** — re-running the same search won't create duplicate rows,
  since existing `jobUrl` values are read before appending.
- **Graceful failure path** — a failed or timed-out run returns a structured
  error response and a Slack alert, instead of the webhook hanging.
- **Separate summary log** — the AI summary is appended to its own sheet tab
  with a timestamp, building a running trend history rather than a one-off text blob.

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| Webhook returns 404 "not registered" | Workflow isn't Active — toggle it on. |
| Apify: "Payment required" | Add a verified card at Apify billing (won't be charged while within free credit for small test runs). |
| Ai Summary: 404 "resource not found" | The Gemini model name was retired — check aistudio.google.com for the current one. |
| Slack: `not_in_channel` | Invite the bot to the channel first. |
| `Format Summary`: "Paired item data unavailable" | Use `.first().json` instead of `.item.json` when referencing nodes like `Get Existing` that don't return a 1:1 item mapping. |

