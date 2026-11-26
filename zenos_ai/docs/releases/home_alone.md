# Release Notes — 2026.12.0 'Home Alone'

**Status:** Public beta 2
**ETA:** <!-- to be announced -->
**Branch:** `feat/2026.12.0`
**Base:** 2026.11.1 (patch on 2026.11.0 'This Is Spinal Tap')

### Betas

- **Beta 1** (2026-11-18): the release as first published.
- **Beta 2** (2026-11-25): Cortex 48 'True North', Autovac's `swap_robot`, mounts that carry their whole tree, the Plant water and electrical summaries, the Inventory stock-entry fix, the Abbot's new shape (requests instead of forces, its own lanes for maintenance and life-safety, pressure and recovery), stale summaries flagged, and Oracle as a preview with seven hosts: Room Manager, Weather, SpaMaster, Service Desk, Taskmaster and Plant's water and electrical.
- **Beta 3:** <!-- placeholder -->

<!-- Minimum Home Assistant version: to confirm. Room Manager's timers use the timer.finished trigger (about 2026.5); Spook needs 2026.11. -->

---

## Summary

The house gets left home alone.

Spinal Tap put locks on the cabinets and a pass in every agent's hand. Home Alone gives the house something to do with them. Security Manager grows from a panel reader into the house's security. It can tell you whether the house is ready to arm and keeps a record of what it did on its own. Once you turn them on, it also lights the house at dusk, lights the way in when you get home after dark, watches the garage doors, steps the alarm aside for the vacuum, and tells you when the range is on with nobody home. Climate learns which rooms each air handler serves and offers to pause the HVAC when a window's left open. A new Weather tool gives every other tool the forecast.

The tools come up to caps: each gets a scored summary, a read cert of its own, and a background summary the agent can rely on. Underneath, the house's background thinking gets a calmer head. The Abbot only decides what runs. Maintenance and life-safety have their own lanes, and a routine refresh is a request, never a force. And a preview, Oracle, lets a component ask whether a change is worth a fresh summary before it pays for one.

Every new behavior is off until you turn it on, and each one acts only on grants you approved.

---

## The Twelve Days

<!-- Placeholder: the twelve days of Christmas, one verse per tool, told as Friday's traps for the Wet Bandits. Written last. The Weather verse credits TheFes. -->

---

## Upgraded User Experiences

Each one is off until you turn it on, unless it says otherwise.

- **Leaving:** ask whether the house is ready to arm, and hear what's in the way: an open window, a jammed lock, a zone the panel says isn't ready. *(A read; always available.)*
- **Leaving, with the vacuum:** five minutes after everyone's gone, the vacuum cleans the rooms that are due.
- **Armed Away, vacuum running:** the alarm steps down to Home while the vacuum cleans, and goes back to Away at the dock. Tap Cancel and the run stops; Away comes back when the vacuum is home.
- **Alarm during a vacuum run:** the run stops, you're told, and the panel is left alone.
- **Someone in the house while you're out:** a room reads occupied while the panel is armed Away, and you get one notice.
- **The range left on:** an appliance you've marked stays on ten minutes with nobody home, and you're told. It's never switched off for you.
- **Dusk and dawn:** the lights you've labeled come on at dusk and go off at dawn.
- **Home after dark:** the landing lights come on as you arrive and go off ten minutes later. A light that was already on stays on.
- **A garage door left open:** the house asks on your phone; Yes closes it. While the house is armed, Away or at Night, it closes unless you say No.
- **High wind with a door open:** the house asks, and closes it if nobody answers.
- **A window open with the heat on:** five minutes in, the house asks whether to turn the HVAC off. Once everything's closed it offers to turn it back on. At night it asks nothing and restores quietly.
- **The morning briefing:** the almanac leads with the weather that matters, such as "hard freeze from Tue 18:00". *(Always on, with the Weather tool.)*
- **"How's the house?":** the agent has security, climate, weather, vacuum, hot tub, service desk, guest and paperwork summaries to answer from, and knows when security is temporarily stepped aside. *(Each summary needs its read cert.)*
- **Guests:** the agent knows who arrives or leaves in the next day, and flags a stay still not released two days after checkout. *(Twenty plugin; needs its read cert.)*
- **Paperwork:** an import Paperless couldn't file is flagged for a week, then listed quietly. *(Paperless; needs its read cert.)*
- **The hot tub:** chemistry is judged against its last 24 hours, so one odd reading is a blip, not an alarm, and falling chlorine gets a warning before it's low.
- **"What did the house do while I was out?":** security keeps a record of what it did on its own, and why. *(Always on.)*
- **Nobody could be asked:** a question that reached no phone answers that straight away, instead of waiting out a tap that can't come. *(Always on.)*
- **A leak, an alarm or smoke:** the agent's summary of it starts at once, ahead of routine work. *(With the background summaries on.)*
- **"How's the house?", honestly dated:** a summary that has gone stale is reported with its age, never as happening now. *(Always on.)*
- **Judgment calls:** ask whether the forecast has changed enough to matter, when's the best time to mow, whether the hot tub is ready for a soak, whether a ticket looks ready to close, whether an overdue task is still real, or whether a room is really empty. Each answer is advice and says what it looked at. *(Preview; off until you turn on decision mode.)*

