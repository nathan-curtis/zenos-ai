# Zen DojoTools AdminTools — v5.3.1

*Ring-2 administrative tools: component registration, cabinet repair, template management, and prompt configuration*

---

## Overview

AdminTools is the **Ring-2 administrative layer** of ZenOS-AI. It handles tasks that fall outside normal runtime behavior: repairing cabinets, pressing schema templates, and loading the AI's identity substrate.

**All tools in this module are admin-only — not MCP-exposed.** They are not accessible to the AI agent and should not be called by Friday during normal operation. Run them via HA Developer Tools → Services.

For KFC component registration (writing Dojo drawers), use `zen_dojotools_scribe` — it is a DojoTools script, **MCP-exposed**, and the correct tool for both Friday and operators.

---

## Scripts in This Module

| Script | Version | MCP-Exposed | Purpose |
|---|---|---|---|
| `zen_admintools_reset_template` | 1.1.0 | **No** | Press zen_template and kfc_template into cabinets |
| `zen_admintools_reset_labels` | 4.5.0 | No | Nuclear: delete all zen_ labels and assignments, trigger Flynn rebuild |
| `zen_admintools_cabinetadmin` | 4.6.0 | No | Inspect, restore, reset, hammer, init, expand_drawer, repair_volumeinfo, or reset_all Ring-0 cabinets |
| `zen_admintools_cabinetadmin_factory` | 1.x | No | Factory-stamp or repair a cabinet's VolumeInfo drawer |
| `zen_admintools_kfc_migration_press` | 1.1.0 | No | One-time migration: seed scheduling fields into KFC drawers |
| `zen_admintools_prompt_loader` | 5.2.0 | No | Load versioned Cortex, Directives, and Purpose (v43 = Rule Zero (default/latest), v42 = The Answer, v40 = Room First, v38 = Kata First). Also manages `zen_summarizer_act_whitelist`, `zen_summarizer_seed_whitelist`, and `fc_mount_callout_whitelist` via `mode=whitelist`. |
| `zen_admintools_run_repair` | 4.5.6 | **No** | Human-confirmed passthrough to versioned maint/ repair scripts |

> **KFC registration:** `zen_dojotools_kungfu_writer` has been removed. Use `zen_dojotools_scribe` — see `dojotools_scribe.yaml` for full documentation.

---

## zen_admintools_reset_template

Presses the `zen_template` seed into the Kata cabinet and the `kfc_template` seed into the Dojo cabinet.

Called automatically by **Flynn gate-3** if the templates are missing at boot. Can be re-run manually if templates become corrupted — it is fully idempotent.

**Not MCP-exposed.** Run via HA Developer Tools → Services, or trigger Flynn's gate-3 check.

### Behavior

- Reads the Kata cabinet and Dojo cabinet (resolved via labels `zen_summary,zen,summary` and `kfc,dojo,zen`)
- If `zen_template` is absent from the Kata cabinet → writes seed template
- If `kfc_template` is absent from the Dojo cabinet → writes seed template
- Uses `force_action: true` — will overwrite if explicitly needed

### When to Use

- After a fresh install if `sensor.zen_agent_health` reports a schema-related error
- If the `kfc_template` drawer in the Dojo appears corrupted or empty
- If Flynn's gate-3 fires a `dojo_loaded: warn` event

---

## zen_admintools_reset_labels

⚠️ **NUCLEAR** — deletes all `zen_` labels and all their entity assignments. No undo.

Use this when label IDs are corrupt, have wrong names, or you need a full label reinstall. For wiping assignments only (labels survive), use `zen_dojotools_labels` with `action_type: reset` instead.

**Not MCP-exposed.** Admin use only.

### What It Does

1. Untags all entities from every label with a `zen_` ID
2. Deletes every `zen_` label (`zen_*` IDs only — non-zen labels are never touched)
3. Fires `zen_resolver_refresh` so resolver sensors re-evaluate immediately

Flynn re-engages automatically: `zen_label_health` flips to `critical` → Stepgate Sentinel fires → Gate 0 recreates labels → Gate 1 re-assigns. No HA restart required (HA 2024.x+ propagates label creation live).

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `confirm` | boolean | `false` | Must be `true` — refuses without it |

