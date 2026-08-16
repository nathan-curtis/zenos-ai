# Zen DojoTools Manifest — v6.3.1 (ZenOS-AI 2026.9.0)
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
| `publish` | Writes three mini-manifests to the household cabinet: `zen_tool_manifest`, `zen_label_manifest`, `zen_automation_manifest`. Also writes the domain routing table — built dynamically from `label_entities('zen_domain_*')`, no hardcoded domain:entity_id list. |
| `mcp_sync` | Reconciles MCP-visible tools against the known tool roster. Requires `mcp_tool_list` (JSON array of entity_ids the agent can see). |
| `bootstrap_stacks` | Auto-registers Lens Bus stack providers. Scans all `zen_stack_*` and `zen_sutra_*` scripts **plus** `script.zen_dojotools_library` (which has no `zen_stack_*` wrapper but is a Lens Bus owner). Calls `tool_manifest` on each, registers any that declare `register_mode` and are not already in `lens_registry`. Idempotent — safe to run repeatedly. See [Lens Bus Auto-Registration](../plugins/lens_bus_autoreg.md). |
| `bootstrap_kfc` | Auto-registers KFC self-declaring tools. Scans tools in `_bkfc_known`, calls `kfc_manifest` on each, writes live drawer mounts into the dojo cabinet. Fires on `homeassistant_start` + daily `00:01`. See [Building a KFC — KF5](../kung_fu/building_a_kfc.md#kf5-self-registering-tools). |
| `health_refresh` | Runs a full per-tool compliance scan and caches the result to the `_health_report` cabinet drawer. `mode=health` reads this cache first, falling back to a live check only if the cache is absent. |
| `domains` | Live tool/domain/peers graph — which tool owns which domain and what else shares that domain. |
| `audit_help` | Scans `mode=help` across every discovered `zen_*` tool — surfaces tools with a missing or malformed help surface. |
| `label_audit` | Read-only gap scan: calls `tool_manifest` on every discovered `zen_*` tool, aggregates `missing_required_labels`/`missing_optional_labels` (`include_optional` default `true`). Reports gaps, creates nothing. |
| `cert_audit` (2026-08-15) | Same fan-out shape as `label_audit`, for KFC certifications instead of labels — calls `tool_manifest` on every discovered tool and aggregates whatever each one self-declares in its own `certs_required` field into `{cert_component: {tool, display_name, ...}}`. This is the live replacement for what was briefly a hand-maintained `.persona_certs/cert_catalog.json` file — `zen_dojotools_persona_editor`'s `cert_grant`/`cert_revoke` read this to validate a cert name isn't a flat typo before the real gate (a live household-admin ack) fires. A tool declaring `certs_required` doesn't need a separate registration step anywhere else — this mode is the single source of truth, calculated fresh on every call, nothing to keep in sync. |
| `repair` | Confirm-gated remediation — wraps `label_audit`'s scan, creates missing labels and syncs KFC-bound label description/icon/color. See [`mode=repair`](#moderepair--label-remediation) below. |
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
| `confirm` | `repair` | Must be `true` to actually create/sync labels. Omit or `false` for a dry-run preview (`would_*` fields). |
| `include_optional` | `label_audit`, `repair` | Default `true` — include optional (not just required) label gaps. `false` scans/repairs required labels only. |
| `force` | `repair` | Rewrite every existing KFC label's description/icon/color this run. Requires `force_ack`. |
| `force_labels` | `repair` | Rewrite only the named KFC label(s). Requires `force_ack`. |
| `force_ack` | `repair` | Must be the exact literal string `yes I meant to do that` for `force`/`force_labels` to take effect — anything else silently downgrades to the safe skip-existing default. |
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

## `mode=repair` — Label Remediation

Deliberately named `repair`, not `repair_labels` — labels are the only fixable [`zen_health_report` `known_issues`](zen_dojotools_systemtools_readme.md) category today, not the only one planned. As more categories become automatable, they dispatch from this same mode by category — callers and Spook's own install-hint text keep pointing at `mode=repair` without ever needing to change.

Today: a confirm-gated wrapper around `label_audit`'s scan+create logic, plus KFC-bound label description/icon/color sync. This is the entry point Spook's `unknown_label` repair issues point callers at — Spook self-clears those specific issues on its own next scan once the underlying label exists; this mode never touches the Issue Registry directly.

```yaml
zen_dojotools_manifest:
  mode: repair
  confirm: true          # omit or false = dry-run preview only
  include_optional: true # default true — false scans required-labels only
```

### Missing-label creation

Scans every discovered `zen_*` tool's `missing_required_labels`/`missing_optional_labels` (from `tool_manifest`'s always-computed fields — see [`zenos_manifest.jinja`](../custom_templates/zenos_manifest_jinja.md)) and creates whatever's missing via `zen_dojotools_labels`. New labels that happen to be KFC-bound (see below) get their description/icon/color baked into the same atomic create call — never created blank then patched.

