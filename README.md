# Intake 🎯
### AI-powered job search assistant, orchestrated entirely by Kestra.

Built for the [WeMakeDevs Kestra Orchestration Challenge](https://wemakedevs.org/orchestration).

---

As an international grad student juggling coursework, projects, and a job search all at once, I was drowning in the logistics. Missing new postings. Forgetting to follow up. Losing track of where things stood.

So I built Intake. Three Kestra flows that run automatically. Everything lands on Telegram. Nothing to babysit.

---

## What It Does

```
Flow 1 (8am daily)
Fetch jobs → Score with Claude → Deduplicate via KV store → Send top matches to Telegram

Flow 2 (6pm daily)  
Check application log → Detect overdue → Draft follow-up email with Claude → Send to Telegram

Flow 3 (Sunday 9am)
Compute pipeline metrics → Analyze with Claude → Pause for human review → Send weekly digest
```

---

## Why Kestra

- If a step fails, the UI shows exactly why and I can re-run just that step
- KV store handles deduplication across runs without needing a database
- Pause and Resume handles human review without building a UI
- Schedule triggers handle everything else
- Kestra is the entire backend

---

## Stack

| Component | Purpose |
|---|---|
| Kestra | Orchestration, scheduling, KV store, human review gate |
| Claude Sonnet | Job scoring, follow-up drafting, pipeline analysis |
| Adzuna API | Job search (free, no scraping) |
| Telegram Bot | Delivery |
| Python | Scripts inside Kestra tasks |

---

## Kestra Features Used

- `Schedule` trigger — all 3 flows
- `http.Request` plugin — Adzuna API call
- `python.Script` tasks — all processing
- `kv.Set` and `kv()` — deduplication store
- `Pause` + Resume — human review before weekly digest sends
- `Log` task — audit trail on every run
- Expressions and outputs passed between tasks

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
| Adzuna App ID + Key | developer.adzuna.com |
| Telegram Token | @BotFather on Telegram |
| Telegram Chat ID | api.telegram.org/bot{TOKEN}/getUpdates |

### 3. Add to Kestra KV Store

Namespaces → intake → KV Store:

```
ANTHROPIC_API_KEY
ADZUNA_APP_ID
ADZUNA_APP_KEY
TELEGRAM_TOKEN
TELEGRAM_CHAT_ID
SEEN_JOB_IDS        []
```

### 4. Load the Flows

Flows → Create → paste each YAML file:
- `flow1_morning_digest.yml`
- `flow2_followup_reminder.yml`  
- `flow3_weekly_analytics.yml`

### 5. Execute Flow 1 to test

Hit Execute. Check your Telegram. If it works, all 3 flows run on schedule automatically.

---

## Screenshots

### All 3 flows running successfully
![Flows](screenshots/all_flows_success.png)

### Flow 1 — Morning job digest on Telegram
![Telegram](screenshots/telegram_digest.png)

### Flow 3 — Human review Pause before sending
![Pause](screenshots/flow3_pause.png)

### Flow topology
![Topology](screenshots/topology.png)

---

## Cost

~$0.05 per week at current Anthropic pricing. A small VPS to self-host Kestra (~$5/month) is the main cost.

---

## License

MIT

## Acknowledgments

Built for the [WeMakeDevs Kestra Challenge](https://wemakedevs.org/orchestration).
Thanks to Kunal Kushwaha and Will Russell for running this.
