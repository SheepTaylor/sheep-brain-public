# Food Agent — recipes, meal log, recommendations

Status: approved 2026-10-20; Phases 0–1 live 2026-10-20, Phase 2 live 2026-10-21 (a09a8c7); design agreed with Antigravity (2 rounds, see
Review); not built. No personal details here: `knowledge/` is published. Recipes and the meal
log live in the agent DB, not in `life/` (agents never write to `life/`).

## Goal

1. Send a recipe to the Telegram bot as pasted text, a link, a photo of a cookbook page or card, or
   a PDF. It comes back as clean text with **per-serving nutrition**, a **course** and tags, and
   it is stored, indexed and searchable.
2. Keep a **log of meals** eaten, with an optional rating.
3. **Recommend** dishes and meals (a main plus a side) from that history.

## Constraints that shape the design

1. **Nutrition is computed, not guessed.** A model asked "how many calories per serving" is off by
   20–30% and gives a different number each time. The model's only job is to read the ingredient
   lines and match each one to a reference food. The numbers come from USDA FoodData Central
   (public domain) in SQLite and are summed in Python. Every recipe records its coverage ("12 of
   14 ingredients"), so a partial figure is never shown as exact.
2. **The bot is text-only today.** `process()` returns early when `msg["text"]` is missing, which
   is the case for photos and documents (captions are in `msg["caption"]`). `process_callback()`
   knows only `p:` and `q:` buttons.
3. **No local vision.** Gemma 4 runs without a vision projector on a CPU-only box. Photos go to
   Claude. Recipes are not private data.
4. **`shared/llm.py` is the only door to models.** `ask_claude()` is text-only. Vision needs a new
   `ask_vision()` that runs through the same `govern()` cap and `record_usage()`.
5. **Privacy routing stays as it is.** `SENSITIVE` sends any message mentioning words like
   "cholesterol" or "allergy" to the local path. Recipe ingest therefore uses explicit commands, and
   so does the deterministic search, so neither depends on that router. A plain-language food question that trips the
   router just gets the local answer. That miss is accepted: the privacy rule wins.
6. **Message budget.** Replies to Sheep's own messages don't count against the 4/day proactive
   budget. Nothing in this plan is proactive by default.

## Architecture

```
Telegram: /recipe <text|url> · photo / album / PDF · /recipes · /ate · /meals · /suggest
        │
agents/telegram_bot.py   photo/document (+caption, albums grouped by media_group_id)
        │                replies to a recipe card are intercepted before feedback/privacy_route
        │                buttons r:<id>:del|keep, m:<meal>:<rating>
        ▼
shared/recipe_ingest.py  to_text():  schema.org Recipe JSON-LD (no model) → pdftotext
        │                            → llm.ask_vision() for photos and scanned PDFs
        │                structure(): model → {title, servings, times, course, cuisine, tags,
        │                             ingredients[{raw, qty, unit, item, prep}], steps}
        ▼
shared/nutrition.py      match():  FTS5 on `item` → top 8 USDA foods → model picks fdc_id + grams
        │                per_serving(): pure Python, coverage %, no model
        ▼
SQLite (migration 015)   recipes, recipe_ingredients, ingredient_match, recipes_fts, meals,
                         usda_food (nutrients as columns) + usda_food_fts, usda_portion
        ▲
shared/food.py           search, meal log, suggestions (SQL ranking, deterministic reasons)
shared/agent_tools.py    search_recipes, get_recipe, meal_history, suggest_meals (read-only)
```

### Ingest

| Input | How it arrives | To text | Model |
|---|---|---|---|
| Pasted text | `/recipe <text>` | as is | — |
| Link | `/recipe <url>` | the page's `schema.org/Recipe` JSON-LD (most recipe sites carry it), else the stripped page text | none for JSON-LD |
| Shared link (Android share → Telegram) | a message that is only a URL, or a page title + URL | as Link | as Link |
| PDF | sent as a document | `pdftotext`; with no text layer, `pdftoppm` → treat as photos | — |
| Photo(s) | sent as a photo, or as an image file (.jpg/.jpeg/.png/.heic document); an album is one recipe | `ask_vision`, transcribe verbatim | Sonnet 5.5 |

A photo sent as a reply to a recipe card appends a page, and the card is re-structured.

**Android share sheet.** Share → Telegram → the bot chat works from any app. Images and PDFs need no
command. A bare shared link is caught before `privacy_route()` (the message is only a URL, or a
one-line title plus a URL) and gets [Save recipe] / [Ignore] buttons, because a link alone isn't
necessarily a recipe. Shared text from a recipe app (multi-line, with an "Ingredients" heading)
gets the same buttons. The buttons are needed because the bot can't add a `/recipe` prefix to a
share. The prompt replies to the shared message, so on a tap the bot reads the URL or text from
`callback_query.message.reply_to_message`, with no stored state (`telegram.send_one` gains
`reply_to_message_id`). HEIC files are converted to JPEG with an installed CLI tool before
vision, and the original is kept.

Then one structuring call (`recipe_structure`, Haiku 5.5) returns strict JSON. Course and tags
are checked against a fixed list in code; anything outside it is dropped, not invented.

- **Course** (one): breakfast, appetizer, soup, salad, main, side, bread, dessert, snack, drink, sauce.
- **Tags** (any): cuisine, main protein, method (slow cooker, grill, air fryer, one pot), diet
  (vegetarian, gluten free), time (≤30 min), occasion (holiday, game day).

The bot replies "Reading…", then the recipe card: title, servings, time, course, tags, nutrition
per serving, coverage, `#id`. It saves immediately. Corrections:
- `/recipe 12 servings 8` and `/recipe 12 course side`;
- anything else: reply to the card with corrected text, and it is re-structured and nutrition is recomputed;
- `/recipe 12 delete` asks first with [Delete] / [Keep] buttons (the deletion approval rule).

Duplicates (same URL, or the same title and ingredient set) get "Already saved as #12".

### Nutrition (`shared/nutrition.py`)

- **Reference data:** FoodData Central *Foundation* + *SR Legacy* CSVs (about 8,000 generic foods,
  with household portions such as "1 cup, chopped"), loaded by
  `python3 -m agents.food_agent load-usda <zip>`. The tracked nutrients are flattened into columns
  on `usda_food` (no nutrient EAV table), and portions go in `usda_portion`. That is a few MB in
  `agent.db`, written once. Restic dedups it, and it can be reloaded at any time. Branded foods are skipped.
- **Matching:** FTS5 runs on the cleaned `item` ("yellow onion"), not the raw line. One batched
  call (`nutrition_match`, Haiku 5.5) picks an `fdc_id` per line and converts to grams,
  preferring the USDA portion table. "Salt to taste", water and garnish count as zero and as
  matched. Matches are cached per normalized line (`ingredient_match`), so the same line always
  gets the same numbers, and the cost falls as the collection grows.
- **Arithmetic:** grams × nutrient per 100 g, summed, ÷ servings. The working is stored per
  ingredient, so `/recipe 12 why` shows it.
- **Per serving:** calories, protein, fat, saturated fat, carbs, fiber, sugar, sodium,
  cholesterol. Coverage under 85% by weight shows as "≈ partial: 2 ingredients unknown".

### Search and query

- `/recipes <words>`: FTS5 over title, ingredients, tags and text, plus filters parsed from the
  words (`dessert`, `quick`, `<500cal`, `vegetarian`). It returns up to 10 lines:
  `#12 Shrimp & grits · main · 35 min · 520 cal`.
- `/recipe 12` shows the full card, and `/recipe 12 scale 2` scales the quantities.
- Plain-language questions reach the conversational agent (Sonnet) through the existing privacy
  router, and the agent uses the new read-only tools `search_recipes`, `get_recipe`, `meal_history` and
  `suggest_meals`.

### Meal log

`meals(id, eaten_on, slot, recipe_id NULL, description, servings, rating, note, created_at)`

- `/ate 12` · `/ate chili dinner` (best title match; if two are close it asks with buttons) ·
  `/ate yesterday takeout pizza` (free text, no recipe) · `/ate 12 2 servings`.
- The reply has [👍 Again] [😐 OK] [👎 Not again] buttons. They are optional, and the latest rating per recipe wins.
- `/meals` shows the last 14 days. `/meals 12` shows when a recipe was last made and how often.
- There are no logging reminders (message budget). With a sparse log, suggestions lean on saved recipes and ratings.

### Suggestions (`/suggest`)

A SQL ranking, pytest-covered, with no model call:

1. Filter by course (dinner = main by default) plus optional `quick` and `<Ncal`.
2. Exclude recipes rated Not again or made in the last 14 days.
3. Order by rating (Again > unrated > OK), then the number of times made in 180 days, then days since last made.
4. One of the three slots goes to a saved recipe that was never made, when one fits.
5. **Meal:** for each main, a side logged with it on the same day before; otherwise the
   best-ranked side.
6. Each suggestion gets a reason built from the data: "rated Again · last made 5 weeks ago".

`/suggest` · `/suggest dessert` · `/suggest quick` · `/suggest week` (5 dinners, no protein
repeated on consecutive days).

## Model routing (`agent/tasks.toml`)

| Task | Model | Why | Est. cost |
|---|---|---|---|
| `recipe_vision` | sonnet | handwriting, cookbook layouts | ~$0.02/photo |
| `recipe_structure` | haiku (local fallback) | strict JSON from clean text | <$0.01 |
| `nutrition_match` | haiku | pick from 8 candidates, convert units; cached | <$0.01, falling |
| conversation (existing) | sonnet | plain-language food questions | as today |

A 100-recipe import is about $2–3. After that, steady use costs cents per month.

## Storage, backup, tests

- Migration `015_food.sql` creates the tables above. Originals go in `~/data/<env>/recipes/<id>/`, and that path is added
  to the restic `BACKUP_DIRS`.
- No new service or timer: ingest runs inline in the bot, and `load-usda` is a one-off CLI.
- pytest covers `nutrition.py` (unit conversion, coverage, sums), suggestion ranking (exclusions,
  explore slot, side pairing, protein spacing), JSON-LD parsing on saved HTML fixtures, the
  `/ate`, `/recipes` and `/recipe` parsers, the bot's photo, album, document, card-reply and callback paths,
  and `ask_vision` accounting. The tests run on a real scratch DB with a ~50-food USDA fixture.
- Evals (`tests/evals/`): 10 recipes with published per-serving labels. Calories must be within ±15%,
  and protein and sodium within ±20%. This set is the ratchet for any model swap.

## Phases

| # | Deliverable |
|---|---|
| 0 | Migration 015, `load-usda`, `nutrition.py` + match cache, CLI `food_agent nutrition "<ingredients>"`, evals (live 2026-10-20; see Phase 0 results) |
| 1 | `/recipe <text|url>` ingest, structuring + taxonomy, card, `/recipes`, `/recipe <id>` (+servings, course, scale, why, delete-with-confirm), card-reply edits, duplicates, share-sheet links/text (live 2026-10-20 with Phase 0; Antigravity code review: 5 bugs fixed, CONFIRMED) |
| 2 | Photos, albums and PDFs: bot media handling, `llm.ask_vision`, originals on disk, restic path; bulk import of the Drive collection (live 2026-10-21, a09a8c7; see Phase 2 results) |
| 3 | Meal log: `/ate`, rating buttons, `/meals`; agent tools for plain-language questions |
| 4 | `/suggest` (+ `quick`, `<Ncal`, course, `week`) with side pairing |

Branch `scratch/food` → staging → master per phase, as usual.

Later options (not in this plan): `/suggest new` (web search for recipes like the top-rated ones);
a weekly dinner idea in the heartbeat (proactive, budgeted); health-aware suggestions through a
whitelisted local-lane signal.

## Decisions (Sheep, 2026-10-20)

1. **Nutrition:** USDA reference data + computed totals.
2. **Whose meals:** Sheep only (no `who` column).
3. **Privacy:** recipes and the meal log are cloud-allowed. They are never joined with
   `health_facts` in a cloud call.
4. **Existing collection:** yes: Drive `archive/Recipes` (about 100 files; Sheep, 2026-10-21). It was copied
   locally with rclone, with `Icings/Family Tree/` left out, and bulk-imported. The originals stay in Drive.

## Review (Antigravity, 2026-10-20)

Round 1: AGREE WITH CHANGES. Adopted: `/recipe` required for text and link ingest; `ask_vision` in
`llm.py` under the governor; album grouping; caption and callback handling; FTS on the cleaned
item; nutrients flattened into columns; the micro-edit DSL replaced by card-reply edits; `/suggest new` and
learned affinities, season nudges, gap curves and the `food_why` model call cut in favor of SQL
ranking. Claude kept, and Antigravity accepted: the USDA data stays in `agent.db` (no second DB,
no exception to `get_db()`), side pairing stays ("dishes and meals"), and `SENSITIVE` is not
narrowed. Round 2: AGREE. Note for Phase 1/2: card replies must be intercepted before
`feedback.record_reply()` and `privacy_route()`.
Round 3 (share sheet, image files, Sheep's decisions): AGREE, with the stateless reply-to
button design and HEIC conversion adopted.

## Phase 0 results (2026-10-20)

- USDA: Foundation (2026-04) + SR Legacy (2018-04), 6,996 foods after skipping baby, fast, restaurant
  and prepared foods, plus 12,598 portions. The data adds about 2.6 MB to the DB, and loading takes 3 s.
  Foundation foods that report fat only as the NLEA total, or report no energy, get fallbacks.
- Matching takes two Haiku calls per recipe (name each line the USDA way, then pick from relaxed
  FTS5 candidates), with one retry on an unreadable reply. A first try that picked straight from
  plain FTS results scored 1/10. Portion-table grams were dropped because the model reads the portions better;
  stated weights are still converted in code.
- Eval against Allrecipes labels: 4–5 of 10 within tolerance (calories ±15%, protein and sodium
  ±20%). Sonnet scored no better at 8× the cost. The gaps that remain are mostly label conventions,
  checked by hand: Allrecipes counts ground meat after cooking losses (meatloaf: 357 cal raw against
  a 284 label), counts "salt to taste", and assumes regular beef where the recipe says lean. The
  ratchet is set at the baseline (4), not the 8 first planned.
- Run-to-run variance on a fresh match is up to ~35% on one nutrient (which broth, how big "2
  potatoes" is). The SDK refuses `temperature` for these models, so stability comes from the
  per-line cache: the first match of a line is kept.
- Cost: about $0.008 per 10-recipe eval run on Haiku.
- Ground meat browned and drained (Sheep's choice): the raw food becomes its USDA cooked crumbles,
  and the weight is scaled by protein. Eval 6/10 on scratch and 7/10 on candidate; ratchet 6.

## Blocked sites (2026-10-20, live dca3872)

Some recipe sites refuse automated reads (Cloudflare challenges, HTTP 403). Tried and rejected:
browser-like headers (still 403), headless Chrome (stuck on the challenge page), Anthropic's
server-side web fetch (url_not_accessible) and Internet Archive Save Page Now (523). Adopted, with Antigravity's agreement:
`fetch()` decodes gzip/deflate (NYT Cooking sends gzip unasked, which hid its recipe data) and zstd
(through the `zstd` CLI, for archive copies). When a site blocks the read, or its page has no recipe data,
the bot uses the newest Internet Archive snapshot (`id_` raw form) and the card says so. With no
snapshot, the reply asks for pasted text. Photos and PDFs (Phase 2) cover the rest: a screenshot,
or Chrome → Share → Print → Save as PDF.

## Phase 2 results (2026-10-21, live a09a8c7)

- **Files:** the file type is read from the contents (Drive files often lack an extension). Text comes from
  pdftotext, docx and oxps XML, and plain text. A long web printout keeps only its recipe pages and their
  neighbours. A file holding only a link is read as that link. Photos are scaled to a 2000 px long edge.
  Photos, scanned PDFs and pictures inside a .docx go to Sonnet vision, one call per recipe. A short PDF
  whose text layer doesn't read as a recipe (scanner OCR garbage) is read again as a scan.
- **Bot:** accepts photos, documents and albums (grouped by `media_group_id` and read 2.5 s after the
  last part). A caption is passed to the model as a hint. A photo sent as a reply to a card adds a page.
  Replies go back to the chat the photo came from (Moose's or Sheep's). Originals are kept in
  `~/data/<env>/recipes/<id>/`, which the nightly restic backup already covers (`/home/sheep/data`).
- **Bulk import** (`food_agent import DIR`): files whose names differ only by "page N" are grouped, and
  photos with timestamped names less than 2 minutes apart are paired (front and back of a card). A PDF
  with 4 or more recipe-like pages is a book and is read page by page. Family-tree folders and
  `Label-*.pdf` shipping labels are never sent to a model. Each result goes to `import-log.json`, so a
  re-run only retries failures.
- **Servings:** when a recipe doesn't say, the model estimates servings. The card marks the figure as
  estimated (migration 016), and with no estimate at all it shows whole-recipe totals.
- **Scratch trial:** 122 of 125 units saved for about $0.45. Misses: a 2015 Allrecipes print link that is
  gone with no archive copy, and a Facebook restaurant post (correctly not a recipe).
- **Antigravity reviews:** 5 findings fixed (add_pages dropping short text, uncapped prompts and an open-ended
  scan fallback, Drive zip reads and picture-only .docx, HEIC temp files, over-eager grouping), plus the
  run trigger after the merge with the menu/Moose work.
- **Live import (2026-10-21):** 121 saved, 1 duplicate, 2 not recipes, 1 failed; 124 recipes in live; 31
  servings estimated; $0.44; originals 53 MB in `~/data/live/recipes/`. Failed: a Mini Applesauce Muffins
  link (site blocks, no archive copy). Not recipes: a Facebook restaurant post, and "Laura's Aunt - Lillian
  Scroggs.pdf", which is a handwritten family letter. That letter was sent to Haiku (OCR text) and Sonnet
  (scan) before being rejected. Nothing was stored, but family content reached a cloud model; the skip
  list only knew folder names.
