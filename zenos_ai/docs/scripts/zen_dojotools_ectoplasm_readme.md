# Zen DojoTools Ectoplasm — 6.0.0

*Spook/HA extended surface wrapper — repairs, areas, floors, entity lifecycle, labels, integrations*

---

## What Ectoplasm Does

Ectoplasm is ZenOS-AI's interface to the extended HA surface provided by the [Spook](https://spook.boo) integration. It handles structural operations that the standard HA service layer does not expose through Jinja or the normal script API:

- **Repairs** — create, remove, ignore, and list HA repair issues
- **Areas** — create, delete, assign/unassign devices and entities
- **Floors** — create, delete, assign/unassign areas
- **Entity lifecycle** — hide, unhide, disable, enable, rename
- **Device lifecycle** — disable, enable
- **Label assignment** — assign/unassign labels to areas and devices (entity label ops stay in `zen_dojotools_labels`)
- **Integration config entries** — disable, enable
- **Housekeeping** — orphan entity cleanup

**Requires:** Spook integration — a listed ZenOS prerequisite. All write actions call Spook services internally.

**Fully MCP-exposed.** Friday can use it directly for structural home management tasks.

---

**`mode` is now the primary selector (v6.0.0), per the project-wide standardization on `mode:` across all dojotools.** `action_type` still works — `mode` takes priority, `action_type` is the fallback — but is a deprecated alias going forward. Every example below using `action_type:` still works unchanged.

## The Confirm Gate

All write actions are **confirm-gated**:

- Omit `confirm_action` or set it to `false` → returns `status: start` with a **preview block** describing exactly what would happen, without executing anything.
- Set `confirm_action: true` → executes.

**Read actions** (`repair_list`, `ghost_list`) execute immediately — no confirm required.

This pattern is intentional. Structural operations (deleting an area, renaming an entity, disabling an integration) are hard to undo. Preview first, confirm second.

```yaml
# Preview (no-op)
action: script.zen_dojotools_ectoplasm
data:
  action_type: area_create
  area_name: "Guest Suite"
  floor_id: ground_floor

# Execute
action: script.zen_dojotools_ectoplasm
data:
  action_type: area_create
  area_name: "Guest Suite"
  floor_id: ground_floor
  confirm_action: true
```

---

## Action Reference

### Reads (no confirm required)

| Action | What It Does | Key Fields |
|---|---|---|
| `repair_list` | List active HA repair issues. Optional domain filter. | `domain` |
| `ghost_list` | List ghost entities (entities whose device or integration no longer exists). | — |

---

### Repairs

| Action | What It Does | Key Fields |
|---|---|---|
| `repair_create` | Create a new HA repair issue. `issue_id` is auto-prefixed with `zenos_` unless already prefixed. | `issue_id`, `title`, `issue_description`, `severity`, `domain` |
| `repair_remove` | Remove a repair issue by ID. | `issue_id` |
| `repair_ignore_all` | Ignore all current repair issues. | — |
| `repair_unignore_all` | Unignore all repair issues. | — |

**Severity values:** `error`, `warning`, `info`. Note: `info` maps to `information` internally for Spook compatibility.

---

### Areas

| Action | What It Does | Key Fields |
|---|---|---|
| `area_create` | Create a new area. Optionally assign to a floor. | `area_name`, `floor_id` (optional) |
| `area_delete` | Delete an area by ID. | `area_id` |
| `area_assign_device` | Assign a device to an area. | `area_id`, `device_id` |
| `area_unassign_device` | Remove a device from its current area. | `device_id` |
| `area_assign_entity` | Assign an entity to an area. | `area_id`, `entity_id` |
| `area_unassign_entity` | Remove an entity from its current area. | `entity_id` |

---

### Floors

| Action | What It Does | Key Fields |
|---|---|---|
| `floor_create` | Create a new floor. | `floor_name`, `floor_level` (optional), `floor_id` (optional — auto-generated if blank) |
| `floor_delete` | Delete a floor by ID. Does not delete its areas. | `floor_id` |
| `floor_assign_area` | Assign an area to a floor. | `floor_id`, `area_id` |
| `floor_unassign_area` | Remove an area from its floor. | `area_id` |

**Floor level convention:** `0` = ground, `1` = first/upper, `-1` = basement. Optional — purely informational.

**Fixed (v6.0.0):** `floor_assign_area`/`floor_unassign_area` previously called the underlying HA service directly and reported `status: success` unconditionally — a silent no-op (e.g. a non-existent floor or area) was reported as a success with no way to detect it. Both actions now route through the REST API, check the actual HTTP status, and report `status: error` with the real response body when the assignment didn't take.

---

### Entity Lifecycle

| Action | What It Does | Key Fields |
|---|---|---|
| `entity_hide` | Hide an entity from the default UI view. | `entity_id` |
| `entity_unhide` | Unhide a previously hidden entity. | `entity_id` |
| `entity_disable` | Disable an entity (stops polling, removes from state machine). | `entity_id` |
| `entity_enable` | Re-enable a disabled entity. | `entity_id` |
| `entity_rename` | Set a new friendly name for an entity. | `entity_id`, `new_name` |

