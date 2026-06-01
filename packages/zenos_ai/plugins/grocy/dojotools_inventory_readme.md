# Zen DojoTools Inventory — v4.44.0

**File:** `packages/zenos_ai/plugins/grocy/grocy.yaml`
**Script:** `zen_dojotools_inventory`
**MCP-exposed — 70 modes**

Household inventory ERP layer. Wraps the Grocy REST API for stock, catalog, shopping, locations, chores, tasks, recipes, units, and batteries. Names resolve to IDs automatically — pass `item=coffee` instead of `product_id=42`.

---

## Quick Reference

| Intent | Mode | Key Fields |
|--------|------|-----------|
| Do we have X? | `stock_check_item` | `item` |
| Where is X? | `stock_where_is_item` | `item` |
| We bought X | `stock_buy_product` | `item`, `amount`, `location` |
| We used X | `stock_consume` | `item`, `amount` |
| Counted X, set total | `stock_inventory_adjust` | `item`, `amount` |
| Move X to another location | `stock_transfer_location` | `item`, `location`, `to_location` |
| What's in the pantry? | `stock_list_by_location` | `location` |
| What's in a HA area? | `stock_area_summary` | `homeassistant_area_id` |
| Full area room view | `stock_area_inventory` | `homeassistant_area_id` |
| Add to shopping list | `shopping_add_product` | `item` |
| Auto-fill from min stock | `shopping_add_missing` | — |
| Register new asset | `stock_register_asset` | `item`, `location` |
| Add to catalog (0 stock) | `catalog_add_product` | `item`, `location` |
| Log a chore done | `chores_execute` | `item` |
| What can I cook? | `recipes_fulfillment` | — |
| Full field reference | `help` | — |

---

## Name Resolution

All `item`, `location`, `to_location`, `store`, and `unit` fields resolve from name to Grocy ID automatically. Pass the explicit `product_id`, `location_id`, etc. to skip resolution and go direct.

`item` also accepts a numeric product ID or the word itself — the resolver handles it.

---

## Mode Groups

### Stock

| Mode | Intent |
|------|--------|
| `stock_check_item` | Stock level + where an item lives |
| `stock_where_is_item` | Locations where an item is stored |
| `stock_entries_for_item` | Detailed stock entries (amounts, BBD, entry IDs) |
| `stock_get_product_details` | Stock-context summary (amount, value, best_before) |
| `stock_overview` | All products currently in stock (paginated) |
| `stock_list_volatile` | Products due soon, overdue, or expired |
| `stock_list_by_location` | Everything stocked at a named location |
| `stock_area_summary` | Containers + item counts for a HA area; includes RM anchor + hazmat safety |
| `stock_area_inventory` | Full denormalized room view — all locations with product names and amounts; one call replaces N location + M product lookups |
| `stock_add_purchase` | Log purchase for an existing catalog product |
| `stock_buy_product` | Create product if missing, then purchase into inventory |
| `stock_consume` | Consume units from stock |
| `stock_open_item` | Mark a product as opened |
| `stock_inventory_adjust` | Reconcile inventory to a measured total |
| `stock_transfer_location` | Move item between locations |
| `stock_entry_update` | Edit a specific stock entry (BBD, location, price, note) |
| `stock_undo_booking` | Undo a stock booking by `booking_id` |
| `stock_register_asset` | 1-call asset registration: create product + stock 1 unit BBD=never + set meta. `hazmat_class` triggers lean RM safety log. |
| `stock_add_by_barcode` | Purchase into stock using a barcode |
| `stock_consume_by_barcode` | Consume from stock using a barcode |

### Catalog

| Mode | Intent |
|------|--------|
| `catalog_list_products` | List all products (paginated) |
| `catalog_find_product` | Find a product by name |
| `catalog_find_by_barcode` | Find a product by barcode |
| `catalog_add_product` | Create a 0-stock catalog entry for a product not yet owned |
| `catalog_delete_product` | Delete a product; stock guard — offer `product_id_keep` for merge or `confirm_action: true` to discard |
| `get_product_meta` | Retrieve canonical product metadata |
| `update_product_meta` | Update declarative product metadata (note, unit, min stock, shopping location, product group) |
| `rename_product` | Rename a product; ID and history unaffected |
| `set_default_location_for_product` | Set the default storage location for new purchases |
| `products_merge` | Merge two products; moves all stock + history; irreversible |

### Locations

