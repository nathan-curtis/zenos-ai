# ZenOS-AI Security Model — GA Reference

**Version:** Architecture record, refreshed for 2026.6.0 'Clue'
**Status:** Historical GA security model with current operator caveats

---

## The Short Version

ZenOS-AI has a complete security architecture. This document records the GA security model:
identity, principal chains, delegation limits, group nesting rules, and the claims engine.
For Clue-era runtime setup, use the getting-started exposure guide and the current
Profile Editor, Identity, FileCabinet, and Postman references.

Do not use this page as the list of tools to expose to Assist. Expose the documented
minimum DojoTools surface; keep AdminTools and raw HA API tools internal unless a human
administrator deliberately enables them for recovery.

---

## What Is Active at GA

### Identity and Principal Resolution

Every principal — human or AI — has:

- A stable GUID and identity hash
- A cabinet drawer (AI User, Family, or User cabinet depending on type)
- A provenance record: origin system, household, family
- An essence capsule: identity, personality, capabilities, lineage

The identity tool (`zen_dojotools_identity`) resolves the full principal chain at runtime:
HA user → Household → Head of Household → AI GUID. This chain is the trust spine.

### Three-Layer Identity Schema

Identity records follow the canonical three-layer structure:

```
Layer 1: familiar     — runtime name, species, role (what Friday calls herself)
Layer 2: companion    — operational capsule: traits, lineage, directives, capabilities
Layer 3: jacket       — provenance: GUID, origin, signature, signed_by, signed_at
```

The jacket is the trust anchor. The familiar is the persona surface. The companion is what
the model reasons from.

### Prompt Integrity — `sensor.zen_prompt_health`

Every AI user cabinet has a prompt health sensor that reports:

| State | Meaning |
|---|---|
| `ok` | Three-layer schema + md5 signature + manifest present |
| `warn` | Signature pending or manifest missing |
| `error` | Legacy schema, missing essence, or cabinet unavailable |

The `prompt_health_check()` macro (in `zen_os_1.jinja`) is the single source of truth.
The sensor and the identity tool's `mode: prompt` both surface from it.

### Delegation and Nesting Hard Rules

Two structural limits are active and enforced at GA:

- **`max_delegation_depth: 2`** — A principal cannot delegate through more than 2 hops.
  Claims at depth 3+ are treated as absent (silent, not errored).
- **`max_nesting_depth: 2`** — Group nesting ceiling. A Family group can contain members
  and subgroups, but ZQ-1 resolution walks at most 2 levels deep.

These are not configuration — they are hard constants in the resolution engine.

### caller_token Plumbing

`caller_token` accept-and-echo plumbing has grown with the tool surface since GA —
41 scripts under `packages/zenos_ai/dojotools/` carry it as of 2026-08-09 (verified via
`grep -rl caller_token`; count includes `zen_stack_timer`/`zen_stack_alarms`, two
internal-only Lens Bus stack providers, not just MCP-exposed tools), up from the 15
present at the original 4.5.0 'Meridian' GA snapshot (see
`20_tool_invocation_and_security.md` §GA Implementation Note for that historical
figure). The field remains a noop — it passes through and is reflected in the
response. The plumbing path from MCP tool call through to response is complete.
SP1 enforcement requires no changes to script internals.

### Profile Snapshot and Restore

The profile editor (`zen_dojotools_profile`) now snapshots `zenai_essence` →
`zenai_essence_prev` before every write. A `mode: restore` call rolls back to the last
known good state. `mode: read` surfaces `prev_snapshot_exists` so you can inspect before
committing. The audit trail is written as HA events (`essence_backed_up`,
`essence_restored`).

---

## The `security_policy` Drawer — What Is Stubbed

The System Cabinet carries a `security_policy` drawer. Fields active at GA are enforced
now. Fields marked SP1 are plumbed but are a no-op until SP1 activates them.

```yaml
security_policy:
  secure: false           # GA: active. false = enforcement off. true activates SP1 mode.
  max_delegation_depth: 2 # GA: active. Hard limit — enforced now.
  max_nesting_depth: 2    # GA: active. Hard limit — enforced now.

  # SP1 fields — plumbed, documented, not enforced until secure: true
  provider: ""            # authentik | oidc | custom
  token_endpoint: ""      # required when provider is set
  validation_mode: ""     # oidc | fido2 | custom
  claims_cache_ttl: 300   # session claims TTL in seconds
```

