# SDLC: Three-Tier Environment Model

## Tiers

| Tier | Branch | Worktree | DB | Purpose |
|---|---|---|---|---|
| **scratch** | feature branch | `~/projects/scratch-<feature>/` | `~/data/scratch-<feature>/agent.db` | Break freely; never promoted directly to live |
| **candidate** | `staging` | `~/projects/candidate/` | `~/data/candidate/agent.db` (refreshed nightly from live) | Last-stop integration testing |
| **live** | `main` | `~/projects/agent-system/` | `~/data/live/agent.db` | Scheduled jobs run here; changes arrive only via merged commits |

Default `AGENT_ENV=scratch` in `shared/db.py` means an unconfigured shell is safe by construction.

## Worktree Setup

```bash
cd ~/projects/agent-system
git worktree add ../candidate -b staging
git worktree add ../scratch-<feature> -b feat/<feature>
```

Multiple working copies of the same repo coexist on different branches. Claude Code in one, Gemini CLI in another, cron in live — none collide.

## Environment Binding: direnv

Each worktree has a `.envrc` that auto-loads on `cd`:

```bash
# ~/projects/agent-system/.envrc
export AGENT_ENV=live

# ~/projects/scratch-tax-categorizer/.envrc
export AGENT_ENV=scratch
export AGENT_SCRATCH_NAME=tax-categorizer
```

Wrong directory = wrong environment. A scratch agent cannot point at the live DB because no env var in the scratch tree resolves to the live path.

## DB Isolation: Snapshot, Don't Share

```bash
# Spin up a scratch DB for a new feature
mkdir -p ~/data/scratch-<feature>
sqlite3 ~/data/live/agent.db ".backup '/home/sheep/data/scratch-<feature>/agent.db'"

# Refresh candidate from live (nightly cron)
sqlite3 ~/data/live/agent.db ".backup '/home/sheep/data/candidate/agent.db'"
```

`.backup` is online-safe with WAL — live agents do not need to be stopped during the copy.

## Promotion Flow

```
scratch-<feature>  →  merge to staging (test in candidate)  →  fast-forward to main (live)
```

Promote to live:

```bash
cd ~/projects/agent-system   # the live worktree
git pull
systemctl --user restart agent-synthesis.timer
```

No PR review process (solo operator). The discipline is the worktree boundary: changes only land in `live` via an explicit `git pull` against the live worktree.

## Markdown Files in Worktrees

`agent-shared/` is a separate repo mounted at a fixed path — all worktrees see the same live knowledge. Worktree isolation applies to application code only. Knowledge edits go through their own git workflow (public repo for `knowledge/`, `skills/`, `persona/`; private repo for `life/`).

## Backups

See `architecture.md` for the risk model. Key points:

- Restic runs nightly against `~/data`, `~/projects`, `~/agent-shared`.
- Weekly `restic copy` mirrors to off-site (Backblaze B2 or S3).
- Quarterly verified restore drill is required — a backup not tested is not a backup.