| Mode | Intent |
|------|--------|
| `locations_list` | List all locations |
| `locations_find` | Find locations by name and/or parent anchor |
| `locations_get_by_area` | Reverse lookup — find the Grocy location anchored to a HA area_id |
| `locations_add` | Create a new location; dupe-guards similar names; `confirm_action: true` bypasses |
| `locations_rename` | Rename a location |
| `locations_update_description` | Update the description field of a location |
| `locations_metadata_set` | Set semantic metadata (subclass, area_id, parent, placement_priority) |
| `locations_delete` | Permanently delete a location (no stock guard) |
| `locations_delete_safe` | Delete after transferring stock to `orphan_to` location |
| `locations_reparent_bulk` | Bulk-set a new parent on multiple child locations |
| `locations_audit` | Scan all locations; report HA area mismatches and parent/child integrity issues |
| `locations_hierarchy_rebuild` | Rebuild `grocy_child_location_ids` for all parents from ground truth; `dry_run: true` to preview |

### Shopping

| Mode | Intent |
|------|--------|
| `shopping_add_product` | Add a specific product to the shopping list |
| `shopping_remove_product` | Remove a product from the shopping list |
| `shopping_add_missing` | Auto-add all items below min stock threshold |
| `shopping_add_overdue` | Add overdue products to the list |
| `shopping_add_expired` | Add expired products to the list |
| `shopping_clear_list` | Clear a shopping list; `done_only: true` to clear only ticked items |

### Chores

| Mode | Intent |
|------|--------|
| `chores_list` | List all chores |
| `chores_find` | Find chores by name |
| `chores_by_area` | Chores for a HA area — discovers via products stocked in area OR `homeassistant_area` userfield on the chore |
| `chores_add` | Create a chore; optionally link a product consumed on execution |
| `chores_edit` | Edit an existing chore (patch — unspecified fields preserved) |
| `chores_execute` | Log a chore as done |
| `chores_undo` | Undo a chore execution by `execution_id` |
| `chores_delete` | Delete a chore by name or `chore_id` |

### Tasks

| Mode | Intent |
|------|--------|
| `tasks_list` | List all tasks |
| `tasks_find` | Find tasks by name |
| `tasks_complete` | Complete a task |
| `tasks_undo` | Undo a completed task |

### Recipes

| Mode | Intent |
|------|--------|
| `recipes_list` | List all recipes (paginated) |
| `recipes_fulfillment` | What can be cooked with current stock |
| `recipe_check_fulfillment` | Can I make a specific recipe with current stock |
| `recipes_add_to_shopping` | Add missing recipe ingredients to the shopping list |
| `recipes_consume` | Mark a recipe as made — consumes ingredients from stock |
| `recipes_copy` | Duplicate a recipe |

### Units

| Mode | Intent |
|------|--------|
| `units_list` | List all quantity units |
| `units_find` | Find a unit by exact name |
| `units_add` | Create a unit; idempotent — returns existing if found |
| `units_update` | Update a unit (rename, note) |
| `unit_conversions_add` | Add a QU conversion (required before changing `qu_id_stock` on a product with stock) |
| `unit_conversions_list` | List conversions; filter by `unit_id` or `product_id` |
| `unit_conversions_delete` | Delete a conversion by `entry_id` |

### Product Groups

| Mode | Intent |
|------|--------|
| `product_groups_list` | List all product groups |
| `product_groups_find` | Find a product group by name (partial, case-insensitive) |

### Batteries

| Mode | Intent |
|------|--------|
| `batteries_list` | List all tracked rechargeable batteries |
| `batteries_get` | Get details for a specific battery |
| `batteries_charge` | Record a charge cycle |

---

## Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `mode` | select | Operation mode. Default: `help`. |
| `item` | text | Name resolved to Grocy ID. Also accepts numeric product ID. |
| `product_id` | number | Explicit product ID — skips name resolution. |
| `barcode` | text | EAN/UPC barcode for barcode-based operations. |
| `amount` | number | Quantity for stock math (buy, consume, transfer, adjust). Default: 1. |
| `unit` | text | Quantity unit name (exact Grocy match, e.g. `each`, `kg`, `l`). |
| `unit_id` | number | Explicit quantity unit ID. |
| `purchase_unit_id` | number | Separate purchase unit for `update_product_meta`. Requires a global QU conversion to stock unit. |
| `to_unit_id` | number | Destination unit ID for `unit_conversions_add`. |
| `price` | number | Optional unit price for purchase transactions. |
| `best_before_date` | date | BBD for stock entries. |
| `location` | text | Source or primary location name. |
| `location_id` | number | Explicit location ID. |
| `to_location` | text | Destination location name for transfers. |
| `to_location_id` | number | Explicit destination location ID. |
| `store` | text | Shopping location name (purchase operations). |
| `shopping_location_id` | number | Explicit shopping location ID. |
| `homeassistant_area_id` | text | HA area ID for area-scoped operations. |
| `location_subclass` | select | Semantic type: `suite`, `room`, `closet`, `workspace`, `container`, `furniture`, `shelf`, `drawer`, `cubby`, `storage`. |
| `parent_location` | text | Parent location name for hierarchy operations. |
| `parent_location_id` | number | Explicit parent location ID. |
| `child_location_ids` | text | JSON array of child IDs for `locations_reparent_bulk`. |
| `orphan_to` | number | Location ID to receive stock before `locations_delete_safe`. |
| `placement_priority` | number | Semantic priority for location resolution. Higher wins. |
| `new_name` | text | New name for rename operations. |
| `note` | text | Note, description, or `locations_update_description` text. |
| `brand` | text | Product brand (stored in Grocy description). |
| `sku` | text | Product SKU (stored in Grocy description). |
| `product_id_keep` | number | Canonical product ID to keep in `products_merge`. |
| `product_id_remove` | number | Duplicate product ID to merge and delete. |
| `product_group_id` | number | Product group ID for `update_product_meta`. |
| `recipe_id` | number | Explicit recipe ID. |
| `chore_id` | number | Explicit chore ID. |
| `task_id` | number | Explicit task ID. |
| `entry_id` | number | Stock entry ID for `stock_entry_update`; conversion row ID for `unit_conversions_delete`. |
| `execution_id` | number | Chore execution ID for `chores_undo`. |
| `booking_id` | number | Booking ID for `stock_undo_booking`. |
| `battery_id` | number | Grocy battery ID. |
| `period_days` | number | Chore recurrence in days. |
| `list_id` | number | Shopping list ID override. |
| `page` | number | Page number (1-based). Default: 1. |
| `per_page` | number | Items per page (1–250). Default: 25. |
| `done_only` | boolean | `shopping_clear_list` — clear only ticked items. Default: false. |
| `report_inventory` | boolean | Return post-purchase inventory summary after buying. Default: true. |
| `confirm_action` | boolean | Bypass dupe guard on `locations_add`; confirm protected delete operations. |
| `hazmat_class` | text | Hazmat major class (e.g. `flammable`, `corrosive`). Triggers RM safety log on `stock_register_asset`. |
| `dry_run` | boolean | Preview without writing (`locations_hierarchy_rebuild`). |
| `show_trace` | boolean | Include resolution details, IDs, and endpoint info in response. |

---

## Common Patterns

```yaml
# Check if we have something
mode: stock_check_item
item: coffee

# Log a purchase
mode: stock_buy_product
item: coffee
amount: 2
location: Pantry

# Consume what was used
mode: stock_consume
item: coffee
amount: 1

# Correct a count after a physical audit
mode: stock_inventory_adjust
item: paper towels
amount: 6

# See everything in a room
mode: stock_area_summary
homeassistant_area_id: kitchen

# Queue up the shopping run
mode: shopping_add_missing

# Register a new permanent asset
mode: stock_register_asset
item: Power Drill
location: Garage
brand: DeWalt
note: 18V cordless, model XR20V

# Log a maintenance chore done
mode: chores_execute
item: Clean litter box
```

---

## Dependencies

| Dependency | Purpose |
|-----------|---------|
| `script.zen_dojotools_filecabinet` | Cabinet reads for RM spatial data (area summary) |
| `script.zen_dojotools_room_manager` | RM anchor + hazmat safety in area operations |
| Grocy REST API | All stock, catalog, and ERP operations |
| Grocy `secrets.yaml` entries | API URL + token (read by `zen_dojotools_grocy`) |

---

## Notes

- **Name resolution:** Grocy product, location, unit, and chore names are matched case-insensitively. Ambiguous matches return an error listing candidates — pass `product_id` / `location_id` to disambiguate.
- **Paging:** `catalog_list_products` and `recipes_list` are paginated. Default `per_page: 25`. Increment `page` to walk the full set.
- **`update_product_meta` constraints:** Cannot change `unit_id` if `qu_id_stock` is null — delete and recreate instead. Setting `purchase_unit_id` requires a global QU conversion to the stock unit to exist first (`unit_conversions_add`).
- **`products_merge` is irreversible** — moves all stock, history, and references from `product_id_remove` into `product_id_keep`, then deletes the source.
- **`show_trace: true`** adds resolution detail, resolved IDs, and the final Grocy endpoint to every response. Useful for debugging name resolution failures.
