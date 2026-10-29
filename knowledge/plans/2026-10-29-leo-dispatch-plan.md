# Leo Phase 3: dispatching Claude Code jobs

Status: AGREED with Antigravity 2026-10-29 (2 rounds, conversation bdb6bdd2-dc98-40a3-8f3c-d798a8b1ee0c;
transcripts in `2026-10-29-leo-dispatch-review/`). Awaiting Sheep's approval. The build waits for Leo
Phases 1 and 2 (Sheep keeps phase order), because Phase 3 uses Phase 2's dev board; Step 0 below
ships before then. This is the "own design round" required by
`2026-10-28-leo-chief-of-staff-plan.md`. No personal details here: `knowledge/` is published.

## What Sheep asked for (2026-10-28)

"Dev = track + dispatch on Approve, never promote." There's no daily cap on dispatches. Merging to
staging and promotion stay with Sheep.

## Flow

```
Sheep → Leo: "have Claude do the food Phase 5 tests"        (or /leo dispatch <words>)
Leo   → card (built in code) + [Approve] [Cancel]           dev_jobs row 'proposed'
Approve → 'queued' → agents/leo_dispatch.py start <id>
        → systemd-run --user --unit=leo-dispatch-<id> (transient; the bot never blocks)
            new worktree + empty schema DB → claude -p (sandboxed) → pytest (sandboxed) → report
Telegram: branch, commits, diffstat, test result, Claude's summary + [Diff] [Discard]
```

## Step 0, ships now and on its own: test and env safety

These gaps exist today, whatever happens to dispatch:
- **`shared/env.py`** honours `AGENT_ENV_FILE`. An empty value or `/dev/null` means load nothing. The
  default is unchanged (`~/.config/agent-system.env`).
- **`tests/conftest.py`** gets an autouse guard that refuses unmocked `ask_claude`, `ask_web` and
  `ask_vision` network calls in every test, next to the existing Telegram, printer and Monarch
  guards. Today a careless test could bill the API key.

## 1. The brief is assembled in code, never written by a model

Untrusted text (emails, web results, recipes, KB proposals) must never reach a job with edit and
shell rights.
- **Leo's dispatch tool takes only `(plan, slug)`:**
  - `plan` must be an existing file in `knowledge/plans/` whose Status line says approved or agreed;
  - `slug` must match `^[a-z0-9][a-z0-9-]{2,39}$`.
  - **Sheep's words come from a contextvar** set by the bot from the incoming message text. They're
    never a tool argument, so nothing Leo read or wrote can enter a brief.
- **The brief is** Sheep's words verbatim, the plan path, the slug, and fixed standing instructions:
  work only in this worktree, follow `CLAUDE.md` and `sdlc.md`, run pytest, commit on the branch,
  don't push or promote, and stop and write a question if the plan doesn't cover the request.
