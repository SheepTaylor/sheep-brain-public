# Moose's own account: Linux user, Claude Code, shared and private knowledge

Status: approved by Sheep 2026-10-26; designed with Antigravity (one round); Phases 1–3 done 2026-10-26 (Phase 3 uses the local exchange repo `/srv/household/agent-system.git` instead of GitHub branch protection, which needs a paid plan); Phase 0 hardening done except Sheep's SSH key (`PasswordAuthentication no` once `~sheep/.ssh/authorized_keys` is populated; Moose does not use SSH; `fail2ban` dropped); Phase 4 open

## Goal

Moose gets her own Claude Code account on this box with a setup like Sheep's. Household,
vehicles, finance and health are shared; people, interests and work are private to each person.
She can build agents; Sheep approves every promotion to live.

## Decisions (Sheep, 2026-10-26)

- Health is shared: both people's records, the MyChart feeds and the `health_*` rows stay in
  Sheep's live system, unchanged. `health` stays in `LOCAL_ONLY_DOMAINS` for every agent.
- Finance is shared: Monarch, bills, budget and goals.
- Moose may edit the shared notes directly as a person. Agents (hers and Sheep's) still go
  through `shared.kb_changes` proposals.
- Private per person: `people/`, `interests/`, `work/`.

## Layout

| What | Where | Owner / mode |
|---|---|---|
| Shared life tree (goals, household, vehicles, finance, health, food, travel, digital) | `/srv/household/life` (own private git repo) | `sheep:household` 2770, `core.sharedRepository=group` |
| Sheep's private life tree | `~sheep/agent-shared/life/{people,interests,work}` | sheep 0700 |
| Moose's private life tree | `~moose/agent-shared/life/{people,interests,work}` | moose 0700 |
| Shared knowledge/skills (read-only for Moose) | `/srv/sheep_brain/agent-shared/{knowledge,skills}` | group read for `household` |
| Live DB (shared finance, food, health) | `~sheep/data/live/agent.db` | sheep only, never shared by file |
| Moose's test seed DB | `/srv/household/seed-agent.db`, nightly `VACUUM INTO`, strips calendar, messages, proposals | `sheep:household` 0640 |
| Moose's live code (read-only to her) | `/srv/household/agent-system-live` | `sheep:household` 2750 |
| Local model key | `/srv/household/config/llama-server.key` | `sheep:household` 0640 |

## Phases

0. **Hardening (sudo).** `~/scripts/security-remediation-2026-10-22.sh` ran 2026-10-22 (`ufw` active on `tailscale0`, `PermitRootLogin no`, Tailscale key expiry disabled). Remaining step: add Sheep's laptop public key to `~sheep/.ssh/authorized_keys`, verify key login, then append `PasswordAuthentication no` to `/etc/ssh/sshd_config.d/10-hardening.conf` (Moose does not log in over SSH; `fail2ban` dropped).
1. **User and Claude Code.** `adduser moose` (no sudo), home 0700, `loginctl enable-linger moose`,
   group `household` (sheep, moose), her SSH key, her device on the tailnet. Node system-wide
   (Sheep's nvm is inside a 0700 home) or her own nvm; `claude` signed in with her own Anthropic
   account. `~/.claude/CLAUDE.md` imports `persona/moose.md` plus the shared architecture,
   tech-stack and SDLC files. Antigravity optional.
2. **Split the life tree.** Create `/srv/household/life` and move the shared domains into it
   with history (`git filter-repo` or a fresh import, Sheep's choice). For each user add
   `git config --global --add safe.directory /srv/household/life`. Add `shared/paths.py`
   (`LIFE_SHARED`, `LIFE_PRIVATE`, `AGENT_DATA`, `PRIVATE_DOMAINS`), routing by domain, and
   replace the hardcoded `~/agent-shared/life` and `~/data` paths in `kb.py`, `kb_changes.py`,
   `bills.py`, `budget.py`, `goals.py`, `maintenance.py`, `db.py`. A single root `_index.md`
   in the shared tree lists every domain. Update the knowledge-base routing skill and `architecture.md`.
3. **Development and promotion.** Moose's GitHub account becomes a collaborator on the
   agent-system repo; branch protection on `staging` and `master` (only Sheep merges). She works in
   `~moose/projects/scratch-*` with her own `~moose/data/scratch-*` seeded from the seed DB, and
   opens pull requests to `staging`. Sheep tests in the candidate worktree, merges to `master`, then:
   shared agents run as sheep from the live worktree as today; her private agents run after Sheep
   runs `git pull --ff-only` in `/srv/household/agent-system-live`.
4. **Her live agents.** `~moose/.config/agent-system.env` (0600) with her own Anthropic API key and
   workspace (spend separate from Sheep's cap). She reads shared finance through the finance API
   (:8090, Tailscale) with a read-only token, never through the DB file. She gets her own Telegram
   bot for her private agents and her approval taps on `people/interests/work` proposals. Sheep's
   bot keeps her current restricted chat. Her user timers carry the `OnFailure=` line. She gets her own
   restic repository and password (Sheep's backup cannot read her home).

## Notes

- The shared Drive mount stays Sheep's only (no `allow_other`); Moose gets her own feed folder if needed.
- Within her own account, "Sheep approves promotion" is an agreement, not something the system
  enforces; what is enforced is that she cannot change Sheep's live code, DB, private tree or Drive mount.
- Interactive Claude Code sessions (either person) that read `health/` send it to Anthropic; the
  local-only rule binds the scheduled agents and the bot.
- Optional later: send her the per-person weekly health pulse (today she gets no reports by Sheep's rule).
