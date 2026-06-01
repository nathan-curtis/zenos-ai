# ZenOS-AI Grocy ERP Plugin

**Version:** 4.44.0
**Script:** `zen_dojotools_inventory`
**Status:** Live (production)

---

## Overview

Zen DojoTools Inventory provides a deterministic, governed control layer between Home Assistant and Grocy ERP.

Two coordinated components:

* **zen\_dojotools\_inventory** — High-level intent router (69 operational modes + `help`)
* **zen\_dojotools\_grocy\_advanced** — Low-level REST dispatcher (CRUD + OpenAPI introspection, internal)

Grocy is treated as the canonical authority for:

* Products and stock entries
* Locations and location hierarchy
* Quantity units
* Recipes
* Shopping lists
* Tasks and chores
* Batteries

This integration enforces strict name resolution, ID determinism, and guarded write operations to prevent silent data corruption.

---

## Design Philosophy

This is not a chat interface.

It is a controlled command router.

### Operating Principles

* All writes are explicit
* All names resolve to IDs before execution
* Ambiguous matches are rejected
* No implicit creation, merging, or deletion
* No unit guessing
* No silent mutation

If Grocy would reject an operation, this layer stops it first.

---

## First-Time Setup

### 1. Add API Key

In `secrets.yaml`:

```
grocy_api_key: <your_api_key>
```

Grocy uses a bare key — no prefix.

### 2. Set Grocy Base URL

After HA loads, set `input_text.grocy_url` via the HA UI:

```
Settings → Helpers → grocy_url → Edit → set value to https://<your-grocy-host>
```

No trailing slash. **Must be HTTPS.** This helper has no `initial:` value by design — set it once and HA persists it. Do not add `initial:` to the helper definition; doing so resets the value on every HA reload.

If Grocy is behind a reverse proxy that redirects HTTP → HTTPS, HA follows the 301 and converts POST requests to GET per RFC behavior. All write operations fail silently on HTTP.

---

## Architecture

### 1. Inventory Helper (Primary Interface)

High-level intent interface. The helper resolves:

* Product names → product IDs
* Location names → location IDs
* Unit names → unit IDs
* Recipe names → recipe IDs
* Task/chore names → IDs

All resolution occurs before write execution. Multiple matches halt execution.

Call `mode=help` for inline reference documentation.

### 2. Grocy Advanced (REST Dispatcher)

Internal CRUD dispatcher. Not intended for direct use. The helper calls advanced.

Features: endpoint normalization, pagination, query search, path parameter injection, OpenAPI help surface, 405 coaching, strict payload enforcement.

---

## Supported Modes

### Stock — Read

| Mode | Description |
|------|-------------|
| `stock_check_item` | Current stock level, locations, expiry for a named product |
| `stock_where_is_item` | Find which Grocy location(s) a product is stocked in |
| `stock_entries_for_item` | Raw stock entry list for a product (amounts, locations, dates) |
| `stock_get_product_details` | Full product record including unit assignments and metadata |
| `stock_overview` | Paginated product stock overview |
| `stock_list_volatile` | Products overdue, expiring, or near min-stock |
| `stock_list_by_location` | All products stocked at a specific location |
| `stock_area_summary` | All Grocy containers in a HA area with item counts, anchor, hazmat safety entries |
| `stock_area_inventory` | Full stock contents for all containers in a HA area |

### Stock — Write

| Mode | Description |
|------|-------------|
| `stock_add_purchase` | Add stock for an existing product at a location |
| `stock_buy_product` | Purchase flow: resolves or creates product, then stocks it |
| `stock_add_by_barcode` | Add stock by barcode scan |
| `stock_consume` | Consume (remove) stock for a product |
| `stock_consume_by_barcode` | Consume by barcode scan |
| `stock_transfer_location` | Move stock from one Grocy location to another |
| `stock_open_item` | Mark a product as opened (sets open date) |
| `stock_inventory_adjust` | Set a product's stock to an explicit amount (inventory correction) |
| `stock_entry_update` | Partial update to a specific stock entry (read-modify-write safety) |
| `stock_undo_booking` | Undo the most recent stock transaction for a product |
| `stock_register_asset` | One-call physical asset registration. If `hazmat_class` provided, logs lean RM safety[] entry. |

### Catalog

| Mode | Description |
|------|-------------|
| `catalog_list_products` | Paginated product list |
| `catalog_find_product` | Find a product by name (exact or partial) |
| `catalog_find_by_barcode` | Look up a product by barcode |
| `get_product_meta` | Fetch full product metadata record |
| `update_product_meta` | Full read-modify-write product metadata update. GET product → merge changes → PUT full object. Supports `unit`/`unit_id` (sets all three qu_id fields), `amount` (min_stock), `shopping_location_id`, `to_location_id` (default consume location), `product_group_id`. Strips null `qu_id_purchase`/`qu_id_price` before merge to prevent Grocy resolving null→system default unit. Sends `description` only when note/brand/sku are provided. |
| `rename_product` | Rename a product (requires new name to not conflict) |
| `set_default_location_for_product` | Set the product's default stock location |
| `products_merge` | Merge two products (requires explicit keep/remove IDs) |

