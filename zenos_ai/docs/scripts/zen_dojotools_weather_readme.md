# Zen DojoTools Weather — v1.1.0

> **2026.12.0 (1.1.0):** Weather is an [Oracle](zen_dojotools_oracle_readme.md) host (KF6.5 preview): `forecast_changed` and `hazard_relevance`. Nothing existing changed, and nothing happens unless the operator turns decision mode on. See [Decisions (KF6.5)](#decisions-kf65).

> **2026.12.0 (1.0.0):** a product weather tool with a locked schema. It keeps what the house forecast script (TheFes' "Weather Forecast") did and adds current conditions, highs and lows, in-code answers, named day parts, a read cert and a KFC.

**File:** `packages/zenos_ai/dojotools/dojotools_weather.yaml`
**Script:** `zen_dojotools_weather`
**Read only.** Nothing here changes a device, so there is no actuation cert.

---

## Overview

Weather now and the forecast, from the house's weather entity. The source is found by label, never by an entity id: the entity labeled `main_weather`, else one labeled `zen_weather` (the older label), else the first `weather.*` entity (the answer says which, as `source.labeled`). What the provider can forecast (daily, hourly, twice_daily) is read from its own `supported_features`. Units are the entity's own, reported in `units`, never converted.

Every answer carries `schema_version: "1.0"`. **The schema is locked: fields are added, never renamed, retyped or removed**, so other tools (Climate first) can read it safely.

---

## Modes

| Mode | What it does |
|---|---|
| `help` (default) | The reference. A bare call never acts. |
| `current` | Conditions now. |
| `forecast` | A window of forecast: an averaged summary, high and low, and flags. |
| `summary` | Scored hazards for the background summary (the seed of the `weather` KFC). |
| `tool_manifest` | Identity, labels and the read cert. |
| `kfc_manifest` | The `weather` KFC definition, for `bootstrap_kfc`. |
| `decision` | Weather's decision templates, for Oracle. Zero inference. |
| `decision_bind` | Called by Oracle, not by an agent: binds the evidence for a template. Needs `zenos.weather.read`. |

### `current`

```json
{ "mode": "current" }
```

Returns `source` (`entity_id`, `labeled`, `forecast_types[]`, `station{}`), `units`, `now{condition, temperature, apparent_temperature, dew_point, humidity, pressure, wind_speed, wind_gust_speed, wind_bearing, cloud_coverage, uv_index, visibility, ozone}` and `as_of`. A field the provider does not have is `null`.

### `forecast`

| Field | Meaning |
|---|---|
| `period` | `auto` (default), `daily`, `hourly` or `twice_daily`. `auto` is hourly for a part of a day, else daily. A daily request falls back to `twice_daily` when the provider has no daily forecast. |
| `length` | Days (daily, twice_daily) or hours (hourly) from the start, 1 to 48. Default 1 day, or 6 hours. |
| `start_date`, `start_time` | An exact start (`YYYY-MM-DD`, `HH:MM:SS`). `start_time` defaults to `00:00:00`. |
| `part` | `now`, `today`, `tomorrow`, `weekend`, `morning`, `afternoon`, `evening`, `night`, `tonight`. A part already over today means tomorrow's. |
| `aggregate`, `include_items` | The averaged summary is always returned; `aggregate=false` or `include_items=true` also returns every item in the window. |
| `time_period_type`, `time_period_length` | The house script's names for `period` and `length`; they still work. |

Day parts come from the household's anchors when they are set (`input_datetime` `zen_morning_start`, `zen_evening_start`, `zen_night_start`; defaults 06:00, 17:00 and 21:00); the afternoon starts at 12:00.

Returns `period{type, start_of_period, end_of_period, items_used, truncated, covered_until}`, `summary{...}`, `flags{...}` and `items` when asked.

`summary` carries the house script's averaged object under the same key names: numbers are averaged (rounded to one decimal), text such as `condition` is the most common value, and `start_of_period` and `end_of_period` are in `period`. It adds `temp_high`, `temp_low`, `precipitation_total`, `precipitation_probability_max` and `wind_speed_max`.

`flags` answers the common questions in code: `rain`, `snow`, `freeze`, `heat`, `wind` and `storm`, each `{expected, first_at, peak}`. Thresholds follow the entity's own units (freeze at 32 °F or 0 °C, heat at 95 °F or 35 °C, wind at 30 mph or 48 km/h and the matching values for m/s and knots).

Errors: `no_weather_source`, `source_unavailable`, `period_unsupported` (with `supported[]`), `start_in_past`, `forecast_unavailable`, `beyond_forecast` (the window starts after the provider's last forecast item; carries `forecast_ends`), `invalid_start` (`start_date` is not a real date), `missing_start` (`start_time` without `start_date`). When the window runs past the end of the forecast, `period.truncated` is true and `period.covered_until` says where the data stops. An hourly window that starts now begins at the provider's first item (this hour or the next), so a length of N reads N items.

### `summary`

Scored in code for the background summary: `urgency` 0 to 5, `action_required` (3 or more), `reasons[]`, `hazards[]` (`reason`, `first_at`, `peak`, `unit`), `now`, `next_24h`, `watch_entities[]`. It reads the next 24 hours (hourly, else the next two days).

The weather KFC is on the **ambient tier**: Trapper Keeper digests urgency 0 to 3, and 4 or 5 is promoted to a line in the attention list. So only what needs action scores 4 or more:

| Reason | Score |
|---|---|
| `storm` (lightning or hail), `damaging_wind`, `hard_freeze`, `extreme_heat` | 4 |
| `freeze`, `heat`, `high_wind` | 3 |
| `snow_expected` | 2 |
| `rain_expected` | 1 |
| `weather_unavailable` (the source is unavailable) | 2 |
| `no_weather_source` | 1 |

---

## Decisions (KF6.5)

Weather is a host for [Oracle](zen_dojotools_oracle_readme.md). Both templates are whole-house: an `area` is `unsupported_scope`. Decide through Oracle: `zen_dojotools_oracle mode=decide host=weather decision_template=<id>`.

| Template | Question | Outcomes | Settled without a model |
|---|---|---|---|
| `forecast_changed` | Has the forecast changed enough since the last weather summary that a fresh one is warranted? | `refresh` / `keep` (refreshes the `weather` kata) | No prior weather summary: `refresh` |
| `hazard_relevance` | Do the forecast's hazards matter to the household right now? | `attention` / `nominal` (a judgment only, no summary) | No hazard: `nominal`. A hazard at urgency 4 or more: `attention`, so a severe hazard is never judged away |

Both policies use one `noul` question at 0.6 and 0.35. With no weather source at all, either template answers `insufficient_evidence` without a model.

**Evidence** (`decision_bind` runs `mode=summary`):
- `forecast_changed`: the last `weather` kata (urgency, attention, inference, age) beside the summary now (urgency, reasons, hazards, conditions, the next 24 hours).
- `hazard_relevance`: the hazards with when they start, conditions now, the next 24 hours, and the household's situation (home mode, people home, local time).
- `evidence_revision`: urgency and each hazard with its start, plus the next day's high, low and condition (`forecast_changed`) or the home mode and people home (`hazard_relevance`).

**Certs:** `zenos.decision.use`, `zenos.decision.component.weather` (level 1, declared in Weather's `tool_manifest`) and `zenos.weather.read`, plus the operator's decision switch.

---

## Labels

| Label | On | Meaning |
|---|---|---|
| `main_weather` | the weather entity | The source. The older `zen_weather` label is still read when `main_weather` is not set. Without either, the first `weather.*` entity is used. |
| `outdoor_temperature`, `outdoor_humidity`, `outdoor_wind_speed`, `rain_rate`, `uv_index` | a local station sensor | When present, its value replaces the provider's for that field in `current` (named in `source.station`). |

---

## Certification

`zenos.weather.read` (level 1, stages `i`, `s` and `l`) gates only the background seed (`summary`) and a live drawer mounted on it (stage `l`). Asking in conversation needs no cert. It is declared in `tool_manifest` with `seed_modes: summary@zenos.weather.read:1`, and the KFC's `meta.requires` names the same cert. Grant it with `persona_editor mode=cert_req_grant cert_component=zenos.weather.read cert_stages=isl`. Since 1.0.4 it is in the `zenos_agent_basic` bundle, so the onboarding grant covers it.

---

## Replacing the house script

The house forecast script (`script.weather_forecasts`, TheFes) is hardcoded to one weather entity and returns only the averaged object. `zen_dojotools_weather mode=forecast` does everything it did (its field names still work) and finds the entity by label. Nothing is removed until parity is proven and the household says so; until then the house script stays.
