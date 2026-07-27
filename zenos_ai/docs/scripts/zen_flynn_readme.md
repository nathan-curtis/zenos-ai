# Flynn — Stepgate Sentinel & Bootstrap Engine — 4.5.6

*ZenOS-AI's boot guard, initializer, and onboarding driver*

---

## Overview

Flynn is the first thing that runs and the last line of defense before ZenOS-AI hands off to Friday.

On every HA startup — and whenever a health sensor changes state — Flynn walks through a **sequential stepgate sequence**. Each gate checks one precondition. If it passes, Flynn advances. If it fails, Flynn repairs what it can, notifies you, and stops. The next health sensor update re-engages the sequence.

Flynn also drives **OOBE** — the first-boot configuration flow that names your AI, seeds your household profile, and stamps the system ready.

Nothing inside ZenOS-AI runs until Flynn says it can.

---

## What Flynn Monitors

Flynn re-runs any time these sensors change state:

| Sensor | What It Watches |
|---|---|
| `sensor.zen_label_health` | Required labels exist and are assigned to entities |
| `sensor.zen_cabinet_health` | Required cabinet entities are present and initialized |
| `sensor.zen_monastery_health` | Schema templates + cabinet content are seeded |

States that re-engage Flynn: `critical`, `error`, `warn`, `ok`

Flynn also re-runs on these events:

| Event | When |
|---|---|
| `zen_event kind: warmup_expired` | 5 min after `ha_start` — warmup window closed, re-evaluate all gates |
| `zen_event kind: cabinet_mounted` | Cabinet mounted — gate 2.5 may now clear |
| `zen_event kind: cabinet_dismounted` | Cabinet dismounted — gate 2.5 may re-engage |

The `warmup_expired` event is fired by `automation.zen_warmup_timer` (also in `flynn.yaml`). It guarantees the warmup gate-down is a real signal, not a side-effect of monastery settling. If monastery never settles (real problem), Flynn will catch it at the 5-min mark rather than leaving sensors stuck at `warmup` indefinitely.

---

## Early Exit

Flynn skips all gates and exits immediately if **all** of the following are true:

- `binary_sensor.flynn_system_ready` is `on`
- All health sensors are `ok` (monastery and cabinet may be `warn` — both are non-blocking)
- OOBE is not pending
- Both `kfc_template` (Dojo) and `zen_template` (Kata) are already seeded
- No cabinets are in `init` state (checked directly via `slot_to_default_entity` — not the health rollup)

The init-state check is evaluated separately from `zen_cabinet_health`. Optional cabinet slots can appear healthy in the rollup even when their underlying entity is still in `init`. Flynn reads `slot_to_default_entity` directly to catch this case.

If the system is already clean and stable, Flynn steps aside.

---

## The Stepgate Sequence

### Gate 0 — Label Creation

**Trigger:** `sensor.zen_label_health == critical`

Labels don't exist yet. Flynn calls DojoTools Labels to create the full required label set, then fires a persistent notification telling you to restart HA. Label creation requires a restart to take effect.

**What you'll see:** Notification — *"Labels Created — Restart Required"*

**What to do:** Restart Home Assistant.

---

### Gate 1 — Label Assignment

**Trigger:** `sensor.zen_label_health == warn`

Labels exist but aren't assigned to the right cabinet entities. Flynn calls `script.flynn_assign_labels` to wire each cabinet sensor to its required labels. Three passes:

1. All slot entities → `zen_cabinet` parent label
2. Each slot → its specific slot label (e.g., `zen_dojo_cabinet`, `zen_kata_cabinet`)
3. Broad labels → default household, family, user, AI user cabinet entities

Once complete, Flynn stops and waits for `zen_label_health` to update.

**Event fired:** `flynn_stepgate_event` (gate: 1, action: labels_assigned)

---

### Gate 2 — Cabinet Initialization

**Trigger:** `sensor.zen_cabinet_health` is not `ok`