### Locations

| Mode | Description |
|------|-------------|
| `locations_list` | All Grocy locations with metadata |
| `locations_find` | Find location by name or ID |
| `locations_get_by_area` | All Grocy locations tagged to a HA area |
| `locations_add` | Create a new location. Dupe guard: CI substring match blocks creation; `confirm_action: true` bypasses. |
| `locations_metadata_set` | Write userfield metadata to a location (area binding, subclass, parent ID, priority) |
| `locations_rename` | Rename a location |
| `locations_update_description` | Update a location's description field |
| `locations_delete` | Delete a location (evicts from PC sync cache) |
| `locations_delete_safe` | Delete only if location is empty; returns error if stocked items exist |
| `locations_reparent_bulk` | Move a set of child locations to a new parent in one operation |
| `locations_audit` | Scan all locations for metadata gaps, orphaned children, area mismatches |
| `locations_hierarchy_rebuild` | One-shot rebuild of parent-child hierarchy from ground truth. `dry_run: true` returns plan. Idempotent. |

#### Location Metadata Fields

Stored in Grocy userfields. Bind locations to HA topology:

| Field | Purpose |
|-------|---------|
| `grocy_parent_location_id` | Parent container ID (PC sync) |
| `homeassistant_area` | HA area_id slug this location belongs to |
| `grocy_location_subclass` | Semantic type: `anchor`, `container`, `zone`, `virtual` |
| `placement_priority` | Sort weight within area |

These fields support `stock_area_summary` and cross-system search indexing.

#### PC Sync (Parent-Child Reconciliation)

The integration maintains a parent-child map derived from ground truth on each rebuild. Design: derive from canonical state — never read-modify-write accumulate. Delete path: pre-fetch snapshot of parent, post-delete evicts child from cache.

### Shopping

| Mode | Description |
|------|-------------|
| `shopping_add_product` | Add a named product to the active shopping list |
| `shopping_remove_product` | Remove a product from the shopping list |
| `shopping_clear_list` | Clear all items from the active shopping list |
| `shopping_add_missing` | Auto-add all products below minimum stock |
| `shopping_add_overdue` | Add all products with overdue stock bookings |
| `shopping_add_expired` | Add all products with expired stock |

### Recipes

| Mode | Description |
|------|-------------|
| `recipes_list` | All recipes with names and IDs |
| `recipes_fulfillment` | Check if ingredients are stocked for a recipe |
| `recipe_check_fulfillment` | Detailed per-ingredient fulfillment check |
| `recipes_add_to_shopping` | Add missing recipe ingredients to shopping list |
| `recipes_consume` | Consume ingredients for a recipe from stock |
| `recipes_copy` | Duplicate a recipe under a new name |

### Chores

| Mode | Description |
|------|-------------|
| `chores_list` | All chores with due dates and assignees (paginated, limit/offset) |
| `chores_find` | Find a chore by name (searches up to 500 rows — no capping) |
| `chores_execute` | Mark a chore complete |
| `chores_undo` | Undo the most recent chore execution (requires execution_id from chores_log) |
| `chores_add` | Create a new chore |
| `chores_edit` | Edit an existing chore by ID — strips `userfields` and `row_created_timestamp`, maps `amount` → `product_amount` as fallback |
| `chores_delete` | Delete a chore by ID |
| `chores_by_area` | Find chores linked to a HA area — dual discovery: (1) products stocked in area, (2) chores tagged with `homeassistant_area` userfield |

#### chores_by_area Setup

Requires a Grocy userfield on the Chores entity:
- Entity: Chores
- Name: `homeassistant_area`
- Caption: `HA Area ID`
- Type: Short text

`discovery` field in results: `'product'` (chore found via stocked product) or `'tagged'` (chore directly tagged with area_id).

### Tasks

| Mode | Description |
|------|-------------|
| `tasks_list` | All open tasks |
| `tasks_find` | Find a task by name |
| `tasks_complete` | Mark a task complete |
| `tasks_undo` | Undo a task completion |

### Units

| Mode | Description |
|------|-------------|
| `units_list` | All quantity units |
| `units_find` | Find a unit by name |
| `units_add` | Create a quantity unit. **Idempotent** — preflight GET checks for exact name match first. Returns `{status: already_exists, unit_id, unit}` if found; only POSTs if unit is absent. Prevents UNIQUE constraint failures from ghost/soft-deleted units. |
| `units_update` | Update a unit's name or note |

### Unit Conversions

| Mode | Description |
|------|-------------|
| `unit_conversions_add` | Create a unit conversion. `unit_id` = from unit, `to_unit_id` = to unit, `amount` = factor, `product_id` = optional product scope (omit for global). |
| `unit_conversions_list` | All unit conversions. Filter by `unit_id` or `product_id`. |
| `unit_conversions_delete` | Delete a conversion by `entry_id`. |

