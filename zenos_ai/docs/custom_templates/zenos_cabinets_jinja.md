# zenos_cabinets.jinja — Cabinet Macro Library

**Version:** 1.0.0 (introduced in 2026.4.0 'Ectoplasm' — current as of 2026.6.0 'Clue' stable / 2026.7.0 'Neo' beta)
**File:** `custom_templates/zenos_ai/zenos_cabinets.jinja`
**Status:** Stable. Required by `zen_os_1.jinja`, `zenos_health.jinja`, `dojotools_filecabinet`, and any template that reads cabinet drawers.

---

## Purpose

`zenos_cabinets.jinja` is the canonical safe-read layer for all ZenOS cabinet I/O in Jinja templates.

Before this library, every caller that needed a drawer value had to hand-roll the FG-38 two-round normalization pattern — unwrap the `{value: ...}` wrapper, guard for JSON-encoded strings, guard for missing keys. That logic was duplicated across `zen_os_1.jinja`, `zenos_health.jinja`, `dojotools_filecabinet`, and more.

This library encapsulates that pattern once. All drawer reads go through it.

---

## Import

```jinja
{%- import 'zenos_ai/zenos_cabinets.jinja' as CABS -%}
```

Import once at the top of the file. Do not import inside loops or per-macro — the template engine re-parses on every call.

---

## Cabinet Schema Assumptions

All cabinets using these macros follow the ZenOS schema:

```yaml
state: online_mounted   # or other cabinet state

attributes:
  variables:
    drawer_name:
      value: {}          # the actual payload — dict, list, or string
      timestamp: "..."   # optional ISO8601 string
      meta: {}           # optional drawer metadata (ro, entity_labels, etc.)
```

`cabinet_drawer_value()` extracts `variables → drawer_name → value`, handling both native-dict and JSON-encoded-string cases at each level (FG-38 safe, two-round normalization).

---

## API Reference

All macros return a **JSON string**. Chain with `| from_json()` to get a native dict/list.

### Cabinet Resolution

| Macro | Returns | Description |
|-------|---------|-------------|
| `cabinet_list_from_label(label)` | JSON array of entity IDs | All `sensor.*` entities with the given HA label |
| `cabinet_id_from_label(label, default='')` | String entity ID | First `sensor.*` entity for the label, or `default` |
| `cabinet_id_from_slot(slot, default='')` | String entity ID | Resolves a ZenOS slot name (`dojo`, `kata`, `household`, etc.) to an entity ID via `zenos_health.jinja` |

---

### Variables (full drawer dict)

| Macro | Returns | Description |
|-------|---------|-------------|
| `cabinet_variables(entity_id, fallback={})` | JSON object | Full `variables` attribute dict from the cabinet. Safe — returns `fallback` if entity is unavailable or `variables` is not a mapping |
| `cabinet_variables_from_label(label)` | JSON object | Resolves label → entity, then returns `cabinet_variables()` |

**Example:**
```jinja
{%- set vars = CABS.cabinet_variables('sensor.zen_dojo_cabinet') | from_json() -%}
```

---

### Drawer Access

| Macro | Returns | Description |
|-------|---------|-------------|
| `cabinet_drawer(entity_id, drawer_key, fallback={})` | JSON object | Full drawer dict `{value, timestamp, meta}`. Auto-parses JSON-encoded strings at the drawer level |
| `cabinet_drawer_value(entity_id, drawer_key, fallback={})` | JSON object/any | The `value` field from the drawer. FG-38 safe: two-round normalization handles JSON-encoded strings at both the drawer and value levels |
| `cabinet_drawer_exists(entity_id, drawer_key)` | JSON boolean | `true` if the drawer key exists in the cabinet's variables |

**When to use which:**
- Use `cabinet_drawer_value()` for almost everything — it returns the data payload directly.
- Use `cabinet_drawer()` only when you need the full drawer struct (e.g., to read `meta.ro` or `timestamp`).

**Example:**
```jinja
{%- set essence = CABS.cabinet_drawer_value(cabinet_entity, 'zenai_essence') | from_json() -%}
{%- set drawer  = CABS.cabinet_drawer(cabinet_entity, 'zenai_essence') | from_json() -%}
{%- set is_ro   = drawer.get('meta', {}).get('ro', false) -%}
```