---

### Device Lifecycle

| Action | What It Does | Key Fields |
|---|---|---|
| `device_disable` | Disable a device (disables all its entities). | `device_id` |
| `device_enable` | Re-enable a disabled device. | `device_id` |

---

### Label Assignment

These actions assign and unassign HA labels to areas and devices. Entity-level label ops (`label_assign_entity`, etc.) remain in `zen_dojotools_labels`.

| Action | What It Does | Key Fields |
|---|---|---|
| `label_assign_area` | Assign a label to an area. | `label_id`, `area_id` |
| `label_unassign_area` | Remove a label from an area. | `label_id`, `area_id` |
| `label_assign_device` | Assign a label to a device. | `label_id`, `device_id` |
| `label_unassign_device` | Remove a label from a device. | `label_id`, `device_id` |

---

### Integrations

| Action | What It Does | Key Fields |
|---|---|---|
| `integration_disable` | Disable an integration config entry. | `config_entry_id` |
| `integration_enable` | Re-enable a disabled integration config entry. | `config_entry_id` |

`config_entry_id` can be found via the REST API (`/api/config/config_entries/entry`) or via Inspect's `device_info` mode (which includes `config_entries` for each device).

---

### Housekeeping

| Action | What It Does | Key Fields |
|---|---|---|
| `orphan_cleanup` | Remove orphaned entities — entities whose device or parent integration no longer exists. | — |

Use `ghost_list` first to preview what would be removed.

---

### Input Numbers (v6.0.0, requires Spook v5.0.0+)

| Action | What It Does | Key Fields |
|---|---|---|
| `input_number_create` | Create a new `input_number` helper. | `number_name` (required), `number_entity_id` (optional — auto-generated if blank), `number_min`, `number_max`, `number_initial`, `number_step`, `number_display_mode`, `number_unit`, `number_icon` |
| `input_number_delete` | Delete an `input_number` helper by entity_id. | `number_entity_id` |

Same confirm-gate pattern as everything else here — preview first, `confirm_action: true` to execute.

---

## Field Reference

| Field | Required | Used By | Description |
|---|---|---|---|
| `mode` | Yes (or `action_type`) | All | Operation to perform (see action tables above). Primary selector as of v6.0.0. |
| `action_type` | Deprecated alias | All | Deprecated fallback for `mode` — still fully supported. |
| `issue_id` | Conditional | `repair_create`, `repair_remove` | Repair issue ID. Auto-prefixed `zenos_` if not already. |
| `title` | Conditional | `repair_create` | Human-readable repair title |
| `issue_description` | Conditional | `repair_create` | Repair body text |
| `severity` | No | `repair_create` | `error`, `warning`, `info`. Default: `warning` |
| `domain` | No | `repair_create`, `repair_list` | HA integration domain. Defaults to `zenos` for create. Filter for list. |
| `area_id` | Conditional | Area/floor/label ops | HA area ID (slug) |
| `area_name` | Conditional | `area_create` | Human-readable area name |
| `floor_id` | Conditional | Floor ops, `area_create` | HA floor ID (slug) |
| `floor_name` | Conditional | `floor_create` | Human-readable floor name |
| `floor_level` | No | `floor_create` | Integer level. Convention: `0`=ground, `1`=first, `-1`=basement |
| `entity_id` | Conditional | Entity ops, `area_assign_entity`, `area_unassign_entity` | HA entity ID |
| `device_id` | Conditional | Device ops, area/label device ops | HA device ID |
| `label_id` | Conditional | Label assignment ops | HA label ID (slug) |
| `new_name` | Conditional | `entity_rename` | Replacement friendly name |
| `config_entry_id` | Conditional | Integration ops | Integration config entry ID |
| `confirm_action` | No | All writes | `true` to execute. Omit/`false` returns preview. Default `false`. |
| `caller_token` | No | All | Opaque token echoed in response for correlation |

---

## Event Emission

All write actions (on execute, not preview) fire a `zen_event` with kind `ectoplasm_<action_type>`:

```json
{
  "kind": "ectoplasm_area_create",
  "action_type": "area_create",
  "area_name": "Guest Suite",
  "caller_token": "...",
  "status": "ok"
}
```

Read operations do not emit events.

---

## Division of Labor

| Concern | Tool |
|---|---|
| Label create / delete / assign to entities | `zen_dojotools_labels` |
| Label assign to areas / devices | `zen_dojotools_ectoplasm` |
| Entity inspection (read-only) | `zen_dojotools_inspect` |
| Entity/area/floor/device read | `zen_dojotools_inspect` registry modes |
| Cabinet management | `zen_admintools_cabinetadmin` |
| HA repair issues / ghost cleanup | `zen_dojotools_ectoplasm` |

---

## Dependencies

| Dependency | Purpose |
|---|---|
| Spook integration | All write operations — Spook exposes the extended HA service surface |
| `script.zen_dojotools_event_emitter` | Post-write event emission |