**Field semantics:** `unit_id` is the source unit (from_qu_id), `to_unit_id` is the destination unit (to_qu_id). The legacy `to_location_id` field is accepted as a fallback for backward compatibility but `to_unit_id` is preferred.

**Global vs product-specific:**
- Global (`product_id` omitted) = real physics only: lb↔oz, g↔kg. Never build global conversions around ghost or null units.
- Product-specific = container↔dose, serving size, packaging/concentration ratios.

### Product Groups

| Mode | Description |
|------|-------------|
| `product_groups_list` | All product groups (categories) |
| `product_groups_find` | Find a product group by name (case-insensitive match) |

### Batteries

| Mode | Description |
|------|-------------|
| `batteries_list` | All tracked batteries |
| `batteries_get` | Get a specific battery record |
| `batteries_charge` | Mark a battery as charged |

---

## Room Manager Integration

`stock_area_summary` is the RM `+grocy` / `+inventory` context slice target. Pass `homeassistant_area_id` to get:

* `locations[]` — Grocy containers in the area sorted by item count, with `is_anchor` and `subclass` flags
* `anchor_location_id` — from RM `spatial.grocy_location_id` (room's primary Grocy container)
* `hazmat_safety[]` — from RM `spatial.safety[]` (fire extinguishers, hazmat entries)
* `total_items`, `stocked_locations`, `total_locations`

`stock_register_asset` accepts `hazmat_class` and writes a lean entry to the RM safety[] drawer:
`{name, type, location_note, grocy_location_id}`.

---

## Asset Registration

`stock_register_asset` handles physical assets (tools, appliances, equipment) in a single call:

1. Resolves or creates the product
2. Stocks one unit at the specified location
3. Sets serial number, purchase date, and note if provided
4. If `hazmat_class` is provided: writes a lean safety entry to RM

Use this for anything you own but don't consume — it joins the spatial index immediately.

---

## Natural-Language Intent Map

Available via `mode=help`, the `big_asks` section maps common intents to modes:

| Intent | Mode |
|--------|------|
| New thing here (physical asset) | `stock_register_asset` |
| New consumable, first time | `stock_buy_product` |
| Got more / restocked | `stock_add_purchase` |
| Where is X? | `stock_where_is_item` |
| What's in the [room]? | `stock_area_summary` |
| We used / consumed X | `stock_consume` |
| Need to buy X / running low | `shopping_add_product` |
| What's expiring? | `stock_list_volatile` |

---

## Unit Governance

Quantity units are first-class governed objects.

Rules:

* Units must exist in Grocy before they can be assigned
* Exact name match preferred; partial fallback allowed
* Multiple matches halt execution
* No inferred pluralization
* No implicit conversion creation
* `units_add` is idempotent — safe to call even if the unit might already exist

---

## Safe Write Operations

Destructive actions are guarded:

* Product merges require explicit keep/remove IDs
* Transfers blocked for tare-weight products
* `stock_entry_update` uses read-modify-write safety (reads full entry, merges only provided fields, writes back)
* Location and product renames require new name
* Purchase requires location when creating product
* Ambiguous name matches block execution
* `locations_delete_safe` refuses if location has stock

---

## Pagination

Object endpoints support:

* `page` (default: 1)
* `per_page` (default: 25)

Offset computed automatically.

---

## Error Model

Failures return structured responses:

* `status`
* `message`
* `mode`
* `details`
* `coaching`

Errors include remediation guidance. Common causes: missing required fields, ambiguous name resolution, invalid unit, missing location, invalid merge request, unsupported Grocy behavior, endpoint misuse.

---

## Null-Unit Product Doctrine

When `qu_id_stock` is null, Grocy maps it to the system default unit on any save. This creates a global conversion that conflicts with real unit relationships and causes cascading data corruption.

`update_product_meta` detects this condition before hitting the API. If `unit_id` is supplied and the product's `qu_id_stock` is null, it returns a 5-step delete-and-recreate recipe instead of attempting a partial update. **There is no patch path — the product must be deleted and recreated.**

Correct creation order when stock and purchase units differ:
1. `catalog_add_product` with `unit_id=<stock_unit>` — sets stock = purchase, no conversion needed
2. `update_product_meta` with `purchase_unit_id=<purchase_unit>` + any remaining metadata

---

## Design Notes

* The `mode` field replaced the old `case` field in v4.11. `mode=help` replaces the old `run=help` pattern.
* `zen_dojotools_inventory` is the STT-safe rename from `zen_dojotools_grocy_helper` (2026-05-15). All cross-tool references use the new name.
* `zen_dojotools_grocy_advanced` is internal — not MCP-exposed. Do not call it directly unless you need raw REST access.
* `unit_conversions_delete` uses an empty-dict payload in the broker's DELETE branch. Any future DELETE-method mode must be added to the same branch or it will silently fail with a `requires_payload` block.
* 70 operational modes + `help` (71 enum entries total).

---

## Final Note

Strict by design.

This module prevents silent inventory corruption, ambiguous merges, unit drift, and unintended destructive writes.

If it blocks you, it is protecting the system.
