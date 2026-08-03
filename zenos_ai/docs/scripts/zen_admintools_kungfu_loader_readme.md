# Zen AdminTools KungFu Loader — v5.2.0

*Deploys KFC (Kung Fu Component) dojo drawers via Scribe, for components that don't self-register*

---

## Overview

**Admin-only — not MCP-exposed.** Run via HA Developer Tools → Services or chained from `zen_admintools_prompt_loader` on a factory run.

Extracted from `zen_admintools_prompt_loader` to keep KFC deployment separate from prompt/cortex operations. Its job is narrowing over time: as tools adopt [KF5 self-registration](zen_kf5_pattern_readme.md), they stop needing this loader entirely and ship themselves via their own `mode=kfc_manifest`, discovered automatically by `zen_dojotools_manifest mode=bootstrap_kfc`. This loader now only carries the KFCs that have **no single owning script** to self-register from, plus one deploy-only component that hasn't migrated yet.

---

## What's Still Here vs. What Moved to KF5

| KFC | Status | Notes |
|---|---|---|
| `zen_system` | **Still shipped from here** (factory) | Meta-monitoring component — watches pipeline health, cabinet resolvers, kill switches, summarizer staleness across the whole system. No single owning script to self-register from. |
| `trapper_keeper` | **Still shipped from here** (factory) | Ambient index aggregator, same reason — cross-cutting, no owning script. |
| `camera_heartbeat` | **Still shipped from here** (deploy-only) | Scheduled camera-cache refresh sibling to `camera_manager`, not yet migrated. |
| `taskmaster` | Moved to KF5 (2026-07-25) | Self-registers via `zen_dojotools_taskmaster mode=kfc_manifest`. |
| `alert_manager` | Moved to KF5 (2026.7.1) | Self-registers via `zen_dojotools_alertmanager mode=kfc_manifest`. |
| `camera_manager` | Moved to KF5 (2026.7.1) | Self-registers via `zen_dojotools_camera mode=kfc_manifest`. |
| `security_manager` | Moved to KF5 (2026.7.1) | Self-registers via `zen_dojotools_security_manager mode=kfc_manifest`. |

The last three were still carrying dead inline `ship_*` blocks in this loader for a while after their KF5 migration landed — pure duplication, since KF5 discovery had already taken over shipping them. Snipped in v5.2.0. **`mode=status` below is unaffected by any of this** — it reads each KFC's actual drawer content directly, regardless of which mechanism shipped it, so status reporting for `alert_manager`/`camera_manager`/`security_manager` still works exactly as before.

---

## Modes

| Mode | Description |
|------|-------------|
| `status` (default) | No-op. Returns deployment status (version, enabled, last-updated, version-match) for all managed KFCs, including the three that migrated to KF5 — reads their drawers directly. |
| `factory` | Deploys `zen_system` and `trapper_keeper` (the two remaining factory-flagged KFCs). Intended to be chained from `zen_admintools_prompt_loader` on a factory run. |
| `deploy` | Deploys KFCs selected by individual boolean fields (`ship_zen_system`, `ship_trapper_keeper`, `ship_camera_heartbeat`) — all default `false`. Use for selective re-ship from HA Developer Tools. |

---

## Fields

| Field | Description |
|-------|-------------|
| `mode` | `status` / `factory` / `deploy`. Default `status`. |
| `ship_zen_system` | Deploy `zen_system` KFC. `factory: true` — auto-ships in factory mode too. |
| `ship_trapper_keeper` | Deploy `trapper_keeper` KFC. `factory: true`. |
| `ship_camera_heartbeat` | Deploy `camera_heartbeat` KFC. Deploy-only, never auto-ships in factory mode. |
| `readback` | If `true`, includes the first 200 chars of each deployed KFC's landed payload for verification. Useful after a factory run to confirm content actually landed. |
| `caller_token` | Opaque pass-through token for caller correlation. Not interpreted by this script. |

---

## Status Report Shape

`mode=status` returns one entry per managed KFC (all seven — the three self-registering ones included) under `kfcs`, each with:

```yaml
kfcs:
  <kfc_name>:
    deployed: true/false       # false if the drawer has never been written
    enabled: true/false        # from the drawer's own meta.enabled field
    version: "1.5.0"           # deployed version, or "not_deployed"
    target: "1.5.0"            # the version this loader/tool expects
    version_match: true/false  # deployed == target
    last_summary_at: "..."     # kata cabinet's updated_at, or "never"
    factory: true/false        # whether this KFC auto-ships in factory mode
```

---

## Related

- [KF5 Self-Registration Pattern](zen_kf5_pattern_readme.md) — how a tool migrates off this loader.
- [AdminTools](zen_dojotools_admintools_readme.md) — `prompt_loader`'s `ship_zen_system` field chains this loader in factory mode.
- [Building a KFC](../kung_fu/building_a_kfc.md)
