# Pipeline Trigger Pattern — Act on the Monk After Kata Is Written

*How to wire an automation that fires a summarizer run and responds to the result*

---

## The Pattern

The ZenOS cognition pipeline runs on a schedule. But sometimes you want to trigger a
specific component immediately — because a door opened, a threshold was crossed, a button
was pressed — and then do something with what the monk said.

The pattern is three steps:

```
1. Your trigger fires
2. zen_event → summary_force (component: your_component)
3. Wait for the kata write → read it → act on it
```

That's it. The Scheduler picks up the `summary_force` event and routes it to the Ninja
Summarizer, which runs the monk and writes the kata drawer. You wait for the write, then
read the kata and branch on the result.

---

## Step 1 — Wire the Trigger

Your automation fires on whatever condition you care about:

```yaml
automation:
  alias: Water Manager — Respond on Flow Stop
  triggers:
    - trigger: state
      entity_id: binary_sensor.water_flow_active
      to: "off"
  action:
    - action: zen_event
      data:
        kind: summary_force
        component: water_manager
        caller_token: "water_flow_stop_{{ now().isoformat() }}"
```

The `component` field must match the drawer key (slugified) in the Dojo cabinet — exactly
what you passed as `key` when you authored the KFC with Scribe.

`caller_token` is optional but useful. It threads through the Ninja Summarizer response and
the emitted event, so you can correlate which run was yours if multiple triggers fire close
together.

---

## Step 2 — Wait for the Kata Write

The Ninja Summarizer runs asynchronously. After firing `zen_event`, wait for the kata drawer
to update. The simplest approach is a short `wait_template` watching the kata cabinet:

```yaml
    - wait_template: >-
        {% set kata = state_attr('sensor.zen_kata_cabinet_resolved', 'variables') %}
        {% set drawer = kata.get('water_manager', {}) if kata is mapping else {} %}
        {% set ts = drawer.get('value', {}).get('ts', '') if drawer is mapping else '' %}
        {{ ts > states('input_text.water_flow_last_summary_ts') }}
      timeout:
        seconds: 60
      continue_on_timeout: true
```

This watches for a newer timestamp in the kata drawer than whatever you stored from the
previous run. Update `input_text.water_flow_last_summary_ts` after each successful read.

**Simpler alternative:** Just wait a fixed delay. If your component's `delay_seconds` is
30 and the AI task entity is responsive, 90 seconds is a safe upper bound for most installs:

```yaml
    - delay:
        seconds: 90
```

Not elegant, but reliable when you don't need precision.

---

## Step 3 — Read the Kata and Act

Once the wait completes, read the kata drawer via FileCabinet and branch on the monk's output:

```yaml
    - action: script.zen_sutra_filecabinet
      data:
        mode: get
        volume_entity_id:
          - "{{ states('sensor.zen_kata_cabinet_resolved') }}"
        key: water_manager
      response_variable: _kata_read

    - variables:
        _kata_val: >-
          {%- set v = _kata_read.get('value', {}) if _kata_read is mapping else {} -%}
          {{ v if v is mapping else (v | from_json if v is string else {}) }}
        _urgency: "{{ _kata_val.get('urgency', 'low') }}"
        _summary: "{{ _kata_val.get('summary', '') }}"

    - choose:
        - conditions: "{{ _urgency in ['high', 'critical'] }}"
          sequence:
            - action: script.zen_dojotools_utilities
              data:
                action_type: announce
                message: "{{ _summary }}"
```

The kata drawer structure is defined by `zen_template.value.structure` in the Kata cabinet.
The standard fields are `urgency`, `summary`, `attention`, `events`, and `ts`. Your
`component_instructions` in the KFC drawer guide the monk toward what to populate.

---

## The Full Automation