### Full Nuclear Reset Sequence

To completely rebuild from scratch (labels + cabinets):

```
1. zen_admintools_reset_labels          — nuke + rebuild all zen_ labels
2. zen_admintools_cabinetadmin          — op: reset_all (wipe cabinets + reseed + Flynn)
```

Two tool calls. Order matters — labels first, then cabinets.

---

## zen_admintools_cabinetadmin

Ring-2 cabinet maintenance tool. Provisions new expansion cabinets, inspects cabinet state, recovers broken cabinets, and manages mount state. Also handles Ring-0 system cabinet repair and nuclear resets.

**Not MCP-exposed.** Admin use only. Destructive modes require explicit safety gates — do not bypass them. Run `mode: help` for a full operation guide.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `mode` | select | `help` | See modes table below |
| `target_cabinet` | entity (sensor) | — | Cabinet to operate on; leave empty for Ring-0 scope (inspect) or not applicable (mount_status, reset_all) |
| `cab_type` | text | `''` | Cabinet type for `init` and `hammer + confirm_init`. E.g. `AI Data Storage Cabinet`, `AI Household Cabinet` |
| `confirm_action` | boolean | `false` | Required for `init` when target state is `unknown` (no recorder history) |
| `confirm_init` | boolean | `false` | `hammer` only — re-stamp fresh VolumeInfo immediately after wipe. Set `cab_type` accordingly |
| `hammer_ok` | boolean | `false` | Required `true` for `hammer` mode |
| `confirm` | boolean | `false` | Required `true` for `reset_all` |
| `source_cabinet` | entity (sensor) | — | `expand_drawer` only — cabinet the drawer currently lives on (target_cabinet is the destination) |
| `source_drawer` | text | — | `expand_drawer` only — leaf drawer key to migrate. Subtree/Cabception parent keys are rejected |
| `space_threshold` | number | — | `expand_drawer` only — abort if either source or target cabinet would cross this capacity percentage |

### Modes

| Mode | Destructive | Description |
|---|---|---|
| `help` | No | Return structured guide: all modes, when to use, field reference |
| `inspect` | No | Read all drawers on the target cabinet and return their values. Always run before any destructive op |
| `init` | Conditional | Stamp a virgin cabinet with VolumeInfo + GUID. Refuses if already initialized (`good`) or has residue (`potentially_bad` — hammer first). Requires `confirm_action: true` when state is `unknown` |
| `hammer` | **Yes** | Clear all drawers + stamp `_hammered` marker. Set `confirm_init: true` to re-stamp immediately. Requires `hammer_ok: true` |
| `restore` | No | Attempt recovery of a cabinet that lost VolumeInfo after HA restart |
| `reset` | **Yes** | Clear all drawers with no audit marker |
| `mount_status` | No | Return mounted/unmounted state for all `zen_cabinet` entities. No target needed |
| `repair_mount` | No | Force `meta.mounted: true` on a stuck cabinet. Does not touch drawer content |
| `repair_dismount` | No | Force `meta.mounted: false` on a stuck cabinet. Mirror of `repair_mount` |
| `expand_drawer` | **Yes** (has rollback) | Atomically migrate one leaf drawer from `target_cabinet` to an expansion cabinet: validates the key is a leaf (rejects subtrees/Cabception children), checks space threshold on both source and target, copies the value, replaces the source entry with a `mount_point` pointer, then round-trip verifies the mount resolves through `zen_dojotools_filecabinet` before declaring success. Any failure at any step rolls back (restores original value, deletes target copy). Requires `source_cabinet`, `source_drawer`, optional `space_threshold` |
| `reset_all` | **NUCLEAR** | Wipe all Ring-0 cabinets + reseed schemas + trigger Flynn bootstrap. `confirm: true` required. User and expansion cabinets untouched |
| `flip_schema_version` | No | Toggle `cab_schema_version` in syscab (0=legacy, 1+=mount-aware). Controls Flynn operating mode |
| `repair_volumeinfo` | No | Targeted repair for a cabinet whose `VolumeInfo` is a JSON string instead of a mapping — parses and rewrites as a proper dict. Also self-heals the missing-header case (cabinet in `init` state with no VolumeInfo at all): stamps a fresh header via `cabinetadmin_factory`, no `hammer`/rearm needed. Skips silently if VolumeInfo is already a valid mapping |

