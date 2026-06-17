# ZenOS-AI Project Cost Tracker Component

**Version:** 0.1.0 (bones)  
**Package:** `packages/zenos_ai/plugins/project/project.yaml`  
**Primary script:** `zen_dojotools_project`

---

## Overview

The Project cost tracker is a multi-backend tool for tracking household and home-improvement projects. It spans three systems:

* **Grocy** — materials, assets, contractor agreements, invoices, and change orders (ERP/inventory layer)
* **Firefly III** — actual spend ledger (transactions tagged by project and phase)
* **FileCabinet** — project manifests and saga event logs (local state, idempotency)

Version 0.1.0 ships the core receipt path, project setup, and reconcile. The contractor, invoice, change order, budget vs actual, and phase summary cases are scaffolded and return `not_implemented` — they are coming in v0.2.0 and v0.3.0.

---

## Backing Store Model

```
FileCabinet (household cabinet)
  project_{project_id}_manifest    — project graph root: phases, IDs, config
  project_{project_id}_events      — idempotency/saga log {md5_key: {status, ids}}

Grocy
  product_groups                   — phases (one group per phase)
  products                         — materials and assets (assigned to phase group)
  stock_entries                    — material receipts (price, vendor, date)
  userentity: project_contract     — contractor agreements
  userentity: project_invoice      — labor/contractor invoices
  userentity: project_change_order — scope changes with cost delta

  Userfields on products:
    project_id, project_phase, cost_estimate, asset_tag, install_date

Firefly III
  transactions                     — actual spend (tagged by project + phase slug)
  categories                       — one per project
```

---

## Core Cases

Use `mode=run` and specify a `case`. Use `mode=help` to get the catalog back from the tool at runtime.

| Case | Status | Intent |
|------|--------|--------|
| `project_list` | Live | List all active projects from the household cabinet |
| `project_setup` | Live | Create project manifest, Grocy userentities, and product userfields. Idempotent. |
| `project_receipt_log` | Live | Primary write path — dual-write: Grocy stock purchase + Firefly transaction + saga log |
| `project_reconcile` | Live | Find saga events in `pending`, `reconcile_needed`, or `failed` state |
| `project_add_contract` | v0.2.0 | Create contractor contract userobject in Grocy |
| `project_add_invoice` | v0.2.0 | Create contractor invoice userobject; optionally log Firefly transaction |
| `project_add_change_order` | v0.2.0 | Record a scope change with cost delta against a phase |
| `project_budget_vs_actual` | v0.3.0 | Aggregate estimated vs actual spend across Grocy + invoices + change orders |
| `project_phase_summary` | v0.3.0 | Phase-level rollup — open work, spend, blockers, documents |

---

## Key Fields

| Field | Required by | Notes |
|-------|-------------|-------|
| `mode` | all | `run`, `help`, or `tool_manifest` |
| `case` | `run` | See case table above |
| `project_id` | most cases | Snake_case slug (e.g. `backyard_2026`) — auto-lowercased and space-replaced |
| `project_name` | `project_setup` | Human display name |
| `phase_id` | `project_receipt_log`, contract/invoice/CO cases | Snake_case phase slug (e.g. `deck_framing`) |
| `phase_name` | `project_setup` | Phase display name |
| `phase_budget` | `project_setup` | Estimated budget for the phase |
| `phases_json` | `project_setup` | JSON list of phase dicts `[{id, name, budget}]` |
| `vendor` | `project_receipt_log`, contract/invoice cases | Vendor or contractor name |
| `product_label` | `project_receipt_log` | Material or asset name — resolves to Grocy product |
| `product_unit` | `project_receipt_log` | Unit of measure; defaults to `each` |
| `quantity` | `project_receipt_log` | Quantity purchased; default 1 |
| `unit_cost` | `project_receipt_log` | Cost per unit |
| `amount` | invoices, contracts, change orders | Total amount |
| `source_account` | `project_receipt_log` | Firefly account name for the transaction |
| `description` | `project_receipt_log` | Transaction description; defaults to `{product_label} — {vendor}` |
| `date` | `project_receipt_log` | `YYYY-MM-DD`; defaults to today |
| `receipt_ref` | `project_receipt_log` | FileCabinet ref, URL, or photo path for the receipt |
| `contract_scope` | `project_add_contract` | Scope description |
| `draw_schedule_json` | `project_add_contract` | JSON list of draw amounts/dates |
| `invoice_number` | `project_add_invoice` | Invoice number or reference |
| `change_order_description` | `project_add_change_order` | Scope change description |
| `cost_delta` | `project_add_change_order` | Cost impact; positive = increase |
| `item_id` | optional on some cases | Grocy product ID or userobject ID — skips name resolution |
| `confirm_action` | `project_add_change_order` | Boolean gate for destructive ops |

---

## project_receipt_log — The Primary Write Path

`project_receipt_log` is where most day-to-day project work lands. One call does all of:

