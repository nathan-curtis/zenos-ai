# ZenOS-AI Grocy Integration

**Version:** 2.12.1
**Status:** UAT PASS (2026-03-02)
**HALMark:** PASS (v0.9.10-draft)

---

## Overview

Zen DojoTools Grocy Integration provides a deterministic, governed control layer between Home Assistant and Grocy ERP.

This module exposes two coordinated components:

* **Zen DojoTools Grocy Helper** — High-level intent router (56 supported cases)
* **Zen DojoTools Grocy Advanced** — Low-level REST dispatcher (CRUD + OpenAPI introspection)

Grocy is treated as the canonical authority for:

* Products
* Locations
* Quantity Units
* Stock Entries
* Recipes
* Shopping Lists
* Tasks / Chores
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

## Architecture

### 1. Grocy Helper (Primary Interface)

High-level intent interface covering 56 operational cases:

* Inventory (check, buy, consume, transfer, adjust)
* Shopping list automation
* Recipe fulfillment and consumption
* Location management (with metadata governance)
* Product metadata and merges
* Unit governance
* Barcode flows
* Tasks and chores
* Battery tracking

Helper resolves:

* Product names → product IDs
* Location names → location IDs
* Unit names → unit IDs
* Recipe names → recipe IDs
* Task/Chore names → IDs

All resolution occurs before write execution.

If multiple matches are found, execution halts.

---

### 2. Grocy Advanced (REST Dispatcher)

Internal CRUD dispatcher for:

* GET
* POST
* PUT
* DELETE

Features:

* Endpoint normalization
* Pagination support
* Query search support
* Path parameter injection
* OpenAPI help surface
* 405 method coaching
* Strict payload enforcement

The Helper calls Advanced.
Users should prefer Helper unless Advanced is explicitly required.

---

## Setup

### 1. Add API Key

In `secrets.yaml`:

```
grocy_api_key: <your_api_key>
```

Grocy uses a bare key — no prefix.

---

### 2. Configure Grocy Base URL

Create or set:

```
input_text.grocy_url
```

Value:

```
https://<your-grocy-host>
```

No trailing slash.

⚠️ if HTTPS is required.

If Grocy is behind a reverse proxy that redirects HTTP → HTTPS, Home Assistant will follow the 301 and convert POST requests to GET per RFC behavior.
All write operations will silently fail if HTTP is used.

---

## Unit Governance

Quantity units are first-class governed objects.

Rules:

* Units must exist in Grocy
* Exact match preferred
* Partial fallback allowed
* Multiple matches halt execution
* No inferred pluralization
* No implicit conversion creation

If "each" does not exist and is required for stock math, the system will attempt to create it once.

---

## Safe Write Operations

Destructive actions are guarded:

* Product merges require explicit keep/remove IDs
* Transfers blocked for tare-weight products
* stock_entry_update uses read-modify-write safety
* Location and product renames require name changes
* Purchase requires location when creating product
* Ambiguous name matches block execution

---

## Supported Functional Domains

### Inventory

* stock_check_item
* stock_buy_product
* stock_add_purchase
* stock_consume
* stock_transfer_location
* stock_inventory_adjust
* stock_entry_update (safe partial update)
* stock_open_item
* stock_undo_booking
* stock_list_volatile
* stock_overview

### Catalog

* catalog_list_products
* catalog_find_product
* catalog_find_by_barcode
* rename_product
* update_product_meta
* products_merge
* set_default_location_for_product

### Locations

* locations_add
* locations_list
* locations_find
* locations_rename
* locations_metadata_set

Location metadata binds Grocy to Home Assistant via:

* grocy_parent_location_id
* homeassistant_area
* grocy_location_subclass
* placement_priority

These fields support cross-system search indexing.

### Units

* units_list
* units_find
* units_add
* units_update

### Recipes

* recipes_list
* recipes_fulfillment
* recipe_check_fulfillment
* recipes_add_to_shopping
* recipes_consume
* recipes_copy

### Shopping

* shopping_add_product
* shopping_remove_product
* shopping_clear_list
* shopping_add_missing
* shopping_add_overdue
* shopping_add_expired

### Tasks & Chores

* tasks_list
* tasks_find
* tasks_complete
* tasks_undo
* chores_list
* chores_find
* chores_execute
* chores_undo

### Batteries

* batteries_list
* batteries_get
* batteries_charge

---

## Pagination

Object endpoints support pagination via:

* page
* per_page

Defaults:

* page: 1
* per_page: 25

Offset is computed automatically.

---

## Error Model

Failures return structured responses with:

* status
* message
* case
* details
* coaching

Common failure reasons:

* Missing required fields
* Ambiguous name resolution
* Invalid unit
* Missing location
* Invalid merge request
* Invalid rename
* Unsupported Grocy behavior
* Endpoint misuse

Errors include remediation guidance.

---

## UAT Coverage

Critical path verified against live Grocy instance:

* locations_add
* stock_buy_product
* stock_check_item
* shopping_add/remove_product
* recipes_list
* recipes_fulfillment
* batteries_list
* stock_list_volatile
* stock_overview
* stock_consume
* units_add
* stock_get_product_details
* recipe_check_fulfillment
* stock_entries_for_item
* catalog_find_by_barcode
* recipes_add_to_shopping
* chores_execute
* recipes_copy
* stock_undo_booking
* stock_open_item
* stock_add_by_barcode
* stock_consume_by_barcode
* stock_entry_update (read-modify-write)

Status: PASS

---

## Intended Usage Model

1. Use Helper with explicit case.
2. Let system resolve IDs.
3. If blocked, adjust inputs.
4. Avoid bypassing governance via Advanced unless necessary.

This layer enforces discipline so higher-level systems (LLMs, automation agents, dashboards) can interact safely with Grocy without corrupting state.

---

## Final Note

Strict by design.

This module prevents silent inventory corruption, ambiguous merges, unit drift, and unintended destructive writes.

If it blocks you, it is protecting the system.
