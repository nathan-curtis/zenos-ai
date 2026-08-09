# ZenOS Lens Bus

**Dispatcher:** `zen_dojotools_lens_dispatch` (`packages/zenos_ai/dojotools/dojotools_dispatcher.yaml`)  
**Registry:** `lens_registry` drawer in household cabinet  
**Bootstrap:** `zen_dojotools_manifest mode=bootstrap_stacks` (runs on HA start + daily 00:00:30)  
**Version:** 1.0.0  

---

## Overview

The Lens Bus enriches LLM context by fanning out typed anchors to registered providers and collecting optional evidence. Consumers pass anchors; providers return evidence items. All provider failures are soft — missing evidence is never a hard error.

```
Consumer (LLM / dojotools)
    ↓  anchors_json=[{"type":"label","id":"dining_room"},{"type":"transaction","id":"1447"}]
zen_dojotools_lens_dispatch
    ↓  routes each anchor type to providers that declared consumes: [label, ...]
zen_stack_firefly        → transaction_evidence
zen_stack_depreciation   → asset_depreciation_evidence
zen_stack_paperless      → document_evidence
zen_dojotools_library    → item_evidence
zen_sutra_wikijs         → article_evidence
zen_dojotools_media_manager  → track_evidence / album_evidence / ...
zen_stack_timer          → timer_evidence
zen_stack_alarms         → alarm_evidence
```

---

## Anchor Types

| Type | Description | Providers |
|------|-------------|-----------|
| `label` | HA label slug | library_catalog, depreciation_assets, paperless_ngx, wiki_js, media, firefly_iii |
| `area_id` | HA area ID | library_catalog, depreciation_assets, paperless_ngx, wiki_js, media |
| `person` | Person identifier | library_catalog, paperless_ngx, wiki_js, media, firefly_iii |
| `zone` | Zone identifier | paperless_ngx, wiki_js, media |
| `concept` | Abstract topic | wiki_js, media |
| `mood` / `activity` | Affective/activity anchors | media |
| `transaction` | Firefly III transaction ID | firefly_iii |
| `product` | Grocy product ID | depreciation_assets |
| `company` / `family` / `household` | Organizational anchors | (future providers) |

---

## Calling the Dispatcher

```yaml
action: script.zen_dojotools_lens_dispatch
data:
  anchors_json: '[{"type":"label","id":"dining_room"},{"type":"transaction","id":"1447"}]'
  consumer: my_tool
  timeout_ms: 1200
response_variable: lens_result
```

Response shape:
```yaml
status: success
evidence:          # flat list of all items from all providers
  - id: txn_1447
    type: transaction_evidence
    ...
  - id: asset:21
    type: asset_depreciation_evidence
    ...
provider_outputs:  # per-provider breakdown
  firefly_iii:
    evidence: [...]
    count: 1
  depreciation_assets:
    evidence: [...]
    count: 1
```

---

## Registry

Stored in `lens_registry` drawer of the household cabinet. Written by each stack's `register` mode. Read by the dispatcher to route anchors to the right providers.

**Registry entry shape:**
```yaml
provider_key:
  provider: depreciation_assets
  tool: zen_stack_depreciation
  configured_by: zen_stack_depreciation
  department: stacks
  consumes: [label, area_id, product]
  returns: [asset_depreciation_evidence]
  failure_policy: soft
  content_policy: standard
  risk_class: read_only
  enabled: true
  status: available
```

**Updating the registry** — run `zen_dojotools_manifest mode=bootstrap_stacks`. It re-registers all discovered `zen_stack_*` scripts (upserts — existing entries are overwritten with current `tool_manifest` values).

---

## Registered Providers

| Provider key | Tool | Anchor types | Returns |
|-------------|------|-------------|---------|
| `firefly_iii` | `zen_stack_firefly` | label, person, transaction | transaction_evidence |
| `depreciation_assets` | `zen_stack_depreciation` | label, area_id, product | asset_depreciation_evidence |
| `library_catalog` | `zen_dojotools_library` | person, label, area_id | item_evidence |
| `media` | `zen_dojotools_media_manager` | label, person, area_id, zone, concept, mood, activity | track/album/artist/playlist/radio/podcast/audiobook/queue/playback_target evidence |
| `paperless_ngx` | `zen_stack_paperless` | label, person, area_id, zone | document_evidence |
| `wiki_js` | `zen_sutra_wikijs` | label, concept, area_id, zone, person | article_evidence |
| `timer` | `zen_stack_timer` | label (room-slug) | timer_evidence — every `timer.*` entity carrying that room's label, with purpose (`room_timer`/`checking_timer`/`nightlight_timer`/`control_burnout`/etc), state, remaining, duration |
| `alarms` | `zen_stack_alarms` | area_id | alarm_evidence — every `sensor.*_next_alarm` entity assigned to that area (Alexa/Echo, fire tablets, far-field-voice TVs), next-alarm time + seconds_until. Personal devices with no area assigned (most phones/watches) don't surface here — see `zen_dojotools_timekeeper mode=alarms` for the full house-wide list including those. |

---

## Provider Contract

Every `zen_stack_*` script must implement:

| Mode | Purpose |
|------|---------|
| `tool_manifest` | Self-description: `provider_key`, `consumes`, `returns`, `register_mode`, `failure_policy` |
| `register` | Write entry into `lens_registry` household cabinet drawer |
| `unregister` | Remove entry from `lens_registry` |
| `health` | Check dependencies (API reachability, etc.) |
| `inspect` | Show provider capabilities and contract |
| `stacks_by_anchor` | Main dispatch: accept `anchor_type` + `anchor_ids`, return `evidence` list |

Providers fail soft: `continue_on_error: true` on all dispatcher calls. Missing evidence = empty list.

---

## Bootstrap Behavior

`bootstrap_stacks` (in `zen_dojotools_manifest`):

1. Discovers all `zen_stack_*` scripts in HA entity registry
2. Calls `mode=tool_manifest` on each to get `provider_key` and `register_mode`
3. If `provider_key` already in registry → re-registers (upserts) with current manifest
4. If new → registers for first time
5. Tracks `registered` (new), `already_registered` (upserted), `skipped` (on skip list), `no_register_mode` (sutras/internal)

Runs on `homeassistant_started` + daily `00:00:30`.

---

## Adding a New Provider

1. Create `zen_stack_{name}.yaml` in the appropriate plugin folder
2. Implement all required modes (tool_manifest, register, unregister, health, inspect, stacks_by_anchor)
3. Declare `consumes: [anchor_type, ...]` in `tool_manifest`
4. Add the new anchor type(s) to `_anchor_groups` in `dojotools_dispatcher.yaml` if not already present
5. Run `bootstrap_stacks` — new provider is auto-discovered and registered
6. No changes needed to the dispatcher's provider routing loop (it reads from `lens_registry`)