Gate 2 behavior depends on the severity of the cabinet health state. Three cases:

**Case A — `error` or `critical` (hard stop)**

One or more required cabinet entities are uninitialized or unavailable. Flynn fires a persistent notification ("Cabinet Check Needed") and stops. Operator action required.

**What to do:** Open a conversation and say "initialize my cabinets", or run `script.zen_admintools_cabinetadmin` with `mode: initialize` directly. Check `sensor.zen_cabinet_health` → `missing_cabinets` to see which slots are affected.

**Case B — `warn`, outside warmup window, not `ha_start` (notify + continue)**

Legacy schema detected — one or more cabinets are on the pre-4.5 schema (state: `Variables`). The system is **fully operational**. Flynn fires a non-urgent notification ("Cabinet Upgrade Available") and continues — no stop.

**What to do:** When convenient, open a conversation and say "upgrade my cabinets". No urgency — nothing is broken.

**Case C — `warn`, inside warmup window OR triggered by `ha_start` (log only, continue)**

Same legacy schema condition but detected during the boot warmup window (≤5 min after last boot) or on `ha_start`. Flynn logs it and **continues** — no notification, no stop. The warmup window exists to avoid notification noise on normal restarts.

Cabinet types Flynn can initialize on demand (Case A):

```
system, dojo, kata, default_household, default_family,
default_user, default_ai_user, history, index
```

**Event fired:** `flynn_stepgate_event` (gate: 2, action: cabinets_initialized) — Case A only.

---

### Gate 2.1 — Virgin Cabinet Auto-Init

**Trigger:** One or more cabinets in `init` state, post-warmup, not `ha_start`

Detects cabinets that are present but genuinely uninitialized — `state=init` with no VolumeInfo. These are virgin cabinets: the sensor has evaluated at least once and found nothing. Flynn calls `script.flynn_initialize_cabinets` to bootstrap them.

**Why this gate exists:** `zen_cabinet_health` rolls up optional slots as healthy even when their underlying entity is still in `init`. New installs were deadlocking — present-but-uninitialized cabinets never appeared in `missing_cabinets`, so Gate 2 never saw them, and nothing ever called cabinetadmin. Gate 2.1 catches them by reading `slot_to_default_entity` directly and checking entity state.

**Safety guards:**
- **Warmup / `ha_start` skip** — during the boot warmup window, the recorder may not have restored variables yet. A live cabinet can transiently show `state=init` before its first re-evaluation completes. Gate 2.1 is skipped entirely until warmup expires. After that, `init` = genuinely virgin.
- **GUID gate** — `flynn_initialize_cabinets` checks VolumeInfo before touching any cabinet. Any cabinet that already has data is silently skipped. Live data is never at risk.

**What you'll see:** No notification — this is silent infrastructure. Cabinets self-initialize and Flynn continues to Gate 3.

**Event fired:** Handled internally by `script.flynn_initialize_cabinets`.

**History cabinet mount lifecycle (2026.7.1):** Gate 2.1's re-init trigger now also fires when any of the 4 history cabinets (`default_household_history`, `default_family_history`, `default_user_history`, `default_ai_user_history`) is `online_unmounted` — mounted but not yet wired to its parent cabinet's `history` mount point. When that happens, Flynn: waits for the cabinet's state to stabilize (bounded `wait_template`s, each with `continue_on_timeout: true`), calls `mount_cabinet`, extracts the mounted cabinet's GUID from its `VolumeInfo` (guarded against the value being a JSON string vs. an already-parsed mapping), then calls `set_mount` to wire that GUID into the parent's `history` mount point — skipping if already wired. If a cabinet has residue data that blocks auto-init (`reason: rearm_required`), Flynn posts a persistent notification asking a human to run `repair_volumeinfo` or `hammer` manually rather than guessing. This whole path degrades gracefully (`continue_on_error: true` throughout) rather than blocking boot — a stuck history cabinet delays its own mount wiring, not the rest of the system, and Gate 2.1 re-evaluates it on the same 5-minute Stepgate Sentinel cadence as everything else (see [Troubleshooting](#troubleshooting) below), not a tight retry loop.

