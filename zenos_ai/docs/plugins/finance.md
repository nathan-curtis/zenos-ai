# ZenOS Finance Tool

**File:** `packages/zenos_ai/plugins/firefly_iii/firefly_iii.yaml`  
**Scripts:** `zen_dojotools_finance` (MCP-exposed), `zen_stack_firefly`, `zen_root_firefly`  
**Version:** 2.4.0  

---

## Overview

Authoritative household ledger tool. All money questions, all Firefly III operations, and all asset depreciation routes through `zen_dojotools_finance`. The LLM never calls `zen_root_firefly` or `zen_stack_firefly` directly.

```
zen_dojotools_finance    MCP surface (mode=run case=...)
    ├── zen_root_firefly         wire protocol — Firefly III REST
    └── zen_codex_finance_depreciation   domain logic — asset depreciation
```

---

## Scripts

### zen_dojotools_finance
MCP-exposed. All modes accessed via `mode=run case={case_name}`.

**Read cases:**
`health`, `catalog`, `account_balance`, `transactions_list`, `transactions_search`, `transaction_get`, `budget_overview`, `bills_list`, `bill_status`, `piggy_banks_list`, `piggybank_status`, `category_spend`, `net_worth_snapshot`, `liability_summary`

**Write cases** (require `confirm=true`):
`account_create`, `account_update`, `transaction_add`, `transaction_update`, `transfer`, `budget_limit_set`, `piggybank_update`, `category_create`

**Compound cases:**
`spend_summary`, `reconcile_prepare`, `transaction_normalize`, `project_account_status`

**Depreciation cases** (routed to `zen_codex_finance_depreciation`):
`depreciable_asset_create`, `depreciable_asset_get`, `depreciable_asset_update`, `depreciable_asset_list`, `depreciable_asset_calculate_schedule`, `depreciable_asset_book_value`, `depreciable_asset_post_depreciation_period`, `depreciable_asset_replacement_plan`, `depreciable_asset_dispose`, `depreciable_asset_report`, `depreciable_asset_firefly_setup`

**Special modes:** `mode=help` (case reference), `mode=index` (live capability map), `mode=inspect` (feature health), `mode=bootstrap_codices` (re-register all codices)

### zen_root_firefly
Internal only. Raw REST dispatcher (GET, POST, PUT, DELETE). Never call directly.

### zen_stack_firefly
Lens Bus provider. Not MCP-exposed. Handles `label`, `person`, and `transaction` anchors. See [lens_bus.md](lens_bus.md).

---

## Codex Architecture

`zen_dojotools_finance` includes a **catch-all codex dispatch slot** — the last choose block in the dispatch sequence. When no known case matches, it reads `codex_registry` from the household cabinet and routes to the registered codex that owns that case.

Adding a new codex:
1. Create the codex script implementing `tool_manifest`, `register`, `help`, and all feature modes
2. Run `bootstrap_codices` — auto-discovered and registered in `codex_registry`
3. Add one branch to the catch-all choose block in `firefly_iii.yaml`

Currently registered codices: `zen_codex_finance_depreciation` (feature_key: `depreciation`)

---

## Key Fields

| Field | Notes |
|-------|-------|
| `mode` | `run` \| `help` \| `index` \| `inspect` \| `bootstrap_codices` \| `tool_manifest` |
| `case` | Case name within `mode=run` |
| `item_id` | Entity ID for get/update operations (transaction ID, account ID, etc.) |
| `item` | Search string for search operations |
| `confirm` | Required `true` for all write operations |
| `dry_run` | Preview without writing (depreciation modes) |
| `include_evidence` | `book_value` only — resolves `evidence_refs` inline via Lens Bus |
| `posting_mode` | `report_only` (default) \| `summary_post` — for depreciation posting |
| `audit_note` | Stored permanently in Grocy notes + Firefly transaction notes |
| `idempotency_key` | Caller-supplied or auto-derived for depreciation write modes |

---

## Firefly III Connection

- URL: `input_text.firefly_url` (e.g. `http://firefly.local:8080`)
- Auth: `!secret firefly_token`
- `zen_dojotools_finance mode=run case=health` checks reachability

---

## zen_stack_firefly — Transaction Anchor

`zen_stack_firefly` was extended to handle `transaction` anchor type. Given a Firefly transaction ID, returns a `transaction_evidence` item with date, amount, description, category, tags, source, destination.

Used by `book_value include_evidence=true` to resolve `transaction:*` entries in `evidence_refs`.

The `transaction` anchor type is also registered in `lens_registry` and supported by `zen_dojotools_lens_dispatch` — consumers can pass `{"type":"transaction","id":"1447"}` in `anchors_json`.
