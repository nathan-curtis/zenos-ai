# Kitchen Sync — Mealie ↔ Grocy Food Sync

*ZenOS-AI Plugin — v4.5.0*

---

## Overview

`kitchen_sync` is the food catalog sync orchestrator for ZenOS-AI. It performs a governed one-way sync between **Mealie** (recipe manager) and **Grocy** (pantry/inventory manager), reconciling food items across both systems.

One direction per run. Preview before applying. Bounded by limit.

**Current status:** `mealie_to_grocy` is implemented. `grocy_to_mealie` direction is not yet implemented — passing it has no effect.

---

## Script

**`script.mealie_grocy_sync_orchestrator`**

---

## Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `direction` | select | `mealie_to_grocy` | `mealie_to_grocy` only (grocy_to_mealie not yet implemented) |
| `mode` | select | `preview` | `preview` = dry run, returns what would change. `apply` = execute. |
| `limit` | number | `10` | Max items to process per run (1–100) |
| `confirmation` | boolean | `false` | Required `true` to execute an `apply` run |
| `default_location` | text | `Pantry` | Default storage location name for new Grocy products. Resolved to Grocy location ID at runtime. |
| `default_unit` | text | `each` | Default unit of measure for new products. Resolved to Grocy unit ID at runtime. |
| `per_page` | number | `50` | Mealie API page size for food list fetches |
| `max_pages` | number | `50` | Max pages to fetch from Mealie (guards against runaway pagination) |

---

## How It Works

**Mealie → Grocy:**

1. Pages through the Mealie foods list via `zen_dojotools_mealie_helper` (`foods_list`)
2. Food names are trimmed and empty names are skipped
3. For each food: looks up the name in Grocy via `zen_dojotools_grocy_advanced`
4. Classifies as `matched_exact`, `matched_ambiguous`, or `would_create`
5. In `apply` mode: creates missing products in Grocy using `zen_dojotools_grocy_advanced`. `default_location` and `default_unit` are resolved to Grocy IDs before the create call.

**Preview mode** always runs safely — no writes, returns the full diff of what would be created or matched.

---

## Response

```json
{
  "tool": "Zen DojoTools Food Sync Orchestrator",
  "direction": "mealie_to_grocy",
  "mode": "preview",
  "evaluated": 25,
  "matched_exact": ["chicken breast", "olive oil"],
  "matched_ambiguous": ["milk"],
  "would_create": ["sumac", "preserved lemon"],
  "created": [],
  "errors": []
}
```

| Field | Description |
|-------|-------------|
| `evaluated` | Total food items examined this run |
| `matched_exact` | Foods found in Grocy by exact name match |
| `matched_ambiguous` | Foods with multiple partial Grocy matches — not auto-created |
| `would_create` | Foods not found in Grocy (preview) or created (apply) |
| `created` | Products actually written to Grocy in `apply` mode |
| `errors` | Any items that failed during processing with reason |

Ambiguous matches are never auto-created. Resolve them manually in Grocy or Mealie, then re-run.

---

## Example: Dry run

```yaml
direction: mealie_to_grocy
mode: preview
limit: 25
```

## Example: Apply sync (first 10 items)

```yaml
direction: mealie_to_grocy
mode: apply
limit: 10
confirmation: true
default_location: Pantry
default_unit: each
```

---

## Requirements

- Mealie integration configured in HA with REST API access (`input_text.mealie_url` set via HA UI)
- Grocy integration configured in HA with REST API access (`input_text.grocy_url` set via HA UI)
- `zen_dojotools_mealie_helper` and `zen_dojotools_grocy_advanced` scripts installed

---

## Related

- `packages/zenos_ai/plugins/mealie/mealie.yaml` — Mealie REST helper
- `packages/zenos_ai/plugins/grocy/grocy.yaml` — Grocy REST helper (`zen_dojotools_inventory`)
- `packages/zenos_ai/plugins/grocy/readme.md` — Grocy plugin docs
