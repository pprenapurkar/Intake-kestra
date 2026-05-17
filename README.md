# Intake 🎯
### AI-powered job search assistant, orchestrated entirely by Kestra.

Built for the [WeMakeDevs Kestra Orchestration Challenge](https://wemakedevs.org/orchestration) · [Kestra Fundamentals Certification](https://academy.kestra.io/kestra-fundamentals)

---

## The Problem

As an international grad student juggling coursework, projects, and a job search simultaneously, the logistics were overwhelming. Missing new postings. Forgetting to follow up. Losing track of application status. The frustration was real — and the project idea came directly from it.

Intake automates the entire operational layer of a job search. Three Kestra flows run on a schedule. Everything lands on Telegram. Nothing to babysit.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        INTAKE                               │
│                                                             │
│   Flow 1          Flow 2              Flow 3                │
│   8am daily       6pm daily           Sunday 9am            │
│                                                             │
│   Adzuna API  →   App Log        →   Metrics                │
│   Deduplicate     Detect overdue     Claude analysis        │
│   Claude score    Claude draft       Human review (Pause)   │
│   Telegram        Telegram           Telegram               │
└─────────────────────────────────────────────────────────────┘
```

---

## Flow 1 — Morning Job Digest (8am daily)

Fetches fresh job listings, scores each one against your profile using Claude, removes anything already seen, and sends the top matches to Telegram every morning.

```
[Schedule: 8am]
       │
       ▼
[http.Request]
Fetch jobs from Adzuna API
(software engineer + backend engineer, Chicago)
       │
       ▼
[python.Script]
Deduplicate via Kestra KV store
Remove job IDs already seen in previous runs
       │
       ▼
[python.Script]
Score each job with Claude
  → match score (1-10)
  → reason it fits your profile
  → missing skills from JD
  → tailored one-line pitch
       │
       ▼
[python.Script]
Filter: keep only score >= 7
Sort by score descending, take top 5
       │
       ▼
[python.Script]
Send digest to Telegram
       │
       ▼
[kv.Set]
Write all seen job IDs back to KV store
Prevents duplicates in future runs (capped at 500 IDs)
       │
       ▼
[log.Log]
Audit trail
```

**Sample Telegram output:**
```
Intake Daily Digest - May 17

Score: 9/10 - Software Engineer at Tempus AI
Why: Strong Python + healthcare AI overlap
Missing: Go, Kubernetes
Link: [apply]

Score: 8/10 - Backend Engineer at Accenture
Why: FastAPI and PostgreSQL match exactly
Missing: Java
Link: [apply]
```

---

## Flow 2 — Evening Follow-up Reminder (6pm daily)

Reads your application log, detects anything overdue, drafts a status-aware follow-up email using Claude for each one, and sends reminders to Telegram.

```
[Schedule: 6pm]
       │
       ▼
[python.Script]
Read application log
Parse status + days since applied for each entry
       │
       ▼
[python.Script]
Detect overdue applications
  → applied >= 7 days, no reply  →  standard follow-up
  → applied >= 14 days, no reply →  ghosted nudge
  → phone screen >= 5 days       →  thank-you + timeline ask
       │
       ▼
[python.Script]
Claude drafts a follow-up email for each overdue app
Status-aware: different tone and content per situation
       │
       ▼
[python.Script]
Send reminders to Telegram
       │
       ▼
[log.Log]
Audit trail
```

**Sample Telegram output:**
```
Intake Follow-up Reminders

⏰ Tempus AI - Software Engineer (9 days ago)
Draft: Hi [Name], I wanted to follow up on my
application for the Software Engineer role...

👻 Google - Software Engineer (16 days ago)
Draft: I appreciate the time to consider my
application. I wanted to check in one final time...
```

---

## Flow 3 — Weekly Analytics Digest (Sunday 9am)

Computes pipeline health metrics, asks Claude to analyze patterns and suggest improvements, then pauses and waits for your manual approval before sending the digest to Telegram.

```
[Schedule: Sunday 9am]
       │
       ▼
[python.Script]
Compute pipeline metrics
  → total applications
  → applied this week
  → response rate %
  → interview count
  → ghosted count (>14 days, no reply)
  → top missing skills across JDs
       │
       ▼
[python.Script]
Claude analyzes the pipeline
  → overall health score (1-10)
  → headline summary
  → top insight
  → one action item for next week
       │
       ▼
[python.Script]
Build formatted digest
       │
       ▼
[flow.Pause]  ◄─── HUMAN REVIEW GATE
Execution pauses here.
You review the digest in Kestra UI.
Click Resume + approve to send.
       │
       ▼  (on approval)
[python.Script]
Send weekly digest to Telegram
       │
       ▼
[log.Log]
Audit trail
```

**Sample Telegram output:**
```
Intake Weekly Report - May 17

🟡 Pipeline Health: 6/10
Solid volume but follow-up rate needs attention.

This Week:
Applied: 8 new roles
Total: 23 applications
Response rate: 26%
Interviews: 2
Ghosted: 4

AI Insight: Your response rate improves on roles
where you mention healthcare AI experience.

Action: Prioritize roles with FHIR or clinical
data keywords next week.
```

---

## Why Kestra

- If a step fails, the Kestra UI shows exactly which task failed and why — re-run just that task, not the entire flow
- KV store handles deduplication across runs without needing a database
- Pause and Resume enables human-in-the-loop review without building a UI
- Open source and self-hosted, no per-run pricing as it scales
- Scales without a rewrite — one user today, ten tomorrow, same YAML

---

## Kestra Features Demonstrated

| Feature | Used In |
|---|---|
| `Schedule` trigger | All 3 flows |
| `http.Request` plugin | Flow 1 — Adzuna API |
| `python.Script` tasks | All 3 flows |
| `kv()` reads | API keys across all flows |
| `kv.Set` write | Flow 1 — persist seen job IDs |
| `outputs` passed between tasks | All 3 flows |
| `flow.Pause` + Resume | Flow 3 — human approval gate |
| `log.Log` | All 3 flows — audit trail |
| Expressions `{{ }}` | All 3 flows |

---

## Setup

### 1. Start Kestra

```bash
docker compose up -d
```

Open `http://localhost:8080`

### 2. Get API Keys

| Key | Where |
|---|---|
| Anthropic | console.anthropic.com |
| Adzuna App ID + Key | developer.adzuna.com (free) |
| Telegram Token | @BotFather → /newbot |
| Telegram Chat ID | api.telegram.org/bot{TOKEN}/getUpdates |

### 3. Add to Kestra KV Store

Namespaces → intake → KV Store → add each:

```
ANTHROPIC_API_KEY
ADZUNA_APP_ID
ADZUNA_APP_KEY
TELEGRAM_TOKEN
TELEGRAM_CHAT_ID
SEEN_JOB_IDS        []
```

### 4. Load the Flows

Flows → Create → paste each YAML:
- `flow1_morning_digest.yml`
- `flow2_followup_reminder.yml`
- `flow3_weekly_analytics.yml`

### 5. Test

Hit Execute on Flow 1. Check Telegram. If a message arrives, all 3 flows will run on schedule automatically.

---

## Stack

| Component | Purpose |
|---|---|
| Kestra | Orchestration, scheduling, KV store, human review |
| Claude Sonnet | Job scoring, follow-up drafting, pipeline analysis |
| Adzuna API | Job search (free, no scraping) |
| Telegram Bot API | Delivery |
| Python | Task scripts |

---

## Cost

~$0.05/week at current Anthropic pricing. A small VPS to self-host Kestra (~$5/month) is the main cost.

---

## License

MIT

## Acknowledgments

Built for the [WeMakeDevs Kestra Challenge](https://wemakedevs.org/orchestration).
Thanks to Kunal Kushwaha and Will Russell for running this.
