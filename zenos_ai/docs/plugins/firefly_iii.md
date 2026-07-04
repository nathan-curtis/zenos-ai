# ZenOS-AI Firefly III Finance Component

**Version:** 2.3.0  
**Package:** `packages/zenos_ai/plugins/firefly_iii/firefly_iii.yaml`  
**Primary script:** `zen_dojotools_finance`  
**Internal REST dispatcher:** `zen_root_firefly` (renamed from `zen_sutra_firefly` in 2.3.0)  
**Lens Bus stack provider:** `zen_stack_firefly` (provider_key: `firefly_iii`)  
**Codex tier:** `zen_codex_finance_depreciation`, `zen_codex_finance_cogs` — see [Codex Tier](#codex-tier)

> **Lens Bus (2026.7.0):** `zen_stack_firefly` is now a fully registered Lens Bus stack provider. It was the first ZenOS plugin to use bootstrap auto-registration — on every HA boot, `zen_dojotools_manifest mode=bootstrap_stacks` discovers and registers it automatically. See [Lens Bus Auto-Registration](lens_bus_autoreg.md).

---

## Overview

Firefly III is the household finance ledger for ZenOS-AI. This is its first pull into the ZenOS toolset. The plugin treats Firefly III as the authoritative source for accounts, transactions, budgets, bills, and savings goals. ZenOS-AI does not maintain its own shadow of financial state — everything reads from and writes to Firefly III directly.

In practical terms:

* Asset, expense, revenue, liability, and cash accounts live in Firefly III.
* Transactions (withdrawals, deposits, transfers) are logged there.
* Budgets, bills, and piggy banks (savings goals) are managed and queried there.
* `zen_dojotools_finance` is the primary script for all AI and MCP use.
* `zen_root_firefly` is the internal raw REST dispatcher — GET, POST, PUT, DELETE — and should stay internal. Call `zen_dojotools_finance` instead.

---

## Why This Component Is Different

Most ZenOS tools own their domain state in FileCabinet drawers. Finance is different: Firefly III is both the external application and the single authority for money. ZenOS does not duplicate balances or maintain its own ledger.

The `spend_summary` case is the exception — it writes a `finance_daily_snapshot` to the household cabinet as a read cache for morning briefings. That snapshot is derived from Firefly; Firefly remains the source of truth.

---

## Core Finance Cases

Use `mode=run` and specify a `case`. Use `mode=help` to get the full catalog back from the tool at runtime.

| Case | Intent |
|------|--------|
| `summary` | Net worth, income, expenses, and left-to-spend for a date range |
| `accounts_list` | All accounts with current balances; filter by account type |
| `account_get` | Single account detail by name or ID |
| `transactions_list` | Recent transactions; filter by account, date range, and type |
| `transactions_search` | Full-text search across transaction descriptions. Also supports tag-based queries (`tag_is:"..."`) — used internally by Lens providers like Firefly's own tag-anchored evidence lookups. |
| `budget_overview` | All budgets — limit, spent, remaining for the current or specified period |
| `bills_list` | All recurring bills — next due date, paid status, overdue flag |
| `piggy_banks_list` | Savings goals — name, current, target, and percentage complete |
| `category_spend` | Total spending for a category over a date range |
| `transaction_add` | Log an expense (withdrawal), income (deposit), or transfer |
| `transaction_update` | Fix an existing transaction — category, description, amount, or date |
| `transfer` | Move money between two asset accounts |
| `spend_summary` | Compound briefing — net worth + budget snapshot + upcoming bills in one call. Writes snapshot to household cabinet. |
| `account_create` | Create a new account (asset, expense, revenue, liability, cash). Asset-type accounts now get `account_role: defaultAsset` set automatically. |
| `account_update` | Update an existing account's fields (name, type, currency, notes). |
| `piggybank_create` | Create a new savings goal (piggy bank) with target amount and optional target date. |
| `piggybank_update` | Update a piggy bank's target or add/remove funds. |
| `budget_create` | Create a new budget category. |
| `budget_limit_set` | Set the spend limit for a budget in a given period. |
| `bill_create` | Create a recurring bill. |
| `bill_status` | Get current status of a bill (next due, paid/overdue). `next_expected_match` is now guarded against Firefly returning a literal `None`/`none` string — previously this could misreport a bill as overdue. |
| `net_worth_snapshot` | Point-in-time net worth across all asset and liability accounts. |
| `liability_summary` | Summary of all liability accounts — balance, interest rate, next payment. |
| `reconcile_prepare` | Pull all unreconciled transactions for a specific account to prepare for reconciliation. |
| `transaction_normalize` | Fix malformed or miscategorized transactions in bulk (recategorize, retag, move account). |
| `project_account_status` | Status of a project/tracking account — balance, linked transactions, progress against target. |
| `transaction_link` | Create a Firefly link (related, refund, paid by) between two transactions. Now confirm-gated — requires `confirm: true` before writing (previously wrote unconditionally). |
| `catalog` | Return the tool's full case catalog with brief descriptions — use for help/routing. |
| `transaction_get` | Fetch a single transaction by `item_id`. Flattens Firefly's split-transaction structure into a flat result. |
| `budget_summary` | Budget-vs-actual for a period: allocated, spent, remaining, `pct_used`, `days_left_in_period`. Filters out budgets with no limit set. Feeds the `finance_budget` KFC component. |
| `finance_rollup` | Compound call — bundles `budget_summary` + `bills_list` + `net_worth_snapshot` in one response. Feeds the `finance_manager` KFC component. |
| `category_create` | Create a new budget category. Idempotent — checks for an existing category with the same name before POSTing. Confirm-gated. |
| `index` | Lists all known cases plus every registered codex — self-discovery/routing aid. |
| `inspect` | Health-checks Firefly reachability plus each registered codex script. |
| `bootstrap_codices` | Scans the known `zen_codex_finance_*` scripts and writes/updates the `codex_registry` entry in FileCabinet. Run once (or after adding a new codex) to make it discoverable via `index`/`report`/`inspect`. |
| `kfc_manifest` | Returns KFC component definitions (`finance_budget`, `finance_bills`, `finance_networth`, `finance_manager`) for the daily-briefing summarizer pipeline. |
| `report` | Composable CFO-style dashboard. Auto-discovers report pieces from every registered codex (any codex mode ending in `_report`) plus the native `rollup`. Supports `include=`/`exclude=` filters to scope which pieces run. |

---

## Key Fields

| Field | Required by | Notes |
|-------|-------------|-------|
| `mode` | all | `run`, `help`, or `tool_manifest` |
| `case` | `run` | See case table above |
| `item` | `account_get`, `transactions_search`, `category_spend` | Name — resolved to ID via Firefly autocomplete |
| `item_id` | `transaction_update` | Explicit Firefly integer ID — skips name resolution |
| `amount` | `transaction_add`, `transfer` | Decimal amount |
| `description` | `transaction_add` | Transaction description |
| `source_account` | `transaction_add`, `transfer` | Source account name |
| `dest_account` | `transaction_add`, `transfer` | Destination account name |
| `category` | `transaction_add` | Category name — resolved to ID |
| `budget` | `transaction_add` | Budget name — resolved to ID |
| `tags` | `transaction_add`, `transfer` | Comma-separated tag string |
| `date` | `transaction_add`, `transfer`, `transaction_update` | `YYYY-MM-DD`; defaults to today |
| `start_date` / `end_date` | range cases | `YYYY-MM-DD`; default current month |
| `account_type` | `accounts_list` | `all`, `asset`, `expense`, `revenue`, `liability`, `cash` |
| `transaction_type` | `transaction_add` | `withdrawal`, `deposit`, or `transfer`; default `withdrawal` |
| `page` / `per_page` | paginated cases | Pagination controls |
| `show_trace` | any | Boolean; debug output |
| `label` | `transaction_add`, `transfer`, create cases | Firefly tag string. Applied as a tag on the created/updated transaction. Used for cross-referencing by external systems. |
| `external_ref` | `transaction_add`, `transaction_update` | External system reference string (e.g., Grocy product ID, Paperless document ID). Stored in Firefly's `external_url` or notes field for cross-linking. |

### Name resolution

`account_get`, `transactions_list`, `category_spend`, `transaction_add`, and `transfer` all resolve human names to Firefly IDs using the autocomplete endpoint before submitting. Exact case-insensitive match is required. If no match is found, the tool returns an error with candidates from the autocomplete response.

---

## spend_summary

`spend_summary` is the recommended compound call for a morning financial briefing. It fires three parallel Firefly requests — `summary/basic`, `budgets`, and `bills` — and merges the results.

It also writes a snapshot to the household FileCabinet:

```yaml
key: finance_daily_snapshot
value: {summary, budgets, bills_due, as_of}
```

This allows other ZenOS tools to read the latest finance snapshot without a live Firefly call. If the household cabinet entity is not resolvable, the snapshot write is skipped and `snapshot_written: false` is returned.

```yaml
zen_dojotools_finance:
  mode: run
  case: spend_summary
```

---

## Codex Tier

Beyond the core cases above, `zen_dojotools_finance` supports **codices** — optional domain-logic modules that plug into the finance tool without living inside `firefly_iii.yaml` itself. A codex is its own sibling YAML file, registered into the household cabinet's `codex_registry` drawer, and dispatched by `zen_dojotools_finance` the same way a core case is.

**Why codices exist:** `firefly_iii.yaml` covers the generic ledger surface (accounts, transactions, budgets, bills). Depreciation and COGS tracking are domain-specific enough that they don't belong in the generic file, but still need to feel like native `zen_dojotools_finance` cases to a caller. The codex pattern gets both: a clean split at the file level, and one unified entry point at the call level.

**Registering a codex:** run `zen_dojotools_finance mode=run case=bootstrap_codices` after adding a new `zen_codex_finance_*` script. It scans the known list of codex scripts and writes/updates their `codex_registry` entries. Use `case=index` to see everything currently registered, `case=inspect` to health-check Firefly plus every registered codex, and `case=report` for a composed dashboard that automatically includes any codex mode ending in `_report`.

### `zen_codex_finance_depreciation` (v1.0.0)

Household asset depreciation. Grocy (`depreciable_asset` userentity) is the system of record for the asset itself; Firefly III is the money ledger; this codex has **zero cabinet footprint** for asset data — schedules are calculated on demand and posting history is derived from Firefly transactions tagged with an idempotency key.

Key modes:

| Mode | Does |
|------|------|
| `calculate_schedule` | Straight-line, declining-balance, or double-declining-balance schedule, with SL/DDB crossover logic |
| `book_value` | Current book value, derived from parsed Firefly posting history (not stored anywhere) |
| `post_depreciation_period` | Idempotent Firefly posting for one period — optionally split into business-use / household-use transactions |
| `asset_allocation_*` | Date-ranged business-use percentage history, for assets whose business/personal split changes over time |
| `dispose` / `dispose_with_post` | Disposal gain/loss calculation, with an optional posting variant |
| `replacement_plan` | Piggy-bank reserve projection for eventual replacement |
| `warranty_check` | Warranty expiration status |
| `cpa_summary` / `report` | CPA-ready summary / composable report piece for `finance` `case=report` |

Setup: `zen_dojotools_finance mode=run case=depreciable_asset_firefly_setup` creates the Grocy schema and Firefly categories. See `mode=help` on the codex for the full workflow sequence. Codex tickets: #10207 (epic), #10208-#10215 (T1-T8).

### `zen_codex_finance_cogs` (v1.2.0)

Cost-of-goods-sold auto-posting. Watches Grocy stock consumption tagged `ha_labels=cogs_tracked`; when a tagged product is consumed, posts a Firefly withdrawal (`quantity × unit_price`) automatically. Cabinet footprint: one drawer (`cogs_accounting` — `enabled`, `default_source_account`, `default_category`), no per-item persistent state — every consume event posts live.

Key modes:

| Mode | Does |
|------|------|
| `cogs_configure` | Write the `cogs_accounting` drawer — pick real Firefly accounts/category for COGS postings |
| `cogs_post` | Confirm-gated, idempotency-keyed posting for one consumption event. Fails soft if Firefly is unreachable |
| `cogs_summary` / `cogs_reconcile` / `cogs_report` | Period rollups comparing Grocy `stock_log` against Firefly postings |

---

## Example Calls

```yaml
# Net worth snapshot for the current month
zen_dojotools_finance:
  mode: run
  case: summary

# Log a grocery withdrawal
zen_dojotools_finance:
  mode: run
  case: transaction_add
  description: "Weekly groceries"
  amount: 134.52
  source_account: "Checking"
  category: "Groceries"
  transaction_type: withdrawal

# Budget status
zen_dojotools_finance:
  mode: run
  case: budget_overview

# Upcoming bills
zen_dojotools_finance:
  mode: run
  case: bills_list

# Search transactions by merchant
zen_dojotools_finance:
  mode: run
  case: transactions_search
  item: "HEB"
  start_date: "2026-06-01"
```

---

## REST Dispatcher — zen_root_firefly

`zen_root_firefly` is the internal GET/POST/PUT/DELETE broker. It normalizes the `endpoint` field — strips leading `/api/v1/` if present — and appends pagination parameters to GET calls. It returns:

```json
{
  "status": "success|error",
  "http_status": 200,
  "result": { "content": { ... } }
}
```

HTTP 200, 201, and 204 map to `success`. All other codes map to `error`. Content is parsed from JSON string to a mapping before returning.

Do not call `zen_root_firefly` directly from AI or MCP integrations. It has no case routing or name resolution. Use `zen_dojotools_finance`.

---

## First-Time Setup

1. Generate a Personal Access Token in Firefly III:

   **Firefly III → Profile → OAuth → Personal Access Tokens → Create New Token**

2. Add to `secrets.yaml`. Include the `Bearer ` prefix in the value:

   ```yaml
   firefly_iii_bearer: "Bearer <your-personal-access-token>"
   ```

3. In Home Assistant, create a helper or set the input text entity `input_text.firefly_url`:

   **Settings → Helpers → Add Helper → Text**  
   Name it `Firefly III URL`, entity ID `input_text.firefly_url`, and set the value to:

   ```text
   https://<your-firefly-host>
   ```

   Do not add a trailing slash. Use HTTPS. HTTP will cause POST-to-GET redirect on writes, which silently breaks all transaction logging.

4. Expose `zen_dojotools_finance` to the conversation agent. Keep `zen_root_firefly` internal.

---

## n8n Bank Feed Integration (Optional — Phase 2)

The YAML notes a planned n8n integration for bank feed → Firefly sync. n8n pushes transactions to Firefly via `POST /api/v1/transactions`. HA-side webhook registration is Phase 2 and not yet implemented.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Reads work, writes fail | `firefly_url` is HTTP or proxy redirects POST→GET | Set `input_text.firefly_url` to HTTPS directly |
| `account_get` returns "No account found" | Name doesn't match exactly | Check candidates list in the error; use Firefly autocomplete spelling |
| `transaction_add` logs without a category | Category name not found or mismatched | Confirm category exists in Firefly; check exact spelling |
| `spend_summary` returns `snapshot_written: false` | `sensor.zen_default_household_cabinet_resolved` not available | Household cabinet not configured; snapshot skipped, briefing data still returned |
| 401 errors on all calls | Bearer token missing or expired | Regenerate Personal Access Token in Firefly III and update `secrets.yaml` |

---

## Version History

| Version | Change |
|---------|--------|
| v2.3.0 | Codex tier introduced: `zen_codex_finance_depreciation` (asset depreciation) and `zen_codex_finance_cogs` (COGS auto-posting), dispatched via new `index`/`inspect`/`bootstrap_codices`/`report`/`kfc_manifest` cases. New cases: `transaction_get`, `budget_summary`, `finance_rollup`, `category_create`. `account_create` sets `account_role: defaultAsset` on asset accounts. `transaction_link` is now confirm-gated. `bill_status`/`bills_list`/`spend_summary` guard against Firefly returning literal `None`/`none` strings. `transactions_search` supports tag-based queries. REST dispatcher renamed `zen_sutra_firefly` → `zen_root_firefly`. |
| v2.2.0 | Complete rebuild. `zen_stack_firefly` Lens Bus stack provider with auto-registration (`register_mode: register`). r-only security gate on the stack provider. 15 new cases: `account_create/update`, `piggybank_create/update`, `budget_create/budget_limit_set`, `bill_create/status`, `net_worth_snapshot`, `liability_summary`, `reconcile_prepare`, `transaction_normalize`, `project_account_status`, `transaction_link`, `catalog`. New fields: `label` (Firefly tag), `external_ref` (Grocy/Paperless cross-link). REST dispatcher: PUT and DELETE commands added. Auto-registration pattern first established here. |
| v1.0.0 | Initial release. `zen_dojotools_finance` with 13 core cases. GET and POST REST. `zen_root_firefly` internal dispatcher. `spend_summary` household cabinet snapshot. |

---

## Source Notes

This page is derived from:

* `packages/zenos_ai/plugins/firefly_iii/firefly_iii.yaml`
* `packages/zenos_ai/plugins/firefly_iii/zen_codex_finance_depreciation.yaml`
* `packages/zenos_ai/plugins/firefly_iii/zen_codex_finance_cogs.yaml`
