---
name: telegram-bot
description: Architecture, query routing, two-tier KB loading, local-only privacy domains, and alert conventions for agents/telegram_bot.py and shared/telegram.py. Use when working on Telegram bot commands, KB routing, or notification alerts.
---

# Telegram Bot

`agents/telegram_bot.py`, running as `agent-telegram-bot.service` (user unit, `Restart=always`).
Stdlib only: long-polls `getUpdates` with a 50 s timeout; only messages from `TELEGRAM_CHAT_ID` (and Moose's chat, below)
are served, everything else is logged and ignored. Replies go through `shared/telegram.py`.

## Query intake
| Message | Handling |
|---|---|
| plain text, `/ask <q>` | route through the life KB, answer with the local model; escalate to Haiku 5.5 when the local answer ends with `CONFIDENCE: low` or the local model is down |
| `/ask more <q>` | same context, answered by Sonnet 5.5 |
| `/ask deep <q>` | same context, answered by Opus 5.5 |
| `/status` | timers (next run, failed units), disk, loaded local model, month spend from `token_usage`, message budget, data feeds (`shared/feeds.py`: Monarch, mail mirror, Drive, Health Connect, MyChart, backups), last run per agent-core agent |
| `/activity [days]` | what ran in the background (`shared/activity.py`): last run per timer-driven unit, model spend per agent, agent-core runs, messages sent and held, data loads, KB proposals. Sheep only |
| `/today`, `/week [days]` | deterministic calendar view (`shared/agenda.py`): non-routine events from primary, Moose and Sheep and the Agent calendar (🤖), merged by exact start, one-car conflicts (🚗), Gamecocks games; refreshes the calendar first when the copy is over 2 h old. Sheep only; never passed to a model |
| `/report` | newest `reports` row (the weekly brief) |
| `/model big\|small` | runs `~/bin/llm-model`, which restarts `llama-server` |
| `/recipe <link or text>` | food agent (`shared/food.py`): structure (Haiku), course/tags from fixed lists, USDA per-serving nutrition, saved; replies with a `🍽 #id` card. `/recipe 12 [servings N \| course X \| scale F \| why \| delete]`; delete asks with [Delete]/[Keep] |
| `/ate …`, `/meals [id]` (Sheep only) | meal log (`shared/meals.py`): log by number, name or free text with date/slot/servings words; ambiguous names ask with `ma:` buttons (stateless; "log as typed" reads the replied-to /ate); logged meals get `ml:<id>:<1/0/-1/undo>` rating buttons; cards show the made/rating line and a 🍴 Ate button in Sheep's chat |
| 🖨 Print on a card, `/recipe <id> print` (both chats) | `shared/recipe_print.py`: one Letter page (headless Chrome PDF) to CUPS `lp` on `RECIPE_PRINTER` / the CUPS default / the only printer (Brother HL-L2390DW). A second tap within 20 s doesn't print again. Tests can never print: `conftest._never_print` stubs recipe_print's subprocess |
| `/recipes [words]` | FTS5 search plus filters: course words (dinner = main), `quick`, `<500cal`, tags, cuisine |
| a reply to a `🍽 #id` card | revises that recipe (re-structure + recompute); intercepted before feedback, questions and the privacy router |
| a photo, document (PDF, Word, image) or album, from either chat | food agent Phase 2: read as one recipe (text locally, else Sonnet vision), caption as a hint, originals kept; as a reply to a `🍽 #id` card it adds a page. Albums are buffered by `media_group_id` in `poll_forever` and read 2.5 s after the last part |
| a photo captioned `label …`/`inv …`, or the reply to the `/label` prompt (Sheep only) | inventory label capture (`shared/inventory.py`, 2026-10-25): Sonnet vision (`label_vision`) reads item/make/model/serial/made/details; 🏷 card with [📚 File N to KB] (`l:file`) / [✕ Discard] (`l:<id>:discard`); photos kept in `~/data/<env>/labels/<id>/`; queued rows (`inventory_labels`) go out as one `kb_changes` proposal per file (author `label capture`, `allow_long_numbers=True` so serials pass) to `household/appliances.md` or `household/inventory.md`; a file with a label proposal still pending is skipped. `/label file` does the same as the button. Checked before the recipe path; Moose's photos always go to recipes |
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
- Callback prefixes in use: `m:` `rs:` `c:` `mdl:` `r:` `p:` `q:` `k:` `ml:` `ma:` `l:`. callback_data must stay ≤ 64 bytes.

## Moose's chat (2026-10-21)
- `TELEGRAM_MOOSE_CHAT_ID` (agent-system.env) makes a second served chat; `person(chat_id)` → "sheep"/"moose".
  Unset = Sheep only, as before.
- Replies: `process()`/`process_callback()` run inside `telegram.replying_to(chat_id)`; only sends with
  `agent="telegram_bot"` follow it (`REPLY_AGENTS`), so budget alerts and every scheduled agent still go to Sheep.
  She gets no reports or notifications (Sheep, 2026-10-21).
- Her commands: `MOOSE_COMMANDS` (menu, bills, afford, recipes, recipe, help; plus /start, /ask) and
  `MOOSE_LAUNCHER`; others reply "isn't available here", and only `m:`, `rs:`, `r:` buttons work for her
  (recipe delete allowed). Questions are unrestricted in scope (health still answered locally).
- Not feedback: her reactions, 👍 messages and replies are not recorded; review questions stay Sheep's.
- Her runs use trigger `telegram:moose`: separate conversation history, `read_only=True` (no remember /
  follow-ups / predictions), a prompt that names her, `MOOSE_ANSWER_SYSTEM` for KB answers; reflection
  skips them and /status's sent count excludes her chat.

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
