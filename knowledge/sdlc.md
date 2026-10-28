# SDLC

This document defines the development lifecycle for the agent system, focusing on the three-tier environment and the promotion flow.

## 1. Environments

| Tier | Branch | Worktree Path | DB Path | Purpose |
|---|---|---|---|---|
| **scratch** | `scratch/*` | `~/projects/scratch-*` | `~/data/scratch-*/agent.db` | Isolated feature development. |
| **candidate** | `staging` | `~/projects/agent-system-candidate` | `~/data/candidate/agent.db` | Integration testing against live DB snapshot. |
| **live** | `master` | `~/projects/agent-system` | `~/data/live/agent.db` | Production environment for scheduled agents. |

## 2. Promotion Flow

The promotion flow ensures that code is tested in isolated and integration environments before reaching production.

### Step 1: Feature Development (Scratch)
- Create a new branch and worktree from `master`:
  ```bash
  git -C ~/projects/agent-system worktree add ~/projects/scratch-my-feature -b scratch/my-feature master
  ```
- Run `direnv allow ~/projects/scratch-my-feature` (`.envrc` automatically sets `AGENT_ENV=scratch` and `AGENT_SCRATCH_NAME=my-feature` from the directory name).
- Develop and test locally (`python3 -m pytest tests/`).

### Step 2: Integration Testing (Candidate)
- Merge scratch branch into `staging`:
  ```bash
  cd ~/projects/agent-system-candidate
  git merge scratch/my-feature
  ```
- Verify behavior in the candidate worktree. The candidate DB is refreshed nightly from live.
- If fixes are needed, apply them in the scratch worktree and re-merge.

### Step 3: Production Release (Live) & Cleanup
- Merge `staging` into `master`:
  ```bash
  cd ~/projects/agent-system
  git merge staging --ff-only
  ```
- Verify behavior in the live worktree.
- Reload or restart any relevant services (`systemctl --user daemon-reload`, e.g., `systemctl --user restart agent-telegram-bot.service`).
- Remove the merged scratch worktree, branch, and scratch DB:
  ```bash
  git -C ~/projects/agent-system worktree remove ~/projects/scratch-my-feature
  git -C ~/projects/agent-system branch -d scratch/my-feature
  rm -rf ~/data/scratch-my-feature
  ```

### Note on .envrc
`.envrc` is identical across all branches and derives `AGENT_ENV` and `AGENT_SCRATCH_NAME` from `basename "$PWD"` (`agent-system` = `live`, `agent-system-candidate` = `candidate`, `scratch-*` = `scratch`). Merges between branches never require `.envrc` conflict resolution.

## 3. Database Management
- All DB access MUST go through `shared.db.get_db()`.
- The candidate DB is refreshed nightly via `candidate-db-refresh.timer`.
- Manual refresh: `systemctl --user start candidate-db-refresh.service`.


## 4. Moose's Workflow (since 2026-10-26)

Moose develops as Linux user `moose`; only Sheep promotes. Plan: `knowledge/plans/2026-10-26-moose-account-plan.md`.

| Step | Moose | Sheep |
|---|---|---|
| Get the code | `git clone /srv/household/agent-system.git ~/projects/moose-base` (once; needs `git config --global --add safe.directory /srv/household/agent-system.git`) | keeps `master`/`staging` there current: `git push household master staging` after each promotion |
| Develop | `git -C ~/projects/moose-base worktree add ~/projects/scratch-<feature> -b moose/<feature> origin/master`; `direnv allow`; seed the DB: `mkdir -p ~/data/scratch-<feature> && cp /srv/household/seed-agent.db ~/data/scratch-<feature>/agent.db`; pytest | |
| Hand over | `git push origin moose/<feature>` (the exchange only accepts `moose/*` branches) | |
| Review + integrate | | `git fetch household`, review, then `git merge household/moose/<feature>` in the candidate worktree (staging); test |
| Release | | `git merge staging --ff-only` in the live worktree, push GitHub and `household`, then `git -C /srv/household/agent-system-live pull --ff-only` (the read-only copy Moose's own agents run from; a default ACL keeps it group read-only) |

- Never name a worktree `agent-system` or `agent-system-candidate`: `.envrc` derives the tier from the directory name.
- `/srv/household/seed-agent.db` is rebuilt nightly from live (finance, health, food kept; Sheep's mail, calendar and agent conversations emptied).
- The exchange repo is not live. Nothing Moose pushes runs until Sheep merges it.
