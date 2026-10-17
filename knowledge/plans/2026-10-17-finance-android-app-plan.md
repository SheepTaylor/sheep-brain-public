# Finance Companion — Android app design

Status: approved design, decisions settled 2026-10-17; not built. No dollar figures here: `knowledge/` is published. Targets stay in
`life/finance/budget.md` and `life/finance/bills.md`.

## Goal

Answer one question in under 5 seconds, from a store aisle: **"Can we afford this right now?"**
The dashboard (balances, budget lines, essential vs non-essential) is the supporting view, not the
headline.

## Constraints that shape the design

1. **The data is up to two days old.** Monarch has no API; the ledger is refreshed from the CSV
   export, every other day from 2026-10-17. A verdict that ignores this will say "yes" too often. The app must show the export
   age everywhere and estimate the unrecorded spending since it.
2. **Two people spend from the same budget.** If both phones check on the same afternoon, each
   needs to see the other's purchase. That requires a small "logged purchases" table on the server.
3. **The budget logic already exists** in `shared/budget.py` (lines, essential split, Amazon
   share, targets) and `shared/bills.py` (`project()`: checking projection against the checking
   floor). The app must call that code, not reimplement it in Kotlin.
4. **Tailscale-only, no cloud.** There is no Play Store, Firebase, or third-party analytics. Services
   bind to the Tailscale IP only.
5. **Instant and deterministic.** No LLM in the request path. An optional "explain" button can
   route to the Telegram bot's model ladder later.

## Architecture

```
Android app (Kotlin, Compose)            home server
  ├─ Afford screen  ──POST /v1/afford──►  agents/finance_api.py  (stdlib ThreadingHTTPServer,
  ├─ Dashboard      ──GET  /v1/dashboard►   bound to Tailscale IP, bearer token per device)
  ├─ Log purchase   ──POST /v1/pending──►     │ shared/budget.py   actuals, targets, essential split
  └─ Glance widget  (WorkManager, 3 h)        │ shared/bills.py    project(): checking low point
         │                                    │ shared/afford.py   NEW: verdict logic (pure, tested)
    Tailscale app on each phone               └ shared.db.get_db() → ~/data/live/agent.db
```

### Server: `agents/finance_api.py` + `shared/afford.py`

- Stdlib `http.server.ThreadingHTTPServer`, JSON in and out. A framework isn't needed for four
  endpoints.
- `agent-finance-api.service` uses `Restart=always` and
  `OnFailure=telegram-alert@%N.service`, and listens on Tailscale IP port 8090 (8081 is
  already taken).
- Auth: one random token per device in `~/.config/agent-system.env`
  (`FINANCE_API_TOKENS=sheep:…,moose:…`), sent as `Authorization: Bearer`. The device name is
  recorded on logged purchases. Tailscale ACLs are the outer wall, and the token makes revoking
  one phone easy.
- Migration `009_fin_pending.sql`:
  `fin_pending(id, created_at, who, amount, bucket, line, note, matched_txn_id, expired_at)`.
  On each Monarch import, a pending row is matched to a real transaction (same amount within
  ±$1, within 10 days) and drops out, or it expires after 21 days.

The API is plain HTTP inside the tailnet (WireGuard already encrypts it). The app's
`network_security_config` allows cleartext to `*.ts.net` only, and pairing links use the
server's MagicDNS name. Pairing works by opening the `agentfinance://pair?…` link (a camera
QR scan or a paste); `finance_api.py link <name>` prints it.

### Endpoints

| Method | Path | Returns |
|---|---|---|
| GET | `/v1/dashboard` | as-of date, export age, balances, checking projection, month-to-date lines, pending |
| POST | `/v1/afford` | `{amount, bucket, line?}` → verdict + reasons + "after" numbers |
| POST | `/v1/pending` | log a purchase (from the verdict screen or manually) |
| DELETE | `/v1/pending/{id}` | undo |

`/v1/dashboard` sketch:
```json
{"as_of": "2026-10-12", "export_age_days": 5,
 "checking": {"balance": 0, "floor": 0, "projected_low": 0, "low_date": "2026-11-14"},
 "accounts": [{"name": "…", "balance": 0, "goal": 0}],
 "month": {"key": "2026-10", "day": 17, "days": 31,
           "total": {"actual": 0, "target": 0, "pace": 0},
           "essential": {"actual": 0, "target": null},
           "non_essential": {"actual": 0, "target": 0, "unrecorded_est": 0, "pending": 0},
           "lines": [{"name": "Dining (incl. delivery)", "actual": 0, "target": 0, "pace": 0}]},
 "pending": [{"id": 1, "who": "moose", "amount": 0, "note": "shoes"}]}
```

### Verdict logic (`shared/afford.py`, pure function, pytest-covered)

Inputs are the amount and a note ("shoes"); keywords in the note pick the budget line
(default Other shopping), and the line picks the bucket. Both the line (when it has a target)
and the bucket are checked, and the worst result wins.

