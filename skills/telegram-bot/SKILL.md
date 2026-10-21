---
name: telegram-bot
description: Architecture, query routing, two-tier KB loading, local-only privacy domains, and alert conventions for agents/telegram_bot.py and shared/telegram.py. Use when working on Telegram bot commands, KB routing, or notification alerts.
---

# Telegram Bot

`agents/telegram_bot.py`, running as `agent-telegram-bot.service` (user unit, `Restart=always`).
Stdlib only: long-polls `getUpdates` with a 50 s timeout; only messages from `TELEGRAM_CHAT_ID`
are served, everything else is logged and ignored. Replies go through `shared/telegram.py`.

## Query intake
| Message | Handling |
|---|---|
| plain text, `/ask <q>` | route through the life KB, answer with the local model; escalate to Haiku 5.5 when the local answer ends with `CONFIDENCE: low` or the local model is down |
| `/ask more <q>` | same context, answered by Sonnet 5.5 |
| `/ask deep <q>` | same context, answered by Opus 5.5 |
| `/status` | timers (next run, failed units), disk, age of `backup.log` / `mirror.log`, loaded local model, month spend from `token_usage`, last observation per agent |
| `/report` | newest `reports` row (the weekly brief) |
| `/model big\|small` | runs `~/bin/llm-model`, which restarts `llama-server` |
| `/recipe <link or text>` | food agent (`shared/food.py`): structure (Haiku), course/tags from fixed lists, USDA per-serving nutrition, saved; replies with a `🍽 #id` card. `/recipe 12 [servings N \| course X \| scale F \| why \| delete]`; delete asks with [Delete]/[Keep] |
| `/recipes [words]` | FTS5 search plus filters: course words (dinner = main), `quick`, `<500cal`, tags, cuisine |
| a reply to a `🍽 #id` card | revises that recipe (re-structure + recompute); intercepted before feedback, questions and the privacy router |
| a bare shared link, or shared text with an Ingredients heading (not a reply) | [Save recipe]/[Ignore] buttons sent as a reply; on tap the bot reads the link/text from `reply_to_message` (no stored state) |

## Menu and buttons (2026-10-21)
- `COMMANDS` in `telegram_bot.py` is the one list of commands: `register_commands()` at startup clears the
  default, all-private-chats and chat scopes (the token once served an OpenClaw bot whose 43 commands still
  showed in ☰ Menu) and sets the chat-scope menu; `HELP` is built from the same table. Add new commands there.
- `/menu` and `/start` send the `LAUNCHER` grid; `m:<target>` callbacks run the command via `MENU_TARGETS`.
- Bare commands that need details (`/afford`, `/recipe`, `/ask [more|deep]`, and [Search…]/[+ New check])
  send a `PROMPTS` text with `force_reply`; a reply whose `reply_to_message.text` equals a prompt runs as
  `<command> <reply>` (no stored state; matched before card replies, feedback and the privacy router).
- `/recipes` with no words: latest 10 plus filter chips (`rs:<query>`, edited in place). List rows are
  `/recipe_12 …` tap links (normalised in `handle()`).
- Recipe cards get [×½] [×2] [Why] [Delete] (`r:<id>:half|x2|why|ask`; Delete asks first) via `send_reply()`,
  which adds them to any reply with a `🍽 #id` line that fits one message.
- `/check` with no args lists with ✕ buttons per check (`c:<id>:ask` → confirm `yes`/`no`); `/model` alone
  shows the current model with [Big] [Small] (`mdl:`).
- Callback prefixes in use: `m:` `rs:` `c:` `mdl:` `r:` `p:` `q:`. callback_data must stay ≤ 64 bytes.

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
