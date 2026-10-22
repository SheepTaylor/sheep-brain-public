# Finance trackers from the 2026-10-22 blind-spot review

Status: CONFIRMED by Antigravity 2026-10-22 (Gemini 3.8 Flash; Argon out of quota), with the revisions below.

The review (Claude and Antigravity, 2026-10-22) found that the finance agents watch month-to-month cash
well, but not the balance sheet, taxes, the survivor case or dated obligations. Sheep answered the
open questions the same day; the facts are in life/finance and life/household (private repo). This plan
adds the trackers. Figures live in the private KB and the DB, never in this file.

## Principles
- Deterministic computers in `shared/goals.py`, one per new goal row in `life/goals/goals.md`, so the
  weekly review, `goal_status` and the heartbeat's status-change gate pick them up with no new agent,
  timer or message type. The heartbeat reports only status changes, so a goal that stays off track
  doesn't nag.
- Dated obligations go in one KB table read by the existing risk watch. No new scheduler.
- Thresholds and assumptions sit in the goals.md target cell or the KB tables, not in code.

## New goals (Financial security section of goals.md)

| Goal | Computer | Status rule |
|---|---|---|
| Cash reserve | cash in checking, savings and emergency fund accounts minus credit card balances (the HELOC excluded), from fin_accounts; note the change since 90 days ago from fin_balances | off if below the target (≥ $X in goals.md) |
| Income covers spending | the last 3 full months: income minus expenses (Monarch category groups; transfers excluded, tax refunds excluded from income) | off if the monthly average is below the target (≥ $0) |
| Survivor income | pension + the larger Social Security benefit (the smaller one stops at the first death), from the bills.md Income table, against 85% of the last 3 full months' spending; note what a fixed pension buys in 10 years at 3% inflation | off if there's a monthly gap; value shows the gap |
| Withholding right-sized | the latest tax refund (or IRS payment) in the last 15 months from fin_transactions; refund ÷ 12 = over-withheld per month | off if the refund is above the target (≤ $X), or if more than $1,000 was owed (under-withheld) |
| Sinking funds funded | transfers into Online Checking (the sinking-fund account in budget.md) in the last 35 days, and its balance | off if less than the target (≥ $X, the monthly sinking-fund total in budget.md) came in over 35 days |
| Big replacements funded | life/household/real_estate.md "Major systems" table (installed year, typical life, cost): items reaching end of life within 3 years, their cost against Emergency Fund + Online Checking | off if due cost exceeds the reserve; watch if something is past its life but funded |

## Dated obligations
- New `life/finance/deadlines.md` (Date | Obligation | What to check): Medicare open enrollment
  (Dec 7), county property tax (Jan 15), the 70½ QCD start, tax return and withholding (Mar 1), Form
  5498 rollover coding, a yearly survivor/beneficiary/credit-freeze check, first RMD.
- `shared/riskwatch.py` adds it to FILES. The "Risk watch" playbook section adds "money deadlines" and
  says "yearly" rows recur: use the next occurrence.

## Deliberately not tracked
- Portfolio allocation or daily prices: Monarch has no holdings; a one-time look at Fidelity instead.
- Roth conversions, paying the HELOC from the IRA, long-term care: one-time decisions for Sheep (and a
  planner), not trackers.
- Card-reward optimisation, MPG, household belongings.

## Tests
pytest per computer against the scratch DB (`tests/test_finance_trackers.py`), plus a riskwatch test
that deadlines.md is in the prompt.

## Rollout
scratch/finance-trackers → staging → master. Add the goals.md rows only at promotion (live reads the
same KB). Run `goals.text()` against a copy of the live DB first.

## Revision 2026-10-22 (Antigravity review)
- Withholding flags both tails (refund too large, or a large amount owed).
- Sinking funds test dollars against a target, so a token transfer doesn't count as funded.
- Replacements are measured against Emergency Fund + Online Checking, not Online Checking alone,
  which is drained each year by travel, holidays and property tax.
- Antigravity suggested retiring "Savings protected" in favour of "Cash reserve". Not done: it is
  Sheep's settled goal, so this is Sheep's call.
