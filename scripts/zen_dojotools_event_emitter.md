Overview

zen_dojotools_event_emitter is the official ZenOS-AI event publisher used to:

Leave breadcrumbs

Emit structured telemetry

Provide trace reconstruction data

Surface subsystem insights

Communicate micro-summaries between Friday, Kronk, Rosie, and the Monastery


It emits small, structured, contract-safe JSON events into the Home Assistant EventBus under the event type:

zen_event

This enables:

Observability across the ZenOS stack

Reconstructable timelines

Lightweight reasoning sharing between agents

A universal debugging signal path that never touches real state


Version 1.1.2 introduces the canonical event schema used by all ZenOS-AI agents.


---

Design Goals

1. Safety-First Emission

Events:

NEVER modify Home Assistant state

NEVER attempt system changes

NEVER include logs, device dumps, or large content

ALWAYS remain human-readable and LLM-friendly


2. Unified Event Shape

All ZenOS components emit the same shape:

{
  "timestamp": "ISO-8601",
  "component": "string",
  "severity": "debug|info|warn|error",
  "kind": "short classifier",
  "summary": "one- or two-sentence description",
  "metadata": {...},
  "kata": {...},
  "monk": {...}
}

3. LLM-Compatible Contract

Every field is curated to be:

Deterministic

Predictable

Small

Easy for Friday or Kronk to consume


4. Works Everywhere

Subsystems expected to use this tool include:

Friday (front-line assistant)

Veronica (supervisor, planner, debugger)

Kronk (Monastery curator)

High Priestess (deep reasoning)

FileCabinet / Index / Library / Room Manager

Warden (security manager)



---

Primary Capabilities

🟦 1. Emit a Structured Zen Event

Posts a small JSON payload to:

event_type: zen_event

Used for:

Agent breadcrumbs

Background task notifications

Micro-insights

State awareness propagation

Debugging signals

Summarizer anchor-points



---

🟩 2. System Log Mirror

Every event also writes a single-line system log entry:

zen_event <component> | <kind> | <severity> | summary="..." | meta={}

This ensures:

HA log traceability

Easy debugging during dev

Log correlation with the EventBus



---

🟨 3. Optional High-Level Attachments

You may include:

metadata → tiny details (never large objects)

kata → small kata snapshot for reasoning

monk → minimal breadcrumb for the Monastery


Each is optional and must stay compact by design.


---

Version 1.1.2 Enhancements

Area	Improvement

Event Contract	Unified schema across all ZenOS agents
Safety	Hard limits on payload size, no looping, no raw logs
Logging	System log mirror with structured meta
Reliability	Ensures timestamp + component + kind always present
LLM Dev UX	Simplified fields + consistent envelope returned



---

Usage Patterns

1. Emit a simple breadcrumb

component: friday
severity: info
kind: heartbeat
summary: Friday completed her scheduled reflection cycle.

2. Emit a monk breadcrumb

component: monastery
severity: debug
kind: kata_emit
summary: Monk #4 posted a reflection kata.
monk:
  thought: "Temperature variance within expected tolerance."

3. Emit a kata state event

component: room_manager
severity: info
kind: state_change
summary: Living room occupancy updated.
kata:
  occupancy: "present"
  confidence: 0.86


---

Event Schema Reference

Required Fields

Field	Type	Purpose

component	string	Who sent the event
kind	string	What category of event this is
summary	string	1–2 sentence human-readable description


Optional but Recommended

Field	Type	Purpose

severity	enum	debug/info/warn/error (defaults to info)
metadata	object	Tiny structured extras
kata	object	Micro state summary
monk	object	Micro reasoning summary


Auto-Injected

Field	Type	Purpose

timestamp	ISO-8601	Generated at emission time



---

Returned Envelope

Every call returns:

{
  "status": "success",
  "message": "Zen event emitted",
  "event": { ...event payload... }
}

Consistent. Predictable. Friday- and Monastery-safe.


---

Example EventBus Output

{
  "event_type": "zen_event",
  "event_data": {
    "event": {
      "timestamp": "2025-11-15T01:45:39.392222",
      "component": "friday",
      "severity": "info",
      "kind": "heartbeat",
      "summary": "Friday performed scheduled cycle check.",
      "metadata": {},
      "kata": {},
      "monk": {}
    }
  }
}


---

When to Use This Tool

Use the event emitter when:

A subsystem wants to leave a breadcrumb

Kronk posts a Kata

Friday completes a high-level action

Rosie starts or completes a cleaning routine

The FileCabinet settles a write

The Index resolves a complex lookup

The Warden detects unusual behavior


Basically:
If it should appear in a timeline or trace, emit it.

Do not use it for:

Tight loops

High-frequency sampling

Dumping large data

Device state mirroring

Sensitive information



---

LLM Guidelines (Critical)

LLMs must follow these constraints:

Payloads MUST remain tiny

Use summary sentences, never paragraphs

Avoid nested data structures beyond one level

Never include device dumps, logs, or long strings

Avoid redundant or repetitive events



---

Compatibility

Supported in:

Home Assistant 2024.11+

ZenOS-AI DojoTools v1.9+

Monastery/Abbot/Kronk stack v2+


Used by:

Friday

Veronica

Kronk

High Priestess

FileCabinet

Zen Indexer

Summarizer Pipeline



---

Changelog — v1.1.2

Added unified schema (timestamp, component, kind, etc.)

Added system log mirroring

Added kata + monk optional attachments

Hardened payload safety

Standardized return envelope

Improved severity handling

Designed for Friday + Kronk awareness sync



---

Status

GA
Safe for full ZenOS-AI deployment across all agents and subsystems.
