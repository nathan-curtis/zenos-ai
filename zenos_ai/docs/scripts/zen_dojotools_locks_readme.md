# Zen DojoTools Locks — 1.4.1

*Room-targetable lock/unlock control for the `lock.*` domain, with a full Keymaster-aware inspect surface*

---

## Overview

`zen_dojotools_locks` closes a gap `zen_dojotools_covers` already closed for `cover.*` — a dedicated, room-aware setter/getter for locks. It's also the reference implementation for `zen_target_resolve.jinja`, the first shared targeting core adopted by a DojoTools setter/getter (Zammad #10308), and for the platform's identity-gate pattern on actuation (stolen and generalized from `zen_dojotools_infra`'s Portainer container-control codex).

Three addressing modes for every read/write mode: `entity_id` (explicit override, beats everything else), `label` (Room Manager label), and `room` (area name/slug or RM room label — both resolve). Room resolution is RM-aware: it unions native HA area assignment with Room Manager's own label-based room tagging, not area assignment alone, so a lock labeled into a room without a native area assignment still resolves.

---

## Modes

| Mode | Description | Writes? |
|---|---|---|
| `discover` | List locks (room/label/entity_id scoped, or whole-house if all omitted) with live state and any extended notes their labels carry | No |
| `status` | Same as `discover` | No |
| `set` | Lock or unlock resolved targets. Gated — see Identity Gate below | Yes |
| `inspect` | Single-lock deep detail — all labels, area context, `ext_lock` aggregate membership, battery/device info, real Keymaster code-slot summary if present | No |
| `stacks_by_anchor` | Lens Bus provider, `anchor_type=area_id` — "what locks does this room have and what state are they in," without the caller needing to know this tool exists | No |
| `register` / `unregister` | Lens Bus registry entry (household cabinet `lens_registry` drawer), same shape as `zen_dojotools_autovac`'s existing lens provider pattern | Yes |
| `kfc_manifest` | KFC component definition — lets summarizer/agent context seed real lock state instead of falling back to raw `label:lock` entity scanning | No |
| `escalation_request` | Request a time-boxed escalation grant (e.g. `code_reveal`) via a live household-admin ack, written to the household cabinet with an expiry | Yes |
| `escalation_status` | Check whether a requested escalation scope is currently granted and unexpired | No |
| `help` | Full mode/field reference | No |
| `tool_manifest` | Self-description via `MF.tool_manifest()` | No |

Bare `discover`/`status` with no filters returns a whole-house lock overview — a real gap fix; earlier builds silently ignored `entity_id=` outside `mode=set`.

---

## Identity Gate on `mode=set`

Read modes (`discover`/`status`/`inspect`) stay open. Actuation (`set`) requires the default agent's own `lock_control` certification, checked via `zen_dojotools_identity mode=resolve_caller_identity required_cert=lock_control required_cert_level=1`. **Fail-closed by design, not by bug** — until `lock_control` is granted via `zen_dojotools_persona_editor mode=cert_grant` (itself gated, see that tool's readme), every `mode=set` call is denied with `error: identity_policy_blocked` or `error: cert_insufficient`. `caller_token` is audit-only here, same as everywhere else in the platform — never a trust input.

This tool self-declares `lock_control` in its own `tool_manifest`'s `certs_required` field — that's what makes it a valid `cert_grant` target at all under the live-calculated catalog (see `zen_dojotools_profile_readme.md`'s certification section; there's no separate hand-maintained cert list to also update).

**Exterior unlock requires a fresh live ack every single call, even with `lock_control` already held (2026-08-15).** Holding the cert is necessary but not sufficient for `action=unlock` on an `ext_lock`-labeled target — every such call additionally fires `zen_dojotools_identity mode=request_live_ack`, no exceptions, because a standing cert was never meant to mean "unattended exterior access forever." An admin can exempt specific targets from this per-call ack via `cert_grant cert_component=lock_control cert_scope=["lock.front_door"]` — the scope list is checked by entity_id, not by label, so scope it to exactly the lock(s) intended. Interior-only actions and read modes are unaffected either way.

**Audit-trail emission on real actuations (2026-08-15):** every successful `mode=set` fires `zen_dojotools_event_emitter`, and two purpose-neutral trigger labels — `lock_unlocked_armed` / `lock_unlocked_night` — are wired so other systems (e.g. REFLEX) can react to an unlock happening while armed or overnight without polling lock state themselves.

---

## Escalation (Time-Boxed Grant)

