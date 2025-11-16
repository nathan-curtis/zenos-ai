📘 ZenOS-AI Script Modules

Welcome to the Script Modules section of the ZenOS-AI documentation.
This directory contains formal documentation for all DojoTools and ZenOS-AI operational scripts that power Friday’s real-time automation, reasoning, telemetry, and system reflexes.

These scripts form Friday’s “hands, nerves, and breadcrumbs” — the tools that talk directly to Home Assistant services, cabinet storage, calendar providers, inspectors, and the Monastery’s internal reasoning pathways.

Each module is fully documented with:

Technical behavior

Expected inputs & outputs

Provider limitations

Safety constraints

Integration flow for LLM agents (Friday, Veronica, Kronk, etc.)


If Friday performs an action, reads something, or leaves a breadcrumb, it probably came from here.


---

📂 Included Script Documentation


---

1. Zen DojoTools Calendar — v1.10.3

File: zen_dojotools.calendar_readme.md
Type: Technical Documentation

Summary:
Provides unified, deterministic access to all Home Assistant calendar entities, including:

Multi-calendar read aggregation

Unified timestamp normalization

Advanced event inspection

Safe create operations

Provider-verified event update/delete (event_id-only)

Label-based targeting

Strict ambiguity prevention & error handling

Fully structured JSON response envelopes


Designed for LLMs, hardened for safety, and consistent across all exit paths.
This is Friday’s and Veronica’s primary interface for anything date- or schedule-related.


---

2. Zen DojoTools Event Emitter — v1.1.2

File: zen_dojotools_event_emitter.md
Type: Technical Documentation

Summary:
A universal, contract-safe telemetry tool for emitting structured ZenOS-AI events onto the Home Assistant EventBus.

Supports the canonical ZenOS event shape:

{
  "timestamp": "ISO-8601",
  "component": "string",
  "severity": "debug|info|warn|error",
  "kind": "classifier",
  "summary": "single-sentence explanation",
  "metadata": {},
  "kata": {},
  "monk": {}
}

Core capabilities:

Emits HA EventBus events of type zen_event

Mirrors each event into system_log.write for trace alignment

Accepts optional kata and monk micro-attachments

Fully safe (no state mutation, no privileged actions)

Designed for breadcrumbs, observability, and Monastery awareness


This is how Friday, Veronica, Kronk, and the High Priestess leave breadcrumbs and micro-summaries for each other.
