# zenos_cabinets.jinja — Cabinet Macro Library

**Version:** 1.7.0 (introduced in 2026.4.0 'Ectoplasm'; ACL macros, `fc_resolve` and the expansion macros in 2026.11.0 'This Is Spinal Tap')
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
| `cabinet_members(entity_id, fallback={})` | JSON object | `members` drawer value |
| `volumeinfo_copies(entity_id)` | JSON object | `{keys, best}`: stray VolumeInfo copies (`ai_cabinet_volumeinfo`, `_ai_cabinet_volumeinfo`) and the one holding an id |
| `cabinet_slot_candidates()` | JSON list | Expansion-slot cabinet entity_ids, from the canonical slot map |
| `cabinet_slot_status(entity_id)` | JSON object | Whether an expansion slot is free (no user drawers, not an expansion slave) and ready (booted) |
| `identity_roster(hh_cab='')` | JSON object | `{users, ai_users, all}` from the household's identity manifest; empty lists on any failure |

---

### ACL v2

The one ACL record Identity writes and FileCabinet enforces: VolumeInfo `acls` `{v: 2, owner, prime, partners, ops}`, principals as cabinet GUIDs. Read it only through these macros. All return JSON except `acl_guid_of` and `acl_op_class`.

| Macro | Returns | Description |
|-------|---------|-------------|
| `acl_guid_of(entity_id)` | String | The cabinet's VolumeInfo `id`, falling back to the legacy `guid` |
| `acl_class_of(entity_id)` | String | The cabinet's stamped class (VolumeInfo `class`), `''` when unstamped |
| `acl_class_plan(entity_id)` | String | The class `acl_migrate` proposes from the cabinet's labels. Only a migration stamps it; the gate never reads a label |
| `acl_default_ops(cls='')` | JSON object | `{r, w, d, a}`: the default row for a class (see below) |
| `acl_builtin_ops(drawer_key)` | JSON object | Fixed rules for credential and approval drawers, `{}` for any other key |
| `acl_lifecycle_tools()` | JSON list | The `@tool` principals that mount, dismount and provision cabinets (`zen_dojotools_provisioner`, `flynn_initialize_cabinets`); added to every class's `a` rule but the vault's |
| `acl_schema_current()` | String | The ACL schema version `acl_migrate` brings an install to (3) |
| `acl_schema_is_current()` | String | `True` when the system cabinet's `acl_schema` stamp is current; Flynn's bootstrap skips the migration then |
| `acl_record(entity_id, idmap={})` | JSON object | `{v, stored_v, class, self, owner, prime, partners, ops, ops_stored, ops_source, unresolved, inherited_from}`. Reads v1 records compatibly. `ops_stored` is what a writer stores back; `idmap` supplies ids for cabinets about to get one. An expansion (VolumeInfo `cabinet_class: expansion` + `hosted_by`) answers with its host's record and names the host in `inherited_from`; one hop, so a host that is itself an expansion is not followed |
| `acl_for(entity_id, drawer_key='')` | JSON object | `{class, self, owner, prime, partners, ops, source, inherited_from}`: effective ops for a drawer, each from builtin, else drawer, else cabinet, else the class default; `source` names where each op came from. `prime` falls back to the household's prime. A mount pointer answers with its cabinet's ACL, never a drawer `acl` |
| `acl_op_class(action)` | String | `r`, `w`, `d` or `a` for a FileCabinet action (`expand_drawer`, `collapse_drawer` and unknown actions are `a`) |
| `acl_check(acl, op, subject)` | JSON object | `{op, who_ok, via, cert, ack, source, reason}`. A caller matched by `@tool:<script>` passes on that tool's own gates (`via: tool`, no rule cert or ack). `subject` is `{guid, origin, tool}` or a bare GUID (an agent call) |
| `acl_enforce_mode()` | String | `audit` or `enforce`: the FileCabinet gate's mode, from the system cabinet's `fc_acl_mode` drawer. Anything but `enforce`, or no drawer, is `audit` |
| `acl_schema_check(entity_id)` | JSON object | `{ok, why}`. `why` can name: no VolumeInfo, no id, legacy guid field, no flags, acls not v2, stray VolumeInfo copies |

`cabinet_acls` and `drawer_acl` are retired.

