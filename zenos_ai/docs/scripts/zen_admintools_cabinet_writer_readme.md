# Zen AdminTools Cabinet Writer — v1.8.0

**File:** `packages/zenos_ai/dojotools/dojotools_cabinet_writer.yaml`
**Script:** `zen_admintools_cabinet_writer`
**Tier:** admintools. Not MCP-exposed.

## Overview

The admin door. Agents write cabinets through `zen_dojotools_filecabinet`. This script is the other door, for structural writes FileCabinet refuses by design: `AI_Cabinet_VolumeInfo`, `meta`, `_label_index`, `_zen_relationships`, the system cabinet and the vault.

Callers are admin-plane scripts and tools that own a structural write: Identity membership, the prompt loader, cabinet admin, Flynn and maintenance scripts. Every write names its `caller` and is logged, so admin writes can be audited now and gated later. The caller name is a record for audit, not authority.

## Modes

| Mode | Fields | Notes |
|---|---|---|
| `get` | `volume_entity`, `key`, `caller` | Reads one drawer. The vault's read rule is the admin door, so this is the one way to read it; Identity reads the cert catalog cache here. Logged at debug. The write gate does not apply to reads. |
| `set` | `volume_entity`, `key`, `value`, `caller`, optional `reason`, `force`, `bootstrap`, `meta`, `acl` | Writes a drawer. The value is written as given (mapping, list or string); `meta` and `acl` pass through to the drawer envelope. |
| `merge` | `volume_entity`, `key`, `value` (a mapping), `caller`, optional `unset`, `reason`, `force`, `bootstrap` | Reads the stored value (a mapping, or a JSON string of one) when it writes, lays the given keys over it and drops the top-level keys named in `unset`. A caller sends only what it changes, so a partial value can never replace the whole drawer. Queued, so concurrent merges serialize. Identity's VolumeInfo writes use it. |
| `remove` | `volume_entity`, `key`, `caller`, optional `reason` | Removes a drawer. |
| `clear` | `volume_entity`, `caller`, `confirm=true`, optional `reason` | Wipes every drawer. |
| `tool_manifest` | | Self-description. |
| `help` | | Usage. The default for a missing or unknown mode, and never a write. |

`volume_entity` must be an online cabinet carrying the `zen_cabinet` label; a `bootstrap` write may also target a cabinet still in `unknown`. `force` is passed to the Volume Redirector for writes that own it, and `bootstrap` stamps a cabinet that is not yet in the manifest (cabinet admin init and restore).

## Verified writes

Each write waits until the cabinet shows it before the door answers: the drawer timestamp for `set` and `merge`, the key gone for `remove`, the drawers gone for `clear`. The response carries `write_verified`. A write that is not visible within 10 seconds answers `status: error`. Callers read-modify-write VolumeInfo, so an early answer would let the next call overwrite this one.

A `set` or `merge` also checks that the stored value equals what it meant to write, and answers `content_verified`. A changed value is an error.

## Gate (audit only)

An agent or console write chain needs cert `zenos.system.cabinet_admin` and would ask a live human. Its `max_level` is 2: level 1 is this door, and level 2 also lets a user session change cabinet labels (Labels 5.10.0). A chain with no user session (boot, Flynn's bootstrap, automations) is exempt. Nothing is refused yet: the decision is logged (`zen_event door_gate_audit` and a `system_log` line) and answered as `gate_audit`: `{mode: audit, exempt, caller_guid, cert, would_deny, reasons, would_ack}`.

## Logging

Every write fires a `zen_event` of kind `admin_cabinet_write` and writes a `system_log` line with the caller, operation, cabinet, key and reason.

## Response

`{status, tool, mode, volume_entity, key, caller, write_verified, content_verified, gate_audit, message, caller_token}`, `content_verified` is always `true` for `remove` and `clear`, and `gate_audit` is present on agent and console chains. A refused call returns `status: error` with a `guarded:` message (a missing caller, an offline or unlabeled cabinet, a missing key or value, or `clear` without `confirm=true`).

## Changelog

| Version | Change |
|---|---|
| v1.8.0 | `get` mode: the one read path into the vault. |
| v1.7.0 | `zenos.system.cabinet_admin` `max_level` 2; level 2 covers cabinet labels. |
| v1.6.1 | Plain cert description for the approval prompt. |
| v1.6.0 | Door gate, audit only: agent and console writes are checked against `zenos.system.cabinet_admin` and answered with `gate_audit`. |
| v1.5.0 | `merge` mode: only the changed keys, laid over the stored value at write time. |
| v1.4.0 | `meta` and `acl` pass through to the drawer envelope. |
| v1.3.0 | A `set` checks the stored value equals the value sent (`content_verified`). |
| v1.2.0 | A `bootstrap` write may target a cabinet still in `unknown`. |
| v1.1.0 | Every write waits for the cabinet to show it and answers `write_verified`. Adds `force` and `bootstrap` passthroughs for cabinet admin. |
| v1.0.0 | The admin door: `set`, `remove` and `clear` for admin-plane cabinet writes. |
