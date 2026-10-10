# 24. Developer Standards

Every chapter so far has described what ZenOS is. This one describes the rules for adding to it without breaking the argument the rest of the book makes: that the graph stays readable, the contracts stay truthful, and authority stays outside the model. A new component that follows these rules extends the twin and the reachable graph. One that does not quietly erodes both.

These standards were written down after the code, not before it. Home Assistant's templating is a strange hybrid that language models routinely get wrong, and as more of ZenOS came to be written with AI help, the rules had to be explicit: HALMark for the code itself, and this chapter for the shape of the components. The build loop that now produces most changes, with one agent implementing, one testing on a separate system, and a human approving every release, only works because the standard is written down.

This chapter states the architectural rules. The full working standard, with the rationale, decision guide, common compositions, and anti-patterns for every class, is [the developer standards document](../developer_standards.md).

## 24.1 Five axes, never collapsed

Every component is described on five independent axes:

| Axis | Question | Values |
|---|---|---|
| Class | What architectural job does it do? | DojoTool, SystemTool, AdminTool, Root, Sutra, Stack, Codex, Boot Orchestrator |
| Stripe | What must exist before it can start? | 0, 1, 2, 3, or outside the sweep |
| Exposure | May an agent call it? | Agent-exposed, internal, operator-only |
| Cognitive role | Does it supply recurring context? | KFC provider, Lens provider, none |
| Runtime form | How does Home Assistant run it? | Script, automation, template or health sensor, REST command |

The axes must not collapse into each other. A Stripe grants no authority. A filename prefix does not replace an explicit exposure decision. A KFC is a context contract, not a security class. Being a script does not make something a DojoTool.

## 24.2 Classes

| Class | Is | Agent exposure |
|---|---|---|
| DojoTool | Friday's supported capability: semantic modes, label-resolved targets | Exposed when it is meant as a capability |
| SystemTool | A controlled runtime capability: health, logs, reloads | Safe modes only |
| AdminTool | Setup, policy, repair, recovery, certification | Never |
| Root | The lowest reusable transport to an external system | Never |
| Sutra | An internal adapter for backend operations | Never |
| Stack | A Lens Bus knowledge provider | Through Library |
| Codex | Executable domain expertise or security-scoped policy | Through a DojoTool or Stack |
| Boot Orchestrator | The one gate-sequenced bootstrap (Flynn) | Not callable at all |

A DojoTool may depend on several internal classes while remaining the only public entry point. That is the point of the classes: structural operations sit in AdminTools, transport and secrets sit in Roots and Sutras, and domain rules sit in Codices when their complexity justifies it.

## 24.3 Stripes

Stripes are dependency order for manifest discovery, health sweeps, and startup. They are never privilege.

| Stripe | Holds |
|---:|---|
| 0 | The foundational storage primitive (`zen_sutra_filecabinet`) |
| 1 | Kernel and system services |
| 2 | Integrations, providers, domain engines: Roots, Sutras, Stacks, Codices |
| 3 | Agent-facing capabilities: ordinary DojoTools |
| Outside the sweep | AdminTools and the Boot Orchestrator |

Dependencies point upward. A higher Stripe may depend on a lower one; a lower Stripe must never need a higher one to become healthy, and any optional call upward fails soft. The Stripe is declared in the manifest, not inferred from a prefix.

## 24.4 Runtime forms and health sensors

A script implements a capability. An automation decides when it runs. A KFC defines what context is assembled. A REST command is narrow transport owned by a Root or Sutra. Timing stays out of business logic: an automation calls a reusable script instead of duplicating it.

Health sensors are load-bearing. Boot, scheduling, and diagnostics make real decisions on them (Chapter 23), so they follow six rules:

1. Anything that gates a state-changing decision is event-triggered, not clock-polled.
2. Every write path that changes what an event-triggered sensor reports fires its refresh event.
3. Where a write and its recompute can race, add a delayed second refresh; never fall back to a live state read.
4. Never collapse safe and unsafe conditions into one enum value without exposing the detail a consumer needs to tell them apart.
5. Never gate a destructive decision on a live state read at boot.
6. A check that depends on a transport call reports whether the check itself succeeded, separately from its answer.

## 24.5 Exposure and consequential writes

Exposure is declared truthfully in the manifest, and installation documentation follows it. A `zen_dojotools_*` name does not make a tool agent-facing; the manifest decides. The recommended household configuration exposes the agent-facing tools and zero entities (Chapter 10).

