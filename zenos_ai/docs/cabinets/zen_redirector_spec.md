ZenOS-AI DojoTools Volume Redirector v2

Overview

Why This Logic Lives In Its Own Automation

The Volume Redirector is intentionally pulled out of FileCabinet and implemented as a standalone automation because it serves two critical architectural purposes:

1. It Is Used By More Than FileCabinet

The redirector is a shared service used by:

FileCabinet

The Zen Library Manifest validator

Any component that needs to read or write cabinet data

Runtime components that need cabinet metadata (e.g., Room Manager, Taskmaster, Trash Trakker, Friday’s internal subsystems)


By isolating the redirector, every subsystem can rely on a single, consistent dispatch point for cabinet-aware operations.

2. Embedding This Into FileCabinet Would Make FileCabinet Brittle

If the redirector logic were baked directly into FileCabinet:

Every change to cabinet routing would require editing FileCabinet’s internal script.

Any user customization (different cabinet layouts, differing entity IDs) would force local forks.

The FileCabinet codebase would balloon with routing logic that does not belong to it.


Separating the redirector keeps FileCabinet clean, stable, and portable.

3. Automations Allow Event-Based Triggering That FileCabinet Cannot Perform

Home Assistant automations:

Can listen to multiple event types.

Can branch routing logic without templating event names (which HA forbids).

Provide a neutral execution environment where brittle elements can be isolated.


FileCabinet cannot do any of this internally — it cannot “listen” for events.

4. The Redirector Is the Only Component Users Must Customize

By moving the most user-edited section — the cabinet entity IDs — into its own automation:

End users only touch a small, clearly marked stub.

All sensitive or complex logic stays untouched and safe.

The system becomes resilient to updates.


5. This Architecture Creates a Clean, Stable, and Extensible Path Forward

New cabinets, new agents, and new subsystems can be added by simply:

1. Adding their entity ID to the Redirector stub.


2. Adding a routing block if needed.



Nothing else changes.

This separation is deliberate, tested, and absolutely required for long-term maintainability.

Home Assistant has one very sharp edge: event_type: cannot be templated. ever.

This is the entire reason the Redirector exists.

FileCabinet needs to translate generic legacy events like:

set_variable_legacy

remove_variable_legacy

clear_variable_legacy


…into cabinet‑specific event calls, such as:

set_variable_friday_s_cabinet

remove_variable_zen_kata_storage_cabinet

clear_variable_my_household_cabinet


Under normal circumstances, we’d simply template the event name and be done. But because HA forbids templated event names, FileCabinet has no way to dynamically dispatch writes.

The Redirector fills that missing capability. It acts as a translation firewall between legacy calls and cabinet‑aware targets.

It:

Validates the target volume

Confirms manifest identity

Ensures GUID consistency

Determines the correct static event name

Emits telemetry

And only then fires the cabinet‑specific write


Without this layer, FileCabinet would be forced into brittle, unsafe hardcoding.


---

Purpose

The Volume Redirector provides:

A safe translation layer for legacy variable events.

Manifest-aligned validation of every cabinet.

A single deterministic routing system for set/remove/clear.

Strict protection from misrouted or unsafe writes.

Full telemetry for debugging and observability.



---

Trigger Events

The redirector listens for:

set_variable_legacy

remove_variable_legacy

clear_variable_legacy

dojo_volume_redirector_probe


Legacy events are routed; probe events are inspected but never written.


---

Key Responsibilities

1. Probe Handling – Non‑destructive validation of a cabinet.


2. Manifest Validation – Ensures the cabinet is known, recognized, and correct.


3. Routing – Selects the correct set/remove/clear_variable_<cabinet> event.


4. Telemetry – Emits detailed result events for debugging.




---

Manifest Validation Logic

Extracts from the Zen Library Manifest:

Manifest entry

Cabinet metadata

Manifest GUID (authoritative)

Cabinet GUID (advertised by sensor)


If they do not match → guid_mismatch = true, which becomes warning or error depending on context.


---

Recognized Volume List

Public-safe examples:

vol_system
vol_summary
vol_household
vol_family
vol_agent
vol_user1
vol_user2
vol_kata
vol_history
vol_agent_history
vol_dojo


---

Canonical Volume Name Mapping

