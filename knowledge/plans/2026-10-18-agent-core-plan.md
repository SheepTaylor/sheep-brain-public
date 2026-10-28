# Agent Core — self-correcting, self-learning redesign

Status: Phases 0–4 LIVE (2026-10-18 to 2026-10-20); Calendar Step 1 LIVE (2026-10-25, `2026-10-25-calendar-plan.md`); KB change proposals LIVE (2026-10-23, `2026-10-23-kb-change-approval.md`). Pending cutover on 2026-11-02 (after 3-week shadow trial): retire `synthesis_agent` and `financial_agent`. Note: sections "Self-learning", "Autonomy tiers", and "Message budget" below are the original 2026-10-18 draft and are **superseded** where noted by "Revision 2026-10-20: Phases 3–5" at line 203. No dollar figures or
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
| Monthly | goal review | Opus | progress per goal |
| Quarterly | recalibration | Opus | goal/target changes proposed; Codex Vitae revisit; family archive check-in |
| Annual | values refinement, estate documents check | Opus | proposals only |

### Guardrails

Messages must end in an action or not be sent; reflective work (judgment, learning) gets
questions, not answers; health stays custom because of privacy. (The brief's "presence over
system" and "earn its keep" principles, and the time-tracking goals, were dropped by Sheep on
2026-10-18.)

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
2. **Self-correction.** `verify.py` claims check, critic pass, `agent_predictions` + grader. Built 2026-10-18:
   up to 2 checked revision rounds (one round let the revision introduce new errors); `alerts` tool
   added after a run re-raised a charge Sheep had dismissed; dry runs void their forecasts.
3. **Learning loop** (revised 2026-10-20, see below). Built 2026-10-20 (1d43cbf): two design points
   settled with Antigravity during the build (learned rules as atomic rows appended at load time, not git
   commits from the live worktree; reflection inputs whitelisted), and its code review found four bugs, fixed.
4. **Consolidate** (revised 2026-10-20). Built 2026-10-20 (f46e2d1) except the retirements, which wait for Nov 2 and Sheep's
   OK; build design agreed with Antigravity (5 changes) and its code review found three bugs, fixed.
5. **Act** (revised 2026-10-20).

## Revision 2026-10-20: Phases 3–5 after an independent review

Reviewed with Antigravity (Gemini 4 Argon) over three rounds; every change below was agreed by both reviewers.
Where it differs from earlier sections of this plan (self-learning items 2, 4–7, the autonomy table, the message
budget), this section wins.

### Phase 3: learning loop
- **Memory (P1).** One `agent_memory` table (kind preference | fact | dismissal, text, source, created_at, expires_at,
  superseded_by) and a `remember` tool. All active rows are injected into the prompt, capped at ~60 rows / ~1.5k
  tokens, newest first; no `recall` search tool, because the model can't search for a dismissal it doesn't know
  exists. Answered questions stay injected alongside. No `lessons.md`, no `health-playbook.md`, no local reflection.
- **Playbook.** `agent/playbook.md` (git) replaces the review's hard-coded instructions; rules learned by
  reflection are edits to it.
- **goal_status() (P2).** A deterministic tool computing each goal's current value, target and status from the
  goals that have a data source, so goal figures stop drifting between runs. `health_signals` gets per-metric
  plausibility bounds and labels "heart rate min" as the daily minimum, not resting heart rate. The matching goals.md wording ("resting HR") is offered
  to Sheep as an edit, not made silently.
- **Reflection (P3, A2).** Monthly, plus a triggered run after ≥3 👎 or ≥3 shipped unverified figures / reviewer
  doubts since the last one. Opus. Inputs: feedback, answered questions, graded forecasts, shipped unverified
  figures and doubts, tool errors and 0-row queries, cost per run. Output: a playbook diff with [Apply]/[Reject]
  buttons and at most 3 new regression cases. Propose-only; no auto-apply and no auto-rollback. Silence is not
  feedback.
- **Revision guard (P4).** If a revision has more unsupported figures than the draft before it, ship the earlier
  draft. On revision rounds the critic sees the prior round's issues and must not reverse a fix it ordered unless
  that fix introduced a factual error.
- **Evals (P4).** Free deterministic replay of past failures in pytest (claims check, parsers, tool mocks); paid
  model/judge evals only through an explicit `python3 -m shared.evals` with a cost cap.
