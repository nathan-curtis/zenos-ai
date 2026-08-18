# Zen AdminTools CertAdmin — v1.0.0

*Admin-only cert issuance/revocation authority for AI persona (ai_user) cabinets*

---

## Overview

**Admin-only — not MCP-exposed (`mcp_exposed: false`).** Never reachable by an agent, even indirectly through MCP. Run via HA Developer Tools → Services.

`cert_grant`, `cert_revoke`, and `cert_list` were moved out of `zen_dojotools_persona_editor` (agent-facing, MCP-exposed) and into this new admintools script. persona_editor's three cert modes now **delegate here**, forwarding only the fields they already declared (`target`, `cert_component`, `cert_level`, `cert_tool`, `cert_constraints`, `cert_scope`, `stats_patch`, `caller_token`). persona_editor's own field schema never declares `waives_live_ack` or `ack_phrase`, so there is nothing for an agent calling persona_editor to pass through even if it tried — that absence is a hard exclusion, not a permission check layered on top of it.

`zen_dojotools_manifest mode=cert_audit` also reads through this same cert-drawer path for `my_certs` — it's a read-only consumer, not a delegate; certadmin owns all writes.

---

## New Capability: `waives_live_ack`

`cert_grant` only. Lets a household admin issue a scoped exemption from a cert's live-ack requirement (e.g. "don't ask me every time I open the garage door") **without** softening the cert system itself — nothing about the underlying gates, catalog, or default live-ack behavior changes. It's purely additive: an operator can request it, `cert_scope` still bounds it to specific entities/areas (never global), and it costs three independent layers of friction, **all required together**:

1. The existing live-ack gate `cert_grant` already required — a real human approves on their device, same as every other grant. This alone proves a live operator is in the loop right now, not just "some caller reached this script."
2. `confirm_action: true`
3. `ack_phrase` must equal, verbatim: `I UNDERSTAND THIS CERT BYPASSES ACKNOWLEDGEMENT` — same shape as the `run_repair`/label-repair `force_ack` pattern elsewhere in this codebase, deliberately a full sentence rather than a boolean so it can't be set by accident or copy-pasted from an unrelated confirm.

Missing any one of the three **silently drops the flag** — the grant still proceeds as a normal, non-waiving `cert_grant` rather than erroring. A half-satisfied bypass request degrades to the safe default instead of refusing the whole call. The response's `waives_live_ack_granted` and `waives_live_ack_note` fields report which of the two outcomes actually happened.

Because persona_editor's schema structurally cannot pass `waives_live_ack`/`ack_phrase`, obtaining a waiver requires calling certadmin directly as an operator — there is no agent-reachable path to it.

---

## Modes

| Mode | Description |
|------|-------------|
| `help` (default) | Structured reference: all modes, field descriptions, the exact `ack_phrase` text. |
| `cert_grant` | Grant/update a KFC certification on `target=` (default AI user cabinet if omitted). Requires `cert_component` + live-ack approval. `waives_live_ack=true` additionally requires `confirm_action=true` + exact `ack_phrase`. |
| `cert_revoke` | Revoke a certification. Requires `cert_component` + live-ack approval. |
| `cert_list` | List all certifications + stats on `target=`. Read-only, no live-ack. |
| `tool_manifest` | UMP self-description (`zenos_manifest.jinja` contract). |

---

## Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `mode` | select | `help` | `cert_grant` \| `cert_revoke` \| `cert_list` \| `tool_manifest` \| `help` |
| `target` | text | `''` | Explicit target `ai_user` cabinet entity_id. Omit to resolve the default AI user. Refuses (does not fall back silently) if an explicit value is passed but doesn't resolve to a real entity. |
| `cert_component` | text | `''` | `kata_key` of the KFC to certify. Required for `cert_grant`/`cert_revoke`. |
| `cert_level` | number (1–4) | `2` | 1=Observer, 2=Advisor, 3=Operator, 4=Autonomous. |
| `cert_tool` | text | `''` | Script or tool used by this persona for the KFC. Optional. |
| `cert_constraints` | text (multiline, JSON array) | `''` | Actions requiring explicit per-action confirmation, even with this cert. |
| `cert_scope` | text (multiline, JSON array) | `''` | Per-target exemptions from a cert's live-ack requirement. Each item is either a bare `entity_id` string (shorthand for allow) or a `{"entity": "...", "acl": "allow"\|"deny"\|"remove"}` object. **Merges** into any existing scope for this `cert_component` — does not replace it. `acl="remove"` deletes one entity from the merged scope without a full `cert_revoke`. |
| `waives_live_ack` | boolean | `false` | `cert_grant` only. See **New Capability** above — requires `confirm_action: true` and exact `ack_phrase` together, in addition to the normal live-ack approval every `cert_grant` already requires. |
| `confirm_action` | boolean | `false` | `waives_live_ack` only — one of the three required gates. |
| `ack_phrase` | text | `''` | `waives_live_ack` only — must equal verbatim: `I UNDERSTAND THIS CERT BYPASSES ACKNOWLEDGEMENT`. |
| `stats_patch` | text (multiline, JSON) | `''` | Optional deep-merge patch into this cert's `stats` block. |
| `caller_token` | text | — | Opaque token passed through by the caller for correlation. Not interpreted by this script. |

---

## Behavior Notes

- **Catalog check** (moved verbatim from persona_editor's `cert_grant`): `cert_component` must be declared by some tool's own `tool_manifest` `certs_required` — there is no separate hand-maintained catalog. `cert_grant` calls `zen_dojotools_manifest mode=cert_audit` to read the live catalog and refuses with the list of available components if `cert_component` isn't in it.
- **Live-ack is unconditional** for both `cert_grant` and `cert_revoke` — requesting `waives_live_ack` does not skip it; it's layer 1 of the 3 required for the waiver itself.
- **Scope merge, not replace**: `cert_grant` merges `cert_scope` entries into any existing scope for that component (ported verbatim from persona_editor). When `waives_live_ack` is actually granted, `waives_live_ack: true` is stamped only onto the merged scope entries that were part of *this* request — existing entries from prior grants keep whatever waiver state they already had unless re-listed in the current call.
- **Target resolution**: refuses if an explicit `target` doesn't resolve to a real entity, or if no default AI user cabinet resolves and none was given — never silently writes to a nonexistent cabinet.
- Certification and revocation both write through `script.zen_dojotools_filecabinet` (`action_type: create`, `force_action: true`) to the `zen_ai_certs` drawer.

---

## Related

- `zen_dojotools_persona_editor` — its `cert_grant`/`cert_revoke`/`cert_list` modes are thin delegates to this script; it is agent-facing and MCP-exposed, this is not.
- `zen_dojotools_manifest mode=cert_audit` — read-only consumer of the same cert-drawer path, surfaces `my_certs`.
- `zen_dojotools_identity mode=request_live_ack` — the live-ack chokepoint used by both `cert_grant` and `cert_revoke`.
