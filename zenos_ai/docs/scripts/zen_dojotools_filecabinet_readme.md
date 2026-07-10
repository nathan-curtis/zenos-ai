# Zen DojoTools FileCabinet — v6.2.0
**File:** `zen_dojotools_filecabinet_readme.md`
**Type:** Technical Documentation
**ZenOS Release:** 2026.7.0 'Neo'

---

## Overview

The **Zen DojoTools FileCabinet** is the primary read/write controller for **Cabinet Volumes** inside ZenOS-AI. Cabinet Volumes function as structured, drawer-based data stores that hold the crew's operational state, context pockets, lookup tables, mappings, micro-memories, relationship links, and other structured data.

This module implements **100% health-aware** operations. All writes — create, update, upsert, delete, move, copy — are blocked if a volume is unhealthy unless explicitly overridden with `force_action`.

The FileCabinet script is the only sanctioned way for an LLM agent to **mutate** Cabinet data.

### Layer Stack

```
zen_dojotools_filecabinet  (DojoTool, MCP-facing — expose this to MCP)
  ├─ stack=wiki    → zen_sutra_wikijs  (optional, requires WikiJS integration)
  └─ stack=cabinet → zen_sutra_filecabinet  (always available)
                        → CABS (reads) + set/remove_variable_legacy (writes)
```

`zen_sutra_filecabinet` is the internal terminus. It is **not** MCP-exposed. Callers always go through `zen_dojotools_filecabinet`.

`stack=wiki` routes to `zen_sutra_wikijs`. FileCabinet is the single surface for both knowledge stacks. The former standalone `zen_dojotools_wikijs` script no longer exists.

---

## Two-Path Rule (Hard Rule)

There are exactly two licensed paths to cabinet data, and they must never be crossed:

| Path | Who uses it | Purpose |
|------|-------------|---------|
| `script.zen_dojotools_filecabinet` | Scripts, automations, AI agents, MCP callers | All reads and writes |
| `zenos_cabinets.jinja` CABS macros | Templates only — and only inside `zen_sutra_filecabinet` | Internal terminus reads |

Scripts must call `script.zen_dojotools_filecabinet`. Templates must import CABS. Never cross these paths — calling the Sutra directly from outside the Sutra creates a loop at prod rename.

---

## Core Responsibilities

### Drawer Operations
Implements the full CRUD+M model:

- `create` — create a new drawer; errors if key exists (use `upsert` or `force_action` to overwrite)
- `update` — full overwrite of an existing drawer (default mode: replace)
- `upsert` — create if absent, update if present (default mode: merge)
- `delete` — remove a drawer and cascade to all children
- `move` — transfer a drawer to another key or cabinet; cascade + index migration
- `copy` — copy a drawer; cascade; inherits source labels if no tags supplied
- `relabel` — update the `_label_index` for a drawer without touching its value

All writes are re-read and verified for correctness via `wait_template`.

### Reads (Four Modes)
1. **Direct Read** (`get`) — read a specific drawer; surfaces subdrawers and mount traversal
2. **Directory** (`list`) — list all non-system drawers in a volume; supports `path_prefix` filter
3. **Label + Text Search** (`search`) — match drawers by label via `_label_index` or text scan across key/title/content
4. **Global Manifest** (`manifest`) — delegates to `zen_dojotools_manifest`

### Labeling and Label Index
FileCabinet maintains the volume's `_label_index`, converting:

```
label → [drawer1, drawer2, ...]
```

Labels are:
- lowercased
- slugified
- deduplicated
- filtered to system label registry (unless `create_label: true`)

Broken `_label_index` formats (legacy Python-literal strings) are repaired in flight.

HA Labels **are** the tag system. There is no separate tags field in the storage envelope — labels live in the index drawer, not on the drawer itself.

### Health-Aware Write Pipeline
Before any write, FileCabinet checks:

- volume health status
- GUID mismatch
- read-only flag (cabinet-level or per-drawer `meta.ro`)
- schema compatibility
- storage warnings

If any red flags exist → write blocked unless `force_action: true`.

`sensor.zenos_system_cabinet` is permanently hard read-only; no flag overrides this.

### Protect-Write Mode
All moves verify the destination write before deleting the source. This prevents data loss if the destination write times out.

---

## Storage Format (v6.2+)

Each drawer is stored as:

```json
{
  "drawer_key": {
    "value": <raw content — string, dict, list, number, bool, or null>,
    "timestamp": "2026-06-16T12:00:00-05:00"
  }
}
```

The `meta` key is reserved for future ACL use and is not yet written by the Sutra.

### dict-in / dict-out Contract

v6 callers must pass native dict/list to `value:`. Do **not** `| tojson` before passing. FileCabinet handles serialization internally.

```yaml
# CORRECT
value: "{{ my_dict }}"

# WRONG — do not do this
value: "{{ my_dict | tojson }}"
```

### v1 Envelope Auto-Detection

