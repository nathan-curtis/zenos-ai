# ZenOS-AI Mealie / Kitchen Plugin

**Version:** 5.22.0
**Package:** `packages/zenos_ai/plugins/mealie/mealie.yaml`
**Primary script:** `zen_dojotools_kitchen`
**Internal REST dispatcher:** `zen_sutra_mealie`
**Companion:** [kitchen_sync](kitchen_sync.md) — Mealie↔Grocy food sync engine

---

## Overview

The Kitchen plugin is the authoritative recipe, meal plan, and food catalog surface for ZenOS-AI. It treats Mealie as the household's service schedule and ingredient dictionary, while Grocy remains the authority for physical stock and purchase tracking.

In practical terms:

* Recipes, meal plans, shopping lists, foods, units, and organizers live in Mealie.
* ZenOS-AI tools call `zen_dojotools_kitchen` for all kitchen and recipe work.
* `zen_sutra_mealie` is the lower-level REST dispatcher. It should stay internal unless `zen_dojotools_kitchen` explicitly delegates there.
* The plugin maintains a bidirectional foreign-key link between Mealie foods and Grocy products via the `grocy_product_id` field in each food's `extras` dict. Sync is handled by the companion `kitchen_sync.yaml`.
* As of 5.22.0, the plugin also covers exec-chef operations beyond lookup/CRUD: fulfillment checking against real stock, recipe costing, weekly food-cost rollups, waste logging, event menu scaling, prep-time scheduling, and leftovers-to-stock booking — see [Cooking & Fulfillment](#cooking--fulfillment) and [Executive Chef](#executive-chef) below.

---

## Why the Kitchen/Inventory Split

Mealie and Grocy serve different roles and use deliberately different vocabulary:

| Layer | Authority | Example |
|-------|-----------|---------|
| Mealie (Kitchen) | Recipe catalog, meal schedule, ingredient identity | Tonight's dinner, what's on the weekly plan, what a recipe needs |
| Grocy (Inventory) | Physical stock, locations, shopping, purchase history | How many cans of tomatoes are in the pantry, when did we buy flour |
| Kitchen Sync | Bidirectional FK bridge | Mealie food "Olive Oil" ↔ Grocy product ID 47 |

For pantry stock, quantities, and purchase tracking, use `zen_dojotools_inventory`. For anything meal or recipe related, use `zen_dojotools_kitchen`.

---

## Modes

### Recipes

| Mode | Description |
|------|-------------|
| `recipes_list` | List all recipes. Supports `search`, `page`, `perPage`, `orderDirection`. |
| `recipes_find` | Search recipes by name or metadata. Requires `search`. |
| `recipes_get` | Get a full recipe by ID. Requires `recipe_id`. |
| `recipes_create` | Create a new recipe. Requires `payload` with at minimum `name`. Optional: `description`, `ingredients`, `instructions`, `tags`, `categories`, `tools`. |
| `recipes_update` | Update an existing recipe. Requires `recipe_id` and `payload`. |
| `recipes_delete` | Delete a recipe. Requires `recipe_id`. Destructive — irreversible. |
| `recipes_get_stats` | Get usage statistics for a recipe. Requires `recipe_id`. |
| `recipes_get_comments` | Get comments on a recipe. Requires `recipe_id`. |
| `recipes_add_comment` | Add a comment to a recipe. Requires `recipe_id` and `text`. |
| `recipes_random` | Get one random recipe suggestion. Answers "suggest something for dinner." |
| `recipes_create_from_url` | Import a recipe by scraping a URL (Mealie's own scraper). Requires `url`. Optional `include_tags`. Returns the fully-populated scraped recipe (Mealie's own endpoint only returns a slug — this case follows up with a GET so the caller gets the whole recipe in one call). |
| `recipes_parse_ingredients` | Runs Mealie's ingredient parser (`nlp`/`brute`/`openai`) over any freetext-only ingredient lines (quantity 0.0, unit null, food null — common on URL-scraped recipes) and writes back structured quantity/unit/food. Already-structured ingredients are left untouched. Requires `recipe_id` or `recipe`. Run this before `recipe_fulfillment` if a scraped recipe's ingredients aren't checking against stock. |
| `recipes_scale` | Client-side proportional recipe scaling (Mealie has no server-side scale API — nutrition doesn't scale even in Mealie's own UI). Preview-only by default; `apply_scale=true` persists via the safe GET→merge→PUT path. Field is `apply_scale`, not `apply` (collides with a Jinja extension name). |

### Cookbooks

| Mode | Description |
|------|-------------|
| `cookbooks_list` | List all cookbooks. |
| `cookbooks_get` | Get a cookbook by `item_id`. |
| `cookbooks_create` | Create a cookbook. Requires `payload` with `name`. |
| `cookbooks_update` | Update a cookbook. Requires `item_id` and `payload`. |
| `cookbooks_delete` | Delete a cookbook. Requires `item_id`. Destructive. |

### Cooking & Fulfillment

The bridge between "what's in a recipe" and "what's actually in the pantry." All fulfillment cases check real Grocy stock via `zen_dojotools_inventory` — none of them guess.

| Mode | Description |
|------|-------------|
| `recipe_fulfillment` | "Can I make this recipe right now?" Requires `recipe_id` or `recipe`. Checks every FK-linked ingredient against stock in one `stock_overview` call. Buckets: `fulfilled`, `insufficient`, `unresolvable` (no food/FK), `check_manually` (unit compatibility can't be verified from `stock_overview` alone — never guessed). Distinct from Grocy's own disconnected recipe engine, which never sees Mealie recipes. |
| `recipe_consume` | Depletes Grocy stock for a cooked recipe in one call. Resolves a safe consume amount per ingredient from real Grocy unit-conversion data (exact unit-id match, bare count, or a product-specific conversion entry) — never guessed math. Anything without a safe conversion path goes to `needs_manual_check` with recipe quantity shown next to real on-hand stock. Stamps `lastMade`. |
| `recipe_plan` | Checks `recipe_fulfillment`; if not fulfillable, auto-adds the missing items via `shopping_list_generate_from_recipe`. Requires `shopping_list_id`/`shopping_list` (no invented default list). Returns `shopping_action`: `not_needed`/`added`/`skipped_no_list`/`failed`. |
| `cookbook_fulfillment` | Resolves a cookbook's `queryFilterString` into its recipe list, then runs `recipe_fulfillment` across up to 2 recipes (capped low — this fans out real per-recipe calls). |
| `recipe_cost` | Household-cost estimate for one recipe. Self-calls `recipe_fulfillment` for resolved quantities, prices each ingredient from Grocy's stock-entry price history, sums to `total_cost` + `cost_per_serving`. Ingredients with no price history are excluded and counted in `unresolved_count` — `total_cost` is stated as a lower bound whenever that's nonzero. Grocy-only; no Firefly/COGS transaction lookup (too heavy per-recipe). |
| `recipe_plan` / `stock_expiry_suggestions` | `stock_expiry_suggestions` cross-references Grocy's near-expiry stock against every recipe's ingredient FKs to answer "what's expiring soon and what could use it up." Read-only. Optional `area` scopes to one HA area. Can be slow on large recipe libraries (cost scales with recipe count) — noted in the response itself. |
| `mealplan_suggest` | Combines `mealplan_week` (find empty slots), `stock_expiry_suggestions`, and `recipe_fulfillment` (capped at 3 candidates) into one suggestion-only case — never schedules anything itself. Optional `area` triggers guest/household allergen flagging via `zen_dojotools_room_manager mode=room_occupant_prefs` (falls back to a room's static household occupant when no guest is staying there), matched against ingredient names with a word-boundary-safe synonym taxonomy (e.g. dairy also catches milk/cream/butter/cheese) — a heuristic, not a certified allergen check, and never silently drops a suggestion. |

### Meal Plans

| Mode | Description |
|------|-------------|
| `mealplan_today` | Today's meal plan grouped by entry type (breakfast/lunch/dinner/snack). Answers "what's for dinner tonight?" |
| `mealplan_week` | This week's plan (Mon–Sun) grouped by date then entry type. Answers "what are we eating this week?" |
| `mealplan_list` | List meal plan entries. Optional date range via `start_date` / `end_date`. |
| `mealplan_get_item` | Get a specific meal plan entry by `item_id`. |
| `mealplan_add_recipe` | Schedule a recipe in the plan. Requires `date` and `entry_type`. Optional: `recipe_id`, `title`, `text`. |
| `mealplan_update_recipe` | Update an existing meal plan entry. Requires `item_id`, `date`, `entry_type`. |
| `mealplan_delete_item` | Delete a meal plan entry. Requires `item_id`. Destructive. |

### Shopping Lists

| Mode | Description |
|------|-------------|
| `shopping_lists_overview` | All Mealie shopping lists with name, linked Grocy list ID, and item counts. Cabinet-index view of the shopping system. |
| `shopping_list_detail` | Items in a specific list with per-row sync status (`ready/pending/note_only/checked`). Requires `item` (list name) or `shopping_list_id`. |
| `shopping_list_get` | List all shopping lists (raw API). |
| `shopping_list_get_items` | Get all items in a list. Requires `shopping_list_id`. |
| `shopping_list_add_item` | Add an item to a shopping list. Requires `item`. Optional: `quantity`, `unit`. |
| `shopping_list_remove_item` | Remove an item by `item_id`. Destructive. |
| `shopping_list_clear_completed` | Remove all completed items from shopping lists. Bulk destructive. |
| `shopping_list_generate_from_recipe` | Generate shopping list items from a recipe. Requires `recipe_id`. |
| `shopping_list_check_item` | Toggle a shopping list item's checked state (Mealie's own checkbox — never exposed by any tool before this). Mealie-only, no Grocy dependency. |
| `receipt_to_stock` | One call: bootstraps FKs via `sync_now`, sums a checked shopping list into `bulk_stock_add`, optional `bulk_product_ref_add` to a linked Paperless receipt doc. Supports `dry_run` (returns a `bom_preview` — exact products/amounts/prices that *would* be booked — without writing anything) and optional per-item `prices` (`{grocy_product_id: price}`, human-confirmed only, never OCR-parsed). Always dry-run this before applying against an already-processed list — re-running against checked items you've already booked will duplicate stock. |
| `shopping_list_link` | Link a Mealie shopping list to a Grocy shopping list by ID. Requires `item` or `shopping_list_id` and `grocy_list_id`. Use `zen_dojotools_inventory mode=shopping_lists_list` to find Grocy list IDs. |
| `shopping_list_sync` | Sync ready items (food-linked, `grocy_product_id` set, unchecked) from a Mealie list to the linked Grocy list. Default `sync_mode=preview`. Requires list to be linked first. |
| `all_lists_sync` | Sync all linked Mealie shopping lists to Grocy in one call. Reads prefs from `household.kitchen_sync` cabinet key; writes defaults on first run. Default mode is `apply`. |

### Foods and Units

| Mode | Description |
|------|-------------|
| `foods_list` | List all foods in Mealie. |
| `foods_find` | Search foods by name. Requires `search`. |
| `food_get` | Get a food by `item_id`. |
| `food_add` | Create a new food. Requires `payload` with `name`. Optional: `description`, `foodGroup`, `label`. |
| `food_update` | Update a food. Requires `item_id` and `payload`. |
| `food_delete` | Delete a food. Requires `item_id`. Destructive. |
| `foods_tag` | Write cross-system link fields into a food's `extras` dict (primarily `grocy_product_id`). Requires `item_id` or `item` (name lookup) and `grocy_product_id`. |
| `foods_audit` | FK coverage health check across all Mealie foods. Reports total, linked (has `grocy_product_id`), unlinked (no FK), coverage %, and last sync summary. Run before `sync_now` to know what needs bootstrapping. |
| `units_list` | List all measurement units. |
| `units_find` | Search units by name. Requires `search`. |
| `unit_get` | Get a unit by `item_id`. |
| `unit_add` | Create a new unit. Requires `unit` (name). Optional: `unit_plural`, `unit_description`. |
| `unit_update` | Update a unit. Requires `item_id` and `unit`. |
| `unit_delete` | Delete a unit. Requires `item_id`. Destructive. |

### Organizers (Categories, Tags, Tools)

| Mode | Description |
|------|-------------|
| `organizers_list` | List all organizers of a given type. Requires `organizer_type` (`categories`, `tags`, or `tools`). |
| `organizers_get` | Get an organizer by `item_id` and `organizer_type`. |
| `organizers_get_by_slug` | Get an organizer by `slug` and `organizer_type`. |
| `organizers_create` | Create a new organizer. Requires `organizer_type` and `payload` with `name`. |
| `organizers_update` | Update an organizer. Requires `organizer_type`, `item_id`, and `payload`. |
| `organizers_delete` | Delete an organizer. Requires `organizer_type` and `item_id`. Destructive. |

### Sync and Notes Resolution

| Mode | Description |
|------|-------------|
| `sync_now` | Trigger Mealie→Grocy food catalog sync. Delegates to `zen_admintools_kitchen_sync`. Preview first (default), then apply with `sync_confirmation=true`. Use `foods_audit` first to see coverage. Set `skip_linked=false` to force re-evaluation of already-linked foods (FK repair). |
| `qty_sync` | Grocy→Mealie stock quantity sync. For each Mealie food with a `grocy_product_id` FK, fetches current Grocy stock and writes `grocy_stock_qty` and `grocy_stock_updated_at` into food extras. Grocy is authoritative — Mealie value is overwritten. Default mode `apply`. |
| `ambiguous_review` | Read the `kitchen_sync_ambiguous_queue` from the household cabinet — foods where `sync_now` found multiple Grocy candidates but no exact match. Returns `food_name`, `food_id`, and candidates list. Resolve by calling `foods_tag` (pick a candidate) or `zen_dojotools_inventory mode=products_merge` (collapse duplicates first). Use `sync_limit` to page. |
| `notes_resolve` | Resolve note-only shopping list items by linking them to Mealie foods and setting `grocy_product_id`. Single-item mode: requires `item` + `grocy_product_id`. Bulk mode: pass `payload` as a JSON list of `{name, grocy_product_id}` objects. Creates the food if it does not exist in Mealie. |
| `notes_promote` | Promote note-only list items to food-linked items after `notes_resolve`. Finds the matching Mealie food (must have `grocy_product_id`) and writes `foodId` back to the list item, making it ready for `shopping_list_sync`. Workflow: `notes_resolve` → `notes_promote` → `shopping_list_sync`. Requires `item` (list name) or `shopping_list_id`. |

---

### Executive Chef

The 5.22.0 gap-closure batch — proactive kitchen operations, not just recipe/stock lookups.

| Mode | Description |
|------|-------------|
| `kitchen_brief` | Weekly food-cost rollup: `recipe_cost` totals across the week's meal plan vs. Firefly's "Groceries" category spend. Dedupes to **one fetch per unique recipe and one Grocy lookup per unique ingredient across the whole week** (not per-meal-occurrence) — an earlier per-meal version re-queried the same ingredient every time it recurred and blew past a single call's timeout. `overall_status` is red/yellow/green (≥40%/≥30%/else); `gm_summary` is one deterministic sentence, not LLM-generated. `grocery_spend_total` can legitimately read `$0` if Firefly has no Groceries-category transactions in the window — that's a real data gap, not a bug. |
| `waste_log` | Logs a discard via Grocy's own `stock_consume` with `spoiled=true` — a real native Grocy field. Requires `item` or `grocy_product_id`, plus `quantity` or `fraction_remaining`. **Deliberately writes nothing else** — no cabinet drawer, no custom ledger (cabinet space is finite). `waste_reason`/`recipe_id` are accepted and echoed back for your own context but are never persisted; Grocy's `stock_log` has no field for either. |
| `waste_summary` | Reads Grocy's own `stock_log` back (`spoiled=1`, optionally scoped by `grocy_product_id`) — the read side of `waste_log`. Read-only, no separate storage. |
| `event_menu_create` | Scales a whole multi-dish menu to a guest count in one call. `payload` is a **comma-separated list** of recipe names/ids, not a JSON array (this install's `from_json` filter rejects top-level JSON arrays). Flags allergens per dish using the same `room_occupant_prefs` lookup `mealplan_suggest` uses. |
| `prep_schedule_set` | Tags a meal-plan entry's own free-text field with a `[PREP:N]` lead-time marker (N minutes) — no new storage, just Mealie's existing field. |
| `prep_brief` | Computes start-by times for today's meal-plan entries from their `[PREP:N]` markers. Also surfaced proactively in `zen_dojotools_taskmaster`'s `briefing` context block (`context.prep_schedule`), so it's push, not just pull. |
| `leftovers_to_stock` | Books a cooked dish's leftover quantity back into Grocy as real stock. Requires `recipe_id` or `recipe`, plus `quantity` (absolute servings) or `fraction_remaining` (e.g. `0.5` for "half left"). Resolves/creates a `"<Recipe Name> (leftovers)"` product; idempotent — repeat bookings of the same dish add to the same product rather than creating duplicates. |

**Meal-time anchoring:** breakfast/dinner windows used by `prep_brief` and `mealplan_suggest`-adjacent logic derive from the household's existing `scheduler_anchors` (`zen_morning_start`/`zen_evening_start` +1h) rather than a separate hardcoded time set — see [`zen_home_mode`](../scripts/zen_home_mode_readme.md) for the anchor mechanism. Lunch/snack use two standalone `input_datetime` helpers instead, since they're clock habits, not Home Mode state transitions.

---

## Key Fields

| Field | Description |
|-------|-------------|
| `case` | The operation to perform (required when `mode=run`). |
| `recipe` | Recipe name (for name-based searches). |
| `recipe_id` | Mealie recipe UUID. |
| `item` | General name argument (food name, shopping list name, etc.). |
| `item_id` | Mealie entity UUID. Bypasses name resolution. |
| `payload` | JSON body for create/update operations. |
| `search` | Free-text search string (for modes that support it). |
| `organizer_type` | One of `categories`, `tags`, `tools`. |
| `entry_type` | Meal plan slot: `breakfast`, `lunch`, `dinner`, `snack`. Default `dinner`. |
| `date` | ISO date (e.g. `2026-06-20`) for meal plan entries. |
| `shopping_list` | Exact shopping list name for name-based list resolution. |
| `shopping_list_id` | Explicit shopping list UUID (skips name resolution). |
| `grocy_product_id` | Grocy product integer ID for cross-system FK operations. |
| `grocy_list_id` | Grocy shopping list integer ID for `shopping_list_link`. |
| `sync_mode` | `preview` (dry run, default) or `apply` (write to Grocy). |
| `sync_confirmation` | Must be `true` to execute an apply sync run. |
| `sync_limit` | Max foods to process per `sync_now` run (1–100, default 10). |
| `include_notes` | Also sync unchecked note-only items via Grocy name dedup. Default `false`. |
| `page` / `perPage` | Pagination controls. `perPage` max 50, default 50. |
| `queryFilter` | Mealie's own filter DSL (e.g. `tags.slug="..."`, `cookbook.id="..."`), forwarded to `recipes_list`/`recipes_find`/`mealplan_list`. |
| `quantity` | Absolute amount for `leftovers_to_stock`/`waste_log` (servings or Grocy units, case-dependent). |
| `fraction_remaining` | Alternative to `quantity` for `leftovers_to_stock`/`waste_log` — e.g. `0.5` for "half left." |
| `waste_reason` | Free-text context for `waste_log`. Echoed back, never persisted (Grocy's `stock_log` has no field for it). |
| `area` | HA area, scopes `stock_expiry_suggestions` and triggers guest/household allergen flagging in `mealplan_suggest`. |
| `apply_scale` | Must be `true` to persist a `recipes_scale` result. Note: field name is `apply_scale`, not `apply`. |
| `dry_run` | For `receipt_to_stock` — when `true`, returns `bom_preview` without writing any stock/FK changes. |
| `prices` | For `receipt_to_stock` — optional `{grocy_product_id: price}` map, human-confirmed only, never OCR-parsed. |

---

## First-Time Setup

1. Add the Mealie long-lived token to `secrets.yaml`:

```yaml
mealie_bearer: Bearer eyJhbGci...
```

To get a long-lived token: log in to Mealie → Profile (top-right avatar) → API Tokens → Create Token. Prefix the result with `Bearer `.

2. Set `input_text.mealie_url` in Home Assistant (Settings → Helpers, or Developer Tools):

```text
http://<your-mealie-host>:<port>
```

No trailing slash. This persists across restarts.

3. Expose `zen_dojotools_kitchen` to the conversation agent. Keep `zen_sutra_mealie` internal.

4. Run `foods_audit` to see how many foods are already linked to Grocy, then `sync_now` with `sync_mode=preview` to preview what would be created before committing an apply run.

---

## Sync Workflow (Mealie → Grocy)

The recommended first-sync sequence:

```yaml
# 1. Check coverage
zen_dojotools_kitchen:
  mode: run
  case: foods_audit

# 2. Preview what sync_now would do
zen_dojotools_kitchen:
  mode: run
  case: sync_now
  sync_mode: preview
  sync_limit: 50

# 3. Apply
zen_dojotools_kitchen:
  mode: run
  case: sync_now
  sync_mode: apply
  sync_confirmation: true
  sync_limit: 50

# 4. Review ambiguous matches (multiple Grocy candidates, no exact match)
zen_dojotools_kitchen:
  mode: run
  case: ambiguous_review
```

The sync engine skips foods that already have a `grocy_product_id` FK by default (`skip_linked=true`). Set `skip_linked=false` to force re-evaluation for FK repair.

---

## Shopping List Sync Workflow

```yaml
# 1. Link a Mealie list to a Grocy list (one-time)
zen_dojotools_kitchen:
  mode: run
  case: shopping_list_link
  item: "HEB"
  grocy_list_id: 3

# 2. Preview the sync
zen_dojotools_kitchen:
  mode: run
  case: shopping_list_sync
  item: "HEB"
  sync_mode: preview

# 3. Apply
zen_dojotools_kitchen:
  mode: run
  case: shopping_list_sync
  item: "HEB"
  sync_mode: apply
```

Items that are note-only (no food entity) require `notes_resolve` → `notes_promote` before they can sync. `all_lists_sync` handles all linked lists in one call.

---

## Source Notes

Derived from `packages/zenos_ai/plugins/mealie/mealie.yaml`.
See also: [kitchen_sync](kitchen_sync.md), [Grocy / Inventory](grocy.md), [Taskmaster](../kung_fu/taskmaster.md) (consumes `prep_brief` in its briefing context).
