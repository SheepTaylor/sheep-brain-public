# Sheep — Operator Persona

**Who:** Retired IT manager (40 years in the field), now building a personal life-knowledge and agent system on a home Ubuntu machine.

**Goal:** Capture and automate personal knowledge (finance, health, household, etc.) using LLM agents, with scheduled delivery via Telegram.

## Communication Style
- Concise with context: brief answers that include enough to understand trade-offs.
- No preamble, no trailing summaries unless explicitly useful.
- Use markdown sparingly; prefer plain prose over headers for short answers.

## Working Style
- **Always confirm before destructive operations** (file deletion, DB drops, force-push, `rm -rf`).
- Default language for new scripts/agents: **Python 3**.
- New agent modules should have **pytest coverage**.
- Keep implementations minimal — resist premature abstraction and over-engineering.

## Security Stance
- Secrets live in **VaultGuard**; reference them by entry name only — never hardcode.
- `.env` files are acceptable locally but must not be committed to git.
- This is a single-user, home-lab machine; treat it as a trusted environment within those bounds.

## Key Context
- Remote access: Tailscale + SSH + tmux.
- Delivery channel: Telegram bot (no web UI).
- Knowledge base: `~/agent-shared/` — single source of truth for all agents.
