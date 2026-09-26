# 17. Principals

A principal is anything that can be the subject of an identity check: a person, an AI persona, a family, or a household. This chapter covers what principals are, how they relate, and how a tool finds out who it is dealing with. Chapter 18 covers what a principal is allowed to do.

## 17.1 Principals are cabinets

Every principal is backed by a cabinet, and the cabinet's type label says what kind of principal it is:

| Principal | Cabinet type label | Notes |
|---|---|---|
| Household | `zen_household_cabinet` | The occupancy boundary. One per install by default. |
| Family | `zen_family_cabinet` | The belonging boundary. Families can nest. |
| Person | `zen_user_cabinet` | A human, linked to a `person.*` entity. |
| AI persona | `zen_ai_user_cabinet` | An agent persona. Holds the persona's essence and its certifications. |

A cabinet's identity is its GUID, stored in its `AI_Cabinet_VolumeInfo` header. Names, friendly names, and entity IDs can change. The GUID does not.

Cabinets come into service through `zen_dojotools_provisioner`. A cabinet waiting in the stacks pool is `init` or `online_unmounted`. Provisioning validates its GUID (present, valid UUID, unique across mounted cabinets), applies the type label, mounts it, and seeds the profile and essence drawers. Deprovisioning reverses it and returns the cabinet to the pool. Provisioning a principal is identity-slot activation, not hardware management. Both provision and deprovision require the `cabinet_lifecycle_control` certification.

## 17.2 Occupancy and belonging

`zen_dojotools_identity` models two separate relationships, and keeping them separate is deliberate.

A **household** is occupancy: who lives here. Members join with `household_add_member` and leave with `household_remove_member`. A household holds two principal slots: the head of household (a person) and the prime AI (an AI persona). The first user provisioned fills the head-of-household slot; the first AI fills the prime slot. `set_principal` replaces either.

A **family** is belonging: who is part of whose group. Families contain people, AI personas, and other families. Members join with `family_add_member`. A person's first family is their default; `set_default_family` changes it. An AI persona joins the household automatically but joins a family only by explicit invitation. Being in the house and being part of the family are different facts.

Joining or leaving a family fires a `zen_event` (`family_member_joined` or `family_member_left`) so an agent can offer to update naming or profile details.

The data model allows families to nest to any depth. Security resolution walks at most two levels. `membership` returns the tree (principals, members, sub-families) to depth two, and `is_member` answers a depth-two membership question directly.

## 17.3 Partners

A partner link is delegation, not a social relationship. `link_partners` writes each principal into the other's `acls.partner`, meaning each is authorized to act on the other's behalf. It works between any pair: person and person, person and AI persona, or two AI personas. `unlink_partners` severs it.

> **Not yet built.** Partner links are recorded but no gated tool reads `acls.partner` when deciding whether to allow an action today. Authorization runs through certification (Chapter 18). Partner-aware authorization is design direction.

## 17.4 Resolving the caller

A gated tool never works out on its own who is calling. It asks identity, through one mode:

`zen_dojotools_identity mode=resolve_caller_identity`

This is the chokepoint for every certification check in the system. The tool passes the certification it needs (`required_cert`, `required_cert_level`), and identity resolves the caller, looks up that caller's certifications, and returns the answer in one response. A tool that resolved the identity itself and then read a certification store separately could be pointed at the wrong store. Doing both in one call closes that gap.

The response carries, among other fields:

| Field | Meaning |
|---|---|
| `policy_status` | `allowed` or `blocked`. A blocked identity fails every certification check regardless of what it holds. |
| `authorized` | Whether the resolved identity holds `required_cert` at or above `required_cert_level`. |
| `cert_level` | The level actually held. |
| `cert_scope` | The per-target scope entries on the matching certification (Chapter 18). |
| `scope_decision` | The scope answer for the tool's target, when the tool passed one: `allow`, `deny`, or `default`. |
| `block_reason` | Why identity was blocked, when it was. |

Every cert-gated tool reads these fields through the shared `resolve_identity_fields()` macro in `zen_os_1.jinja`, so a change to this response shape is a one-file edit.

### Simulated identity and the policy switch

Identity resolution delegates to an authentication root, `zen_root_authentik`. Today that root is a stub, and resolution returns `sim_mode: true`, meaning the identity is the install's default agent rather than a cryptographically verified caller.

Whether simulated identity is acceptable is a whole-install policy, not a per-tool choice. It lives in the household cabinet at `integrations_config.identity.sim_mode_allowed`, and the factory stamps an explicit value into it on every install. The default is `false`. When simulated identity is not allowed, `resolve_caller_identity` returns `policy_status: blocked`, and every certification check fails closed.

> **Not yet built.** There is no cryptographic binding between an MCP session and a specific persona cabinet. Every call currently resolves to the default agent, so "authorized" means "the default agent holds this certification." `caller_token` is threaded through the tool surface and returned unchanged, and `caller_id` is free-text audit metadata. Neither is an identity claim, and nothing may treat them as one. When real session binding replaces the stub, every tool inherits per-principal evaluation through the same chokepoint without changes of its own.

## 17.5 The identity manifest

Resolving every principal on every prompt would be expensive, so identity caches the roster. `build_identity_manifest` writes a `zen_identity_manifest` drawer to the household cabinet. The Scheduler rebuilds it on Home Assistant start and at midnight, and every roster-changing operation triggers a rebuild. The prompt loader reads the manifest first and falls back to live resolution when it is missing or stale (Chapter 16).

## 17.6 Presence and consent

When identity resolves a person, it can include a presence block: zone, whether they are home, and which area they are in. Each part is consent-gated in the person's own profile (`tracking.gps_zone`, `tracking.room`). If a person has not consented to room tracking, identity does not report their room, even when the data exists.

<!-- nav -->
---

[← Context Construction](16_context_construction.md) · [Contents](00_toc.md) · [Certification →](18_certification.md)
<!-- /nav -->
