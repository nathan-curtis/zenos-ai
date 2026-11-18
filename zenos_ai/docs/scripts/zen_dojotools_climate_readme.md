# Zen DojoTools Climate — v2.5.0

**Script:** `zen_dojotools_climate`
**File:** `packages/zenos_ai/dojotools/dojotools_climate.yaml`

> **2026.12.0 (2.4.0):** the `climate` KFC (`mode=kfc_manifest`: one component, ambient tier, seeded by `summary`, cert `zenos.climate.read`) and the Weather feed. `summary` carries `outdoor` (`temperature`, `humidity`, `condition`, `temp_high`, `temp_low`, `unit`, `freeze_expected`), read from `zen_dojotools_weather mode=summary` when that tool is installed (`{available: false}` otherwise), and a `freeze_forecast` reason (urgency 2, ambient) when a freeze is forecast.

> **2026.12.0 (2.3.0):** `mode=summary` is a scored, read-only, whole-house answer (urgency 0 to 5, reasons, units, attention) for the climate KFC to seed from, and `name` is optional now (a bare call reaches help). The new read cert `zenos.climate.read` (level 1, stage `s`) gates only the background seed; asking in conversation needs nothing. Nothing in `summary` sets a thermostat.

> **2026.12.0 (2.2.2):** Climate moved out of the Utilities bundle into its own file. The script is unchanged: same entity, fields, behavior and cert. See the [Utilities readme](zen_dojotools_utilities_readme.md) for the other canonical domain tools.

## Overview (also owns `fan.*`)