```yaml
automation:
  alias: Water Manager — Respond on Flow Stop
  mode: single
  triggers:
    - trigger: state
      entity_id: binary_sensor.water_flow_active
      to: "off"
  action:
    # 1. Fire the summarizer for this component
    - action: zen_event
      data:
        kind: summary_force
        component: water_manager
        caller_token: "water_flow_stop_{{ now().isoformat() }}"

    # 2. Wait for the kata to update (or timeout)
    - delay:
        seconds: 90

    # 3. Read kata result
    - action: script.zen_sutra_filecabinet
      data:
        mode: get
        volume_entity_id:
          - "{{ states('sensor.zen_kata_cabinet_resolved') }}"
        key: water_manager
      response_variable: _kata_read

    # 4. Unpack
    - variables:
        _kata_val: >-
          {%- set v = _kata_read.get('value', {}) if _kata_read is mapping else {} -%}
          {{ v if v is mapping else (v | from_json if v is string else {}) }}
        _urgency: "{{ _kata_val.get('urgency', 'low') }}"
        _summary: "{{ _kata_val.get('summary', '') }}"
        _attention: "{{ _kata_val.get('attention', []) }}"

    # 5. Act on urgency
    - choose:
        - conditions: "{{ _urgency in ['high', 'critical'] }}"
          sequence:
            - action: script.zen_dojotools_utilities
              data:
                action_type: announce
                message: "{{ _summary }}"
        - conditions: "{{ _urgency == 'medium' }}"
          sequence:
            - action: notify.mobile_app_default_user
              data:
                message: "{{ _summary }}"
      default:
        - action: script.zen_dojotools_event_emitter
          data:
            component: water_manager
            severity: info
            kind: flow_stop_nominal
            summary: "{{ _summary }}"
```

---

## Why `mode: single`

This automation fires on physical events. If flow stops and starts in rapid succession,
you do not want multiple instances accumulating. `mode: single` drops the second trigger
while the first run is still waiting. Use `mode: queued` if you want them to serialize
instead of drop.

---

## What You Need in the KFC

For this pattern to work well, your component's `component_instructions` should tell the
monk to produce actionable urgency levels and a concise summary the automation can use
directly:

```yaml
component_instructions: >
  Produce urgency (low/medium/high/critical), a one-sentence summary for announcement,
  and an attention list of active anomalies. Urgency is high if flow is detected in Away mode
  or sustained >30 minutes. Critical if >2 hours. Low if all nominal.
```

The better the instructions, the more useful the automation's branch logic becomes.

---

## Calling the Ninja Summarizer Directly (MCP Path)

If your automation is triggered by the agent itself (not a hardware sensor), you can call
the Ninja Summarizer directly and inspect the response inline — no wait required:

```yaml
    - action: script.zen_dojotools_ninja_summarizer
      data:
        kung_fu_component_id: water_manager
        post_to_kata_cabinet: true
        caller_token: "agent_flow_check_{{ now().isoformat() }}"
      response_variable: _ninja_result

    - variables:
        _monk_data: "{{ _ninja_result.monk.data | default({}) }}"
        _urgency: "{{ _monk_data.get('urgency', 'low') }}"
```

The response comes back synchronously when called as a script action. The kata write happens
inside the same call when `post_to_kata_cabinet: true`.

This is the cleanest path when you control the caller. The `zen_event` → Scheduler path is
better when you want the component to run on its own clock and you're reacting to real-world
events.

---

## What's Coming

The current pattern requires either a fixed wait or a polling loop to know when the kata
is ready. A future release is exploring an event emission path at the end of the pipeline —
a signal the summarizer fires after the kata write completes, which automations could listen
for directly instead of waiting on time.

This would make the trigger-wait-read pattern considerably cleaner, particularly for
time-sensitive physical events. Nothing is wired yet — there are architectural pieces that
need to land first — but it's the direction.

Until then, the fixed-delay approach works reliably on any install where the AI task entity
is local and responsive.

---

## Related Docs

- [Zen DojoTools Summarizers](zen_dojotools_summarizers_readme.md) — Ninja Summarizer + SuperSummary reference
- [Zen DojoTools Scheduler](zen_dojotools_scheduler_readme.md) — trigger IDs and force events
- [Zen DojoTools Scribe](zen_dojotools_scribe_readme.md) — authoring the KFC component this pattern fires against
- [Zen DojoTools FileCabinet](zen_dojotools_filecabinet_readme.md) — kata read path
