# Your First Alert

> **Version:** 2026.9.0 'Steel Magnolia' | **Last Updated:** Sep 2026

*The fastest way to prove ZenOS-AI can get your attention.*

---

You've finished first-run setup. Your AI knows the home, Flynn has brought the cabinets online, and now you want one simple thing: make ZenOS-AI raise an alert, show you where it lives, and clear it again.

In plain terms: something happens, ZenOS-AI decides it's worth telling you, and a notification shows up. That's it — this walkthrough just proves that loop works before you rely on it for anything real.

Under the hood, that loop looks like this (only worth knowing if you're curious — you don't need to understand this to follow the steps below):

```
alert_fire event -> AlertManager -> active alerts list -> notification
```

No `notify.admin_devices` group is required. No Ninja summary is required for this first test. The AlertManager automation is always listening for `zen_event` events, and the MCP-facing tool `zen_dojotools_alertmanager` gives your AI a simple way to fire, list, and clear alerts.

This is the smallest version of the larger ZenOS loop:

```text
something happens -> decide if it matters -> surface attention -> ask a human if needed -> clear, suppress, or escalate
```

Later, "something happens" might be a camera analysis, an AutoVac blocker, a water leak, or a utility fault. The same attention layer handles dedup and priority so the system does not shout twice about the same active condition.

The ideal first real-world version feels like this:

```text
camera/tool sees context
  -> component decides it may matter
  -> AlertManager creates one active alert
  -> Postman routes it using the person's profile
  -> the person answers: acknowledge, alarm/escalate, cancel, or "he's fine"
  -> the alert clears or becomes higher priority
```

For example: the back fence camera sees someone in a hat riding a lawn mower near the fence. The system should not jump straight to panic. A good governed alert is closer to: "I do not think this is an issue, but is this okay?" If JoeUser has quiet hours off and work-hours routing on, Postman can ask JoeUser through the configured channel. When JoeUser says "he's fine," the alert can be acknowledged or cleared instead of escalating.

```mermaid
flowchart LR
  Camera["Back fence camera context"]
  Component["Camera or Security component"]
  Gate{"Looks attention-worthy?"}
  AlertManager["AlertManager creates one active alert"]
  Drawer["_zen_active_alerts drawer"]
  Postman["Postman applies JoeUser profile"]
  Question["Ask: Is this okay?"]
  Choice{"JoeUser response"}
  Ack["Acknowledge or clear"]
  Alarm["Escalate alarm"]
  Cancel["Cancel or suppress"]

  Camera --> Component --> Gate
  Gate -- "No" --> Cancel
  Gate -- "Yes" --> AlertManager --> Drawer --> Postman --> Question --> Choice
  Choice -- "He's fine" --> Ack
  Choice -- "Alarm" --> Alarm
  Choice -- "Cancel" --> Cancel
```

---

## What AlertManager Does

AlertManager is a fire-once alert system.

When an alert fires, it stores the alert key in the household cabinet drawer `_zen_active_alerts`. If the same key fires again while it is still active, AlertManager suppresses the duplicate. When the condition clears, the key is removed and that same alert can fire again later.

Severity controls how much attention the alert gets:

| Severity | Notification | Priority context |
|---|---|---|
| `info` | Sent to the chosen target | No |
| `warn` | Sent to the chosen target | No |
| `error` | Sent to the chosen target | Yes — writes priority context for the AI |

By default, alerts expire after 24 hours. Use `clear_after_minutes: 0` only for alerts that should stay active until someone explicitly clears them.

---

## Step 1 — Confirm AlertManager Exists

After installing, AlertManager should be present as:

- `automation.zen_alert_manager`
- `script.zen_dojotools_alertmanager`
- `sensor.zen_priority_context`

Ask your AI:

> "List active alerts."

It should call `zen_dojotools_alertmanager` with `mode=list`. A clean system returns zero active alerts. If the tool is missing after first install, fully restart Home Assistant once. New script entities do not always appear in the conversation agent tool schema after a script reload.

You can also check directly in HA:

1. Open **Developer Tools -> States**
2. Search for `sensor.zen_priority_context`
3. It should usually read `clear`

---

## Step 2 — Fire a Test Alert

The easiest path is conversational:

> "Fire a test ZenOS alert called first_alert_test. Make it a warning and send it as a persistent notification."

Your AI should call `script.zen_dojotools_alertmanager` with these fields:

```yaml
mode: fire
alert_key: first_alert_test
message: "ZenOS first alert test."
severity: warn
notify_target: persistent
clear_after_minutes: 30
```

