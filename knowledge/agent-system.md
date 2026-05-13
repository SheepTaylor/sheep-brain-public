# Agent System

## Runtime Contracts

### Environment Tiers
Controlled by `AGENT_ENV` (set via `direnv` + `.envrc`):
| Value | Worktree | DB |
|---|---|---|
| `scratch` | `~/projects/scratch-<name>` | `~/data/scratch-<name>/agent.db` |
| `candidate` | `~/projects/agent-system-candidate` | `~/data/candidate/agent.db` |
| `live` | `~/projects/agent-system` | `~/data/live/agent.db` |

### DB Access Rule
**Always** use `shared.db.get_db()`. Never open a DB file path directly. The function resolves the correct path from `AGENT_ENV`.

### Observations Table Schema
```sql
CREATE TABLE observations (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_name  TEXT NOT NULL,
    domain      TEXT NOT NULL,
    payload_json TEXT NOT NULL,   -- JSON string; dual-written for forward compatibility
    created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
```

## Agent Roles
| Agent | Trigger | Output |
|---|---|---|
| Domain agents (financial, health, etc.) | systemd timer (scheduled) | Rows in `observations` |
| Synthesis agent | Weekly timer | Telegram message with cross-domain summary |

## Systemd Timers
No live timers yet — system is still in development. When active:
- `candidate-db-refresh.timer` — nightly live→candidate DB snapshot
- `agent-synthesis.timer` — weekly synthesis run
- Timer unit files live in `~/.config/systemd/user/`

## Life KB Write Rule
Agents **read** `~/agent-shared/life/`; they **never write** to it. All KB edits are human-curated.