---

### Gate 3 — Schema Seeding + Content Bootstrap

**Trigger:** `sensor.zen_monastery_health` is not fully settled, OR schema templates are missing

Gate 3 has three distinct sub-steps:

#### 3a — Resolver Stability Check

Before doing anything, Flynn checks that the three cabinet resolver sensors have settled:

- `sensor.zen_dojo_cabinet_resolved`
- `sensor.zen_kata_cabinet_resolved`
- `sensor.zen_system_cabinet_resolved`

If any resolver is `unknown`, `unavailable`, or empty — Flynn waits. Label assignment from Gate 1 needs time to propagate.

#### 3b — Schema Seed (always runs)

Regardless of monastery health state, Flynn checks whether:
- Dojo cabinet has a `kfc_template` drawer
- Kata cabinet has a `zen_template` drawer

If either is missing, Flynn calls `zen_admintools_reset_template` to press both templates. This is infrastructure — it runs on every boot if needed and is fully idempotent.

#### 3c — Content Bootstrap (critical only)

If `zen_monastery_health == critical`, Flynn calls `script.flynn_bootstrap_content` with your configured helper values:

- `input_text.zenos_conversation_agent`
- `input_text.zenos_household_name`
- `input_text.zenos_primary_user`
- `input_text.zenos_persona_name`

> `warn` state = degraded but functional. Schema seed runs (3b), content bootstrap does not. System continues.

**Bootstrap does the following:**

1. **Validates conversation agent** — if missing or unavailable, stops with notification. Nothing else proceeds without an agent.
2. **Auto-resolves reasoning task** — reads `input_text.zenos_reasoning_task`. If unset and exactly one conversation entity exists, auto-writes it. Multiple found with none configured → stops with notification.
3. **Auto-resolves AI task entity** — reads `input_text.zenos_ai_task_entity`. Same auto-resolve logic. Zero found = non-fatal (summarizer degrades gracefully).
4. **Seeds household name** — non-destructive write to `_household_name` drawer.
5. **Seeds AI persona essence** — writes default legacy-schema essence with `identity.name = 'your AI'` as placeholder (non-destructive — skips if drawer already exists). Three-layer cabinets already have a real name stamped at mint; this step is a no-op for them.
6. **Seeds schema templates** — calls `reset_template` if either template drawer is missing.
7. **Loads prompt substrate** — calls `zen_admintools_zenos_prompt_loader` (Cortex, Directives, Purpose).
8. **Wires default family graph** *(4.5.6)* — calls `household_add_family` for the default family cabinet, then `family_add_member` for the default user and AI user. All three calls are guarded (skips if cabinet unavailable) and idempotent (no-op if already wired). Closes a gap where cold builds left the default family as an orphan in the identity graph.
9. **Seeds home mode timers** *(4.5.6)* — each `input_datetime` schedule anchor and quiet/work bounds helper is individually checked and seeded to its default time if still bare (`''`, `unknown`, or `unavailable`). Each timer has its own guard — partially-configured installs are safe.
10. **Flags completion** — calls `flynn_unified_engine` with `action_type: flag_complete`.

**Event fired:** `flynn_stepgate_event` (gate: 3, action: bootstrap_complete)

---

### Gate 3.5 — OOBE

Runs after Gate 3, with a 10-second delay to let Gate 3 writes settle.

**OOBE is pending when:**
- The AI user cabinet's persona name resolves to `''` or `'your AI'` — checked via `essence_probe()` which handles both legacy (`identity.name`) and three-layer (`jacket.name`) schemas
- AND no `_oobe_complete` flag exists in the cabinet

**Pending name values:** `''`, `'your AI'`, `'unknown'` (all treated as placeholder — system will prompt for a real name)

