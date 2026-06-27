# Zen DojoTools Manifest — v6.1.0 (ZenOS-AI 2026.7.0 'Neo')
**File:** `zen_dojotools_manifest_readme.md`
**Type:** Technical Documentation

---

## Overview

The **Zen DojoTools Manifest** is the ZenOS-AI system manifest broker. As of v6.0.0 it is no longer a cabinet-only scanner — it is the single introspection lens for the entire system: cabinets, tools, identity, labels, automations, and topology.

The broker is **pure read-and-aggregate**. It calls authoritative subsystem scripts and assembles their results. It does not make repairs, rewrite data, or mutate any target system. Mode `cabinets` retains the zero-persistence, read-only cabinet scanner behavior from v5.x. All other modes follow the same discipline.

The crew — Friday, Veronica, Kronk, the High Priestess — rely on this broker as their ground truth for the system's internal geography.

---

## Discovery Model

v6.0.0 replaced the old label-registry approach with **entity namespace scanning**.

Tools are discovered by scanning Home Assistant entity namespaces:

| Namespace prefix | Tier |
|---|---|
| `script.zen_dojotools_*` | dojotools |
| `script.zen_admintools_*` | admintools |
| `script.zen_stack_*` | stacks |
| `script.zen_sutra_*` | sutra |

The domain+prefix combination is canonical. There is no static list maintained anywhere. Any script that appears in these namespaces and is in a callable state (`on` or `off`) is treated as a live tool. Scripts in any other state are tracked as unavailable.

The `tier` field on inputs to modes like `tools`, `audit`, and `autotag` can filter results to a single namespace.

---

## Modes

The `mode` field routes the broker to the appropriate subsystem. Default is `cabinets`.

| Mode | What it does |
|---|---|
| `cabinets` | Cabinet volume scanner — full structural and health snapshot of every AI_Cabinet volume. Default behavior, preserved verbatim from v5.x. |
| `identity` | Household, agent, and persona roster. Calls `zen_dojotools_identity` with `build_identity_manifest`. |
| `labels` | Full label index. Calls `zen_dojotools_labels`. |
| `tools` | Discovers all `zen_*` scripts by namespace scan. Optional `name` field for single-tool lookup; optional `tier` field to filter by namespace. |
| `automations` | Roster of all `automation.zen_*` entities including state and last-triggered timestamps. |
| `structure` | Lens registry topology. |
| `audit` | Gap analysis: unlabeled, broken, and ghost tools. Optionally filtered by `tier`. |
| `health` | Aggregate health roll-up across subsystems. |
| `autotag` | Tag discovered tools. Optionally filtered by `tier`. |
| `publish` | Writes three mini-manifests to the household cabinet: `zen_tool_manifest`, `zen_label_manifest`, `zen_automation_manifest`. Also writes the domain routing table. |
| `mcp_sync` | Reconciles MCP-visible tools against the known tool roster. Requires `mcp_tool_list` (JSON array of entity_ids the agent can see). |
| `bootstrap_stacks` | Auto-registers Lens Bus stack providers. Scans all `zen_stack_*` and `zen_sutra_*` scripts **plus** `script.zen_dojotools_library` (which has no `zen_stack_*` wrapper but is a Lens Bus owner). Calls `tool_manifest` on each, registers any that declare `register_mode` and are not already in `lens_registry`. Idempotent — safe to run repeatedly. See [Lens Bus Auto-Registration](../plugins/lens_bus_autoreg.md). |
| `all` | Full system manifest. Calls subsystems directly and aggregates. If `force_refresh: true`, runs `publish` first to refresh cached drawers. |
| `tool_manifest` | Self-description. See below. |

### Mode fields

| Field | Relevant modes | Description |
|---|---|---|
| `mode` | all | Broker operation. Default: `cabinets`. |
| `show_hidden` | `cabinets` | Include hidden/system volumes. |
| `show_stacks` | `cabinets` | Include `online_unmounted` (stacks) cabinets. Default false. |
| `extended` | `cabinets`, `all` | Return full metadata (cabinets) or extended manifests (all). |
| `name` | `tools` | Filter to a single tool by entity_id. |
| `tier` | `tools`, `audit`, `autotag` | Filter by namespace tier: `dojotools`, `admintools`, `stacks`, `sutra`. |
| `force_refresh` | `all` | Bypass cached drawer reads, force live subsystem calls. |
| `mcp_tool_list` | `mcp_sync` | JSON array of entity_ids visible to the calling agent via MCP. |
| `caller_token` | all | Governance stub — echoed in response. |

---

## Domain Routing Table

`mode=publish` writes a domain routing table to the household cabinet under `zen_tool_manifest`. This table is the system's authoritative answer to "which tool owns which domain."

It is defined statically in the broker and written out on every `publish` run. Agents that need to route a user request to the right tool should read this table rather than guessing from tool names.

