# Agent System

## Component Roles

| Component | Type | Reads | Writes |
|---|---|---|---|
| Domain agents (financial, schedule, health) | Scheduled | live DB + relevant `life/` files | observation rows in shared DB |
| Synthesis agent | Scheduled (weekly) | observation log + `life/` indexes | synthesis report row + Telegram push |
| Routing agent (Telegram bot) | On-demand | `life/_index.md` → domain `_index.md` → leaf files | nothing |
| Claude Code / Gemini CLI | Interactive | `agent-shared/` tree + project-local files | code in worktrees |

**Critical constraint:** No agent — scheduled or interactive — writes to `agent-shared/life/`. The life KB is human-curated through git. This eliminates agent-vs-agent file-write conflicts on life/ entirely.

## Life Knowledge Routing

Queries against `life/` use a two-tier routing pattern to keep token cost flat:

1. Always loaded: `life/_index.md` (~500 tokens, prompt-cached).
2. Router selects domain → loads `<domain>/_index.md` (~300 tokens, prompt-cached).
3. Router selects 1–2 leaf files (~1,500 tokens each).

Typical per-query token cost: ~3,500 tokens regardless of total KB size. See `skills/life-knowledge-routing.md` for the full routing protocol.

## Shared Database Conventions

- Single SQLite file per environment tier: `~/data/<env>/agent.db`.
- WAL journal mode enabled in `shared/db.py` for concurrent reads + writes.
- All database access goes through `shared/db.py` — no agent opens its own connection.

### Dual-Write Observation Rows

Every observation row carries both a rich-text assessment and a structured JSON payload:

```
text:    "Spending on convenience food increased 40% this week"
payload: {"category": "convenience_food", "delta_pct": 40,
          "baseline_window_days": 28, "absolute_delta_usd": 87.50}
```

Both are written in the same transaction by the domain agent. The synthesis agent reads the text (Claude reasons better over prose); deterministic queries, dashboards, and future visualisation read the JSON.

### Schema

Schema migrations live in `shared/migrations/` and are applied on `init_db()`.

## Environment-Aware Database Path

`shared/db.py` resolves the correct DB path from environment variables:

```python
ENV = os.environ.get("AGENT_ENV", "scratch")   # default to safest
SCRATCH = os.environ.get("AGENT_SCRATCH_NAME", "default")

DB_PATHS = {
    "live":      Path.home() / "data/live/agent.db",
    "candidate": Path.home() / "data/candidate/agent.db",
    "scratch":   Path.home() / f"data/scratch-{SCRATCH}/agent.db",
}
```

Default is `scratch` — an unconfigured shell cannot accidentally reach the live DB.

## Cost Containment

- The Telegram bot is decoupled from the Claude API. Routing and templated responses run free; only the synthesis agent (weekly) and a subset of complex Telegram queries call the API.
- Prompt caching is exploited via static `_index.md` files at the root and domain levels. Fresh user query content is the only uncached portion of each call.
- Token usage is monitored via `TrackedAnthropic` wrapper feeding a Streamlit dashboard backed by SQLite.

## Scheduling

Agents run on **systemd user timers** (not cron) in Phase 6 onwards. Timers provide cleaner logs and easier monitoring than crontab entries. Domain agents run on their own cadence; synthesis agent runs weekly.

## Telegram Interface

Telegram is the only end-user interface. No web dashboard is in scope. The routing agent translates a user's natural-language question into a life-KB query, calls the Claude API with the relevant loaded context, and returns the response.
