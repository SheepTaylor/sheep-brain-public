# Consistency check: the KB against the data, and against itself

Status: proposed 2026-10-23, not started. Idea from the Every guide to OpenAI's Dot: its headline
example was an agent that noticed, unasked, a formula error in a budget sheet that was off by $5,700.

## Problem

Our checks look at what the agents *say*: `shared/verify.py` (every figure traces to evidence) and
`shared/critic.py` (a second model reads the draft). `riskwatch` turns dated obligations into
follow-ups, and `shared/feeds.py` (live 2026-10-23) checks that data keeps arriving. Nothing checks
the *source documents*: a `budget.md` line that no longer matches what Monarch shows being spent,
a `bills.md` amount the latest statement email contradicts, an insurance premium marked `[verify]`
that a renewal notice in the mail mirror could settle, or two KB files that state the same balance
differently. The agents then reason faithfully from a wrong premise, and verify and critic pass it,
because the figure *is* in the evidence.

## Design

A weekly job, `agents/consistency_agent.py` (Sunday 06:30, before the Monday review). It compares
the KB against the data with deterministic code. The model only writes up mismatches the code has
already found and proposes the fix. It never decides whether something is a mismatch.

### 1. Checks (deterministic, each a small function returning findings)

| Check | KB side | Data side | Finding when |
|---|---|---|---|
| Budget vs actuals | `budget.md` targets (`budget.load_targets`) | trailing 3 full months by category (`goals._flows`, `fin_transactions`) | a category runs > 25% and > $50/mo off its target for 3 months running |
| Bills vs statements | `bills.md` amounts and due days (`bills.load_config`) | `bill_emails` (statement/scheduled amounts and due dates) | the last 2 statements differ from the configured amount by > 10%, or the due day moved |
| Bills vs payments | `bills.md` pay-from account and rule | `bills.verify_month` results | the same bill missed its expected match 2 months running |
| Subscriptions vs charges | `subscriptions.md` price and cadence | recurring merchant charges in `fin_transactions` | price changed, a charge appears for one marked cancelled, or a listed one hasn't charged in 2 cycles |
| Insurance `[verify]` items | `insurance.md` rows tagged `[verify]` | `mail.search` on the carrier's sender for premium / renewal / declarations notices | a notice states a premium or term, compared with what the row says |
| Account roster | account names and last-4 digits mentioned in `finance/*.md` | `fin_accounts` (open/closed, owner) | the KB names an account Monarch shows closed, or an open account the KB never mentions |
| Cross-file facts | the same named figure in two finance files (e.g. HELOC balance in `credit.md` and `cash.md`) | each other | values differ by more than rounding (only figures dated within 30 days of each other) |

Out of scope: `health/` and `people/` (local-only lane; a later local-model variant could check
MyChart facts against `health/*.md`), and anything needing a web lookup.

### 2. Write-up and fix (one Sonnet call, only when there are findings)

- Input: the findings (KB quote, file, line, data figures with their source rows). Never whole files.
- Output per finding: one plain sentence, and where the fix is mechanical, a unified diff against `life/`.
- The diff goes through `python3 -m shared.kb_changes propose --author consistency_agent --why ...`,
  so nothing changes until Sheep taps Apply. Findings with no mechanical fix (e.g. "spending on
  dining has run $180/mo over the $250 target since July: raise the target or treat it as a goal")
  become a question with buttons via `shared/questions.py`.
- Claims check (`verify.check`) on the write-up against the findings; no critic, since the
  mismatches are already established by code.

### 3. Noise control

- `agent_state` key `consistency`: a hash per finding (check, file, subject). A finding is reported
  once, then again only if it is still there 30 days later or its figures move by > 10%.
  Rejecting a proposal or answering "it's fine" mutes that finding for 90 days (stored with the reason).
- At most 5 findings a week, highest dollar impact first. The rest wait.
- One message, budgeted (it can be held for the digest). Silent when there's nothing new.

### 4. Cost

Deterministic checks are free. One Sonnet call on weeks with findings: about $0.05–0.10, under
$0.50 a month. Registered as task `consistency` in `agent/tasks.toml`.

## Phases

1. **Shadow (2 weeks):** run the checks and log findings to `agent_runs.detail_json`, send nothing.
   Review the log with Sheep and tune the thresholds.
2. **Report:** the weekly message with findings and questions, no diffs.
3. **Propose:** add the `kb_changes` diffs for mechanical fixes (amounts, prices, dates, closed accounts).

## Open questions for Sheep

- Is Sunday morning right, or should findings ride in the Monday review instead of a separate message?
- The budget threshold: is 25% / $50 a month for 3 months the right sensitivity?
- Should a `[verify]` item that the mail settles be proposed straight away, given it's what the tag asks for?