Three cases:

| Situation | What Flynn Does |
|---|---|
| Input helper has a name + cabinet has placeholder | Writes persona name to essence. Dismisses notification. |
| Input helper is empty + cabinet has placeholder | Creates "Welcome — Let's name your AI" notification. Directs to conversation or Agent Builder. |
| Cabinet already has a real name OR `_oobe_complete` flag | Dismisses any pending OOBE notification. Nothing to do. |

The fastest path through OOBE: set `input_text.zenos_persona_name` in Settings → Helpers, then restart or let Flynn re-run. Flynn will write the name and clear the gate.

The conversational path: start chatting with your AI agent — Flynn's Agent Builder (mode: oobe) walks through the Q&A setup interactively.

---

### Gate 4 — System Ready

**Trigger:** All health sensors green + OOBE complete + monastery and agent health confirmed

Fires **once on transition** (not on subsequent green re-runs). Before creating the notification, Flynn verifies that `zen_monastery_health` is `ok` or `warn` and `zen_agent_health` is `ok` or `warn`, and that no init-state cabinets remain. Reaching Gate 4 is necessary but not sufficient — all three conditions must hold.

Creates a persistent notification:

> *"ZenOS-AI: System Ready — Everything's looking good. All gates green. Handing off to the real talent."*

---

## Agent Builder (`script.flynn_build_agent`)

An interactive configuration tool for setting up the system through conversation. **MCP-exposed.**

| Mode | What It Does |
|---|---|
| `help` | Returns setup instructions and current state. Auto-fires on first call. |
| `status` | Read-only: returns current persona name, household name, primary user. |
| `build` | Non-destructive writes: household name, persona name, primary user, family members. |
| `oobe` | Delegates to the full OOBE protocol. |

All `build` writes are non-destructive — existing values are never overwritten.

---

## Companion Selects

Four `template: select` entities provide UI-level dropdowns for the critical input helpers. Each select reads the current value of its corresponding `input_text` and writes back to it on change. The `input_text` entities remain the canonical source of truth — selects are overlay controls only.

| Select Entity | Drives | Options Source |
|---|---|---|
| `select.zenos_persona` | `input_text.zenos_persona_name` | AI user cabinet essence names |
| `select.zenos_primary_user` | `input_text.zenos_primary_user` | `person.*` with `user_id` attr (login-bound only) |
| `select.zenos_conversation_agent` | `input_text.zenos_conversation_agent` | All `conversation.*` domain entities |
| `select.zenos_ai_task` | `input_text.zenos_ai_task_entity` | All `ai_task.*` domain entities |

Options resolve dynamically at render time. The persona select only shows personas with a valid name in their essence — `unknown`, `your AI`, and empty strings are excluded.

**Fastest setup path:** Use these selects from Settings → Helpers instead of typing entity IDs manually.

---

## Health Sensor → Gate Map

| Sensor State | Gate Triggered | Action |
|---|---|---|
| `zen_label_health: critical` | Gate 0 | Create labels, notify, stop |
| `zen_label_health: warn` | Gate 1 | Assign labels to entities |
| `zen_cabinet_health: error/critical` | Gate 2 (hard stop) | Initialize missing cabinets — operator action required |
| `zen_cabinet_health: warn` (outside warmup) | Gate 2 (non-blocking) | Schema upgrade notification — system continues |
| `zen_cabinet_health: warn` (warmup/ha_start) | Gate 2 (log only) | Logged, system continues — no notification |
| Init-state cabs detected (post-warmup) | Gate 2.1 (silent, auto) | Auto-initialize virgin cabinets via `flynn_initialize_cabinets` |
| History cabinet `online_unmounted` (post-warmup) | Gate 2.1 (silent, auto) | Mount + wire GUID into parent's `history` mount point; notifies human only if residue blocks auto-init |
| `zen_monastery_health: critical` | Gate 3 | Full content bootstrap |
| `zen_monastery_health: warn` | Gate 3 (partial) | Schema seed only |
| All green + monastery/agent confirmed + OOBE complete | Gate 4 | System ready notification |