### Init classifier

`init` mode classifies the target before acting:

| Class | Conditions | Action |
|---|---|---|
| `virgin` | No VolumeInfo, no `_label_index`, no `_context`, no non-system drawers, no GUID | Delegate to `cabinetadmin_factory` → stamp |
| `good` | VolumeInfo present + GUID present | Refuse: `already_initialized` |
| `potentially_bad` | Any other residue | Refuse: `repair_required` — hammer first |

`unavailable` state (recorder not yet restored) always blocks init. `unknown` state (no recorder history) blocks unless `confirm_action: true`.

### Ring-0 Cabinets

When `target_cabinet` is empty, all 14 Ring-0 cabinets are targeted:

```
sensor.zenos_system_cabinet
sensor.zenos_system_history_cabinet
sensor.zenos_dojo_cabinet
sensor.zenos_kata_cabinet
sensor.zenos_history_cabinet
sensor.zenos_scratchpad_cabinet
sensor.zenos_default_household_cabinet
sensor.zenos_default_household_history_cabinet
sensor.zenos_default_family_cabinet
sensor.zenos_default_family_history_cabinet
sensor.zenos_default_user_cabinet
sensor.zenos_default_user_history_cabinet
sensor.zenos_default_ai_user_cabinet
sensor.zenos_default_ai_user_history_cabinet
```

### When to Use

- **inspect** — routine health check; see what's in a cabinet without touching it
- **restore** — cabinet appears empty after a crash; stamp a marker to confirm it's mounted
- **reset** — nuke a cabinet's contents cleanly (e.g., clear scratchpad, reset history)
- **hammer** — full wipe with audit trail; last resort before re-init
- **init** — fresh cabinet initialization; sets VolumeInfo metadata
- **reset_all** — full nuclear cabinet reset + fires Flynn via `zen_cabinet_health` state change. Does **not** call `reset_template` directly — cabinets are still genuinely virgin at this point, and writing template content into them would make cabinetadmin's own classifier see them as `potentially_bad` instead of `virgin`. Flynn's own gate-3 calls `reset_template` idempotently once cabinets are re-stamped and resolvers are settled. If you want to customize the sequence (skip a cabinet, change order), run the steps individually — that is exactly what `reset_all` orchestrates under the hood.

---

## zen_admintools_cabinetadmin_factory

Factory tool for stamping or repairing a cabinet's `AI_Cabinet_VolumeInfo`, `_label_index`, and `_zen_relationships` drawers.

Use this when a cabinet is missing its metadata header — for example, after creating a new cabinet entity that has never been initialized, or after a cabinet loses its VolumeInfo drawer.

**Not MCP-exposed.** Admin use only.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `confirm_action` | boolean | `false` | Must be `true` — script exits otherwise |
| `cabinet_entity` | entity (sensor) | — | Target cabinet |
| `cabinet_type` | select | `AI Data Storage Cabinet` | Cabinet class (determines flag auto-profile) |
| `cabinet_guid` | text | — | Optional; generates new GUID if blank |
| `force_new_guid` | boolean | `false` | Force GUID regeneration even if one exists |
| `auto_profile` | boolean | `true` | Auto-detect security flag profile from cabinet type |
| `explicit_flag_profile` | select | `standard` | Override flag profile: `system`, `secure`, `public`, `standard` |
| `mount_scan` | boolean | `true` | Scan existing drawers for `[mount:GUID]` links |
| `owner_person` | entity (person) | — | Owner reference (HA person entity) |
| `partner_person` | entity (person) | — | Partner reference |
| `family_guid` | text | — | Family GUID for ACL |

### Flag Profiles (Auto-Detected)

| Cabinet Type | Profile |
|---|---|
| Household / Family | `system` |
| User / AI User | `public` |
| Kata / Archive | `system` |
| Chat History | `secure` |
| Default | `standard` |

### When to Use

- Newly created cabinet entity that needs its VolumeInfo drawer stamped
- Cabinet metadata is corrupt or missing after an upgrade
- GUID needs to be regenerated (e.g., cloned from another install)

