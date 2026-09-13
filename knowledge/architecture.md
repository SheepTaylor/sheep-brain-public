# System Architecture

## Host
Dedicated home server / mini PC running Ubuntu 24.04. Always-on; remote access via Tailscale + SSH + tmux. No web dashboard — Telegram is the sole end-user interface.

## Layers

```
Interactive layer               Scheduled layer
  Claude Code (claude CLI)        systemd timers → domain agents
  Gemini CLI                      synthesis agent (weekly)
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
`~/agent-shared/life/` uses a two-tier routing structure:
- `life/_index.md` → domain `_index.md` → 1–2 leaf files per query
- Token target: ≤ 3,500 tokens per query; the full `life/` tree is **never** bulk-loaded.
- Agents read `life/`; no agent writes directly to it (human-curated only).

## Data Assets (Irreplaceable)
1. `~/data/live/agent.db` — accumulated observations and financial history
2. `~/agent-shared/life/` — curated knowledge base markdown files

### Backup status (as of 2026-09-08)

| Layer | State |
|---|---|
| Local restic repo | `/srv/sheep_brain/backups/restic`, nightly at 02:00 via crontab (`backup-script.sh`) |
| Coverage | `~/data` (all tiers), `/srv/sheep_brain/{data,projects,agent-shared}`, `db-snapshots/` |
| SQLite consistency | `~/scripts/snapshot-dbs.py` runs `VACUUM INTO` before each backup; restore from `db-snapshots/<tier>/agent.db`, not the raw WAL-mode file |
| Retention | `forget --keep-daily 14 --keep-weekly 8 --keep-monthly 12 --prune` in both scripts |
| **Offsite mirror** | `rclone:ObsidianVault:backups/restic-mirror` (Google Drive), nightly at 03:00 via `mirror-script.sh` (`restic copy` + forget + `check --read-data-subset=10%`) |
| Repo password | `~/.config/restic/password`, **same for both repos**. Must also be stored off-machine (VaultGuard entry `restic_repo_password`) or the mirror is unreadable after a disk loss |
| Restore drill | First drill 2026-09-08 from the mirror (see `knowledge/plans/2026-09-08-local-ai-system-plan.md` Phase 0). `restore-drill.timer` repeats it quarterly and posts PASS/FAIL to Telegram (since 2026-09-13). |

Remaining gap: Google Drive is a single provider under one Google account. Acceptable for
~20 MB of data; revisit (B2 or a Tailscale peer) if the account is ever at risk.

**Restore from the mirror** (works with only the password and a Google login):
```bash
rclone config   # re-create remote "ObsidianVault" (Google Drive) on the new machine
restic -r rclone:ObsidianVault:backups/restic-mirror -p <password-file> restore latest --target /restore
```

## Google Drive layout (reorganised 2026-09-12)

Mounted at `~/mnt/gdrive` by `rclone-gdrive.service` (remote `ObsidianVault:`). A `README.md` at
the Drive root repeats this table.

| Folder | Purpose | Written by |
|---|---|---|
| `feeds/` | The only place agents read. `feeds/monarch/` holds the weekly Monarch export (`Transactions_*.csv`, `Balances_*.csv`, newest pair wins); `feeds/mychart/` holds the Epic MyChart record export (`HealthSummary_*.zip`, newest wins, loaded by `health_sync_agent`); Health Sync should target `feeds/` so its `Health Sync <type>` folders land there. | Sheep, Health Sync app |
| `life/<domain>/` | Source documents for the KB; subfolders mirror `~/agent-shared/life/` (finance, health/sheep, health/janice, household, vehicles, people, interests/…). Human-curated. | Sheep |
| `system/` | Plans, frameworks (Codex Vitae, values, Life System Architecture), skills for the agent system itself. | Sheep |
| `archive/` | Inert: old Checkbook workflow notes (`archive/checkbook/`; the databases were trashed 2026-09-12), 2025 transaction downloads, Amazon orders, untitled leftovers. | nobody |
| `backups/` | restic mirror (see Backup status above). | restic |

Move files with `rclone moveto` from the server rather than the Drive web UI: moves are server-side
either way, but the mount's `--dir-cache-time 1h` makes web-UI changes invisible locally for up to an hour.
