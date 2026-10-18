# Agent Core — self-correcting, self-learning redesign

Status: direction approved 2026-10-18; open decisions at the end; not built. No dollar figures or
personal details here: `knowledge/` is published. The monthly API cap lives in
`~/.config/agent-system.env` as `AGENT_MONTHLY_CAP_USD`.

## Why

Today every timer runs a script that does deterministic work and then asks a model for a
paragraph. The model decides nothing, calls no tools, never follows up, and never learns whether
its paragraph was useful. A better model barely improves that. The redesign moves judgment into
one agent with tools, memory and a feedback loop, so the system improves as models do and as it
accumulates evidence about what Sheep finds useful.

## Shape

```
SENSORS (deterministic, keep)        AGENT CORE                       ACTUATORS (tiered)
 monarch_sync, health_sync,           shared/agent.py tool loop          send_telegram
 health_connect, gmail bills,  ──►    wakes on: Telegram msg,     ──►    schedule_followup
 gamecocks, event sources             heartbeat, follow-up, event        propose_kb_edit
 → SQLite facts, no model prose       memory + playbook + lessons        calendar / gmail drafts
                                              ▲      │
              LOCAL HEALTH LANE               │      ▼
 health_facts ─► Gemma pipeline ─► health_signals (whitelisted schema, no record text)
                                              │
                        LEARNING LOOP: runs → feedback → graded predictions
                        → weekly reflection → playbook/lesson proposals → evals → adopt
```

The deterministic modules stay as plain functions and become tools: `shared/bills.py`
(`project()`), `shared/budget.py`, `shared/afford.py`, `shared/ledger.py`, `shared/kb.py`.

### Agent loop

A plain Anthropic Messages API tool-use loop in `shared/agent.py`, not a framework: minimal deps,
testable against a scratch DB, and every call already flows through `shared/llm.py` accounting.
Initial tools (read-only plus messaging):

| Tool | Notes |
|---|---|
| `query_db(sql)` | read-only connection, `fin_*`, `bill_emails`, `event_checks`, `observations`, `reports`; health tables refused |
| `read_kb(path)` / `search_kb(q)` | two-tier routing; `health/` and `people/` refused |
| `cashflow()`, `budget_status()`, `afford(amount, line)` | wrappers over existing modules |
| `health_signals()` | output of the local lane only (below) |
| `web_search` | server-side, as `ask_web` today |
| `send_telegram(text)` | every send gets a message id tied to the run, for feedback |
| `schedule_followup(when, why)` | generalizes `event_checks` into an agent task queue |
| `remember(note)` / `recall(q)` | agent memory, see Learning |

## Model routing: best model per task, within the cap

A task registry (`agent/tasks.yaml`) names each task's model, chosen by eval score per dollar,
not by a fixed ladder. Starting assignments:

| Task | Model | Why |
|---|---|---|
| Health lane: read record, compute signals, health summary, health critic | local Gemma | privacy rule: the record never leaves the box |
| PII/health scrub gate on every outbound prompt | local Gemma + regex | defence in depth |
| Email extraction, classification | local, Haiku fallback | volume, simple |
| Daily heartbeat triage ("is anything worth saying?") | Haiku 5.5 | cheap, runs every day, usually says no |
| Telegram conversations, weekly review, follow-ups | Sonnet 5.5 | multi-step tool use, numbers must be right |
| Weekly reflection, monthly goal review, eval judging on changes | Opus 5.5 | infrequent, highest-leverage reasoning |

A budget governor in `shared/llm.py` reads `token_usage` month-to-date: Telegram alert at 80% of
the cap; at 100% every task drops one tier (Opus→Sonnet→Haiku→local) until the month rolls over.
Re-assigning a task after a new model ships = run the evals, edit one line.

## Self-correction (within a run)

1. **Claims check.** Before any `send_telegram`, every number and date in the draft must appear
   in a tool result from the same run. A deterministic checker (`shared/verify.py`) extracts
   figures and matches them against the run's tool outputs; misses go back to the agent once
   ("these figures are unsupported: …"), then the message ships with them removed or flagged.
2. **Critic pass on high-stakes output** (money moves, health signals, anything proposing an
   action): a second model call reviews draft + evidence and returns pass / fix list. Health
   drafts use the local critic.
3. **Tool errors are observations, not crashes.** Errors return to the agent with the message; it
   retries or degrades. Three failures → `notify_agent`.
4. **Freshness guard.** Tools report data age (Monarch export, Health Connect, MyChart); the
   agent must state staleness rather than answer as if current.