When friendly_name isn't sufficient, the redirector falls back to a known map:

ai_summary

household

family

agent_private

user_private

zen_kata

zen_history

agent_history

zen_dojo

unknown



---

Probe Result Event – dojo_volume_redirector_probe_result

Contains:

status: ready | warning | error

manifest_entry_found

recognized_volume

manifest_guid

cabinet_guid

guid_mismatch

failure_reason


Probe mode allows callers to check correctness without writing anything.


---

Routing Result Event – dojo_volume_redirector_result

Contains:

status: success | error

action: set | remove | clear

event type chosen

volume IDs

key + value length

timestamp option

manifest/cabinet GUIDs

manifest flags

result or failure



---

Default Case

If a volume is valid but no route matches:

status: error
failure: unmatched_volume_route


---

Safety Guarantees

Redirector enforces:

No writes to unknown volumes.

No writes missing keys.

No writes missing manifest entries.

Halt on GUID mismatch.

Clear telemetry on every failure.



---

Public Version Notes

All person‑specific and private cabinet references are sanitized for GitHub. Logic remains exact.


---

New in v2: Probe, Hardening & Debug Features

🔍 Probe Mode

A safe “what would happen if…” call.

Returns structured fields:

recognized_volume

manifest_entry_found

manifest_guid vs cabinet_guid

guid_mismatch

failure_reason


Probe mode is heavily used by:

Friday

Kronk

The Monastery

Testing tools



---

🛡️ Hardening

Version 2 introduces significant safety:

GUID validation

Manifest ancestry checks

Strict volume recognition

Early-stop guards on failures


If anything is suspicious → stop, emit telemetry, do not write.


---

🧪 Debug Telemetry

If log_events: true in the triggering event:

The entire transaction is logged

Includes resolved cabinet and routing path

Useful for development and deep troubleshooting



---

Setup Requirements

The redirector cannot auto-discover cabinets.
The user must configure the correct entity IDs and cabinet slugs.

1. Insert Your Own Cabinet Entity IDs

Example:

vol_kata: sensor.zen_kata_storage_cabinet

Every vol_* entry must match your real HA entity IDs.


---

2. Ensure Correct Cabinet Event Handlers Exist

Each cabinet must expose three event handlers, and they must match exactly the trio emitted by your Fes Trigger Sensors:

set_variable_<cabinet_slug>_cabinet
remove_variable_<cabinet_slug>_cabinet
clear_variable_<cabinet_slug>_cabinet

These event names are generated by your cabinet’s Fes Trigger Sensor, and the redirector depends on them.

If you’re unsure what your cabinet exposes, refer to: Fes Trigger Sensor Documentation → “Cabinet Event Handler Requirements”

That guide explains how to:

Identify each cabinet’s slug

Verify the exposed events

Confirm the expected handlers exist


If these handlers are missing or mismatched, you will see failures such as:

unmatched_volume_route

volume_not_recognized


This is intentional — the redirector refuses to guess and protects data integrity.


---

3. What Happens If Something Is Missing

You may see:

volume_not_recognized

volume_missing_from_manifest

unmatched_volume_route

guid_mismatch


These failures are intentional — they protect your data.


---

Conclusion

The ZenOS-AI DojoTools Volume Redirector v2 exists because Home Assistant has sharp edges, cabinets need protection, and FileCabinet deserves to stay elegant instead of becoming a lumbering event-router goblin.

This module serves as the universal, authoritative router for all cabinet-bound writes — legacy, agent, subsystem, and system-level. It:

Prevents accidental or unsafe writes

Maintains manifest and GUID integrity

Provides centralized observability

Keeps FileCabinet lean and maintainable

Ensures every agent and subsystem writes exactly where it should


By isolating routing into a standalone automation, we give developers and end‑users a safe, extensible, future-proof foundation. Adding a cabinet? Easy. Adding a new subsystem that needs cabinet access? Even easier.

For deeper details, examples, or to extend this functionality, see:

Zen Library Manifest Spec

Fes Trigger Sensors Guide

FileCabinet v3 Architecture Notes

ZenOS-AI Kata & Cabinet Design Guide


This redirector is the glue that makes the whole Cabinet ecosystem coherent. Treat it with respect, keep the entity IDs updated, and it will quietly run your world. 