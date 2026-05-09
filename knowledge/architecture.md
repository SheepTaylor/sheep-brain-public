# System Architecture

## Overview

Personal AI-augmented knowledge and agent system running on a dedicated Ubuntu machine. Three intertwined goals:

1. Capture and curate life knowledge in structured, machine-readable form that LLM agents can query efficiently.
2. Run a multi-agent runtime against that knowledge plus operational data, delivering intelligence via Telegram.
3. Provide a stable production environment that supports experimentation without risking irreplaceable state.

## Three Architectural Pillars

| Pillar | What it is | Why it matters |
|---|---|---|
| Canonical knowledge layout (`~/agent-shared/`) | Single source of truth shared by Claude Code, Gemini CLI, and scheduled agents | Eliminates duplication and drift; one place to update facts |
| Two-tier routing on the life KB | Hierarchical markdown tree with `_index.md` routers at each level | Keeps token cost flat as the KB grows; static indexes stay in prompt cache |
| Three-tier environment (`scratch / candidate / live`) | Git worktrees + env-aware DB paths + direnv, plus restic backups | Allows free experimentation while live agents run against real data |

## High-Level Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Ubuntu host machine                            │
│                                                                       │
│   ┌──────────────────────┐       ┌─────────────────────────────┐    │
│   │  Interactive layer    │       │   Scheduled layer            │    │
│   │  - Claude Code        │       │   - cron / systemd timers    │    │
│   │  - Gemini CLI         │       │   - Domain agents            │    │
│   │  - VS Code (Remote)   │       │   - Synthesis agent          │    │
│   └──────────┬───────────┘       └────────────┬────────────────┘    │
│              │                                  │                      │
│              └────────────┬─────────────────────┘                      │
│                           │ reads                                      │
│   ┌───────────────────────▼─────────────────────────────────────┐    │
│   │         ~/agent-shared/  (single source of truth)            │    │
│   │   persona/   knowledge/   skills/   life/                    │    │
│   └───────────────────────┬─────────────────────────────────────┘    │
│                           │                                           │
│   ┌───────────────────────▼─────────────────────────────────────┐    │
│   │  ~/data/{live,candidate,scratch-*}/agent.db   (SQLite WAL)   │    │
│   └──────────────────────────────────────────────────────────────┘   │
│                           │                                           │
│   ┌───────────────────────▼─────────────────────────────────────┐    │
│   │   Telegram bot (routing agent → domain context → Claude API) │    │
│   └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└──────────────────┬───────────────────────────────────────────────────┘
                   │ Tailscale + SSH
                   ▼
              Mobile / remote
```

## Data Flows

| Flow | Path |
|---|---|
| User asks Telegram a question | Telegram bot → routing agent → loads relevant `life/*.md` → Claude API → response |
| Scheduled domain agent runs | systemd timer fires → agent reads `agent-shared/` + queries live DB → writes observation row |
| Weekly synthesis | systemd timer fires → synthesis agent reads observation log + life KB → produces report → pushes to Telegram |
| Engineer experiments | `cd scratch-foo` → direnv flips env to scratch → all reads/writes hit isolated DB and worktree |

## Key Boundaries

**Knowledge vs. secrets.** `agent-shared/life/` contains facts and pointers, never secrets. Pointer files name VaultGuard entries; they never contain account numbers or passwords.

**System knowledge vs. life knowledge.** `agent-shared/knowledge/` is project and system documentation (architecture, tech stack, SDLC). `agent-shared/life/` is personal world facts. Different consumers, different load patterns, deliberately separated.

**Interactive vs. scheduled writes.** Interactive sessions (Claude Code, Gemini CLI) and scheduled agents both read `agent-shared/`, but only scheduled agents write to live observation tables. Interactive sessions write to scratch DBs.

**No agent writes to `life/`.** The life KB is human-curated through git. Agents read it; they never modify it.

## Irreplaceable Assets

Two assets underpin the entire risk model:

1. **SQLite operational database** (`~/data/live/agent.db`) — observations, financial history, schedules.
2. **Curated markdown knowledge base** (`~/agent-shared/life/`) — personal world model.

Everything else is reproducible from git and install scripts. Backups are built around protecting these two first.

## Remote Access

All remote access via Tailscale (zero-config private network) + SSH + tmux. No public-internet-exposed services on the host.
