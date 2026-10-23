# Local Gmail mirror: pull mail to the server, process it locally

Status: LIVE 2026-10-23. Drafted by Antigravity; reviewed and built by Claude Code (see "Review changes"). First sync 22,827 messages / 2.0 GB; candidate check: all 201 bill emails from live IMAP over 120 days identical in the mirror, plus 10 Dominion emails Gmail's word-based `from:` misses. mail-sync.timer every 30 min. Pending: restic include (Sheep), open decisions below.

## Problem

Agents read Gmail live over IMAP. `shared/gmail.py` runs a Gmail search (`X-GM-RAW`) on every
call, and `bills_agent` is its only user. That works for one narrow query. But each new email
use means another live round trip to Google. Nothing is kept locally, so there is no archive
that is independent of Google and nothing to back up. Thunderbird is on the machine, but it is
a desktop app: it syncs only while it is open, and it owns its mbox and `.msf` files.

Size today: `[Gmail]/All Mail` holds 22,825 messages (IMAP STATUS, 2026-10-23). The disk has
150 GB free.

## Design

```
Gmail ──IMAP, pull only──▶ mbsync (systemd timer, every 30 min)
                               │
                               ▼
                   ~/data/mail/gmail/  (Maildir, one mirror shared by all tiers)
                               │  read-only
                               ▼
              mail_index_agent ──▶ agent.db: mail_messages (metadata only)
                               │
              consumers: bills_agent (local search), later a triage digest
```

### 1. Mirror (mbsync, no Python)

- `apt install isync`. The config is `infra/mbsyncrc`, symlinked to `~/.mbsyncrc`. It holds no secret.
- Only one Gmail folder is synced: `[Gmail]/All Mail`. It already contains Inbox, Sent and
  every label, so syncing label folders as well would only create duplicates. Spam and Trash
  are excluded.
- The sync can never change Gmail:
  `Sync Pull`, `Create Near`, `Remove None`, `Expunge None`, `CopyArrivalDate yes`.
  Nothing is pushed back, and no flags are changed on Google's side.
- Authentication uses the existing app password:
  `PassCmd "sed -n 's/^GMAIL_APP_PASSWORD=//p' ~/.config/agent-system.env"`.
  There is no second copy of the secret.
- Mail deleted in Gmail stays in the mirror, because `Expunge None` only applies to the near
  (local) side. That makes the mirror an archive. A message can only be purged locally by Sheep.
- Units: `infra/mail-sync.service` and `infra/mail-sync.timer`, every 30 min, with
  `Persistent=true` and `OnFailure=telegram-alert@%n.service`, the existing alert template.
- **First sync:** Gmail caps IMAP downloads at about 2.5 GB/day, so the initial pull may take
  more than one day. mbsync resumes where it stopped, so no special handling is needed. The
  first sync is run by hand in tmux.
- Thunderbird is untouched. It stays Sheep's mail reader, connected to Gmail over IMAP.

### 2. Index (`shared/mail.py` + `agents/mail_index_agent.py`)

- The index agent walks `cur/` and `new/` in the Maildir with the stdlib `mailbox.Maildir`, opened read-only.
  It never moves, renames or flags a file.
- It records metadata only, in a new table `mail_messages`:
  - `msg_id` (primary key; dedupe on Message-ID);
  - `maildir_key`, `received`, `sender`, `subject`, `size`;
  - `domain` (see section 3) and `indexed_at`.
- Bodies stay in the Maildir. They are parsed on demand with the existing `gmail.parse()`, so
  `agent.db` stays small, and so does the nightly copy to candidate.
- The agent is incremental: it skips keys already indexed. It runs as `ExecStartPost` of
  `mail-sync.service`, so there is no separate timer.
- `shared/mail.py` exposes `search(conn, since, senders)`, which returns the same dicts as
  `gmail.fetch()`, and `body(key)`.
- The mirror is a single copy for all tiers. Each tier's DB has its own `mail_messages` index,
  and every tier reads the same files read-only.

### 3. Privacy and untrusted input

- **Every message is local-only by default.** `domain` is set by deterministic sender rules,
  kept in the same TOML style as `tasks.toml` (`agent/mail_rules.toml`):
  - known finance senders (the bills list, Monarch, banks) → `finance`. These may use
    `ask_claude` per `tasks.toml`;
  - MyChart, pharmacies, insurers → `health`, and family addresses → `people`. These are
    `LOCAL_ONLY_DOMAINS`, so they only ever reach `ask_local`;
  - everything else → `unclassified`, treated as local-only.
- Email is untrusted input. Mail-reading code gets no write tools, cannot send, and cannot
  propose KB changes. Its output goes only to `agent.db` rows and Telegram text. This is the
  same rationale as the KB-change plan.

### 4. Switch the bills agent to the mirror

- `bills_agent.sync_emails` already takes `fetch=` as a parameter. Production passes
  `mail.search`, bound to the DB connection.
