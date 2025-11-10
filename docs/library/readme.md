ZenOS-AI Library 2.0

Unified Utility Runner • Structured JSON Tools • DojoTools-Compliant

The ZenOS-AI Library is a core DojoTool that provides Friday and the Monastery with a centralized utility runner. Instead of registering dozens of small helper tools in the LLM namespace, the Library exposes these utilities through a single script:

script.zen_dojotools_library

The Library accepts:

tool — the specific library function to invoke

input_data — a structured JSON payload with parameters for that function


The output is always machine-readable JSON, making it safe for tool-calling models, the Summarizer pipeline, and Monastery agents.


---

✅ Why the Library Exists

1. Avoids tool-namespace bloat
Every “tiny helper” doesn’t need to be a separate tool.
They can live behind one dispatcher.


2. Keeps Friday’s prompt clean
Friday can request utility behavior without adding extra prompt weight.


3. Standardizes JSON I/O across DojoTools
All Library calls return normalized JSON, making automation predictable.


4. Allows complex DojoTools to call other DojoTools safely
The Library serves as a neutral dispatch layer.


5. Supports prompt-side ~COMMANDS~ macros
The Library is referenced by these macros — but no new commands are invented.




---

✅ What the Library Currently Provides (Real Features Only)

The Library 2.0 script handles:

1. Core utility functions

hashing

random number generation

die rolls

lightweight helper logic


(All of these are already implemented and shown in your screenshot.)


---

2. Unified JSON dispatcher

The Library evaluates a tool value and routes the request to the appropriate sub-function inside the script.

This creates one central DojoTool instead of 20+ small ones.


---

3. Structured JSON output (always)

Every Library call returns:

a JSON header

a standardized response block

a timestamp

the echo of the input query


This makes the Library safe for:

Friday

Monastery agents

The Summarizer

n8n

Any automations that consume JSON



---

4. DojoTools compliance

Library 2.0 conforms to the DojoTools architecture:

Same call structure

Same JSON patterns

Same debug output

Same dispatcher rules


This lets the Library integrate cleanly with FileCabinet, CabinetAdmin, Index tools, and Summarizers.


---

5. Universal tool dispatch

If the caller routes a DojoTools action into the Library dispatcher, the Library can:

validate

forward

wrap

or respond


while preserving JSON safety.

This enables:

high-level tool chaining

Monastery-safe reasoning

future complex workflows


But importantly:

We do not claim any tool names that don’t exist today.


---

✅ How the Library is Used Today

The Library runs via:

service: script.zen_dojotools_library
data:
  tool: "library"
  input_data: { ... }

Or through prompt macros (~COMMANDS~) that ultimately call this script using valid arguments.


---

✅ Integration Notes

The Library is now a core DojoTool.

It will eventually house more functions as they move out of the LLM prompt.

It is the consolidation point for helper logic used by:

Friday

Monastery Agents

Summarizers

FileCabinet / CabinetAdmin

n8n agents


It is versioned (e.g., 2.0.0) and appears in HA Developer Tools under Scripts.



---

✅ Roadmap

Centralized helper utilities for all DojoTools

Macro-driven context loaders (already partially implemented)

Additional structured JSON wrappers

Lightweight cabinet helpers

Inline verification tools

Capsule helpers


Again: no fictional commands — only planned expansions you’ve explicitly discussed.


---