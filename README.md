# Automation & QA Developer — Take-Home Assessment

**Candidate:** Manan Sharma  
**Date:** June 2, 2026  

---

## Repository Structure

```
├── README.md                                          ← This file
│
├── task1/
│   ├── Task1_QA_Report_MananSharma.pdf                ← Bug report (PDF)
│   └── Task1_QA_Report_MananSharma.md                 ← Bug report (Markdown source)
│
├── task2/
│   ├── Task2_Workflow_MananSharma.json                ← n8n workflow export
│   ├── README_Task2.md                                ← Workflow documentation (1 page)
│   └── screenshots/
│       ├── workflow_canvas.png                        ← (a) Workflow canvas
│       └── discord_execution_output.png               ← (b) Successful execution + Discord message
│
└── bonus/
    ├── Bonus_UptimeMonitor_MananSharma.json           ← n8n uptime monitor export
    └── screenshots/
        ├── uptime_monitor_canvas.png                  ← Workflow canvas
        └── uptime_daily_execution.png                 ← Daily summary execution
```

---

## Task 1 — Web App QA & Debug Report

**App tested:** [RealWorld "Conduit"](https://demo.realworld.show/) — Angular frontend + demo backend

**Testing approach:**
- Manual testing of all main user flows (home feed, articles, sign-up, login, profiles, navigation)
- HTTP header inspection via `curl -sI`
- Source code review on [GitHub](https://github.com/realworld-apps/angular-realworld-example-app)
- Edge case testing (empty inputs, invalid emails, non-existent users)

**7 issues found:**

| # | Issue | Severity |
|---|-------|----------|
| 1 | Missing security headers (no CSP, X-Frame-Options, HSTS) + server version leak | Critical |
| 2 | Page `<title>` never updates across routes | Medium |
| 3 | No Open Graph / meta tags — broken social sharing | Medium |
| 4 | Tags not editable when editing articles (confirmed [bug #343](https://github.com/realworld-apps/angular-realworld-example-app/issues/343)) | High |
| 5 | Outdated Ionicons v2.0.1 from external CDN without SRI | Medium |
| 6 | No client-side validation on registration form | Medium |
| 7 | Non-existent user profile shows blank page, no error | Low |

**Root-cause analysis** written for Issue #1 — includes a concrete Nginx config fix.

---

## Task 2 — n8n API Integration Workflow

**Workflow:** "Daily Tech Digest - GitHub & README Enrichment"

| Requirement | Implementation |
|-------------|----------------|
| **Trigger** | Schedule Trigger — every 1 hour |
| **First API** | GitHub Search API (`/search/repositories`) — trending JS repos |
| **Transformation** | Code node: filters forks, sorts by stars, keeps top 5, reshapes fields |
| **Second API** | GitHub README API (`/repos/{owner}/{repo}/readme`) — enriches popular repos |
| **Conditional** | IF node: `stars > 1000` → 🔥 Popular (with README enrichment) vs 📦 Rising |
| **Output** | Discord webhook — sends formatted daily digest |
| **Error handling** | `onError: continueErrorOutput` on all HTTP nodes; errors alert Discord |
| **Credentials** | Webhook URL as credential — not hardcoded in repo (placeholder used) |

**See:** [`task2/screenshots/discord_execution_output.png`](task2/screenshots/discord_execution_output.png) for the live Discord message

---

## Bonus — Uptime Monitor

**Workflow:** "Uptime Monitor - Conduit RealWorld App"

| Feature | Details |
|---------|---------|
| **Ping frequency** | Every 5 minutes |
| **Retry logic** | 3 retries, 5-second wait between attempts |
| **Response time tracking** | Tracked in milliseconds per check |
| **Alert** | Discord notification if status ≠ 200 (after retries exhausted) |
| **Daily summary** | Separate 9 AM cron trigger sends uptime report to Discord |
| **Error handling** | Connection failures, non-200 responses, Discord delivery failures — all handled |

---

## How to Run

1. Install n8n (`npx n8n` or Docker or [n8n Cloud](https://app.n8n.cloud))
2. Import JSON files via Workflows → Import from File
3. Update the Discord webhook URL in the HTTP Request nodes (replace `YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN`)
4. Execute workflows
