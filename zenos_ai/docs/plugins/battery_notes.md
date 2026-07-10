# ZenOS-AI Battery Notes Plugin

**Version:** 1.2.1
**Package:** `packages/zenos_ai/plugins/battery_notes/battery_notes.yaml`
**Lens Bus stack provider:** `zen_stack_battery` (provider_key: `battery_notes`, internal — not MCP-exposed)
**HALMark:** PASS 2026-07-04

> **Auto-registration (2026.7.0):** `zen_stack_battery` participates in Lens Bus bootstrap registration. On every HA boot, `zen_dojotools_manifest mode=bootstrap_stacks` discovers and registers it automatically — no manual `lens_registry` writes needed. See [Lens Bus Auto-Registration](lens_bus_autoreg.md).

---

## Overview

Battery Notes is the battery-health surface for ZenOS-AI. Unlike the other Lens Bus stack providers (Firefly, Paperless-NGX, WikiJS), it does not talk to a self-hosted backend service — it reads attributes off sensors created by a **third-party HACS integration**:

**[Battery Notes](https://github.com/andrew-codechimp/ha-battery-notes)** (`andrew-codechimp/ha-battery-notes`) must be installed via HACS and configured against your devices. It creates a `battery_plus` sensor per tracked device with `battery_type`, `battery_quantity`, `battery_low`, `battery_last_replaced`, and `battery_last_reported` attributes. `zen_stack_battery` is a thin ZenOS adapter over those sensors — it does not track battery state itself, and without the HACS integration installed there are no `battery_plus` sensors to read and evidence is always empty.

This is why it's a hybrid: the ZenOS-side wiring (Lens Bus contract, KFC component, cabinet-free design) is native, but the underlying data source is an external integration outside ZenOS's control.

Domain is `maintenance` (not `person`/`label` like the other stack providers) — it answers "what needs a battery," not "what belongs to whom."

---

## Lens Bus Integration

`zen_stack_battery` participates in the Lens Bus as a provider of `battery_evidence`, anchored on `area_id` only (no label/person/zone anchor support).

| Property | Value |
|----------|-------|
| Consumes | `area_id` |
| Returns | `battery_evidence` |
| Security | `r-only` (`rwxda` model: `required: r`, `allowed: [r]`, `denied: [w, x, d, a]`) |
| Failure policy | `soft` |
| Content policy | `standard` |
| Risk class | `read_only` |

### Modes

| Mode | Description |
|------|-------------|
| `stacks_by_anchor` | Fetch `battery_evidence` for `area_id` anchors — per-device battery level, type, quantity, low status, days since replaced |
| `kfc_manifest` | Returns the `battery_health` KFC component definition for Ninja Summarizer wiring |
| `register` | Register this provider in the household `lens_registry` cabinet drawer |
| `unregister` | Remove this provider from `lens_registry` |
| `health` | Reports whether any `battery_plus` sensors exist at all (i.e. whether the HACS integration is installed/configured) |
| `inspect` | Static capability descriptor — consumes/returns/security_act/description |
| `audit` | Read-only compliance self-attestation — `security_violations` + notes |
| `tool_manifest` | Full tool manifest declaration (`tier: stacks`, `provider_key: battery_notes`, `register_mode: register`) |
| `help` | Mode listing (default when called with no mode/action) |

`mode` is the primary selector; `action` is accepted as an alias for lens-dispatcher compatibility, per the project-wide move to standardize all dojotools/stack calls on `mode:`.

### Battery evidence shape

```json
{
  "id": "batt_kitchen_sensor_kitchen_smoke_detector_battery_plus",
  "type": "battery_evidence",
  "provider": "battery_notes",
  "area_id": "kitchen",
  "entity_id": "sensor.kitchen_smoke_detector_battery_plus",
  "device_name": "Kitchen Smoke Detector",
  "battery_pct": 12,
  "battery_type": "CR123A",
  "battery_quantity": 1,
  "battery_type_and_quantity": "1x CR123A",
  "battery_low": true,
  "battery_low_threshold": 15,
  "last_replaced": "2026-01-15",
  "last_reported": "2026-06-30T08:00:00",
  "days_since_replaced": 170,
  "anchor_type": "area_id",
  "anchor_id": "kitchen",
  "content_redacted": false
}
```

`days_since_replaced` is `null` when `last_replaced` is empty or unparseable — it is never computed against a bad timestamp.

**Serialization note (v1.2.1):** the evidence list is round-tripped through `tojson`/`from_json` explicitly rather than relying on HA's implicit native-type coercion. A native `datetime` value in `battery_last_reported` was previously breaking that implicit round-trip, silently turning the whole evidence list into a string — which the Lens dispatcher's `is not string` guard then dropped entirely, so `stacks_by_anchor` looked like it had matched nothing even when devices matched. Both `battery_last_replaced` and `battery_last_reported` are explicitly stringified before being placed in the evidence dict for this reason.

### Using via Lens Dispatch

```yaml
zen_dojotools_lens_dispatch:
  anchor_type: area_id
  anchor_ids: '["kitchen", "living_room"]'
```

Battery Notes is queried automatically alongside every other registered provider for those anchors — there is no dedicated public MCP surface for it (matches the "internal only" declaration in the file header).

---

## KFC Component: `battery_health`

Seeded via `zen_dojotools_inventory mode=battery_status`, summarized daily at midnight and noon. Surfaces devices with low batteries grouped by area, cross-referenced against Grocy disposable stock to show whether replacements are on hand. See `kfc_manifest` mode for the full component definition (drift threshold 24h, cooldown 24h).

---

## Known Gaps

- **External dependency risk**: because the data source is a HACS integration, not a ZenOS-controlled backend, `health` mode reporting "0 battery_plus sensors" can mean either "integration not installed," "integration installed but no devices configured," or "integration briefly unavailable" — it does not currently distinguish between these.
- `stacks_by_anchor` only supports `anchor_type: area_id` — devices without an HA area assignment will never surface as evidence for any anchor.

---

## Dependencies

| Dependency | Purpose |
|------------|---------|
| HACS **Battery Notes** integration (`andrew-codechimp/ha-battery-notes`) | Source of all `battery_plus` sensors and attributes — required, not optional |
| `zen_dojotools_filecabinet` | `lens_registry` reads/writes for register/unregister |
| `Zen Household Cabinet` (label `zen_household_cabinet`) | Required for register/unregister — both modes error cleanly if no cabinet entity is found |