If you are calling it yourself from **Developer Tools -> Actions**, use:

```yaml
action: script.zen_dojotools_alertmanager
data:
  mode: fire
  alert_key: first_alert_test
  message: "ZenOS first alert test."
  severity: warn
  notify_target: persistent
  clear_after_minutes: 30
```

What should happen:

- HA shows a persistent notification
- `_zen_active_alerts` gets a `first_alert_test` entry
- `sensor.zen_priority_context` stays `clear` because this is only a warning

If you prefer Developer Tools, fire the event directly:

```yaml
event: zen_event
event_data:
  event:
    kind: alert_fire
    alert_key: first_alert_test
    message: "ZenOS first alert test."
    severity: warn
    notify_target: persistent
    clear_after_minutes: 30
```

---

## Step 3 — Prove Dedup Works

Fire the exact same alert again:

> "Fire first_alert_test again."

You should not get a second notification. That is the point. AlertManager treats `alert_key` as the dedup key, so a still-active key is a no-op.

This behavior is what keeps a noisy sensor from sending the same alert over and over while the underlying condition remains true.

---

## Step 4 — List Active Alerts

Ask:

> "List active alerts."

The result should include:

- `alert_key: first_alert_test`
- `message`
- `severity: warn`
- `fired_at`
- `expires_at`
- `age_min`
- `in_priority_inject: false`

This comes from the household cabinet drawer `_zen_active_alerts`. You normally do not need to read that drawer yourself; the tool is the safer front door.

---

## Step 5 — Try an Error Alert

Now test the priority context path:

> "Fire a test error alert called first_error_test. Use persistent notification."

Expected tool fields:

```yaml
mode: fire
alert_key: first_error_test
message: "ZenOS first error alert test."
severity: error
notify_target: persistent
clear_after_minutes: 30
```

Developer Tools -> Actions form:

```yaml
action: script.zen_dojotools_alertmanager
data:
  mode: fire
  alert_key: first_error_test
  message: "ZenOS first error alert test."
  severity: error
  notify_target: persistent
  clear_after_minutes: 30
```

What should change:

- HA shows a persistent notification
- `_zen_active_alerts` gets `first_error_test`
- `_zen_priority_inject` gets a matching priority entry
- `sensor.zen_priority_context` changes to `active`

This is the part your AI sees in its context frame. Error alerts are not just notifications; they become situational awareness.

---

## Step 6 — Clear the Alerts

Clear each test alert:

> "Clear first_alert_test and first_error_test."

Expected tool fields:

```yaml
mode: clear
alert_key: first_alert_test
```

```yaml
mode: clear
alert_key: first_error_test
```

Then ask:

> "List active alerts."

You should see no active alerts, and `sensor.zen_priority_context` should return to `clear`.

If you are testing and want to wipe all active alerts:

```yaml
action: script.zen_dojotools_alertmanager
data:
  mode: clear_all
```

Use `clear_all` carefully on a live home, since it clears real alerts too.

---

## Step 7 — Connect a Real Device (required before certifications work)

Everything above used `notify_target: persistent` deliberately — it proves the pipeline with zero setup. But a persistent HA notification is not enough for what's coming next: **certifications** (locks, covers, alarm, infra, room overrides — see the [Security & Certification manual](security_certification_manual.md)) require a real push notification reaching a real phone, every single time, with no fallback. If you did [Install Step 3.5](install.md#step-35--set-up-your-mobile-notification-path), your phone can already receive a notification — this step connects that to Postman so ZenOS can actually route one to you.

Seed the minimum needed — your own user profile, with `push_targets` pointing at the notify service from Install Step 3.5:

```yaml
action: script.zen_dojotools_postman
data:
  mode: author_policy
  scope_id: sensor.zen_default_user_cabinet_resolved
  policy_key: postman_profile
  policy_type: update_existing
  channel_definition:
    push_targets:
      - Admin Devices        # must match a notify.* target name you can reach —
                              # see the table below for how this resolves
    urgency_tiers:
      low:      { channels: [push] }
      medium:   { channels: [push] }
      high:     { channels: [push, tts] }
      critical: { channels: [push, tts] }
    away_policy: { push: allow, tts: block }
```

**`push_targets` is the field that matters.** Postman turns the value into a `notify.*` service name:

| `push_targets` value | Dispatches to |
|---|---|
| `Admin Devices` | `notify.admin_devices` |
| `Default User Phone` | `notify.default_user` |
| *(anything else)* | `notify.<slugified value>` |

