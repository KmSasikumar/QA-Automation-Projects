# Task 2 — n8n "Morning Brief" Workflow

**File:** `Task2_Workflow_SasiKumarYadav.json` (import into n8n: Workflows → Import from File)
**Bonus:** `Bonus_UptimeMonitor_SasiKumarYadav.json`

## What it does
Every hour (testing cadence — change the Schedule Trigger to a daily 7:00 AM cron for
production), the workflow pulls the current top HackerNews stories, enriches the top
candidates with per-story details, ranks them, and posts a top-5 digest to Discord.

## APIs used, and why
- **HackerNews Firebase API** (`/v0/topstories.json`, then `/v0/item/{id}.json`) — free,
  no API key, no rate-limit paperwork, and it naturally fits the two-call pattern the
  brief asks for: one call lists story IDs, a second call enriches each story with its
  title, score and URL. Secrets stay out of the workflow entirely; the only credential
  is the Discord webhook, read from the `DISCORD_WEBHOOK_URL` environment variable
  (n8n Credentials/env store), never hard-coded in the export.

## Flow
1. **Schedule Trigger** — every 1 hour (testing).
2. **HN Top Stories** — GET the top-story ID list (retry ×3, error output wired).
3. **Take First 10 IDs** (Code) — bounds the enrichment fan-out so one run can't fire
   hundreds of requests.
4. **HN Item Details** — GET each story (retry ×3, error output wired).
5. **Top 5 Digest** (Code) — drops deleted/dead/non-story items, sorts by score
   descending, keeps the top 5, formats one line per story.
6. **Hot Day?** (IF) — if the top story's score is **≥ 200**, the digest is prefixed
   with a "HOT DAY" header; otherwise a normal header is used.
7. **Post to Discord** — POSTs `{ "content": <digest> }` to the webhook from env.

## Error handling
Both HTTP fetch nodes use **Continue On Fail (error output)** with retries. Any fetch
failure routes to **Log Failure**, which logs a structured line and posts a "Morning
brief FAILED…" notice to the same Discord channel — the run never dies silently, and a
notification outage can't mask a fetch outage because the failure is also in the
execution log.

## Bonus — uptime monitor
Pings the app (`APP_URL` env, defaults to the RealWorld demo) every 5 minutes with
3 retries; records totals/last status in workflow static data; any non-200 (or failed
request) posts an alert to the same Discord channel; a second trigger posts a daily
9:00 AM summary with the OK/total ratio.