Old v1-envelope drawers — where `value` is a dict with `v: 1` and a `content` key — are auto-detected on GET, LIST, and SEARCH and unwrapped transparently. The response `value` field contains the unwrapped content. Callers never see the envelope structure.

### content → value Migration

FC GET no longer returns a `content` key. All callers read `.get('value', {})` from the response. The `content` key is an internal v1 artifact that is unwrapped silently.

---

## CabCeption — Nested Drawer Trees

**Version:** 6.2.0

CabCeption is the `/` path separator feature that allows drawers to form virtual nested trees inside a single cabinet volume. No structural changes are required to the volume — the tree is implied by key naming.

### How It Works

A path like `character_sheet/inventory` is stored as two independent drawer keys:
- `character_sheet` — the parent drawer (may hold its own value)
- `character_sheet/inventory` — the child drawer

The `/` separator is preserved through slugify. Each segment is slugified individually. The cabinet resolves depth as it traverses.

### Path Resolution

When you call `get` on a path, FC:
1. Reads the direct key
2. Scans for immediate children (`character_sheet/` prefix, one level deep)
3. Returns `has_children: true` and `subdrawers: [...]` if children exist

The response is additive — existing callers that do not use CabCeption are not affected.

### Cascade Behavior

`move`, `delete`, and `copy` all cascade to children via prefix scan:

- **delete** `character_sheet` → also deletes `character_sheet/inventory`, `character_sheet/stats`, etc.
- **move** `character_sheet` → `char_sheet` → also moves all children; child writes are verified before source delete
- **copy** `character_sheet` → `character_sheet_backup` → subtree copied

### inspect with Subdrawers

`inspect drawer_key` surfaces subdrawers inline. Use `inspect` to explore tree structure without reading all child values.

### Path Naming

Paths are normalized on input: leading `/` is stripped, each segment is slugified independently, and the `/` separators are preserved. The underscore-prefix rule (requires `force_action` for `_`-prefixed keys) applies to the leading segment only.

---

## Three Drawer Types

### Regular Drawer

A drawer that stores a value directly. The standard case — any drawer that is not a VirtualDrawer or LiveDrawer.

```json
{
  "my_drawer": {
    "value": {"color": "blue", "size": 12},
    "timestamp": "2026-06-16T12:00:00-05:00"
  }
}
```

### VirtualDrawer (Mount Point)

A softlink / mount point. Set via `set_mount`. When FC reads a VirtualDrawer, it transparently redirects to another cabinet's drawer of the same key.

A VirtualDrawer has `mount_point: true` in its `value`:

```json
{
  "my_alias": {
    "value": {
      "mount_point": true,
      "target_guid": "<cabinet GUID>",
      "target_entity_id": "sensor.zenos_expansion_cabinet_1"
    },
    "timestamp": "..."
  }
}
```

Mount traversal is a redirect, not execution. The Sutra follows the mount chain up to depth 8. Loop detection and depth-limit guards are enforced.

Mounts are set with `set_mount` (requires `key` + `target_guid`). They are removed with `remove_mount`. The `clone` action copies mount point drawers through to the destination (noted in the response as `mounts_copied`).

Same-volume mounts are blocked — they create a traversal loop.

### LiveDrawer (Tool-Call Mount)

A LiveDrawer absorbs the KF4 schema. It is a VirtualDrawer variant where the mount contains `fc_args` — a JSON object describing a tool call to execute on read.

**LiveDrawer fields inside `fc_args`:**

| Field | Purpose |
|-------|---------|
| `tool` | Script name to call (e.g. `zen_dojotools_inventory`) |
| `params` | Params dict passed to the tool call |
| `cache.ttl_seconds` | Cache TTL in seconds |
| `cache.digest_fields` | Optional: field paths to extract from the tool response for caching |

**Read behavior:**

- **Warm cache** (within TTL): FC fires the tool call, returns the live result, and refreshes the cache sub-drawer (`<key>/cache`).
- **Cold cache** (auto-expired or never populated): FC returns the cached value from the sub-drawer and signals stale via `mount_traversal.cache: miss`.
- **Bare read** (no TTL configured): FC always returns the cache; never empty because bare reads never expire.

Auto-expiry is managed by FC. The caller does not manage staleness.

LiveDrawers are set via `set_mount` with `fc_args` instead of `target_guid`:

```yaml
action_type: set_mount
volume_entity_id: sensor.my_cabinet
key: live_inventory
fc_args: >
  {
    "tool": "zen_dojotools_inventory",
    "params": {"mode": "summary"},
    "cache": {"ttl_seconds": 300, "digest_fields": ["result.count"]}
  }
```

Tool calls are gated by a whitelist stored in `sensor.zenos_system_cabinet` (`fc_mount_callout_whitelist`). If the drawer's `fc_args.tool`/mode combination is not covered by the whitelist, the read returns `tool_call_blocked`.

### Whitelist Entry Formats

The whitelist supports three entry formats, parsed via a `tool[:mode]` split:

