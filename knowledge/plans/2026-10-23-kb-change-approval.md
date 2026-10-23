# KB change approval: agents propose, Sheep approves, the bot applies

Status: agreed with Antigravity 2026-10-23 (round 2); awaiting Sheep's approval to build

## Problem

`life/` is human-curated, and today no agent writes to it. Claude Code is blocked from
editing it, rightly: a session that has read an email or a web page could be steered into
changing Sheep's knowledge base. But useful fixes now stall. Example: the bills.md weekly
transfer rule needed a one-cell change on 2026-10-23 and had to be handed to Sheep.

Sheep (2026-10-23): agents may modify KB entries **with Sheep's preapproval of the changes**.

## Design

Agents never write `life/` themselves. They file a proposed change; only a tap by Sheep on
Telegram applies it. The Telegram bot applies it as a git commit in the `life` repo.

### 1. Propose (Claude Code, Antigravity)

```
python3 -m shared.kb_changes propose --author claude-code --why "<one line>" <patch.diff>
```

- The patch is a unified diff against `life/` HEAD.
- The proposal is refused unless all of these hold:
  - every path is an existing `.md` file under `life/`, or a new one; no deletes or renames (deletion stays a manual act, per the approval rules);
  - the diff fits inline in one Telegram message (at most 3,000 characters); larger restructuring stays a manual commit by Sheep;
  - paths are normalized and relative, with no `..` and nothing under `.git/`;
  - `git apply --check` passes;
  - no secret-looking strings (API keys, `sk-`, tokens, passwords, full account numbers of 8+ digits).
- It is stored in a new table `kb_changes`: id, author, why, diff, base commit, per-file blob hashes, status, chat/message ids, commit sha, timestamps.
- Each touched file's blob hash (`git hash-object`) is stored. A new file is stored as null, meaning it must not exist when the change is applied.
- Telegram gets one message: "📚 KB change #N (author): why", the diff inline as an HTML-escaped `<pre>` block, and [Apply] [Reject] buttons. It is sent with `send_one`, like reflection proposals, so it is never held in the outbox.

### 2. Approve, only by Sheep's tap on Telegram

- Button data `k:<id>:apply|reject|revert`. It is accepted only from Sheep's chat id, the same check as `p:` proposals.
- On Apply, the bot first claims the proposal atomically: `UPDATE kb_changes SET status='applying' WHERE id=? AND status='pending'`. It proceeds only if exactly one row changed, so a double tap is harmless. The proposal also expires if it is more than 7 days old, checked on the tap.
- It then refuses (status `stale`, nothing applied, Sheep told why) unless all of these hold:
  - `life/` is on `master`;
  - the git index is empty;
  - none of the touched files has uncommitted edits (`git status --porcelain -- <files>` is empty), so Sheep's own Obsidian edits are never overwritten;
  - every touched file's blob hash still matches the proposal.
- Otherwise the bot runs `git apply`, stages only the touched files, and commits them in the `life` repo with the message `kb-change #N: <why>` plus trailers `Proposed-by: <author>` and `Approved-by: Sheep (Telegram, <timestamp>)`.
- The message is then edited to "✓ Applied (commit abc1234)" with a [Revert] button.
- Revert first checks that the touched files are clean, then runs `git revert --no-edit` of that commit. If the revert fails, it runs `git revert --abort`, leaving the files exactly as they were, and reports the conflict on Telegram.
- If any git step fails, the status becomes `failed` with the error, and the working tree is left as it was.

### 3. What agents cannot do, and the limits of that guarantee

- There is no approve command anywhere. The only path to applying a change is the Telegram callback from Sheep's chat id. If the bot is down, Sheep can still edit and commit `life/` by hand as today.
- `shared.kb_changes` exposes `propose` on its command line. The apply function is called only by the bot's callback handler.
- Claude Code settings get:
  - an allow rule for `python3 -m shared.kb_changes propose*`;
  - deny rules for direct edits under `life/`, for git commands that commit, apply or revert in the `life` repo, and for `python3 -c` imports of `kb_changes`.
- Antigravity gets the same instruction in its rules file. Its permission prompts cover file edits and shell commands in plan and accept-edits modes.
- Agents still never `git push` the `life` repo.
- **Limit, stated plainly:** this binds agents working inside their permission systems (Claude Code's rules and classifier, Antigravity's prompts). It cannot stop arbitrary code already running as `sheep`, which can write any file Sheep owns. That was equally true before this design. Closing it would mean running the bot as a separate Unix user that owns `life/`, which is out of scope for a single-user home server (see Alternatives).

### 4. Audit and recovery

Every applied change has three records:
- a `life` commit with the proposal id and approver;
- a `kb_changes` row with the original diff;
- the Telegram message.

`life/` is in the nightly restic backup, and Revert is one tap.

### Scope

- Phase 1: interactive Claude Code and Antigravity sessions only.
- Scheduled agents (reflection, review) do not propose KB changes. They keep using `agent_proposals` for playbook rules. Widening that is a later decision for Sheep.

### Docs to update on rollout

- `knowledge/architecture.md`: "no agent writes directly to life/" becomes "no agent writes directly to life/; Sheep-approved changes are applied by the Telegram bot from kb_changes".
- `knowledge/agent-system.md`: add the propose command.

## Alternatives rejected

- **A separate Unix user that owns `life/`, with the bot running as that user.** This is an OS-enforced gate, but it changes ownership for Obsidian, Drive sync and Sheep's own editing. Too heavy for the threat it closes.
- **A terminal `approve` command for Sheep (round 1 draft).** Removed: a TTY check doesn't stop a process from faking a terminal, and Sheep can already commit `life/` by hand.
- **Allow Claude Code to edit `life/` after Sheep says yes in chat.** A "yes" in a session can't be verified (an injected instruction can claim approval). There is no audit trail, and Antigravity would need its own copy of the rule.
- **Reuse `agent_proposals`.** Its status flow fits, but it stores rule text, not diffs, and applying means editing playbook.md, not committing to another repo. A sibling table keeps both simple. The bot handler and buttons follow the same pattern.

## Size

About 200 lines: `shared/kb_changes.py`, a bot handler, migration 017, and pytest coverage (a temp git repo stands in for `life/`).

## Revision log

**Round 1, 2026-10-23.** Antigravity (Gemini 3.8 Flash High; Argon was out of quota) said NOT AGREED and asked for five changes. All five were accepted:
1. Remove the terminal approve command.
2. Show the diff inline only, capped at 3,000 characters.
3. Require a clean index and clean touched files, and commit only the touched files.
4. Abort a conflicting revert.
5. Claim the proposal atomically before applying.

Agreed as drafted: blob-hash staleness checks (new files recorded as null), path validation, and a separate `kb_changes` table limited to interactive proposers. Added after the review: the guarantee's limit, stated explicitly, and the separate-Unix-user alternative.

**Round 2, 2026-10-23.** Antigravity confirmed that all five changes are reflected correctly and accepted the explicit limit and the rejected separate-user alternative. **VERDICT: AGREED.**