---

## Troubleshooting

### Stuck at Gate 0

`zen_label_health: critical` — labels haven't been created yet, or HA hasn't restarted after creation.

**Fix:** Check for the "Labels Created" notification. Restart HA.

---

### Stuck at Gate 1

`zen_label_health: warn` — labels exist but aren't assigned. Flynn should self-resolve on the next trigger. If it doesn't:

**Check:** Developer Tools → States → `sensor.zen_label_health` attributes. Look at `required_labels` vs what's assigned.

**Manual fix:** Run `script.flynn_assign_labels` directly from Developer Tools → Services.

---

### Stuck at Gate 2 — Cabinet Check Needed

`zen_cabinet_health: error or critical` — one or more cabinets uninitialized or unavailable.

**Check:** `sensor.zen_cabinet_health` attributes → `missing_cabinets` list.

**Manual fix:** Run `script.zen_admintools_cabinetadmin` with `mode: initialize` and the specific cabinet entity.

---

### Gate 2 — Cabinet Upgrade Available notification

`zen_cabinet_health: warn` — this is **not a stuck gate**. The system is fully operational. One or more cabinets are on legacy schema (state: `Variables`).

**What to do:** Open a conversation and say "upgrade my cabinets" when convenient. Or dismiss the notification and proceed — nothing will break.

If you keep seeing this notification on every non-warmup restart, it means the cabinet hasn't been touched since the upgrade notification first appeared. The upgrade is one-way and non-destructive — data is intact.

---

### Stuck at Gate 3 — "Conversation Agent Required"

Flynn stopped in `bootstrap_content` because `input_text.zenos_conversation_agent` is empty or the entity is unavailable.

**Fix:** Settings → Helpers → use the **ZenOS: Conversation Agent** select — it shows all available `conversation.*` entities as a dropdown. Or set `zenos_conversation_agent` manually. Verify the entity is available in Developer Tools.

---

### Stuck at Gate 3 — "Reasoning Task / AI Task Entity Required"

Multiple conversation or ai_task entities found and neither helper is configured.

**Fix:** Settings → Helpers → set `zenos_reasoning_task` or `zenos_ai_task_entity` to the correct entity ID explicitly.

---

### OOBE Notification Won't Dismiss

`binary_sensor.flynn_system_ready` is likely still off. Check upstream health sensors — if anything is warn/error, Flynn won't reach Gate 3.5 to dismiss it.

Alternatively: set `input_text.zenos_persona_name` to your desired AI name and trigger a restart or manually run the Stepgate Sentinel.

---

### `dojo_loaded: warn` in logs

Resolver sensors haven't settled after label assignment. Flynn gates on resolver state in Gate 3a — this is expected briefly after a restart. Give it 30–60 seconds. If it persists, check that `zen_dojotools_labels` created the label and that `zenos_dojo_cabinet` entity exists.

---

## Flynn as Prompt Fallback

Flynn is not just a boot guard — he is the fallback persona when the system cannot build a valid prompt. `render_prompt()` in `zen_os_1.jinja` checks for Flynn conditions before assembling Friday's prompt.

### Mode Detection Chain

`render_prompt(ai_entity)` evaluates these conditions in order:

| Priority | Condition | `_flynn_reason` |
|----------|-----------|-----------------|
| 1 | `input_boolean.zen_flynn_override` is `on` | `override` |
| 2 | `ai_entity` is blank or whitespace | `blank_persona` |
| 3 | persona label explicitly set to `flynn` | `explicit` |
| 4 | `identity_resolve_source()` returns error | error key from resolver |

If any condition matches, `prompt_system_flynn()` is returned — a hardcoded system prompt with zero cabinet dependencies.

### input_boolean.zen_flynn_override

