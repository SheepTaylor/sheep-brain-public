# Local AI System Plan

**Date:** 2026-09-08 · **Owner:** Sheep · **Status:** Approved 2026-09-08. Phases 0, 1, 2, 3, 5 done (3 and 5 on 2026-09-13); Phase 4 (KB population, human-curated) and 6 to do
**Companion files:** `~/projects/local-ai/` (runtime), `~/opt/llama.cpp`, `~/models/`

---

## 1. What you actually have (measured today)

### Hardware

| Item | Fact | What it means for AI |
|---|---|---|
| Machine | Dell OptiPlex 7020 **SFF** (chassis type 15), board 02YYK5 | Low-profile PCIe x16 slot only; ~255 W PSU; bus-powered cards only |
| CPU | i7-4770, 4 cores / 8 threads, Haswell (2013), AVX2 + FMA + F16C, **no AVX-512** | llama.cpp runs fine; 4 threads is the sweet spot (8 is slower) |
| RAM | 32 GB DDR3, **maxed for this board** | Fits one ~17 GB model plus Claude Code; effective bandwidth ~16 GB/s is the hard ceiling on token speed |
| GPU | Intel HD 4600 integrated | Useless for inference |
| Disk | 238 GB SATA SSD, 178 GB free | Room for ~10 models; model load ~0.5 GB/s |
| Network | ~100–180 Mbit down; Tailscale up (`sheep-dell` 100.67.145.91) | 17 GB model = 15–25 min download |
| Power/uptime | Always on, load average 0.00 | The box is idle 99% of the time. That is the resource to exploit. |

### Measured throughput (llama.cpp b10809, CPU, 4 threads)

| Model | Size on disk | Prompt processing | Generation | Verdict |
|---|---|---|---|---|
| Qwen3.5-4B Q4_K_M | 2.5 GB | 24.5 tok/s | 6.3 tok/s | Usable for background jobs; too slow for chat |
| Gemma 4 26B-A4B Q4_K_S (MoE, 4B active) | 16.5 GB | 23.4 tok/s | 5.7 tok/s | **Same speed as the 4B, far more capable.** MoE only activates 4B params per token, so bandwidth cost matches the small model. This is the default now. |

Rule of thumb on this box: a 500-token prompt costs ~20 s, and every 100 words of answer costs ~25 s. A nightly summary job is fine. A conversation is not. The surprise: the 26B mixture-of-experts model runs at the same speed as the 4B dense one, so there is no reason to run the small model except to free RAM.

### Software state

| Layer | State |
|---|---|
| Claude Code 2.1.263, Gemini CLI 0.42, Node 24, Python 3.12 | Working, up to date |
| Local inference | **None existed this morning.** Now: llama.cpp + 2 models + `llama-server` systemd user service (see §5, Phase 1) |
| Agent system (`~/projects/agent-system`) | Skeleton. `financial_agent.py` writes **random numbers**; `synthesis_agent.py` prints a **mock** Telegram push to the journal. 123 observations of fake data. |
| Telegram bot | Does not exist. `skills/telegram-bot.md` is an empty TODO. |
| Life KB (`life/`) | 10 domain indexes, **3 leaf files** (`crv_2018.md`, `utilities.md`, `estate_pointers.md`). Indexes point at ~35 files that don't exist. |
| Raw material for the KB | Sitting on the Drive mount (reorganised 2026-09-12 into `life/`, `system/`, `archive/`; layout in `architecture.md`): `checkbook.db`, 2024–2026 budgets and balance sheets, Health Sync sleep/HR/energy CSVs, both health summaries, CR-V manuals/insurance, homeowner's policy, Codex Vitae, values framework, Life System Architecture |
| VaultGuard | `knowledge/vaultguard.md` is an empty TODO. No secrets store exists on this machine. |
| Backups | restic nightly, **same disk as the data**. `mirror-script.sh` exits 1 by design. No off-machine copy of `agent.db`. |
| Docs vs reality | `tech-stack.md` lists `tmux` and `sqlite3` CLI; **neither is installed**. `architecture.md` says timers aren't live; they are (financial daily, synthesis weekly). |

---

## 2. The honest framing

This machine cannot run anything near frontier quality, and no upgrade that fits its case changes that. What it *can* be is better than a GPU box for your stated goal:

