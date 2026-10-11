# Oytra — Automation & QA Developer Take-Home Assessment

**Candidate:** Sasi Kumar Yadav · **Date:** 11 October 2026

## Contents

| Deliverable | File |
|---|---|
| Task 1 — QA & Debug Report (PDF) | `Task1_QA_Report_SasiKumarYadav.pdf` (attached on Internshala; identical text: `Task1_QA_Report.md`) |
| Task 2 — n8n Morning Brief workflow | `Task2_Workflow_SasiKumarYadav.json` |
| Task 2 — README (APIs, transform, error path) | `Task2_README.md` |
| Bonus — Uptime Monitor workflow | `Bonus_UptimeMonitor_SasiKumarYadav.json` |
| Task 2 — canvas + successful-execution screenshots | attached on Internshala (`Task2_Screenshot_Canvas.png`, `Task2_Screenshot_Execution.png`) |

## Task 1 summary
The hosted demo API (api.realworld.io) was down (HTTP 530) during testing, so the official RealWorld
backend was run locally and exercised over its REST API: sign-up, login, article CRUD, favorites, follows,
comments, plus edge cases. **8 issues found** — including invalid emails accepted at registration, no login
rate limiting, a pagination crash that leaks server internals, and stored script payloads — each with repro
steps, severity and suspected cause, plus a root-cause analysis of the pagination crash. Correct behaviours
(duplicate checks, cross-user authorization, idempotent favorites) were verified too.

## Task 2 summary
An n8n workflow that builds a **morning brief** from the HackerNews API (free, keyless): hourly schedule
trigger → top-story IDs → per-item enrichment calls → Code node filters/ranks to a top-5 digest → IF branch
on the top story's score (≥ 200 = "hot day" header) → digest posted to Discord via a webhook URL read from
the environment (no secrets in the export). Both fetch nodes retry and route failures to a logging +
failure-notice branch, so nothing fails silently. The bonus workflow pings the app every 5 minutes, alerts
on any non-200, and posts a daily uptime summary. Full details in `Task2_README.md`.
