# Zen DojoTools Covers — 6.0.0 (codename: ZenShade)

*Room-aware cover, blind, shade, and smart-vent manager*

---

## Overview

`zen_dojotools_covers` manages the `cover.*` domain by label taxonomy (`zen_cv_blackout`/`zen_cv_sheer`/`zen_cv_curtain`/`zen_cv_vent`/`zen_cv_primary`) with scene presets (`movie`/`work`/`morning`/`sleep`) and direct `set`/`discover`. As of v6.0.0, this is the second tool (after `zen_dojotools_locks`) to adopt the shared `zen_target_resolve.jinja` targeting core and the platform's identity-gate pattern on actuation — it was the other hand-rolled targeting implementation the shared core was originally extracted from, so this migration was flagged as pending back when `zen_target_resolve.jinja` first shipped.

## Modes

| Mode | Description | Writes? |
|---|---|---|
| `discover` | List covers (room/label/entity_id scoped) with live state | No |
| `set` | Direct cover position/action on resolved targets. Gated — see Identity Gate below | Yes |
| `scene_set` | Apply a named scene (`movie`/`work`/`morning`/`sleep`) across a room's window covers, barrier-class covers always excluded | Yes |
| `setup` | Commissioning/label-coverage helper | No |
| `help` | Full mode/field reference | No |

`discover`/`set`/`scene_set` response shape is unchanged from v5.1.0 — only target resolution and actuation gating changed in v6.0.0; regression-verified against the prior field set.

## Barrier Auto-Exclusion (unchanged since v5.1.0)

`device_class in [garage, door, gate]` is always excluded from `scene_set` — these are security/access barriers, not window-light covers. Direct `mode=set` with an explicit `entity_id` still bypasses the scene exclusion, now behind the identity gate below.

## Identity Gate on `mode=set` (v6.0.0)

New `cover_control` certification, same asymmetric-risk shape as `lock_control`/`security_control`:

- **Closing** a cover increases enclosure/security — cert-only (`cover_control` held, no live ack required per call).
- **Opening** a cover is cert-only *unless* it's a barrier-class cover (`device_class in [garage, door, gate]`) — in which case it's cert **and** a fresh live household-admin ack every single call, exact parallel to locks' exterior-unlock gate and `security_manager`'s disarm gate. Non-barrier window covers (blinds/shades/curtains) stay cert-only for open too — opening the living room blinds isn't a security event; opening the garage door is.
- Same per-target `cert_scope` admin-override as locks: `cert_grant cert_component=cover_control cert_scope=["cover.garage_door"]` exempts one specific barrier from the every-call ack. A mixed-coverage call (some scoped, some not) still asks for the whole set.
- Self-publishes `cover_control` in `tool_manifest.certs_required` — this is what makes it a valid `cert_grant` target under the live-calculated catalog (`zen_dojotools_manifest mode=cert_audit`), no separate hand-maintained cert file.

## Dependencies

- `custom_templates/zenos_ai/zen_target_resolve.jinja` — shared targeting core
- `custom_templates/zenos_ai/zen_os_1.jinja` — `envelope()` macro
- `zen_dojotools_identity` — `resolve_caller_identity` (set gate), `request_live_ack` (barrier-open ack)
- `zen_dojotools_persona_editor` — `cover_control` certification grant/revoke
- `zen_dojotools_manifest` — `mode=cert_audit` reads this tool's self-declared `certs_required`

## Cross-References

- [DojoTools Locks](zen_dojotools_locks_readme.md) — the identity-gate pattern this tool adopted, including the `cert_scope` mechanism
- [DojoTools Identity](zen_dojotools_identity_readme.md) — `resolve_caller_identity`, `request_live_ack`
- [DojoTools Profile Editor](zen_dojotools_profile_readme.md) — `cert_grant`/`cert_revoke`
- [Script Modules](readme.md) — return path to the internal tool map

---

## Version History

| Version | Change |
|---------|--------|
| v6.0.0 (2026-08-15) | Identity-gate "cadillac" pass matching locks/security_manager. Migrated onto `zen_target_resolve.jinja`. Full SESE + `envelope()` conversion. New `cover_control` cert (asymmetric: close is cert-only, barrier-open is cert+live-ack, non-barrier-open is cert-only). `cert_scope` per-target ack override. Self-publishes `certs_required` for the live cert catalog. Fixed a `dry_run` preview bug that was blind to a granted `cert_scope` override — a dry-run would report "would require ack" for a target actually exempted. |
| v5.1.0 | Baseline — barrier auto-exclusion classification, scene presets, label taxonomy. |
