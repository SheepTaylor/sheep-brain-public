# Agent Core — self-correcting, self-learning redesign

Status: direction and decisions approved 2026-10-18 (see Decisions); not built. No dollar figures or
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

A task registry (`agent/tasks.toml`) names each task's model, chosen by eval score per dollar,
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

## Autonomy tiers (decided 2026-10-18)

`agent/autonomy.toml` lists each actuator as `off | propose | act`. Sheep's rule: **spending money
and deleting files always need approval**; everything else may act, logged and reversible.

| Actuator | Tier | Notes |
|---|---|---|
| `send_telegram`, `schedule_followup`, `remember` | act | within the message budget below |
| Calendar: create/update reminders and events | act | agent-created events tagged so they can be found and undone |
| Gmail: draft replies, label/file mail | act | sending stays `propose` until a track record exists (outward-facing) |
| KB edits (`life/`, Codex Vitae, goals) | act via git commit, reported in the next digest | one-line revert; deletions of files or sections → `propose` |
| Playbook/lessons edits from reflection | propose → act for low-risk classes after a track record | auto-rollback on falling feedback |
| Anything that spends money (orders, bookings with charges, subscriptions) | propose, always | |
| File or record deletion (Drive, KB, DB rows, email trash) | propose, always | |
| Moving money between accounts | off | never an actuator |

## Message budget (decided 2026-10-18)

Two to four proactive Telegram messages a day; critical messages are never held back.

- **Slots:** morning digest (~07:30) carries everything non-urgent from the last 24 h; optional
  evening note only if something new is worth it. That leaves 1–2 for timely items. Replies to
  Sheep's own messages and approval prompts he asked for don't count.
- **Queue, don't drop:** a non-critical item that would exceed the budget waits for the next
  digest. The heartbeat ranks the queue by goal relevance.
- **Critical (bypass the budget):** projected checking below the floor or a missed/failed bill
  payment; a security or backup failure; a health signal the local lane marks urgent; a hard
  deadline within 48 h (renewal, filing, appointment); an approval that blocks something time-bound.
  Unit failures are batched into the digest unless the unit is one of the sensors behind a
  critical check.
- **Learns:** a 👎 on a critical message tightens that class; the reflection run reports the
  daily count and how many criticals were warranted.

## Goals become first-class

`life/goals/` (human-curated; draft written 2026-10-18 from Drive `system/frameworks` and the
"Living Well With AI" brief): `values.md` (priorities plus the tests every recommendation must
pass), `goals.md` (active goals by life subsystem, each with signal, source, target and cadence),
`codex-vitae.md` (the cognitive framework template, unfilled). Health goals are evaluated in the
local lane; only `health_signals` reach Claude.

Review loops follow the Life System Architecture's time scales:

| Loop | Run | Model | Output |
|---|---|---|---|
| Daily | heartbeat + morning digest | Haiku triage, Sonnet when there's something to say | digest; critical alerts any time |
| Weekly | review conversation ("council": doctor, best friend, future 85-year-old self) | Sonnet; health section local | what moved, where drifting, smallest next action per domain, ≤ 2 questions (one may be a Codex Vitae question) |
| Weekly | reflection on the agent itself | Opus | playbook/lesson proposals, eval additions |
| Monthly | goal review + "earns its keep" audit | Opus | progress per goal; subsystems unused for 90 days proposed for retirement |
| Quarterly | recalibration | Opus | goal/target changes proposed; Codex Vitae revisit; Sheep's time spent on the system |
| Annual | values refinement, estate documents check | Opus | proposals only |

### Guardrails from the brief

The "Living Well With AI" brief warns that a personal system can become maintenance that replaces
living. The design builds that in: messages must end in an action or not be sent; reflective work
(judgment, learning) gets questions, not answers; the monthly audit retires what isn't used and
prefers an off-the-shelf tool where one does the job; Sheep's weekly hours on the system are a
tracked goal; health stays custom because of privacy.

## What retires

`financial_agent`, `budget_agent`, `health_agent` (paragraph part), `synthesis_agent`, most of
the bot's routing in `telegram_bot.py` (it becomes transport + agent call; `/status` stays
deterministic as the model-down fallback). `sports_analyst` and `gamecocks_docs` merge into one
sensor + tool. Sensors, `finance_api`, notify/OnFailure, backups and SDLC tiers stay.

## Phases (each runs in shadow before it replaces anything)

0. **Foundations.** `agent_runs`, `agent_feedback` (Telegram reactions), budget governor,
   `tasks.toml`. No behaviour change. Built 2026-10-18 (migration 010).
1. **Agent core, read-only.** `shared/agent.py`, the tools above, `health_signals` from the local
   lane. Shadow the Monday review alongside the current synthesis for 3 weeks. Built 2026-10-18: no
   `send_telegram`/`schedule_followup`/`remember` tools yet (the review runner sends; memory and
   follow-ups move to Phase 3); `health_signals` is computed in code, not by Gemma. Shadow weeks:
   Oct 19, Oct 26, Nov 2.
2. **Self-correction.** `verify.py` claims check, critic pass, `agent_predictions` + grader.
3. **Learning loop.** Playbook, lessons, memory, weekly Opus reflection, eval suite seeded from
   history.
4. **Consolidate.** Daily Haiku heartbeat replaces the financial/budget nags; Telegram becomes
   the agent; retire the paragraph agents one at a time.
5. **Act.** KB edits, calendar and Gmail-draft actuators per the autonomy table; spending and
   deletion always via approval.

## Decisions

1. Health split: local Gemma lane, Claude for the rest; best model per task within the monthly cap.
2. Autonomy and message budget: as above.
3. Goals: drafted in `life/goals/` for Sheep's review; "proposed" targets need confirmation.
4. Only Sheep uses the system (one Telegram chat); other family members appear only as subjects
   in `people/` and `health/`, which stay in the local lane.
