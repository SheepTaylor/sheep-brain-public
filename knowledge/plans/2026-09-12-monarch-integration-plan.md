# Monarch Money as the financial data source

**Date:** 2026-09-12 · **Owner:** Sheep · **Status:** Decided 2026-09-12: manual CSV export only (Sheep confirmed from Monarch's in-app docs that no API is available). CSV path built on `scratch/monarch` the same day.

## 1. Why

The financial agent reads the Android Checkbook export on the Drive mount. The newest
file is from 2025-05-30, it covers four hand-entered accounts, and the categories are
free text. Monarch already aggregates every account (bank, cards, brokerage, loans),
normalises merchants, assigns categories, tracks recurring bills, budgets, and daily
balances. Replacing the checkbook feed with a Monarch feed gives the agents a complete,
current ledger without any manual data entry.

## 2. Research findings (2026-09-12)

### 2.1 Ways to get data out of Monarch

| Route | What you get | Automation | Risk |
|---|---|---|---|
| **A. Unofficial GraphQL API** via the `monarchmoneycommunity` Python package (PyPI 1.5.2, fork of `hammem/monarchmoney`) | Accounts, transactions (paged, date-filtered, with category/merchant/tags/notes/pending/recurring flags), categories, budgets, cash flow, recurring streams, per-account balance history, holdings, credit history | Full: nightly systemd timer | Reverse-engineered, unsupported by Monarch. Endpoint moved to `api.monarch.com` in Jan 2026 and broke every stale client. Login can be refused with `CAPTCHA_REQUIRED`; the fork then needs a browser-cookie login. Sessions last months when saved. |
| **B. Manual CSV export** (Settings → Data → Download Transactions; Accounts → balance history) | Transactions: `Date, Merchant, Category, Account, Original Statement, Notes, Amount, Tags` (+ transaction ID). Balances: one row per account per day | Manual, drop file on Drive | Zero API risk. Monarch recommends ≤ 5,000 rows per download. Only as current as the last export. |
| **C. Community MCP servers** (`jamiew/monarch-mcp`, `robcerda/monarch-mcp-server`) | Same data as A, exposed as Claude tools | Interactive only | Wrap route A; useful for ad-hoc questions in Claude Code, not for scheduled agents. |
| **D. Official API** | None. As of Aug 2026 Monarch publishes no customer API, OAuth, or MCP server. The "Public API" on status.monarch.com is only the status-page JSON. | | |

The original `monarchmoney` 0.1.15 package still hard-codes `api.monarchmoney.com`
and is dead. `monarchmoney-enhanced` is abandoned in favour of a TypeScript port.
`monarchmoneycommunity` is the one Python fork maintained through 2026 (gql 4.0,
aiohttp, oathtool for TOTP; `login(email, password, mfa_secret_key=...)`,
`login_with_cookies(cookie_header)`, `save_session()` / `load_session()` to a pickle
under `~/.mm/`, `get_transactions(limit, offset, start_date, end_date, ...)`).

### 2.2 Decision

**Superseded 2026-09-12: route B (manual CSV export) only.** Sheep checked Monarch's in-app documentation; the API is not available, and the unofficial client is not worth the credential and breakage risk for a home system. Original reasoning kept below for the record.

~~Route A as the nightly feed, route B as the fallback and bootstrap.~~ Both land in
the same SQLite tables, so every agent downstream is indifferent to which one filled
them. If Monarch ever blocks scripted login for good, the system degrades to a
monthly CSV drop rather than to nothing.

Credentials: Monarch email, password, and TOTP secret go in `~/.config/agent-system.env`
as `MONARCH_EMAIL`, `MONARCH_PASSWORD`, `MONARCH_MFA_SECRET`, backed by VaultGuard
entries `monarch_email`, `monarch_password`, `monarch_mfa_secret`. Google-login users
must first set a password in Monarch (Settings → Security). Enabling TOTP MFA on the
Monarch account is a prerequisite and also a security improvement.

Privacy: raw rows never leave the box. Only the local model sees transaction text.
Cloud models (Haiku/Sonnet) get aggregates from the summary, as today.

## 3. Data model (migration `003_monarch.sql`)

```sql
CREATE TABLE IF NOT EXISTS fin_accounts (
    id            TEXT PRIMARY KEY,      -- Monarch account id
    name          TEXT NOT NULL,
    type          TEXT,                  -- depository, credit, investment, loan, ...
    subtype       TEXT,
    institution   TEXT,
    is_asset      INTEGER NOT NULL,
    balance       REAL,
    balance_date  TEXT,
    hidden        INTEGER NOT NULL DEFAULT 0,
    updated_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS fin_transactions (
    id              TEXT PRIMARY KEY,    -- Monarch transaction id (also in CSV export)
    account_id      TEXT NOT NULL,
    date            TEXT NOT NULL,       -- YYYY-MM-DD
    amount          REAL NOT NULL,       -- negative = expense, Monarch sign convention
    merchant        TEXT,
    original_desc   TEXT,
    category        TEXT,
    category_group  TEXT,                -- income / expense / transfer
    tags            TEXT,                -- comma-joined
    notes           TEXT,
    pending         INTEGER NOT NULL DEFAULT 0,
    is_recurring    INTEGER NOT NULL DEFAULT 0,
    hide_from_reports INTEGER NOT NULL DEFAULT 0,
    source          TEXT NOT NULL,       -- 'api' | 'csv'
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS ix_fin_txn_date ON fin_transactions(date);

CREATE TABLE IF NOT EXISTS fin_balances (   -- daily snapshot per account
    account_id  TEXT NOT NULL,
    date        TEXT NOT NULL,
    balance     REAL NOT NULL,
    PRIMARY KEY (account_id, date)
);

CREATE TABLE IF NOT EXISTS fin_recurring (
    id          TEXT PRIMARY KEY,
    merchant    TEXT,
    amount      REAL,
    frequency   TEXT,
    next_date   TEXT,
    account_id  TEXT,
    updated_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS fin_sync_log (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   DATETIME DEFAULT CURRENT_TIMESTAMP,
    source      TEXT NOT NULL,           -- 'api' | 'csv'
    status      TEXT NOT NULL,           -- ok | auth_failed | captcha | error
    rows_upserted INTEGER DEFAULT 0,
    detail      TEXT
);
```

Everything is upsert-by-id, so re-runs and overlapping CSV drops are idempotent.
Budgets are left out of the first cut; `get_budgets()` can feed a later `fin_budgets`
table once the spending agent is stable.

## 4. Components to build

| # | Component | Purpose |
|---|---|---|
| 1 | `shared/monarch.py` | Thin wrapper over `monarchmoneycommunity`: `connect()` (load saved session, else login with TOTP, raise `AuthBlocked` on `CaptchaRequiredException`), `fetch_accounts()`, `fetch_transactions(since)`, `fetch_balances(days)`, `fetch_recurring()`. Async internally, sync functions exposed. Session pickle at `~/.config/monarch-session.pickle`, mode 600. |
| 2 | `shared/monarch_csv.py` | Parse the Monarch transactions CSV and balance-history CSV from `~/mnt/gdrive/feeds/monarch/`. Same output dicts as component 1. |
| 3 | `shared/ledger.py` | Upsert into the `fin_*` tables; `summarize(conn, today)` producing the same shape as `checkbook.summarize()` plus: net worth, cash vs credit vs investment split, spend by category group (30d vs prior 30d), recurring bills due in 14 days, pending count, top merchants, largest transactions, unusual transactions (> 2× the 90-day median for that merchant or category). Deterministic; no model calls. |
| 4 | `agents/monarch_sync_agent.py` | Nightly 23:30. Tries API (incremental: `since = last synced date − 7 days` to catch late-posted and edited rows); on `AuthBlocked` or any failure falls back to newest CSV on Drive; writes `fin_sync_log`. Exit non-zero only when both sources fail. |
| 5 | `agents/financial_agent.py` (rewrite) | Reads from `ledger.summarize()` instead of `checkbook`. Keeps the existing observation format so `synthesis_agent` needs no change. `days_stale` now means days since the last successful sync. Local model phrases the summary as today. |
| 6 | `infra/agent-monarch-sync.{service,timer}` | Systemd user units, `EnvironmentFile=~/.config/agent-system.env`, `After=` ordering so financial runs at 00:00 with fresh data. |
| 7 | `scripts/monarch-login.py` | One-off interactive helper: performs the first login (TOTP, or pasting a browser `Cookie:` header when CAPTCHA blocks the password route) and saves the session pickle. Documented in `agent-system.md`. |
| 8 | Tests | `tests/test_monarch_csv.py` (fixture CSV), `tests/test_ledger.py` (upsert idempotency, summary numbers on a seeded scratch DB), `tests/test_monarch_sync_agent.py` (API stub raising `AuthBlocked` → CSV fallback path). No live Monarch calls in tests. |
| 9 | Retire `shared/checkbook.py` | Delete module and tests once the ledger has been populated for 7 days and the Checkbook history has been imported (Monarch supports CSV import, so the 2025 checkbook rows can be pushed into Monarch once if you want them there; not required). |

Telegram bot (Phase 3) later gains `/spend`, `/bills`, `/networth` that read `fin_*`
tables through the same `ledger.summarize()`; nothing here blocks Phase 3.

## 5. Phased plan (revised 2026-09-12 for CSV only)

**Operating routine:** roughly weekly, in Monarch on the web, Settings → Data → Download Transactions and the balance-history download, saved into `~/mnt/gdrive/feeds/monarch/` (the `feeds/monarch` folder in Drive; another folder works with `MONARCH_EXPORT_DIR`). The nightly financial run loads the newest pair automatically. When the last transaction is more than 7 days old the daily note says an export is due.

| Phase | State |
|---|---|
| M0 prerequisites | Done: exports on Drive. No Monarch credentials needed. |
| M1 API spike | Dropped. |
| M2 tables and importers | Done on `scratch/monarch` (CSV importer, ledger, tests). |
| M3 agents | Done: `monarch_sync_agent` runs as `ExecStartPre` of the financial unit; financial agent reads the ledger. Promote via staging → master. |
| M4 later | ~~Retire `checkbook.py`~~ (done 2026-09-12); `fin_recurring` from Monarch's recurring export if one appears; budgets; Telegram `/spend` `/bills` `/networth`. |

Original phase text follows.


### Phase M0 — Prerequisites (you, ~20 min)
1. In Monarch: set a password if you sign in with Google; enable TOTP MFA and copy the "Two-factor text code" (the TOTP secret).
2. Create VaultGuard entries `monarch_email`, `monarch_password`, `monarch_mfa_secret`; add the three vars to `~/.config/agent-system.env`.
3. Do one manual export (Settings → Data → Download Transactions, and one balance-history CSV) into `~/mnt/gdrive/feeds/monarch/`. This is the bootstrap and the fallback test fixture.

### Phase M1 — Spike (1 session, scratch worktree `scratch/monarch`)
- `pip install monarchmoneycommunity` in the worktree venv; add to `requirements.txt`.
- Run `scripts/monarch-login.py`; confirm `get_accounts()` and a 30-day `get_transactions()` work from this box. Record whether the password+TOTP route works or the cookie route is needed.
- Exit criterion: session pickle saved, account list printed, no CAPTCHA.
- If the API is blocked outright: skip to M2 with CSV only and revisit the API in a month.

### Phase M2 — Ledger tables and importers (1 session)
- Migration 003, `shared/ledger.py`, `shared/monarch_csv.py`, `shared/monarch.py`, tests.
- Load the full history once (API in 90-day pages, or the CSV bootstrap).
- Exit criterion: `sqlite3 ~/data/scratch-monarch/agent.db "select count(*), min(date), max(date) from fin_transactions"` matches Monarch's own count.

### Phase M3 — Sync agent and financial agent rewrite (1 session)
- Components 4, 5, 6. Merge to staging, run in candidate for three nights, then master.
- Exit criterion: the daily observation reports today's balances and a 30-day category breakdown, and `days_stale` is 0 or 1.

### Phase M4 — Clean-up and next uses (later)
- ~~Retire `checkbook.py`~~ (done 2026-09-12). Add `fin_budgets` and budget-vs-actual to the summary.
- Feed the finance leaf files in `life/finance/` (cash, credit, subscriptions) from `fin_accounts` and `fin_recurring`; still human-curated, but the model draft comes from real data.
- Weekly synthesis gets net-worth trend and upcoming bills.

## 6. Cost and risk

- Running cost: zero cloud tokens. Sync is network I/O; summary is deterministic; phrasing is the local model.
- Main risk is Monarch closing the unofficial API. Mitigation: the CSV path is built in the same phase, and `fin_sync_log` plus the existing `days_stale` reporting will make an outage visible in the next daily note.
- Credential risk: password and TOTP secret sit in a mode-600 env file on a single-user Tailscale-only box, the same posture as the Anthropic and Telegram keys. Do not put them in the repo, `.envrc`, or the Drive mount.

## Sources
- Monarch help: [Downloading Transaction or Account History](https://help.monarch.com/hc/en-us/articles/15526600975764-Downloading-Transaction-or-Account-History), [Importing Transactions Manually](https://help.monarch.com/hc/en-us/articles/4409682789908-Importing-Transactions-Manually)
- Python clients: [bradleyseanf/monarchmoneycommunity](https://github.com/bradleyseanf/monarchmoneycommunity), [hammem/monarchmoney](https://github.com/hammem/monarchmoney) (dead endpoint), [keithah/monarchmoney-enhanced](https://github.com/keithah/monarchmoney-enhanced) (unmaintained)
- Endpoint change, Jan 2026: [home-assistant/core#161069](https://github.com/home-assistant/core/issues/161069)
- MCP servers: [jamiew/monarch-mcp](https://github.com/jamiew/monarch-mcp), [robcerda/monarch-mcp-server](https://github.com/robcerda/monarch-mcp-server)
- No official API: [openbudget.sh summary](https://www.openbudget.sh/blog/does-monarch-money-connect-to-claude), [status.monarch.com/public-api](https://status.monarch.com/public-api)
