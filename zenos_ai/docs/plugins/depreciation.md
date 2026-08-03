# ZenOS Asset Depreciation

**Files:** `packages/zenos_ai/plugins/firefly_iii/zen_codex_finance_depreciation.yaml`  
**Scripts:** `zen_codex_finance_depreciation`, `zen_stack_depreciation`  
**Entry point:** `zen_dojotools_finance` (MCP-exposed) — all depreciation modes route through here  
**Version:** 1.0.0  

---

## Overview

Tracks depreciable household assets in Grocy, calculates straight-line depreciation schedules, posts transactions to Firefly III with full idempotency and audit trail. Friday calculates; Firefly records. No tax advice.

**Tier ladder:**
```
zen_dojotools_finance     MCP surface — dispatches to codex via catch-all
zen_codex_finance_depreciation   domain logic — asset schema, schedule, posting
zen_root_firefly          wire protocol — Firefly III REST calls
```

---

## Architecture

### Storage
- **Asset records:** Grocy `userentity-depreciable_asset` — one userobject per asset
- **Userobject ID** = `asset_id` in all modes
- **No cabinet footprint** — no Friday cabinet drawers used for asset data
- **Schedule:** calculated on-demand from asset fields + Firefly transaction history

### Idempotency
- Every write produces an `idempotency_key` (caller-supplied or derived: `depr:{mode}:{asset_id}:{date}`)
- Keys stored in Grocy `notes` field and Firefly transaction tags
- Posting the same period twice returns `already_posted` — safe to retry

### Reversal (not deletion)
- Never delete asset records or Firefly transactions
- `compensate` mode posts a Firefly reversing transaction tagged `reverses:{original_key}`

---

## Modes

### Setup (run once)
| Mode | Purpose |
|------|---------|
| `depreciable_asset_firefly_setup` | Create Grocy userentity schema + Firefly categories. Pass `asset_id` after create to add piggy bank. Idempotent. |

### Asset CRUD
| Mode | Required fields | Notes |
|------|----------------|-------|
| `depreciable_asset_create` | `name`, `cost_basis`, `useful_life_months`, `placed_in_service_date`, `depreciation_method` | Returns `asset_id` (Grocy userobject ID) |
| `depreciable_asset_get` | `asset_id` (int or name) | |
| `depreciable_asset_update` | `asset_id`, `asset_data` (JSON dict of changed fields) | Patches only changed fields. `evidence_refs` in `asset_data` takes precedence over stored value. |
| `depreciable_asset_list` | — | All assets, summary fields |

### Calculation
| Mode | Notes |
|------|-------|
| `depreciable_asset_calculate_schedule` | Pure calculation, no Firefly write. Returns full period-by-period schedule. |
| `depreciable_asset_book_value` | Current paper value. Reads Firefly transaction history to compute accumulated depreciation. Pass `include_evidence=true` to resolve `evidence_refs` inline. |
| `depreciable_asset_replacement_plan` | Monthly reserve calculation + risk flag (high/medium/low). |
| `depreciable_asset_report` | Portfolio summary — all assets, aggregate cost basis, book value, replacement exposure. |

### Posting
| Mode | Key fields |
|------|-----------|
| `depreciable_asset_post_depreciation_period` | `asset_id`, `period` (YYYY-MM), `posting_mode` (`report_only`\|`summary_post`), `confirm=true` to write |
| `depreciable_asset_dispose` | `asset_id`, `disposal_type`, `disposal_date`, `disposal_proceeds` — marks asset, stops future depreciation, calculates gain/loss |
| `compensate` | `idempotency_key` — reverses a prior write |

---

## evidence_refs

JSON list of typed cross-references stored on the asset. Format: `["type:id", ...]`.

**Common types:**
- `product:389` — linked Grocy product (grocy_product_id)
- `transaction:1447` — Firefly purchase or depreciation transaction

**Setting refs:**
```
depreciable_asset_update asset_id=21 asset_data={"evidence_refs":"[\"product:389\",\"transaction:1447\"]"} confirm=true
```