---

### Volume Drawer Access (pre-fetched dict)

For cases where the variables dict is already loaded (e.g., inside a loop over `cabinet_variables()` output), use the `volume_*` variants — they operate on a dict directly rather than fetching from an entity.

| Macro | Returns | Description |
|-------|---------|-------------|
| `volume_drawer(volume, drawer_key, fallback={})` | JSON object | Drawer dict from a pre-fetched variables dict |
| `volume_drawer_value(volume, drawer_key, fallback={})` | JSON object/any | Value field from a drawer in a pre-fetched dict |
| `volume_guid(volume, fallback='')` | String | GUID from `AI_Cabinet_VolumeInfo.id` in a pre-fetched volume dict |

---

### VolumeInfo Helpers

| Macro | Returns | Description |
|-------|---------|-------------|
| `cabinet_volume_info(entity_id, fallback={})` | JSON object | `AI_Cabinet_VolumeInfo` value block — GUID, friendly_name, description, flags, acls |
| `cabinet_volume_info_exists(entity_id)` | JSON boolean | `true` if VolumeInfo exists and is non-null |
| `cabinet_guid(entity_id, fallback='')` | String | GUID (`id` field) from VolumeInfo — plain string, not JSON |
| `cabinet_guid_compat(entity_id, fallback='')` | String | GUID with fallback to legacy `guid` field — use for pre-4.5.6 cabinets |
| `cabinet_friendly_name(entity_id, fallback='')` | String | `friendly_name` from VolumeInfo |
| `cabinet_acls(entity_id, fallback={})` | JSON object | ACLs block from VolumeInfo |
| `cabinet_members(entity_id, fallback={})` | JSON object | `members` drawer value |

---

### GUID Generation

| Macro | Returns | Description |
|-------|---------|-------------|
| `cabinet_guid_new()` | String | Random GUID (8-4-4-4-12 hex). Not cryptographically secure. |
| `cabinet_guid_new_v4()` | String | RFC 4122 UUID v4 with correct version/variant bits. Use this for new cabinets. |

---

### Utilities

| Macro | Returns | Description |
|-------|---------|-------------|
| `safe_parse_json_string(raw, fallback={})` | JSON string | Safely parses a string that may contain JSON; passes through mappings unchanged |
| `safe_get_nested(obj, keys, fallback='')` | JSON string | Traverses a nested dict by a list of keys; returns fallback if path missing |

**Example — safe nested access:**
```jinja
{%- set name = CABS.safe_get_nested(essence, ['jacket', 'name'], '') | from_json() -%}
```

---

## FG-38 Safety

`cabinet_drawer_value()` encapsulates the canonical FG-38 two-round normalization pattern:

1. Fetch `variables` from `state_attr` — guard for non-mapping
2. Get drawer by key — guard for JSON-encoded string
3. Get `.value` field — guard for JSON-encoded string again

Callers do **not** need to re-implement this. `cabinet_drawer_value()` returns a native dict/list after `| from_json()`. Do not call `| from_json()` twice.

---

## Used By

| File | Usage |
|------|-------|
| `zen_os_1.jinja` | All cabinet drawer reads — essence, VolumeInfo, ACLs, system drawers, manifest, compact index |
| `zenos_health.jinja` | Essence check in health macros |
| `dojotools_filecabinet.yaml` | `initial_volume` and `prewrite_volume` reads |
| `dojotools_identity.yaml` | Profile and VolumeInfo reads |
| `dojotools_profile.yaml` | Profile drawer reads |
| `dojotools_admintools.yaml` | KFC and manifest reads |
| `dojotools_core.yaml` | Core drawer reads |
| `dojotools_manifest.yaml` | Manifest drawer reads |
| `sensors/zenos_cabinet_health.yaml` | Cabinet variable enumeration for health scoring |

---

→ **[Documentation Hub](../readme.md)**
→ **[zen_os_1.jinja reference](zen_os1_jinja.md)**
