# Peer Review: Calendar, Tasks, and Cross-Domain Insights (Design Draft v1)

**Reviewer:** Antigravity (agy)  
**Context:** [`plans/2026-10-18-agent-core-plan.md`](file:///home/sheep/agent-shared/knowledge/plans/2026-10-18-agent-core-plan.md), [`agents/heartbeat_agent.py`](file:///home/sheep/projects/agent-system/agents/heartbeat_agent.py), [`shared/telegram.py`](file:///home/sheep/projects/agent-system/shared/telegram.py), [`shared/riskwatch.py`](file:///home/sheep/projects/agent-system/shared/riskwatch.py), [`agents/gamecocks_docs.py`](file:///home/sheep/projects/agent-system/agents/gamecocks_docs.py), and `/tmp/calendar-design/draft-v1.md`.

---

## Objections & Suggestions

### 1. The "CC" Calendar Dependency and Fuzzy Deduplication
* **Issue:** Section A proposes an ongoing deduplication engine that fuzzy-matches events across 10 calendars within a $\pm15\text{ min}$ window and consumes the "CC" calendar's LLM descriptions as enrichment.
* **Why:** 
  1. The draft notes in Finding 19 that CC's source is unknown (likely an external Apps Script or Gemini scheduled action). Building an agent that architecturally depends on parsing the outputs of an unmaintained third-party LLM script running at 05:44 is backwards and brittle.
  2. Fuzzy title/location matching across shared calendars in a household of two is hazardous. If Janice has an appointment in Lexington at 10:00 and Sheep has one in Columbia at 10:00, or if both have haircuts or dentist visits at similar times, fuzzy matching risks false-positive merges or false-negative duplicate spam.
  3. Fuzzy string matching requires non-trivial dependencies or fragile heuristics, violating Sheep's standard library and minimal implementation requirements.
* **Proposed change:**
  Do not build a fuzzy deduplication engine. Treat "Moose and Sheep" (and Sheep's primary) as the sole human ground truth. Before writing code, determine with Sheep what CC is. If CC was an earlier automated attempt at calendar enrichment, retire it and let our own local agent pipeline enrich events directly from authoritative sources ([`mail_mirror`](file:///home/sheep/projects/agent-system/agents/mail_index_agent.py), bills, and health sync). If CC must stay, sync only "Moose and Sheep" and ignore CC during ingestion.

---

### 2. Premature Google Tasks Sync & OAuth Scope Expansion
* **Issue:** Section A designs a `tasks_sync` sensor, a new `tasks` SQLite table, and expands the OAuth scope to include Google Tasks (`tasks.googleapis.com`), while Section C adds `/todo` writing to Google Tasks.
* **Why:**
  1. Finding 5 explicitly admits: *"Google Tasks: no connector or OAuth on the box today; usage unknown (ask Sheep)."* Designing an entire sync subsystem and expanding OAuth scopes before confirming whether Sheep actually uses Tasks is classic premature engineering.
  2. Google Tasks is a distinct API from Google Calendar. Requesting both Calendar and Tasks scopes increases friction on personal, unverified Google accounts (`rick.taylor2030@gmail.com`).
  3. Sheep interacts with the agent primarily via Telegram. If Sheep wants quick task tracking (`/todo`), an internal SQLite table requires zero external APIs, zero extra OAuth scopes, and zero network failure modes.
* **Proposed change:**
  Cut Google Tasks API and `tasks_sync` from v1 entirely. Keep OAuth strictly scoped to Calendar. If Sheep wants `/todo` in Telegram, back it with a minimal local `agent_tasks` table in SQLite (`shared/db.py`). Only revisit Google Tasks sync if Sheep explicitly confirms he depends on the Google Tasks app UI on his Pixel/tablet.

---

### 3. Privacy: LLM Sensitivity Classifier vs Deterministic Boundaries
* **Issue:** Section A proposes using local Gemma to classify events into `medical / people-private / ordinary`, and allows cloud tools to see redacted summaries (*"appointment (private), 10:45–11:45, Lexington"*).
* **Why:**
  1. The core architectural boundary approved in [`plans/2026-10-18-agent-core-plan.md`](file:///home/sheep/agent-shared/knowledge/plans/2026-10-18-agent-core-plan.md#L58-L59) is absolute: medical and people data **never leave the box**. Relying solely on an LLM prompt to classify sensitivity fails open if Gemma misclassifies a doctor's name, clinic, or family appointment.
  2. Passing redacted time, duration, and geographic location to Claude leaks metadata and pattern-of-life information to the cloud without giving Claude any actionable value. Claude does not need to know Sheep has a private appointment in Lexington to review monthly budgets or check food recipes.
* **Proposed change:**
  Retain deterministic, fail-closed boundaries:
  1. Whitelist calendars: Subscribed calendars (Gamecocks, US Holidays) are cloud-safe. Personal and shared family calendars ("Moose and Sheep", "Taylor Family") are classified locally using the existing [`SENSITIVE`](file:///home/sheep/projects/agent-system/agents/telegram_bot.py#L40-L45) regex (matching medical specialties, clinical terms, doctor prefixes, family roles) as a hard floor.
  2. Any appointment flagged as medical, private, or family must stay **100% in the local lane** (evaluated only in [`heartbeat_agent.py`](file:///home/sheep/projects/agent-system/agents/heartbeat_agent.py) or locally generated Telegram replies).
  3. Cloud tools (`calendar_window()`) must never receive redacted medical placeholders—they simply do not see private/medical entries at all.

---

### 4. Heartbeat Gate: "Non-Routine Items Today" Breaks Exception-Based Silence
* **Issue:** Section B proposes adding *"calendar has non-routine items today"* as an activation condition for the 07:30 morning digest in [`heartbeat_agent.py`](file:///home/sheep/projects/agent-system/agents/heartbeat_agent.py).
* **Why:**
  1. [`heartbeat_agent.py`](file:///home/sheep/projects/agent-system/agents/heartbeat_agent.py#L10) is designed on the contract: *"Nothing → silent."*
  2. In an active household, almost every day has some non-routine item (an errand, dinner with friends, voting, a haircut, an appointment). Triggering on any non-routine item turns the heartbeat from an exception-based alert into an unconditional daily morning newspaper.
  3. As Finding 2 notes, Sheep's Pixel 10 already displays calendar notifications and home screen widgets. Pushing ordinary events over Telegram duplicates the phone OS and burns the daily Telegram message budget ([`shared/telegram.py`](file:///home/sheep/projects/agent-system/shared/telegram.py#L7-L11)).
* **Proposed change:**
  Do not allow ordinary calendar events to wake a silent heartbeat on their own. The morning digest should only include a "Today" calendar block if:
  - The heartbeat was **already triggered** by another system change (new bills, Monarch ledger import, feed issue, or due follow-up), OR
  - The calendar event itself represents a specific cross-domain collision (e.g. travel day impacting checking balance, or Gamecocks kickoff broadcast info).
  If the day only contains standard scheduled appointments, let the Pixel calendar handle the reminders and allow the heartbeat to remain silent.

---

### 5. Polling Frequency: 30-Minute Polling Timer is Excessive
* **Issue:** Section A specifies a 30-minute systemd timer polling Google Calendar v3 REST via incremental `syncToken`.
* **Why:**
  1. None of the existing agents run every 30 minutes. All batch agents run on daily/weekly timers ([`agent-financial.timer`](file:///home/sheep/projects/agent-system/infra/agent-financial.timer), [`agent-bills.timer`](file:///home/sheep/projects/agent-system/infra/agent-bills.timer), [`agent-heartbeat.timer`](file:///home/sheep/projects/agent-system/infra/agent-heartbeat.timer)).
  2. Sheep is retired; calendar events are planned days or weeks in advance, not updated minute-to-minute. 48 sync runs per day creates unnecessary disk I/O, network traffic, and risk of `syncToken` expiry or rate limits.
* **Proposed change:**
  Run `calendar_sync` on a relaxed cadence:
  - 2–3 times daily: e.g. at 07:00 (as an `ExecStartPre=` or prior timer to the 07:30 heartbeat) and once at 17:00.
  - Opportunistically on demand if Sheep runs `/today` or `/week` in Telegram and the local SQLite cache is older than 2 hours.

---

### 6. Evening Heads-Up: Unclear Triggers and Message Budget Impact
* **Issue:** Section B proposes a new evening Telegram slot (~1–3 per month) for *"fasting, early start (< 09:00), travel day, a cancellation deadline tomorrow"*.
* **Why:**
  1. There is no evening timer in the system today.
  2. Triggering on *"early start (< 09:00)"* will fire much more often than 1–3 times a month and will quickly become annoying noise. An early start does not require agent intervention; Sheep sets an alarm clock.
  3. The daily proactive budget is 2–4 messages and is already frequently exceeded (Finding 3). Adding another proactive slot without hard deterministic gating risks further budget degradation.
* **Proposed change:**
  Eliminate the generic "early start" trigger. If an evening alert is introduced, restrict it to a strictly deterministic check running at 19:30 that fires **only** if an event tomorrow contains an explicit actionable prerequisite (e.g., medical fasting instructions or a non-refundable booking cancellation deadline expiring that night). If nothing requires preparation before sleep, no process runs.

---

### 7. Grounding of Cross-Domain Insights (Section D)
* **Issue:** Several proposed cross-domain insights in Section D assume data or capabilities the system does not possess:
  - *"flag it before booking/at booking time"*: Calendar sync only reads events *after* they are booked.
  - *"Appointment tomorrow × MyChart: prep list (meds list, questions, last results)"*: MyChart C-CDA XML summaries do not contain clinic-specific preparation orders for upcoming appointments.
  - *"steps fall ~40% on travel days"*: Pure retrospective LLM trivia with zero decision value, violating Sheep's rule: *"Keep implementations minimal — resist premature abstraction and over-engineering."*
* **Why:** The system must not promise insights that cannot be verified or grounded against real SQLite tables.
* **Proposed change:**
  Prune Section D down to the three concrete, high-leverage insights grounded in current data:
  1. **Booked Travel vs Cash Flow:** Matching trip date ranges from the calendar against [`bills.project()`](file:///home/sheep/projects/agent-system/shared/bills.py) to flag when an upcoming trip overlaps with a projected low balance floor.
  2. **Gamecocks Game Day Context:** Pulling opponent, kickoff time, and TV broadcast details into the morning digest from [`gamecocks_docs.py`](file:///home/sheep/projects/agent-system/agents/gamecocks_docs.py) / web checks on game days.
  3. **Birthdays & Annual Milestones:** A 7-day heads-up in the digest for entries in the birthdays calendar or `people/` KB to allow time to send a card.

---

## Points of Agreement with Claude's Draft

I strongly agree with the following design choices in `draft-v1.md`:

1. **Dedicated "Agent" Secondary Calendar for Actuators (Section C):**
   Writing follow-ups, renewals, and cancellation reminders to a dedicated secondary Google Calendar ("Agent") rather than polluting Sheep's primary calendar with `privateExtendedProperty=agent_core=1` is a distinct improvement over the 2026-10-20 plan. It provides complete blast-radius isolation, distinct visual styling on the Pixel/tablet, and trivial 1-click cleanup if Sheep ever wants to clear agent-generated entries.
2. **Social Media & Messaging Out of Scope (Section E):**
   Firmly declaring X, Meta, Instagram, SMS, and WhatsApp out of scope is correct. Third-party APIs are either non-existent, paid, or require brittle on-device automation that directly contradicts the home-lab Python architecture.
3. **No Third-Party SDKs (Section A):**
   Using standard library `urllib` for Google Calendar v3 REST calls with the refresh token secured in `~/.config/` (mode 600) adheres to project conventions and avoids introducing bloated Google API client dependencies.
4. **Non-Duplication of Phone Alarms (Section B):**
   Relying on the Google Calendar Android app for timed event alarms and routine habit reminders (pills, blinds, pet meds), rather than attempting to duplicate them over Telegram, is the correct operational posture.
5. **On-Demand Telegram Commands (Section B):**
   Providing `/today` and `/week` as user-initiated Telegram commands (replies are unbudgeted) gives Sheep full visibility on demand without imposing proactive noise.