1. **Monthly room.** Room = target − (actual + pending + estimated unrecorded). The estimated
   unrecorded spending is the trailing 90-day daily average × the days since the newest
   ledger transaction.
2. **Pace (non-essential only).** A purchase is ahead of pace when the bucket's spending after
   it exceeds target × (share of the month elapsed + 10%). Essential bills are too lumpy for this.
3. **No cash check in the verdict.** Card purchases count against the month's budget only
   (Sheep, 2026-10-17). The dashboard shows the checking projection separately.

| Verdict | Rule | Example headline |
|---|---|---|
| **Fits** (green) | room after purchase ≥ 0 and pace OK | "Fits: $X left for 15 days" |
| **Tight** (amber) | fits, but uses >50% of the remaining room or runs ahead of pace | "Tight: $84 left for 15 days" |
| **Over** (red) | the line or the bucket goes below 0 | "Over: $309 over for October" |

Every verdict lists its reasons, including "Data as of Oct 12; ~$Y estimated since." Over is
advice, not a block: the screen still offers "Log it anyway".

## App

- **Platform:** `minSdk = targetSdk = 37` (Android 17), Kotlin, Jetpack Compose with Material 3
  and dynamic color, single activity, edge-to-edge, predictive back.
- **Libraries (minimal):** OkHttp and kotlinx.serialization for the network, DataStore for
  settings, Android Keystore to encrypt the token, WorkManager for widget refresh, and Glance
  for the widget. There is no DI framework, no Room (the server is the source of truth, and the
  last dashboard JSON is cached in a file for offline display), and no analytics.
- **Repo:** new `~/projects/finance-android` (Gradle, built from the command line on the
  server with the Android command-line tools). The release APK is signed with a key in
  `~/.config/android/` (mode 600) and installed by sideloading or by `adb` over Tailscale.

### Screens

1. **Afford (launch screen).** A large amount field with the numeric keypad already open,
   bucket chips (Non-essential is the default, plus Essential), optional line chips (Shopping,
   Dining, Gifts, Personal), and a Check button. The result card shows the verdict color, a
   one-line headline, 2–3 reasons, and the buttons **Log it** and **Not now**. Target: launcher
   → verdict in two taps plus typing the amount.
2. **Dashboard.** An as-of banner (amber at 3+ days, red at 5+ days with "time to export").
   Below it:
   - The checking card: balance, the floor line, and the projected low with its date (a
     sparkline of the 35-day projection).
   - Savings and Emergency Fund against their goals.
   - The month-to-date total against target, with a pace tick.
   - Essential and non-essential as two bars, each with a pace tick and pending spending drawn
     hatched.
   - The budget lines list, where tapping a line shows its top merchants.
3. **Pending.** Logged purchases not yet matched to the ledger, showing who logged each one
   and with swipe-to-undo.
4. **Settings.** Pairing scans a QR code (`python3 -m agents.finance_api qr moose`, run from ~/projects/agent-system)
   that holds the server URL and the token.

Also: a **home-screen widget** ("Non-essential room: $X · as of Oct 12"; tapping it opens
Afford) and an optional **Quick Settings tile** that opens Afford directly.

Privacy: the app sets `FLAG_SECURE` (no screenshots or recents thumbnail), and a biometric
unlock is optional. Both phones see everything, including an Accounts screen with every
balance and net worth; there are no per-device restrictions.

## Phases

| # | Deliverable | Where |
|---|---|---|
| 0 | `/afford 120 shoes` Telegram command using `shared/afford.py` (built 2026-10-17; budget.py gained the detail lines) | `scratch/finance-api` |
| 1 | `finance_api.py`, `shared/dashboard.py`, `shared/pending.py`, migration 009, pending matching in `monarch_sync_agent`, pytest (built 2026-10-17, 6081769, on staging) | same branch → staging → master |
| 2 | Android read-only dashboard + pairing (built 2026-10-17, finance-android 3e44360; server 5be3e5e on staging) | `finance-android` |
| 3 | Afford screen + log purchase + undo (built 2026-10-17, app 874531e v0.3.0; server a6e5a14 adds explicit line + GET /v1/lines, on staging) | |
| 4 | Widget, QS tile, offline cache | |

## Decisions (Sheep, 2026-10-17)

1. **Targets:** the Oct 8 goals budget, now in `life/finance/budget.md`, sets non-essential room
   from its Non-essential line and line room from the detail table (for example "Other shopping,
   clothing, home, gifts" for shoes). Phase 0 adds those detail lines to `shared/budget.py`.
2. **Card purchases** count against this month's budget only. Checking is touched only by the
   projected autopay.
3. **Both phones see everything:** the same view on each, including balances.
4. **The export runs every other day,** so the unrecorded-spend estimate covers at most about
   2 days.
