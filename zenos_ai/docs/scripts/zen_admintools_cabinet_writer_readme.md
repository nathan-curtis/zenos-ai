# Zen AdminTools Cabinet Writer — v1.1.0

**File:** `packages/zenos_ai/dojotools/dojotools_cabinet_writer.yaml`
**Script:** `zen_admintools_cabinet_writer`
**Tier:** admintools. Not MCP-exposed.

## Overview

The admin door. Agents write cabinets through `zen_dojotools_filecabinet`. This script is the other door, for structural writes FileCabinet refuses by design: `AI_Cabinet_VolumeInfo`, `meta`, `_label_index`, `_zen_relationships`, the system cabinet and the vault.

Callers are admin-plane scripts and tools that own a structural write: Identity membership, the prompt loader, cabinet admin, Flynn and maintenance scripts. Every write names its `caller` and is logged, so admin writes can be audited now and gated later. The caller name is a record for audit, not authority.

## Modes

| Mode | Fields | Notes |
|---|---|---|
| `set` | `volume_entity`, `key`, `value`, `caller`, optional `reason`, `force`, `bootstrap` | Writes a drawer. The value is written as given (mapping, list or string). |
| `remove` | `volume_entity`, `key`, `caller`, optional `reason` | Removes a drawer. |
| `clear` | `volume_entity`, `caller`, `confirm=true`, optional `reason` | Wipes every drawer. |
| `tool_manifest` | | Self-description. |
| `help` | | Usage. The default for a missing or unknown mode, and never a write. |

`volume_entity` must be an online cabinet carrying the `zen_cabinet` label. `force` is passed to the Volume Redirector for writes that own it, and `bootstrap` stamps a cabinet that is not yet in the manifest (cabinet admin init and restore).

## Verified writes

Each write waits until the cabinet shows it before the door answers: the drawer timestamp for `set`, the key gone for `remove`, the drawers gone for `clear`. The response carries `write_verified`. A write that is not visible within 10 seconds answers `status: error`. Callers read-modify-write VolumeInfo, so an early answer would let the next call overwrite this one.

## Logging

Every write fires a `zen_event` of kind `admin_cabinet_write` and writes a `system_log` line with the caller, operation, cabinet, key and reason.

## Response

`{status, tool, mode, volume_entity, key, caller, write_verified, message, caller_token}`. A refused call returns `status: error` with a `guarded:` message (a missing caller, an offline or unlabeled cabinet, a missing key or value, or `clear` without `confirm=true`).

## Changelog

| Version | Change |
|---|---|
| v1.1.0 | Every write waits for the cabinet to show it and answers `write_verified`. Adds `force` and `bootstrap` passthroughs for cabinet admin. |
| v1.0.0 | The admin door: `set`, `remove` and `clear` for admin-plane cabinet writes. |