### KFC label description/icon/color sync

A label is "load-bearing" for a KFC component when that component's own `kfc_manifest` entry declares `label: <name>`. That component's `component_summary` is the authoritative description for the label.

**Existing** KFC labels are **skipped by default** — no content-diff check, just "already exists → don't touch it." This is deliberate: HA has no native `update_label` service (only `create_label`/`rename_label`/`remove_label`), so "update" is actually implemented as **delete → create → re-tag every entity that carried it**. Entity associations are restored from a pre-capture, but **AREA and DEVICE associations are not** (`label_entities()` only sees entities) — permanently gone if the delete/recreate cycle runs on a label anything besides an entity was ever tagged with. That risk isn't worth paying on every routine repair run for a label that hasn't actually changed.

Two explicit overrides, both requiring `force_ack` to be the **exact literal string** `yes I meant to do that` — anything else silently downgrades the request to the safe skip-existing default, surfaced as `force_blocked: true` in the response, never a silent no-op:

| Field | Effect |
|-------|--------|
| `force=true` | Shotgun — rewrites every existing KFC label this run. |
| `force_labels=["autovac", ...]` | Scalpel — rewrites only the named label(s), everything else still skipped. |

A successful force-rewrite returns a `warning` field spelling out exactly what needs manual reapplication (any area/device association on that label).

### Response fields

| Field | Description |
|-------|--------------|
| `status` | `repaired` \| `issues_found` \| `clean` (dry-run: `issues_found` \| `clean`) |
| `tools_with_gaps` / `tools_with_optional_gaps` | Per-tool missing-label detail |
| `created_labels` | All labels created this run (plain + KFC) |
| `kfc_labels_created_with_description` | New KFC labels, created with description/icon/color in one call |
| `kfc_label_descriptions_synced` | Existing KFC labels actually rewritten this run (force path only) |
| `kfc_labels_skipped_existing` | Existing KFC labels left alone |
| `would_create` / `would_sync_kfc_label_descriptions` / `would_skip_existing_kfc_labels` | Dry-run equivalents |
| `force` / `force_labels` / `force_blocked` | Echo of what was requested and whether it was honored |
| `warning` | Populated on a force-rewrite or a blocked force request — see above |
| `errors` | Per-tool `tool_manifest` call failures, if any |

### Automatic firing

`zen_manifest_repair_labels` automation runs `mode=repair confirm=true` on HA startup and daily at `00:00:15` — staged to fire before `bootstrap_stacks`/`bootstrap_kfc` (`00:00:30`/`00:01:00`), since those depend on tools already carrying their declared labels. `confirm=true` is safe here specifically because label creation is scoped, additive-only, and non-destructive — unlike `reload_all`/`restart`, this does not require live operator approval. Idempotent — a clean system is a fast no-op (one `tool_manifest` scan per tool, no writes).

---

## `mode=tool_manifest` — Self-Description

When called with `mode=tool_manifest`, the broker returns its own structured self-description and exits immediately without running any subsystem. This is the UMP (Unified Manifest Protocol) contract: every tool that supports `tool_manifest` describes itself in a standard shape.

The self-description is produced by `MF.tool_manifest()` from `zenos_ai/zenos_manifest.jinja`:

```yaml
tool: zen_dojotools_manifest
display_name: System Manifest Broker
tier: dojotools
version: 6.2.0
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
