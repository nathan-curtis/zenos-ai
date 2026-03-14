# Zen DojoTools History — 4.2.0

*Recorder statistics query engine for HA long-term statistics*

---

## Overview

`zen_dojotools_history` provides structured access to Home Assistant's Recorder statistics database. It returns time-bucketed historical data for sensors — energy totals, temperature means, flow rates, whatever your sensors track over time.

It does not return raw event logs, state change history, or logbook entries. It queries the **statistics** layer — the aggregated, long-term data HA stores for sensors with a valid `state_class`.

The script will not create, update, or delete history. That's intentional and permanent.

---

## Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `action_type` | select | `help` | `read` or `help` |
| `statistic_ids` | entity (multi) | — | Sensors to query — must have `state_class` and `unit_of_measurement` |
| `start_time` | datetime | — | Query window start (ISO 8601 UTC) |
| `end_time` | datetime | — | Query window end (ISO 8601 UTC) |
| `period` | select | `hour` | `5minute`, `hour`, `day`, `week`, `month` |
| `types` | select (multi) | — | `change`, `last_reset`, `max`, `mean`, `min`, `state`, `sum` |
| `units` | object | — | Optional unit conversion map (e.g., `{"Wh": "kWh"}`) |
| `context` | text | — | Optional label for trace/logging |

---

## Actions

### `read`

Queries the Recorder database. Returns the statistics object directly from `recorder.get_statistics`.

**Required:** `statistic_ids` and `types` must both be non-empty.

```yaml
action_type: read
statistic_ids:
  - sensor.energy_meter_main
period: day
types:
  - change
start_time: "2026-03-07T00:00:00+00:00"
end_time: "2026-03-14T00:00:00+00:00"
units:
  Wh: kWh
```

---

### `help`

Returns the full field reference, usage notes, and examples. Run this first if you're unsure which `types` value to use.

---

## Choosing the Right `types` Value

| Type | Use For |
|---|---|
| `change` | Cumulative sensors — energy meters, water flow totals. Returns the delta over the period. |
| `mean` | Averaged conditions — temperature, humidity, power draw. |
| `min` / `max` | Peaks and troughs within each period. |
| `state` | Last state value in each period bucket. |
| `sum` | Running total at end of each period (cumulative sensors). |
| `last_reset` | Timestamp of last counter reset (utility meters). |

For **energy meters**: use `change` — it gives you consumption per period, not the raw meter reading.

For **temperature/humidity**: use `mean` — it averages across the period bucket.

---

## Sensor Requirements

The sensor must have:
- `state_class` set to `measurement`, `total`, or `total_increasing`
- `unit_of_measurement` defined
- Statistics enabled in Recorder settings (Settings → System → Recorder)

Check a sensor's statistics eligibility in Developer Tools → States → look for `state_class` in attributes, or use `zen_dojotools_inspect`.

---

## Period Guidance

Avoid `5minute` and `hour` periods for long time ranges — the result set will be large and may exceed the AI's context window. Use the largest period that answers the question:

- **Energy for the past week?** → `period: day`
- **Temperature trend this month?** → `period: day` or `week`
- **Peak power in the last hour?** → `period: 5minute` (short range only)

---

## Why No Create / Update / Delete

The script rejects writes with named error codes:

| Action | Error | Code |
|---|---|---|
| `create` | Temporal Paradox | H-0001 |
| `delete` | Redaction Denied | H-0002 |
| `update` | Zen Violation | H-0003 |

History is immutable. This is not configurable.