---

## Added

- **Security Manager 5.10.0 to 5.29.0:** readiness, temporary overrides, the activity ledger, `lens`, the Zen Security Sentry, vacuum sync, the occupied-while-away notice, dusk and dawn lighting, arrival lights, exterior covers, the appliance sentry, a camera look when a watched cover opens, the weather close on any `cover_close_signal`, and `summary` behind `zenos.security.read`. See [Security Manager](../components/security_manager.md#readiness-overrides-and-activity).
- **Autovac 5.8.0 to 5.14.2:**
  - its own scored component, behind `zenos.autovac.status`;
  - `swap_robot`: replace your robot vacuum, and Autovac moves to the new one and keeps the old one's maintenance records;
  - the Roborock Qrevo Edge 3 Pro parts catalog; a part not confirmed for the exact model is tracked but never reordered;
  - a cleaning run you started that never actually began is reported;
  - schedules added, edited and removed from the tool;
  - `stop` and `run_state`;
  - `run_on_away`;
  - the vacuum's security settings.

  See [Autovac](../components/autovac.md#replacing-the-robot).
- **Weather 1.0 to 1.1 (new):** current conditions, the forecast by day part, and scored hazards for freeze, heat, storm, wind, rain and snow, behind `zenos.weather.read`. The source is the entity labeled `main_weather`. It's also an Oracle host, four templates behind `zenos.decision.component.weather`. See [Weather](../scripts/zen_dojotools_weather_readme.md).
- **Climate 2.3.0 to 2.5.1:** a scored summary and the `climate` component, and the envelope offer. Room Manager's `hvac_coverage` says which rooms each air handler serves. See [Climate](../scripts/zen_dojotools_climate_readme.md).
- **Taskmaster 0.10.0, 0.11.0:** briefing and almanac read the Weather tool: the next 24 hours' high and low, and hazards with when they start. It's also an Oracle host (never medications). See [Taskmaster](../kung_fu/taskmaster.md).
- **Spa Master 5.7.0 to 5.10.0:** a scored summary and the `spamaster` component, behind `zenos.spa.read`, and an Oracle host. Chemistry is judged against its 24-hour trend. See [Spa Master](../components/spamaster.md).
- **Service Desk 0.6.0 to 0.8.0 (Zammad plugin):** a scored summary and the `servicedesk` component, behind `zenos.service.read`, and an Oracle host. See [Zammad](../plugins/zammad.md).
- **Rolodex 1.16.0 to 1.16.2 (Twenty plugin):** a scored summary of guest stays (rooms and dates, never names) that seeds the `rolodex` component, behind `zenos.rolodex.read`. See [Twenty](../plugins/twenty.md).
- **Library 6.16.0, 6.16.1:** a scored summary of the Paperless inbox and failed imports (counts and file names, never content) and the first `library` component, behind `zenos.library.read`. See [Library](../scripts/zen_dojotools_library_readme.md).
- **A plugin that's installed but not set up scores 0** in the Service Desk, Rolodex and Library summaries, instead of reporting itself unreachable every hour.
- **Oracle 0.4.0 (new, preview, off by default):** typed judgments through Home Assistant's `ai_task.evaluate`. 21 templates across seven components (Room Manager, Weather, SpaMaster, Service Desk, Taskmaster, Plant water and electrical), each naming its own outcomes, and `ask` for an agent's own questions as advice. It never acts. Behind the operator's decision switch, `zenos.decision.use` and each component's decision cert (`zenos.decision.component.<component>`). See [Oracle](../scripts/zen_dojotools_oracle_readme.md#the-hosts) and [the story](#an-oracle-in-a-pear-tree).
- **Room Manager 5.19.2 to 5.21.0:**
  - `home_overview` and `summary` carry a temporary security override;
  - `summary` reports a room occupied while armed Away;
  - `hvac_coverage`;
  - an Oracle host: `resummary_warranted`, and per room `room_occupancy` and `hold_review`, behind `zenos.decision.component.room_manager`.

  See [Room Manager](../components/room_manager.md).
- **Postman 5.6.0:** a call that reached nobody answers `no_channel`.
- **SystemTools 5.4.3, 5.5.0:**
  - the health report tells a Spook that failed to load from one that isn't installed;
  - it names the security watchers and any disabled automation.

  See [SystemTools](../scripts/zen_dojotools_systemtools_readme.md).
- **Labels 5.11.0, 5.11.1:** tagging or untagging through the tool re-arms the watchers that read that label. Labeling in the Home Assistant UI still needs the watcher turned off and on.
- **Plant 5.14.0 to 5.16.0:** water and electrical summaries, the `water` and `electrical` components, behind `zenos.water.read` and `zenos.electrical.read`, from two new Plant codexes. Both are Oracle hosts; a decision never touches a valve or a breaker. See [The Plant codexes](../components/plant_manager.md#the-plant-codexes).
- **FileCabinet 6.41.0:** `list` and `fleet` report drawer sizes, so you can see what fills a cabinet. See [FileCabinet](../scripts/zen_dojotools_filecabinet_readme.md#list).

---

## The Coffee Bin

<!-- Draft: Nathan to review. -->

The house buys its coffee in one-pound bags and grinds from a bin. The bin weighs the same every time, so it's the tare. Weigh the bin, take the bin away, and what's left is coffee, counted in grams. Stock is the sealed bags at 454 g each, plus whatever is in the bin.

Grocy handles tare weights, but it expects every stock write as the total on the scale, and ZenOS was sending "bought one bag". A purchase was refused, a use was read as a new total, and a count left out the container. Friday hit it first.

2026.11.1 taught Inventory to set up a tare-weight product. It also made `amount` the net quantity in every stock write, so you say what you bought or used and ZenOS does the arithmetic Grocy expects. That's the shape of this release: the capability first, then the tools that use it.

---

## An Oracle in a Pear Tree

<!-- Draft: Nathan to review. -->

Every background summary costs the same, whether the house changed or not. A trigger fires, the Ninja gathers the evidence, and a model writes the whole picture again. Most of the time the answer is "nothing much".

Home Assistant 2026.11 added a different kind of model call, `ai_task.evaluate`. It doesn't write anything. It answers a question: yes or no as a probability, one of a few choices, or a score. Oracle is ZenOS's door to it.

With Oracle, a component can ask a small question about its own evidence before it pays for a full summary: *does this matter enough to write the picture again?* Room Manager asks first. It sets the rooms as they are now beside its last summary, and asks whether someone in the household would need a fresh one. On the reference house the question took under a second. A decision still gathers the room evidence; what a "keep" saves is the summary's model call and its thousands of tokens. Whether that adds up is something we'll measure, not promise.

Weather asks next, two ways: has the forecast changed enough to rewrite, and do its hazards matter to the people at home right now? Some answers never reach a model. No hazard is nothing to worry about, and a hazard at urgency 4 or more always matters, so a severe storm is never judged away.

Then the rest of the house joined in. The spa, the service desk, the task list and the plant's water and power each ask their own questions, 21 in all. Some are about whether to rewrite a summary. Others are the judgment calls a person makes all day: is the hot tub ready for a soak, is this ticket done, is that overdue chore still real, is the back office really empty or just quiet? Every host puts its rules first. A reading out of range is a problem without asking anyone, and a smoke or leak state is never judged at all.

Oracle judges and never acts. A decision can say a summary is warranted; it can't request one, force one, or touch a device. The component owns its questions and its thresholds, and a caller only picks which question to ask. An agent with a higher grant can ask questions of its own, over facts it supplies, and gets advice back, never a decision.

In this release Oracle is a preview you try by hand. Nothing in the background calls it yet. It's off until you turn on decision mode, `input_boolean.zen_decision_enabled`, which no agent can reach, and grant `zenos.decision.use` and the decision cert of each component you want to ask. See [Oracle](../scripts/zen_dojotools_oracle_readme.md).

---

## Read this before upgrading

- **Home Alone is built for the pending Spook fix (5.11.2 or later).** Until it ships, follow the [Spook advisory](../../../README.md): Spook 5.6 to 5.10 can't load on Home Assistant 2026.11, and 5.11.0 and 5.11.1 freeze a ZenOS install.
  - Without a working Spook, the Zen Security Sentry can't run. Temporary overrides answer `spook_required`, and vacuum sync can't be turned on. Autovac's schedule editing needs Spook 5.11.0 or later.
  - Everything else in this release runs without it. The Sentry is its own automation, so a Spook problem never stops the Auto-Arm Watcher.
- **Climate has its own file,** `dojotools_climate.yaml`, out of the Utilities bundle.
- **The weather source is the entity labeled `main_weather`.** `zen_weather` is still read.
- **Grant `zenos_agent_parity` again (one phone approval).**
  - It's built from the summaries you have on, so it picks up their new read certs:
    - `zenos.security.read` (the security summary now seeds from Security Manager's own `summary`);
    - `zenos.autovac.status`, `zenos.weather.read`, `zenos.climate.read` and `zenos.spa.read`;
    - `zenos.water.read` and `zenos.electrical.read`, for the Plant summaries;
    - with the Zammad plugin, `zenos.service.read`;
    - with the Twenty plugin, `zenos.rolodex.read`, and with Paperless, `zenos.library.read`. These two are in no standard bundle, because guest stays and file names can be personal, so only parity or a single grant gives them.
  - An existing grant doesn't gain new members by itself.
  - With `seed_cert_mode` on enforce, a summary whose cert is missing stops. In audit, the default on an upgrade, it logs `seed_cert_audit` and keeps running.
- **Restart, or run `zen_dojotools_systemtools mode=zen_resolver_refresh`, after upgrading.** Room timers now expire through Home Assistant's `timer.finished` trigger on their labels, and a TV sleep timer gets its new `tv_sleep_timer` label at the next resync. Until then, the TV sleep timer doesn't start.
- **Restart Home Assistant after upgrading.** If you reload instead: custom templates first (the summarizers and the recovery router import a new macro, and cabinet mounts need the new cabinet templates), then automations, wait, then scripts. Autovac's controller and the new watchers need the automations before the scripts. The Scheduler starts its work through a new script, `zen_abbot_launch`, so background summaries start once scripts are reloaded.
- **Load the new prompt once:** run `zen_admintools_prompt_loader mode=load` after updating to move Friday to Cortex 48 'True North'. An install keeps the prompt it last loaded until you do.
- **Four new automations:** the Zen Security Sentry, the Zen Climate Envelope Watcher, Zen DojoTools Housekeeping and Zen Safety Admit.
- **Your own trigger automations should send `summary_request`, not `summary_force`.** A force still works, but it's for maintenance now. See [the Scheduler](../scripts/zen_dojotools_scheduler_readme.md#requesting-a-run).
- **Oracle needs Home Assistant 2026.11 and an AI Task entity that supports evaluation.** Without one, it answers `evaluator_unavailable`, and nothing else changes. Its `ask` fields reach an MCP client after a restart and a client reconnect.
- **Autovac's `clean` answers `dispatched`, not `ok`.** Anything that reads `ok` from it needs updating.
- **Includes 2026.11.1,** with its security fixes.

---

## Changed

- **With the cover watch on, a cover left open while the house is secure is closed unless someone says No.**
  - "Secure" means the panel is armed, or the home mode is Away, Night or Night-Late.
  - That replaces `cover_day_policy` then, even one set to `ask`, and a cover left open while the house sleeps closes quietly.
  - Turn it off with `cover_secure_close_enabled: false`, or keep Yes-only with `cover_armed_policy: ask`.
  - With the cover watch off, nothing closes.
- **The security background summary seeds from Security Manager's `summary`.** It used to seed from Room Manager's `home_overview`. It needs `zenos.security.read`.
- **One weather label, `main_weather`, picks the weather source for every tool:** Weather, Room Manager's `home_overview`, Taskmaster and the dashboard sensors. `zen_weather` is still read.
- **Room timers expire on their labels.** They used to expire on Home Assistant's event for every timer. TV sleep timers are found by the `tv_sleep_timer` label, applied at resync.
- **Friday's start-up context is about 29% smaller, and she routes every request through one rule:** the right specialist first, and a refusal is final. That's Cortex 48, 'True North'. See [AdminTools](../scripts/zen_dojotools_admintools_readme.md).
- **The agent switches any helper toggle or switch you ask for,** even ones not exposed to Assist.
- **A mount carries its whole tree.** A drawer under a link to another cabinet now lives in that cabinet, and a link is never written over data that would leave it split between two. See [FileCabinet](../scripts/zen_dojotools_filecabinet_readme.md#virtualdrawer-mount-point).
- **Household and family cabinet admin (expand, mount, ACL) is open to anyone holding `zenos.system.cabinet_admin` with the live approval,** not only the owner. Personal cabinets stay owner-only.
- **The Abbot only decides which background summaries run.** See [the Scheduler](../scripts/zen_dojotools_scheduler_readme.md).
  - Maintenance (reloads, the manifests, garbage collection, display and Rolodex checks) has its own automation, so a script reload no longer waits behind summaries.
  - A leak, alarm or smoke event starts its summaries at once, with no queue.
  - A routine refresh is a `summary_request`, through the normal gates; forces are for maintenance only. Autovac, garbage collection, Flynn's health refresh and the recovery router all send requests now.
  - SuperSummary runs every quarter hour, at start, and when a component's urgency asks for it, at most once every 5 minutes.
  - At start, components launch 30 seconds apart, and one with no summary yet is filled. Flynn's start-up force is gone.
  - The recovery router works out when each summary is stale from that component's own schedule, recovers the `system` tier too, and logs why it ran one.
  - SuperSummary uses the same rule to flag stale component summaries: it gives their age and never presents them as current. See [Freshness](../scripts/zen_dojotools_summarizers_readme.md#freshness).
- **The summarizer kill switches can't be exposed to an agent.** They carry `zen_never_expose`, so Ectoplasm refuses them and Flynn keeps them unexposed.
- **`sensor.zen_scheduler_queue_depth` measures pressure: Scheduler runs plus Ninja runs.** A dashboard card on it shows different numbers. See [the sensor reference](../sensors/readme.md#sensorzen_scheduler_queue_depth).
- **Autovac:**
  - Exactly one vacuum may carry the `autovac` label. With two, Autovac uses neither.
  - A bare call answers help (it answered `status`).
  - The pre-run briefing push and the run confirmations are settings, off on a fresh setup and kept on for an upgraded one.

---

## Fixed

- **A missing notify service no longer ends a Postman call.** On a fresh install the default `notify.admin_devices` doesn't exist, and the error stopped the whole call: no wait for the tap, no spoken message, no log. Each push now runs on its own.
- **Security's panel, perimeter and camera components read your house.** They named one house's entities and hours. They now read by label.
- **Cancel on the vacuum notice stops the vacuum.** It sent the vacuum home and left Away armed while it was still moving. It now stops the run through Autovac and keeps the panel at Home until the vacuum docks.
- **`auto_arm_config` reads with no patch,** where it answered `missing_auto_arm_patch`. A patch of only numbers or booleans is accepted.
- **Autovac before setup:**
  - Its write modes answered ok and stored nothing; they now answer `autovac_not_initialized`.
  - `config` passed as an object is read.
  - `setup create_labels` creates the labels it names.
- **The Rolodex background summary runs on its own.** Its component had no seed, so it only ran when forced.
- **SystemTools' `scheduler_anchors` repairs anchors a reset left at 00:00.** Every set was refused.
- **Text answers a number as text.** A value like `"18"` reported an error after it had been written.
- **Climate's `read` reports `hvac_mode`.** It was always null.
- **History and room occupancy really move to their cabinets.** A link to another cabinet redirected only its own key, so `history` and `room_occupancy` drawers stayed in the household cabinet. Flynn and Room Manager now create their roots in the target, and Room Manager `setup` moves the data before it links.
- **Autovac never empties its drawer.** After the drawer was expanded, the controller read it without following the mount and wrote empty rooms back. It now reads through the mount, and nothing writes over a drawer it couldn't read.
- **A covers set that was refused says so.** It answered success around the refusal.
- **The background summaries stop re-running fresh components.** The recovery router aged every summary from its creation time.
- **The Scheduler's housekeeping jobs all run.** They shared one first-match choice, so display session expiry, the resolver refresh, the Rolodex stay checks, the identity manifest and the compact index never ran from it.
- **A healthy install gets new default labels.** Flynn's all-green exit came before it assigned them, so a label added in a release never reached an install with every gate green. It now assigns them at start.
- **A background summary whose seed answers with a list finishes.** The Ninja stopped with "'list object' has no attribute 'get'" (Logistics intake).
- **The recovery router leaves cache slots and views alone.** It re-ran Security's perimeter cache and a FileCabinet view every 10 minutes, which hid components that really were stale.
- **Setting up a robot's parts no longer puts several parts on one product,** and finds a location on a house with more than 500.
- **Editing a Grocy chore can turn it off or set its amount to 0.**
- **The leak watch closes shutoff valves,** not only switches.
- **A removed component no longer stops the Kung Fu bootstrap.**
- **Editing a stock entry's best-before date saves.** Inventory's `stock_entry_update` addressed the entry in a way Grocy refused, then reported the values you asked for as saved. It now writes to the right entry and reads it back from Grocy before reporting success.

---

## Thanks

The Weather tool grows out of TheFes' Weather Forecast tool, which ZenOS ran as the house forecast script, keeping its fields. TheFes also wrote the Music Assistant LLM tool, and the trigger-based template sensors that hold data in their attributes, the pattern every ZenOS cabinet is built on. Thank you.

---

Keep the change, ya filthy animals.