If your notify service from Step 3.5 was `notify.mobile_app_johns_iphone` and you don't already have an `admin_devices` group wrapping it, either create that group in HA (Settings → Devices & Services → Helpers → Notify Group), or set `push_targets` to whatever name resolves to your actual service.

Now fire a real test through Postman instead of the persistent path:

```yaml
action: script.zen_dojotools_alertmanager
data:
  mode: fire
  alert_key: first_postman_test
  message: "Testing Postman alert routing to a real device."
  severity: warn
  notify_target: postman
  channel_hint: push
```

**Expected:** a real push notification arrives on your phone. If it doesn't, run `zen_dojotools_postman mode=resolve target=person.<you> urgency=5 channel_hint=push` first — it tells you what *would* happen without dispatching, which is the fastest way to see whether the problem is the profile, the target name, or the device itself.

This one seed is enough for certifications to work (they bypass quiet-hours/work-hour gates deliberately, so the fuller household/family policy isn't required just for this). For the complete picture — routing by person, quiet hours, TTS, multiple devices — see **[Notification Routing Guide](notification_routing.md)** next.

---

## Notification Targets

For a first test, use `notify_target: persistent`. It works without extra setup and proves the AlertManager pipeline.

Other targets:

| Target | Use when |
|---|---|
| `persistent` | You want an HA persistent notification. Best first test. |
| `postman` | You have Postman profiles configured and want push, TTS, or Teams routing — see Step 7 above. |
| `mobile` | Legacy/simple mobile path if supported by your install. |

If the persistent test works but Postman does not, the problem is in Postman/profile routing, not AlertManager.

---

## Turning Real Conditions Into Alerts

The test above proves the alert pipeline. Real sensors still need an automation, KFC, or tool to decide when to fire and clear alerts.

For a simple smoke detector example:

```yaml
automation:
  - id: zenos_smoke_alert_example
    alias: ZenOS Smoke Alert Example
    triggers:
      - trigger: state
        entity_id: binary_sensor.your_smoke_detector
        to: "on"
        for:
          seconds: 5
    actions:
      - event: zen_event
        event_data:
          event:
            kind: alert_fire
            alert_key: smoke_detector_alarm
            message: "Smoke detector is alarming."
            severity: error
            notify_target: persistent
            clear_after_minutes: 0

  - id: zenos_smoke_clear_example
    alias: ZenOS Smoke Clear Example
    triggers:
      - trigger: state
        entity_id: binary_sensor.your_smoke_detector
        to: "off"
        for:
          seconds: 10
    actions:
      - event: zen_event
        event_data:
          event:
            kind: alert_clear
            alert_key: smoke_detector_alarm
```

Replace the entity ID before using this. Do not copy a safety automation blindly; test it with a harmless binary sensor first.

The older `alert_manager` KFC and `alert_when_*` labels still describe one summarizer-driven monitoring pattern, but they are not required for the direct AlertManager test in this guide.

---

## Troubleshooting

**I fired the same alert twice and only got one notification.**
Good. That means dedup is working. Clear the alert before firing it again.

**`sensor.zen_priority_context` did not change for a warning.**
Correct. Only `severity: error` writes priority context.

**The tool says queued but I do not see the alert immediately.**
The tool emits an event, and the automation handles the event asynchronously. Wait a moment, then run `mode=list`.

**The tool is missing from my AI's tool list.**
Fully restart Home Assistant once. New script entities may not appear in MCP/tool schemas after script reload alone.

**Persistent notification works, but phone push does not.**
AlertManager is working. Check Postman routing, mobile app notify services, and any sleep/away gates.

---

## What's Next

You've just seen the current alert path:

```
event or tool -> AlertManager -> active alert drawer -> notification -> optional AI priority context
```

If you completed Step 7 above, your household and user cabinets already have a working push path — certifications will work when you need them. Next reads:

- **[Notification Routing Guide](notification_routing.md)** — the full household/family policy (quiet hours, work hours, multiple people) beyond the one-user seed from Step 7
- **[AlertManager reference](../components/alertmanager.md)** — full current behavior, TTL, priority context, and tool modes
- **[Understanding KF4](../kung_fu/understanding_kf4.md)** — how summarizer-driven components fit beside direct event alerts
- **[Entity Exposure](entity_exposure.md)** — what to expose to your AI and what to keep behind tools

---

-> **[Back to Getting Started](readme.md)**