> **Backup first.** While cabinetadmin_factory is designed to preserve existing data (Repair/Restamp path preserves GUID and mounts), a full HA backup before any schema repair operation is strongly recommended. Cabinet data is not version-controlled — a bad stamp cannot be undone except from backup.

> **RP2 note.** On a live RP2 installation, any write to `AI_Cabinet_VolumeInfo` triggers a full state re-derivation on the cabinet sensor — the state will briefly show `init` until the boot-touch event advances it. `cabinetadmin_factory` automatically fires `cabinet_boot_touch` after every write (both Init and Repair paths), so the cabinet will advance to `online_mounted` within seconds. No manual intervention needed.

---

## zen_admintools_kfc_migration_press

One-time migration tool. Seeds `trigger_subscriptions`, `delay_seconds`, and `kata_key` into the 12 existing KFC drawers that predated the Dojo-driven scheduler (KF4 RC2).

This script has already been run on all production installs. It exists for reference and for fresh installs that import pre-KF4 drawer exports.

**Not MCP-exposed.**

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `dry_run` | boolean | `true` | If true, logs intended writes without executing them |

### Behavior

- Iterates all 12 component drawers in the Dojo cabinet
- Merges `trigger_subscriptions`, `delay_seconds`, and `kata_key` into each using `combine()` — **does not overwrite existing fields**
- Idempotent — safe to re-run

### When to Use

Only needed if you are importing KFC drawers that were written before KF4 RC2. New installs and components written via `zen_dojotools_scribe` already include these fields.

---

## zen_admintools_prompt_loader

Loads the AI's identity substrate: **Cortex**, **Directives**, and **Purpose**. Also manages the summarizer act and seed whitelists via `mode=whitelist`. These three variables define how Friday reasons, what she prioritizes, and how she accesses the knowledge graph.

**Not MCP-exposed.** This is a configuration and prompt-engineering tool, not a runtime script. The loaded values persist in the AI cabinet and are read by the prompt engine at inference time.

### What Gets Loaded

| Variable | Purpose |
|---|---|
| `Purpose` | Role definition — what ZenOS-AI is and what it manages |
| `Directives` | 14 behavioral rules: tone, safety, confirmation patterns, tool preferences |
| `Cortex` | Full reasoning substrate — schema references, DojoTools index, behavior rules, error policy, library access patterns |

### System Cabinet Authorization

`sensor.zenos_system_cabinet` (syscab) is **hard read-only** to `zen_dojotools_filecabinet` — all write actions are wire-blocked, no force bypass. The prompt loader is the designated write path for syscab. This is intentional: it prevents any agent or automation from rewriting the AI's own prompt substrate at runtime.

On every run, the prompt loader also stamps `meta.mounted: true` on syscab — ensuring the cabinet is in `online_mounted` state after load. This is idempotent and safe to re-run.

> **Backup first.** Before running the prompt loader against a production install — especially a version upgrade — take a full HA backup. The syscab Cortex is not version-controlled at the drawer level. If a load is interrupted or the wrong version is selected, restoring from backup is the only recovery path.

### When to Use

