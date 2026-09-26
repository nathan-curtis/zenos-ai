# 11. Contracts

Labels tell an agent what a node is. Contracts tell it what a tool is: what it does, what it needs, what it returns, and what it requires of the caller. Every tool carries three contracts, and each is machine-readable and checkable.

| Contract | Answers | Surface |
|---|---|---|
| Manifest | What is this tool, what does it depend on, what does it require? | `mode=tool_manifest` |
| Help | How do I call it? | `mode=help` |
| Envelope | What shape will the answer have? | Every response |

## 11.1 The manifest

Every tool describes itself by calling the shared `tool_manifest()` macro in `custom_templates/zenos_ai/zenos_manifest.jinja` from its own `mode=tool_manifest` branch. A field added to the macro propagates to every compliant tool at once.

The fields that matter most:

| Field | Meaning |
|---|---|
| `tool`, `display_name`, `tier` | Identity and class (Chapter 12) |
| `version` | The tool's version. This is the canonical version of the tool (11.4). |
| `health` | Whether the tool is configured and working, with notes |
| `required_labels`, `optional_labels` | The vocabulary it depends on (Chapter 10) |
| `missing_required_labels`, `missing_optional_labels` | Computed on every call against the live label registry |
| `certs_required` | The certifications it enforces, per mode and level (Chapter 18) |
| `dependencies` | The tools it calls, each as `{tool, required, uses}` |
| `inference` | Whether it calls a model, what kind of call, and estimated tokens |
| `mcp_exposed` | Whether it is meant to be agent-reachable |
| `consumes`, `returns`, `lens_provider`, `stack` | Its Lens Bus role, when it has one |
| `modes`, `risk_class`, `limitations`, `prerequisites`, `fallback` | How it behaves and what can go wrong |

`inference` is always reported. A tool that never calls a model gets the false-shaped default automatically. `dependencies` is what ToolMap walks to build the declared dependency graph (Chapter 12). Both are declarations, not observations: the manifest says what the tool believes about itself, and the audit modes check that belief against reality.

`zen_dojotools_manifest` aggregates the manifests. `cert_audit` builds the certification catalog. `label_audit` finds missing labels. `audit` reports drift between what tools declare and what they do. `toolmap` walks `dependencies`.

## 11.2 Help

Every agent-callable `zen_dojotools_*` tool answers `mode=help` with the same shape, modeled on `zen_dojotools_locks`:

`{status: help, message: <string>}`

The message covers modes, fields, examples, and gotchas. Tools whose native selector is not named `mode` (some use `operator`, `action_type`, `action`, or `tool`) accept `mode` as an alias that takes precedence, and the legacy field still works. A caller can always send `mode=help` to any tool and get documentation back, not a side effect.

Help exists so a tool's top-level description can stay short. The description is what a model reads when choosing a tool, and it is resent on every turn, so it says what the tool is for and points at `mode=help`. The detail lives in help, fetched only when the agent needs it. This is a large share of how tool descriptions came down to a quarter to a third of their previous size in 2026.10.0.

`zen_dojotools_manifest mode=audit_help` checks every tool against this contract. A tool is compliant only if `mode=help` returns `{status: help, message}` and `mode=tool_manifest` returns a complete manifest. The report says which half failed. Internal-only tiers (`zen_sutra_*`, `zen_stack_*`, `zen_codex_*`) are reported as exempt, because they are never agent-callable and each is fronted by a compliant host tool. `zen_dojotools_help mode=help tool=<name>` proxies to any tool's help, and returns `status: unsupported` rather than passing a tool's live data off as documentation.

## 11.3 The envelope

A tool that has been through the single-exit pass returns every response in the canonical envelope, built by `envelope()` in `zen_os_1.jinja`:

```text
{
  status:         success | error        (did the call itself work)
  mode:           the mode that ran
  tool:           the tool's canonical name
  result:         the tool's own answer
  system_message: a priority notice for the agent, or null
  caller_token:   echoed back unchanged
}
```

`status` is the execution outcome only. Whatever the tool has to say about the world lives in `result`, in whatever shape that tool defines. A caller that reads a tool's response through `response_variable` reads domain fields from `.result`. A tool that forwards another tool's response as its own unwraps `.result` first, so its callers never see a double envelope.

`system_message` is a side channel. It reads the same priority-notice drawer the prompt uses, filtered to entries meant for tool responses, so a notice can reach an agent in the middle of a tool call instead of waiting for the next turn. It is usually `null`.

Single exit and the envelope go together. A tool computes `result` along whatever branch applies and leaves through one stop, wrapped in one envelope. It cannot leave early along a path that skipped its own checks. Every branch after a possible early answer is guarded on "no result yet", which is what stops a certification denial from being overwritten by the mode logic that would otherwise run next.

The envelope lives in `zen_os_1.jinja` on purpose. That file is already a hard dependency of the prompt and of Flynn, so importing it adds no new failure point. A missing `{% import %}` target in Home Assistant's Jinja is not a soft error: it kills the whole call.

Most DojoTools, AdminTools, and plugin tools return the envelope. The ones that do not yet are listed in the `envelope()` reference in `custom_templates/zen_os1_jinja.md`. Check a tool's own reference before assuming either shape.

## 11.4 Versions

A tool's `tool_manifest` version is its canonical version. It is the number `mode=help`, `audit_help`, and ToolMap report, so it is the number every other mention must match: the header comment, the help text, the description, and the tool's readme. A file that bundles several scripts has a version per script. An automation with no manifest uses its file header.

<!-- nav -->
---

[← Vocabulary](10_vocabulary.md) · [Contents](00_toc.md) · [Components →](12_components.md)
<!-- /nav -->
