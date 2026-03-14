# Zen DojoTools Core — 4.2.0

*FileCabinet garbage collector — drawer lifecycle management*

---

## Overview

`dojotools_core.yaml` contains the FileCabinet garbage collector: `zen_dojotools_filecabinet_gc`.

The GC is the custodian of cabinet hygiene. It runs daily at midnight (triggered by the Scheduler) and enforces the drawer lifecycle — expiring active drawers, deleting expired hidden drawers, and evicting legacy stale entries. It is also MCP-exposed for on-demand runs and manual drawer recycling.

---

## zen_dojotools_filecabinet_gc

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `action_type` | select | `gc` | `gc`, `recycle`, `hide`, `unhide` |
| `key` | text | — | Drawer key (required for recycle, hide, unhide) |
| `dry_run` | boolean | `false` | Report what would change without making changes (gc only) |
| `max_age_hours` | number | `24` | Legacy eviction threshold in hours (1–168) |
| `post_supersummary` | boolean | `true` | Trigger SuperSummary after eviction completes |
| `volume_entity_id` | entity (sensor) | — | Target cabinet. Defaults to Kata cabinet |

---

## Drawer Lifecycle

The GC enforces a Unix-style visibility model:

| State | Key Pattern | Lifecycle Rule |
|---|---|---|
| Active | `foo` | Expires → hidden |
| Hidden | `.foo` | Expires → deleted |
| System | `_foo` | Never touched by GC |

Expiry is driven by `meta.expires_after` in the drawer's metadata. If that field is absent, the legacy `timestamp` field is used with the `max_age_hours` threshold.

### Full Lifecycle Rules

```
Active drawer with expires_after in the past  →  hide (rename foo → .foo)
Hidden drawer with expires_after in the past  →  hard delete (.foo removed)
Active drawer with legacy timestamp > max_age_hours, no expires_after  →  hard delete
Drawer with meta.no_autoexpire: true  →  never touched
System prefix (_foo) or explicit protected key  →  never touched
```

### Protected Drawers

The following are always skipped regardless of expiry state:

- Any key starting with `_`
- `AI_Cabinet_VolumeInfo`
- `zen_template`
- `zen_summary`
- `kata_template`
- `kata_structure`

---

## Actions

### `gc`

Full cabinet scan. Processes all drawers in all AI Data Storage cabinets (resolved dynamically — no hardcoded list).

```yaml
action_type: gc
```

Run with `dry_run: true` first to see what would be evicted before committing:

```yaml
action_type: gc
dry_run: true
```

```json
{
  "status": "dry_run",
  "action": "gc",
  "cabinets_scanned": ["sensor.zenos_kata_cabinet", "..."],
  "would_evict_count": 3,
  "kept_count": 18,
  "protected_count": 5
}
```

After a real run with evictions, GC optionally fires `script.zen_dojotools_supersummary` to update the active summary with post-eviction state. Controlled by `post_supersummary` (default `true`).

---

### `recycle`

Hides a specific drawer and sets its expiry to now + 24 hours. The drawer will be hard-deleted by the next GC run after expiry.

```yaml
action_type: recycle
key: hot_tub_kata_2026_03_10
```

Use recycle when you want to retire a drawer gracefully — it stays accessible (hidden) briefly, then auto-deletes.

---

### `hide`

Moves a drawer to hidden state without changing its expiry.

```yaml
action_type: hide
key: some_drawer
```

Useful for archiving a drawer you might want to retrieve later. Unlike recycle, no expiry is set — the drawer stays hidden until explicitly unhidden or until a future `expires_after` timestamp is reached.

---

### `unhide`

Restores a hidden drawer to active visibility.

```yaml
action_type: unhide
key: some_drawer
```

Pass the base name (without the leading dot). The GC finds `.some_drawer`, copies it to `some_drawer`, and removes the hidden copy.

---

## Schedule

The GC runs automatically at `daily_midnight` via the Scheduler. No manual trigger needed for routine operation.

The daily run uses default settings:
- `action_type: gc`
- `post_supersummary: true`
- `max_age_hours: 24`

---

## Response Format

### gc

```json
{
  "status": "complete",
  "action": "gc",
  "cabinets_scanned": ["sensor.zenos_kata_cabinet"],
  "max_age_hours": 24,
  "dry_run": false,
  "supersummary_triggered": true
}
```

### recycle / hide / unhide

```json
{
  "status": "complete",
  "action": "recycle",
  "drawer": "some_drawer",
  "hidden_as": ".some_drawer",
  "expires_after": "2026-03-15T00:00:00+00:00"
}
```

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `sensor.zen_kata_cabinet_resolved` | Default GC target cabinet |
| `library_index.jinja` → `parse_index_command` | Enumerate all AI Data Storage cabinets |
| `script.zen_dojotools_supersummary` | Post-eviction summary trigger |
| HA `set_variable_legacy` / `remove_variable_legacy` events | Drawer create / delete |
