# ZenOS-AI Firefly III Finance Component

**Version:** 1.0.0  
**Package:** `packages/zenos_ai/plugins/firefly_iii/firefly_iii.yaml`  
**Primary script:** `zen_dojotools_finance`  
**Internal REST dispatcher:** `zen_sutra_firefly`  
**Lens Bus stack provider:** `zen_stack_firefly` (provider_key: `firefly_iii`)

> **Lens Bus (2026.7.0):** `zen_stack_firefly` is now a fully registered Lens Bus stack provider. It was the first ZenOS plugin to use bootstrap auto-registration — on every HA boot, `zen_dojotools_manifest mode=bootstrap_stacks` discovers and registers it automatically. See [Lens Bus Auto-Registration](lens_bus_autoreg.md).

> **v.next:** Deeper refactor queued — expanded endpoint coverage and cabinet-backed config.

---

## Overview

Firefly III is the household finance ledger for ZenOS-AI. This is its first pull into the ZenOS toolset. The plugin treats Firefly III as the authoritative source for accounts, transactions, budgets, bills, and savings goals. ZenOS-AI does not maintain its own shadow of financial state — everything reads from and writes to Firefly III directly.

In practical terms:

* Asset, expense, revenue, liability, and cash accounts live in Firefly III.
* Transactions (withdrawals, deposits, transfers) are logged there.
* Budgets, bills, and piggy banks (savings goals) are managed and queried there.
* `zen_dojotools_finance` is the primary script for all AI and MCP use.
* `zen_sutra_firefly` is the internal raw REST dispatcher — GET, POST, PUT, DELETE — and should stay internal. Call `zen_dojotools_finance` instead.

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
| `transactions_search` | Full-text search across transaction descriptions |
| `budget_overview` | All budgets — limit, spent, remaining for the current or specified period |
| `bills_list` | All recurring bills — next due date, paid status, overdue flag |
| `piggy_banks_list` | Savings goals — name, current, target, and percentage complete |
| `category_spend` | Total spending for a category over a date range |
| `transaction_add` | Log an expense (withdrawal), income (deposit), or transfer |
| `transaction_update` | Fix an existing transaction — category, description, amount, or date |
| `transfer` | Move money between two asset accounts |
| `spend_summary` | Compound briefing — net worth + budget snapshot + upcoming bills in one call. Writes snapshot to household cabinet. |

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

## REST Dispatcher — zen_sutra_firefly

`zen_sutra_firefly` is the internal GET/POST/PUT/DELETE broker. It normalizes the `endpoint` field — strips leading `/api/v1/` if present — and appends pagination parameters to GET calls. It returns:

```json
{
  "status": "success|error",
  "http_status": 200,
  "result": { "content": { ... } }
}
```

HTTP 200, 201, and 204 map to `success`. All other codes map to `error`. Content is parsed from JSON string to a mapping before returning.

Do not call `zen_sutra_firefly` directly from AI or MCP integrations. It has no case routing or name resolution. Use `zen_dojotools_finance`.

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

4. Expose `zen_dojotools_finance` to the conversation agent. Keep `zen_sutra_firefly` internal.

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

## Source Notes

This page is derived from:

* `packages/zenos_ai/plugins/firefly_iii/firefly_iii.yaml`
