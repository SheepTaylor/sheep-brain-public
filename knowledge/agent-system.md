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

## Model Routing (`shared/llm.py`)
Cost is a standing constraint: each agent uses the cheapest model that does its job.

| Function | Backend | Use for |
|---|---|---|
| `ask_local(prompt, system=None, ...)` | llama-server on 127.0.0.1:8080 (Gemma 4 26B-A4B), free, private | Anything touching raw ledger/health/family data; drafts; classification |
| `ask_claude(prompt, system, model=CHEAP)` | `claude-haiku-4-5` default; pass `BETTER` (`claude-sonnet-5`) when quality matters; `BEST` (Opus) never by default | Escalations, weekly synthesis |
Both raise `ModelUnavailable` so callers degrade a step instead of failing.
`system` must be the static part (persona, KB indexes) so it prompt-caches; volatile text goes in `prompt`.

## Secrets
`~/.config/agent-system.env` (mode 600, never committed) holds `ANTHROPIC_API_KEY` (plus `ANTHROPIC_WORKSPACE_ID` if the key is org-scoped), `TELEGRAM_BOT_TOKEN`,
`TELEGRAM_CHAT_ID`; optional `MONARCH_EXPORT_DIR`, `HEALTH_EXPORT_DIR`. Loaded by systemd `EnvironmentFile=` and by `shared/env.py`
for interactive runs. VaultGuard entry names: `anthropic_api_key`, `telegram_bot_token`, `telegram_chat_id`.

## Agent Roles
| Agent | Trigger | Input | Output |
|---|---|---|---|
| `monarch_sync_agent` | `ExecStartPre` of the financial unit | newest `Transactions_*.csv` and `Balances_*.csv` in `feeds/monarch/` on the Drive mount (manual Monarch export: Settings → Data → Download; Monarch has no API) via `shared/monarch_csv.py` | upserts into `fin_accounts`, `fin_transactions`, `fin_balances`; one `fin_sync_log` row |
| `financial_agent` | daily 00:00 | `fin_*` tables via `shared/ledger.summarize()`; exits 1 with a message if the tables are empty (checkbook fallback retired 2026-09-12) | one `observations` row: local-model sentence + JSON summary (net worth, 7/30-day spend by category and merchant, needs-review count, `days_stale` since the last transaction; over 7 days means a fresh export is due) |
| `health_sync_agent` | `ExecStartPre` of the health unit | newest `HealthSummary_*.zip` in `feeds/mychart/` on the Drive mount (Epic MyChart → Sharing → Download my record; IHE XDM zip of C-CDA `DOC*.XML`) via `shared/ccda.py` | upserts into `health_facts` (one row per problem, medication, allergy, immunization, lab result, vital, procedure, encounter, device, care-team member, social item, screening topic, clinical note); one `health_sync_log` row; skips the parse if that exact file was already loaded; re-applies `health_overrides` (Sheep's corrections, e.g. `python3 -m shared.health override 'hm|DTaP/Tdap/Td Vaccine (1 - Tdap)' --status completed --note '...'`) after every load |
| `health_agent` | weekly Sun 00:00 | `health_facts` via `shared/health.summarize()`; exits 1 if empty | one `observations` row (domain `health`): local-model paragraph + JSON summary (active problems/meds, allergies, last vitals, abnormal results 90d, screenings due within 30d, last/upcoming encounters, `days_since_sync`; over 120 days nags for a fresh export) + Telegram note. **Local model only** — the record never leaves the machine |
| `synthesis_agent` | weekly Mon 00:00 | last 7 days of `observations` | `reports` row + Telegram message; local draft → Sonnet 5 final → Telegram, degrading a step at a time |
| `telegram_bot` | always on (`agent-telegram-bot.service`) | messages from `TELEGRAM_CHAT_ID` only | answers from the life KB via `shared/kb.py` (two-tier routing, local model, Haiku on low confidence, `/ask more` Sonnet, `/ask deep` Opus; health/people never leave the box); `/status`, `/report`, `/model`. See `skills/telegram-bot.md` |
| `notify_agent` | `OnFailure=telegram-alert@%n.service` on every unit; `restore-drill.service` | unit name (+ journal tail) or stdin | Telegram alert |

## Systemd Timers
Live in `~/.config/systemd/user/`; tracked copies in `infra/`. Install with `cp infra/*.service infra/*.timer ~/.config/systemd/user/ && systemctl --user daemon-reload` (then `enable --now` any new unit).
- `agent-financial.timer` — daily
- `agent-synthesis.timer` — weekly
- `agent-health.timer` — weekly (Sunday), health sync then health pulse
- `candidate-db-refresh.timer` — nightly live→candidate DB snapshot
- `restore-drill.timer` — quarterly `~/scripts/restore-drill.sh` from the offsite mirror, result to Telegram
- `agent-telegram-bot.service` — the routing bot (not a timer; `Restart=always`)
- `telegram-alert@.service` — template pulled in by `OnFailure=` on every unit above
- `llama-server.service` — local model API (see `knowledge/plans/2026-09-08-local-ai-system-plan.md`)

## Life KB Write Rule
Agents **read** `~/agent-shared/life/`; they **never write** to it. All KB edits are human-curated.