- After a fresh install, if the Cortex is empty or missing
- When upgrading the Cortex schema (new version of the reasoning contract)
- When adjusting behavioral rules for a specific deployment

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `mode` | select | `load` | `load` — stamp Purpose/Directives/Cortex into syscab. `whitelist` — manage act or seed whitelists. |
| `cortex_version` | select | `latest` | `latest` or `43` = Rule Zero (default). `42` = The Answer. `40` = Room First. `38` = Kata First. Only used when `mode=load`. |
| `ship_zen_system` | boolean | `true` | When ON (default), chains `zen_admintools_kungfu_loader` in factory mode — deploys `zen_system` and `trapper_keeper`. `taskmaster`, `alert_manager`, `camera_manager`, and `security_manager` all self-register via KF5 instead (see each tool's own `mode=kfc_manifest`) and are no longer shipped from here. Turn OFF to load the prompt only, skipping all KFC deployment. |
| `sim_mode_allowed` | boolean | `false` | Stamped into `integrations_config.identity.sim_mode_allowed` on every factory run. OS-level policy switch (see `zen_dojotools_identity resolve_caller_identity`) — `false` (default) fails closed on any simulated/shunted identity result until real Authentik/OIDC (SP1) is live. Leave off unless you deliberately want simulated identity resolution accepted platform-wide. |
| `whitelist_type` | select | — | `mode=whitelist` only. `act` = `zen_summarizer_act_whitelist`. `seed` = `zen_summarizer_seed_whitelist`. |
| `action_type` | select | `list` | `mode=whitelist` only. `list` \| `add` \| `remove` \| `reset`. |
| `item` | text | — | `mode=whitelist add/remove` only. Event kind string (act) or script name (seed). |

Use the `cortex_version` field to select which version to load. The three primitives (Purpose, Directives, Cortex) are versioned together as a set:

| Version | Codename | Notes |
|---|---|---|
| `38` | Kata First | Kata/supersummary hierarchy first. INDEX FIRST elevated. GetLiveContext last resort. |
| `40` | Room First | Room Manager `home_overview` as spatial map before any room-aware task. |
| `42` | The Answer | v10.0.0. INSTALLATION OVERRIDE (GetLiveContext blocked). WHO/WHAT/WHEN/WHERE/WHY/HOW tool map. MANAGED MACHINES directive. `inventory` replaces `grocy_helper` in core and domain tools. |
| `43` / `latest` | Rule Zero | DojoTools supersede all HA built-ins. Not preference. Authority. Domain routing table in directives. Successor to v42 'The Answer'. |

Selecting `latest` or passing no `cortex_version` loads v43.

### Whitelist Management

`mode=whitelist` manages three whitelists in syscab, selected via `whitelist_type`:

| `whitelist_type` | Drawer | Field | Entry format |
|---|---|---|---|
| `act` | `zen_summarizer_act_whitelist` | — | Event kind string |
| `seed` | `zen_summarizer_seed_whitelist` | `allowed_scripts` | Script name |
| `fc` | `fc_mount_callout_whitelist` | `allowed_tools` | `tool` (deprecated) \| `tool:*` (preferred) \| `tool:mode` (KF5 mode-scoped) |

`seed` and `act` replace the deleted `zen_admintools_summarizer_act` and `zen_admintools_summarizer_seed` standalone scripts. `fc` is the FileCabinet live-drawer callout whitelist used by KF5 self-registering tools — see [Building a KFC — KF5](../kung_fu/building_a_kfc.md#kf5-self-registering-tools).

```yaml
# Add Room Manager as a seed source
action: script.zen_admintools_prompt_loader
data:
  mode: whitelist
  whitelist_type: seed
  action_type: add
  item: zen_dojotools_room_manager
```

**`fc`-type add validation:** an `add` request containing a bare `*` (unrestricted wildcard) or a `tool:` entry with an empty mode suffix is rejected outright — the whitelist is never written, and the response is an error naming the offending entry with the expected formats (`tool:mode`, `tool:*`, or plain `tool` — deprecated). This exists specifically so a KF5 tool's `kfc_manifest` self-registration step can never accidentally (or maliciously) open a wildcard or malformed entry on the FC callout whitelist.

### KFC Schema v1.4.0 and Seed Whitelist

`zen_admintools_reset_template` now presses the v1.4.0 `kfc_template` seed into the Dojo cabinet **and** seeds `zen_summarizer_seed_whitelist` into the System cabinet if the drawer is missing (idempotent). Both are seeded at Flynn gate-3 on boot.

`kfc_template` changes from v1.3.x:

- `seed` field added — optional. Tool-first context descriptor: `{"tool": "...", "params": {...}}`. When defined in a KFC, the Ninja Summarizer calls this tool directly instead of running HyperIndex.
- `area_seed` field added — optional. Location-first variant of `seed`. `{{area_id}}` slot in params is filled at runtime by the Ninja Summarizer's `area_id` input. Used for per-area rollup patterns with Room Manager.
- `schema_version` bumped to `"1.4.0"`.

Flynn redeploys the template on next warmup when the version guard detects a stale `kfc_template` drawer. Existing KFCs are unaffected — both fields are optional and absent by default.

### Custom Prompt Material

If you want to ship completely custom Purpose, Directives, or Cortex content, copy the prompt loader script, make your changes, and fire it. The loader is a standard HA script — there's nothing special about it beyond the version-select logic. Custom forks are your own maintenance surface; ZenOS ships the versioned canonical set and that's the extent of it.

---

## zen_admintools_summarizer_seed / zen_admintools_summarizer_act

> **Removed in v5.1.0.** Both standalone scripts have been deleted. Whitelist management is now consolidated into `zen_admintools_prompt_loader mode=whitelist`. Use `whitelist_type=seed` for the seed whitelist and `whitelist_type=act` for the act whitelist.

---

## zen_admintools_run_repair

Human-confirmed passthrough to versioned one-time maintenance scripts. Each repair action targets a specific upgrade path and is designed to run once on the relevant install class. All actions require `confirm_action: true` — the script refuses without it.

**Not MCP-exposed.** Operator use only. Do not expose to unattended LLM pipelines.

### Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `repair_action` | select | — | Which repair to run. See table below. |
| `entity_id` | text | — | Cabinet entity ID. Required for `stamp_cab_guid_4_5_6` only. |
| `confirm_action` | boolean | `false` | Must be `true` — refuses without it. |
| `caller_token` | text | — | Opaque correlation token (echoed in response). |

### Repair Actions

| Action | What It Does |
|---|---|
| `identity_family_repair_4_5_6` | Wire the default family cabinet into the household graph. Targets pre-4.5.6 installs missing the family link. |
| `stamp_cab_guid_4_5_6` | Stamp a GUID onto a single cabinet. Requires `entity_id`. Targets installs where a cabinet was created without a GUID. |
| `roster_guid_repair_4_5_6` | Stamp GUIDs and re-wire all default cabinet roster entries. Targets pre-provisioner installs only — destructive on live provisioner installs. |

### When to Use

Run only when directed by an upgrade path document or a Nyx UAT report. These scripts are not idempotent in the general case — they were designed for specific transition states. If uncertain, run `identity_family_repair_4_5_6` with `dry_run` first if the script supports it; otherwise inspect the target cabinet with `cabinetadmin mode: inspect` before committing.

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `script.zen_dojotools_filecabinet` | Drawer reads and writes |
| Zen Dojo Cabinet | KFC component registry |
| Zen Kata Cabinet | Summary template storage |
| `sensor.zen_*_cabinet` (Ring-0 set) | cabinetadmin targets |

---

## Version History

| Version | Change |
|---------|--------|
| v5.3.1 | prompt_loader: new `sim_mode_allowed` field (default `false`) — stamped into `integrations_config.identity.sim_mode_allowed` on every factory run. OS-level SP1 policy switch (see `dojotools_identity.yaml` `resolve_caller_identity`). Fail-closed default, explicit on every install. `taskmaster` no longer inline-shipped by `ship_zen_system` — it self-registers via KF5. |
| v5.3.0 | cabinetadmin: `expand_drawer` (atomic drawer migration with rollback), `repair_volumeinfo` self-heals missing-header case. prompt_loader: `fc`-type whitelist add rejects bare `*`/empty-suffix `tool:` entries; field renamed `allowed_action_types` → `allowed_tools` (KF5 mode-scoped whitelist). |
| v5.2.0 | Cortex v43 "Rule Zero" added as latest. DojoTools supersede all HA built-ins — domain routing table in directives. v42 'The Answer' retained as prior slot. |
| v5.1.0 | Cortex v42 "The Answer" (v10.0.0) added as latest. Trimmed to 3 version slots: v42/v40/v38. `mode=whitelist` added to prompt_loader — absorbs `zen_admintools_summarizer_act` and `zen_admintools_summarizer_seed` (both deleted). Dispatcher compat shim routes legacy `summarizer_act` calls to `prompt_loader mode=whitelist type=act`. |
| v4.6.1 | KFC schema v1.4.0: `seed` and `area_seed` fields. `reset_template` now seeds `zen_summarizer_seed_whitelist` into syscab. |
| v4.6.0 | Cortex v39 (Home First). Dispatcher spamaster route. |