**Principals in `who`:** cabinet GUIDs, `*` (anyone), `@owner`, `@prime`, `@partner` (any of `partners`), `@self` (the cabinet itself), `@internal` (a tool called the engine, not the agent-facing FileCabinet tool) and `@tool:<script>` (an internal call from that script). An empty `who` means only the admin door does that op.

**Default ops by class:**

| Class | r | w | d | a |
|---|---|---|---|---|
| system | `*` | door | door | door |
| vault | `@tool:zen_dojotools_identity` | same | same + ack | door |
| dojo | `*` | `@internal` | `@internal` | door |
| kata | `*` | `@owner`, `@internal` | `@internal` | door |
| history | `@owner`, `@prime`, `@internal` | `@internal` | `@internal` + ack | door |
| household, family | `*` | `@prime`, `@internal` | `@prime` + ack | `@owner` + cert `zenos.system.cabinet_admin:1` + ack |
| user | `@self`, `@partner`, `@prime`, `@internal` | same | `@partner`, `@prime` + ack | `@owner` + cert + ack |
| ai_user | `@self`, `@prime`, `@internal` | `@self`, `@internal` | `@self` + ack | `@self` + cert `zenos.identity.acl_take_ownership:1` + ack |
| scratchpad | `*` | `*` | `*` | `@prime` |
| unclassified | `*` | `@prime`, `@internal` | `@prime` | door |

**Built-in locks** (win over every other rule): `zen_ai_certs`, `integrations_config` and `alert_response_*` take `w` and `d` from `@internal` only; `cert_admin_console_bypass` and `fc_acl_mode` are admin door only.

---

### Mounts and Drawer Shape

| Macro | Returns | Description |
|-------|---------|-------------|
| `fc_resolve(entity_id, drawer_key, max_hops=8)` | JSON object | `{status, vol, key, depth, visited, reason, loop_at, fc_args, entry[, expanded_root]}`: the real target behind a mount chain. `status` is `absent`, `direct`, `resolved`, `live`, `not_found_in_target`, `unresolvable`, `loop_detected` or `depth_limit_exceeded`. Softlinks and expansions are one chain: at each cabinet, a key it doesn't hold under an expanded root hops into that expansion, and a softlink there is followed on; every hop lands in `visited`. FileCabinet's one mount resolver |
| `expansion_root_of(entity_id, drawer_key)` | JSON object | `{root, target}`: the nearest ancestor of the key that is a pointer into an expansion this cabinet hosts, and that expansion. Empty strings when there is none. The key itself is not checked |
| `expansion_plan(entity_id, drawer_key)` | JSON object | What a move, copy or delete of the key touches: `mode` is `self` (the key is an expanded root), `ancestor` (it sits under one), `contains` (its tree holds expanded roots) or `none`, with `roots: [{root, target, scope}]` |
| `cabinet_hosts_expansion(entity_id)` | String | `true` when any expansion slot is stamped as hosted by this cabinet; a cheap check before building the view |
| `cabinet_resolved_view(entity_id)` | JSON object | `cabinet_expanded_view` with each pointer into an expansion this cabinet hosts replaced by the drawer the expansion holds: the tree as its owner sees it. Other mount pointers stay. `search` reads it |
| `label_index(entity_id)` | JSON object | The cabinet's `_label_index` as `{label: [drawer keys]}`, `{}` when absent or unreadable. Keyed by each drawer's own path, so an expanded drawer's labels stay in its host's index |
| `label_index_keys(idx, keys)` | JSON object | `{key: [labels]}` for the given keys; keys without a label are left out |
| `label_index_apply(idx, drop, add)` | JSON object | The index with the keys in `drop` removed from every label and `add` `{key: [labels]}` added; a label left with no drawer goes |
| `cabinet_expanded_view(entity_id)` | JSON object | The cabinet's drawers as one tree: its own, plus each expansion's keys under the expanded root it doesn't hold itself (children added after the move). `get` (children) and `list` read it |
| `cabinet_drawer_value_mounted(entity_id, drawer_key, fallback={})` | JSON object/any | Like `cabinet_drawer_value`, following one mount hop |
| `drawer_normalize(env)` | JSON object | `{ok, changed, value, meta, repairs}`: a drawer envelope normalized to `{value, meta}` (nested envelopes unwrapped, expiry moved into `meta`). `ok: false` means do not write |

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
