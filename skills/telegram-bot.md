# Telegram Bot

`agents/telegram_bot.py`, running as `agent-telegram-bot.service` (user unit, `Restart=always`).
Stdlib only: long-polls `getUpdates` with a 50 s timeout; only messages from `TELEGRAM_CHAT_ID`
are served, everything else is logged and ignored. Replies go through `shared/telegram.py`.

## Query intake
| Message | Handling |
|---|---|
| plain text, `/ask <q>` | route through the life KB, answer with the local model; escalate to Haiku 4.5 when the local answer ends with `CONFIDENCE: low` or the local model is down |
| `/ask more <q>` | same context, answered by Sonnet 5 |
| `/ask deep <q>` | same context, answered by Opus 5 (the only way Opus is ever used) |
| `/status` | timers (next run, failed units), disk, age of `backup.log` / `mirror.log`, loaded local model, month spend from `token_usage`, last observation per agent |
| `/report` | newest `reports` row (the weekly brief) |
| `/model big\|small` | runs `~/bin/llm-model`, which restarts `llama-server` |

## Two-tier KB load (`shared/kb.py`)
1. `life/_index.md` → local model names ONE domain (validated against the index; keyword fallback if the model is down).
2. `<domain>/_index.md` → leaf links; one level of sub-index is followed (`health/sheep/`). Files the index names but that don't exist yet are dropped; if none exist the domain index itself is the context with the note "has no leaf files yet".
3. The model picks 1–2 existing leaves (skipped when only one exists). Leaf text is capped at 14,000 chars (~3,500 tokens).

Health and people domains are `LOCAL_ONLY_DOMAINS`: never escalated, `/ask more` and `/ask deep` are answered locally with the note "health stays local".

## Response formatting
Answer text, blank line, then a footer: `— <model> · <files> [· confidence medium|low] [· routing notes]`.
The answering prompt asks for a trailing `CONFIDENCE:` line, which is stripped before sending.
Long replies are chunked at 4,000 chars by `shared/telegram.py`.

## Measured (2026-09-13, Gemma 4 26B-A4B on the OptiPlex)
"When was the last oil change on the CR-V?" → `2026-04-15 — local · vehicles/crv_2018.md` in 18 s.
A health question routes to `health/sheep/medications.md` in ~30 s and stays local.

## Alerts (`agents/notify_agent.py`)
`OnFailure=telegram-alert@%n.service` on every agent unit posts `FAILED: <unit> on <host>` plus the
last 8 journal lines. `notify_agent.py send <title>` posts stdin (used by `restore-drill.service`).
