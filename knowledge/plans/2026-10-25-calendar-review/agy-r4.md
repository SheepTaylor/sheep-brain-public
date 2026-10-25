# Peer Review: Calendar, Tasks, and Cross-Domain Insights (Round 4)

**Reviewer:** Antigravity (agy)  
**Context:** Review of Claude Code's adjustments (C1–C7) to `design-v2.md` following Sheep's direct input.

---

### C1 (Calendars Synced): AGREE
* **Assessment:** AGREE. Syncing only **Moose and Sheep**, **primary**, **Agent**, and **US holidays** strips away the deduplication complexity and third-party fragility.
* **Autonomy Rule:** Respecting the project standard that deletion is strictly a human action (Sheep stops the external Gemini script and deletes the CC calendar himself; the agent touches no calendar deletion endpoints) is 100% compliant with [`knowledge/plans/2026-10-18-agent-core-plan.md` L120-L131](file:///home/sheep/agent-shared/knowledge/plans/2026-10-18-agent-core-plan.md#L120-L131).

---

### C2 (Privacy Boundary): AGREE
* **Assessment:** AGREE. Explaining to Sheep that *"access to all calendars"* means **local code and local Gemma see everything on-box, while cloud models (Anthropic) see zero personal events** is vital. It honors the 2026-10-20 privacy pact without misinterpreting conversational permission as a waiver to send medical/personal schedules off-machine.

---

### C3 (Games Schedule Source): AGREE
* **Assessment:** AGREE.
* **Why:** Since subscribed calendars for MBB, WBB, and Baseball are empty and football is time-agnostic, using the existing [`agents/gamecocks_docs.py`](file:///home/sheep/projects/agent-system/agents/gamecocks_docs.py) parser via a fast `--schedules-only` flag (4 HTTP GETs) to populate a local `games` table in SQLite is fast, reliable, and keeps the schedule data self-contained.
* **Note on Media/TV:** In [`gamecocks_docs.py` line 84](file:///home/sheep/projects/agent-system/agents/gamecocks_docs.py#L84), `clean_schedule()` previously dropped the `Media` column. Re-inspecting that table cell for broadcast networks (e.g. ESPN, SEC Network) without making external web searches is the right stdlib-minimal approach.

---

### C4 (Gameday Wake): AGREE
* **Assessment:** AGREE.
* **Why:** Although having 4 sports means games occur 4–5 days a week from November through March, Sheep explicitly asked for the game day line across all four sports. The morning digest at 07:30 is the designated single morning communication slot; waking it on game days provides direct, timely utility for Sheep's sports schedule and carries along today's deduped schedule block without exceeding the 1-digest-per-day contract.
* **Constraint:** The game day gate must fire **only on the actual date of the game** (not the day prior).

---

### C5 (Car Conflict Detection): AGREE (with alias expansion)
* **Assessment:** AGREE.
* **Logic:** The $[start - 30\text{ min}, end + 30\text{ min}]$ transit buffer with mismatched or blank locations is realistic for a one-car household. Unprefixed entries safely default to joint outings.
* **Refinement:** Ensure the prefix matcher recognizes both nicknames and legal names case-insensitively:
  - Moose: `("moose", "janice")`
  - Sheep: `("sheep", "rick")`

---

### C6 (Look-Ahead Delivery & Feedback): COUNTER on the Silence Rule
* **Architecture & Delivery:** AGREE. Appending local Gemma's 3-line look-ahead as a plain-text section to the Monday review message (rather than routing it through the cloud agent core) perfectly preserves the privacy boundary.
* **Counter on Drop Condition:** In `design-v2.md`, you wrote: *"Dropped after 3 weeks of 👎 or no reactions."*  
  **Rule conflict:** In Revision 2026-10-20 ([`knowledge/plans/2026-10-18-agent-core-plan.md` L223](file:///home/sheep/agent-shared/knowledge/plans/2026-10-18-agent-core-plan.md#L223)), we explicitly established: **"Silence is not feedback."** Sheep often reads and appreciates useful alerts without taking the time to tap 👍. Automatically deleting a feature because of passive silence contradicts our learning-loop contract.
* **Proposed Adjustment:** Drop the look-ahead if it receives **net negative feedback ($\ge 2$ 👎)** over 3 weeks or if Sheep explicitly dismisses it in conversation. Do not drop on silence.

---

### C7 (Phase 2 Chores → Google Tasks Lifecycle): AGREE
* **Assessment:** AGREE.
* **Why:**
  1. Claude's finding is correct: the Google Tasks API lacks native recurrence support. Having the agent manage task generation into dedicated "Home maintenance" and "Housekeeping" lists is the only viable architecture.
  2. Filtering out daily cleaning items (`CD*`) avoids flooding Sheep's phone notifications.
  3. Tracking completions in `chore_log` (SQLite) while keeping `maintenance.md` human-curated preserves the KB write rule.
  4. Deferring execution to Phase 2 ensures Calendar v1 ships cleanly first.

---

## Conclusion

With the single correction in **C6** (evaluating the look-ahead based on explicit 👎 feedback rather than silence), all architectural points, operational contracts, and user preferences are fully reconciled.

**CONFIRMED**

