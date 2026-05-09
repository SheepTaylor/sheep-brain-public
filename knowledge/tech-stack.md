# Tech Stack

## Host

- **OS:** Ubuntu 24.04 LTS (dual-boot alongside Windows 11)
- **Hardware:** Existing laptop; 50–80 GB partition for Ubuntu, 1 TB NTFS data drive mounted via `ntfs3`

## Installed Tools

| Tool | Purpose | Install path |
|---|---|---|
| Node.js LTS (via nvm) | Claude Code and Gemini CLI runtime | `nvm install --lts` |
| Claude Code | Primary agentic dev tool | `npm install -g @anthropic-ai/claude-code` |
| Gemini CLI | Secondary agentic dev tool | `npm install -g @google/generative-ai-cli` |
| Python 3.12 + pip | Agent runtime, FastAPI, SQLite tooling | apt + pip |
| direnv | Per-worktree environment variable loading | apt |
| restic | Encrypted, deduplicating backup | apt |
| tmux | Persistent terminal sessions | apt |
| openssh-server | Remote shell access | apt |
| Tailscale | Zero-config private network for remote access | tailscale install script |
| VS Code (Windows host) + Remote-SSH | Editor | Microsoft installer |

## Sharing Mechanics: Claude Code + Gemini CLI

Both tools load shared content via `@path/to/file.md` import syntax.

`~/.claude/CLAUDE.md`:
```markdown
@~/agent-shared/persona/sheep.md
@~/agent-shared/knowledge/architecture.md
@~/agent-shared/knowledge/tech-stack.md
@~/agent-shared/knowledge/sdlc.md

# Life knowledge available at ~/agent-shared/life/
# Load on demand via the two-tier router; do not import wholesale.
```

`~/.gemini/GEMINI.md` carries an identical import set. `~/.gemini/settings.json` adds:
```json
{ "includeDirectories": ["~/agent-shared"] }
```

Per-project `CLAUDE.md` / `GEMINI.md` files import the relevant project knowledge plus needed skills. The `agent-system` project additionally references the life tree because its routing agent reads from it.

## MCP Servers

`~/agent-shared/mcp-servers.json` is the canonical source of truth for MCP server configuration. A sync script writes from it into both `~/.claude.json` (Claude Code) and `~/.gemini/settings.json` (Gemini CLI). Never edit the tool-specific files directly.

## What Does Not Auto-Share

- Claude Code slash commands (`~/.claude/commands/`) — no Gemini equivalent.
- Claude Code `settings.json` permissions — no Gemini equivalent.

These remain platform-specific and are excluded from the shared layer by design.

## Agent Runtime: Python Stack

- **FastAPI** — HTTP layer for any internal agent APIs.
- **SQLite (WAL mode)** — single `agent.db` per environment tier; all access via `shared/db.py`.
- **`TrackedAnthropic` wrapper** — monitors token usage, feeds a Streamlit dashboard backed by SQLite.

See `agent-system.md` for runtime architecture and database conventions.

## Filesystem Layout

```
~/
├── .claude/           CLAUDE.md, settings.json, commands/
├── .claude.json       user-scope MCP server configs
├── .gemini/           GEMINI.md, settings.json
├── agent-shared/      single source of truth (this repo + life/ private repo)
│   ├── persona/
│   ├── knowledge/
│   ├── skills/
│   └── life/
├── projects/
│   ├── agent-system/  live worktree (main)
│   ├── candidate/     staging worktree
│   └── scratch-*/     feature worktrees
├── data/
│   ├── live/agent.db
│   ├── candidate/agent.db
│   └── scratch-*/agent.db
└── backups/restic/
```
