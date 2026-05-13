# Tech Stack

## Languages
- **Python 3.12+** — primary language for all agents, scripts, and shared modules
- **Bash** — install scripts and one-off admin tasks only
- **Markdown** — life knowledge base (`~/agent-shared/life/`)

## Package Management
- `pip` + `venv` — one venv per project worktree
- No `uv`, no Poetry; keep it stdlib-standard

## Database
- **SQLite** via Python's `sqlite3` module (no ORM)
- All access through `shared.db.get_db()` — never open a DB file directly
- Per-environment DB path: `~/data/<env>/agent.db`

## LLM Runtimes
- **Claude Code** (`claude` CLI) — interactive sessions, agentic tasks
- **Gemini CLI** — parallel/secondary queries
- Anthropic SDK (`anthropic` Python package) — for any programmatic Claude API calls

## Infrastructure
| Tool | Purpose |
|---|---|
| `direnv` + `.envrc` | Per-worktree env vars (`AGENT_ENV`, `AGENT_SCRATCH_NAME`) |
| `systemd` user timers | Scheduled agent jobs (domain agents, synthesis, DB refresh) |
| `restic` | Off-machine encrypted backups |
| `Tailscale` | Secure remote access; services bind to Tailscale IP only |
| `Miniflux` | Self-hosted RSS reader, Tailscale-only |
| `tmux` | Session persistence for interactive Claude/Gemini sessions |

## Testing
- `pytest` — all agent modules; run with `python3 -m pytest tests/`
- No mocking of the database; tests use a real scratch DB

## Conventions
- Secrets referenced by VaultGuard entry name only; never hardcoded
- `.env` files allowed locally, never committed
