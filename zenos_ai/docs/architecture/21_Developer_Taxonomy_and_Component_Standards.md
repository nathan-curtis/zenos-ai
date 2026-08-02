# ZenOS-AI Developer Taxonomy

**Status:** Draft architecture standard
**Source baseline:** ZenOS-AI `feat/2026.9.0`
**Audience:** ZenOS-AI core developers, plugin authors, integration maintainers, and reviewers

## 1. Purpose

Home Assistant provides the executable primitives used by ZenOS-AI: scripts, automations, template sensors, REST commands, helpers, and packages. Those primitives explain how code runs. They do not explain its architectural responsibility.

ZenOS-AI adds a vocabulary that identifies:

- who or what should call a component;
- how much authority it holds;
- where it sits in the dependency graph;
- whether it is part of Friday's callable tool surface;
- whether it contributes scheduled cognition through a Kung Fu Component (KFC);
- how it should be packaged and distributed.

For example, a DojoTool and a Root may both be Home Assistant scripts. The DojoTool is a supported semantic capability for an agent. The Root is an internal transport primitive. Treating them as interchangeable because both are scripts would erase a security and abstraction boundary.

This document defines the canonical component classes, Stripes, exposure rules, packaging conventions, and the decision process for choosing the correct design.

## 2. The Five Independent Axes

Every significant ZenOS-AI component should be described across five independent axes.

| Axis | Question answered | Typical values |
|---|---|---|
| Class or tier | What architectural job does this perform? | DojoTool, AdminTool, Sutra, Root, Stack, Codex, Boot Orchestrator |
| Stripe | What must exist before this can be initialized or inspected? | 0, 1, 2, 3, or *outside the sweep* |
| Exposure | May an agent call this directly? | MCP-exposed, internal, operator-only |
| Cognitive role | Does this generate or supply recurring context? | KFC provider, KFC seed, Lens provider, none |
| Runtime form | How does Home Assistant execute it? | script, automation, template/health sensor, REST command |

These axes must not be collapsed into one another.

- A Stripe does not grant or deny authority.
- A filename prefix does not replace an explicit exposure decision.
- A KFC is a context contract, not a security class.
- Being implemented as a script does not make a component a DojoTool.
- A DojoTool can depend on several internal classes while remaining the only public entry point.

## 3. Architectural Classes

### 3.1 DojoTools

**Prefix:** `zen_dojotools_*`
**Default Stripe:** 3
**Typical caller:** Friday, another approved agent, a ZenOS orchestration component, or an automation
**Default exposure:** MCP-exposed when intended as an agent capability

A DojoTool is the supported semantic capability surface of ZenOS-AI. It accepts a request expressed in household or system terms, validates it, applies relevant identity and policy controls, invokes internal implementation layers, and returns structured evidence.

Examples:

- `zen_dojotools_taskmaster`
- `zen_dojotools_room_manager`
- `zen_dojotools_spa_manager`
- `zen_dojotools_inventory`
- `zen_dojotools_library`
- `zen_dojotools_filecabinet`

Taskmaster demonstrates the intended shape. It exposes task-oriented modes, chooses among To Do, Radar, Inventory, and CRM based on task shape and configured capability, and preserves each backend's native model. The agent receives one coherent capability without receiving each backend's raw interface.

#### Why DojoTools exist

- They provide stable, intent-oriented contracts for agents.
- They constrain broad Home Assistant and external-service APIs.
- They centralize validation, confirmation gates, identity checks, and policy.
- They return predictable JSON suitable for downstream reasoning.
- They keep backend changes from becoming prompt or agent changes.
- They reduce the number of directly exposed tools.

#### DojoTool best practices

1. Design modes around user or system intent, not HTTP verbs or implementation details.
2. Use `mode` as the canonical selector. Retain `action_type` only where compatibility requires it.
3. Provide `tool_manifest`; provide `kfc_manifest` when the tool owns KFC definitions.
4. Return structured responses with stable `status`, `mode`, result/evidence, warnings, and trace information where appropriate.
5. Fail soft on optional integrations. Report unavailable enhancements without breaking the baseline capability.
6. Validate identifiers against authoritative ZenOS or Home Assistant sources before writes.
7. Apply preview and `confirm_action` patterns to consequential changes.
8. Pass `caller_token` through the call chain even when a current backend runs in simulation mode.
9. Route through internal Roots, Sutras, Stacks, or Codices instead of embedding every layer in the public contract.
10. Expose only modes suitable for the intended caller. An internal helper mode does not automatically belong in the agent-facing description.

#### Use a DojoTool when

- Friday should perform or request the capability directly;
- the operation can be expressed as a meaningful household or ZenOS intent;
- several backend operations should appear as one supported capability;
- the capability needs policy, validation, or progressive enhancement;
- an automation and an agent should share the same implementation.

### 3.2 SystemTools and kernel services

**Typical prefix:** `zen_dojotools_systemtools` or another explicitly kernel-labeled DojoTool
**Default Stripe:** 1 for kernel services; 3 for ordinary public utility surfaces
**Typical caller:** ZenOS runtime, Flynn, health sweeps, or Friday through a restricted facade

SystemTools concern ZenOS itself rather than a household domain. They provide health reporting, event emission, runtime inspection, garbage collection, and other controlled system services.

SystemTool is a role within or adjacent to the DojoTool namespace, not a universal filename prefix. The manifest's Stripe and exposure fields make the distinction explicit.

#### Why this role exists

- Core runtime services must be available before normal user-facing tools.
- Friday needs safe diagnostic access without receiving unrestricted administrative authority.
- Health sweeps require predictable, low-level service checks.

#### Best practices

- Separate read-only health and inspection modes from repair modes.
- Keep structural mutation in AdminTools.
- Mark kernel dependencies and Stripe explicitly.
- Ensure health reporting remains useful during partial startup failure.
- Avoid dependencies on Stripe 2 or 3 components unless the mode clearly degrades when they are unavailable.
- When a diagnostic check itself depends on a network call (an internal HA REST call included), distinguish "checked, and the thing is absent" from "couldn't check." Collapsing both into one boolean produces a confident false negative the moment the check's own transport fails — see §5.2.

