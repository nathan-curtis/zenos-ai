# Zen DojoTools Labels — 2026.6.0 'Clue'

*Create, read, update, delete, and assign labels in the Home Assistant label index*

---

## Overview

`zen_dojotools_labels` is the core primitive for label management in ZenOS-AI. It is how the system creates, inspects, and assigns the labels that power the HyperIndex, KFC components, and cabinet resolution.

Labels are the connective tissue of the system. The Scheduler, Ninja Summarizer, and HyperIndex all work by traversing labels — not by reading hardcoded entity lists. If a label isn't right, the index finds the wrong things (or nothing).

This script is **MCP-exposed**. Friday can read the label index and create or update labels. Write operations (`create`, `update`, `delete`, `reset`) are gated behind `confirm: true` — they require an explicit operator confirmation before executing.

---

## Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `action_type` | select | `read` | `create`, `read`, `update`, `delete`, `tag`, `untag`, `reset` |
| `label_list` | list of text | `[]` | Label names (case-insensitive). Required for create, update, delete, tag, untag |
| `target_entities` | list of entity_ids | `[]` | Entities to tag/untag. Required for tag and untag |
| `new_description` | text | — | Description to set on the label (create and update) |
| `new_icon` | text | — | MDI icon slug (e.g. `mdi:water`). Omit to leave unset (create and update) |
| `new_color` | text | — | HA label color. Omit to use `primary` (create and update) |
| `force_rewrite` | boolean | `false` | `update` only — skip the REST pre-read and write only the fields you pass; unspecified fields (icon, color, description) are cleared rather than preserved |
| `confirm` | boolean | `false` | Required `true` for create, update, delete, and reset |

---

## Actions

### `read`

Returns the label index. Three modes depending on inputs:

| Inputs | Returns |
|---|---|
| Nothing | Full sorted label index |
| `label_list` only | Label details (description) for each label |
| `target_entities` only | Labels assigned to each entity |
| Both | Both of the above |

```json
{
  "status": "ok",
  "action": "read_index",
  "target": "*",
  "index": ["zen_cabinet", "zen_dojo_cabinet", "zen_kata_cabinet", ...]
}
```

Use this first before any write operation to see what already exists.

---

### `create`

Creates new labels in the HA label index. Requires `confirm: true`.

Without confirmation, returns a `status: start` preview showing which labels would be created and which already exist — useful for dry-running before committing.

Accepts `new_description`, `new_icon`, and `new_color` to stamp meaning on the label at creation time. `new_description` is recommended — it travels inline with every entity in Inspect output and is the primary semantic token for that label.

```yaml
action_type: create
label_list:
  - water
  - irrigation
  - pool
new_description: "Domain label for all water-related entities; label:water."
confirm: true
```

```json
{
  "status": "ok",
  "action": "create",
  "created_labels": ["water", "irrigation", "pool"],
  "already_existed": [],
  "message": "Create Action Complete."
}
```

---

### `update`

Updates an existing label's metadata: description, icon, and/or color. Requires `confirm: true`.

HA does not support in-place label mutation — update is implemented as delete → recreate → retag. All entity assignments are preserved automatically; the label returns with the same slug and all the same entities tagged.

**Patch semantics (default):** Before recreating, `update` performs a REST pre-read to fetch the existing `icon`, `color`, and `description`. Fields you don't specify are carried forward from the existing label. Pass `force_rewrite: true` to skip the pre-read — unspecified fields will be cleared to HA defaults.

Without confirmation, returns a `status: start` preview showing what would change.

```yaml
action_type: update
label_list:
  - water
new_description: "Domain label for all water-related entities; label:water. Includes spa, irrigation, and leak sensors."
confirm: true
```

```json
{
  "status": "ok",
  "action": "update",
  "updated_labels": ["water"],
  "preserved_from_existing": {"icon": "mdi:water", "color": "blue"},
  "message": "Update Action Complete."
}
```

> **Note:** `update` is non-destructive on entity assignments — retag happens automatically. You do not need to re-run `tag` after updating a label.

---

### `delete`

Removes labels from the HA label index. Requires `confirm: true`.

> **Warning:** Deleting a label removes it from all entities that carry it. The HyperIndex will no longer find those entities via that label. KFC components that reference the label will stop returning data.

---

### `tag`

Applies one or more labels to one or more entities. No confirmation required.

HA handles the fan-out — all labels are applied to all entities in a single call.

```yaml
action_type: tag
label_list:
  - water
  - irrigation
target_entities:
  - sensor.garden_moisture_zone_1
  - sensor.garden_moisture_zone_2
  - sensor.irrigation_valve_front
  - binary_sensor.irrigation_running
```

---

### `untag`

Removes one or more labels from one or more entities. No confirmation required.

---

### `reset`

Wipes all entity assignments from every `zen_` label. Labels survive — only the assignments are cleared. Use this when label IDs are correct but entity tagging is wrong or missing (for example, after a partial install or a label migration gone sideways).

`confirm: true` is required. The script will refuse without it.

After reset, `zen_resolver_refresh` is fired automatically. Flynn re-engages on the next `zen_label_health` state change and re-assigns via Gate 1.