- **No approved plan, no job.** If no plan covers the request, Leo says it needs a plan first (with an
  Antigravity round, per Sheep's rule) and offers no [Approve].
- **Editing:** a reply to a `proposed` card corrects Sheep's words. It's intercepted in `_message()`
  before `feedback.record_reply` and `privacy_route`, and the card is rebuilt for a fresh [Approve].

## 2. The job's environment is built from scratch

- **`leo_dispatch.py` builds the child environment as a clean dict** for both `claude` and the
  post-job `pytest`. It holds only:
  - `HOME`, `PATH` (system + the nvm node bin) and `LANG`;
  - `AGENT_ENV=scratch`, `AGENT_SCRATCH_NAME=<slug>`, `PYTHONPATH=<worktree>`;
  - `AGENT_ENV_FILE=/dev/null`;
  - `GIT_AUTHOR_*` and `GIT_COMMITTER_*`, so git needs nothing from `~/.config`.
- **Second layer:** `systemd-run` also gets `-p UnsetEnvironment=` for the user manager's variables
  (`SSH_AUTH_SOCK`, `DBUS_SESSION_BUS_ADDRESS`, `DISPLAY`, `WAYLAND_DISPLAY`, `XAUTHORITY`,
  `GPG_AGENT_INFO`).
- **Why:**
  - `AGENT_ENV=live` and `PYTHONPATH` would otherwise come from the bot's unit, pointing at the live
    DB and code.
  - `ANTHROPIC_API_KEY` would bill the API outside the cap's accounting, and `env.load()` would read it
    straight from disk.
  - With none of these, Claude Code uses Sheep's subscription login, and nothing can reach Telegram.

## 3. Claude Code: settings from outside the worktree, sandbox locked

Settings live in `~/data/scratch-<slug>/claude-settings.json`. That's outside the worktree, because
Claude Code ignores the key sandbox locks when they're set in a project's own settings. `.claude/` is
also added to the repo `.gitignore`. The command is:

```
claude -p <brief> --settings <that file> --setting-sources '' --strict-mcp-config
       --disable-slash-commands --no-session-persistence --permission-mode acceptEdits
       --permission-prompts none --model sonnet --max-turns 50
```

`--setting-sources ''` keeps Sheep's own user settings and allow rules out of the job.
`--max-turns` is hidden from `--help` in 2.1.312 but enforced under `-p`.

- **`sandbox`:**
  - `enabled: true`, `failIfUnavailable: true` and `allowUnsandboxedCommands: false` (no
    `dangerouslyDisableSandbox` retries). `autoAllowBashIfSandboxed` stays on: the kernel sandbox is
    the boundary.
  - **`filesystem.allowWrite`:** the worktree, `~/data/scratch-<slug>` and `/tmp`.
  - **`filesystem.denyWrite`:** in `agent-system/.git`, the files `HEAD`, `index`, `packed-refs`,
    `refs/heads/master`, `refs/heads/staging`, `refs/remotes`, `worktrees/agent-system-candidate`,
    `hooks` and `config`. A worktree's commits must write to the live repo's `.git`; this keeps the
    branches that matter read-only.
  - **`filesystem.denyRead`:**
    - under `~/.config`: `agent-system.env*`, `google-*`, `monarch-*`, `rclone` and `restic`;
    - `~/data/live`, `~/data/candidate` and `~/data/mail`;
    - `/srv/household/life`, `/srv/household/seed-agent.db` and `~/agent-shared/life`;
    - `~/mnt/gdrive`, `~/.ssh`, `~/.gnupg` and `~/.claude/projects`.
  - **Network:** Claude Code's own API connection only; no other domains allowed.
- **`permissions`:**
  - **allow:** `Edit` and `Write`.
  - **deny:** the same paths as `denyRead`, written with `//` absolute paths (a single `/` means
    relative to the project). Bash is denied for `git push`, `git merge`, `git checkout master` and
    `git checkout staging`, `systemctl`, `sudo` and `ssh`. `WebFetch` and `WebSearch` are denied too.
- **Timeout:** the wrapper runs `claude` with a 7,200 s subprocess timeout, so it can still record
  `timed_out`, report and start the next job. `RuntimeMaxSec=7500` on the unit is only a backstop.

## 4. Before and after the job

- **Worktree:** the dispatcher (plain Python, not Claude) creates it from `master` and runs
  `direnv allow`. It refuses if the worktree path or the branch already exists.
- **Database:** `~/data/scratch-<slug>/agent.db` is a fresh, schema-only `init_db()`. No seed copy:
  the seed keeps everyone's health records (Sheep shares health with Moose) and untrusted rows.
- **Taint check:** before and after, the dispatcher records `git status --porcelain` and `HEAD` for
  `agent-system`, `agent-system-candidate`, `~/agent-shared` and `/srv/household/life`. Any change
  marks the job **tainted**: a critical alert, and nothing is reported as success.
- **Post-job tests run sandboxed too.** That run imports code the job could have edited
  (`conftest.py`, `env.py`). So the wrapper runs `python3 -m pytest tests/ -q` under `bwrap`, with:
  - `/` read-only;
  - writes only to the worktree, the scratch data dir and a private `/tmp`;
  - tmpfs over every `denyRead` path;
  - `--unshare-net` and `--die-with-parent`.
- **What's reported:** the commits on the branch, the diffstat, the pass/fail counts from that
  pytest run (not Claude's claims), and the last 1,500 characters of Claude's final message.

## 5. Report and buttons

- **Done card:** slug, commits, files changed, tests and Claude's summary. If Claude stopped with a
  question, the card leads with it.
- **[Diff]** sends the diffstat and the first 3,000 characters of the diff.
- **[Discard]** asks to confirm (deletion needs Sheep's OK), then removes that job's worktree, branch
  and scratch DB only.
- Merging to staging happens in Sheep's own Claude Code session. Leo never merges, pushes or
  promotes.
- The dev board (Phase 2) shows proposed, queued, running and finished jobs.

## 6. Jobs table and limits

- `dev_jobs(id, created_at, words, plan, slug, status, unit, started_at, finished_at, result_json,
  chat_id, message_id)`.
  - **Status** is one of proposed, cancelled, queued, running, done, failed, timed_out, tainted,
    discarded.
  - It's `CLOUD_READABLE` (Sheep's own words and repo facts) and in `seed.EMPTIED`.
- **One job at a time.** Later approvals queue up. Before any start, `running` rows whose unit is no
  longer active are reaped as `failed` and reported, so a crash can't jam the queue. There's no
  daily cap (Sheep).
- Sheep's chat only.

## Tests (pytest, real scratch DB; a fake `claude` script, never the real CLI)

- **Step 0:** `AGENT_ENV_FILE=/dev/null` loads nothing, and an unmocked `ask_claude` in a test fails loudly.
- **Brief:**
  - built from the contextvar words, the plan and the slug only;
  - injected text in a Leo turn can't change it;
  - a bad slug or an unapproved plan is refused.
- **Environment:** exactly the listed variables. No `ANTHROPIC_API_KEY`, `TELEGRAM_*` or
  `SSH_AUTH_SOCK`.
- **Settings and command:**
  - the settings file is outside the worktree and has the sandbox locks and the `//` deny paths;
  - the command line has every flag above.
- **Sandboxed pytest:** the `bwrap` argv covers every `denyRead` path and `--unshare-net`. A
  tmp-dir integration test confirms a test inside it can't read a planted secret file.
- **Taint check:** a fake change in a watched repo marks the job tainted, with a critical alert.
- **Queue:** a second approval waits; a finished or reaped job starts the next; a timeout is recorded
  as `timed_out`.
- **Collisions and Discard:** an existing branch or worktree is refused; Discard needs a confirm and
  removes only that job's worktree, branch and DB.
- **Callbacks:** card edits are intercepted before feedback and the privacy route. Moose is refused.

## Cost

Claude Code runs on Sheep's subscription, so there's no API spend, and the brief costs no model call.
Each job is bounded by `--model sonnet`, `--max-turns 50` and 2 h.

## Open for Sheep

1. Jobs use the subscription's usage limits. Run them any time, or only overnight?
2. When a job finishes, push its branch to GitHub (never merged), or keep it local until he reviews it?