**Resolving refs inline** (on `book_value`):
```
depreciable_asset_book_value asset_id=21 include_evidence=true
```
Returns `resolved_evidence` array:
- `transaction:*` → full `transaction_evidence` item from `zen_stack_firefly`
- other types → `{ref, type, resolved: false, reason: unsupported_ref_type}` (graceful)

Auto-populate: `grocy_product_id` on the asset automatically adds `product:{id}` to `evidence_refs` on create/update.

---

## Lens Bus

`zen_stack_depreciation` is the Lens Bus provider for this codex.

| Anchor type | Matches on | Returns |
|-------------|-----------|---------|
| `label` | `room_anchor` field contains the label slug | `asset_depreciation_evidence` |
| `area_id` | `room_anchor` == anchor_id | `asset_depreciation_evidence` |
| `product` | `grocy_product_id` == anchor_id | `asset_depreciation_evidence` |

Consumers call `zen_dojotools_lens_dispatch`, not `zen_stack_depreciation` directly. Registered in `lens_registry` household cabinet drawer as `depreciation_assets`.

**Evidence shape:**
```yaml
id: asset:{userobject_id}
type: asset_depreciation_evidence
asset_name, asset_id, room_anchor
cost_basis, current_book_value, accumulated_depreciation
replacement_estimate, depreciation_method, status
placed_in_service_date, anchor_type, anchor_id
confidence: explicit
```

---

## Write Contract

All write modes:

| Field | Behavior |
|-------|---------|
| `dry_run=true` | Validates + previews, writes nothing. Overrides `confirm`. |
| `confirm=true` | Required to execute writes (except `dry_run`). |
| `idempotency_key` | Caller-supplied or auto-derived. Check `evidence.idempotency_key` in response. |
| `audit_note` | Appended to Grocy notes + Firefly transaction notes permanently. |
| `caller` | Capability-checked against `allowed_callers`. Unknown caller → `unauthorized`. |

---

## Depreciation Methods

| Method | Behavior |
|--------|---------|
| `straight_line` | `(cost_basis - salvage_value) / useful_life_months` per period |
| `manual_no_post` | Calculates schedule, all periods default `post_status: skipped`. Never posts to Firefly. |

---

## Common Field Reference

| Field | Type | Notes |
|-------|------|-------|
| `cost_basis` | decimal | Total cost to put in service: price + delivery + install + tax |
| `salvage_value` | decimal | Estimated value at end of life. Default 0. |
| `useful_life_months` | integer | Service life. Appliances: 60-120, HVAC: 180, roof: 240-360 |
| `placed_in_service_date` | YYYY-MM-DD | When depreciation starts |
| `replacement_cost_estimate` | decimal | Future replacement cost. Used for reserve calculation, not depreciation. |
| `room_anchor` | string | Used by Lens Bus for area_id/label matching |
| `grocy_product_id` | string | Grocy product numeric ID. Auto-populates `product:*` evidence ref. |
| `business_use_percent` | decimal | 0-100 |
| `household_use_percent` | decimal | 0-100 |
| `status` | enum | `active` \| `disposed` \| `sold` \| `donated` \| `scrapped` \| `lost` \| `retired` |

---

## Troubleshooting

| Code | Cause | Fix |
|------|-------|-----|
| `setup_required` | Grocy userentity schema not created | Run `depreciable_asset_firefly_setup` first |
| `already_posted` | Idempotency working correctly | Use `compensate` + re-post if original was wrong |
| `already_done` | Write succeeded in a prior call | Check `evidence` envelope for original result |
| `unauthorized` | `caller` not in `allowed_callers` | Call `tool_manifest` to inspect whitelist |
| `method_does_not_post` | Asset uses `manual_no_post` | Use `book_value` instead of posting |
| `book_value == cost_basis` | No periods posted yet | Correct for new asset |
| Schedule row count wrong | Bad date format | `placed_in_service_date` must be YYYY-MM-DD |

---

## Phase 2 Backlog

T9: Monthly scheduler automation  
T10: Batch post all due periods  
T11: Piggy bank reserve sync  
T12/T13: Project/room report views  
T14: Disposal with Firefly transaction post  
T15: Reconcile local schedule vs. Firefly  
