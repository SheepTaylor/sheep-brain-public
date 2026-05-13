# System Architecture

## Host
Dedicated home server / mini PC running Ubuntu 24.04. Always-on; remote access via Tailscale + SSH + tmux. No web dashboard — Telegram is the sole end-user interface.

## Layers

```
Interactive layer               Scheduled layer
  Claude Code (claude CLI)        systemd timers → domain agents
  Gemini CLI                      synthesis agent (weekly)
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

Both are backed up off-machine via restic.
