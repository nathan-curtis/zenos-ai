# 9. Components

The last piece of the ontology is the component vocabulary: the kinds of thing ZenOS is built from, and how they find each other at runtime. A component's class says who may call it, what it may call, and whether an agent ever sees it. Chapter 20 is the developer reference for building each class. This chapter is the reader's map.

## 9.1 Classes

| Class | Prefix | Role | Agent-exposed |
|---|---|---|---|
| DojoTool | `zen_dojotools_*` | A capability offered to agents: semantic modes, label-resolved targets, envelope responses | Yes, when intended as a capability |
| SystemTool | `zen_dojotools_systemtools` and kernel services | Platform health, logs, reloads, configuration surfaces | Safe modes only |
| AdminTool | `zen_admintools_*` | The configuration and recovery plane: certifications, cabinet repair, resets, loaders | No |
| Root | `zen_root_*` | Raw transport to an external system (an API, an auth provider) | Never |
| Sutra | `zen_sutra_*` | An internal adapter that does the work behind a DojoTool | Never |
| Stack | `zen_stack_*` | A Lens Bus provider that answers anchor queries with evidence | No, routed through Lens dispatch |
| Codex | `zen_codex_*` | Domain policy and rules a DojoTool delegates to | No, routed through its host |
| Kung Fu Component | KFC drawer | A summarized domain view the Monastery keeps fresh (Chapter 10) | Mounted as context, not called |
| Boot orchestrator | Flynn | Staged startup and health gating | Never |

A name is a strong hint, not a guarantee. A `zen_dojotools_*` script can be internal-only when it exists as a provider implementation, and its manifest says so through `mcp_exposed`. Exposure is a declared policy, not an inference from the name.

The common composition for a consequential capability is layered: an agent calls a DojoTool mode, the DojoTool resolves targets and delegates policy to a Codex, the Codex checks identity and certification through the chokepoint (Chapter 14), asks for a live acknowledgement if its policy requires one (Chapter 16), and only then calls the Root that talks to the outside system. Configuration of that capability lives in an AdminTool the agent cannot reach.

## 9.2 Kung Fu Components and self-registration

A Kung Fu Component (KFC) is a domain the Monastery summarizes: security, alerts, tasks, the physical plant, and so on. Its definition lives in a drawer in the Dojo cabinet: what labels it reads, what triggers it subscribes to, its pipeline tier, its staleness ceiling, and its instructions (Chapter 10).

Under KF5, a tool owns its own KFC. The tool carries the `zen_kfc_provider` label and answers `mode=kfc_manifest` with its component definition. `zen_dojotools_manifest mode=bootstrap_kfc` runs on Home Assistant start and daily, finds every tool labeled `zen_kfc_provider`, collects each `kfc_manifest`, and mounts the result into the Dojo cabinet. Nobody authors a KFC drawer by hand. Adding a tool that owns a domain adds that domain to the Monastery.

## 9.3 The Lens Bus

The Lens Bus is how a tool asks "what does the house know about this?" without knowing who knows it. The question is an anchor: a label, a person, an area, or a zone. The answers are evidence from whichever providers consume that anchor type.

Providers are Stacks (and some DojoTools acting as providers). Each declares in its manifest what anchor types it consumes and what evidence types it returns, and registers itself into the household cabinet's `lens_registry` drawer with `mode=register`. `zen_dojotools_manifest mode=bootstrap_stacks` runs on start and daily and registers any provider that declares a registration mode and is not already registered.

`zen_dojotools_lens_dispatch` is the one consumer entry point. It groups the anchors by type, calls only the providers whose declarations match, applies soft-failure semantics (a failed provider is reported, not fatal), deduplicates evidence, and returns one merged answer. It unwraps each provider's `.result` before merging, so enveloped and not-yet-enveloped providers merge identically. Library is the Lens owner for documents, tickets, wiki pages, and catalog items, and consumers such as Inspect, Index, and Room Manager ask Library rather than calling providers directly.

## 9.4 ToolScan and ToolMap

System-wide questions about tools (which tools exist, what they require, what they depend on) are answered by fanning out across every tool's manifest.

`zen_admintools_toolscan` is the shared collector. It exists as its own script because Home Assistant refuses to let a script call itself, so a manifest mode that needs to call every tool, including the manifest tool, cannot do it from inside that tool. `zen_dojotools_manifest` modes `toolmap`, `domains`, `label_audit`, `cert_audit`, and `audit` all collect through ToolScan.

`zen_dojotools_manifest mode=toolmap` walks the `dependencies` each tool declares and builds the dependency graph: what depends on Identity, what breaks if FileCabinet is unhealthy, whether a missing dependency is actually missing or just undeclared. It is a declared graph, not runtime tracing. The tools say what they need, and ToolMap checks that it is there.
