# Release Notes — 2026.12.0 'Home Alone'

**Status:** Beta 1
**Branch:** `feat/2026.12.0`
**Base:** 2026.11.1 (patch on 2026.11.0 'This Is Spinal Tap')

<!-- Minimum Home Assistant version: to confirm. Room Manager's timers use the timer.finished trigger (about 2026.5); Spook needs 2026.11. -->

---

## Summary

The house gets left home alone.

Spinal Tap put locks on the cabinets and a pass in every agent's hand. Home Alone gives the house something to do with them. Security Manager grows from a panel reader into the house's security. It can tell you whether the house is ready to arm and keeps a record of what it did on its own. Once you turn them on, it also lights the house at dusk, lights the way in when you get home after dark, watches the garage doors, steps the alarm aside for the vacuum, and tells you when the range is on with nobody home. Climate learns which rooms each air handler serves and offers to pause the HVAC when a window's left open. A new Weather tool gives every other tool the forecast.

Core changes little. The tools come up to caps: each gets a scored summary, a read cert of its own, and a background summary the agent can rely on.

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
- **"How's the house?":** the agent has security, climate, weather, vacuum, hot tub and service desk summaries to answer from, and knows when security is temporarily stepped aside. *(Each summary needs its read cert.)*
- **"What did the house do while I was out?":** security keeps a record of what it did on its own, and why. *(Always on.)*
- **Nobody could be asked:** a question that reached no phone answers that straight away, instead of waiting out a tap that can't come. *(Always on.)*

---

## Added

- **Security Manager 5.10.0 to 5.26.2:** readiness, temporary overrides, the activity ledger, `lens`, the Zen Security Sentry, vacuum sync, the occupied-while-away notice, dusk and dawn lighting, arrival lights, exterior covers, the appliance sentry, and `summary` behind `zenos.security.read`. See [Security Manager](../components/security_manager.md#readiness-overrides-and-activity).
- **Autovac 5.8.0 to 5.12.0:**
  - its own scored component, behind `zenos.autovac.status`;
  - schedules added, edited and removed from the tool;
  - `stop` and `run_state`;
  - `run_on_away`;
  - the vacuum's security settings.

  See [Autovac](../components/autovac.md#run-on-away).
- **Weather 1.0 (new):** current conditions, the forecast by day part, and scored hazards for freeze, heat, storm, wind, rain and snow, behind `zenos.weather.read`. The source is the entity labeled `main_weather`. See [Weather](../scripts/zen_dojotools_weather_readme.md).
- **Climate 2.3.0 to 2.5.1:** a scored summary and the `climate` component, and the envelope offer. Room Manager's `hvac_coverage` says which rooms each air handler serves. See [Climate](../scripts/zen_dojotools_climate_readme.md).
- **Taskmaster 0.10.0:** briefing and almanac read the Weather tool: the next 24 hours' high and low, and hazards with when they start. See [Taskmaster](../kung_fu/taskmaster.md).
- **Spa Master 5.7.0 to 5.7.2:** a scored summary and the `spamaster` component, behind `zenos.spa.read`. See [Spa Master](../components/spamaster.md).
- **Service Desk 0.6.0 to 0.6.2 (Zammad plugin):** a scored summary and the `servicedesk` component, behind `zenos.service.read`. See [Zammad](../plugins/zammad.md).
- **Room Manager 5.19.2 to 5.20.1:**
  - `home_overview` and `summary` carry a temporary security override;
  - `summary` reports a room occupied while armed Away;
  - `hvac_coverage`.

  See [Room Manager](../components/room_manager.md).
- **Postman 5.6.0:** a call that reached nobody answers `no_channel`.
- **SystemTools 5.4.3, 5.5.0:**
  - the health report tells a Spook that failed to load from one that isn't installed;
  - it names the security watchers and any disabled automation.

  See [SystemTools](../scripts/zen_dojotools_systemtools_readme.md).
- **Labels 5.11.0, 5.11.1:** tagging or untagging through the tool re-arms the watchers that read that label. Labeling in the Home Assistant UI still needs the watcher turned off and on.

---

## The Coffee Bin

<!-- Draft: Nathan to review. -->

The house buys its coffee in one-pound bags and grinds from a bin. The bin weighs the same every time, so it's the tare. Weigh the bin, take the bin away, and what's left is coffee, counted in grams. Stock is the sealed bags at 454 g each, plus whatever is in the bin.

Grocy handles tare weights, but it expects every stock write as the total on the scale, and ZenOS was sending "bought one bag". A purchase was refused, a use was read as a new total, and a count left out the container. Friday hit it first.

2026.11.1 taught Inventory to set up a tare-weight product. It also made `amount` the net quantity in every stock write, so you say what you bought or used and ZenOS does the arithmetic Grocy expects. That's the shape of this release: the capability first, then the tools that use it.

---

## An Oracle in a Pear Tree

<!-- Placeholder: only if it ships. ai_task.evaluate (HA 2026.11+), off by default. -->

---

## Read this before upgrading

- **Spook is required for temporary overrides.** On Home Assistant 2026.11, Spook 5.6 to 5.10 can't load.
- **Climate has its own file,** `dojotools_climate.yaml`, out of the Utilities bundle.
- **The weather source is the entity labeled `main_weather`.** `zen_weather` is still read.
- **Grant `zenos_agent_parity` again (one phone approval).**
  - It's built from the summaries you have on, so it picks up their new read certs:
    - `zenos.security.read` (the security summary now seeds from Security Manager's own `summary`);
    - `zenos.autovac.status`, `zenos.weather.read`, `zenos.climate.read` and `zenos.spa.read`;
    - with the Zammad plugin, `zenos.service.read`.
  - An existing grant doesn't gain new members by itself.
  - With `seed_cert_mode` on enforce, a summary whose cert is missing stops. In audit, the default on an upgrade, it logs `seed_cert_audit` and keeps running.
- **Restart, or run `zen_dojotools_systemtools mode=zen_resolver_refresh`, after upgrading.** Room timers now expire through Home Assistant's `timer.finished` trigger on their labels, and a TV sleep timer gets its new `tv_sleep_timer` label at the next resync. Until then, the TV sleep timer doesn't start.
- **Reload automations, wait, then reload scripts.** Autovac's controller and the new watchers need the automations first.
- **Two new automations:** the Zen Security Sentry and the Zen Climate Envelope Watcher.
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
- **Autovac:**
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
- **SystemTools' `scheduler_anchors` repairs anchors a reset left at 00:00.** Every set was refused.
- **Text answers a number as text.** A value like `"18"` reported an error after it had been written.
- **Climate's `read` reports `hvac_mode`.** It was always null.

---

## Thanks

The Weather tool grows out of TheFes' Weather Forecast tool, which ZenOS ran as the house forecast script, keeping its fields. TheFes also wrote the Music Assistant LLM tool, and the trigger-based template sensors that hold data in their attributes, the pattern every ZenOS cabinet is built on. Thank you.

---

Keep the change, ya filthy animals.
