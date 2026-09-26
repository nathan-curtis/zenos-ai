# 4. Principles

These are the rules ZenOS holds itself to. Each one is enforced somewhere specific in the code, and each section says where. A principle with no enforcement point is a wish, and this chapter does not list wishes.

## 4.1 The substrate is local. Inference is pluggable.

Everything that constitutes the house (state, memory, policy, identity, certification) lives inside Home Assistant on hardware you own. Cabinets are Home Assistant entities. Certifications are cabinet drawers. Policy is YAML and Jinja in your config directory. None of it depends on a cloud service to exist or to be read.

Inference is a different matter, and I do not pretend otherwise. The summarizer pipeline calls whatever `ai_task` entity `input_text.zenos_ai_task_entity` points at. That can be a local model or a hosted one. The architecture's commitment is that the provider is replaceable and holds nothing: it receives a prompt, returns a result, and keeps no state the house depends on.

## 4.2 Structure around stochastic inference.

A language model does not produce the same output twice. Volume 1 claimed that identical input produces a byte-identical result. That is not true, and nothing needs it to be true.

What ZenOS makes deterministic is everything around the model: what goes in, what shape must come back, and what happens when the shape is wrong. A summarizer builds its prompt from declared sources, asks for a declared schema, and validates the result before writing it. A response that does not parse into a real, non-empty object is logged and dropped. It is never written as if it were valid (see Chapter 14). The model is allowed to be probabilistic. The pipeline is not.

## 4.3 The graph is the ground truth. The model holds nothing.

The house is described by the graph: Home Assistant's entities, devices, areas, and labels, plus the cabinets, drawers, and topology ZenOS adds. Anything the system knows lives in that graph, where it can be inspected, backed up, and repaired.

No agent keeps private memory the house depends on. When Friday needs to know something, she reads it from the graph through a tool. When the system needs to remember something, it writes a drawer. If a model is swapped out mid-conversation, nothing the house knows goes with it.

## 4.4 Tools, not entities.

An agent reaches the house through tools, not through raw entity exposure. Every agent-facing tool is a `zen_dojotools_*` script that resolves its own targets through labels, reads state itself, and returns a structured result. The recommended configuration exposes the tools and zero entities.

This is not only a context-budget decision, though it is one. A tool can enforce what raw entity exposure cannot: target resolution, preview before actuation, certification checks, and a consistent response shape. Chapter 9 covers it in full.

## 4.5 One exit, one shape.

Every tool that has been through the single-exit pass computes its result along whatever branch applies and leaves through exactly one exit, returning the canonical envelope built by `envelope()` in `zen_os_1.jinja`:

`{status, mode, tool, result, system_message, caller_token}`

`status` answers whether the call itself succeeded. `result` carries the tool's own answer. A caller never has to guess which of a dozen shapes it got back, and a tool cannot leave early along a path that skipped its own checks. Chapter 11 lists the contract, and the tools that do not yet follow it.

## 4.6 Fail loud. Fail closed.

When something is missing, malformed, or not permitted, the system says so, and the answer is no.

A consequential action checks the caller's certification through `resolve_caller_identity` before it acts, and a denial returns a structured reason built by `cert_denial()`, not a silent skip. There are 83 `required_cert` checks across the tool surface today. If Flynn's boot checks fail, `render_prompt()` falls back to `prompt_system_flynn()`, a prompt with no cabinet dependencies, rather than handing the agent a broken identity. A summarizer that gets garbage writes nothing. In every case the failure is visible in a response, an event, or a health sensor.

## 4.7 A human is the last gate on what matters.

Certification decides whether an agent may attempt an action. For the actions that deserve it (disarming the alarm, unlocking an exterior door, opening a garage door, stopping a container, granting a certification), a certified agent still has to get a fresh acknowledgement from a real person, every time, through `zen_dojotools_identity mode=request_live_ack`. Twelve files route through that one chokepoint.

No certification level waives that acknowledgement on its own. A narrow, scoped exception can be granted for a specific target, and that grant is itself an administrative act that requires a human.

## 4.8 Declare what you are. Say what you are not.

Every tool describes itself through `mode=tool_manifest`: its version, its dependencies, the certifications it requires, its health. Every agent-callable tool answers `mode=help`. Flynn and ToolMap build their picture of the system from those declarations, not from a hand-maintained list.

The same rule applies to this book. If something is designed but not built, it is labeled **Not yet built**, and it does not appear in the body text as if it runs.

<!-- nav -->
---

[← CoALA Without Knowing It](03_coala_without_knowing_it.md) · [Contents](00_toc.md) · [Home Assistant as Substrate →](05_home_assistant_substrate.md)
<!-- /nav -->
