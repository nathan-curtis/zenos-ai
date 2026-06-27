# ZenOS-AI Mealie / Kitchen Plugin

**Version:** 5.8.0
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
See also: [kitchen_sync](kitchen_sync.md), [Grocy / Inventory](grocy.md).
