# Release Notes — 2026.12.0 'Home Alone'

**Status:** Beta 1
**Branch:** `feat/2026.12.0`
**Base:** 2026.11.1 (patch on 2026.11.0 'This Is Spinal Tap')

---

## Summary

<!-- The story: the house is left to Friday, the Wet Bandits come calling, and someone tells it afterward, one day of Christmas at a time. Each day is a tool brought up to caps (capabilities), told as one more trap. Core changes little; every tool gets the glow-up. -->

---

## What's New

<!-- One line per day, linking to its section. Filled as the days are settled. -->

---

## The Twelve Days

<!-- Draft order. Each verse links to the plain notes for its tool. Only shipped work gets a verse. -->

### On the first day: readiness
<!-- Security Manager 5.10.0, 5.16.0: can the house arm now, and what's in the way; reads the panel's own zones; blind when nothing was checked. -->

### On the second day: temporary overrides, the ledger and lens
<!-- Security Manager 5.11.0 to 5.13.0, 5.22.0: a recorded, time-limited change with a ceiling nothing can raise; what security did on its own; the agent knows while one is in force. -->

### On the third day: the Sentry
<!-- Security Manager 5.14.0, 5.15.0, 5.18.0, 5.23.1: ends overrides on time, recovers a missed dock; needs Spook. -->

### On the fourth day: vacuum sync
<!-- Security Manager 5.18.0: Away to Home while the vacuum cleans, back at the dock; Cancel stops the run. -->

### On the fifth day: run on Away
<!-- Autovac 5.11.0, 5.12.0: stop, run_state, run_on_away. -->

### On the sixth day: Autovac's glow-up
<!-- Autovac 5.8.0 to 5.10.0: its own scored component, schedules from the tool, notices as settings. -->

### On the seventh day: occupied while away
<!-- Security Manager 5.19.0, Room Manager 5.19.3. -->

### On the eighth day: dusk and dawn
<!-- Security Manager 5.20.0. -->

### On the ninth day: arrival lights
<!-- Security Manager 5.21.0. -->

### On the tenth day: exterior covers
<!-- Security Manager 5.23.0: left open, Night close, damaging wind. -->

### On the eleventh day: the envelope
<!-- Climate 2.3.0 to 2.5.x: scored summary and component, air-handler coverage (Room Manager 5.20.0), the envelope offer. -->

### On the twelfth day: the weather
<!-- Weather 1.0.x, Taskmaster 0.10.0 almanac hazards. Built on TheFes' Weather Forecast tool: credit TheFes in the verse. -->

<!-- Not yet placed: the appliance sentry (Security Manager 5.24.0); the house watches itself (SystemTools 5.5.0, Postman 5.5.0/5.6.0, Labels 5.11.x); the locks coordinator (not built yet). -->

---

## The Coffee Bin

<!-- The tare story (Inventory 5.9.3 to 5.9.6, shipped in 2026.11.1). Generic: a coffee bin, 1 lb bags, grams. -->

---

## An Oracle in a Pear Tree

<!-- Only if it ships. ai_task.evaluate (HA 2026.11+), off by default. -->

---

## Read this before upgrading

- **Spook is required for temporary overrides.** On Home Assistant 2026.11, Spook 5.6 to 5.10 can't load.
- **Climate has its own file**, `dojotools_climate.yaml`, out of the Utilities bundle.
- **The weather source is the entity labeled `main_weather`.** `zen_weather` is still read.
- **New read certs:** `zenos.autovac.status`, `zenos.weather.read` and Climate's read cert. Without them, those components leave the prompt.
- **Autovac:** a bare call answers help; the pre-run briefing push and run notices are off on a fresh setup and kept on for an upgraded one; reload automations, wait, then reload scripts.
- **Two new automations:** the Zen Security Sentry and the Zen Climate Envelope Watcher.
- **Includes 2026.11.1**, with its security fixes.

---

## Changed

---

## Fixed

---

## Thanks

The Weather tool grows out of TheFes' Weather Forecast tool, which ZenOS ran as the house forecast script, keeping its fields. TheFes also wrote the Music Assistant LLM tool, and the trigger-based template sensors that hold data in their attributes, the pattern every ZenOS cabinet is built on. Thank you.

---

Keep the change, ya filthy animals.