| Domain | Script |
|---|---|
| `rooms` | `script.zen_dojotools_room_manager` |
| `food` | `script.zen_dojotools_kitchen` |
| `knowledge` | `script.zen_dojotools_filecabinet` |
| `calendar` | `script.zen_dojotools_calendar` |
| `identity` | `script.zen_dojotools_identity` |
| `lights` | `script.zen_dojotools_lights` |
| `climate` | `script.zen_dojotools_climate` |
| `covers` | `script.zen_dojotools_covers` |
| `todos` | `script.zen_dojotools_todo` |
| `announce` | `script.zen_dojotools_announce` |
| `security` | `script.zen_dojotools_security_manager` |
| `media` | `script.zen_dojotools_media_manager` |
| `contacts` | `script.zen_dojotools_rolodex` |
| `tickets` | `script.zen_dojotools_servicedesk` |
| `inventory` | `script.zen_dojotools_inventory` |
| `wiki` | `script.zen_dojotools_filecabinet` |
| `finance` | `script.zen_dojotools_finance` |
| `system` | `script.zen_dojotools_manifest` |

Design note: room-oriented queries prefer `room_manager` over raw entity state queries.

---

## `mode=tool_manifest` — Self-Description

When called with `mode=tool_manifest`, the broker returns its own structured self-description and exits immediately without running any subsystem. This is the UMP (Unified Manifest Protocol) contract: every tool that supports `tool_manifest` describes itself in a standard shape.

The self-description is produced by `MF.tool_manifest()` from `zenos_ai/zenos_manifest.jinja`:

```yaml
tool: zen_dojotools_manifest
display_name: System Manifest Broker
tier: dojotools
version: 6.0.0
health:
  configured: true
  status: ok
  notes: []
required_labels: [dojotools]
optional_labels: [manifest, audit]
consumes: []
returns: [system_manifest]
icon: mdi:sitemap
color: blue
```

Response variable: `result` (containing `system_manifest`).

---

## Cabinet Scanning (`mode=cabinets`)

This is the v5.x behavior, preserved verbatim as one mode among many.

The cabinet scanner enumerates all Home Assistant sensor entities, detects those containing `AI_Cabinet_VolumeInfo`, and produces a complete structural and health snapshot. It performs **zero persistent health storage** — all health is computed fresh on every run. The only write it performs is emitting the compiled manifest JSON to `zen_library_manifest` in the Family Cabinet.

No repairs. No rewrites. No silent mutations. The cabinet scanner is strictly an MRI — never the surgeon.

### What it scans

**Volume metadata** — GUID, friendly name, schema version, context blocks, flags, timestamps (`last_changed`, `last_updated`).

**Drawers** — all drawers except system/hidden drawers, drawers starting with `_` or `.`, and `AI_Cabinet_VolumeInfo` (always hidden). Mount-point drawers are annotated with their target `entity_id` or `volume_id`.

**ACLs** — reads `acls` from volume metadata and normalizes across `entity_guid`, `family_guid`, and `household_guid` categories.

**Labels** — canonical label list with hidden/system label filtering. Override with `show_hidden: true`.

**Health** (runtime-only, never stored):
- `ready` — all conditions nominal
- `warning` — capacity threshold exceeded (default: 80%)
- `error` — GUID mismatch or unreadable volume

Write access is blocked on: read-only flag, schema version mismatch, storage warning. Read access is blocked only in severe cases.

**Capacity** — character-based, max 131,072 chars per volume, warning at 80%.

### Output shape (per volume)

```json
{
  "<entity_id>": {
    "entity_id": "sensor.example_cabinet",
    "friendly_name": "Example Cabinet",
    "context": {},
    "metadata": {
      "id": "<guid>",
      "schema_version": 1.0,
      "labels": [],
      "timestamps": {
        "last_changed": "ISO-8601",
        "last_updated": "ISO-8601"
      }
    },
    "stats": {
      "drawer_count": 0
    },
    "capacity": {
      "chars_used": 0,
      "chars_max": 131072,
      "percent_used": 0.0,
      "warning": false
    },
    "access": {
      "writeable": true,
      "readable": true
    },
    "drawers": [
      "drawerA",
      "drawerB [mount:sensor.target_volume]"
    ],
    "drawer_index": {
      "label1": ["drawerA"]
    },
    "acls_by_category": {
      "allow": [],
      "deny": []
    },
    "health": {
      "status": "ready",
      "guid_mismatch": false,
      "storage_warning": false,
      "writeable": true,
      "readable": true
    }
  }
}
```

---

## MCP Exposure

The manifest broker is exposed to agents via MCP as `zen_dojotools_manifest`. Agents should call `mode=tool_manifest` first to confirm the tool is live and read its self-description. Use `mode=all` for a full system snapshot; use `mode=publish` before `mode=all` with `force_refresh: true` if stale cached data is a concern.

The `mcp_sync` mode exists specifically to reconcile what an agent can see through MCP against what the broker knows is registered. Use it to detect gaps in MCP tool visibility.