`mode=escalation_request` is for capabilities that need a *standing* grant rather than one action's approval (e.g. revealing a Keymaster code for a limited window) — distinct from the one-shot `mode=set` cert gate above. Delegates ack-wait timing to `zen_dojotools_identity mode=request_live_ack` (shared chokepoint, hard-capped at 150s regardless of what's requested here). On approval, writes `{scope, granted_at, expires_at, reason}` to the household cabinet's `lock_escalations` drawer, keyed by scope. `mode=escalation_status` reads it back and reports whether the grant is still within its expiry window.

```yaml
zen_dojotools_locks:
  mode: escalation_request
  escalation_scope: code_reveal
  # escalation_reason: "..."       # optional, defaults to a generic message
  # escalation_minutes: 15          # optional, default 15
```

---

## `mode=inspect` — Keymaster Integration

For a Z-Wave lock with Keymaster managing code slots, `mode=inspect` surfaces a real per-slot summary — **PINs are never surfaced, `pin_set` is boolean-only** (derived from whether the slot's state is available, not from reading the PIN itself). Also reports parent/child Keymaster sync topology where applicable (`keymaster.is_parent` / `parent_lock` / `child_locks`).

The underlying mechanism took three wrong turns before landing right, worth knowing if you're extending this: `zwave_js.get_config_parameters` doesn't work for this (always empty, no error either way — Z-Wave config parameters are exposed as their own device entities, not a bulk service response), and Keymaster's per-lock entities can't be found via area scan or `integration_entities('keymaster')` (Keymaster's `script.*` entities aren't area-assigned, and aren't registered under a `keymaster` integration domain regardless of naming). The real mechanism: Keymaster runs a **separate HA device per lock**, found by grabbing any one `binary_sensor.*_code_slot_*` entity matching the lock's squashed object_id, resolving `device_id()` from there, then walking `device_entities()` on that device for every slot's fields.

---

## Dependencies

- `custom_templates/zenos_ai/zen_target_resolve.jinja` — shared targeting core (`resolve_targets`, `resolve_room_entities`, `entity_extended_notes`, `discover_targets`)
- `custom_templates/zenos_ai/zen_os_1.jinja` — `envelope()` macro (response shape)
- `zen_dojotools_identity` — `resolve_caller_identity` (set gate), `request_live_ack` (escalation + every exterior unlock)
- `zen_dojotools_persona_editor` — `lock_control` certification grant/revoke
- `zen_dojotools_manifest` — `mode=cert_audit` reads this tool's self-declared `certs_required`
- `zen_dojotools_event_emitter` — audit trail on real actuations
- `zen_dojotools_autovac` — lens provider registration precedent

## Cross-References

- [DojoTools Identity](zen_dojotools_identity_readme.md) — `resolve_caller_identity`, `request_live_ack`
- [DojoTools Profile Editor](zen_dojotools_profile_readme.md) — `cert_grant`/`cert_revoke` for `lock_control`
- [Script Modules](readme.md) — return path to the internal tool map

---

## Version History

| Version | Change |
|---------|--------|
| v1.4.1 (2026-08-15) | Exterior unlock requires a fresh live ack every call, even with `lock_control` already held — `cert_scope` admin-override lets specific targets skip this. Audit-trail emission on real actuations (`zen_dojotools_event_emitter`, `lock_unlocked_armed`/`lock_unlocked_night` trigger labels). Self-publishes `lock_control` in `tool_manifest.certs_required` for the live cert catalog (`zen_dojotools_manifest mode=cert_audit`), replacing what was briefly a hand-maintained catalog file. |
| v1.4.0 | Identity gate on `mode=set` (stolen from the Portainer container-control codex). Time-boxed escalation infra (`escalation_request`/`escalation_status`), cabinet-tunable ack-wait timing via `zen_dojotools_identity mode=request_live_ack`. Real parent/child Keymaster sync topology surfaced in `mode=inspect`. |
| v1.3.0 | "Cadillac" pass (Zammad #10309): `mode=inspect` deep detail, `mode=stacks_by_anchor` (Lens Bus provider), `mode=register`/`unregister`, `mode=kfc_manifest`. `discover`/`status` now accept `entity_id=` (previously silently ignored outside `mode=set`). Bare `discover`/`status` with no filters returns whole-house overview instead of erroring. |
| v1.2.0 | Adopted the canonical `envelope()` shape and single-entry/single-exit structure (#10304 SESE audit target). |
| v1.1.0 | First tool built on the shared `zen_target_resolve.jinja` core (#10308). All three addressing modes, RM-aware room resolution, `entity_extended_notes` surfaced on `discover`. |