### 3.3 AdminTools

**Prefix:** `zen_admintools_*`
**Stripe:** Outside the normal manifest bootstrap sweep. AdminTools are invoked directly by Flynn or an operator, not health-swept or discovered via `bootstrap_stacks`/`bootstrap_kfc`, so they are not required to declare a Stripe the way MCP-exposed and Lens-registered components are. If an AdminTool's manifest declares one anyway (for documentation or dependency-graph clarity), it should reflect the Stripe of the subsystem it repairs — e.g. a cabinet-repair tool documents itself as Stripe-0-adjacent — not a Stripe that implies it participates in the sweep.
**Typical caller:** Flynn, a trusted administrator, onboarding/recovery flows, or a deliberately privileged agent surface
**Default exposure:** internal or operator-only

AdminTools alter ZenOS structure, security policy, capability policy, installation state, or recovery state. Examples include cabinet repair, template loading, manifest/KFC installation, ACL configuration, migrations, and reset operations.

The word **administrative** refers to the ZenOS control plane. It does not mean every powerful real-world action belongs in an AdminTool. Restarting a container, operating security equipment, or performing another high-impact action may remain a normal DojoTool capability when a Codex and SP1 enforce the appropriate scoped authority.

Examples:

- `zen_admintools_cabinetadmin`
- `zen_admintools_prompt_loader`
- `zen_admintools_reset_template`
- `zen_admintools_portainer_acl`

#### Why AdminTools exist

- Structural operations have a larger blast radius than ordinary household actions.
- Repair and bootstrap often need authority that Friday should not normally possess.
- Separating these operations makes audits and exposure reviews tractable.
- Flynn can operate the recovery layer without broadening the normal agent surface.

#### Exposure rule

AdminTools are not part of Friday's ordinary MCP surface. If an agent needs to perform a powerful operation, build a purpose-specific DojoTool and place the authorization, certification, acknowledgement, and fail-closed rules in its Codex/SP1 path. Do not expose the AdminTool as a shortcut.

Portainer demonstrates the separation:

- `zen_dojotools_infra` is the agent-facing capability;
- its container-control Codex evaluates scoped authority and action class;
- `zen_root_portainer` performs internal transport;
- `zen_admintools_portainer_acl` configures the allow-list and policy and remains outside MCP.

#### AdminTool best practices

1. Default to internal or operator-only exposure.
2. Require explicit targets. Never infer a broad destructive scope.
3. Use preview, confirmation, and idempotent repair modes.
4. Report exact impact, fallback, and recovery guidance.
5. Keep secrets and raw credential material out of responses.
6. Make bootstrap and repair safe to repeat where possible.
7. Let Flynn call the smallest necessary operation rather than a universal administrator mode.
8. Separate policy writers from action tools. A DojoTool/Codex may evaluate its ACL; only the AdminTool should configure it.
9. Fire the relevant health-sensor refresh event on every write completion (see §5.2). A trigger-based health sensor that nothing ever wakes up is a silent staleness bug, not a passing test.

#### Use an AdminTool when

- the operation changes configuration or security policy;
- it repairs or migrates ZenOS structures;
- it installs, stamps, or removes runtime components;
- it changes an ACL, manifest, template, cabinet schema, or bootstrap state;
- failure could impair the operating system rather than one household task.

### 3.4 Roots

**Prefix:** `zen_root_*`
**Default Stripe:** 2, except a foundational primitive may be Stripe 0
**Typical caller:** DojoTool, Sutra, Stack, Codex, or Flynn
**Default exposure:** internal, never directly MCP-exposed

A Root is the lowest reusable integration or protocol primitive. It knows how to communicate with a backend and returns a deliberately shaped low-level result. Roots may own REST calls, authentication headers, endpoint construction, transport error normalization, and secret-safe response shaping.

Examples:

- `zen_root_firefly`
- `zen_root_wikijs`
- `zen_root_portainer`
- `zen_root_authentik`

`zen_root_authentik` also shows that a Root may begin as an honest stub. It establishes the stable call contract and a single `sim_mode` transition point before the live OIDC implementation exists.

#### Why Roots exist

- Several higher layers can reuse one backend client.
- Credentials and transport concerns stay in one auditable place.
- Public tools avoid leaking native API shapes.
- Network implementation can change without redesigning domain logic.
- Responses can be filtered before raw backend data reaches other layers.

#### Root best practices

1. Keep the contract narrow and transport-oriented.
2. Never expose a Root directly to an agent.
3. Resolve configuration from approved cabinet locations and secrets from Home Assistant secrets.
4. Do not return raw payloads that may contain credentials, environment variables, tokens, or excessive PII.
5. Normalize HTTP and transport failures into stable internal results — and surface the failure as a *distinct* outcome from "checked and found nothing," never collapsed into the same field (see §5.2).
6. Avoid embedding household policy or conversational interpretation.
7. Support caller tracing without treating token shape as authenticated identity.
8. For a stub, declare `sim_mode` explicitly and centralize the future flip point.

#### Use a Root when

- multiple components need the same backend transport;
- the service requires authentication or careful payload filtering;
- the native API is too broad or unstable to repeat across callers;
- a clean protocol boundary materially improves testing and security.

### 3.5 Sutras

**Prefix:** `zen_sutra_*`
**Default Stripe:** 2; `zen_sutra_filecabinet` is Stripe 0
**Typical caller:** DojoTools, Stack providers, Codices, and internal runtime components
**Default exposure:** internal

A Sutra is an internal operational broker or integration adapter. It understands a backend's functional vocabulary and may compose Roots, REST commands, Home Assistant services, or cabinet operations into reusable internal actions.

Examples include the Grocy, Mealie, Zammad, Paperless, Music Assistant, and FileCabinet Sutras.

#### Root versus Sutra

| Root | Sutra |
|---|---|
| Owns protocol and transport primitives | Owns reusable backend operations |
| Speaks endpoints, methods, and shaped payloads | Speaks service-specific functions and workflows |
| Avoids domain policy | May contain backend-specific operational rules |
| Often used by several higher layers | Often supports one integration facade and its related providers |