Defined in `flynn.yaml`. Auto-creates on HA reload. Toggle this on to force Flynn mode for any conversation regardless of the selected persona — useful for maintenance, debugging, and testing without changing the persona selector.

Default state: `off`. Safe to leave off indefinitely.

### input_boolean.zenos_sim_mode_override

Defined in `flynn.yaml`. **Default state: `on`.** Controls the OS-level SP1 identity policy switch — `integrations_config.identity.sim_mode_allowed` (read by `zen_dojotools_identity` `resolve_caller_identity`, the platform-wide identity/cert chokepoint).

The automation **"Flynn: SP1 sim_mode Override — stamp every boot"** (`flynn_sim_mode_override_stamp`) writes `sim_mode_allowed` in the household cabinet to match this toggle on every HA startup and immediately on any manual flip of the toggle. This is an idempotent, reversible override: while real Authentik/OIDC (SP1) isn't live yet, leaving the toggle `on` keeps simulated identity resolution allowed platform-wide so tools that gate on `resolve_caller_identity`'s `policy_status` aren't silently blocked.

Flip the toggle `off` (or remove/disable the automation) to fall back to the fail-closed default (`sim_mode_allowed: false`) once real identity resolution is live.

### prompt_system_flynn()

Hardcoded system prompt — no Jinja cabinet reads. Works on a bare install, works when cabinets are corrupt, works when the identity resolver fails. Flynn will always answer.

### active_notification()

`active_notification()` macro in `flynn_onboarding.jinja` scans `persistent_notification.flynn_*` for active notifications and returns the highest-priority one with context label. Flynn's prompt reads this and acknowledges the active notification in its opening — giving coherent UX during first-boot and error states.

All four Flynn persistent notification messages are written in Flynn's voice and include an invitation to open the conversation agent.

---

## Events Reference

| Event | When | Payload |
|---|---|---|
| `flynn_stepgate_event` | After each gate action | `gate`, `action`, `timestamp` + gate-specific fields |
| `flynn_onboarding_updated` | Unified engine state changes | `type`, `drawer`, `key`, `timestamp` |

---

## Dependencies

| Dependency | Role |
|---|---|
| `sensor.zen_label_health` | Gate 0/1 trigger + label assignment data |
| `sensor.zen_cabinet_health` | Gate 2 trigger + missing cabinet list |
| `sensor.zen_monastery_health` | Gate 3 trigger |
| `script.zen_dojotools_labels` | Gate 0 label creation |
| `script.zen_admintools_cabinetadmin` | Gate 2 cabinet initialization |
| `script.zen_admintools_reset_template` | Gate 3 schema seed |
| `script.zen_admintools_zenos_prompt_loader` | Gate 3 prompt substrate |
| `script.zen_dojotools_filecabinet` | All cabinet reads and writes |
| `binary_sensor.flynn_system_ready` | Early exit gate |
| `input_boolean.zen_flynn_override` | Forces Flynn mode regardless of persona |
| `input_boolean.zenos_sim_mode_override` | Drives the SP1 sim_mode boot-stamp automation — controls `integrations_config.identity.sim_mode_allowed` |
| `zen_os_1.jinja` → `render_prompt()` | Calls `prompt_system_flynn()` when Flynn conditions met |
| `flynn_onboarding.jinja` → `active_notification()` | Surfaces active Flynn notifications in prompt |
| `input_text.zenos_conversation_agent` | Bootstrap validation |
| `input_text.zenos_persona_name` | OOBE persona name |
| `input_text.zenos_primary_user` | Bootstrap primary user seed |
| `input_text.zenos_reasoning_task` | Auto-resolved at bootstrap |
| `input_text.zenos_ai_task_entity` | Auto-resolved at bootstrap |
| `select.zenos_persona` | Companion UI for persona name |
| `select.zenos_primary_user` | Companion UI for primary user |
| `select.zenos_conversation_agent` | Companion UI for conversation agent |
| `select.zenos_ai_task` | Companion UI for AI task entity |
