# Pipeline Trigger Pattern — Act on the Monk After Kata Is Written

*How to wire an automation that asks for a summarizer run and responds to the result*

---

## The Pattern

The ZenOS cognition pipeline runs on a schedule. But sometimes you want a specific
component to run now, because a door opened, a threshold was crossed or a button was
pressed, and then do something with what the monk said.

The pattern is three steps:

```
1. Your trigger fires
2. zen_event → summary_request (component: your_component)
3. Wait for the kata write → read it → act on it
```

The Scheduler picks up the `summary_request` event and admits the component through its
normal gates (no subscription needed, no declared delay), then the Ninja Summarizer runs
the monk and writes the kata drawer. You wait for the write, then read the kata and branch
on the result.

Use `summary_request`, not `summary_force`. A request is the routine path: it goes through the
Scheduler's tier and depth gates like any trigger. Forces are for maintenance only. See
[Requesting a run](zen_dojotools_scheduler_readme.md#requesting-a-run).

---

## Step 1 — Ask for the Run

```yaml
automation:
  - alias: Water Manager — Respond on Flow Stop
    mode: single
    triggers:
      - trigger: state
        entity_id: binary_sensor.water_flow_active
        to: "off"
        not_from: [unknown, unavailable]
    actions:
      - variables:
          _asked_at: "{{ now().timestamp() }}"
      - event: zen_event
        event_data:
          event:
            kind: summary_request
            component: water_manager
```

The `component` field must match the drawer key (slugified) in the Dojo cabinet: exactly
what you passed as `key` when you authored the KFC with Scribe.

A request can be shed. Under pressure the Scheduler drops `ambient` components (and, at a high
pressure, `keeper` ones). A shed request isn't retried; the drain router recovers the component
later from its kata age. If your automation must have an answer, give it a timeout and a fallback
(Step 2).

---

## Step 2 — Wait for the Kata Write

Every successful kata write stamps `last_run_at` (Unix seconds, as a string). Wait until it's
newer than the moment you asked:

```yaml
      - wait_template: >-
          {%- set kv = state_attr(states('sensor.zen_kata_cabinet_resolved'), 'variables') or {} -%}
          {%- set d = kv.get('water_manager') -%}
          {%- set v = d.get('value', d) if d is mapping else {} -%}
          {%- set v = (v | from_json({})) if v is string else v -%}
          {{ v is mapping and (v.get('last_run_at') | float(0)) > (_asked_at | float(0)) }}
        timeout:
          seconds: 120
        continue_on_timeout: true
      - if: "{{ not wait.completed }}"
        then:
          - stop: "No fresh kata (shed, busy, or the summarizers are off)"
```

The timeout should cover a busy queue: the Ninja runs one summary at a time. A local model
usually answers in under a minute.

**The `kata_emit` event.** Every Ninja run also fires `zen_event` with `kind: kata_emit` and
`metadata.component_slug` when it finishes. A listener can use it instead of the wait, but it
fires for every component, so filter on `component_slug`, and check `metadata.status`.

---

## Step 3 — Read the Kata and Act

The kata fields are set by the Kata cabinet's `zen_template`. The ones an automation usually
branches on:

| Field | Type |
|---|---|
| `urgency` | 0 to 10 integer |
| `action_required` | boolean |
| `attention` | one line: what needs looking at |
| `general_inference` | one line: what the monk concluded |
| `last_run_at` | Unix seconds, as a string |

```yaml
      - variables:
          _kata: >-
            {%- set kv = state_attr(states('sensor.zen_kata_cabinet_resolved'), 'variables') or {} -%}
            {%- set d = kv.get('water_manager') -%}
            {%- set v = d.get('value', d) if d is mapping else {} -%}
            {%- set v = (v | from_json({})) if v is string else v -%}
            {{ v if v is mapping else {} }}
      - variables:
          _urgency: "{{ _kata.get('urgency', 0) | int(0) }}"
          _line: "{{ _kata.get('attention') or _kata.get('general_inference') or '' }}"
      - choose:
          - conditions: "{{ _urgency >= 7 and _line != '' }}"
            sequence:
              - action: script.zen_dojotools_announce
                data:
                  message: "{{ _line }}"
                  urgency: "{{ _urgency }}"
          - conditions: "{{ _urgency >= 4 and _line != '' }}"
            sequence:
              - action: notify.mobile_app_your_phone   # REPLACE
                data:
                  message: "{{ _line }}"
```

Pick your thresholds to match your component's `drift_threshold` and push floor. The kata is
the monk's opinion: keep a deterministic rule (a leak shutoff, a smoke alarm) in its own
automation, never behind this wait.

---

## Why `mode: single`

This automation fires on physical events. If flow stops and starts in rapid succession,
you do not want several runs waiting on the same kata. `mode: single` drops the second trigger
while the first is still waiting. Use `mode: queued` if you want them to serialize instead.

---

## What You Need in the KFC

The automation is only as useful as the component's instructions. Tell the monk what each
urgency means for this component, and what belongs in `attention`:

```yaml
component_instructions: >
  urgency 0 to 10. 7 or more if flow is detected while the house is Away, or has run over
  2 hours. 4 to 6 if it has run over 30 minutes. 0 to 2 if all is nominal. attention: one
  sentence a person could hear read aloud.
```

---

## Calling the Ninja Summarizer Directly (MCP Path)

An agent, or a tool acting for one, can call the Ninja Summarizer directly and read the
response inline, with no wait:

```yaml
    - action: script.zen_dojotools_ninja_summarizer
      data:
        kung_fu_component_id: water_manager
        post_to_kata_cabinet: true
      response_variable: _ninja_result
```

The response comes back when the run finishes, and the kata write happens inside the same call
when `post_to_kata_cabinet: true`. This skips the Scheduler's gates: the run starts now, whatever
the pressure. Prefer the `summary_request` path from an automation that reacts to the house, so
the Scheduler can pace it.

---

## Related Docs

- [Zen DojoTools Summarizers](zen_dojotools_summarizers_readme.md) — Ninja Summarizer + SuperSummary reference
- [Zen DojoTools Scheduler](zen_dojotools_scheduler_readme.md) — trigger IDs, requests and maintenance forces
- [Zen DojoTools Scribe](zen_dojotools_scribe_readme.md) — authoring the KFC component this pattern fires against
- [Zen DojoTools FileCabinet](zen_dojotools_filecabinet_readme.md) — kata read path
