# Leo — chief of staff agent (plan, 2026-10-28)

Status: agreed with Antigravity 2026-10-28 (2 rounds, CONFIRMED; conversation
d2ffce63-9ff3-4b65-832d-5f28cb9f0dc4, transcripts in `2026-10-28-leo-review/`). APPROVED by
Sheep 2026-10-28.

## What Sheep asked for
An agent named LeoMcGarry ("Leo"), Sheep's chief of staff. He manages Sheep's work with the
Claude Code agents, can read all the knowledge, advises on decisions, records statuses, and
manages software development and execution. Two meetings a day: a morning briefing (most of
the morning digest, questions and changes, the day's tasks, mood and energy) and an evening
debrief (results, tomorrow, mood and energy). Tone: Leo McGarry with President Bartlet,
seasons 1–2. Goal: help Sheep reach his goals and live by his values (`life/goals/`).

## Sheep's answers (2026-10-28)
1. Meetings start when Sheep says "Leo". Leo prepares at 07:30 and 18:00. If Sheep hasn't
   started one, the pack goes out by itself at 10:00 / 21:00 (no teaser), and only if it has
   something in it.
2. Leo takes over the heartbeat digest and the chat agent: one voice. Bills alerts, failure
   alerts and critical messages keep their own path.
3. Mood and energy stay local. Leo (Claude) sees only 1–5 scores and a 7-day trend.
4. Dev work: Leo tracks it, and may start a headless Claude Code job in a scratch worktree
   after Sheep taps Approve. He never promotes anything.

## Persona
Study: `~/projects/leo-research/leo-persona-study.md` (all 44 S1–S2 scripts from
westwingwiki.com, 168 Leo–Bartlet scenes; Antigravity re-checked the counts: median line 5
words, "sir" 105, "Mr. President" 99, "Jed" 2). It lives in `agent/playbook.md` as sections
**Leo** (Sheep only), **Briefing** and **Debrief**, so the reflection agent's learned rules
apply to it like any other task.
- Calls Sheep "sir"; Sheep calls him Leo.
- Short lines. Opens with the clock and the top item; no greeting paragraph, no summary.
- Gives his straight read on a judgment call, then asks for Sheep's call, then "Yes, sir" and
  records it. He argues before the decision, never after it.
- When unsure he says so plainly ("I don't know", "I'm not sure"), not with a rating word.
- Names a mood pattern once, from the scores ("third evening at energy 2"), never lectures.
- Every meeting ends with a short list: what Sheep is doing, what Leo is doing, by when.
- Mornings brisk (Situation Room); evenings warmer, the after-hours Oval Office voice.
- Defends the other agents' work when it's right and owns their mistakes when it isn't.
- Not borrowed: show quotes, a catchphrase in every message, politics, the backstory, or
  keeping things from the boss for his own good. Leo never withholds from Sheep.
- values.md still applies: act, don't just inform; explain why; Sheep decides; approval for
  spending money or deleting files.
- A meeting should take ≤ 5 minutes and ≤ 6 Leo messages unless Sheep keeps it going.

## How a meeting works
- **No meeting object.** Each message is a normal stateless turn (as the chat agent works
  today): `recent_exchanges()` threads the conversation, and today's pack in
  `agent_state['leo:YYYY-MM-DD']` is put in the prompt. "Leo" / `/leo` / the [Start] button
  delivers the pack and the check-in buttons. Decisions are saved during the turn with
  `remember` (new memory kind `decision`). No idle timer and no separate minutes call.
- Photos, recipe replies, label shots and Moose's chat keep their existing routes; routing
  happens before Leo sees a message. Moose never gets Leo: her chat stays the neutral
  household assistant, with no mood scores, picks or dev board.
- Runs on `shared/agent.py` (task `leo`, Sonnet 5.5) with critic off and at most one
  revision. Today's and tomorrow's dates and the local time are always in the prompt so the
  claims check doesn't flag "10:00" or "next 10 days".
- **The pack** is built without a model: the heartbeat gate's findings (bills, Monarch import,
  goal changes, feeds, due follow-ups, held messages), decisions waiting on Sheep (KB and
  reflection proposals, unanswered questions), today's picks, and one line when it isn't
  zero: "Waiting on you: 2 dev items (…)". Held messages and follow-up results go in as
  they are, never through a model (today's heartbeat rule). Cursors, follow-ups and the
  outbox move on only once the pack is delivered.

**Morning:** energy and mood (two rows of 1–5 buttons); the pack, most important first;
decisions with buttons; 1–3 tasks for today sized to his energy, with [Swap] [Drop] [Add];
the closing list. Sheep can ask questions or make changes at any point.

**Evening:** today's picks, each with [Done] [Not today] [Drop]; what changed since morning;
energy and mood; anything for tomorrow; tomorrow's short list.

## Mood and energy
- `checkins` table (day, kind, energy, mood, words, created_at). `query_db` refuses it, like
  `cal_events`, and the nightly seed DB for Moose empties it. Scores come from button taps,
  with no model involved. Words are saved only when Sheep replies to the check-in prompt
  (or uses `/note`); they are stored raw and never go to the cloud.
- Leo sees `mood_signal()`: today's scores, 7-day averages, a trend word. Numbers and
  direction only, like `health_signals`.
- Narrower than Sheep's answer, on purpose: a mood remark made in ordinary conversation is
  handled by the existing privacy pre-router (health terms already stay local) rather than
  being redacted from what Leo sees. Otherwise "I'm worn out, skip the yard work" would reach
  Leo without the request in it.

## The task pool (computed, not a new task system)
- Picked from: chores due from `maintenance.md` / `cleaning.md` plus `chore_log` (the
  calendar plan's table and rule: due = max(chore_log, md)); `deadlines.md` within 30 days;
  the weekly review's "For you this week" actions; agent follow-ups that need Sheep; and a
  small `todos` table only for things Sheep adds himself ("Leo, add: call Lowe's").
- Picking is rule-based first (due date, the order in values.md, effort against energy, not
  passed over 3 days running). Leo then words the list and may swap one item if he says why.
- [Done] on a chore writes `chore_log`. No KB proposals for routine chores: one per chore
  would be noise, and a second proposal on the same file goes stale.
- This makes the calendar plan's Google Tasks lists unnecessary (question below).

## Software development
- **Board** (`/leo board`, and a `dev_board` tool Leo calls when asked): each plan's status
  line and next phase from `knowledge/plans/`, scratch worktrees with unmerged commits,
  `staging` ahead of `master`, dated items waiting on Sheep. Built from files and git only,
  once at 07:30. Leo reminds; he does not edit plans.
- **Dispatch** (Phase 3, which gets its own design round with Antigravity first): Leo drafts a
  job brief from Sheep's own words and the plan, and Sheep taps [Approve]. It runs as a
  transient `systemd-run --user` unit, so the bot never blocks. The environment is set
  explicitly (`AGENT_ENV=scratch`, `PYTHONPATH` = the worktree, a seeded scratch DB) so a job
  cannot inherit the bot's `AGENT_ENV=live`. The worktree gets a `.claude/settings.json` that
  denies edits outside it, `git push`, `systemctl` and network fetches. Before and after the
  job, `git status` is checked on `agent-system` and `/srv/household/life`. When it finishes,
  the branch, test result and summary go to Telegram. No untrusted text (emails, web results,
  recipes) ever goes into a brief. Merging to staging and promotion stay with Sheep.

## Phases
0. **Morning briefing in the heartbeat slot.** Playbook sections, `checkins`, the pack builder
   (the heartbeat gate moves into it), "Leo" / [Start], check-in buttons, the 10:00
   self-send, deliver-then-advance. Leo takes over `heartbeat_agent`'s unbudgeted status and
   its place in reflection's cloud senders. The pack's local-only lines (held local messages,
   calendar lines) stay out of cloud reflection prompts. This phase adds no new proactive
   message, so it can go live before Nov 2.
1. **Tasks and evening debrief.** The pool, `todos`, `chore_log`, picks, [Done]/[Not today]/
   [Drop]. The evening pack is a new proactive message, so it goes live only after the Nov 2
   retirements (synthesis, then financial_agent) and Sheep's OK. Calendar step 2's Today block
   joins the morning pack at the same point.
2. **Board and chat.** The dev board; plain text outside meetings goes to Leo, replacing the
   chat agent and keeping the same privacy pre-router; the Monday briefing carries the weekly
   review's headline.
3. **Dispatch.** After the design round.

Each phase: pytest on a real scratch DB; persona evals (replayed meetings graded on brevity, a
recommendation with its reason, and the closing list); scratch → candidate → live; Sheep's OK
before live.

## Cost
Chat turns since Oct 15 have averaged $0.032 each (heartbeat $0.006). Two meetings of about
5 turns comes to roughly $0.35 a day, or $8–15 a month. October so far: $4.18 of the $30 cap.
Mood scoring costs nothing (buttons).

## Sheep's answers to the last questions (2026-10-28)
1. "Sir" is fine.
2. Google Tasks is dropped from the calendar plan; Leo's picks and `chore_log` replace it.
3. No weekly cap on dispatched Claude Code jobs for now.