- **Follow-ups (P5).** `event_checks` gains `kind` (web | agent) and `run_id`; `schedule_followup(due_date, topic,
  question, context)` adds agent follow-ups, which `event_check_agent` runs through `agent.run_agent`. In Phase 3 a
  follow-up sends its own message; from Phase 4 it feeds the digest. `/check` lists both kinds.
- **Tool fixes (P11).** `query_db` / `db_schema` say that account names carry "(...xxxx)" suffixes and to match
  with LIKE; a 0-row result from an `=` / `IN` comparison on names appends that hint.

### Phase 4: consolidate
- **Heartbeat (P7).** Daily, gated by a deterministic diff (new bill emails, a new Monarch import, due follow-ups,
  wearable anomalies, a stale Monarch export (>7 days), queued messages); the model runs only when something
  changed. The digest gets the same checks as the review: claims check, critic only if it proposes an action (P12).
- **Message budget.** A per-day counter in `telegram.send` for non-critical proactive messages; over budget → an
  outbox row flushed into the next digest. Nothing is dropped; no ranking model.
- **Conversational bot (P6, A1).** Slash commands stay deterministic. Plain text goes through a local privacy
  pre-router first: health/people questions are answered entirely in the local lane; if the local router is down,
  anything matching health/people keywords is refused for cloud ("local model unavailable"). Everything else goes
  to the agent loop with the claims check and at most one revision; the critic runs only if an actuator/propose
  tool was called.
- **Risk watch, read-only (P8, moved from Phase 5).** An agent pass reads insurance, subscriptions, vehicles,
  maintenance and estate files and turns dated obligations into follow-ups due 35 days before the obligation
  (deduplicated by topic + date). Gated on file sha256/mtime in `agent_state`, plus a pass on the 1st of each month
  for seasonal maintenance. No date parser over free-form markdown; nothing new for Sheep to curate.
- **Retirements (P7, A3).** `synthesis_agent` only after the shadow comparison ends (Nov 2) and Sheep's explicit OK;
  then `financial_agent` entirely (its stale-export nag moves to the heartbeat). Stop the now-unread
  `observations` writes in `bills_agent`, `budget_agent`, `event_check_agent` and `health_agent`; the table stays as
  history and `/status` shows last runs from `agent_runs` and the sync logs. `budget_agent` (deterministic monthly
  note) and `health_agent`'s Sunday local health note stay.

### Phase 5: act
- **KB edits (P9).** Act for notes code doesn't parse (e.g. Codex Vitae answers): git commit, reported in the next
  digest with a [Revert] button. Propose ([Commit]/[Discard] with the diff) for files code parses (bills.md,
  budget.md, goals/*.md, */_index.md); their parsers must load cleanly before and after. Removing content is always
  propose. health/ and people/ stay off-limits to cloud tools.
- **Email (P10).** Drafts via IMAP APPEND to Gmail Drafts with the existing app password. Sending (smtplib, same
  password) only for a class Sheep approves after the monthly reflection reports its draft-approval record; no
  promotion state machine.
- **Calendar (Sheep wants calendar entries, 2026-10-20).** *Superseded 2026-10-25 by `2026-10-25-calendar-plan.md`: the system now reads Sheep's calendars locally (cloud tools never see them) and writes only to a dedicated "Agent" calendar.* Google Calendar v3 REST via stdlib `urllib` (no Google
  client library), one-time OAuth consent with the consent screen set to "In production" (in "Testing", refresh
  tokens expire every 7 days); the refresh token lives in a mode-600 file under ~/.config, never in git.
  Privacy: every read passes `privateExtendedProperty=agent_core=1`, so the agent sees only events it created and
  never Sheep's own entries (medical and family appointments stay invisible to Claude). Act tier for its own
  events: dated follow-ups (renewals, maintenance; not health appointments) become all-day events with a reminder,
  upserted idempotently by a deterministic key (`agent_key=<topic>:<date>`), each reported in the digest with an
  [Undo] button; `/check cancel <id>` removes its event too. Events it didn't create are untouchable.

## Decisions

1. Health split: local Gemma lane, Claude for the rest; best model per task within the monthly cap.
2. Autonomy and message budget: as above.
3. Goals: drafted in `life/goals/` for Sheep's review; "proposed" targets need confirmation.
4. Only Sheep uses the system (one Telegram chat); other family members appear only as subjects
   in `people/` and `health/`, which stay in the local lane.