```yaml
action_type: reset
confirm: true
```

```json
{
  "status": "ok",
  "action": "reset",
  "labels_cleared": 16,
  "message": "Soft reset complete. 16 zen_ label(s) cleared. All assignments wiped. zen_resolver_refresh fired — Flynn will re-assign on next label health check."
}
```

> **This is the soft reset.** For the nuclear option (delete labels entirely and trigger full Flynn rebuild), use `script.zen_admintools_reset_labels`.

---

## Scope-Change Pattern

To move entities from one label scope to another (e.g. reassigning a camera from `interior` to `exterior`):

```yaml
# 1. Remove from old scope
action_type: untag
label_list: [interior]
target_entities: [camera.hallway]

# 2. Add to new scope
action_type: tag
label_list: [exterior]
target_entities: [camera.hallway]
```

No confirm required for tag/untag. Changes take effect on the next trigger cycle — no redeploy or reload needed.

## Footguns

- **`delete`** is permanent and drops ALL entity/area associations instantly. Any KFC or heartbeat that references the deleted label will stop finding entities. No undo.
- **`install`** resets all `zen_` labels to canonical definitions (icons, colors, descriptions). It clobbers any customizations made since install. Never run mid-session without explicitly asking the operator — it will undo label description edits that Scribe and other tools may have written.
- **`confirm: true`** on writes is a safety gate, not a formality. The script refuses without it.

---

## Response Status Values

| Status | Meaning |
|---|---|
| `ok` | Action completed |
| `start` | Validation passed, awaiting confirmation (create/delete with `confirm: false`) |
| `partial` | Some labels were skipped (already existed on create, or not found on delete) |
| `error` | Validation failure or missing required input |

---

## Label Best Practices

### Write descriptions at creation time

Every label should have a description. It's one sentence, written once, and it travels inline with every entity in every Inspect call. Without a description, the AI sees a slug — a tag with no meaning. With one, she sees a concept.

The format: one sentence, include `label:slug` for self-reference, stay under 255 characters.

```
# Good
"Domain label for all water-related entities; label:water. Includes spa, irrigation, and leak sensors."

# Weak — no self-reference, too vague
"Water stuff"
```

Use `new_description` on `create`. Use `update` to refine descriptions later — retag is automatic.

---

### Cross-entity labels outperform single-use labels

The HyperIndex traverses the label graph — it finds all entities that share a label and assembles them into a structured snapshot. A label applied to 20 related entities produces a rich, contextualized Kata. A label applied to only one entity produces a thin one.

**Design labels around domains, not individual devices.**

```
# Good — water is a domain label shared across everything in that domain
water → sensor.water_main_flow
water → sensor.water_leak_kitchen
water → sensor.water_leak_basement
water → binary_sensor.irrigation_running
water → switch.irrigation_valve_zone_1

# Weak — single-entity labels add overhead without adding context
water_main_flow_sensor → sensor.water_main_flow
```

### Layer broad and specific labels together

Broad labels give the Ninja Summarizer its domain scope. Specific labels let the Index do targeted queries. Both serve different purposes.

```
# Broad domain label (feeds KFC component)
water → sensor.spa_water_temp
water → sensor.spa_ph

# Specific sub-label (targeted queries)
spa → sensor.spa_water_temp
spa → sensor.spa_ph
spa → sensor.spa_filter_state
```

The KFC drawer uses `label: water` to get the full water picture. A targeted `zen_dojotools_index` query can use `label: spa` to get spa-specific data. No conflicts, richer coverage.

### Prefix conventions

ZenOS-AI system labels use `zen_` prefixes (e.g., `zen_cabinet`, `zen_dojo_cabinet`). Your home labels should use a flat, readable namespace — `water`, `security`, `energy`, `spa`, `lighting`, `hvac`. Keep slugs short and lowercase.

### Labels are cheap — tag generously

Unlike entity exposure (which costs context window), labels cost nothing at rest. Tag every sensor that might be relevant to a domain. The Ninja Summarizer will evaluate relevance at summarization time. It's easier to remove a label later than to wonder why a sensor wasn't showing up in a Kata.

### The label-to-KFC connection

Every KFC component has a `label` field in its Dojo drawer. That label is what the HyperIndex traverses to build the component's entity snapshot. If you create a new label and forget to tag entities with it, the component runs with an empty dataset.

Checklist when adding a component:
1. Create the label (`create`)
2. Tag all relevant entities (`tag`)
3. Reference the label in the KFC drawer (`zen_dojotools_scribe`)
4. Dry-run with `force_summary` (post=false) and verify the Kata has data

---

## Dependencies

| Dependency | Purpose |
|---|---|
| HA native `labels()` | Read label index |
| HA native `label_description()` | Read label metadata |
| `homeassistant.create_label` | Create labels |
| `homeassistant.delete_label` | Delete labels |
| `homeassistant.add_label_to_entity` | Tag entities |
| `homeassistant.remove_label_from_entity` | Untag entities |
| `zen_resolver_refresh` event | Fired after reset to re-trigger resolver sensors |
