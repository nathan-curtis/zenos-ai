# AutoVac Quick Start

> **Version:** 2026.9.0 'Steel Magnolia' | **Last Updated:** Sep 2026
>
> **Requires:** Basic install complete (see [Install Guide](install.md) and [First Run](first_run.md))
> **Time:** ~15 minutes
> **What you get:** Your AI tracking your robot vacuum's cleaning schedule, consumable wear, and room-by-room history — and alerting you before things become a problem.

---

AutoVac is ZenOS-AI's vacuum management component. Once set up, your AI knows which rooms need cleaning, when the brushes are getting tired, and whether the robot is behaving or just hiding under the couch.

Five steps. Start to finish. This is the fast path — it gets scheduled runs and briefings working, nothing more. If you also want Postman policy seeding, Grocy consumables tracking, wear-sensor alerting, and the full AlertManager test suite, use **[AutoVac First Setup](autovac_first_setup.md)** instead (or come back to it later — nothing here conflicts with it).

---

## Step 1 — Tag Your Vacuum Entity

AutoVac finds your vacuum through HA labels, not by entity ID. You'll tag the vacuum entity and a few optional sensors to unlock features.

**In Home Assistant:** Settings → Labels → create a label called `autovac` if it doesn't exist. Then go to your vacuum device, find its entity, and assign the `autovac` label.

**Before you can tag schedules:** You need at least one schedule entity to exist. In HA go to Settings → Helpers → + Create Helper → Schedule. Create one for each run window you want — e.g. `schedule.autovac_weekday_morning`, `schedule.autovac_weekday_evening`. Name them clearly; AutoVac finds them purely by label, so the name doesn't matter to the script — but you'll thank yourself later when you're looking at a list of helpers.

| Label | Entity to tag | Required? |
|---|---|---|
| `autovac` | Your robot vacuum entity (e.g. `vacuum.rosie`) | **Required** |
| `autovac_schedule` | Your schedule helper(s) for each run window | **Required** for scheduled runs |
| `autovac_dnd` | A time-based or input_boolean Do Not Disturb sensor | Optional |
| `autovac_water_low` | Low-water binary sensor (mop-capable robots) | Optional |
| `autovac_current_room` | Sensor that reports which room the robot is in | Optional |

If you're not sure which entity is the vacuum, go to Settings → Devices and find your robot. The primary entity is usually `vacuum.<name>`.

> Label objects are created in Step 2 if they don't exist yet. But the vacuum entity and schedule helpers must exist before you can assign them — HA can't label something that isn't there.

---

## Step 2 — Run Setup

Ask your AI:

> "Set up autovac"

Or with the explicit call:

> "Run autovac mode setup"

Your AI calls `zen_dojotools_autovac` in `mode=setup`. It scans your labels, checks what's wired, and tells you what it found and what's missing.

**If labels are missing**, ask:

> "Set up autovac and create any missing labels"

This runs `mode=setup` with `create_labels=true`, which creates the label objects in HA automatically. You'll still need to assign entities to them (Step 1), then run setup again.

**What a healthy first-run setup response looks like:**

```
vacuum entity: found (vacuum.rosie)
rooms: 0 configured
labels: all present
schedule_entities_found: 2
suggestions: add rooms via mode=configure (Step 3)
```

If setup reports missing entities or labels, fix those first before moving on. Setup is your go/no-go gate.

---

## Step 3 — Configure Your Rooms

AutoVac tracks cleaning history and scheduling per room. You need to tell it which rooms exist and how often each should be cleaned.

Ask your AI to add each room:

> "Configure autovac room living_room with days between cleans 5"

> "Configure autovac room kitchen with days between cleans 3, requires mop"

> "Configure autovac room office with days between cleans 7, enabled false"

Room slugs should match your HA area names (lowercase, underscores for spaces). If your HA area name differs from the slug, you can specify it:

> "Configure autovac room master_bedroom with area id Master Bedroom"

**Settings you can configure per room:**

| Setting | What it does | Default |
|---|---|---|
| `days_between` | How many days before this room is considered overdue | 7 |
| `enabled` | Whether this room is included in schedule elections | true |
| `requires_mop` | Mop this room if water tank is available | false |
| `segment` | Map segment ID for direct-to-room runs (Roborock/similar) | none |
| `area_id` | HA area name if it differs from the room slug | slug |