A small integration may use a Sutra without a separate Root. Create both only when the transport layer is genuinely reused or complex enough to deserve its own boundary.

#### Sutra best practices

1. Keep it internal and omit it from the normal agent tool bundle.
2. Normalize backend quirks, pagination, and service-specific data shapes.
3. Expose narrowly useful internal modes instead of mirroring an entire external API.
4. Return enough detail for the calling DojoTool to produce good evidence.
5. Keep user-facing policy and prose in the DojoTool.
6. Use a Root when transport logic would otherwise be duplicated.
7. Declare Stripe 2 unless the Sutra is a true foundational primitive.
8. Any write path a Sutra owns that other components key their own state off of (a health sensor, a cache, a resolver) must fire the corresponding refresh event on completion. Do not assume implicit reactive dependency tracking will notice a write made through an imported macro — it frequently will not (see §5.2).

### 3.6 Stack providers

**Prefix:** `zen_stack_*`
**Default Stripe:** 2
**Typical caller:** `zen_dojotools_library` through the Lens Bus
**Default exposure:** internal and indirectly reachable through Library

A Stack provider adapts a knowledge source to the common Lens Bus contract. Library accepts generic knowledge verbs and routes them to the registered provider. The provider translates those verbs to its native backend and returns normalized evidence.

Examples:

- `zen_stack_paperless`
- `zen_stack_radar`
- `zen_stack_media`
- `zen_stack_firefly`
- `zen_stack_battery`
- `zen_stack_depreciation`

#### Why Stacks exist

- Library remains the single agent-facing knowledge broker.
- Providers can be added without adding a direct agent tool.
- Evidence shape, provenance, limitations, and health are normalized.
- Generic verbs such as `get`, `find`, `list`, and `by_anchor` can work across heterogeneous sources.
- Providers can self-register and fail soft when unconfigured.

#### Stack best practices

1. Implement the Lens contract instead of inventing a parallel query surface.
2. Register through the supported manifest/registration path.
3. Declare anchors, consumed types, returned evidence types, health, and limitations.
4. Route agent calls through Library.
5. Preserve provenance and redaction policy in returned evidence.
6. Fail soft when the provider is absent or unconfigured.
7. Delegate native backend work to a DojoTool, Sutra, Root, or Codex as appropriate.
8. Give a Stack its own `tool_manifest` self-describe branch even when it mostly proxies to an owning DojoTool. A generic proxy fallthrough that never intercepts `mode=tool_manifest` returns the owning tool's identity instead of the Stack's own — indistinguishable from calling the backing tool directly, and inconsistent with every Stack that does self-describe correctly.

#### Use a Stack when

- the source contains searchable or retrievable knowledge;
- its results should participate in Library and Lens queries;
- consumers benefit from common anchors such as area, person, label, mood, item, or company;
- the provider should be pluggable without expanding the MCP surface.

### 3.7 Codices

**Prefix:** `zen_codex_*` when independently addressable; a Codex may also be an explicitly bounded internal policy module within its owning DojoTool
**Default Stripe:** 2
**Typical caller:** DojoTools, automations, Stack providers, or related domain components
**Default exposure:** internal

A Codex is executable domain expertise. It contains durable business rules, calculations, state transitions, or derived models that are too substantial to bury in a general facade or backend adapter.

Examples:

- `zen_codex_finance_depreciation`
- `zen_codex_finance_cogs`
- the container-control Codex inside `zen_dojotools_infra`

Depreciation schedules, business-use allocation, disposal handling, and inventory-to-ledger COGS transitions are domain rules rather than transport details. They belong in Codices. The same is true for security-scoped operational policy: resolving an action to `r`, `x`, or `d`; checking an allow-list and certification; requiring live acknowledgement; and failing closed are executable policy, so Portainer container control is also a Codex.

#### Why Codices exist

- Deep domain logic can be tested and versioned independently.
- Several callers can reuse the same authoritative rules.
- The public DojoTool remains understandable.
- Transport changes do not disturb domain calculations.
- A Codex may provide a related Stack for derived knowledge without exposing the Codex directly.
- SP1 can safely enable powerful DojoTool capabilities without converting them into AdminTools.

#### Codex best practices

1. Keep calculations deterministic and evidence-backed.
2. Declare assumptions, limitations, and confidence or completeness where relevant.
3. Separate backend transport into a Root or Sutra.
4. Keep conversational wording and routing in the public DojoTool.
5. Make state transitions idempotent where practical.
6. Preserve source identifiers needed for audit and reconciliation.
7. Add a Stack provider when derived domain data should be available through Lens.
8. For security-scoped actions, classify each mode under `rwxda`, resolve caller identity through the SP1 chokepoint, enforce certification and ACL requirements, and fail closed.
9. Require live human acknowledgement where the action policy calls for it. An acknowledgement supplements scoped authority; it does not replace identity or certification.

#### Use a Codex when

- the component embodies a recognizable body of domain expertise;
- business rules are substantial, reusable, or independently versionable;
- correctness requires calculations or multi-step state transitions;
- burying the logic inside a generic DojoTool would make that tool difficult to review.
- the capability needs reusable, security-scoped action policy under SP1.

### 3.7.1 SP1 security-scoped capability pattern

SP1 separates **what a tool can do** from **which resolved principal may perform a particular action on a particular target now**. This allows an MCP-exposed DojoTool to offer consequential operations safely while keeping its configuration and transport surfaces internal.

The current pattern is RBAC-like, but it is not full role-based access control. A household role does not directly grant a broad permission such as "Portainer administrator." Authorization is composed from several narrower facts:

| Policy dimension | Portainer implementation |
|---|---|
| Principal | `resolve_caller_identity` resolves the acting identity |
| Capability | The resolved identity carries the named `infra_container_control` certification |
| Capability strength | The certification has a numeric level compared with the action's configured minimum |
| Action class | `rwxda`; current container writes use `x` or `d` |
| Resource scope | Target container must match the admin-managed `controllable_containers` allow-list |
| Contextual approval | `d` actions require a live household-admin acknowledgement |
| System policy | Simulated identity is allowed or blocked centrally by the SP1 policy switch |
| Audit evidence | Denials and executions emit events with their specific reason |

