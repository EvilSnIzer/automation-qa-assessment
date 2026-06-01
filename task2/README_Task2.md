# Task 2 — n8n Workflow: Daily Tech Digest

**Author:** Manan Sharma  
**Workflow File:** `Task2_Workflow_MananSharma.json`

---

## APIs Used

| API | Endpoint | Why I Chose It |
|-----|----------|----------------|
| **GitHub REST API** (Primary) | `GET /search/repositories` | Free, no auth required (60 req/hr), returns rich structured data. Searches for trending JavaScript repos with >100 stars. |
| **GitHub REST API** (Enrichment) | `GET /repos/{owner}/{repo}/readme` | Second endpoint on same API. Fetches README metadata for popular repos so the team gets a direct link to documentation. |
| **Discord Webhook** (Output) | `POST` to webhook URL | Free, instant setup, no OAuth complexity. URL stored as credential — not hardcoded in the public repo. |

## What the Transformation Does

The **Code node "Transform - Top 5 Repos"** performs three operations:
1. **Filters out** forked repositories and repos without descriptions (noise reduction)
2. **Sorts** by `stargazers_count` descending (most popular first)
3. **Slices** to top 5 and **reshapes** each result to keep only: name, URL, description, stars, forks, language, and open issues

## How the Conditional Branch Works

The **IF node** checks: `stars > 1000`

- **True path (🔥 Popular):** Repos get enriched with a README API call, then formatted with a README link
- **False path (📦 Rising):** Repos skip enrichment and are labeled as up-and-coming

This gives the team an instant view of established vs. emerging projects.

## How the Error Path Behaves

Every HTTP Request node uses `onError: continueErrorOutput`:

| Failure Point | What Happens | Result |
|---------------|-------------|--------|
| GitHub API fails (rate limit, timeout) | Routes to `Handle API Error` → sends alert to Discord | Team is notified; workflow doesn't crash |
| README fetch fails (repo has no README) | Routes to `Format Popular Repos` anyway | Gracefully handles missing data with fallback |
| Discord webhook fails (invalid URL, network) | Routes to `Handle Discord Error` | Logs the failure; digest was generated even if delivery failed |

**No silent failures.** Every error path either notifies or logs.

## Screenshots

- **`screenshots/workflow_canvas.png`** — Full workflow canvas showing all 13 nodes
- **`screenshots/discord_execution_output.png`** — Discord message showing successful digest delivery

## How to Run

1. Import `Task2_Workflow_MananSharma.json` into n8n
2. Update the Discord webhook URL in the "Send to Discord" and "Send Error to Discord" nodes
3. Click **Execute Workflow** or wait for the hourly schedule trigger
