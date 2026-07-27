# Zen DojoTools Provisioner — v5.1.0

**File:** `packages/zenos_ai/dojotools/dojotools_provisioner.yaml`
**Script:** `zen_dojotools_provisioner`
**MCP-exposed**

Provisions, deprovisions, or replaces a cabinet-backed identity (`ai_user` / `user` / `family` / `household`) — pulling a stacks cabinet into service or returning one to the stacks pool. This is the write-path counterpart to the stacks model: cabinets live as `online_unmounted` spares until provisioned into a typed, labeled, `online_mounted` identity.

---

## Modes

| Mode | Description |
|------|-------------|
| `provision` (default) | Pulls a stacks cabinet (`init` or `online_unmounted`) into service. Validates GUID, applies the type label, mounts, optionally preloads profile data, fires an identity manifest rebuild. Rolls back (strips the label) if the mount doesn't land within the health-gate timeout. |
| `deprovision` | Dismounts a cabinet, strips its type label, returns it to the stacks pool. Blocked if the cabinet holds a `zen_default_*` label — transfer the default elsewhere first. |
| `replace` | Deprovisions `replace_cabinet`, then provisions `target_cabinet` — same `cab_type` for both. Cheaper than a raw deprovision+provision because stacks make cabinet turnover a labeling operation, not a data migration. |

---

## Fields

| Field | Required | Description |
|-------|----------|--------------|
| `mode` | No | `provision` (default) / `deprovision` / `replace`. |
| `cab_type` | provision, replace | `ai_user`, `user`, `family`, or `household` — determines which type label gets applied. |
| `target_cabinet` | Yes | Cabinet entity to provision (must be `init`/`online_unmounted`) or deprovision (must be `online_mounted`). |
| `replace_cabinet` | replace only | Existing cabinet to deprovision before `target_cabinet` is provisioned. |
| `person_entity` | No | HA `person.*` entity. If given, `friendly_name` and the entity_id are preloaded into the profile drawer during provisioning. |
| `profile_payload` | No | JSON string of additional key/value pairs to press into the profile drawer. Merged with `person_entity` data if both are given. |
| `caller_token` | No | Opaque pass-through for caller correlation. |

> Known cosmetic issue: `cab_type`'s selector options list currently has `tool_manifest` listed twice (copy-paste duplicate) — harmless (just renders the same dropdown option twice), not yet cleaned up.

---

## Provision Flow

1. **Stacks gate** — `target_cabinet` must be `init` or `online_unmounted`. A cabinet in `init` (no VolumeInfo yet) is auto-stamped via `zen_admintools_cabinetadmin_factory` first, then waited on (15s timeout) to reach `online_unmounted` before continuing.
2. **GUID gate** (FG-38 safe — two-round VolumeInfo normalization), runs *before* any label is applied so a failure needs no rollback:
   - GUID must be present (`AI_Cabinet_VolumeInfo.id`).
   - Must match UUID format.
   - Must be unique across every currently `online_mounted` `Zen Cabinet`-labeled entity.
3. **Label + mount** — applies the type label (`zen_ai_user_cabinet` / `zen_user_cabinet` / `zen_family_cabinet` / `zen_household_cabinet`) plus the universal `zen_cabinet` parent label (required for health sensors and `label_entities()` enumeration to see the cabinet at all), then mounts (`meta.mounted: true`).
4. **Health gate** — waits up to 10s for the cabinet to reach `online_mounted`. **Rolls back** (strips the type label, cabinet returns to stacks) on timeout rather than leaving a half-mounted cabinet with a label but no confirmed state.
5. **Directory preload** (optional) — writes a `profile`/`_user_profile`/`_family_profile`/`_household_profile` drawer (key depends on `cab_type`) from `person_entity` + `profile_payload`, merged.
6. **Essence seed** — `ai_user` and `user` cab types get a blank `zenai_essence` drawer seeded (core GUID/timestamps, empty jacket fields) so Profile Editor read/write is live immediately — `mode: write` with no fields self-heals this later if it's ever deleted.
7. Fires `zen_event` (`kind: cabinet_provisioned`) and a separate `identity_manifest_rebuild` event.

## Deprovision Flow

1. **Mounted gate** — `target_cabinet` (or `replace_cabinet`, in `replace` mode) must be `online_mounted`.
2. **Default-label gate** — blocked if the cabinet holds any `zen_default_household_cabinet`/`zen_default_family_cabinet`/`zen_default_user_cabinet`/`zen_default_ai_user_cabinet` label. Transfer the default to another cabinet first — this script won't silently orphan a default.
3. Strips the type label and the `zen_cabinet` parent label, dismounts (`meta.mounted: false`), waits up to 10s to confirm `online_unmounted`.
4. Fires `zen_event` (`kind: cabinet_deprovisioned`) and `identity_manifest_rebuild`.

## Replace Flow

Runs deprovision against `replace_cabinet`, then provision against `target_cabinet` with the same `cab_type`, and returns both sub-results combined.

---

## Response Shape

```yaml
# provision
status: success | error
message: "Cabinet 'sensor.x' provisioned as user. GUID: ..."
cabinet: sensor.x
cab_type: user
guid: "..."
profile_loaded: true/false
caller_token: "..."

# deprovision
status: success | error
message: "Cabinet 'sensor.x' deprovisioned (online_unmounted)."
cabinet: sensor.x
verified: true/false
caller_token: "..."

# replace
status: success
message: "replace complete. Deprovisioned 'sensor.old', provisioned 'sensor.new' as user."
deprovisioned: { ...deprovision result... }
provisioned: { ...provision result... }
caller_token: "..."
```

Error responses (`missing target_cabinet`, `invalid cab_type`, `missing replace_cabinet`, `target not in stacks`, `guid missing`/`guid invalid format`/`guid collision`, `health gate timeout`, `not mounted`, `default label gate`) all return `{status: error, message, caller_token}` and stop immediately — no partial state changes past whatever gate failed.

---

## Example

```yaml
mode: provision
target_cabinet: sensor.zen_expansion_cab_04
cab_type: user
person_entity: person.new_household_member
```

---

## Related

- [Cabinet Spec](../cabinets/cabinet_spec.md)
- [Building a KFC](../kung_fu/building_a_kfc.md) — cabinet/label mechanics this script relies on.