This is best described as **security-scoped capability authorization**. It combines capability-based access control, action and resource attributes, policy state, and contextual human approval. Household roles may help resolve principals or approval targets, but roles are not the sole authorization primitive.

The canonical composition is:

```text
Friday
  -> DojoTool semantic mode
    -> Codex action policy
      -> SP1 identity and certification chokepoint
      -> configured ACL / allow-list
      -> optional live human acknowledgement
      -> Root transport
```

Portainer container control is the reference example:

- `containers_list` and `container_get` require the Codex to be configured but do not require a control certification;
- `container_restart` and `container_start` are `x` actions and default to certification level 2;
- `container_stop` and `container_remove` are `d` actions and default to certification level 3;
- every `x` or `d` target must resolve to exactly one live container;
- every `x` or `d` target must match the admin-managed resource allow-list;
- `d` actions additionally require a live household-admin acknowledgement;
- ambiguity, absent configuration, disallowed target, blocked identity policy, insufficient certification, failed acknowledgement dispatch, decline, and timeout all deny the action;
- every denial carries a machine-readable reason and emits an audit event;
- configuration and allow-list changes remain in the non-exposed AdminTool;
- raw Portainer transport remains in the non-exposed Root.

#### Current implementation boundary

Chef contains the complete authorization call path but does not yet cryptographically bind an MCP session to a specific persona cabinet. Therefore:

- `caller_token` is threaded through the path and inspected by the Authentik Root, but Authentik is presently a declared simulation stub;
- `caller_id` is free-text audit metadata and must never be trusted as an identity claim;
- `resolve_caller_identity` currently resolves calls as the default agent;
- authorization therefore means "the default agent is certified, or the action is denied";
- `sim_mode_allowed` is a central policy gate. If simulated resolution is disallowed, identity resolution withholds the principal and the action fails closed;
- when live SP1 identity binding replaces the stub, callers inherit real per-principal evaluation through the same chokepoint without each Codex implementing its own authentication system.

Do not describe the current implementation as full RBAC. Do not build a per-`caller_id` permission map around unauthenticated strings. Both would claim a security boundary the runtime cannot yet prove.

#### Gate order

Security-scoped Codices should use a deterministic, fail-closed gate sequence:

1. Confirm the feature is configured.
2. Resolve the requested resource and require an unambiguous target.
3. Confirm the resource is within the configured allow-list or scope.
4. Map the requested mode to its `rwxda` action class and minimum certification level.
5. Resolve caller identity through the shared SP1 chokepoint and request the named certification check in the same call.
6. Deny when central identity policy is blocked.
7. Deny when the certification is absent or below the required level.
8. Obtain contextual human acknowledgement when the action policy requires it.
9. Execute through the internal Root.
10. Emit success or denial evidence with a specific reason.

Checking identity and then reading a caller-selected certification store separately is forbidden. The shared resolver must check the certification against the identity it resolved.

The operation's subject matter may be administrative. Its architectural class remains DojoTool plus Codex because it is a supported, policy-scoped agent capability. AdminTool is reserved for changing the policy or ZenOS control plane behind that capability.

### 3.8 Kung Fu Components

**Form:** a KFC definition returned through `mode=kfc_manifest` and mounted as a Dojo drawer
**Typical caller:** manifest bootstrap, Scheduler, Ninja Summarizer, and SuperSummary pipeline
**Exposure:** context contract rather than direct capability

A Kung Fu Component defines how a coherent slice of the home becomes scheduled cognition. It identifies the component, scope label, instructions, seed tool, trigger subscriptions, thresholds, cooldowns, delay, and security requirements.

A KFC may use two input paths:

- a seed call that returns purpose-built structured context;
- an index/label path used as a fallback or direct entity surface.

Taskmaster demonstrates both. Its `briefing` seed produces scored task context. Its `taskmaster` label supplies an index-only fallback when the seed is unavailable.

#### KFC best practices

1. The tool that owns the domain should self-register its KFC through `mode=kfc_manifest`.
2. Treat the KFC drawer as the authoritative component specification.
3. Treat the label as the scope and HyperIndex as the data layer.
4. Keep component instructions factual, bounded, and explicit about fallback behavior.
5. Define trigger subscriptions according to actual information freshness needs.
6. Use cooldown, drift threshold, and delay to prevent noisy or synchronized dispatch.
7. State safety and interpretation rules in both executable controls and KFC instructions when defense in depth is appropriate.
8. Do not create a standalone KFC file when its owning tool can return the contract from the same package file.

### 3.9 Boot Orchestration (the Flynn/Stepgate pattern)

