# Calendar, tasks and cross-domain insights: design draft v1 (Claude, 2026-10-25)

Status: DRAFT for peer review. Nothing built. Sheep approves before any change.

## Sheep's request (2026-10-25)
Expand the agent system to everyday calendar and task tracking. Sources he named: Google Drive,
Google Calendar, Google Tasks; Pixel 10 (Android 17) and Samsung Tab A10; X, Meta, Instagram;
SMS, WhatsApp, Telegram. Personal calendars, calendars shared with Moose (Janice) and family,
subscribed Gamecocks football/basketball/baseball. Wants: a layout that gives timely notifications
without being overwhelming, and insights synthesized across all domains.

## What the data shows (examined 2026-10-25, Aug 1 – Dec 31 window)

Calendars (10, read via the claude.ai Google Calendar connector):
| Calendar | Owner | What's in it |
|---|---|---|
| Moose and Sheep | shared (Sheep + Janice) | The real working calendar: both people's appointments (many medical), haircuts, dinners, pet meds (Simparica, heartworm), "Chase Statement", "Pay Boa card", Brita filter, voting; daily "Open blinds"/"Close blinds" recurring 08:00/18:00 |
| rick.taylor2030@gmail.com (primary) | Sheep | Daily habit reminders: "Call mom" 18:30, "Take Flowmax" 17:30 (since Oct 2); travel/restaurant reservations auto-added from Gmail |
| CC | Sheep (owner) | Machine-made duplicate of Moose and Sheep + Gmail bookings, every event created ~05:44 daily, descriptions enriched by an LLM ("Plan to arrive by", "Cost", "Cancel before…", "Nothing to eat after midnight", "Possibly for: James"). Source not on this machine (no code, no Claude scheduled task) — probably a Gemini scheduled action or Apps Script |
| rick@jricktaylor.com | Sheep (Workspace) | Only 2010-era yearly birthdays (5 in the window) |
| Taylor Family | Janice's domain calendar | Family events incl. other relatives' medical appointments (Laura, Janice's mother), birthdays, trips |
| USC Football / M+W Basketball / Baseball | subscribed | All-day events, no kickoff time, no TV network. Football remaining: Oct 31 @OU, Nov 7 TAMU, Nov 14 @ARK, Nov 21 UGA, Nov 28 @Clemson |
| US Holidays | subscribed | |

Findings:
1. **Duplication**: the same appointment appears 2–3 times (Moose and Sheep + CC + primary). Any digest must dedupe or it triples noise.
2. **The phone already nags**: Moose and Sheep uses default reminders; blinds twice daily, pills, pet meds, calls. Timed reminders already reach the Pixel (and the tablet). Telegram must not repeat them.
3. **Telegram is already over budget**: 2026-10-25 had 9 budgeted proactive messages (target 2–4), partly a one-off model test, but every night financial_agent (00:01) and health_agent notes (00:05) also send. Calendar must add ~0 new daily messages.
4. **Sensitive content**: the calendars are full of medical appointments (Sheep's colonoscopy, urologist, PSA bloodwork; Janice's doctors; relatives'). The approved agent-core plan (2026-10-20) said the agent reads ONLY its own events (`agent_core=1`) so medical/family entries stay invisible to Claude. This request needs reading them → privacy design must change explicitly.
5. **Google Tasks**: no connector or OAuth on the box today; usage unknown (ask Sheep).
6. **Social / messaging**: X, Meta, Instagram send ~nothing to Gmail (1 X email in 30 days). No usable personal APIs (X API is paid; Meta/Instagram give no personal-feed API; Instagram Basic Display retired 2024). SMS/WhatsApp have no server-side access; only an on-phone app could read them. Propose: out of scope.
7. Useful enrichment already exists: CC's prep notes (fasting, bring driver, cancellation deadlines, costs); mail mirror (`~/data/mail/gmail`) has the booking emails; bills agent projects checking; riskwatch makes dated follow-ups; gamecocks docs agent has schedules.

## Proposed design

### A. Sensors (deterministic, local, free)
- `calendar_sync` (every 30 min, systemd timer): Google Calendar v3 REST via stdlib urllib with the OAuth refresh token already planned for Phase 5 (consent screen "In production", token mode-600 under ~/.config). Pulls a whitelist of calendars (Moose and Sheep, primary, CC, Taylor Family, Gamecocks ×4, birthdays) for −7…+60 days into SQLite `cal_events` (calendar, event id, start, end, all_day, title, location, description, recurring flag, sensitivity class). Uses syncToken incremental sync.
- **Dedupe** into `cal_items`: same start (±15 min) and fuzzy-matching title/location across calendars = one item; CC's description kept as the enrichment (prep notes). Recurring routine reminders (blinds, Call mom, pills, Brita, pet meds) flagged `routine` and never surfaced (the phone handles them).
- **Sensitivity classifier (local Gemma, cached per event)**: medical / people-private / ordinary. Medical and other-relatives' events never leave the box: cloud tools see them as "appointment (private), 10:45–11:45, Lexington" or not at all.
- `tasks_sync`: Google Tasks API (same OAuth, scope tasks) → `tasks` table. Pending Sheep's answer on how he uses Tasks.

### B. Notification layout (no new daily messages)
| Layer | Channel | What | Volume |
|---|---|---|---|
| Timed reminders | Google Calendar app on the Pixel (as today) | The event alarms Sheep already set | unchanged; system never duplicates |
| Morning digest 07:30 (existing heartbeat) | Telegram | New "Today" block: non-routine items today and anything needing prep tomorrow; tasks due today/overdue; Gamecocks game today (kickoff + TV from the gamecocks docs/ESPN) | 0 new messages; the gate gains "calendar has non-routine items today" |
| Evening heads-up (planned evening slot) | Telegram, budgeted | Only when tomorrow needs prep before morning: fasting, early start (< 09:00), travel day, a cancellation deadline tomorrow | ~1–3 per month |
| Weekly look-ahead | inside the Monday review (existing) | Next 14 days: busy days, trips, conflicts (two people, one car?), cost of upcoming trips vs cash-flow projection, deadlines | 0 new messages |
| On demand | Telegram `/today`, `/week`, `/todo` (add/list/done) | | replies, unbudgeted |

Prerequisite to stay at 2–4/day: finish the planned retirements (financial_agent nightly message; nightly health notes folded into the digest or weekly).

### C. Actuators (act tier, logged, reversible)
- Calendar writes only to a dedicated "Agent" calendar Sheep subscribes to (cleaner than tagging events in primary): riskwatch renewals, cancellation-deadline reminders, follow-ups. Never edits his events.
- Tasks writes only to an "Agent" task list; `/todo` adds to Sheep's default list (he asked for it).
- [Undo] in the digest, as already planned.

### D. Cross-domain insights (the synthesis)
The weekly review agent gets two new read tools: `calendar_window(days)` (deduped, sensitivity-redacted) and `tasks_open()`. Examples it can draw from data that exists today:
- Trips on the calendar × bills projection: a trip like the Oct 9–12 Lebanon stay ($455.54 hotel, cost already in CC's notes) landing in a week the bills agent projects checking low (it warned today of $3,743 on Nov 17) → flag it before booking/at booking time.
- Cancellation deadlines in CC notes × follow-ups → reminder the day before.
- Calendar-entered bill reminders ("Chase Statement", "Pay Boa card") × bills agent verification → tell Sheep which calendar reminders the bills agent now covers (he may delete them; propose only).
- Travel/busy days × steps and sleep (local lane only): "steps fall ~40% on travel days" type observations, reported as health_signals.
- Appointment tomorrow × MyChart (local lane only): prep list (meds list, questions, last results) in the evening heads-up, composed locally.
- Gamecocks game days × family events: conflicts flagged; game kickoff in the digest.
- Birthdays (people domain, local lane): 7-day heads-up in the digest so there's time to send a card.

### E. Out of scope (recommended)
X, Meta, Instagram, SMS, WhatsApp: no reliable data path without on-phone automation or paid APIs; low signal. Revisit only if Sheep names a concrete need (e.g. "remind me of WhatsApp family-group plans").
Tablet: same Telegram account; suggest muting the bot on the tablet so the Pixel is the one place alerts ring.

### F. Open questions for Sheep
1. Privacy: may the system read your own calendars at all? If yes, is "local model sees everything, cloud sees medical/family entries only as 'private appointment'" acceptable?
2. Google Tasks: what do you use it for today, and on which lists?
3. CC calendar: what creates it? Keep it (its prep notes are useful) or replace it with the agent?
4. Which calendars count? Taylor Family includes other relatives' appointments — include or skip?
5. Evening heads-up: OK as a rare budgeted message, or digest only?
6. Gamecocks: which sports get a game-day line (football only, or basketball/baseball too)?