**Populating the SP1 fields at GA is a no-op.** They are stored but not read by any
enforcement path. Do not populate them and expect enforcement — it is not active.

---

## What Activates at SP1

Setting `secure: true` in `security_policy` is the single activation gate for the full
enforcement layer. When active:

- `caller_token` field is validated against the external provider
- Claims are resolved via the HyperIndex fold on the principal entity
- Tool access is gated by the resolved claim set
- Family cabinet `partner_registry` and AI jacket `principals[]` are cross-validated
- Delegation chain depth is checked at claim resolution time (already enforced structurally)

**ZenOS is the policy plane. The external provider is the auth plane.** The provider is
pluggable — the interface is the same regardless of whether you wire Authentik, a custom
OIDC stack, or something else. ZenOS does not care what's behind the endpoint, only that
the token validates.

---

## The Claims Engine — HyperIndex

The HyperIndex is the claims resolution engine at SP1. An index fold on a principal entity
produces a canonical claims dict from the label graph. ZQ-1 reads that dict at compile time
— that IS the authorization context for the call.

No separate claims service is required. The label graph already encodes the authorization
structure. The fold is the resolution step.

At GA, the index fold runs for context assembly. At SP1, the same fold runs for claims —
same engine, different consumer.

---

## Family and Partnership Registry (SP1 Schema — Stubbed at GA)

The Family Cabinet carries two new drawers with schema defined but not yet enforced:

**`family_registry`** — M:N human membership:
```json
{
  "members": [
    { "person": "<entity_id>", "role": "member|hoh|guest", "household": "<guid>" }
  ]
}
```

**`partner_registry`** — M:N AI partnership:
```json
{
  "partnerships": [
    { "ai": "<guid>", "principals": ["<guid>"], "role": "aipartner|prime_ai" }
  ]
}
```

Bidirectional validation at SP1: the AI jacket `principals[]` field and the family
`partner_registry` must agree. If they diverge, the claim is rejected.

---

## ACL Hierarchy

When a principal requests access to a drawer, the resolution order is:

1. Drawer-level ACL (most specific — wins)
2. Cabinet-level ACL
3. Family group policy
4. Household policy
5. Global restrictions (deny always overrides allow)

This hierarchy is the structural spec at GA. Enforcement at the tool layer activates at SP1.

---

## For Operators: What You Need to Know at GA

**You don't need to configure anything for security at GA.** The defaults are safe:

- `secure: false` means the enforcement gate is open — your AI has full access to what
  it's exposed to, scoped by labels and cabinets as designed.
- Delegation and nesting limits are enforced structurally regardless.
- Prompt integrity is monitored — `sensor.zen_prompt_health` tells you if something
  is wrong with the identity capsule before the agent starts reasoning from a broken state.

**When SP1 ships**, you will populate `provider` and `token_endpoint`, set `secure: true`,
and the system activates. No architectural changes required — the plumbing is already there.

**KFC certification grants are gated independently of SP1** (2026-08-15). `zen_dojotools_persona_editor mode=cert_grant`/`cert_revoke` previously had zero gating — any MCP caller could self-certify itself for any capability, including one an identity gate was built the same day to protect. Now closed with two non-optional checks regardless of SP1 status: the target certification must appear in the live-calculated cert catalog (`zen_dojotools_manifest mode=cert_audit`, fanning out to every tool's own self-declared `certs_required` — briefly a hand-maintained JSON file the same day, replaced same-day once that proved to be an admin burden), and every grant/revoke requires a fresh live household-admin acknowledgment, which is the actual security boundary — catalog membership alone only catches typos. See `zen_dojotools_profile_readme.md`'s certification section for the mechanism.

---

## Related

- `09_Identity_Architecture.md` — full identity data model, ACL rules, Squirrel Safe / Content Safe filters
- `roadmap.md` — SP1 timeline and scope
- `docs/scripts/zen_dojotools_identity_readme.md` — identity tool reference, `request_live_ack`/`cert_list`
- `docs/scripts/zen_dojotools_profile_readme.md` — `cert_grant`/`cert_revoke` gating
- `docs/scripts/zen_dojotools_locks_readme.md` — identity-gate pattern applied to lock actuation
- `sensor.zen_prompt_health` — prompt integrity sensor