A write with real impact validates its target, states the anticipated impact, previews where it can, requires `confirm_action` for destructive or structural changes, enforces caller authority, returns an audit trace, and says how to recover.

## 24.6 The tool contract

The contracts in Chapter 12 are rules for every new or revised DojoTool, SystemTool, and AdminTool:

* **Manifest.** Every tool implements `mode=tool_manifest` through the shared `tool_manifest()` macro, declaring at least its name, class, version, health, Stripe, exposure, labels, `dependencies`, and `inference`. The manifest version is the script's canonical version; every other mention follows it, one version per script, never a bundle number.
* **Envelope and single exit.** Build one response and return it once, through `envelope()`. A denial is set early and every later branch checks it. A caller of another tool unwraps `.result` before reading fields.
* **Help.** `mode=help` returns `{status: help, message}` and passes `audit_help`.
* **Certification.** Resolve the caller through `resolve_caller_identity` and read it with `resolve_identity_fields()`. Pass `required_cert` and a level, using dotted `zenos.*` names where a hierarchy applies. Treat `scope_decision` as authoritative: `deny` blocks, `allow` exempts the target from the per-call acknowledgement, `default` falls to the tool's own rule. Refuse through `cert_denial()`. Declare every certificate the tool enforces as `certs_required`, so the catalog builds itself.
* **Human in the loop.** An action that needs a person calls `request_live_ack` and proceeds only on `approved`.

## 24.7 Packaging

One coherent capability per file. The scripts, automations, REST commands, KFC definitions, Stack adapters, Roots, Sutras, and Codices that make up one capability live together, so copying one file installs it and removing one file leaves nothing orphaned. A KFC lives with the code that seeds it and self-registers from it. A capability too large for one reviewable file gets one dedicated folder, not a scatter of files. Small durable runtime state goes in a cabinet drawer, not a new helper entity. One-time migrations live in `maint/` and never ship in the public surface.

## 24.8 Review checklist

Before a component is accepted:

* Its caller, authority, class, prefix, and Stripe agree with each other.
* Exposure is explicit, minimal, and matches the manifest.
* Structural operations are in an AdminTool; transport and secrets in a Root or Sutra.
* Security-scoped actions use the identity chokepoint and never trust caller-supplied identity.
* Knowledge access goes through Library and a Stack that describes itself.
* Its KFC self-registers and is co-located with it.
* Optional integrations fail soft.
* Writes validate targets and enforce confirmation and authority.
* Responses are structured, bounded, and free of secrets.
* The manifest is accurate and its version is canonical.
* It returns through `envelope()` with a single exit, and unwraps the tools it calls.
* `mode=help` conforms, and cert-gated modes declare `certs_required`, honor `deny`, and refuse through `cert_denial()`.
* Any boot-path or health-sensor change follows the six rules in 24.4.

The full checklist, with the boot-orchestration and health-sensor detail behind each item, is in [the developer standards document](../developer_standards.md).

The next chapter is the standard applied: Room Manager v3, where labels, inference, live state, effects, and authorization all meet in one component.

<!-- where -->
## 24.9 Where to look

Every claim in this chapter can be checked in the code. These are the places to start.

* The manifest macro: [`zenos_manifest.jinja`](../../../custom_templates/zenos_ai/zenos_manifest.jinja) (`macro tool_manifest`). Docs: [zenos_manifest_jinja.md](../custom_templates/zenos_manifest_jinja.md).
* The envelope, identity field reader, denial shape, and scope check: [`zen_os_1.jinja`](../../../custom_templates/zenos_ai/zen_os_1.jinja) (`macro envelope`, `macro resolve_identity_fields`, `macro cert_denial`, `macro cert_scope_check`). Docs: [zen_os1_jinja.md](../custom_templates/zen_os1_jinja.md).
* Help and manifest compliance audit: [`dojotools_manifest.yaml`](../../../packages/zenos_ai/dojotools/dojotools_manifest.yaml) (`audit_help`). Docs: [zen_dojotools_manifest_readme.md](../scripts/zen_dojotools_manifest_readme.md).
<!-- /where -->

<!-- nav -->
---

[← Resilience](23_resilience.md) · [Contents](00_toc.md) · [Doc hub](../readme.md) · [Room Manager v3 Reference →](25_room_manager_v3.md)
<!-- /nav -->
