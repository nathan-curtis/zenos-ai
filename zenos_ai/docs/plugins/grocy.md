# ZenOS-AI Grocy Inventory Component

**Version:** 5.6.0  
**Package:** `packages/zenos_ai/plugins/grocy/grocy.yaml`  
**Primary script:** `zen_dojotools_inventory`  
**Internal REST dispatcher:** `zen_sutra_grocy`  
**KFC provider (sibling file):** `zen_sutra_logistics` — `packages/zenos_ai/plugins/grocy/sutra_logistics.yaml`, see [Logistics KFC Components](#logistics-kfc-components)

---

## Overview

Grocy is the governed inventory, location, shopping, and chore surface for ZenOS-AI. The plugin does not treat Grocy as a loose pantry lookup. It treats Grocy as a durable ERP-style system with explicit IDs, strict ambiguity checks, and controlled write paths.

In practical terms:

* Products, stock entries, locations, units, shopping lists, chores, recipes, tasks, and batteries live in Grocy.
* ZenOS-AI tools call `zen_dojotools_inventory` for normal work.
* `zen_sutra_grocy` is the lower-level REST dispatcher and should stay internal unless the inventory tool explicitly points there.
* AutoVac and SpaMaster use Grocy for consumable parts and supplies, but each stores its own local catalog pointer in its cabinet config.

---

## Why This Component Is Different

Most ZenOS tools own their domain state directly in cabinets. Grocy is different: it is both an external application and a shared physical inventory graph. ZenOS stores local intent and catalog bindings, while Grocy remains the authority for stock, locations, chores, shopping, and product IDs.

That split is intentional:

| Layer | Authority | Example |
|-------|-----------|---------|
| Component cabinet | Tool-specific catalog bindings | AutoVac part key `hepa_filter` maps to Grocy product ID + wear sensor |
| Grocy | Physical inventory truth | Current spare filters in stock, storage location, shopping list entry |
| Room Manager | Spatial context | Area anchor, safety notes, room context slices |
| Postman | Human acknowledgement | "Spare on hand, replace this part?" |

---

## Inventory Flow

```mermaid
flowchart LR
  subgraph Components["Domain Components"]
    AutoVac["AutoVac"]
    SpaMaster["SpaMaster"]
    RoomManager["Room Manager"]
  end

  subgraph LocalState["ZenOS Local State"]
    AVDrawer["household.autovac.grocy_catalog"]
    SpaConfig["household.spa_config.grocy_catalog"]
    Spatial["room_topology spatial data"]
  end

  subgraph GrocyLayer["Grocy Inventory Layer"]
    Inventory["zen_dojotools_inventory"]
    Advanced["zen_dojotools_grocy_advanced"]
    Grocy["Grocy API"]
  end

  subgraph HumanLoop["Human Loop"]
    Shopping["Shopping list"]
    Chores["Maintenance chores"]
    Postman["Postman notification"]
  end

  AutoVac --> AVDrawer
  SpaMaster --> SpaConfig
  RoomManager --> Spatial
  AVDrawer --> Inventory
  SpaConfig --> Inventory
  Spatial --> Inventory
  Inventory --> Advanced --> Grocy
  Inventory --> Shopping
  Inventory --> Chores
  AutoVac --> Postman
  SpaMaster --> Postman
  Postman --> AutoVac
  Postman --> SpaMaster
```

Read this as two truths meeting in the middle: the component knows what a part means inside its domain, and Grocy knows whether the part exists, where it is, and whether it needs to be purchased or logged as used.

---

## Core Inventory Modes

| Mode | Use |
|------|-----|
| `stock_check_item` | Current stock level + storage location for a product |
| `stock_where_is_item` | Find where a product is stored — returns identical data to `stock_check_item`; prefer `stock_check_item` to avoid a redundant call |
| `stock_entries_for_item` | Raw stock entries for a product |
| `stock_buy_product` | Create or resolve a product, then add stock |
| `stock_add_purchase` | Add purchased stock for an existing product |
| `stock_consume` | Remove stock after use or replacement. Optional `spoiled: true` marks the consumption as waste/spoilage in Grocy's native stock log instead of a normal consume (default `false`). |
| `waste_history` | Read-only: Grocy's own `spoiled=true` consume bookings (native waste log), optionally scoped to one `product_id`. `GET /objects/stock_log?query[]=spoiled=1` |
| `bulk_stock_add` | Add stock (purchase) for a BOM list of products, resolving names with exact+fuzzy fallback. Each `bom` entry may carry its own `price` (real per-item price, e.g. from a receipt) overriding the call-level `price` for that item only. Requires `confirm_action: true` unless `dry_run: true`. `dry_run` (default `false`) skips the writes and returns a `preview` status per item. |
| `bulk_stock_reconcile_recent` | Reverse a duplicate/erroneous `bulk_stock_add` booking. For each product in `bom`, sums stock entries created at/after `since_timestamp` and subtracts that from the product's current total via `stock_inventory_adjust` — deliberately not `stock_undo_booking` or `stock_consume` (see code comments for why). `dry_run` defaults `true`; requires `confirm_action: true` to actually write. |
| `shopping_add_product` | Add a specific product to the shopping list. Returns `{product_id, product_name, amount, list_id, action}` |
| `shopping_remove_product` | Remove a product from the shopping list. Returns `{product_id, product_name, amount, list_id, action}` |
| `stock_area_summary` | Area-level container and stock count rollup |
| `stock_area_volatile` | Volatile items (overdue/due_soon/expiring) scoped to a HA area |
| `stock_area_inventory` | Full denormalized room inventory: locations, products, amounts |
| `room_brief` | Chores + stock summary for a HA area in one call. Chore discovery uses three paths: (1) chore area-tagged via `homeassistant_area` userfield, (2) chore product stocked at area location, (3) product `ha_labels` contains area slug. |
| `chores_by_area` | Maintenance chores connected to an HA area |
| `chores_execute` | Mark a chore complete |
| `chores_tag` | Set `homeassistant_area` and/or `entity_id` userfields on an existing chore — required for `chores_by_area` and `room_brief` area discovery |
| `tasks_add` | Create a new Grocy task |
| `tasks_tag` | Set `homeassistant_area` userfield on an existing task |
| `stock_entry_update` | Edit a stock entry (best_before_date, location, price). Returns `{product_id, product_name, entry_id, updated_entry}`. **Fixed (v5.3.1):** calling this with only `entry_id` (no product context to resolve first) previously left the pre-fetch `current_entry_base` as `none`, silently sending an empty PUT payload. It now fetches the entry directly via `objects/stock?query[]=id=<entry_id>` when only `entry_id` is given, and derives the true `stock_id` from that fetched record rather than trusting the caller-supplied `entry_id` directly. |
| `stock_register_asset` | Register a permanent physical asset and stock one unit |
| `locations_metadata_set` | Bind Grocy locations to HA area and parent/container metadata |
| `provision_bom` | Native BOM provisioner — unit resolution, product create/find, meta update, HA label tagging, chore create/find, chore tag, installed stock seed — all in one call. Requires `bom` (JSON string) and `confirm_action: true`. Returns `catalog` dict keyed by part key with `product_id`, `unit_id`, `chore_id`, `storage_location_id`, `consume_location_id`, `is_new`. |
| `userfields_list` | List all Grocy userfield schema definitions. Optional `entity=` filter |
| `userfields_create` | Create a new userfield on a Grocy entity. Idempotent — returns `already_exists` if `entity.field_name` exists. Required: `entity`, `field_name`, `field_caption` |
| `userfields_deploy` | Idempotent deploy of canonical ZenOS userfield schema — creates missing fields, skips existing, stamps `schema_version` to household cabinet |
| `userfields_repair` | Diff live Grocy schema vs canonical. Reports `ok/missing/type_drift/unknown` + version delta. Fixes missing unless `dry_run=true` |
| `userfields_delete` | Delete a userfield schema definition by ID. Warns if canonical. Requires `confirm_action=true` |
| `userentities_list` | List all custom Grocy userentity types |
| `userentities_create` | Create a new custom userentity type. Required: `item` (snake_case slug), `field_caption`. Returns `entity_qualifier` — the string used in `userfields_*` and `userentity_values_*` calls |
| `userentities_delete` | Delete a custom userentity and all its userobjects. IRREVERSIBLE. Requires `confirm_action=true`. Preview without it. |
| `userobjects_list` | List userobjects. Optional `userentity_id` filter. |
| `userobjects_create` | Create a userobject under a userentity. Required: `userentity_id`. Returns `created_id` — use immediately with `userentity_values_set` to attach field values |
| `userobjects_delete` | Delete a userobject. Requires `userobject_id`, `confirm_action=true` |
| `userentity_values_get` | Read all userfield values for a userobject. Required: `entity` (the `userentity-{name}` form), `userobject_id` |
| `userentity_values_set` | Write userfield values for a userobject. Required: `entity`, `userobject_id`, `values_json` (JSON dict string) |

Use `mode=help` on `zen_dojotools_inventory` for the complete 96-operation catalog.

**`slim_objects` field:** Pass `slim_objects: true` on any large object list fetch (products, locations, chores) to return only `{id, name}` per item — prevents template overflow on big catalogs.

---

## Area And Location Model

Grocy locations can be bound into Home Assistant topology with userfield metadata:

| Field | Purpose |
|-------|---------|
| `homeassistant_area` | HA area slug associated with the Grocy location |
| `grocy_parent_location_id` | Parent container ID for nested storage |
| `grocy_location_subclass` | Semantic role such as `anchor`, `container`, `zone`, or `virtual` |
| `placement_priority` | Sort order inside the area |

This is what lets Room Manager ask "what inventory belongs to this room?" without hand-maintained entity lists. The important room-facing modes are:

* `stock_area_summary`: compact count and anchor view for a room or area.
* `stock_area_volatile`: volatile items (overdue, due soon, expiring) scoped to a HA area.
* `stock_area_inventory`: denormalized detailed view with product names and amounts.
* `chores_by_area`: maintenance chores discovered through stocked products or direct `homeassistant_area` tags.

---

## stock_area_volatile

Returns all volatile (overdue, due-soon, or expiring) items whose `location_id` falls inside the given HA area.

**Input:** `homeassistant_area_id` (required) — HA area slug (e.g., `kitchen`).

**How it works:**

1. GET `/objects/locations` — collect all Grocy locations tagged with `userfields.homeassistant_area` matching the requested area. Build the area location ID set.
2. GET `stock/volatile` — fetch the whole-house overdue/due_soon/expiring lists.
3. For each item in all three lists: include it if `item.location_id` is a member of the area location ID set.

Area membership is determined by the stock entry's storage location (`item.location_id`), not the product's default location. Items without a `location_id` are excluded.

**Response shape:**

```json
{
  "status": "success",
  "mode": "stock_area_volatile",
  "area_id": "<area_slug>",
  "location_ids": [<int>, ...],
  "overdue": [...],
  "due_soon": [...],
  "expiring": [...],
  "counts": {
    "overdue": <int>,
    "due_soon": <int>,
    "expiring": <int>
  }
}
```

Items in `overdue`, `due_soon`, and `expiring` are the full Grocy volatile entry objects — the same shape returned by `GET stock/volatile`.

**If the area has no tagged locations:** `location_ids` is `[]` and all three item lists are empty (no error).

**Example:**

```yaml
zen_dojotools_inventory:
  mode: stock_area_volatile
  homeassistant_area_id: kitchen
```

---

## AutoVac Integration

AutoVac uses Grocy for robot consumables: bags, filters, brushes, pads, and other model preset parts.

```mermaid
flowchart TD
  Provision["AutoVac consumables provision"]
  Robot["Resolve vacuum entity and serial number"]
  Locations["Create or resolve robot, dock, bin, and spare locations"]
  Products["Create or resolve part products from model preset"]
  Catalog["Write household.autovac.grocy_catalog"]
  Wear["autovac_wear sensors"]
  Check["check_wear"]
  Stock["Grocy stock check"]
  Decision{"Spare on hand?"}
  Replace["Postman: replace part"]
  Shop["Add product to shopping list"]
  Chore["Execute linked maintenance chore"]

  Provision --> Robot --> Locations --> Products --> Catalog
  Catalog --> Check
  Wear --> Check
  Check --> Stock --> Decision
  Decision -->|yes| Replace --> Chore
  Decision -->|no| Shop --> Replace
```

AutoVac stores its Grocy bindings in the `autovac` drawer under `grocy_catalog`. Each part entry can include:

* `product_id`
* `name`
* `sku`
* `storage_location_id`
* `installed_location_id`
* `min_stock`
* `chore_id`
* `category`
* `wear_sensor_key`
* `wear_threshold`
* `wear_entity`

The operational loop is:

1. `mode=consumables action=provision` builds the catalog and writes the cabinet binding.
2. `mode=check_wear` reads wear sensors from `autovac_wear` labels and compares them to catalog thresholds.
3. If a part is worn, AutoVac checks stock through Grocy.
4. If stock exists, Postman tells the human a spare is available.
5. If stock is missing, AutoVac adds the item to the Grocy shopping list.
6. `action=log_replaced` consumes one spare and executes the linked chore when `chore_id` exists.
7. `action=log_purchased` adds new spares to the configured storage location.

See also: [AutoVac](../components/autovac.md).

---

## SpaMaster Integration

SpaMaster uses Grocy for spa parts and chemistry supplies. It combines preset-driven catalog creation with stock, shopping, and maintenance chores.

```mermaid
flowchart TD
  Setup["SpaMaster setup"]
  Preset["Model and chemistry presets"]
  Location["Create or resolve spa area location"]
  Parts["Create part products"]
  Chems["Create chemistry supply products"]
  Chores["Find or create maintenance chores"]
  Catalog["Write spa_config.grocy_catalog"]
  Status["consumables status"]
  Shopping{"Low or out?"}
  AddShop["add_to_shopping"]
  Replace["log_replaced"]
  Purchase["log_purchased"]
  Consume["Grocy stock_consume"]
  AddStock["Grocy stock_add_purchase"]

  Setup --> Preset --> Location
  Location --> Parts --> Catalog
  Location --> Chems --> Catalog
  Parts --> Chores --> Catalog
  Catalog --> Status --> Shopping
  Shopping -->|yes| AddShop
  Replace --> Consume --> Chores
  Purchase --> AddStock
```

SpaMaster stores its Grocy bindings in `spa_config.grocy_catalog`. The catalog is split into `parts` and `chem_supplies`, then combined at runtime for consumable actions.

The operational loop is:

1. `mode=consumables action=provision` loads a spa preset, creates or reuses products, creates or reuses chores, and writes the catalog.
2. `action=status` reports current stock and reorder flags.
3. `action=add_to_shopping` queues all low or out items to the Grocy shopping list.
4. `action=log_replaced part=<key>` consumes stock and executes the linked maintenance chore when present.
5. `action=log_purchased part=<key> amount=<n>` adds purchased stock to the part's configured storage location.

See also: [SpaMaster](../components/spamaster.md).

---

## First-Time Setup

1. Add the Grocy API key to `secrets.yaml`:

```yaml
grocy_api_key: <your_api_key>
```

2. In Home Assistant, set `input_text.grocy_url` to the HTTPS base URL for Grocy:

```text
https://<your-grocy-host>
```

Do not add a trailing slash. Use HTTPS, not HTTP. If a reverse proxy redirects HTTP to HTTPS, Home Assistant may follow the redirect in a way that changes POST requests into GET requests, which breaks writes.

3. Expose `zen_dojotools_inventory` to the conversation agent if the agent should help with household inventory. Keep `zen_sutra_grocy` internal — it is the low-level REST dispatcher and is not MCP-exposed.

4. Provision domain catalogs from the owning tools:

```yaml
zen_dojotools_autovac:
  mode: consumables
  action: provision
  model_preset: roborock_s8_pro_ultra
```

```yaml
zen_dojotools_spamaster:
  mode: consumables
  action: provision
  model_preset: caldera_utopia_florence_2024
```

---

## Getting Started — Area Inventory

This walkthrough assumes Grocy is reachable (API key and URL configured) and at least one HA area exists. The goal: one room's inventory visible to Room Manager and queryable by area.

### Step 1 — Tag a Grocy location with an HA area

Create the Grocy location in the Grocy UI if it does not exist. Then bind it to an HA area slug:

```yaml
zen_dojotools_inventory:
  mode: locations_metadata_set
  location_id: 12              # Grocy location ID (integer)
  homeassistant_area: kitchen  # HA area slug
  grocy_location_subclass: anchor
```

Use `anchor` for the primary location of the area. Use `container` for nested shelves or bins inside the anchor. A room can have multiple tagged locations — all of them contribute to area queries.

Audit the binding after tagging:

```yaml
zen_dojotools_inventory:
  mode: locations_audit
```

Returns all Grocy locations with their `homeassistant_area` userfield. Confirm the slug matches an active HA area.

### Step 2 — Add products and stock them

In the Grocy UI, set each product's default location to the tagged location. Purchased stock defaults to that location. Alternatively, use `stock_buy_product` to create a product and stock it in one call:

```yaml
zen_dojotools_inventory:
  mode: stock_buy_product
  item: "Dish Soap"
  amount: 2
  location_id: 12    # anchor location for kitchen
```

### Step 3 — Verify the area summary

```yaml
zen_dojotools_inventory:
  mode: stock_area_summary
  homeassistant_area_id: kitchen
```

Returns container count, product count, and stock totals for the area. Correct counts confirm the area binding is working.

### Step 4 — Check volatile items

```yaml
zen_dojotools_inventory:
  mode: stock_area_volatile
  homeassistant_area_id: kitchen
```

Returns overdue, due-soon, and expiring items whose stock entries are located inside the area. Empty results are normal if no items have open or best-by dates set.

### Room Manager reads this automatically

Once locations are tagged, Room Manager's `+inventory` slice calls `stock_area_volatile` for the room on every expand. No additional configuration is needed:

```yaml
zen_dojotools_room_manager:
  mode: room
  area_id: kitchen
  output_fields: "+inventory"
```

The `inventory` slice in the response carries the kitchen's volatile items from this point forward.

---

## Null Unit Safety

`stock_buy_product` and `stock_add_purchase` both guard against products with `qu_id_stock = null`. A null unit product cannot receive stock — Grocy rejects the transaction. Instead of silently failing, both modes now return immediately:

```json
{
  "status": "error",
  "message": "Product {id} has no stock unit. Set a unit via update_product_meta before adding stock.",
  "fix": "mode=update_product_meta product_id=<id> unit_id=<id>"
}
```

**Root cause:** `catalog_add_product` called without `unit_id` creates products with `qu_id_stock = null`. Grocy permanently blocks unit changes via API once stock history exists. The only recovery is delete and recreate with the correct unit specified at POST time.

**Prevention:** `catalog_add_product` now resolves the instance default unit ("each" → "Piece" fallback) when `unit_id` is omitted, before creating the product. `provision_bom` always passes `unit_id` — use it for any multi-part provisioning scenario.

**`unit_conversions_add` global guard:** Calling `unit_conversions_add` without a `product_id` (global scope) now requires `confirm_action: true`. Global unit conversions affect all products in the instance and are a common source of accidental side effects. Scoped (per-product) conversions do not require confirmation.

**`units_add` reactivation (v5.3.1):** if a unit name matches an existing but inactive unit, `units_add` now reactivates it instead of erroring on the duplicate name.

---

## update_product_meta

Update fields on an existing product. Notable fields (v5.3.1):

| Field | Purpose |
|-------|---------|
| `best_before_days` | Default shelf-life-after-purchase for this product, in days. `-1` disables. Feeds Perishable Storage Coaching's recommendation flow. |
| `due_days_after_open` | Days until due once opened (distinct from `best_before_days`, which is unopened shelf life). |
| `no_own_stock` | Grocy's "does not have its own stock" flag — for parent/grouping products that track child products' stock instead of their own. |

---

## Perishable Storage Coaching

New in v5.3.1. Locations and products can be tagged `catalog_class: perishable_storage` or `catalog_class: non_perishable`. Classification is inherited by walking up the location's parent chain (`grocy_child_location_ids`) if not set directly, so tagging one shelf tags its whole sub-tree.

- **`stock_audit_perishable`** — paginated cross-reference of all locations/stock against never-expire products stored in a `perishable_storage`-classed location. Surfaces products that likely should have a `best_before_days` set but don't.
- **Perishable Storage Coach hook** — fires on `stock_buy_product`/purchase/transfer into a perishable-classed location. If the product has no `best_before_days` set, the response includes a soft recommendation to set one — never blocks the write.

---

## COGS Coaching

New in v5.3.1, works alongside [`zen_codex_finance_cogs`](firefly_iii.md#codex-tier). Products tagged with the HA label `cogs_tracked` get cost-of-goods-sold treatment:

- **COGS Zone Coach** — buy/purchase/transfer/adjust actions into a `cogs_zone`-tagged location warn (non-blocking) if the product isn't `cogs_tracked` yet.
- **COGS Read Coach** — `stock_check_item`/`stock_where_is_item`/`stock_entries_for_item` warn on cost-basis gaps for tracked products.
- **COGS Undo Coach** — `stock_undo_booking` warns that Firefly is not automatically reversed; a matching reversal must be posted separately via the finance codex.
- **Auto-post hook** — consuming a `cogs_tracked` product automatically calls `zen_dojotools_finance mode=run case=cogs_post`, posting the COGS transaction to Firefly.

---

## Battery Management

New in v5.3.1. Grocy-side tracking of rechargeable batteries as stock, cross-referenced against the HA "Battery Notes" integration for non-rechargeable device batteries. This is distinct from the Lens Bus `zen_stack_battery` provider ([Battery Notes Plugin](battery_notes.md)) — that provider answers "what needs a battery, by area" for the Lens Bus; these Grocy modes answer "manage rechargeable battery stock and charge cycles" and "cross-reference Battery Notes sensors against Grocy stock for replacement/shopping decisions." They read the same underlying HA sensors but serve different callers.

| Mode | Purpose |
|------|---------|
| `batteries_due` | Rechargeable batteries bucketed by charge status: `overdue`, `due_today`, `due_soon`, `ok` |
| `batteries_journal` | Charge-cycle history, optionally filtered to one battery |
| `batteries_charge_undo` | Undo a specific charge cycle by its journal entry ID |
| `battery_status` | Low-battery triage across the house — groups HA Battery Notes devices by area with a Grocy stock cross-reference |
| `battery_overdue` | Devices not replaced in `days_threshold` days (template-only, reads Battery Notes sensors) |
| `battery_replace` | Atomic: logs the replacement in Battery Notes (`battery_notes.set_battery_replaced`) AND consumes one unit from Grocy stock |

---

## Logistics KFC Components

`zen_sutra_logistics` (`packages/zenos_ai/plugins/grocy/sutra_logistics.yaml`) is a sibling KFC-manifest-only file — no FileCabinet calls, no REST calls, pure `zen_dojotools_manifest mode=bootstrap_kfc` registration. It declares two components:

| Component | Seeds | Trigger |
|---|---|---|
| `logistics_intake` | `zen_dojotools_inventory mode=stock_list_by_location` (Kitchen Island catalog intake) | Daily midnight + noon |
| `logistics_volatile` | `zen_dojotools_inventory mode=stock_list_volatile` (overdue/expiring stock) | Daily midnight + noon |

Battery KFC was deliberately moved out of this file to `zen_stack_battery` (see [Battery Notes Plugin](battery_notes.md)) — it is not duplicated here.

---

## Userfields Schema Management

ZenOS uses Grocy userfields to bind inventory objects to HA topology. The canonical schema defines 13 fields across locations, products, chores, and tasks.

### Deploying the Schema

On a fresh Grocy install, run:

```yaml
zen_dojotools_inventory:
  mode: userfields_deploy
```

Idempotent — safe to re-run. Creates any missing fields, skips existing ones, stamps `schema_version: "1.0.0"` to the household cabinet under `grocy_schema`.

### Auditing the Schema

```yaml
zen_dojotools_inventory:
  mode: userfields_repair
  dry_run: true
```

Returns `deployed_version`, `healthy: true/false`, and counts for `ok / missing / type_drift / unknown`. Use `dry_run: false` (or omit) to fix missing fields in place.

`unknown` fields are ones Grocy has that are not in the ZenOS canonical schema — `products.mealie_ingredient_id` is a common example (Mealie-owned, expected). Not a bug.

### Listing All Fields

```yaml
zen_dojotools_inventory:
  mode: userfields_list
  entity: locations   # optional filter: locations|products|chores|tasks
```

### Canonical Field Reference

| Entity | Field | Purpose |
|--------|-------|---------|
| `locations` | `homeassistant_area` | HA area slug bound to this Grocy location |
| `locations` | `grocy_parent_location_id` | Parent container ID for nested storage |
| `locations` | `homeassistant_entity_id` | HA entity_id representing this location |
| `locations` | `homeassistant_labels` | JSON array of HA labels |
| `locations` | `placement_priority` | Sort order for conflict resolution |
| `locations` | `grocy_location_subclass` | `suite\|room\|container\|furniture\|shelf\|drawer\|...` |
| `locations` | `grocy_child_location_ids` | JSON array of child location IDs |
| `products` | `homeassistant_area` | Direct product-to-area link |
| `products` | `hazmat_class` | `flammable\|corrosive\|toxic\|oxidizer\|other` |
| `products` | `zen_asset_type` | `consumable\|asset\|equipment` |
| `products` | `catalog_class` | `perishable_storage\|non_perishable` (also used on `locations` — see [Perishable Storage Coaching](#perishable-storage-coaching)) |
| `products` | `cogs_tracked` (`ha_labels`) | Not a userfield — an HA label. Presence triggers COGS auto-posting on consume; see [COGS Coaching](#cogs-coaching) |
| `userentity-depreciable_asset` | `tax_context`, `owner_entity`, `paid_by_entity`, `reimbursement_status`, `allocation_method`, `business_area_anchor`, `evidence_refs`, `tax_review_status` | Tax/allocation fields feeding `zen_codex_finance_depreciation` — see [Firefly III](firefly_iii.md#codex-tier) |
| `chores` | `homeassistant_area` | HA area ID — required for `chores_by_area` and `room_brief` discovery |
| `chores` | `homeassistant_entity_id` | HA schedule or todo entity linked to this chore |
| `tasks` | `homeassistant_area` | HA area ID for area-based task queries |

---

## ERP Object Substrate — Userentities and Userobjects

### Built-in entities vs. custom userentities

Grocy ships with a fixed set of built-in entity types: **products**, **locations**, **chores**, **tasks**, **recipes**, **shopping lists**, **batteries**. Their schemas are set — you can extend them with userfields, but you can't change their core structure.

**Userentities** are custom object types you define entirely. Each userentity is a named type (`room`, `vehicle`, `appliance`), and each instance of that type is a **userobject**. You define the schema yourself via userfields, create objects of that type, and read/write values per object.

The distinction:

| | Built-in (e.g. `location`) | Userentity (e.g. `room`) |
|---|---|---|
| Schema | Fixed by Grocy | Fully custom — you define every field |
| Instances | Locations created via Grocy UI or `locations_add` | Userobjects created via `userobjects_create` |
| Field values | Stored as userfields on built-in object | Stored via `userentity_values_set` per userobject |
| Queried by | `mode=locations_list` etc. | `mode=userobjects_list userentity_id=<id>` |
| Use case | Grocy storage bins, pantry zones | HA-aware rooms, vehicles, appliances, assets |

### The `room` userentity pattern

Grocy `locations` model *where things are stored* — a pantry shelf, a garage bin. They're fine for that. But a **room** in the ZenOS sense is something richer: it has an HA area slug, a floor, sensors, topology links, and chores/tasks attached to it.

A `room` userentity stores that extended room model inside Grocy's XRM layer:

```yaml
# Step 1: Create the userentity type (once)
zen_dojotools_inventory:
  mode: userentities_create
  item: room
  field_caption: "Home Room"
# → returns entity_qualifier: "userentity-room"

# Step 2: Define fields for this type
zen_dojotools_inventory:
  mode: userfields_create
  entity: userentity-room
  field_name: ha_area_id
  field_caption: "HA Area ID"

zen_dojotools_inventory:
  mode: userfields_create
  entity: userentity-room
  field_name: floor_id
  field_caption: "Floor"

# Step 3: Create a room object (one per room)
zen_dojotools_inventory:
  mode: userobjects_create
  userentity_id: <id from step 1>
# → returns created_id: 7

# Step 4: Write values
zen_dojotools_inventory:
  mode: userentity_values_set
  entity: userentity-room
  userobject_id: 7
  values_json: '{"ha_area_id": "kitchen", "floor_id": "ground_floor"}'

# Read back
zen_dojotools_inventory:
  mode: userentity_values_get
  entity: userentity-room
  userobject_id: 7
# → {values: {ha_area_id: "kitchen", floor_id: "ground_floor"}}
```

Once rooms exist as userobjects, you can attach chores and inventory stock to them directly in Grocy — the same graph that already tracks your consumables now tracks the spaces they live in. Room Manager remains the canonical topology source; the Grocy userentity is the inventory anchor.

---

## Troubleshooting

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| Grocy reads work but writes fail | `grocy_url` is HTTP or proxy redirects POST | Set `input_text.grocy_url` to HTTPS directly |
| AutoVac says consumables are not provisioned | `autovac.grocy_catalog` missing | Run AutoVac `mode=consumables action=provision` |
| SpaMaster says consumables are not provisioned | `spa_config.grocy_catalog.parts` missing | Run SpaMaster `mode=consumables action=provision` |
| Room inventory is empty | Grocy locations are not tagged with `homeassistant_area` | Use `locations_metadata_set` or `locations_audit` |
| Maintenance chores do not appear by area | Chores are not linked to products or area userfield is missing | Check `chores_by_area` setup |
| Product update is blocked by unit error | Existing product has a null or incompatible quantity unit | Follow the tool's delete/recreate or conversion guidance |

---

## Source Notes

This page is derived from:

* `packages/zenos_ai/plugins/grocy/readme.md`
* `packages/zenos_ai/plugins/grocy/grocy.yaml`
* `packages/zenos_ai/plugins/grocy/sutra_logistics.yaml`
* [AutoVac](../components/autovac.md)
* [SpaMaster](../components/spamaster.md)
* [Firefly III — Codex Tier](firefly_iii.md#codex-tier)
* [Battery Notes Plugin](battery_notes.md)
