# Calendar, tasks and cross-domain insights: agreed design v3 (2026-10-25)

Claude + Antigravity (Gemini 3.8 Flash High; Argon was out of quota), 4 rounds, CONFIRMED (round 4 covered Sheep's answers).
Conversation 293fcfc8-8f2f-4df1-9d31-024fab950c23. Status: approved by Sheep 2026-10-25. Step 1 built (see Build log).

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
  singleEvents=true, −7…+60 days, the whole window re-read and replaced each sync (simpler than
  syncToken, which Google won't combine with timeMin/timeMax; cancelled instances drop out), all-day dates kept as dates (Gamecocks UTC all-day events), zoneinfo America/New_York.
- Calendars: Moose and Sheep, primary, Agent, US holidays. CC (Gemini artifact; Sheep stops the
  Gemini process, then deletes it himself), Taylor Family (nothing to act on) and birthdays
  (social media covers them) are not synced. "Access to all calendars" (Sheep, 2026-10-25) means
  local code and local Gemma; cloud models still see none of his events.
- Games: subscribed calendars are empty for MBB/WBB/baseball, so the source is the
  gamecocksonline.com schedule tables gamecocks_docs.py already parses. New `--schedules-only`
  refresh (4 fetches) as ExecStartPre of the heartbeat → `games` table (sport, date, opponent,
  home/away, location, time or TBA, TV from the Media column if present). Year inferred from season.
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
3. Car conflict (one car): a Moose/Janice-prefixed and a Sheep/Rick-prefixed timed event whose
   [start−30 min, end+30 min] overlap at different or unknown places. Unprefixed = both go.
   Also two of Sheep's appointments ≤ 30 min apart in different places.
4. Gamecocks game today, all four sports, on game day only: opponent, home/away, time and TV when
   assigned, else "time TBA".
5. Chore or task due today or overdue (Phase 2).

## Phase 2: dropped 2026-10-28 (Sheep)
Google Tasks is out. Chores from maintenance.md and cleaning.md reach Sheep through Leo's daily picks
(`2026-10-28-leo-chief-of-staff-plan.md`): [Done] writes `chore_log`, due = max(chore_log, md),
maintenance.md "Last performed" stays human-curated.

## Local look-ahead (adopted 2026-10-25, experimental)
Gemma, Monday, appended as plain text after the review: next 14 days deduped + games + due chores +
health_signals → max 3 lines, each must name a conflict or trade-off; no restating the calendar. $0.
Dropped on ≥ 2 👎 in 3 weeks or if Sheep dismisses it; silence is not feedback.

## Out of scope
X, Meta, Instagram, SMS, WhatsApp: no usable personal API, ~no email signal (1 X email in 30 days).
Tablet: mute the bot there so the Pixel is the one place alerts ring.

## One-time recommendations (Sheep acts, no code)
- Calendar bill reminders ("Chase Statement", "Pay Boa card") duplicate the bills agent's checks.
- Stop the Gemini process that fills CC, then delete the CC calendar.
- The Gamecocks football calendar becomes redundant (optional unsubscribe).

## Testing
pytest, real scratch DB via shared.db.get_db(), recorded JSON fixtures for urlopen: normal +
recurring sync, 410 resync, token revoked → feed, dedupe/merge, DST Nov 1, all-day UTC dates,
cancelled instances, gate rules, evening keywords.

## Cost
Phase 1–2: $0 model cost (deterministic). Optional look-ahead: local, $0.

## Build log
- 2026-10-25 step 1 (scratch/calendar): migration 021 (cal_events, cal_sync_log, agent_cal_events,
  games); shared/gcal.py (OAuth auth-start/auth-finish, sync, Agent-calendar upsert/remove);
  shared/agenda.py (/today, /week text, dedupe, routine filter, one-car conflicts); gamecocks_docs
  --schedules-only (year from the weekday); feeds "Google Calendar" + "Gamecocks schedules";
  agent tool calendar_reminder (risk watch puts obligations on the Agent calendar); query_db and
  db_schema refuse cal_events. TV networks: the schedule pages carry none, so the line has no TV.
- Step 2 (digest Today block, action-item gate, 19:30 evening check, Monday local look-ahead,
  [Undo] for agent reminders) waits for the nightly financial_agent/health_agent sends to stop.
