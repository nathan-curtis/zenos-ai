# ZenOS-AI Kitchen Sync (Mealie↔Grocy Sync Engine)

**Version:** 5.6.0 (internal version at runtime; file header says 5.4.0)
**Package:** `packages/zenos_ai/plugins/mealie/kitchen_sync.yaml`
**Script:** `zen_admintools_kitchen_sync`
**Companion:** [mealie](mealie.md) — ships in the same `plugins/mealie/` folder

---

## Overview

`kitchen_sync.yaml` is the Mealie↔Grocy food catalog sync engine. It is **not** exposed to conversational agents or MCP surfaces directly. All LLM and MCP access goes through `zen_dojotools_kitchen` (`mealie.yaml`), which delegates to this script via `case=sync_now`.

The two files are tightly coupled. Changes to food lists, `foods_tag` behavior, or the `kitchen_sync` cabinet key affect both. Do not move or rename either file without updating the other.

---

## What It Does

Kitchen Sync maintains a bidirectional foreign-key link between Mealie foods and Grocy products:

* **Mealie side:** `extras.grocy_product_id` (integer) written into each Mealie food object.
* **Grocy side:** `userfields.products.mealie_ingredient_id` (Mealie food UUID) written onto the Grocy product.

This FK bridge is what makes shopping list sync (`shopping_list_sync` and `all_lists_sync`) work — only food items with a `grocy_product_id` FK are considered "ready" for sync.

---

## Sync Direction

Currently only `mealie_to_grocy` is implemented. The `grocy_to_mealie` direction is defined in the field schema but is not executed.

---

## How the Sync Works

On each run, the engine:

1. Paginates through all Mealie foods (up to `max_pages` pages at `per_page` foods per page).
2. For each food, checks whether it already has a `grocy_product_id` FK set. If `skip_linked=true` (default), those are skipped — so `limit=N` always processes N **unlinked** foods.
3. For each unlinked food, calls `zen_dojotools_inventory mode=catalog_find_product` with the food name.
4. Match resolution:
   - **Exact match:** both FKs are written immediately (Mealie `extras.grocy_product_id` + Grocy `userfields.mealie_ingredient_id`).
   - **Ambiguous (multiple candidates, no exact match):** food is added to `kitchen_sync_ambiguous_queue` in the household cabinet. Requires manual resolution via `case=ambiguous_review` + `foods_tag`.
   - **No match:** in `preview` mode, food is listed under `would_create`. In `apply` mode, a new Grocy product is created via `catalog_add_product` and both FKs are written.
5. Writes the ambiguous queue to the household cabinet (`kitchen_sync_ambiguous_queue`) at the end of every run.

---

## Fields

| Field | Default | Description |
|-------|---------|-------------|
| `direction` | `mealie_to_grocy` | Sync direction. Only `mealie_to_grocy` is active. |
| `mode` | `preview` | `preview` = dry run. `apply` = write to Grocy. |
| `limit` | `10` | Max foods to process per run (1–100). Applies after `skip_linked` filtering. |
| `confirmation` | `false` | Must be `true` to execute an `apply` run. |
| `default_location` | `Pantry` | Default Grocy storage location for newly created products. |
| `default_unit` | `each` | Default Grocy unit for newly created products. |
| `per_page` | `50` | Foods fetched per page from Mealie. |
| `max_pages` | `50` | Maximum pages to paginate (caps the total foods examined). |
| `skip_linked` | `true` | Skip foods that already have `grocy_product_id` set. Set `false` for FK repair runs. |

---

## Response Summary

The sync returns a summary dict after every run:

```json
{
  "status": "success",
  "tool": "Zen AdminTools Kitchen Sync",
  "version": "5.6.0",
  "summary": {
    "evaluated": 10,
    "matched_exact": 7,
    "matched_ambiguous": 1,
    "would_create": 2,
    "created": 0,
    "errors": 0
  }
}
```

In `preview` mode, `created` is always 0 and `would_create` shows what an apply run would create.

---

## Cabinet Keys

| Key | Description |
|-----|-------------|
| `kitchen_sync_ambiguous_queue` | List of `{food_name, food_id, candidates[]}` objects for foods that matched multiple Grocy products but no exact name. Written after every run. |
| `kitchen_sync_last_run` | Last run summary (written by `all_lists_sync`). |
| `kitchen_sync` | Prefs for `all_lists_sync` — `include_notes`, `default_location`, `default_unit`. Written with defaults on first run if missing. |

---

## Calling via Kitchen

Do not call `zen_admintools_kitchen_sync` directly from automations or agents. Use `zen_dojotools_kitchen`:

```yaml
# Preview
zen_dojotools_kitchen:
  mode: run
  case: sync_now
  sync_mode: preview
  sync_limit: 50

# Apply
zen_dojotools_kitchen:
  mode: run
  case: sync_now
  sync_mode: apply
  sync_confirmation: true
  sync_limit: 50

# FK repair (re-evaluate already-linked foods)
zen_dojotools_kitchen:
  mode: run
  case: sync_now
  sync_mode: apply
  sync_confirmation: true
  sync_limit: 100
  # pass skip_linked: false via sync_default_* or via direct call to kitchen_sync
```

---

## Troubleshooting

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| `apply` run exits immediately with no output | `confirmation` not set to `true` | Pass `sync_confirmation: true` via Kitchen or `confirmation: true` direct |
| Foods stuck in `ambiguous_queue` | Multiple Grocy products match the food name | Use `case=ambiguous_review` to inspect, then `foods_tag` to pin the correct Grocy ID |
| New products created with wrong unit | `default_unit` not set | Pass correct unit via `sync_default_unit` on the Kitchen call |
| `grocy_product_id` set but `shopping_list_sync` still shows `pending` | Food exists in Mealie but list item has no `foodId` | Run `notes_promote` on the list to link list items to their Mealie foods |

---

## Source Notes

Derived from `packages/zenos_ai/plugins/mealie/kitchen_sync.yaml`.
See also: [mealie](mealie.md), [Grocy / Inventory](grocy.md).