You can always check a room's current config:

> "Show me the autovac config for the kitchen"

And see all rooms at once:

> "Autovac status"

---

## Step 4 — Provision Consumables

This step connects your robot to your Grocy inventory so your AI can track brush and filter wear.

> **Note:** This requires Grocy to be set up and the inventory tool to be working. If you skipped the Grocy setup, you can come back to this step later — AutoVac works without it, but you won't get consumable alerts.

Ask your AI:

> "Provision autovac consumables for a [your model here]"

Your AI needs to know your robot's model to pick the right parts catalog. Available presets: `roborock_s7_plus`, `roborock_s7_maxv_ultra`, `roborock_s8_plus`, `roborock_s8_pro_ultra`, `roborock_q7_max_plus`, or `roborock_generic_dock` / `roborock_generic_dock_ultra` / `roborock_generic_dock_nomop` for unknown/generic models. If you're not sure, use the generic that matches your dock type.

Your AI registers the vacuum as an inventory asset in Grocy and creates a catalog of wear parts (main brush, HEPA filter, side brushes, mop pad, etc.) linked to the robot's wear sensors.

When it's done, ask:

> "What's the autovac consumables status?"

You should see each part with its current wear percentage and stock level. If anything is below the threshold, your AI will flag it and, if you're out of spares, add it to the Grocy shopping list automatically.

---

## Step 5 — Dry Run

Everything is wired. Let's confirm it's actually working.

Ask your AI:

> "Give me an autovac status report"

A healthy first report looks like this:

```
Vacuum: docked, battery 100%
Rooms: living room overdue (8 days), office ok, kitchen ok
Consumables: all healthy
Last run: yesterday — 3 rooms, 42 min
Next elected: living room
```

If the vacuum is currently cleaning, you'll see the active room and run duration. If consumables are low, you'll see which ones and by how much.

**If rooms show "overdue" immediately** — that's expected on first setup. The last-cleaned timestamps start at zero. Run a cleaning cycle and they'll update. You can also tell your AI when you last cleaned a room:

> "Tell autovac the living room was cleaned two days ago"

---

## What Happens Next

Once setup is complete, AutoVac runs automatically. About 30 minutes before each scheduled run, your AI sends a briefing — a TTS announcement and a push notification with your options:

| Button | What it does |
|--------|-------------|
| **"Go now!"** | Starts the vacuum immediately using the current elected rooms; the scheduled slot won't double-fire |
| **"Skip this run"** | Skips this run only — other scheduled runs today still happen |
| **"Pause all day"** | Pauses all scheduled runs until midnight; stops the robot if it's already running |

If you don't respond, the vacuum starts automatically when the schedule fires.

After each run, your AI:
- Analyzes the cleaning map (if a map camera is labeled `autovac`)
- Updates last-cleaned timestamps for each room
- Checks wear sensors against thresholds
- Refreshes the home context summary

To pause scheduling for a day (guests, baby sleeping, etc.):

> "Pause the vacuum today"

To send the robot to a specific room right now:

> "Send the vacuum to clean the kitchen"

---

## Troubleshooting

**"No vacuum entity found"**
The `autovac` label isn't assigned to your vacuum entity, or the entity isn't visible to HA. Go back to Step 1.

**"No rooms configured"**
You haven't added any rooms yet. Go back to Step 3.

**"Consumables not provisioned"**
Run Step 4. Until you do, consumable alerts won't fire.

**Setup passes but status shows wrong vacuum**
If you have more than one vacuum, make sure only the right one has the `autovac` label.

**Room slugs not matching**
Use `mode=configure room=<slug> config={"area_id": "<your_ha_area>"}` to tell AutoVac which HA area that room corresponds to.

**Briefing never arrives**
Confirm Postman is seeded and your push target resolves: `zen_dojotools_postman mode=resolve target=person.yourname urgency=4 channel_hint=push`. Expected: `would_dispatch: true`.

---

## Related

- [AutoVac First Setup](autovac_first_setup.md) — full commissioning guide with Postman policies, Grocy, and controller wiring
- [AutoVac Reference](../components/autovac.md) — complete mode and field reference
- [First Alert](first_alert.md) — if you haven't set up notifications yet, consumable alerts won't reach you
