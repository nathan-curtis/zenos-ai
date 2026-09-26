# 18. Certification

A certification is a named capability, held at a level, by an AI persona. A tool that does something consequential declares which certification it requires and at what level, and it refuses the action unless the caller holds it. This is the authorization model ZenOS runs today. It replaced the ACL and session-token design in Volume 1, which never shipped.

## 18.1 The catalog is declared, not maintained

There is no hand-maintained list of certifications. Each tool declares the certifications it enforces in its own `tool_manifest`, under `certs_required`:

```text
{'cert': 'calendar_control', 'level': 2, 'modes': ['delete']}
```

`zen_dojotools_manifest mode=cert_audit` fans out across every tool's manifest and assembles the live catalog from those declarations. Granting a certification that no tool declares is refused, which catches typos and dead names. The catalog currently holds 45 distinct certification names across the tool surface.

The declaration and the gate are separate pieces of code, and they can drift. A tool that enforces a mode its manifest does not list will under-report itself in the catalog. `mode=audit_help` and `cert_audit` exist partly to catch that.

## 18.2 Names

Certification names come in two forms.

**Flat names** describe one capability on one tool family: `lock_control`, `cover_control`, `security_control`, `lighting_control`, `calendar_control`, `pii_disclosure_control`.

**Dotted names** live in the `zenos.*` namespace and form a hierarchy: `zenos.media.generate`, `zenos.media.print_control`, `zenos.media.playback_control`, `zenos.comms.dispatch`, `zenos.identity.profile_write`, `zenos.system.reload_restart`, `zenos.system.home_config_write`.

Dotted names inherit, the same way X.509 policy identifiers do. Holding a parent node satisfies a check against any descendant at a dot boundary: holding `zenos.media` at level 2 satisfies a check for `zenos.media.playback_control` at level 2. When a caller holds both a parent and a more specific child, the most specific match wins, so a narrow grant can restrict below what a broad grant would imply. A flat name only ever matches itself. The resolution lives in `resolve_caller_identity` (Chapter 17).

## 18.3 Levels

A level is an integer, 1 and up. A gated action declares the minimum level it needs, and holding the certification at or above that level satisfies it. Levels have no global meaning. Each tool defines its own scale, typically putting a destructive or broader action one level above the ordinary one:

| Tool | Certification | Level 1 | Level 2 or higher |
|---|---|---|---|
| Mail, Teams | `pii_disclosure_control` | send (Mail `create`, Teams `send`) | edit the mail whitelist (2) |
| Calendar, To Do | `calendar_control`, `todo_control` | (none defined) | delete (2) |
| Infra | `infra_container_control` | (none) | restart or start (2); stop or remove (3) |
| Room Manager | `room_topology_edit` | structural edits | delete an area (2) |

## 18.4 Where certifications live

Certifications are stored on the AI persona that holds them: the persona's cabinet, drawer `zen_ai_certs`, under `kfc_certifications`, keyed by certification name. Each entry records:

| Field | Meaning |
|---|---|
| `level` | The level held. |
| `certified` | The date it was granted. |
| `tool` | The tool the grant was issued against. |
| `scope` | Optional per-target scope entries (18.5). |
| `constraints` | Optional additional restrictions, interpreted by the consuming tool. |

`zen_dojotools_identity mode=cert_list` answers two questions in one call: what certifications exist (the live catalog) and what the resolved caller actually holds.

> **Not yet built.** Certifications do not expire and are not checked against a revocation list at use time. A grant stays in force until it is revoked explicitly. Expiry and lifecycle are design direction.

## 18.5 Scope

A certification can carry scope: a list of entries naming specific targets and what the grant means for each one.

```text
{'entity': 'lock.front_door', 'acl': 'allow'}
{'entity': 'lock.back_door',  'acl': 'deny'}
```

The shared `cert_scope_check()` macro in `zen_os_1.jinja` answers one question for a given target: `allow`, `deny`, or `default` (not mentioned). Tools use that answer in two ways.

**`deny` is a hard block.** A target covered by a `deny` entry is refused outright, ahead of any other check, and no live acknowledgement is offered. This is how you certify an agent for locks in general while guaranteeing it can never touch one specific door.

**`allow` covers the per-call acknowledgement.** Tools with a "ask a human every time" tier (exterior unlocks, opening a barrier-class cover, disarming the alarm, destructive container actions) skip that ask for targets the caller's scope explicitly allows. Targets not mentioned still ask. If a single call touches both covered and uncovered targets, the whole call asks. It never splits into a partial silent action.

> **Not yet built.** CertAdmin records a `waives_live_ack` flag on scope entries, and setting it requires a live acknowledgement, `confirm_action: true`, and a verbatim acknowledgement phrase. No gated tool reads that flag today. The tools treat any `allow` scope entry as covering the per-call ask, so the effective control is the scope grant itself, which does require a live acknowledgement to issue. Making the tools require `waives_live_ack` specifically is an open design decision.

## 18.6 Bundles

A bundle is a named set of certifications granted together, at one level, with one live acknowledgement instead of one per member. Bundles come from two sources, which `cert_audit` merges into one view:

1. **Declared:** a `bundle` tag on a certification's own entry in a tool's `certs_required`. Changing it requires a code change.
2. **Custom:** the household cabinet's `cert_bundles_custom` drawer, editable at runtime through `zen_admintools_certadmin mode=cert_bundle_set`. This is the place for sets that must be editable without a redeploy, such as a recovery set.

A bundle grant is deliberately simple: uniform level, no per-member scope, no per-member constraints. Anything finer than that is a set of single grants.

No shipped tool currently declares a bundle tag. Bundles exist today only as custom household definitions.

## 18.7 What a denial looks like

A tool that refuses an action for lack of certification builds its answer with the shared `cert_denial()` macro, so every denial in the system has the same shape: `status`, `error` (for example `cert_insufficient` or `identity_policy_blocked`), a human-readable `message`, `can_request`, and `request_via`.

`can_request` is computed centrally from the identity policy alone: true unless the caller's identity was itself blocked. No tool decides for itself whether a caller is eligible to ask. A tool requires the certification. Whether the caller may request it is the certification system's question, answered in one place.

<!-- nav -->
---

[← Principals](17_principals.md) · [Contents](00_toc.md) · [The Authority Ladder →](19_the_authority_ladder.md)
<!-- /nav -->
