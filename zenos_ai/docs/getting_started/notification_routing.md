# ZenOS-AI: Notification Routing Guide

> **Version:** 2026.9.0 'Steel Magnolia' | **Last Updated:** Sep 2026

---

## Why `notify.admin_devices` Alone Isn't Enough

**If you did [First Alert's Step 7](first_alert.md#step-7--connect-a-real-device-required-before-certifications-work), your user profile already has a working push path** — certifications and personal alerts work. This doc covers the rest: the household-wide policy (life-safety bypass, quiet hours), routing to more than one person, and troubleshooting if something still isn't arriving.

ZenOS-AI doesn't call your `notify.*` services directly. All notifications — alerts, summaries, responses — route through `zen_dojotools_postman`, the household communications layer. Postman reads a **policy** from your cabinets before dispatching, and that policy tells it which `notify.*` service to use and when.

If you've created `notify.admin_devices` and it works from Developer Tools but ZenOS isn't delivering notifications, the policy hasn't been seeded yet.

---

## The Delivery Chain

```
alert_manager (KFC component)
  → writes kata when conditions match
  → fires zen_event: kind=alert_fire
    → KF4 action pipeline
      → zen_dojotools_postman mode: resolve_and_dispatch
        → reads postman_profile from household, family, user cabinets
          → dispatches to notify.<your-service>
```

The policy is read at dispatch time. No policy = no dispatch.

---

## One-Time Setup: Seed Your Postman Profile

Run these three `author_policy` calls once. They're idempotent — safe to re-run to update. **If you already ran First Alert's Step 7, Step 2 below is done** — re-running it with the same values is harmless, or skip straight to Steps 1 and 3.

### Step 1 — Household Policy (sleep gate + life safety)

```
zen_dojotools_postman
  mode: author_policy
  scope_id: sensor.zen_default_household_cabinet_resolved
  policy_key: postman_profile
  policy_type: update_existing
  channel_definition:
    life_safety_bypass: 9
    sleep_gate:
      block_below_urgency: 9
    work_gate:
      block_tts: false
```

This sets the house ceiling — urgency 9+ always gets through regardless of sleep/work mode.

### Step 2 — User Policy (which devices, which channels)

```
zen_dojotools_postman
  mode: author_policy
  scope_id: sensor.zen_default_user_cabinet_resolved
  policy_key: postman_profile
  policy_type: update_existing
  channel_definition:
    push_targets:
      - Admin Devices          # → notify.admin_devices
    urgency_tiers:
      low:
        channels: [push]
      medium:
        channels: [push]
      high:
        channels: [push, tts]
      critical:
        channels: [push, tts]
    away_policy:
      push: allow
      tts: block
```

**`push_targets` is the key field.** The value must match a notification target name exactly. Postman derives the `notify.*` slug from it:

| push_targets value | Dispatches to |
|---|---|
| `Admin Devices` | `notify.admin_devices` |
| `Family Devices` | `notify.family_devices` |
| `Default User Phone` | `notify.default_user` |
| `Secondary User Phone` | `notify.secondary_user` |

If you followed [First Alert](first_alert.md) and created `notify.admin_devices`, use `"Admin Devices"` here.

### Step 3 — Family Policy (seed empty for now)

```
zen_dojotools_postman
  mode: author_policy
  scope_id: sensor.zen_default_family_cabinet_resolved
  policy_key: postman_profile
  policy_type: update_existing
  channel_definition: {}
```

Family escalation isn't implemented yet — an empty seed satisfies the resolver.

---

## Urgency Tiers

Postman maps urgency (1–10) to tiers:

| Urgency | Tier | Default channels |
|---|---|---|
| 1–3 | low | push |
| 4–6 | medium | push |
| 7–8 | high | push + TTS |
| 9–10 | critical | push + TTS (bypasses all gates if ≥ `life_safety_bypass`) |

Alert Manager fires at `urgency: 7` (error severity) by default. That maps to `high` → push + TTS.

---

## Testing End-to-End

After seeding, test the full chain from Developer Tools → Services:

```
zen_dojotools_postman
  mode: resolve_and_dispatch
  urgency: 5
  title: Postman Test
  message: If you see this, routing is working.
```

Check the notification arrives on your target device. If it doesn't:

1. Run `mode: resolve` first — it returns what would happen without dispatching. Check `tier`, `channels`, `push_targets`, and `resolved_push_service`.
2. Verify `notify.admin_devices` (or your target service) works via Developer Tools → Actions directly.
3. Check `sensor.zen_default_user_cabinet_resolved` has a `postman_profile` drawer with `push_targets` set.

---

## Common Failures

| Symptom | Cause | Fix |
|---|---|---|
| ZenOS doesn't deliver notifications, but `notify.admin_devices` works directly | postman_profile not seeded — no policy, no dispatch | Run the three `author_policy` calls above |
| Notifications delivered to wrong device | `push_targets` pointing at wrong notify group | Update user postman_profile with correct target name |
| TTS fires but push doesn't (or vice versa) | urgency tier channel list | Update `urgency_tiers` in user postman_profile |
| Nothing delivered during night mode | Sleep gate blocking | Lower `sleep_gate.block_below_urgency` or set `breakthrough: true` for critical alerts |
| `zen_dojotools_notification_router` calls still in your config | That tool is deprecated | Replace with `zen_dojotools_postman mode: resolve_and_dispatch` |

---

## Related

- [First Alert Guide](first_alert.md) — wiring up alert_manager end-to-end
- [DojoTools Postman Reference](../scripts/zen_dojotools_postman_readme.md) — full field reference
