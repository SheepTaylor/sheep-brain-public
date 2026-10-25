# Calendar, tasks and cross-domain insights: agreed design v2 (2026-10-25)

Claude + Antigravity (Gemini 3.8 Flash High; Argon was out of quota), 3 rounds, CONFIRMED.
Conversation 293fcfc8-8f2f-4df1-9d31-024fab950c23. Status: awaiting Sheep's approval. Nothing built.

## Principles
- The phone (Google Calendar app) keeps every timed reminder; the system never repeats them.
- Personal and family calendar events never reach a cloud model. No classifier: everything about
  them is deterministic local code, appended to Telegram as plain lines.
- Calendar can wake Telegram only when something needs Sheep to act. "Nothing → silent" stays.
- No proactive calendar message ships until the nightly financial_agent / health_agent sends stop.

## Phase 1: Calendar (read + Agent calendar)
- One OAuth identity (rick.taylor2030@gmail.com), stdlib urllib, consent screen "In production",
  refresh token mode 600 under ~/.config. Scopes: calendar.readonly + calendar.events for the Agent
  calendar only (writes restricted to its calendarId in code).
- `shared/calendar.py` sync, run as ExecStartPre of the 07:30 heartbeat and the 19:30 evening check,
  and on demand for /today, /week when the cache is > 2 h old. No standalone timer.
  singleEvents=true, −7…+60 days, syncToken incremental, 410 → full resync, cancelled instances
  removed, all-day dates kept as dates (Gamecocks UTC all-day events), zoneinfo America/New_York.
- Calendars: Moose and Sheep, primary, Taylor Family (if Sheep says yes), Gamecocks (chosen sports),
  US holidays, birthdays (if shared from the Workspace account), CC (only if Sheep keeps it, read as
  optional description text).
- Dedupe: exact start minute (or all-day date); identical titles collapse, different titles join
  with " / ". Routine recurring reminders (blinds, Call mom, pills, pet meds, Brita) are never shown.
- Feed health in shared/feeds.py: token revoked → "run python3 -m shared.calendar_auth"; last
  sync > 36 h → late.
- New "Agent" calendar replaces the 2026-10-20 agent_core=1 tagging: riskwatch renewals, cancellation
  deadlines, follow-ups as all-day events with one popup; [Undo] in the digest.
- Telegram: /today, /week (unbudgeted replies) ship first, before the budget prerequisite.

## Notification layout
| Layer | When | Content | Wakes Telegram? |
|---|---|---|---|
| Phone | as Sheep set | his own reminders | n/a |
| Morning digest 07:30 | daily, only if gated | "Today" block: today's deduped non-routine items, tasks due (Phase 2) | only for an action item (below); otherwise rides along when the digest goes out anyway |
| Evening check 19:30 | daily, silent unless | tomorrow's event text has prep keywords (fasting / nothing to eat / bring a driver / prep instructions) or a cancellation deadline tonight/tomorrow | yes, budgeted |
| Monday review | weekly | the action lines from the week, plus (optional) local look-ahead | existing message |
| On demand | any time | /today, /week, /todo | replies |

Action items that may wake the digest (all deterministic):
1. New trip (multi-day lodging event) first seen whose dates overlap a projected checking low from
   bills.project(), or still inside its cancellation window — once.
2. Cancellation deadline within 48 h.
3. Two appointments ≤ 30 min apart in different places, or Sheep and Janice both away at the same
   time (car conflict — if they share one car).
4. Birthday in 7 days (if Sheep wants it).
5. Gamecocks game today (if opted in; kickoff/TV from the gamecocks docs when available).
6. Task due today or overdue (Phase 2).

## Phase 2: Google Tasks (after Phase 1 is stable and Sheep answers how he uses Tasks)
- Same OAuth client, add tasks scope. Read his lists → due/overdue into the gate and Today block.
- /todo <text> [date] adds to his default list; /todo lists open items. No agent-owned list.

## Optional: local look-ahead (Tier 2, Sheep's choice)
Gemma, Monday, local section: next 14 days + tasks + health_signals → max 3 lines, each must name a
conflict or trade-off; no restating the calendar. $0. Experimental; dropped if it draws 👎.

## Out of scope
X, Meta, Instagram, SMS, WhatsApp: no usable personal API, ~no email signal (1 X email in 30 days).
Tablet: mute the bot there so the Pixel is the one place alerts ring.

## One-time recommendations (Sheep acts, no code)
- Calendar bill reminders ("Chase Statement", "Pay Boa card") duplicate the bills agent's checks.
- CC duplicates Moose and Sheep: keep or retire depending on what it is.

## Testing
pytest, real scratch DB via shared.db.get_db(), recorded JSON fixtures for urlopen: normal +
recurring sync, 410 resync, token revoked → feed, dedupe/merge, DST Nov 1, all-day UTC dates,
cancelled instances, gate rules, evening keywords.

## Cost
Phase 1–2: $0 model cost (deterministic). Optional look-ahead: local, $0.
