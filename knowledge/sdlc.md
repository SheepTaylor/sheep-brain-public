# SDLC

This document defines the development lifecycle for the agent system, focusing on the three-tier environment and the promotion flow.

## 1. Environments

| Tier | Branch | Worktree Path | DB Path | Purpose |
|---|---|---|---|---|
| **scratch** | `scratch-*` | `~/projects/scratch-*` | `~/data/scratch-*/agent.db` | Isolated feature development. |
| **candidate** | `staging` | `~/projects/agent-system-candidate` | `~/data/candidate/agent.db` | Integration testing against live DB snapshot. |
| **live** | `master` | `~/projects/agent-system` | `~/data/live/agent.db` | Production environment for scheduled agents. |

## 2. Promotion Flow

The promotion flow ensures that code is tested in isolated and integration environments before reaching production.

### Step 1: Feature Development (Scratch)
- Create a new branch: `git checkout -b scratch/my-feature`
- Create a new worktree: `git worktree add ~/projects/scratch-my-feature -b scratch/my-feature`
- Configure `.envrc`:
  ```bash
  export AGENT_ENV=scratch
  export AGENT_SCRATCH_NAME=my-feature
  ```
- Run `direnv allow`.
- Develop and test locally.

### Step 2: Integration Testing (Candidate)
- Merge scratch branch into `staging`:
  ```bash
  cd ~/projects/agent-system-candidate
  git merge scratch/my-feature
  ```
- Verify behavior in the candidate worktree. The candidate DB is refreshed nightly from live.
- If fixes are needed, apply them in the scratch worktree and re-merge.

### Step 3: Production Release (Live)
- Merge `staging` into `master`:
  ```bash
  cd ~/projects/agent-system
  git merge staging --ff-only
  ```
- Verify behavior in the live worktree.
- Restart any relevant services (e.g., `systemctl --user restart agent-synthesis.timer`).

### Note on .envrc
.envrc is committed to git but is environment-specific. When merging between branches (e.g., scratch → staging), you must resolve conflicts in .envrc by keeping the target branch's AGENT_ENV value.

## 3. Database Management
- All DB access MUST go through `shared.db.get_db()`.
- The candidate DB is refreshed nightly via `candidate-db-refresh.timer`.
- Manual refresh: `systemctl --user start candidate-db-refresh.service`.