**Form:** a health-sensor-triggered automation sequence ("Stepgate Sentinel") plus a small set of companion scripts it calls
**Typical prefix:** `flynn_*` (deliberately outside every other class's prefix convention — see rationale below)
**Stripe:** Outside the sweep, same reasoning as AdminTools. Boot orchestration doesn't get discovered by `bootstrap_stacks`/`bootstrap_kfc`; it *is* the thing that makes the rest of the sweep possible.
**Typical caller:** Home Assistant `homeassistant: start`, health-sensor state-change triggers, and specific `zen_event` kinds (`warmup_expired`, `cabinet_mounted`, `cabinet_dismounted`)
**Default exposure:** not MCP-exposed; not a callable capability at all in the DojoTool sense

A Boot Orchestrator is the class of component responsible for bringing ZenOS from "Home Assistant just started, nothing is guaranteed to exist yet" to "every Stripe is healthy, identity is bootstrapped, the agent is ready" — with zero human intervention on a correctly-configured install, and a clear, actionable notification when human intervention really is required.

This is deliberately not a DojoTool, AdminTool, Root, Sutra, Stack, or Codex. It doesn't accept a semantic request from an agent (DojoTool), it doesn't perform one operator-invoked structural change (AdminTool), and it isn't a backend adapter (Root/Sutra/Stack). It is a **stateful gate sequence**: an ordered series of preconditions, each of which either passes (advance), self-repairs (call a companion script, then re-evaluate), or hard-stops with an actionable notification. Flynn (`flynn.yaml`, `flynn_oobe.yaml`) is currently the only component in this class. Do not build a second one without a clear reason — this pattern exists to have exactly one thing gate-sequencing platform bootstrap, not several competing sequencers.

#### Why this needs to be a named, documented class

Every one of a batch of real production bugs found in one week of dedicated fresh-install testing was a violation of a rule that existed only as tribal knowledge, not as a written standard:

- A health sensor recomputed on a `time_pattern` clock and caught a multi-step write mid-flight, misclassifying a cabinet that was simply not finished being stamped yet as broken.
- A gate's condition treated a rolled-up enum value (`warn`) as universally safe, when that enum silently collapsed several genuinely different underlying conditions — some safe, one not.
- A gate's condition trusted a live `states(entity_id)` read at boot, when cabinet state is well-established as untrustworthy in the first several seconds after Home Assistant starts (this is also why `flynn_initialize_cabinets` has its own independent GUID gate — state can lie while `VolumeInfo` is real, and no boot-time code may ever treat a bare state read as authoritative over that).
- An early-exit optimization checked only health-sensor rollups and had no awareness that a *different* gate further down had its own independent re-entry condition — so on any already-healthy system, the optimization fired every single cycle and permanently prevented that later gate from ever being reached.
- A completion event that a health sensor needed in order to recompute was simply never fired by three of the four write paths that changed the state the sensor reported on.

None of these were exotic. Each was a small, locally-reasonable-looking piece of code that violated a rule the author didn't know existed because the rule had never been written down. That is exactly the failure mode this taxonomy exists to prevent.

#### Boot orchestration rules

1. **Gates are ordered and sequential.** Each gate checks exactly one precondition. A gate that passes advances to the next; a gate that fails either self-repairs (calls a companion script, then stops and waits for the resulting state change to re-trigger the sequence) or hard-stops with an actionable, specific notification. Never combine two unrelated preconditions into one gate's condition.
2. **Never trust a live `states(entity_id)` read as authoritative at boot.** Recorder-restored state can be transiently wrong in the first moments after `homeassistant: start`. Any write-worthy decision made from live state needs an independent, harder-to-fool confirmation (a GUID/identity field actually present in the data, not just an enum saying it should be) before anything destructive or state-changing happens.
3. **Distinguish "genuinely safe / not yet bootstrapped" from "actively broken" at the finest grain the data supports**, never at the grain of a rolled-up summary enum alone. If a health sensor's own state value is a priority-collapsed rollup of several distinct underlying conditions, a boot gate consuming it must inspect the underlying per-item detail (an attribute map, a missing-items list) before deciding to hard-stop, not trust the top-level enum alone.
4. **An early-exit / short-circuit optimization must know about every later gate's own re-entry conditions**, not just the health-sensor rollups it was originally written against. When a later gate grows an additional way to re-enter (a durable completion flag, a different trigger), the early-exit must be updated to require that same condition, or it will silently prevent the later gate from ever being reached on exactly the systems most likely to hit it first (already-healthy, long-running systems).
5. **A trigger-based health sensor is only as fresh as the events that wake it up.** If a sensor is deliberately event-triggered rather than polled (see §5.2 for why), every write path capable of changing what that sensor reports on must fire the triggering event on completion — audited exhaustively, not assumed. A write path that "obviously" should trigger a refresh and doesn't is a silent staleness bug with no error anywhere.
6. **Durable one-time-completion state belongs in a cabinet drawer, not a volatile sensor value or an `input_text` helper reused for a different purpose.** A live health sensor's current value answers "is this healthy right now" — it cannot answer "has this genuinely succeeded at least once, ever," because a system can drift from `critical` to `warn` to `ok` without a bootstrap step in between ever having actually run. For that class of fact, write a small, explicit drawer (e.g. `flynn_bootstrap_state: {completed, completed_at, gate_version}`) on the relevant cabinet, and — critically — only stamp it after a fresh read-back confirms the write it depends on genuinely landed. Never stamp a completion flag optimistically just because a sequence reached its end without raising an error; a silently-rejected sub-write (see rule 8) should not read as success.
7. **Backfilling a new durable flag onto existing installs is not free — verify it actually reaches a healthy system, not just a broken one.** A broken test system can accidentally be exempt from an early-exit path in a way that makes a backfill look like it works when it has in fact only ever reached systems that were already failing some unrelated check. Test the backfill specifically against a long-running, fully-healthy system before shipping it.
8. **A `force_action: false` (or equivalent "soft" write flag) does not mean what a comment says it means unless the underlying write path is re-verified.** A reserved/underscore-prefixed key, a protected drawer, or any other write path with its own unconditional guard will silently reject a "soft" write regardless of intent — this can go unnoticed indefinitely because the write path doesn't raise an error, it just no-ops. If a comment claims a specific conditional behavior ("only writes if empty"), verify that behavior against the actual write path's guard logic, not just the comment's word.

## 4. Stripes

Stripes describe dependency order for manifest discovery, health sweeps, and preferred startup state. They do not describe user privilege.

| Stripe | Meaning | Typical contents |
|---:|---|---|
| 0 | Foundational storage primitive | `zen_sutra_filecabinet` |
| 1 | ZenOS kernel and system services | core services, system health, Kata garbage collection |
| 2 | Integration, provider, and domain layer | Roots, Sutras, Stacks, Codices |
| 3 | User-facing semantic capabilities | ordinary DojoTools and their KFC-facing facades |
| *(outside the sweep)* | Not manifest-discovered at all | AdminTools, the Boot Orchestrator (§3.9) |

Dependency direction should normally move upward:

```text
Stripe 0 -> Stripe 1 -> Stripe 2 -> Stripe 3
```

Higher Stripes may depend on lower Stripes. A lower Stripe should not require a higher Stripe to become healthy. Optional calls upward must fail soft and must not create a startup cycle.

### Stripe selection rules

- Choose Stripe 0 only for a primitive whose absence prevents the rest of ZenOS from resolving core state.
- Choose Stripe 1 for OS services required before provider and public capability checks.
- Choose Stripe 2 for adapters, external integrations, domain engines, and Lens providers.
- Choose Stripe 3 for normal agent-facing capabilities.
- Declare the Stripe in the manifest. Do not rely solely on prefix inference.
- If a component is not part of the manifest bootstrap sweep at all (AdminTools, the Boot Orchestrator), say so explicitly rather than guessing a Stripe number for it — see §3.3 and §3.9.

## 5. Scripts, Automations, KFCs, and Health Sensors

Use each runtime form for its proper responsibility.

| Form | Responsibility |
|---|---|
| Script | Implements a callable capability or internal operation |
| Automation | Decides when a capability runs in response to events or schedules |
| KFC | Defines what contextual component should be assembled and interpreted |
| Health sensor (template/trigger) | Derives continuously readable system state used to gate other components — see §5.2, this is not a throwaway concern |
| REST command | Performs a narrow transport operation, normally owned by a Root or Sutra |

Keep timing out of business logic. An automation should call a reusable script rather than duplicate its implementation. A KFC should identify its seed and subscriptions rather than embedding the full orchestration pipeline.

### 5.1 Health sensors are load-bearing infrastructure, not passive reporting

A ZenOS health sensor (`zen_label_health`, `zen_cabinet_health`, `zen_monastery_health`, and the rest of the stack) is not just a dashboard readout. Boot orchestration, KFC scheduling, and operator-facing diagnostics all key real decisions off these values. Every rule in this subsection exists because violating it produced a real, silent bug.

### 5.2 Health sensor design rules

1. **Prefer event-driven triggers over `time_pattern` clocks for anything that gates destructive or state-changing decisions.** A periodic tick has no awareness of whether a multi-step write elsewhere is mid-flight; it can observe a component in a transient, partially-written state and misclassify it as broken. Trigger the sensor on `homeassistant: start`, a dedicated internal tick event, and an explicit "something relevant just changed" event instead. A time-based poll is acceptable for sensors that only ever *report*, never *gate*.
2. **Every write path capable of changing what an event-triggered sensor reports on must fire that sensor's triggering event on completion.** Audit this exhaustively per write path — "should probably trigger a refresh" is not the same as verifying it does. A sensor that's correctly trigger-based but never actually triggered is worse than a polled one: it looks fresh (the last-updated timestamp is recent) while reporting stale data.
3. **If the underlying write and the sensor's recompute can race** (the write's own completion event fires before the data the write changed is actually queryable elsewhere — e.g. a registry lookup that hasn't caught up to a tag that was just applied), a single immediate refresh event is not sufficient. Add a second, delayed refresh. Do not "fix" this by switching to a `states(entity_id)` read or a `trigger: state` on the underlying entity — see rule 5.
4. **Do not collapse genuinely distinct conditions into one summary enum value that consumers can't disambiguate**, especially when some of those conditions are safe (not yet bootstrapped) and others are not (actively broken). If a rollup must exist for display purposes, also expose the underlying per-item detail as an attribute (a missing-items list, a per-slot state map) so a consumer that needs to make a real decision can inspect the actual condition instead of trusting the collapsed label. A downstream gate that trusts `warn` as universally safe because "it's usually fine" will eventually be wrong in exactly the case that matters.
5. **Never gate a destructive or write-triggering decision on a live `states(entity_id)` read at boot.** Recorder-restored state is untrustworthy in the first moments after HA starts, and is not the same guarantee as an authoritative identity/content field actually being present in the underlying data. This is a hard rule, not a style preference — violating it is how cabinet data gets destroyed. See the boot-orchestration GUID-gate pattern in §3.9.
6. **A diagnostic check that itself depends on a network or transport call must distinguish "checked, and it's genuinely absent" from "the check itself failed to complete."** Trusting a response body's shape alone (e.g. "it parsed as JSON, so it must be a valid result") without first confirming the transport call actually succeeded (HTTP status, timeout, auth failure) produces a confident false negative indistinguishable from the real thing being absent. Surface the check's own success/failure as an explicit field separate from the check's answer.

## 6. Packaging and Co-location Standard

ZenOS-AI prefers **one coherent component package per file**.

If several scripts, automations, REST commands, manifests, Stack adapters, Roots, Sutras, Codices, and KFC definitions describe one deployable capability, keep them together in the same YAML file whenever the result remains reasonably reviewable.

This convention is intentional:

- copying one file installs one coherent capability;
- the KFC travels with the code that understands and seeds it;
- pluggable knowledge can be distributed without hidden companion files;
- manifests, configuration, health checks, and implementation evolve together;
- removal does not leave orphaned automations or component definitions;
- developers can understand the complete vertical slice in one place.

### Mandatory KFC co-location preference

Every KFC that can reasonably share a file with its owning capability should do so. The preferred pattern is:

```yaml
rest_command:          # only when needed
  ...

script:
  zen_root_example:    # optional internal transport
    ...

  zen_sutra_example:   # optional internal operations
    ...

  zen_stack_example:   # optional Lens provider
    ...

  zen_dojotools_example:
    # mode=tool_manifest
    # mode=kfc_manifest
    # public capability modes
    ...

automation:
  ...                  # only component-specific triggers not owned by shared Scheduler
```

The owning DojoTool should normally emit its KFC definition through `mode=kfc_manifest`. Avoid separate static KFC manifests when self-registration is possible.

### Small durable runtime state: use a cabinet drawer, not a new helper entity

When a component needs to persist a small, structured piece of runtime state — a completion flag, an active-notification marker, a last-verified timestamp — that doesn't rise to the level of a full DojoTool-managed resource, prefer a dedicated drawer on the relevant cabinet over creating a new `input_text`/`input_boolean` helper or relying on a Home Assistant domain's own state machine.

This matters for two concrete reasons:

- **Helper sprawl.** Every ad hoc helper is one more thing to declare, document, and expose in `configuration.yaml`, for state that's really just a structured value belonging to a cabinet ZenOS already manages.
- **Some domains' state machines are not what they appear to be.** `persistent_notification` entities, for example, do not reliably reflect into the template-readable state machine on every Home Assistant version — a `states('persistent_notification.' ~ id) == 'notifying'` check can silently and permanently return the wrong answer with no error anywhere. A cabinet drawer, read through the standard FileCabinet path, does not have this failure mode.

`flynn_active_notification` (which notification, if any, Flynn should acknowledge in its opening) and `flynn_bootstrap_state` (has first-bootstrap genuinely completed) are the reference examples — both are plain drawers on the default household cabinet, written alongside the "real" side effect (the actual HA notification call; the actual bootstrap writes) rather than replacing it, and read back through the normal cabinet-drawer path rather than through the domain's own state machine.

### Folder exception for large components

A single file becomes inadvisable when code volume or complexity impairs review, testing, ownership, or safe maintenance. In that case, keep every related file in one dedicated component folder.

Firefly III is the reference pattern. Its Root, DojoTool/Stack surface, finance Codices, and supporting resources remain grouped as one plugin even though placing the entire implementation in one YAML file would be unwieldy.

Use a component folder when one or more of the following apply:

- the file is large enough that navigation obscures architectural boundaries;
- multiple substantial Codices have independent versions or tests;
- transport, domain logic, and provider logic each require significant implementation;
- generated or service-specific supporting files would pollute the main package;
- reviewers cannot safely reason about changes in one monolithic file.

Folder rules:

1. One folder represents one deployable integration or coherent domain capability.
2. Keep all related Roots, Sutras, Stacks, Codices, KFC providers, automations, and documentation inside that folder.
3. Provide an obvious primary package file or README that explains the dependency and exposure graph.
4. Do not scatter companion files across generic `dojotools`, `plugins`, and `automations` directories merely because their Home Assistant runtime forms differ.
5. Preserve a clean copy/install boundary for the whole folder.
6. Avoid splitting a component solely to make files aesthetically short.

### Packaging decision

| Situation | Package shape |
|---|---|
| One DojoTool with a KFC and modest helpers | One YAML file |
| Plugin with Root, facade, Stack, and manageable implementation | One YAML file |
| Large integration with substantial transport and several domain modules | One dedicated folder |
| Shared OS primitive used by unrelated components | Core file in the appropriate ZenOS system package |
| One-time migration or historic repair | `maint/`, never bundled into the public capability surface |

## 7. Exposure and Security Rules

### Default exposure matrix

| Class | Default agent exposure |
|---|---|
| DojoTool | Expose when intended as an agent capability |
| SystemTool | Expose only safe facade modes |
| AdminTool | Do not expose; configure through operator/Flynn control-plane flows |
| Root | Never expose |
| Sutra | Never expose |
| Stack | Route through Library |
| Codex | Route through a DojoTool or Stack; may enforce SP1-scoped actions internally |
| KFC | Mounted as context, not directly exposed |
| Boot Orchestrator | Never expose — not a callable capability at all |
| Maintenance script | Never expose |

Exposure must be declared truthfully in the manifest and reflected in installation documentation. A component named `zen_dojotools_*` may still be internal when it exists only as a provider implementation. Naming is useful for discovery; policy remains explicit.

### Consequential writes

For writes with meaningful impact:

- validate the target;
- state the anticipated impact;
- provide preview where feasible;
- require `confirm_action` for destructive or structural changes;
- enforce caller authority;
- return a useful audit trace;
- provide fallback or recovery guidance.

## 8. Manifest Standard

Every Stripe 1 or higher component should implement `tool_manifest` using the shared `MF.tool_manifest()` macro.

At minimum, declare:

- canonical tool name;
- display name;
- tier/class;
- version;
- health;
- Stripe;
- exposure;
- consumed and returned semantic types;
- required and optional labels;
- risk and security requirements when relevant;
- impact, prerequisites, preferred state, limitations, and fallback behavior where useful.

KFC-owning tools should also implement `kfc_manifest`. Lens providers should declare provider registration information, anchors, and evidence types.

The implementation currently contains both singular and plural tier values in older and newer surfaces. New development should follow the repository's active manifest conventions consistently within the target release, and a future schema migration should normalize vocabulary centrally rather than through isolated file edits.

## 9. Decision Guide

Start with the caller and responsibility.

1. **Should Friday call it directly?**
   Build a DojoTool facade.

2. **Does it change ZenOS configuration, capability ACLs, schemas, manifests, or recovery state?**
   Build an AdminTool and keep it outside the ordinary agent surface.

3. **Is it the lowest-level reusable transport to an external service?**
   Build a Root.

4. **Does it normalize useful backend operations above transport?**
   Build a Sutra.

5. **Does it make knowledge retrievable through Library and the Lens Bus?**
   Build a Stack provider.

6. **Does it embody substantial reusable domain expertise or security-scoped action policy?**
   Build a Codex.

7. **Should the subsystem contribute scheduled context or summaries?**
   Add a self-registered KFC to the owning tool.

8. **Does an event or schedule decide when it runs?**
   Use an automation or the shared Scheduler to invoke the script/KFC.

9. **Is this a precondition that must be true before ZenOS itself is usable — labels exist, cabinets are initialized, identity is bootstrapped?**
   This belongs in the Boot Orchestrator's gate sequence (§3.9), not a new automation. Extend Flynn's existing sequence rather than building a second, competing bootstrap path.

10. **Can the complete capability remain understandable in one file?**
    Keep it in one file, including its KFC and related components.

11. **Would one file become unsafe or unreasonable to maintain?**
    Create one dedicated component folder and keep the complete vertical slice together.

## 10. Common Compositions

### Simple agent capability

```text
Friday -> DojoTool -> Home Assistant service
```

Use for a bounded capability where Home Assistant already supplies a safe service.

### External service integration

```text
Friday -> DojoTool -> Sutra -> external service
```

Use when backend-specific operations deserve an internal adapter.

### Reusable transport integration

```text
Friday -> DojoTool -> Sutra/Codex -> Root -> external service
```

Use when several modules share authentication, endpoints, and response filtering.

### Security-scoped operational capability

```text
Friday -> DojoTool -> Codex -> SP1/ACL/ack gates -> Root -> controlled system
```

Use when an agent should perform a powerful action under bounded, auditable authority. Keep policy configuration in a non-exposed AdminTool and transport in a non-exposed Root.

### Knowledge provider

```text
Friday -> Library -> Stack -> Sutra/DojoTool/Root -> knowledge source
```

Use for documents, tickets, media, finance evidence, or another searchable provider.

### Scheduled cognition

```text
Event -> Scheduler -> KFC -> seed DojoTool -> Ninja -> Kata -> SuperSummary -> Friday
```

Use when the system should proactively assemble and summarize context.

### Boot orchestration

```text
homeassistant:start / health-sensor state change
  -> Stepgate gate sequence (Flynn)
    -> self-repair script (label assign, cabinet init, content bootstrap)
    -> health-sensor refresh event
    -> re-trigger, re-evaluate next gate
```

Use only for the one platform bootstrap sequence. Do not build a second gate sequence for a subsystem-specific startup concern — extend the existing one, or use an ordinary automation if the concern doesn't actually gate platform-wide readiness.

### Large domain plugin

```text
plugin folder/
  primary package and public facade
  Root transport
  one or more Codices
  Stack provider
  KFC definitions
  component documentation
```

Use Firefly III as the reference when one-file packaging would make the component harder to reason about.

## 11. Anti-patterns

- Exposing a Root or Sutra because it is convenient during development.
- Mirroring every endpoint of an external API as DojoTool modes.
- Placing domain calculations inside a REST broker.
- Treating every high-impact action as an AdminTool instead of designing a scoped DojoTool/Codex capability.
- Giving Friday a generic AdminTool or arbitrary service-call escape hatch.
- Creating a new direct MCP tool for every knowledge provider instead of using Library and Stacks.
- Hardcoding KFC entity lists instead of using labels and HyperIndex.
- Storing a KFC far from the tool that owns its seed and interpretation rules.
- Splitting one modest component across several directories based on runtime form.
- Building a single enormous file after complexity has made review unsafe.
- Inferring exposure or authority from Stripe alone.
- Creating automations that duplicate script business logic.
- Returning raw backend payloads that may contain secrets or irrelevant sensitive data.
- Building a second gate-sequencing bootstrap automation instead of extending the existing Boot Orchestrator.
- Polling a health sensor on a `time_pattern` clock when it gates a destructive or write-triggering decision.
- Trusting a rolled-up health-sensor enum (`warn`, `ok`) for a consequential decision without checking whether the underlying condition is actually the safe or unsafe member of that rollup.
- Gating a boot-time write decision on a live `states(entity_id)` read instead of an authoritative identity/content field.
- Adding a write path to a component with a trigger-based health sensor without auditing whether that path fires the sensor's refresh event.
- Reaching for a new `input_text`/`input_boolean` helper, or trusting a domain's own state machine, for small durable state that a cabinet drawer would serve more reliably.

## 12. Review Checklist

Before accepting a new component, reviewers should confirm:

- [ ] The public caller and intended authority are clear.
- [ ] The class and filename prefix match the component's responsibility.
- [ ] The Stripe matches its dependency position, or the component explicitly documents that it sits outside the manifest sweep.
- [ ] Agent exposure is explicit and minimal.
- [ ] Structural operations are isolated in an AdminTool.
- [ ] Transport and secrets are isolated in an internal Root or Sutra when warranted.
- [ ] Domain rules are separated into a Codex when complexity justifies it.
- [ ] Security-scoped capabilities use the shared SP1 identity/certification chokepoint rather than trusting caller-supplied identity fields.
- [ ] Action class, certification level, resource scope, contextual acknowledgement, and denial reasons are explicit where applicable.
- [ ] Knowledge access routes through Library and a Stack provider, and a Stack's `tool_manifest` self-describes rather than falling through to its owning tool's identity.
- [ ] Any KFC self-registers from the owning component where practical.
- [ ] The KFC, tool, helpers, and component-specific automation are co-located.
- [ ] The package is one file unless complexity clearly justifies one dedicated folder.
- [ ] Optional integrations fail soft and preserve baseline behavior.
- [ ] Writes validate targets and enforce appropriate confirmation and authority.
- [ ] Responses are structured, bounded, and free of secrets.
- [ ] Manifest metadata accurately describes health, exposure, dependencies, risk, and evidence.
- [ ] If this touches boot orchestration: every write-completion path fires the relevant health-sensor refresh event; no boot-time write decision trusts a live `states(entity_id)` read alone; any early-exit optimization accounts for every gate's re-entry conditions, not just health-sensor rollups.
- [ ] If this adds or changes a health sensor: it is event-triggered rather than clock-polled if it gates any consequential decision; a rolled-up enum value exposes enough underlying detail for a real consumer to disambiguate safe-vs-broken; a diagnostic check distinguishes "checked, absent" from "check itself failed."

## 13. Short Form

- **DojoTool:** Friday's supported capability.
- **SystemTool:** a controlled ZenOS runtime capability.
- **AdminTool:** privileged setup, policy, repair, or recovery authority.
- **Root:** lowest reusable backend transport.
- **Sutra:** internal backend operations adapter.
- **Stack:** Lens Bus knowledge provider.
- **Codex:** executable domain expertise or security-scoped action policy.
- **KFC:** scheduled context contract.
- **Boot Orchestrator:** the one gate-sequenced bootstrap automation (Flynn) that brings ZenOS from cold start to ready — never trust live state at boot, never poll a sensor that gates a write, never let an early-exit forget a later gate's re-entry condition.
- **Automation:** decides when work begins.
- **Health sensor:** load-bearing infrastructure other components gate real decisions on — event-triggered when it gates anything consequential, never collapsing safe-vs-broken into one opaque enum value.
- **Stripe:** dependency order, never privilege.
- **Package:** one coherent deployable capability, preferably one file; one dedicated folder when complexity demands it.
