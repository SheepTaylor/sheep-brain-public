# System Architecture

## Host
Dedicated home server / mini PC running Ubuntu 26.04 (upgraded from 24.04 on 2026-10-22). Always-on; remote access via Tailscale + SSH + tmux (`ufw` restricts inbound to `tailscale0`). No web dashboard — Telegram is the sole end-user interface.

## Layers

```
Interactive layer               Scheduled layer
  Claude Code (claude CLI)        systemd timers → domain agents
  Antigravity (agy CLI / IDE)     synthesis agent (weekly)
  Telegram bot (agent-telegram-bot.service, life KB Q&A, /status)
         │                               │
         └──────────────┬────────────────┘
                        ▼
          ~/agent-shared/   (single source of truth)
          persona/  knowledge/  skills/  life/
                        │
                        ▼
          ~/data/<env>/agent.db   (SQLite operational DB)
```

## Key Directories
| Path | Purpose |
|---|---|
| `~/agent-shared/` | Shared KB: persona, architecture, tech-stack, SDLC, life domains |
| `~/projects/agent-system` | Live production worktree (`AGENT_ENV=live`) |
| `~/projects/agent-system-candidate` | Integration testing (`AGENT_ENV=candidate`) |
| `~/projects/scratch-*` | Feature branches (`AGENT_ENV=scratch`) |
| `~/data/<env>/agent.db` | Per-environment SQLite DB |

## Life Knowledge Base
Split in two since 2026-10-26 (`knowledge/plans/2026-10-26-moose-account-plan.md`):
- **Shared** `/srv/household/life/` (group `household`: Sheep and Moose): the root `_index.md` and goals, household,
  vehicles, finance, health, food, travel, digital. Both may edit it directly as people.
- **Private** `~/agent-shared/life/` per user: `people/`, `interests/`, `work/`. Sheep's and Moose's are separate.
- `shared/paths.py` resolves each domain to its tree from `AGENT_LIFE_SHARED` / `AGENT_LIFE_PRIVATE` in
  `agent-system.env`. Moose's Telegram questions never reach Sheep's private domains.

Both trees use a two-tier routing structure:
- `life/_index.md` → domain `_index.md` → 1–2 leaf files per query
- Token target: ≤ 3,500 tokens per query; the full `life/` tree is **never** bulk-loaded.
- Agents read `life/`; no agent writes directly to it (human-curated only). To change it, an interactive agent (Claude Code, Antigravity) files a proposal with `python3 -m shared.kb_changes propose --author <agent> --why "<reason>" <patch.diff>` (unified diff, paths relative to `life/`); Sheep approves each one with a Telegram tap and the bot commits it. Proposals apply to the shared tree only; private-domain edits are made by hand. See `knowledge/plans/2026-10-23-kb-change-approval.md`.

## Data Assets (Irreplaceable)
1. `~/data/live/agent.db` — accumulated observations and financial history
2. `/srv/household/life/` and `~/agent-shared/life/` — curated knowledge base markdown files

### Backups

Nightly restic backup to a local repo plus an offsite Google Drive mirror, with a
quarterly restore drill. Coverage, retention, credential location and the restore
procedure are documented in `life/digital/backups.md` (private repo), kept out of
this file because it is published publicly.

## Google Drive layout (reorganised 2026-09-12)

Mounted at `~/mnt/gdrive` by `rclone-gdrive.service` (remote `ObsidianVault:`). A `README.md` at
the Drive root repeats this table.

| Folder | Purpose | Written by |
|---|---|---|
| `feeds/` | The only place agents read. `feeds/monarch/` holds an optional Monarch CSV export (`Transactions_*.csv`, `Balances_*.csv`, newest pair wins), loaded only if the nightly pull from Monarch's MCP server fails (`shared/monarch_mcp.py`, since 2026-10-23); `feeds/mychart/<person>/` holds each person's Epic MyChart record export (`HealthSummary_*.zip`, newest wins, loaded by `health_sync_agent`; a zip in the wrong folder is refused, not merged), and each subfolder holds one more provider's MyChart account (e.g. `feeds/mychart/sheep/MUSC/`, Sheep's urologist), tracked separately; `feeds/Health Connect.zip` is the phone's daily Health Connect export (Galaxy Watch via Samsung Health, Withings scale; overwritten in place, loaded by `health_connect_agent`). The old Health Sync CSVs (Feb–Mar 2026) are in `archive/health-sync/`. | Sheep, the phone's Health Connect export |
| `life/<domain>/` | Source documents for the KB; subfolders mirror `~/agent-shared/life/` (finance, health/<person>, household, vehicles, people, interests/…). Human-curated. | Sheep |
| `system/` | Plans, frameworks (Codex Vitae, values, Life System Architecture), skills for the agent system itself. `system/apps/` holds the finance Android app APK (no secrets; phones pair separately). | Sheep, agents (apps/) |
| `archive/` | Inert: old Health Sync CSVs (`archive/health-sync/`), old Checkbook workflow notes (`archive/checkbook/`; the databases were trashed 2026-09-12), 2025 transaction downloads, Amazon orders, untitled leftovers. | nobody |
| `backups/` | restic mirror (see `life/digital/backups.md`). | restic |

Move files with `rclone moveto` from the server rather than the Drive web UI: moves are server-side
either way, but the mount's `--dir-cache-time 1h` makes web-UI changes invisible locally for up to an hour.