- **Staleness fallback:** if the newest `mail-sync` success is more than 6 h old, the agent
  falls back to `gmail.fetch` (live IMAP) and logs a warning, so bills never silently miss
  mail. `gmail.fetch` stays as that fallback.

### 5. Backup

- Add `~/data/mail/` to the restic include list (see `life/digital/backups.md`). Expect a few
  GB; deduplication makes later nightly runs small.

## Tests (pytest, real scratch DB)

- A fixture Maildir is built in `tmp_path` from `.eml` files in `tests/fixtures/mail/`: plain,
  HTML-only, duplicate Message-ID, missing Date, and non-UTF-8 messages.
- Indexing is idempotent: running it twice gives the same rows, and the Maildir files are byte-identical before and after.
- Sender rules: the finance, health, people and unclassified cases, plus the default of local-only.
- `mail.search` returns the same shape as `gmail.fetch`. The bills extraction tests pass with
  `fetch=mail.search`.
- Staleness: when the last sync is older than 6 h, the agent calls the fallback.

## Rollout (per sdlc.md)

1. Create `scratch/mail-mirror`. Install isync, run the first sync by hand into `~/data/mail/`, then build the indexer and its tests.
2. Merge into candidate. Run bills on candidate against the mirror and compare the extracted bills with live IMAP for 3 days.
3. Merge into live. Enable `mail-sync.timer`, then add the path to restic.

Out of scope: sending mail, Gmail labels (mbsync does not see `X-GM-LABELS`), attachments
beyond what `gmail.parse` already reads, and full-text search (the Maildir can be grepped; add
SQLite FTS only if a consumer needs it).

## Alternatives considered

- **Thunderbird as the store:** rejected. Thunderbird is a GUI app, it syncs only while
  running, its compaction rewrites mbox files while they may be read, and failures can't be
  monitored.
- **Gmail API:** rejected. It needs an OAuth app and token refresh, and it does not produce a
  local archive.
- **Keeping live IMAP only:** this is today's setup. It is fine for one query, but it gives no
  archive and no offline reads, and each new email use is another trip to Google.

## Review changes (2026-10-23, built this way on scratch/mail-mirror)

1. **Only live would ever be indexed.** `ExecStartPost` runs in the live unit, so candidate and
   scratch DBs would never get `mail_messages`, and the rollout's candidate comparison could
   not run. `mail.search()` now indexes incrementally before it queries, so every tier catches
   up when it reads; the `ExecStartPost` index keeps live warm.
2. **Staleness had no source.** Nothing wrote a "mail-sync success" anywhere a tier could read.
   The unit now touches `~/data/mail/.last-sync` after a successful mbsync; `mail.fresh()` checks
   its age. It is the same for every tier.
3. **`msg_id` as primary key loses files and re-parses duplicates.** Messages without a
   Message-ID would collide, and a duplicate's key would never be recorded, so it would be
   re-read every run. The key is `maildir_key`; `msg_id` is indexed and readers dedupe on it.
   `msg_id` is normalized exactly as `gmail.parse` does, so `bill_emails` dedupes across the
   mirror and the IMAP fallback.
4. **Missing Date.** `gmail.parse` gives `received = None` and bills skips such mail; the index
   falls back to the file's mtime, which `CopyArrivalDate` sets to Gmail's arrival date.
5. **Alert template.** The repo uses `OnFailure=telegram-alert@%N.service` (`%N`, not `%n`).
6. **mbsync rename.** With `Sync Pull`, a message deleted in Gmail is flagged `T` on the near
   side, which renames its file; Maildir keys stay stable, and `mail.body()` retries the lookup
   once if a sync renames a file while it is being read.
7. **Rule order.** A sender matching both finance and a local-only domain is classified
   local-only. The finance list also takes the billers' senders from `bills.md` automatically,
   so the two lists do not drift. A sender's domain is fixed when the message is first indexed.
8. **`~/.mbsyncrc` symlink dropped.** The unit runs `mbsync -c infra/mbsyncrc gmail`; the first
   hand sync uses the same command. `UserCmd` reads `GMAIL_ADDRESS` from the env file too.
9. Spam and Trash are not in All Mail, so "excluded" needs no config.

Still for Sheep: `sudo apt install isync` (no sudo for agents), the first sync in tmux, and the
restic include (the backup config is outside the repo).

## Open decisions for Sheep

1. **Phase-3 use:** what should be built on the mirror first? Options: a daily triage digest in
   the morning heartbeat, with local-model ranking of what needs a reply or action; extraction
   of receipts and orders into the finance tables; or nothing beyond bills for now.
2. **Archive semantics:** keep mail locally after it is deleted in Gmail (as designed), or mirror deletions?
3. **Backup:** add `~/data/mail/` to restic (recommended), or treat Gmail as the backup?