| Format | Meaning |
|---|---|
| `tool` | Plain tool name — all modes reachable via a live drawer. **Deprecated, backcompat only.** |
| `tool:*` | All modes reachable — explicit wildcard, preferred over the bare-name form above. |
| `tool:mode` | Restricted to exactly one mode — **required for any tool with write-capable modes.** |

Adding a bare `*` or a `tool:` entry with an empty mode suffix is rejected by Admintools on add.

Mode-scoping exists so a live drawer can expose a self-describing or read-only mode (e.g. `kfc_manifest`) without opening any write path on that tool. `zen_dojotools_finance`, for example, is scoped `zen_dojotools_finance:kfc_manifest` only — no live drawer can reach a finance write mode. See [Building a KFC — KF5](../kung_fu/building_a_kfc.md#kf5-self-registering-tools) for the self-registration flow that writes these entries.

---

## stack= Routing

The `stack` field on `zen_dojotools_filecabinet` determines the backend:

| `stack` value | Routes to | Notes |
|---------------|-----------|-------|
| `cabinet` (default) | `zen_sutra_filecabinet` | Always available |
| `wiki` | `zen_sutra_wikijs` | Optional; requires WikiJS integration |
| `all` | Read-only aggregate | List, health, stacks_by_anchor only |

`stack=wiki` is how FileCabinet provides wiki surface access. There is no separate `zen_dojotools_wikijs` script. Specify `stack=wiki` explicitly for all wiki operations — the default is `cabinet`.

Cross-stack copy and move are supported via `destination_stack`.

---

## Action Model

### Action Aliases

| Alias | Resolves to |
|-------|-------------|
| `read` | `get` |
| `write` | `upsert` |
| `scan` | `list` |

### `manifest`
Delegates to `zen_dojotools_manifest`. Returns the full Cabinet manifest. Supports `show_hidden_volumes` and `show_stacks`.

### `get`
Read a specific drawer. Returns `value`, `tags`, `updated_at`, and optionally `drawers` (immediate children) and `subdrawers` (full CabCeption paths) if the key has children.

Mount traversal is transparent — if the drawer is a VirtualDrawer, FC follows the chain and returns the resolved value with `mount_traversal` metadata. If it is a LiveDrawer with a warm cache hit, FC fires the tool call and returns the result.

### `list`
List all non-system drawers in a volume. Supports `path_prefix` to filter by key prefix. Returns `key`, `title`, `description`, `created_at`, `updated_at` for each drawer.

### `create`
Requires `path` and `value` (or `title`/`description`). Errors if the key already exists — use `upsert` or `force_action: true` to overwrite.

### `update`
Full overwrite of an existing drawer. Default mode is `replace`. Pass `merge: true` to force deep-merge instead. Errors if the key does not exist — use `upsert` or `force_action: true` to create-if-missing.

### `upsert`
Creates if absent, updates if present. Default mode is `merge` (deep-merge into existing value). Pass `merge: false` or use `mode: replace` to force replace. Atomic — no separate existence check needed from the caller.

### `delete`
Removes the drawer, prunes `_label_index`, verifies deletion, then cascades to all children via prefix scan.

### `move`
Transfers a drawer to `destination_path` (same cabinet) or `destination_volume_entity_id` (cross-cabinet). Cascades to all children. Each child write is verified before the source child is deleted. Label index is migrated: source index is pruned, destination index is updated with moved labels.

**Mounted drawer protection:** If the drawer is a VirtualDrawer or LiveDrawer, `move` returns `status: error, code: mounted_drawer` with the mount type. Use `set_mount` at the destination key + `delete` at the source instead. Pass `force_action: true` to bypass — the full mount config transfers intact (the mount_point structure moves with the entry, not just `.value`). Without `force_action`, the operation is hard-blocked; a plain move without it would silently strip the mount config, leaving a dead entry at the destination.

### `copy`
Copies a drawer to `destination_path`. Cascades subtree. If no `tags` are supplied, the source drawer's existing labels are inherited at the destination.

**Mounted drawer protection:** Same guard as `move`. Copying a mount without `force_action: true` is blocked to prevent silent mount config loss at the copy destination.

### `relabel`
Updates the `_label_index` for an existing drawer without touching its value. Use this to fully replace a drawer's labels rather than adding to them.

### `search`
Label match via `_label_index` (pass `anchor_ids`) plus full-text scan against key, title, content, and description (pass `query`). Both filters can be combined.

### `clone`
Copies the full content of one cabinet into another. Health checks run on both source and destination before any write.

- Copies all non-system drawers from source to destination
- Mount point drawers are copied through (noted in response as `mounts_copied`)
- Blocked if source and destination are the same cabinet

**GUID transfer** (optional, requires `transfer_guid: true` + `force_action: true`):
- Moves GUID ownership from source to destination — Highlander mode
- Clears source drawers after transfer
- Marks source VolumeInfo with `needs_restamp: true` so the stamper assigns a new identity on next pass

> **Known limitation:** Large cabinets (10+ drawers) may exceed script timeout during a GUID transfer pass. If Highlander returns `{}` or times out, use a standard clone to move the data, then manually stamp the destination. Data gets there intact — you lose single-step atomicity.

### `set_mount`
Creates a VirtualDrawer or LiveDrawer at the specified key.

- Cross-cabinet redirect mount: requires `key` + `target_guid`
- Tool-call (LiveDrawer) mount: requires `key` + `fc_args`
- Blocked if `target_guid` resolves to the same volume (loop guard)
- Blocked if the key exists and is not already a mount point, unless `force_action: true`

### `remove_mount`
Removes a mount point drawer. Errors if the key does not exist or is not a mount point.

### `mount_cabinet`
Transitions a cabinet from `online_unmounted` to `online_mounted`. Idempotent if already mounted.

### `dismount_cabinet`
Transitions a cabinet to `online_unmounted` (puts it in the stacks). Blocked if:
- The cabinet holds a `zen_default_*` label
- The cabinet is a pipeline cabinet (`zen_dojo_cabinet` or `zen_kata_cabinet`) and the summarizer pipeline is live

### `set_cabinet_ro`
Stamps `cabinet_ro: true` in the cabinet's `AI_Cabinet_VolumeInfo.flags`. Blocked on `sensor.zenos_system_cabinet`.

### `inspect`
Returns the Lens Bus registry entry for the FileCabinet provider. Surfaces subdrawers inline when called on a CabCeption path.

### `health`
- No `volume_entity_id` → system-wide cabinet status: mounted/unmounted counts, pipeline switch states
- With `volume_entity_id` → single-volume availability check

### `register` / `unregister`
Lens Bus registration. Registers or removes the FileCabinet provider entry in the household cabinet's `lens_registry` drawer.

### `stacks_by_anchor`
Lens Bus provider: returns cabinet volumes matching the given anchor (label by default). Used by the Lens Bus to discover evidence sources.

### `audit`
Returns the FileCabinet provider's security and content policy declaration.

---

## Output Structure

All outputs follow a consistent envelope:

```json
{
  "status": "success|warning|error|info|not_found|not_supported",
  "tool": "zen_sutra_filecabinet",
  "action": "<action performed>",
  "key": "<drawer key>",
  "volume": "<entity_id>",
  "value": {},
  "title": "",
  "description": "",
  "created_at": "",
  "updated_at": "",
  "expires_after": null,
  "no_autoexpire": false,
  "no_autorecycle": false,
  "tags": [],
  "has_children": true,
  "subdrawers": ["parent/child1", "parent/child2"],
  "drawers": {"child_key": {"is_mount": false, "updated_at": "..."}},
  "mount_traversal": {},
  "write_verified": true,
  "health_snapshot": {},
  "trace": {"caller_token": ""}
}
```

Fields present depends on the action. `value` is present on `get`. `write_verified` is present on writes. `has_children` and `subdrawers` are present only when children exist.

---

## Valid Drawer Contract

FileCabinet treats a cabinet as valid only when the target cabinet passes the Cabinet Specification and the requested drawer can be represented as a wrapper with a `value` field.

```json
{
  "drawer_name": {
    "value": {},
    "timestamp": "2026-06-16T12:00:00-05:00"
  }
}
```

The nested `value` may be an object, array, string, number, boolean, or null. Higher-level tools impose stricter shapes:

| Drawer Kind | Expected Payload |
|---|---|
| `_user_profile` | Mapping with person/profile fields |
| `_family_profile` | Mapping with family display metadata |
| `_household_profile` | Mapping with household name/address/timezone fields |
| `members` | Mapping with `users`, `ai_users`, and `families` lists |
| `zenai_essence` | AI persona essence in `core / jacket / companion` shape |

For the full cabinet validity rules, see [Cabinet Specification](../cabinets/cabinet_spec.md). For identity-specific drawer shapes, see [Profile Editor](zen_dojotools_profile_readme.md) and [Identity](zen_dojotools_identity_readme.md).

---

## Visibility Model

Drawers in a cabinet have three visibility tiers:

| Tier | Description |
|------|-------------|
| Active / described | Regular drawer with `title` or `description` in the envelope; surfaced by list and inspect |
| Hidden / undescribed | Regular drawer with no title/description; surfaced by list but not featured in summaries |
| System-protected | Keys prefixed with `_` (e.g. `_label_index`) or `AI_Cabinet_VolumeInfo`; require `force_action: true` to write; `sensor.zenos_system_cabinet` is hard read-only |

---

## Labeling Model

Drawer labels live in the volume's `_label_index` drawer, not on the drawer itself:

```
_label_index.value = { "profile": ["my_drawer"], "preferences": ["my_drawer", "settings"] }
```

Labels are:
- trimmed
- lowercased
- slugified
- filtered to system-wide available labels (unless `create_label: true`)
- attached at drawer creation, update, upsert, or copy

Label targets for `search` support:
- comma-separated string
- newline-separated string
- list format
- slugified comparison

Use `relabel` to fully replace labels on a drawer (the index is rebuilt from scratch for that key). Adding `tags` on create/update/upsert is additive — it appends, not replaces.

---

## GC Lifecycle

Drawers support two GC fields set on write:

| Field | Behavior |
|-------|----------|
| `expires_after` | ISO 8601 datetime after which the drawer may be GC'd |
| `no_autoexpire` | `true` suppresses auto-expiry even past `expires_after` |
| `no_autorecycle` | `true` suppresses the auto-recycle pass |

These fields are stored in the drawer envelope by the Sutra. GC runs independently; FC does not enforce expiry at read time.

### GC Operator (`zen_dojotools_kata_gc`)

`zen_dojotools_kata_gc` is a **MCP-exposed, destructive operator**. It is the only tool that permanently deletes or recycles kata drawers. It must not be called without operator intent.

| Mode | What it does | Destructive? |
|------|-------------|--------------|
| `gc` | Evict expired drawers from the kata cabinet | **Yes — permanent delete** |
| `recycle` | Move a named drawer to `.recycle/<key>` (soft delete) | Yes — drawer is hidden |
| `hide` | Rename drawer to `.hidden/<key>` (suppress from GC) | Yes — path change |
| `unhide` | Restore a hidden drawer to its original key | No |

**Guardrails:**

- Always run with `dry_run: true` first. The response shows exactly which drawers would be evicted with their age and reason — no changes are made.
- `gc` mode only evicts drawers whose `expires_after` timestamp has passed **or** whose age exceeds `max_age_hours`. It never evicts drawers with `no_autoexpire: true`.
- `recycle` and `hide` require an explicit `key`. They do not operate on trees — only the named drawer is affected. Children are **not** cascaded.
- There is no undo for `gc`. Recycled drawers (`.recycle/<key>`) can be restored manually via FileCabinet `move`, but GC-evicted drawers are gone.
- This tool is **operator-only**. Agents should not call it autonomously. If an agent needs to clean up its own session state, it should set `expires_after` on write and let the scheduler-driven GC handle eviction.

**Scheduled GC:** The system scheduler calls `zen_dojotools_kata_gc mode=gc` daily (via `dojotools_core.yaml` automation). Manual invocation is for maintenance or emergency use only.

---

## Safety Features

### Health Blocking
Writes blocked when:
- volume health status is `error`
- GUID mismatch
- `writeable` flag is false
- storage warning (unless `force_action: true`)
- `cabinet_ro` flag set (unless volume is syscab, in which case no flag helps)
- per-drawer `meta.ro` flag

### Write Verification
Every write uses `wait_template` (30-second timeout) to confirm the updated timestamp is visible in state before returning. `write_verified: true` in the response means the post-write read confirmed the change landed.

### Move Safety
For `move`, the destination write is verified before the source is deleted. If the destination write times out, the operation returns `status: partial` and leaves the source intact. No data loss on timeout.

### Child Cascade
`delete`, `move`, and `copy` cascade to children via prefix scan. For `move`, each child write is individually verified before its source is deleted.

### Concurrent-Operation Awareness
Script-level `mode: queued` (Sutra) and `mode: parallel` (DojoTool) prevent deadlock while allowing concurrent callers.

### JSON Validation
Malformed JSON always returns a structured error — never a partial write.

---

## Usage Examples

### List drawers in a volume
```yaml
action_type: list
volume_entity_id: sensor.household_cabinet
stack: cabinet
```

### Read a drawer
```yaml
action_type: get
volume_entity_id: sensor.household_cabinet
path: favorite_color
stack: cabinet
```

### Read a CabCeption nested drawer
```yaml
action_type: get
volume_entity_id: sensor.my_cabinet
path: character_sheet/inventory
stack: cabinet
```

### Read drawers by label
```yaml
action_type: search
volume_entity_id: sensor.household_cabinet
anchor_ids: "profile, preferences"
stack: cabinet
```

### Create a drawer
```yaml
action_type: create
volume_entity_id: sensor.household_cabinet
path: favorite_color
value: "blue"
tags: "preferences, profile"
stack: cabinet
```

### Upsert a drawer (atomic create or update)
```yaml
action_type: upsert
volume_entity_id: sensor.household_cabinet
path: settings
value:
  theme: dark
  language: en
tags: preferences
stack: cabinet
```

### Update with explicit replace mode
```yaml
action_type: update
volume_entity_id: sensor.household_cabinet
path: settings
value:
  theme: light
merge: false
stack: cabinet
```

### Delete a drawer (cascades to children)
```yaml
action_type: delete
volume_entity_id: sensor.my_cabinet
path: character_sheet
force_action: true
stack: cabinet
```

### Move a drawer (same cabinet)
```yaml
action_type: move
volume_entity_id: sensor.source_cabinet
path: settings
destination_path: settings_archive
stack: cabinet
```

### Move cross-cabinet
```yaml
action_type: move
volume_entity_id: sensor.source_cabinet
path: settings
destination_path: settings
destination_volume_entity_id: sensor.dest_cabinet
stack: cabinet
```

### Copy a drawer
```yaml
action_type: copy
volume_entity_id: sensor.household_cabinet
path: template_profile
destination_path: user_profile_copy
stack: cabinet
```

### Relabel a drawer
```yaml
action_type: relabel
volume_entity_id: sensor.household_cabinet
path: my_drawer
tags: "profile, active"
stack: cabinet
```

### Set a VirtualDrawer (cross-cabinet mount)
```yaml
action_type: set_mount
volume_entity_id: sensor.my_cabinet
path: shared_config
target_guid: "abc-123-def-456"
stack: cabinet
```

### Set a LiveDrawer (tool-call mount)
```yaml
action_type: set_mount
volume_entity_id: sensor.my_cabinet
path: live_inventory
fc_args: >
  {
    "tool": "zen_dojotools_inventory",
    "params": {"mode": "summary"},
    "cache": {"ttl_seconds": 300}
  }
stack: cabinet
```

### Remove a mount point
```yaml
action_type: remove_mount
volume_entity_id: sensor.my_cabinet
path: shared_config
stack: cabinet
```

### Clone a cabinet (content copy)
```yaml
action_type: clone
volume_entity_id: sensor.source_cabinet
destination_cabinet: sensor.dest_cabinet
stack: cabinet
```

### Clone with GUID transfer (Highlander mode)
Moves identity ownership. Source is cleared and marked for restamping.
```yaml
action_type: clone
volume_entity_id: sensor.source_cabinet
destination_cabinet: sensor.dest_cabinet
transfer_guid: true
force_action: true
stack: cabinet
```

### Mount a cabinet
```yaml
action_type: mount_cabinet
volume_entity_id: sensor.expansion_cabinet_1
stack: cabinet
```

### Dismount a cabinet
```yaml
action_type: dismount_cabinet
volume_entity_id: sensor.expansion_cabinet_1
stack: cabinet
```

### System-wide health check
```yaml
action_type: health
stack: cabinet
```

### Single-volume health check
```yaml
action_type: health
volume_entity_id: sensor.household_cabinet
stack: cabinet
```

### Full cabinet manifest
```yaml
action_type: manifest
stack: cabinet
```

### Wiki read (stack=wiki)
```yaml
action_type: get
path: /projects/my-page
stack: wiki
```

---

## Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `action_type` | yes | Action to perform. Aliases: `read`=`get`, `write`=`upsert`, `scan`=`list` |
| `stack` | no | `cabinet` (default) \| `wiki` \| `all` |
| `volume_entity_id` | for cabinet | Real cabinet sensor entity_id (e.g. `sensor.zenos_expansion_cabinet_1`). Do not pass resolver sensors. |
| `path` | varies | Drawer key (cabinet) or page path (wiki). Segments slugified individually; `/` preserved as CabCeption separator |
| `value` | for writes | Drawer content. Native dict/list/string/number — do NOT `\| tojson` before passing |
| `title` | for create/upsert | Document or drawer title |
| `description` | no | Short description |
| `merge` | no | `true` = force deep-merge; `false` = force replace; omit = action default |
| `tags` | no | HA label names. Comma-separated string or list. Additive on write; use `relabel` to fully replace |
| `create_label` | no | `true` = auto-create HA labels not yet in registry |
| `destination_path` | for move/copy | Target key (same-stack) |
| `destination_stack` | no | `wiki` or `cabinet` for cross-stack copy/move |
| `destination_volume_entity_id` | for cross-cab | Target cabinet entity for cross-cabinet move/copy |
| `destination_cabinet` | for clone | Target cabinet entity for clone |
| `path_prefix` | no | Filter `list` results to keys starting with prefix |
| `query` | for search | Text to match against key, title, content, description |
| `anchor_type` | no | Lens anchor type: `label` (default) \| `concept` \| `area_id` \| `zone` \| `person` |
| `anchor_ids` | for search/stacks_by_anchor | Comma-separated or JSON array of anchor IDs |
| `target_guid` | for set_mount | Cabinet GUID for cross-cabinet VirtualDrawer mounts |
| `fc_args` | for LiveDrawer | JSON object with `tool`, `params`, `cache` for tool-call mounts |
| `force_action` | no | `create`: overwrite existing key. `set_mount`: overwrite non-mount. `clone`: implied by `transfer_guid` |
| `expires_after` | no | ISO 8601 datetime after which drawer may be GC'd |
| `no_autoexpire` | no | `true` suppresses GC expiry |
| `no_autorecycle` | no | `true` suppresses GC recycle |
| `show_hidden_volumes` | no | `manifest`: include hidden/system volumes |
| `show_stacks` | no | `manifest`: include `online_unmounted` cabinets |
| `transfer_guid` | no | `clone`: Highlander mode — GUID travels to destination. Requires `force_action: true` |
| `caller_token` | no | Governance stub — echoed in response trace |

---

## Why FileCabinet Exists

The FileCabinet module is the crew's **write-controller**, the system's filesystem driver, the data gatekeeper.

It enforces:
- health
- safety
- consistency
- structure
- label intelligence
- schema boundaries
- child cascade integrity
- mount traversal and loop detection
- concurrent write protection

Without FileCabinet:
- drawers could corrupt
- labels could become inconsistent
- manifests could desync
- volumes could lose schema integrity
- agents could overwrite each other
- child trees would be left as orphans on delete
- mount chains would loop forever

This is the **authoritative, hardened interface** that lets the crew mutate state without breaking their world.

---

## Troubleshooting

### Cabinet data disappeared after restart
Data written to a cabinet is gone after an HA restart. This is most likely a recorder timing issue — new entities may not have completed a write cycle before the restart. Don't write anything critical to a cabinet in the same session it was created. Let the install settle first.

### Cabinet appears unavailable / health sensors show errors that don't match
Check for a `_2` duplicate in Settings → Entities. This happens when the canonical entity name was already in the registry from a previous install. Your package is wired to `sensor.zenos_dojo_cabinet` but HA is serving `sensor.zenos_dojo_cabinet_2`. You'll spin forever with no obvious error.

Fix: confirm you have a backup, then Settings → Entities → find the `_2` entity → delete or reset via UI. Restart HA. The cabinet re-attaches to the correct entity ID and state restores from recorder.

> Back up before touching the entity registry. No backup, no recovery path.

### Write returns `volume not healthy` but the cabinet looks fine
Check `_h.get('writeable', true)` in the health snapshot. A cabinet that is `online_mounted` but has `writeable: false` in its health attribute will block all writes without a visible error message. Inspect the cabinet entity's `health` attribute directly in Developer Tools → States.

### Mount traversal returns `loop_detected`
Two drawers are pointing at each other via `mount_point`. Use `remove_mount` to clear one of the pointers and re-wire the chain. The traversal aborts at depth 8 even without a strict loop.

### LiveDrawer returns `tool_call_blocked`
The `fc_args.action_type` or `fc_args.tool` is not in the `fc_mount_callout_whitelist` stored in `sensor.zenos_system_cabinet`. Check the whitelist value in Developer Tools, add the tool to the allowed list, then retry.

### dismount_cabinet blocked: pipeline is live
Turn off the summarizer pipeline switches first. The error message names the specific switches that are on. Use `zen_dojotools_systemtools` (tool=select_control, mode=toggle) or the HA helper UI to disable them, then retry the dismount.

### CabCeption children not found after move
If a `move` times out mid-cascade, the parent has moved but some children may remain at their source keys. Run `list` on the source cabinet with `path_prefix` set to the old parent key to find stranded children, then move them individually.

---

## Tapestry — `mode=weave` (v6.2.0)

Tapestry is a read-only report compiler on FileCabinet. It composes multiple cabinet drawers — from any combination of cabinets — into a single compiled nested dict. Definitions are stored as labeled drawers (`zen_tapestry` label) and executed by name.

### Three Tapestry Modes

| Mode | What It Does |
|------|-------------|
| `weave` | Execute a stored or inline definition. Pass `weave_drawer=` (name of a drawer containing a definition), `weave_label=` (find definition by label), or inline `weave_sources=` (JSON sources array). |
| `weave_preview` | Dry run: compile + size check + depth_map + `preview_token`. No side effects. Always run this before `weave_save`. |
| `weave_save` | Write a definition to a cabinet drawer. Requires `preview_token` (generated by `weave_preview`, valid for the current clock minute) + `force_action: true`. |

### Tapestry Fields

| Field | Description |
|-------|-------------|
| `weave_drawer` | Name of a stored definition drawer to execute |
| `weave_label` | Label to find a stored definition — first match wins |
| `weave_sources` | Inline JSON sources array (bypasses stored definition) |
| `weave_max_depth` | Max recursion depth: 1–3 (default 2). Depth 3 = 3 unrolled passes. |
| `preview_token` | Required by `weave_save`. Derived from `weave_preview` — no crypto, clock-minute + source fingerprint. Expires at minute rollover. |

### Definition Schema

Stored as a drawer value (JSON):

```json
{
  "sources": [
    {
      "key": "os",
      "cabinet": "sensor.zenos_system_cabinet",
      "drawer": "os_release",
      "mask": ["os_version", "build"],
      "rename": {"os_version": "version"}
    },
    {
      "key": "nested",
      "weave": "another_tapestry_drawer_name"
    }
  ],
  "description": "optional label shown in preview"
}
```

`mask` is required on any source drawer with more than 20 keys. `rename` maps old keys to new keys after masking.

### Hard Limits

- Max 10 sources per definition
- Max depth 3 (unrolled recursion — 3 passes, cycle detection across all passes)
- Output truncated at 8KB (`truncated: true` + `dropped_sources` list in response)
- Any output >2KB: response includes `virtual_drawer_hint` with a call shape to save the result as a LiveDrawer

### Tapestry Implementation Notes

**CabCeption path keys must not be slugified whole.** `key | lower | slugify` converts `/` to `-` — a lookup for `scratch/zen_os_snapshot` becomes `scratch-zen-os-snapshot` and misses. Correct pattern: `key.split('/') | map('lower') | map('slugify') | join('/')`. Applies at all weave compile pass sites (definition load, passes 0/1/2 for both drawer and nested weave reads).

**`weave_label` reads `_label_index`, not HA entity labels.** FC's `tags:` on upsert updates the cabinet's `_label_index` drawer — it does NOT apply HA entity labels to the cabinet sensor. `weave_label` looks up the label key in each cabinet's `_label_index` drawer directly, then reads only the listed drawer names. `label_entities()` is not the canon here.

### What Tapestry Is For

Tapestry is how you build a composed knowledge surface from multiple live drawers without writing a custom tool call for each combination. The first real Tapestry definition (`scratch/zen_os_snapshot`) composes system cabinet + household cabinet slices into a single coherent snapshot. Because the definition is stored as a labeled drawer, Friday can discover and execute it by name — she doesn't need to know which cabinets are involved.

---

## `fleet` and `expansion_sitrep` Modes

`fleet` and `expansion_sitrep` are MCP-exposed modes for cabinet fleet management and expansion cabinet status respectively. Both modes delegate to the Sutra and appear in the MCP schema selector after a script reload.

| Mode | What It Does |
|------|-------------|
| `fleet` | Lists all known cabinet volumes with health, mount state, GUID, and capacity. System-wide view. |
| `expansion_sitrep` | Returns the status of all expansion cabinet mounts: which are online_mounted, which are online_unmounted (stacks), which have pending mount activation. |

---

## Version History

| Version | Change |
|---------|--------|
| v6.2.0 | **CabCeption + Tapestry.** Nested drawer trees via `/` path separator. `get` returns `subdrawers`+`has_children` when children exist (additive). `move`/`delete`/`copy` cascade to all children via prefix scan. `move` child writes verified before source delete. `copy` cascades subtree. `inspect` surfaces subdrawers inline. **Tapestry:** `weave`/`weave_preview`/`weave_save` modes — multi-cabinet drawer composer; definitions stored as `zen_tapestry`-labeled drawers; cycle detection; depth 3 unrolled; `preview_token` gate on save. `move`/`copy` now blocked on mounted drawers (VirtualDrawer/LiveDrawer) by default — requires `force_action: true` to transfer mount config intact. `fleet` and `expansion_sitrep` added to MCP enum. |
| v6.1.2 | v5-parity repair pass (A1–A11). |
| v6.0.0 | **Major rewrite.** Two-script architecture: `zen_dojotools_filecabinet` (MCP face) + `zen_sutra_filecabinet` (internal terminus). `stack=` routing. `stack=wiki` absorbs the wiki surface — `zen_dojotools_wikijs` retired. `upsert` action added. LiveDrawer / VirtualDrawer support via `set_mount`/`remove_mount`. `dict-in/dict-out` contract (no `tojson` before passing value). `content` key retired from GET response — callers read `.value`. `mount_cabinet`/`dismount_cabinet` actions. Lens Bus integration (`stacks_by_anchor`, `register`, `unregister`, `inspect`, `audit`). |
| v4.7.2 | `key='*'` preserved through slugify; both `'*'` and `''` route to directory listing without erroring. |
| v4.7.1 | **Write-lockout hotfix.** `mode: queued / max: 2` prevents single-slot deadlock. Event dispatch `\| tojson` on CREATE + UPDATE. Verification comparison `\| tojson` on both sides — type-safe JSON string comparison. Root cause: v4.7.0 stored raw Python repr in cabinets; wait_template type mismatch caused 30s timeout → "Already running" floods → all writes dropped. v4.7.0 must not be deployed. |
| v4.7.0 | Global normalization. `set_timestamp` defaults to `true`. All writes produce `{value, timestamp, meta}` struct. `_` prefix reads no longer silently strip the underscore when `force_action` is omitted. |

---

## Cross-References

- [Cabinet Specification](../cabinets/cabinet_spec.md) — valid cabinet, drawer, person, family, household, and AI-user shapes
- [HyperIndex Overview](../zen_hyperindex/zen_hyperindex_overview.md) — how indexed entities and drawer blurbs feed graph reasoning
- [DojoTools Index](zen_dojotools_index_readme.md) — search/correlation layer that can request drawer blurbs through Inspect
- [DojoTools Inspect](zen_dojotools_inspect_readme.md) — expansion layer that asks FileCabinet for label-targeted drawer context