1. Reads the project manifest from FileCabinet to get phase and Firefly tag context.
2. Computes an idempotency key from `project_id | phase_id | vendor | date | amount | product_label` (MD5).
3. Checks the saga event log — returns `already_logged` if a matching complete event exists, `reconcile_needed` if a previous attempt left a pending event.
4. Writes a `pending` saga event.
5. Finds or creates the Grocy product by name.
6. Logs a Grocy stock purchase with quantity, unit cost, and vendor.
7. Logs a Firefly withdrawal tagged `{project_id}-{phase_id}` under the project's Firefly category.
8. Writes the final saga event with both IDs and `status: complete | reconcile_needed | failed`.

The total amount is `amount` if provided; otherwise `quantity × unit_cost`.

```yaml
zen_dojotools_project:
  mode: run
  case: project_receipt_log
  project_id: backyard_2026
  phase_id: deck_framing
  vendor: "Home Depot"
  product_label: "Pressure-treated 2x8x12"
  quantity: 8
  unit_cost: 23.43
  source_account: "Checking"
```

### Saga Status Values

| Status | Meaning |
|--------|---------|
| `complete` | Both Grocy and Firefly writes succeeded |
| `reconcile_needed` | One write succeeded, the other failed — or a previous pending event exists |
| `failed` | Both writes failed |
| `already_logged` | Identical event already complete — idempotent no-op |

Use `project_reconcile` to surface all events needing attention for a project.

---

## project_setup

`project_setup` is idempotent — safe to re-run. It:

1. Creates or updates the project manifest in the household FileCabinet.
2. Ensures the four Grocy userentities exist: `project_contract`, `project_invoice`, `project_change_order`, `project_event`.
3. Ensures the five product userfields exist on Grocy products: `project_id`, `project_phase`, `cost_estimate`, `asset_tag`, `install_date`.
4. Builds phase records with Firefly tag slugs (`{project_id}-{phase_id}`).

Grocy product group creation per phase is deferred to v0.2.0.

```yaml
zen_dojotools_project:
  mode: run
  case: project_setup
  project_id: backyard_2026
  project_name: "Backyard Upgrade 2026"
  phases_json: '[{"id":"deck_demo","name":"Deck Demo","budget":2000},{"id":"deck_framing","name":"Deck Framing","budget":8500}]'
```

---

## project_reconcile

Returns all saga events for a project that are in `pending`, `reconcile_needed`, or `failed` state. Use this to find any receipt log call that partially failed.

```yaml
zen_dojotools_project:
  mode: run
  case: project_reconcile
  project_id: backyard_2026
```

Response includes `total_events`, `attention_count`, and the full `needs_attention` list with the saga event detail for each.

---

## First-Time Setup

Prerequisites: Grocy and Firefly III must already be configured with their respective plugins. See:

* [Grocy](grocy.md) — `input_text.grocy_url` and Grocy API key
* [Firefly III](firefly_iii.md) — `input_text.firefly_url` and Firefly III bearer token

Per-project setup:

1. Run `project_setup` once per project with a `project_id`, `project_name`, and optional `phases_json`.
2. Verify the manifest appeared in the household cabinet by running `project_list`.
3. Start logging receipts with `project_receipt_log`.

No additional HA helpers or secrets are needed beyond those already set up for Grocy and Firefly III.

---

## Roadmap Notes

As of v0.1.0, the following cases return `not_implemented`:

* `project_add_contract` — v0.2.0 via Grocy `project_contract` userentity
* `project_add_invoice` — v0.2.0 via Grocy `project_invoice` userentity + optional Firefly transaction
* `project_add_change_order` — v0.2.0 via Grocy `project_change_order` userentity; `cost_delta` adjusts phase budget in manifest
* `project_budget_vs_actual` — v0.3.0; aggregates saga events + invoices + change order deltas per phase
* `project_phase_summary` — v0.3.0; phase rollup from manifest + saga events

A CRM layer (Twenty or EspoCRM) and a service desk (Zammad) are noted in the YAML as planned but not yet decided.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `project_receipt_log` returns `reconcile_needed` | A previous call left the saga in `pending` state | Run `project_reconcile` to inspect; resolve Grocy or Firefly side manually, then mark or re-log |
| `project_receipt_log` returns `already_logged` | Identical receipt was logged before | Expected behavior — no action needed |
| Grocy product not found and create fails | Grocy unreachable or API key wrong | Check `input_text.grocy_url` and Grocy API key in `secrets.yaml` |
| Firefly transaction not created (`firefly_ok: false`) | Firefly unreachable, bad bearer token, or account name not found | Verify `input_text.firefly_url` is HTTPS and bearer token is current |
| `project_list` shows no projects | No manifests written yet | Run `project_setup` for each project |
| `project_setup` userentity creation silently skipped | Userentity already exists | Expected — setup is idempotent; check `userentities_list` via `zen_dojotools_inventory` |

---

## Source Notes

This page is derived from:

* `packages/zenos_ai/plugins/project/project.yaml`
* [Grocy](grocy.md)
* [Firefly III](firefly_iii.md)