## Self-learning (across runs)

1. **Run traces.** `agent_runs` table: trigger, model, tools called with args, final output, cost,
   verify/critic results. Everything else learns from this.
2. **Feedback capture.** Every agent message accepts a Telegram reaction (👍 / 👎) or a reply;
   stored in `agent_feedback` linked to the run. Silence is recorded as weak negative for
   unsolicited messages.
3. **Predictions graded against reality.** When the agent forecasts something checkable
   (checking low point, month-end budget line, bill amount, "export arrives by Tuesday"), it
   records it in `agent_predictions`; a deterministic job grades it when actuals land. Calibration
   by task feeds the reflection run.
4. **Memory.** `agent_memory`: facts Sheep told it, what it already said (no repeats), what was
   dismissed. Recalled by the agent via tool, not bulk-loaded.
5. **Playbook + lessons.** `agent/playbook.md` (versioned in git) is the agent's standing
   instructions: what's worth a message, tone, thresholds. `agent/lessons.md` holds distilled
   rules learned from feedback, each citing the runs that produced it.
6. **Weekly reflection (Opus).** Reads the week's traces, feedback, graded predictions, verify
   failures and cost; proposes playbook/lesson edits as a diff, plus any new tool or saved query
   it wished it had. Sheep approves in Telegram (tier: propose). After a track record the
   low-risk class (thresholds, phrasing) moves to auto-apply with automatic rollback if the next
   week's feedback drops.
7. **Evals as the ratchet.** `tests/evals/`: golden cases built from real history (past Telegram
   Q&A with confirmed answers, past bills reports, afford verdicts, graded predictions). Any
   change to playbook, tools, prompts or model assignment runs the evals; a regression blocks
   adoption. The reflection run adds a case for every confirmed mistake, so the same error is
   caught forever.

### Health lane learning

Same tables, local only: health messages carry feedback; a local reflection pass proposes edits to
`agent/health-playbook.md`. Today Gemma runs a fixed pipeline (it can't reliably drive a tool
loop); when a local model can, the lane gets the same loop with health tools. `health_signals` is
a whitelisted schema (metric, direction, magnitude, window, due-count, staleness) with no names,
diagnoses or record text; the scrub gate rejects anything else.

## Autonomy tiers

`agent/autonomy.yaml` lists each actuator as `off | propose | act`. Everything starts at
`propose` (Telegram ask, 👍 to execute) except `send_telegram`, `schedule_followup` and
`remember`. Promotion is Sheep's call, informed by the approval rate the reflection run reports.
Moving money is never an actuator.

## Goals become first-class

`life/goals/` (human-curated): Codex Vitae and values moved in from Drive `system/`, plus the
active goals, each with a measurable signal bound to existing data (budget lines, checking floor,
debt payoff date, sleep/steps, people contact cadence). The weekly review and monthly goal review
report progress against these, not raw summaries, and end with one suggestion and at most two
questions; answers are written to memory or proposed as goal edits.

## What retires

`financial_agent`, `budget_agent`, `health_agent` (paragraph part), `synthesis_agent`, most of
the bot's routing in `telegram_bot.py` (it becomes transport + agent call; `/status` stays
deterministic as the model-down fallback). `sports_analyst` and `gamecocks_docs` merge into one
sensor + tool. Sensors, `finance_api`, notify/OnFailure, backups and SDLC tiers stay.

## Phases (each runs in shadow before it replaces anything)

0. **Foundations.** `agent_runs`, `agent_feedback` (Telegram reactions), budget governor,
   `tasks.yaml`. No behaviour change.
1. **Agent core, read-only.** `shared/agent.py`, the tools above, `health_signals` from the local
   lane. Shadow the Monday review alongside the current synthesis for 3 weeks.
2. **Self-correction.** `verify.py` claims check, critic pass, `agent_predictions` + grader.
3. **Learning loop.** Playbook, lessons, memory, weekly Opus reflection, eval suite seeded from
   history.
4. **Consolidate.** Daily Haiku heartbeat replaces the financial/budget nags; Telegram becomes
   the agent; retire the paragraph agents one at a time.
5. **Act.** KB edit proposals (git commit on 👍), calendar and Gmail-draft actuators, autonomy
   promotion.

## Open decisions

1. Active goals and access to Codex Vitae / values in Drive `system/` to seed `life/goals/`.
2. Initial autonomy list beyond the defaults.
3. Message volume target (e.g. heartbeat silent unless something crosses a threshold).
4. Whether Janice is a user (second Telegram chat, shared memory) or only a subject.
