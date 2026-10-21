# QA & Automation — RealWorld QA Report + n8n Workflows

**Author:** Sasi Kumar Yadav · **Date:** 11 October 2026

A personal project in two parts: a hands-on QA & debug audit of the RealWorld (Conduit)
reference application, and a pair of n8n automation workflows (a HackerNews morning
brief and an uptime monitor).

## Contents

| Part | File |
|---|---|
| Part 1 — QA & Debug Report | `QA_Report_RealWorld.md` |
| Part 2 — n8n Morning Brief workflow | `MorningBrief_Workflow.json` |
| Part 2 — README (APIs, transform, error path) | `MorningBrief_README.md` |
| Extra — Uptime Monitor workflow | `UptimeMonitor_Workflow.json` |

## Part 1 summary — QA & debug report
The hosted demo API (api.realworld.io) was down (HTTP 530) during testing, so the official RealWorld
backend was run locally and exercised over its REST API: sign-up, login, article CRUD, favorites, follows,
comments, plus edge cases. **8 issues found** — including invalid emails accepted at registration, no login
rate limiting, a pagination crash that leaks server internals, and stored script payloads — each with repro
steps, severity and suspected cause, plus a root-cause analysis of the pagination crash. Correct behaviours
(duplicate checks, cross-user authorization, idempotent favorites) were verified too.

## Part 2 summary — n8n workflows
An n8n workflow that builds a **morning brief** from the HackerNews API (free, keyless): hourly schedule
trigger → top-story IDs → per-item enrichment calls → Code node filters/ranks to a top-5 digest → IF branch
on the top story's score (≥ 200 = "hot day" header) → digest posted to Discord via a webhook URL read from
the environment (no secrets in the export). Both fetch nodes retry and route failures to a logging +
failure-notice branch, so nothing fails silently. The companion uptime-monitor workflow pings the app every
5 minutes, alerts on any non-200, and posts a daily uptime summary. Full details in `MorningBrief_README.md`.
