# Agent System

## Runtime Contracts

### Environment Tiers
Controlled by `AGENT_ENV` (set via `direnv` + `.envrc`):
| Value | Worktree | DB |
|---|---|---|
| `scratch` | `~/projects/scratch-<name>` | `~/data/scratch-<name>/agent.db` |
| `candidate` | `~/projects/agent-system-candidate` | `~/data/candidate/agent.db` |
| `live` | `~/projects/agent-system` | `~/data/live/agent.db` |

### DB Access Rule
**Always** use `shared.db.get_db()`. Never open a DB file path directly. The function resolves the correct path from `AGENT_ENV`.

### Observations Table Schema
```sql
CREATE TABLE observations (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_name  TEXT NOT NULL,
    domain      TEXT NOT NULL,
    payload_json TEXT NOT NULL,   -- JSON string; dual-written for forward compatibility
    created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### Token Usage Table (migration 002)
Every model call made through `shared/llm.py` writes one row:
`token_usage(agent_name, model, input_tokens, output_tokens, cache_read_tokens, cache_write_tokens, cost_usd)`.
`shared.llm.month_cost_usd()` sums the current month.

## Model Routing (`shared/llm.py`, `agent/tasks.toml`)
Each task uses the best model for the job within the monthly API cap; health and people data stay local.
Which model a task uses is set in `agent/tasks.toml` (aliases `local`, `haiku`, `sonnet`, `opus`) and read with
`shared.tasks.model_for(task)`; change models there, not in code. Unknown task names raise `KeyError`.

| Function | Backend | Use for |
|---|---|---|
| `ask_local(prompt, system=None, ...)` | llama-server on 127.0.0.1:8080 (Gemma 4 26B-A4B), free, private | Anything touching raw ledger/health/family data; drafts; classification |
| `ask_claude(prompt, system, model=CHEAP)` | `CHEAP` `claude-haiku-5-5`, `BETTER` `claude-sonnet-5-5`, `BEST` `claude-opus-5-5`; thinking switched off per model (`THINKING_OFF`), except Opus 5.5, which can't disable it and runs at effort `low` instead | Escalations, weekly synthesis, `/ask` tiers |
| `ask_web(prompt, system, model=CHEAP)` | same models + server-side web search | Event checks |
Both raise `ModelUnavailable` so callers degrade a step instead of failing.
`system` must be the static part (persona, KB indexes) so it prompt-caches; volatile text goes in `prompt`.

**Budget governor.** `AGENT_MONTHLY_CAP_USD` in `~/.config/agent-system.env` caps cloud spend per calendar month
(unset = no governor). Telegram alerts once at 80% and once at 100% (flags in `agent_state`); at or over the cap
every cloud call drops one tier (Opus → Sonnet → Haiku → `ModelUnavailable`, so callers fall back to local).

## Agent core foundations (migration 010, Phase 0 of `plans/2026-10-18-agent-core-plan.md`)
- `agent_runs`: one row per unit of agent work via `with shared.runs.run(agent, trigger, input=...) as r: r.output = ...`
  (status `ok`/`error`). Model calls inside the block set `token_usage.run_id`; `runs.cost_usd(id)` sums them. The
  Telegram bot records one run per incoming message.
- `agent_messages`: every Telegram message delivered by `shared.telegram.send(text, agent=..., critical=...)`, with
  the Telegram message id, sender and run. `critical` marks alerts that will bypass the daily message budget
  (unit failures, bank alerts, projected low balance, cap reached).
- `agent_feedback`: Sheep's reactions (latest state per message; 👍-type = +1, 👎-type = −1) and replies to agent
  messages, captured by the bot (`allowed_updates` includes `message_reaction`). `/status` shows the 7-day tally.
- `agent_state`: small persistent flags (budget alerts so far).

## Self-correction (Phase 2)
- `shared/verify.py` claims check: every amount, percentage and date in a draft must appear in the run's evidence
  (system context, task prompt, tool results, web citations; rounding to the unit or cent shown, ±5% for "about",
  "~" or "k" figures) or be a sum/difference/ratio of supported figures the draft cites. Counts under 10 and
  years are not checked. Our own revision requests are never evidence.
- `shared/critic.py`: second opinion (task `critic`, Opus 5.5) on the draft against the evidence; flags unsupported
  facts, names, causes, actions and health statements beyond the signals; JSON issues.
- `shared/agent.run_agent(..., checks=True, critic=..., extract=...)` returns `AgentResult(text, run_id,
  unverified, critic, doubts)`; problems from both checks go back to the agent for up to `MAX_REVISIONS` (2)
  rounds, each re-checked.
- `agent_predictions` (migration 011): forecasts the agent records (`balance` of an account on a date,
  `budget_line` month total). `python3 -m shared.predictions` (ExecStartPost of `agent-financial.service`)
  grades open ones once Monarch covers the date: `actual`, `error`, `abs_pct`; status open/graded/ungradable/void.

## Secrets
`~/.config/agent-system.env` (mode 600, never committed) holds `ANTHROPIC_API_KEY` (plus `ANTHROPIC_WORKSPACE_ID` if the key is org-scoped), `TELEGRAM_BOT_TOKEN`,
`TELEGRAM_CHAT_ID`; optional `AGENT_MONTHLY_CAP_USD` (budget governor), `MONARCH_EXPORT_DIR`, `HEALTH_EXPORT_DIR`, `GMAIL_ADDRESS` + `GMAIL_APP_PASSWORD` (a Google app password; read-only IMAP for the bills agent). Loaded by systemd `EnvironmentFile=` and by `shared/env.py`
for interactive runs.

## Agent Roles
| Agent | Trigger | Input | Output |
|---|---|---|---|
| `monarch_sync_agent` | `ExecStartPre` of the financial unit | newest `Transactions_*.csv` and `Balances_*.csv` in `feeds/monarch/` on the Drive mount (manual Monarch export: Settings → Data → Download; Monarch has no API) via `shared/monarch_csv.py` | upserts into `fin_accounts`, `fin_transactions`, `fin_balances`; one `fin_sync_log` row |
| `financial_agent` | daily 00:00 | `fin_*` tables via `shared/ledger.summarize()`; exits 1 with a message if the tables are empty (checkbook fallback retired 2026-09-12) | one `observations` row: local-model sentence + JSON summary (net worth, 7/30-day spend by category and merchant, needs-review count, `days_stale` since the last transaction; over 7 days means a fresh export is due) |
| `bills_agent` | daily 07:10 (`agent-bills.timer`) | `life/finance/bills.md` (income schedule, bills with ledger/email match rules, balance goals), bill emails from Gmail over IMAP (`shared/gmail.py`, read-only, only the senders named in bills.md; local model extracts kind/amount/dates, Haiku fallback), `fin_*` tables | `bill_emails` rows (migration 007); Telegram alert once per projected low point under a balance goal; once a month, when the Monarch export covers it, a report of every bill (statement amount, payment date and amount, balance before → after), income received, unlisted payments over $50, and the next 5 weeks' projection (`reports` `bills_monthly:YYYY-MM` + one observation). Matching and projection are deterministic (`shared/bills.py`) |
| `health_sync_agent` | `ExecStartPre` of the health unit | newest `HealthSummary_*.zip` in `feeds/mychart/` on the Drive mount (Epic MyChart → Sharing → Download my record; IHE XDM zip of C-CDA `DOC*.XML`) via `shared/ccda.py` | upserts into `health_facts` (one row per problem, medication, allergy, immunization, lab result, vital, procedure, encounter, device, care-team member, social item, screening topic, clinical note); one `health_sync_log` row; skips the parse if that exact file was already loaded; re-applies `health_overrides` (Sheep's corrections, e.g. `python3 -m shared.health override 'hm|DTaP/Tdap/Td Vaccine (1 - Tdap)' --status completed --note '...'`) after every load |
| `health_connect_agent` | daily 23:30 (`agent-health-connect.timer`) | `feeds/Health Connect.zip` on the Drive mount, rewritten daily by the phone (Android Health Connect → Export and import → Scheduled export; Samsung Health carries the Galaxy Watch, Withings the scale) via `shared/health_connect.py` | one `health_facts` row per metric per day, category `wearable`, id `hc|<metric>|<date>` (steps from Samsung Health only, heart rate avg/min/max, sleep hours on the wake date + deep/REM minutes, SpO2, weight, body fat, exercise minutes); the first export was a full history, the scheduled ones a rolling ~90-day window that opens mid-day, so each load skips the export's first 2 days and re-upserts the rest; one `health_sync_log` row (source `health_connect`); skips the file if size and mtime are unchanged |
| `health_agent` | weekly Sun 00:00 | `health_facts` via `shared/health.summarize()`; exits 1 if empty | one `observations` row (domain `health`): local-model paragraph + JSON summary (active problems/meds, allergies, last vitals, abnormal results 90d, screenings due within 30d, last/upcoming encounters, `days_since_sync`, watch trends (7-day vs prior 28-day averages, last weigh-in; nags after 3 days without watch data); over 120 days nags for a fresh export) + Telegram note. **Local model only** — the record never leaves the machine |
| `synthesis_agent` | weekly Mon 00:00 | last 7 days of `observations` | `reports` row + Telegram message; local draft → Sonnet 5.5 final → Telegram, degrading a step at a time; health/people observations get a local-only section and never reach Claude |
| `event_check_agent` | daily 08:15 (`agent-event-check.timer`) | `event_checks` rows (migration 008) whose `due_date` has arrived; added with `/check YYYY-MM-DD topic \| question [\| baseline]` in Telegram (`/check cheap ...` for Haiku, `/check` lists, `/check cancel <id>`) or `python3 -m shared.event_checks add|list|cancel` | Sonnet 5.5 by default (Haiku 5.5 with `cheap`; accuracy on live figures is worth the ~$0.10) + server-side web search (`shared.llm.ask_web`, basic `web_search_20250305`: Haiku 5.5 rejects the 20260209 variant; Haiku ~$0.02 per check, searches billed at $0.01 each) → one Telegram message per check + one `observations` row (domain `world`). A `STATUS: not yet` answer is retried daily, up to 3 days, and the last attempt is sent anyway; model errors mark the check `failed` after 3 tries |
| `telegram_bot` | always on (`agent-telegram-bot.service`) | messages from `TELEGRAM_CHAT_ID` only | answers from the life KB via `shared/kb.py` (two-tier routing, local model, Haiku on low confidence, `/ask more` Sonnet, `/ask deep` Opus 5.5; health/people never leave the box); `/status`, `/report`, `/bills` (3-week cash-flow outlook), `/model`. See `skills/telegram-bot.md` |
| `review_agent` | weekly Mon 07:45 (`agent-review.timer`), shadow trial from 2026-10-19 alongside `synthesis_agent` | the agent core (`shared/agent.py` tool loop, model `weekly_review` in `agent/tasks.toml`, Sonnet 5.5) with read-only tools from `shared/agent_tools.py`: `query_db` (read-only connection, one SELECT, health tables denied), `db_schema`, `read_kb` (health/ and people/ refused), `cashflow`, `budget_status`, `afford_check`, `data_freshness`, `alerts` (bank/bill alerts already sent, with Sheep's dismissals), `health_signals` (whitelisted trends/counts computed locally, no record text), `record_prediction` (its only write: `agent_predictions`), server web search; system context = persona + `life/goals/` + life index | one Telegram message ("🧪 Agent core review, shadow trial week N": what moved vs goals, drift, next actions, ≤ 2 questions; the review is taken from `<review>` tags, markdown stripped) + `reports` row `shadow_review:YYYY-MM-DD`; never an observation. Self-correction: claims check + Opus critic, up to 2 checked revision rounds; the footer flags figures not found in the data and the reviewer's remaining doubts. Run trace in `agent_runs` (tools, turns, cost, `checks` per round); caps 15 turns / $0.75 per loop; ~$0.35 per review. `--dry-run` prints without sending and voids its forecasts |
| `notify_agent` | `OnFailure=telegram-alert@%n.service` on every unit; `restore-drill.service` | unit name (+ journal tail) or stdin | Telegram alert |

## Systemd Timers
Live in `~/.config/systemd/user/`; tracked copies in `infra/`. Install with `cp infra/*.service infra/*.timer ~/.config/systemd/user/ && systemctl --user daemon-reload` (then `enable --now` any new unit).
- `agent-financial.timer` — daily
- `agent-synthesis.timer` — weekly
- `agent-bills.timer` — daily 07:10, bill emails → verification, cash-flow projection, monthly bills report
- `agent-health.timer` — weekly (Sunday), health sync then health pulse
- `agent-health-connect.timer` — daily 23:30, Health Connect export → `health_facts`
- `agent-event-check.timer` — daily 08:15, due one-off event checks → web search → Telegram
- `agent-review.timer` — weekly Mon 07:45, agent core weekly review (shadow trial)
- `candidate-db-refresh.timer` — nightly live→candidate DB snapshot
- `restore-drill.timer` — quarterly `~/scripts/restore-drill.sh` from the offsite mirror, result to Telegram
- `agent-telegram-bot.service` — the routing bot (not a timer; `Restart=always`)
- `telegram-alert@.service` — template pulled in by `OnFailure=` on every unit above
- `llama-server.service` — local model API (see `knowledge/plans/2026-09-08-local-ai-system-plan.md`)

## Life KB Write Rule
Agents **read** `~/agent-shared/life/`; they **never write** to it. All KB edits are human-curated.