GET+SET for `climate` **and, as of 2026-08-16, `fan`** entities — domain auto-detected from `name` after resolution, no separate mode needed. Closes a real gap: before this, there was no general-purpose fan primitive anywhere in the codebase (only `spa_manager`'s hardcoded hot-tub jets/air-purifier fan calls, not usable generally). Inspects `supported_features` before applying — fails closed on unsupported feature, same philosophy for both domains (fan capability detection via the real `FanEntityFeature` bitmask). Multiple setters may be sent in one call. Omit all to read state and capabilities.

GET response includes a `topology_context` block: open doors/windows, area temperature/humidity sensors, adjacent HVAC bleed portals (via Room Manager), and a natural vent advisory. A top-level `domain` field on both GET and SET responses reports which domain was actually resolved.

| Field | Description |
|---|---|
| `name` | Entity ID or shorthand name — `climate.*` or `fan.*` |
| `room` | Area for topology context in GET mode (auto-detected if omitted) |
| `temperature` | Target temperature (single setpoint) |
| `target_low` / `target_high` | Auto/heat-cool range |
| `hvac_mode` | HVAC operating mode — read first to see supported modes |
| `fan_mode` | Climate entity's own internal fan mode (e.g. thermostat fan setting) — **not** the same as `fan_percentage`/`fan_preset_mode` below, which target a real `fan.*` entity |
| `swing_mode` | Swing mode |
| `preset` | Preset mode (eco, sleep, away, etc.) |
| `humidity` | Target humidity (if supported) |
| `aux_heat` | ~~Auxiliary heat~~ — `not_implemented`. `climate.set_aux_heat` was removed from HA with no replacement (surfaced by Spook v6 sweep). |
| `power` | Power control — `on`/`off`. Domain-routed: `hvac_mode` for climate, fan power for a `fan.*` target |
| `fan_percentage` | Fan speed 0–100%. `fan.*` targets only |
| `fan_preset_mode` | Fan preset (e.g. `low`/`medium`/`high`/`auto`). `fan.*` targets only |
| `fan_oscillate` | Oscillation on/off. `fan.*` targets only |
| `fan_direction` | Fan direction. `fan.*` targets only |
| `get` | Any truthy value forces read-only |
| `mode` | `read` forces a read-only reply. `set` requires at least one real setter field — returns `missing_setter` if none given. `tool_manifest` for self-description. Omitting `mode` still works identically. |

**Identity gate (2026-08-16):** GET stays open. Any real setter requires the `zenos.actuation.climate_control` certification — cert-only, no live-ack tier (same shape as ZenLux's `zenos.actuation.lighting_control`; nothing on a thermostat or fan is exterior-security-grade). `cert_scope` deny is enforced as a hard block via the shared `cert_scope_check` macro. See the [Security & Certification System operator manual](../getting_started/security_certification_manual.md).

**SESE conversion note:** this tool doesn't fit the mode-dispatch `choose:` shape the other gated tools use — multiple setters can combine in one call, always could. Uses a flat-guard restructure instead: every stop point sets `result` and falls through, every later block gains `result is not defined and` on its own condition. GET keeps its own terminal exit, same read/write split every tool in this file uses.


---

## `mode=summary`

```json
{ "mode": "summary" }
```

Reads every `climate` entity once and scores it in code, for the ambient tier (Trapper Keeper digests urgency 0 to 3; 4 or 5 is promoted to an attention line, so an error scores 4 or more). Returns `urgency`, `action_required` (3 or more), `reasons[]`, `unit_count`, `home_mode`, `units[]` (up to 8: `entity_id`, `name`, `area`, `mode`, `action`, `current`, `target`, `unit`, `main`, `unavailable`, `open_openings[]`), `attention[]` and `watch_entities[]`. Freeze and overheat read the unit's own current temperature (Fahrenheit when the entity's `max_temp` is 50 or more, Celsius otherwise).

| Reason | Score | When |
|---|---|---|
| `freeze_risk` | 5 | A unit reads 45 °F (7 °C) or less |
| `overheat` | 4 | A unit reads 90 °F (32 °C) or more |
| `main_thermostat_unavailable` | 4 | The thermostat labeled `main_thermostat` is unavailable or unknown |
| `thermostat_unavailable` | 3 | Any other unit is unavailable or unknown |
| `conditioning_with_opening` | 2 | A unit is heating or cooling while a door or window in its area is open |
| `freeze_forecast` | 2 | The Weather tool forecasts a freeze in the next 24 hours |
| `conditioning_while_away` | 1 | A unit is heating or cooling while the home mode is Away |
| `paused_for_envelope` | 1 | A unit the envelope offer turned off (household said yes) is still off; listed in `envelope_paused[]` |
| `no_climate_units` | 0 | The house has no climate entity to score |

`zenos.climate.read` (level 1, stages `i`, `s` and `l`: the summary seed and a live drawer mounted on it) is declared in `tool_manifest` with `seed_modes: summary@zenos.climate.read:1`. Grant it with `persona_editor mode=cert_req_grant cert_component=zenos.climate.read cert_stages=isl`. The `climate` KFC is defined by `mode=kfc_manifest` (ambient tier, `pipeline_tier: ambient`, subscriptions `hourly_trigger`, `ha_start`, `home_mode_updates`, `force_summary`).

**Not scored.** Climate entities that are not room comfort are skipped and listed under `excluded`: anything labeled `spa_climate`, `zen_plant_hot_tub`, `zen_plant_thermal` or `zen_rm_ignore`. A hot tub held at 103 °F on purpose is not "overheating" (2.3.1; found on the first real house).

## `mode=envelope`: the offer to turn the HVAC off when the house is open

```json
{ "mode": "envelope" }
{ "mode": "envelope", "envelope_offer_enabled": true }
```

**The envelope** is every door, window or opening sensor labeled `exterior` (`binary_sensor`, device class `door`, `window`, `opening` or `garage_door`). Label the sensors with `zen_dojotools_labels mode=tag label_list=["exterior"] target_entities=[...]`. Leave the garage door unlabeled if it shouldn't count. Interior doors are never part of it. The watcher reads the labeled sensors when it starts. Labeling through `zen_dojotools_labels` (tag or untag) re-arms it for you (Labels 5.11.0). After labeling in the Home Assistant UI, turn the Zen Climate Envelope Watcher off and on, or restart: reloading automations is not enough, because a reload skips an automation whose config has not changed.

**With `envelope_offer_enabled` on** (default off; writing it needs `zenos.system.home_config_write` level 1, reading needs nothing):

1. An envelope opening is open for five minutes while heating or cooling runs. The house announces it (Announce tool) and sends a Yes/No to the household's phones (Postman, five minutes to answer). The message carries the outdoor temperature when the Weather tool is installed.
2. **Yes** turns off the units that serve the open area and saves each unit's mode in the household drawer `climate_envelope`. If everything was closed while the question waited, nothing is turned off. **No** or no answer leaves the HVAC alone, and the offer isn't repeated until the envelope closes and opens again.
3. Once every opening has been closed for two minutes, the house offers the same way to turn them back on. **Yes** puts each unit back in its saved mode, but only units still off (a unit someone changed in the meantime is left alone). The saved record is cleared after any answer.
4. **Quiet while the house sleeps** (the same rule as Postman's sleep gate: quiet hours on, or the home mode Night or Night-Late). Nothing is announced or asked, and the open offer waits for the next time the envelope opens. A Yes meant "off until it's closed", so if everything closes while the house sleeps, each unit is put back in its saved mode quietly (only units still off). A courtesy note is saved and sent to the household's phones when the house wakes (the home mode leaves Night or Night-Late, or quiet hours end), because Postman drops a message sent into the sleep gate.
5. **The agent knows.** While the HVAC is paused, a one-line Priority Inject entry (`provider_id: climate_envelope`, urgency `urgent`) sits in the agent's active context, saying what is open, since when, and that the household said yes. It is cleared when the units go back on or the record is cleared. `mode=summary` also gives the reason `paused_for_envelope`.

**Which units an opening turns off:** the `hvac_air_handler` (or `hvac_air_handler_<n>`) label the opening's area shares with thermostats (see Room Manager `mode=hvac_coverage`). Without one, a thermostat in the same area. Without that, every unit that is running. Spa and thermal loads are never touched.

Returns `enabled`, `envelope[]` (`entity_id`, `name`, `area`, `open`), `open_now[]`, `shut_off` (the saved units and modes while the HVAC is off for an opening, else `null`) and `how_to`.

The trigger is the automation **Zen Climate Envelope Watcher**, in this package. It calls the internal `mode=envelope_check` (refused for any caller with a user, so an agent can't turn the HVAC off through it) and does nothing while the setting is off.