**An always-on, private, model-agnostic orchestrator that owns your data, runs your routines, and calls whichever model is best that year.**

For an individual, "preparing for ASI" is not about owning compute. Compute gets cheaper every year and you rent it by the token. It is about four things that don't come from a vendor:

1. **Your data in your hands, structured.** The life KB and `agent.db` are the asset. Today they are mostly empty while the real data sits unstructured on Drive.
2. **A harness you understand and can swap models in.** One function, `ask(model, prompt)`, with local and cloud behind it. When Claude 6 or a 3B model that beats today's 30B arrives, you change one line.
3. **A written statement of your values that agents are bound by.** You've already drafted it (the "AI Ethics, Values, and Life Alignment Framework", "Codex Vitae"). It needs to become a file every agent loads, not a Google Doc.
4. **Practice.** Weekly hands-on time with agents, MCP, and skills. The 40 years of IT management are the moat; the tooling is what changes.

The local model's job is narrow and important: **anything touching health, finances, or family stays on this box by default.** Cloud models get pointers, aggregates, and questions, not the raw ledger.

---

## 3. Brainstorm: what this box should do (ranked by value ÷ effort)

| # | Idea | Local model role | Cloud role | Effort |
|---|---|---|---|---|
| 1 | **Real financial agent.** Read `checkbook.db` (Drive) nightly; compute balance, 7/30-day spend by category, anomalies; write real observations. | Draft the one-paragraph assessment from the numbers | None (numbers are deterministic; keep it local) | 1 day |
| 2 | **Real synthesis → Telegram.** Weekly cross-domain report actually delivered to your phone. | First-pass summary of the week's observations | Sonnet 5 writes the final report from the local draft + KB indexes (cached) | ½ day |
| 3 | **Telegram assistant with two-tier KB routing.** "When is the CR-V inspection due?" from your phone. | Route + answer lookups (measured: correct on the oil-change test) | Escalate reasoning questions to Haiku 4.5, then Sonnet 5 | 2 days |
| 4 | **Health agent.** Health Sync CSVs (sleep, HR, energy) → weekly trend + flags. Never leaves the box. | Summaries | None | 1 day |
| 5 | **KB population sprint.** Drive PDFs/docs → markdown leaf files via the existing `pdf2md.py`, local model drafts, **you approve and commit** (keeps the human-curated rule). | Extraction and first drafts | Haiku 4.5 for messy PDFs | 3–5 evenings |
| 6 | **Values/constitution file.** Turn the Drive values framework + Codex Vitae into `persona/values.md`; load it in every agent's system prompt. | — | Claude Code session to help you tighten it | 1 evening |
| 7 | **Voice notes.** Gemma 4 E4B accepts audio; Telegram voice message → local transcription → same routing. | Transcribe + answer | — | 1 day (after #3) |
| 8 | **AI-landscape digest.** Miniflux RSS (install script exists in scratch) → nightly local summary → Telegram. Track ASI progress without doom-scrolling. | Summarize 30 articles/night (batch, overnight is fine) | — | 1 day |
| 9 | **Closing Ranks assistant.** Chapter continuity checks (names, dates, places) run locally over `~/writing/closing-ranks`; prose work stays in Claude Code. | Consistency checks | Claude Code for prose | ½ day |
| 10 | **Financial email observations.** Finish `scratch-financial-emails`; Gmail is already connected to Claude Code. | Classify | Extraction | Already started |
| 11 | **Failure alerts.** Any failed systemd unit pings Telegram (`OnFailure=`). Cheap insurance. | — | — | 1 hour |

Deliberately **not** recommended: a vector DB / RAG stack (your May decision was right, KB is small and routed), a web dashboard (Telegram is the interface), running a coding agent on a local model (6 tok/s makes it useless; Claude Code is already the coding agent), Docker (adds nothing on a single-user box).

---

## 4. Target architecture

```
                 Telegram (phone)                      Claude Code / Gemini CLI (tmux, Tailscale)
                        │                                             │
                        ▼                                             ▼
             ┌─────────────────────┐                     ┌─────────────────────┐
             │  telegram_bot.py    │                     │  interactive work   │
             │  (routing agent)    │                     └──────────┬──────────┘
             └──────────┬──────────┘                                │
                        │                                           │
   systemd timers ──►  agents/*.py  ──────────────►  shared/llm.py  ◄────────┘
   (financial, health,                               ask(tier, ...)
    synthesis, digest)                                 │         │
                                              ┌────────┘         └─────────┐
                                              ▼                            ▼
                                  llama-server :8080              Anthropic API
                                  (local, private data)           (Haiku/Sonnet, cached prefixes)
                                              │
              ~/agent-shared/  ◄── reads ──────┤────── writes ──►  ~/data/live/agent.db
              persona/ values/ life/           │                    observations, reports,
              (human-curated, git)             │                    token_usage
                                              ▼
                                   restic → local repo → rclone → Google Drive (offsite)
```

**Model policy (one table, enforced in `shared/llm.py`). Cost is an ongoing constraint, so each agent gets the cheapest model that does its job:**

| Job | Model | Why |
|---|---|---|
| Raw health, ledger rows, family details | `local` only | Never leaves the machine; free |
| KB lookups, classification, drafts, digests, financial and health assessments | `local` | Free, private, fast enough overnight |
| Telegram escalation (question the local model can't answer well) | `claude-haiku-4-5` first; `claude-sonnet-5` if the answer is inadequate | $1/$5 per M tokens vs $2/$10 vs $5/$25 |
| Weekly synthesis report | `claude-sonnet-5` with cached system prefix | Quality matters once a week; still ~5× cheaper than Opus |
| Hard reasoning, writing you'll keep | `claude-opus-5`, on request only (`/ask deep ...`) | Reserved, never a default |
| Coding | Claude Code (already) | — |

---

## 5. Phased plan

### Phase 0 — Close the backup hole (DONE 2026-09-08)

Done, using the Google Drive rclone remote you already had (no new account):

| Step | Result |
|---|---|
| Offsite repo | `rclone:ObsidianVault:backups/restic-mirror`, initialised with the local repo's chunker params so copies deduplicate |
| `mirror-script.sh` | Rewritten: `restic copy` → `forget --prune` → `check --read-data-subset=10%`. Old version kept as `.bak-20260908` |
| First mirror run | 42 of 42 snapshots copied, check passed |
| Retention | Local repo pruned 126 → 42 snapshots (14 daily / 8 weekly / 12 monthly). Same policy now runs inside `backup-script.sh` nightly and in the mirror script |
| Schedule | Local backup 02:00 nightly (unchanged); mirror now **nightly** at 03:00 (was weekly Sunday) |
| Restore drill | Restored from the **mirror** into a temp dir: `agent.db` integrity ok, 123 observations = live, all 96 life KB files byte-identical. Repeatable with `~/scripts/restore-drill.sh` |
| Docs | `knowledge/architecture.md` backup section rewritten to match |

**Two things only you can do:**

1. Store the restic password off this machine. Both repos use `~/.config/restic/password`. If the SSD dies, the mirror on Drive is unreadable without it. Put it in VaultGuard / your password manager as `restic_repo_password`.
2. `sudo apt install -y tmux sqlite3` (type `! sudo apt install -y tmux sqlite3` in this session). Needs your password; nothing else in the plan does.

### Phase 1 — Local model runtime (DONE today)

Installed and running, nothing needs sudo:

| Piece | Location |
|---|---|
| llama.cpp prebuilt b10809 (CPU/AVX2) | `~/opt/llama.cpp` |
| Qwen3.5-4B Q4_K_M (fallback when RAM is needed elsewhere; `llm-model small`) | `~/models/` |
| Gemma 4 26B-A4B Q4_K_S (**default**, always loaded, ~17 GB RAM) | `~/models/` |
| `llama-server.service` (user unit, enabled, 127.0.0.1:8080, OpenAI-compatible) | `~/.config/systemd/user/` |
| API key (local only) | `~/.config/llama-server.key` |
| `llm "question"` and `llm-model small|big` | `~/bin/` |

Verified: `echo <numbers> | llm "what is the trend?"` returns a sensible two-sentence answer in ~10 s on Gemma (it noticed the balance was trending down; the 4B model did not).

To expose it to your laptop over Tailscale later, change `--host 127.0.0.1` to `--host 100.67.145.91` in the unit; the API key already gates access.

Optional additions when wanted: `ggml-org/gemma-4-E4B-it-GGUF` (4.6 GB, **audio + image input**, for voice notes), plus its 60 MB MTP drafter file which llama.cpp uses to speed generation.

### Phase 2 — Make the agents real (DONE 2026-09-08)

Built in `scratch/real-agents`, promoted through `staging` to `master`. 27 tests, real scratch DB, HTTP mocked only.

| Piece | What it does now |
|---|---|
| `shared/llm.py` | `ask_local` (llama-server, free, private) and `ask_claude` (Haiku 4.5 default, Sonnet 5 for synthesis, Opus never by default). Static system text is cache-marked. Every call writes a `token_usage` row (migration 002) with an estimated cost; `month_cost_usd()` sums the month. Both raise `ModelUnavailable` so agents degrade a step instead of failing. |
| `shared/checkbook.py` | Reads the Android Checkbook export (`archive/checkbook/` on the Drive mount, stopped 2025-05); deterministic balances, 7/30-day spend vs prior 30, top payees, largest expense, uncleared items, and **`days_stale`**. |
| `financial_agent.py` | Real numbers replace the random mock. The local model only phrases them. First real observation: total balance across 4 accounts, spend down vs prior month, and **"the ledger has not been updated in 466 days"**. |
| `synthesis_agent.py` | Last 7 days of observations → local draft → Sonnet 5 final (persona + KB index as cached prefix) → Telegram. Without an API key it sends the local draft; without Telegram it stores the report and says so. |
| `shared/telegram.py`, `shared/env.py` | Delivery and secrets loading from `~/.config/agent-system.env` (mode 600, never committed; systemd units load it via `EnvironmentFile=`). |
| `infra/` | Agent unit files now tracked in git. |

**Finding that changes Phase 4:** the newest checkbook export is from 2025-05-30. The Android app stopped being exported (or used) 15 months ago, and the Flutter replacement in `~/projects/checkbook_register` is not producing data yet. The financial agent will keep saying so every night until a current ledger exists. Options: resume exporting the Android app to Drive, finish the Flutter app with a Drive export, or switch the agent to bank CSV downloads (the `transaction_download` files on Drive show the format).

**To finish the loop, fill in `~/.config/agent-system.env`:**

| Variable | Where to get it | Unlocks |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | Telegram → @BotFather → `/newbot` | Weekly brief on your phone |
| `TELEGRAM_CHAT_ID` | message @userinfobot, copy the numeric id | same |
| `ANTHROPIC_API_KEY` | console.anthropic.com → API keys (needs a small prepaid balance; separate from Claude Code) | Sonnet 5 polish of the weekly brief (~$0.02/week). Optional: without it the local draft goes out. |

### Phase 3 — Telegram routing bot (DONE 2026-09-13: `agents/telegram_bot.py`, `shared/kb.py`, `agent-telegram-bot.service`; see `skills/telegram-bot.md`. Voice notes still to do.)

- `agents/telegram_bot.py`, long-polling (`getUpdates`), stdlib only, as a `systemd --user` service.
- Allowlist your Telegram user ID; ignore everyone else.
- Flow per `skills/life-knowledge-routing.md`: `life/_index.md` → domain `_index.md` → 1–2 leaf files → local model answers. If the local model replies with a confidence marker below threshold or the question is analytical, escalate to Haiku 4.5 with the same context (Sonnet 5 on `/ask more`).
- Commands: `/ask`, `/status` (timers, disk, last backup, model loaded), `/model big|small`, `/report` (last synthesis).
- Later: voice notes via Gemma 4 E4B.

### Phase 4 — Fill the knowledge base (the actual asset)

One evening per domain, in this order: **finance → health → household → vehicles → people → digital**. For each:

1. Pull sources from the Drive mount (`pdf2md.py` in scratch already converts PDFs).
2. `cat source.md | llm "Extract facts as markdown bullets for <domain>/<leaf>.md; no account numbers"` for a first draft.
3. You edit, verify, commit to the private `life` repo. Pre-commit hook scans for account-number patterns (your R9 requirement).
4. Write `persona/values.md` from your values framework and Codex Vitae; add it to `CLAUDE.md`, `GEMINI.md`, and every agent system prompt.

Exit criterion: every file named in a domain `_index.md` exists, and the Telegram bot answers one real question per domain within 3,500 tokens.

### Phase 5 — Operations (DONE 2026-09-13 except `tmux`/`sqlite3`, which need sudo)

- `OnFailure=telegram-alert@%n.service` on every agent unit; the alert unit posts the failing unit name to Telegram.
- `~/scripts/restore-drill.sh` on a quarterly timer (mirror check already runs nightly).
- `/status` in Telegram shows token spend this month from `token_usage`.
- Update `tech-stack.md` and `architecture.md` to match reality (local runtime, timers live, tmux/sqlite3 installed, offsite mirror configured).

### Phase 6 — Hardware decision, after 30 days of real use

Decide with data (`token_usage` and journal timings), not enthusiasm. See §7.

---

## 6. Running costs

Anthropic list prices (Sept 2026), per million tokens: Haiku 4.5 $1 in / $5 out; Sonnet 5 $2 / $10; Opus 5 $5 / $25. Cache reads are a fraction of the input price.

| Job | Tokens | Model | Cost |
|---|---|---|---|
| Weekly synthesis (6 k cached context, 2 k fresh, 1.5 k out) | ~10 k | Sonnet 5 | ~$0.02 |
| Telegram escalation (4 k context, 500 out) | ~5 k | Haiku 4.5 | ~$0.01 |
| 20 escalations + 4 syntheses a month | | | **well under $0.50/month** |
| Everything local (financial, health, digest, lookups) | | Gemma 4 on this box | $0 and private |

Your remaining Fable credits are for *building*, not running. Running this system costs cents a month.

## 7. Hardware options (in order of my recommendation)

| Option | Cost | What you get | Notes |
|---|---|---|---|
| **A. Nothing now** | $0 | Everything in §3 works overnight at 6 tok/s | Recommended until Phases 2–4 are done; the bottleneck is empty KB, not tokens/sec |
| B. RTX A2000 12 GB (used) | ~$400–650 | Fits the 7020 SFF: low-profile, 70 W bus-powered, no PSU change. 4B models ~40 tok/s, 12B Q4 fully in VRAM ~25 tok/s, 26B-A4B partial offload | The only meaningful upgrade this chassis accepts. Skip the 6 GB version. Needs `llama-vulkan` or CUDA build (both prebuilt) |
| C. New mini PC with unified memory (Ryzen AI Max+ 395, 64–128 GB) | ~$1,500–2,200 | 70B-class Q4 at usable speed, 26B-A4B at 40+ tok/s, real local chat | Keep the OptiPlex as the agent/backup host. This is the "serious local AI" tier if you want it |
| D. Mac mini M4 Pro 64 GB | ~$2,000 | Similar to C, excellent llama.cpp support | Different OS; breaks your Ubuntu-native rule |

Not worth it: more RAM (maxed), a faster SSD (loading isn't the bottleneck), any card needing a 6-pin connector (PSU can't).

---

## 8. What to spend the remaining Fable credits on

In order. Each is one Claude Code session in a scratch worktree.

1. **Phase 3** — Telegram routing bot. (~$10–15)
2. **Phase 4 kickoff** — finance and health domains populated from Drive, values file drafted from your framework. (~$10)
3. **Phase 5** — alerts, restore drill timer, docs updated. (~$5)

That leaves margin. When credits run low, the same work continues on Opus 5 through your normal subscription.

---

## Appendix A — Commands cheat sheet

```bash
systemctl --user status llama-server         # is the local model up
llm-model big                                # Gemma 4 26B-A4B, the default (~35 s load, 17 GB RAM)
llm-model small                              # Qwen3.5-4B when you need the RAM back
llm "question"                               # ask it
cat notes.md | llm "summarize in 5 bullets"  # pipe a file
curl -s localhost:8080/v1/models -H "Authorization: Bearer $(cat ~/.config/llama-server.key)"
journalctl --user -u llama-server -n 50
```

## Appendix B — Files created today

- `~/opt/llama.cpp/` — prebuilt llama.cpp b10809
- `~/models/Qwen3.5-4B-Q4_K_M.gguf`, `~/models/gemma-4-26B-A4B-it-UD-Q4_K_S.gguf`
- `~/.config/systemd/user/llama-server.service` (enabled), `~/.config/llama-server.env`, `~/.config/llama-server.key`
- `~/bin/llm`, `~/bin/llm-model`
- `~/projects/local-ai/` — unit file copy + README
- this document

Nothing under `~/projects/agent-system`, `~/agent-shared/life`, or `~/data` was modified.
