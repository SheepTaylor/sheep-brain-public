# Peer Review: Calendar, Tasks, and Cross-Domain Insights (Round 3)

**Reviewer:** Antigravity (agy)  
**Context:** Stress-testing the combined architecture across synthesis depth, OAuth topologies, and operational edge cases.

---

### A. Cross-Domain Synthesis Depth vs Noise: COUNTER
* **The tension:** With strict privacy preventing raw personal entries from reaching Claude, are 4 deterministic rules too thin to deliver Sheep's request for *"insights synthesized across all domains"*?
* **Analysis of the Gemma counter-option:**
  1. Sheep's foundational guardrail ([`plans/2026-10-18-agent-core-plan.md` L172](file:///home/sheep/agent-shared/knowledge/plans/2026-10-18-agent-core-plan.md#L172)) states: **"Messages must end in an action or not be sent; reflective work (judgment, learning) gets questions, not answers."**
  2. Local Gemma 4 26B-A4B given an unconstrained prompt over a 14-day list will almost certainly produce generic conversational filler (*"Monday looks busy with two appointments"*, *"No exercise events scheduled this week"*). For a retired 40-year IT manager, unprompted narrative statements with no decision attached are pure noise and will attract immediate 👎 feedback.
  3. However, deterministic synthesis does **not** have to be thin. In fact, code-level heuristics are far more capable of surfacing high-leverage friction points than an LLM guessing at priorities:
     - **Transit / Schedule density:** Consecutive appointments with $\le 30\text{ min}$ between them in different locations (e.g. West Columbia vs Lexington).
     - **Household car/resource conflicts:** Sheep and Janice having appointments across town at the same time.
     - **Actionable prep gating:** Doctor visits with explicit keywords (`"fasting"`, `"bring a driver"`, `"prep"`) surfaced the evening prior.
     - **Financial friction:** Booked trip dates overlapping projected checking low-point floors via [`bills.project()`](file:///home/sheep/projects/agent-system/shared/bills.py).
     - **Contractual deadlines:** Non-refundable cancellation deadlines expiring within 48 hours.
     - **Social:** 7-day advance notice for birthdays to mail physical cards.
* **Proposed Resolution for the Design Doc:**  
  Present Sheep with a clear choice between two tiers:
  - **Tier 1 (Recommended — Actionable Deterministic Synthesis):** Zero narrative fluff. Only fires when a specific conflict, cash collision, deadline, or prep action is detected. 100% reliable, zero hallucinations, $0 cost.
  - **Tier 2 (Optional Experimental Local Look-Ahead):** A local Gemma section in the Monday review. If selected, it must be constrained by a strict template forbidding calendar restatements ("busy Tuesday") and requiring every sentence to cite an explicit trade-off or conflict.

---

### B. Single OAuth Identity: AGREE
* **Assessment:** AGREE without reservation.
* **Why:**
  1. Managing two Google OAuth client registrations (client ID + secret), consent screens, refresh tokens, and token expiration lifecycles across personal Gmail (`rick.taylor2030@gmail.com`) and Google Workspace (`rick@jricktaylor.com`) is unjustifiable overhead for 5 annual birthdays.
  2. Google Calendar sharing is native: Sheep opens `rick@jricktaylor.com`, shares the birthdays calendar with `rick.taylor2030@gmail.com` (read-only). The Google Calendar API for the Gmail account immediately lists that secondary calendar under `/users/me/calendarList` with its own `calendarId`.
  3. Alternatively, the 5 birthdays can simply be curated into `~/agent-shared/life/people/` or added to "Moose and Sheep".

---

### C. Edge Cases, Failure Modes, and Technical Traps: AGREE + ADDITIONS
We agree on all points raised. Below are the concrete technical contracts required to address them:

1. **Failure Modes & Feed Health ([`shared/feeds.py`](file:///home/sheep/projects/agent-system/shared/feeds.py)):**
   - Add a `calendar(conn, now)` feed check.
   - If the refresh token exchange returns HTTP 400/401 (`invalid_grant`), report: `Feed("Google Calendar", "OAuth token revoked or expired (run python3 -m shared.calendar_auth)", detail)`.
   - If the last successful sync is older than 36 hours, flag as late.
   - **Google Calendar 410 Gone:** When an incremental `syncToken` expires or becomes invalid, Google returns HTTP 410. The sync engine must catch 410, clear the stored `syncToken`, and automatically perform a clean full resync.
2. **Recurring Event Expansion (`singleEvents=true`):**
   - The REST call must pass `singleEvents=true` and `timeMin`/`timeMax` ($-7\dots+60\text{ days}$). Without `singleEvents=true`, Google returns recurring events as abstract `RRULE` objects rather than discrete instances, completely breaking date-based dedupe and daily digests.
   - Google Calendar returns cancelled instances as `{"id": "...", "status": "cancelled"}`. SQLite `cal_events` upsert must delete or mark cancelled instances so deleted appointments disappear from the agent's view.
3. **Time Zones & All-Day Event Normalization:**
   - Timed events return `dateTime` with ISO offset (`2026-10-25T14:00:00-04:00`); all-day events return a bare `date` (`{"date": "2026-10-31"}`).
   - Subscribed sports schedules (Gamecocks) often store all-day events in UTC (`2026-10-31T00:00:00Z`). In US Eastern Time (UTC-4), naive timezone conversion shifts the event to the prior evening (`2026-10-30 20:00`). All-day events must be explicitly parsed as date-only objects pinned to the local calendar date without UTC offset shifting.
4. **Daylight Saving Time (DST Transition on Nov 1, 2026):**
   - The US falls back from EDT (UTC-4) to EST (UTC-5) at 02:00 on Sunday, Nov 1, 2026 (right in the initial sync window).
   - Python's standard library `zoneinfo.ZoneInfo("America/New_York")` must be used for all timestamp arithmetic so that 25-hour transition days and exact-start comparisons (`(date, start_time)`) do not drift.
5. **Testing Approach (SDLC & Tech-Stack Conformance):**
   - Zero mocking of SQLite: all tests run against a scratch DB via `shared.db.get_db()`.
   - Recorded JSON fixture mocks for `urllib.request.urlopen` covering:
     - Normal sync with single and recurring events.
     - HTTP 410 `syncToken` expiration handling and resync recovery.
     - HTTP 400/401 token revocation error reporting into `shared/feeds.py`.
     - Exact-start deduplication and title-merging logic.
     - Pruning of events outside the $-30\text{ day}$ rolling window.

---

## Remaining Disagreements
None. Every trade-off across privacy, deduplication, notification noise, OAuth scoping, and failure handling is fully aligned and grounded in the system's runtime contracts.

**CONFIRMED**

