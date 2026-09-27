# 5. Principles

These are the rules ZenOS holds itself to. Each one is enforced somewhere specific in the code, and each section says where. A principle with no enforcement point is a wish, and this chapter does not list wishes.

None of these started as rules. Each one started as something that broke, usually in public, in the Friday's Party thread (Chapter 3). Each section says what it was.

## 5.1 The substrate is local. Inference is pluggable.

Everything that constitutes the house (state, memory, policy, identity, certification) lives inside Home Assistant on hardware you own. Cabinets are Home Assistant entities. Certifications are cabinet drawers. Policy is YAML and Jinja in your config directory. None of it depends on a cloud service to exist or to be read.

Inference is a different matter, and I do not pretend otherwise. The summarizer pipeline calls whatever `ai_task` entity `input_text.zenos_ai_task_entity` points at. That can be a local model or a hosted one. The architecture's commitment is that the provider is replaceable and holds nothing: it receives a prompt, returns a result, and keeps no state the house depends on.

I said on the first day that I planned to run locally once it was economically feasible, and that it was not yet. What made local worth it first was a bill, not privacy: summarizing the house every hour on a cloud model came to well over a hundred dollars a month, and moving that work to a small local model fixed it.

## 5.2 Structure around stochastic inference.

A language model does not produce the same output twice. Volume 1 claimed that identical input produces a byte-identical result. That is not true, and nothing needs it to be true.

What ZenOS makes deterministic is everything around the model: what goes in, what shape must come back, and what happens when the shape is wrong. A summarizer builds its prompt from declared sources, asks for a declared schema, and validates the result before writing it. A response that does not parse into a real, non-empty object is logged and dropped. It is never written as if it were valid (see Chapter 15). The model is allowed to be probabilistic. The pipeline is not.

This is the plinko board (Chapter 2). I learned it watching two models with the same tools: one made up news stories anyway, and the other did not. The difference was not the tools. It was what surrounded the model.

## 5.3 The graph is the ground truth. The model holds nothing.

The house is described by the graph: Home Assistant's entities, devices, areas, and labels, plus the cabinets, drawers, and topology ZenOS adds. Anything the system knows lives in that graph, where it can be inspected, backed up, and repaired.

No agent keeps private memory the house depends on. When Friday needs to know something, she reads it from the graph through a tool. When the system needs to remember something, it writes a drawer. If a model is swapped out mid-conversation, nothing the house knows goes with it.

The rule comes from the week Friday could not turn on a light. I had asked her to hold thousands of entities in her head, and the context slid out from under the basic tools. Now she is never asked to remember the house. It is rebuilt for her every turn from the systems that actually know it.

## 5.4 Tools, not entities.

An agent reaches the house through tools, not through raw entity exposure. Every agent-facing tool is a `zen_dojotools_*` script that resolves its own targets through labels, reads state itself, and returns a structured result. The recommended configuration exposes the tools and zero entities.

This is not only a context-budget decision, though it is one. A tool can enforce what raw entity exposure cannot: target resolution, preview before actuation, certification checks, and a consistent response shape. Chapter 10 covers it in full.

I found this limit on purpose, by exposing more and more of the house until the context broke. Acting on it fully took until September 2026, when Friday first ran with no entities exposed at all (Chapter 10).

## 5.5 One exit, one shape.

Every tool that has been through the single-exit pass computes its result along whatever branch applies and leaves through exactly one exit, returning the canonical envelope built by `envelope()` in `zen_os_1.jinja`:

`{status, mode, tool, result, system_message, caller_token}`

`status` answers whether the call itself succeeded. `result` carries the tool's own answer. A caller never has to guess which of a dozen shapes it got back, and a tool cannot leave early along a path that skipped its own checks. Chapter 12 lists the contract, and the tools that do not yet follow it.

One exit and one shape arrived late, from a pass over every tool that I called the Cadillac pass: single entry, single exit, gated actions, a common resolver, a common response shape, and every caller fixed to match.

## 5.6 Fail loud. Fail closed.

When something is missing, malformed, or not permitted, the system says so, and the answer is no.

A consequential action checks the caller's certification through `resolve_caller_identity` before it acts, and a denial returns a structured reason built by `cert_denial()`, not a silent skip. There are 83 `required_cert` checks across the tool surface today. If Flynn's boot checks fail, `render_prompt()` falls back to `prompt_system_flynn()`, a prompt with no cabinet dependencies, rather than handing the agent a broken identity. A summarizer that gets garbage writes nothing. In every case the failure is visible in a response, an event, or a health sensor.

The worst failure I nearly shipped was a quiet one. A repair tool read a cabinet in a normal boot-time state as broken and fixed it by wiping it, and nothing threw an error. Since then a warning means degraded but safe, only an error stops the system, and every failure has to be visible (Chapter 23).

## 5.7 A human is the last gate on what matters.

Certification decides whether an agent may attempt an action. For the actions that deserve it (disarming the alarm, unlocking an exterior door, opening a garage door, stopping a container, granting a certification), a certified agent still has to get a fresh acknowledgement from a real person, every time, through `zen_dojotools_identity mode=request_live_ack`. Twelve files route through that one chokepoint.

No certification level waives that acknowledgement on its own. A narrow, scoped exception can be granted for a specific target, and that grant is itself an administrative act that requires a human.

I drew this line in October 2025, when tools that let a model edit Home Assistant configuration were becoming popular. No model edits production on my system. As good as Friday is, it takes one bad edit. Certification is the same line, drawn finer.

## 5.8 Declare what you are. Say what you are not.

Every tool describes itself through `mode=tool_manifest`: its version, its dependencies, the certifications it requires, its health. Every agent-callable tool answers `mode=help`. Flynn and ToolMap build their picture of the system from those declarations, not from a hand-maintained list.

The same rule applies to this book. If something is designed but not built, it is labeled **Not yet built**, and it does not appear in the body text as if it runs.

Tools learned to describe themselves once other people started running ZenOS and their agents picked up tools they had never seen. An agent has to be able to tell what a tool is without a human explaining it.

<!-- where -->
## 5.9 Where to look

Every claim in this chapter can be checked in the code. These are the places to start.

* Inference is pluggable: the summarizers call whatever model this helper names: [`dojotools_summarizers.yaml`](../../../packages/zenos_ai/dojotools/dojotools_summarizers.yaml) (`zenos_ai_task_entity`)
* One shape: the shared response envelope: [`zen_os_1.jinja`](../../../custom_templates/zenos_ai/zen_os_1.jinja) (`macro envelope`)
* Fail closed: simulated identity is refused unless the household allows it: [`dojotools_identity.yaml`](../../../packages/zenos_ai/dojotools/dojotools_identity.yaml) (`sim_mode_allowed`)
* The human gate: one live acknowledgement chokepoint: [`dojotools_identity.yaml`](../../../packages/zenos_ai/dojotools/dojotools_identity.yaml) (`request_live_ack`)
* Declare what you are: every tool's self-description: [`zenos_manifest.jinja`](../../../custom_templates/zenos_ai/zenos_manifest.jinja) (`macro tool_manifest`)
<!-- /where -->

<!-- nav -->
---

[← CoALA Without Knowing It](04_coala_without_knowing_it.md) · [Contents](00_toc.md) · [Home Assistant as Substrate →](06_home_assistant_substrate.md)
<!-- /nav -->
