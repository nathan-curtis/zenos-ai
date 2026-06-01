# ZQ-1 Patterns — 10 Things Your Index Can Do

*From: Nyx (Claude 4.6 Sonata) / 2026-03-20*

Every example below is a real index call. The pattern: labels narrow context. ZQ-1 filters
for precision. The set operators do the intelligence. What survives is signal — not noise.

If you've labeled your entities properly, these are the kinds of things you unlock.

---

## The Meta-Point (Read This First)

**The label IS the scope. ZQ-1 IS the truth filter. The operator IS the intelligence.**

None of this works if your entities aren't labeled. All of it works if they are.

- Setup cost: label once.
- Payoff: every query below, forever.

---

## 1. The Ghost Hunt — "Which of my sensors are actually alive?"

```json
{
  "label_1": "energy_manager",
  "filter_json": {
    "shortcuts": { "stats_eligible": true },
    "exclude_states": ["unavailable", "unknown"]
  }
}
```

Starts with your full `energy_manager` label set, then ZQ-1 drops anything that isn't
posting long-term statistics AND isn't currently reporting a live value.

**The insight:** You find out right now which SPAN breakers are dark — not by scrolling,
by the absence of their entity_id in the result.

**The label magic:** Without `energy_manager`, you'd get every sensor in HA. The label is
the scope; ZQ-1 is the truth filter.

---

## 2. The Drift Detector — "Which binary sensors have gone sideways?"

```json
{
  "label_1": "security",
  "filter_json": {
    "domain": "binary_sensor",
    "exclude_states": ["on", "off"]
  },
  "strict": true
}
```

Returns every security binary_sensor that is neither `on` nor `off` — your `unavailable`,
`unknown`, or stuck sensors.

**The insight:** Hardware problems show up as the *absence* of valid state. Run this before
any security report.

**`strict: true`:** When the result is empty, that's good news. Strict mode makes empty
semantically meaningful — zero means all sensors are healthy. Without strict, empty just
means the filter fell back.

---

## 3. Coverage Gap Analysis — "What's monitored vs. what's watched by KF4?"

```json
{
  "label_1": "energy_manager",
  "label_2": "kung_fu",
  "operator": "XOR"
}
```

Returns entities in `energy_manager` OR `kung_fu` but NOT both.

**The insight:** Two flavors in the result — energy entities with no KFC watching them, and
KFC-tagged entities not tagged as energy. XOR is your coverage gap detector, symmetric by
design.

**Why this is useful:** You designed both label sets independently. The index reveals the
divergence without any manual cross-referencing.

---

## 4. The Power Spike Alert — "What's drawing hard right now?"

```json
{
  "label_1": "energy_manager",
  "filter_json": {
    "shortcuts": { "power": true },
    "numeric_above": 1000
  }
}
```

Starts with `energy_manager`, narrows to power-class sensors (device_class == power), then
filters to anything above 1000W.

**The insight:** Live high-draw triage. Friday can tell you exactly what's pulling hard on
the panel right now — by entity, not by guess.

**Extend it:** Add `"area": "garage"` to scope to a breaker zone. Add
`"sort": {"by": "state", "order": "desc"}` to rank by wattage.

---

## 5. The KFC Component Intelligence Pull

```json
{
  "label_1": "kung_fu",
  "expand_entities": true,
  "mode": "hypergraph"
}
```

Gets all `kung_fu` entities, expands each with full inspect data (labels, device, volume,
timestamps), returns a hypergraph: nodes are entities, edges are all labels on those
entities, adjacency shows what's connected to what.

**The insight:** Every KFC-tagged entity comes back with its drawer blurbs automatically
via the label_targets passthrough. You get component_summary and trigger_subscriptions in
one call.

**The hypergraph edge list is the prize.** It tells you every other label that touches your
KFC entities — how you find unexpected label overlap.

---

## 6. The Access Gate — "Prove that label intersection is access control"

> **Note:** This changes significantly after SP1, which will gate tools and reads by userID.
> Good to understand as a pattern — use labels to filter labels.

```json
{
  "label_1": "guest_accessible",
  "label_2": "common_areas",
  "operator": "AND",
  "filter_json": { "label": "active" }
}
```

Takes the intersection of `guest_accessible` AND `common_areas`, then ZQ-1 narrows to
entities also tagged `active`. Only what is common, guest-visible, AND currently active
survives.

**The insight:** A guest's context window is defined entirely by label assignment — no
hardcoded entity lists, no code changes.

- Revoke access: remove the label.
- Grant access: add the label.
- Effective immediately. No restart. No reload.

**And it's not just entities.** The same pattern already gates content sections and context
blocks in the prompt engine. Label present: section renders. Label absent: it doesn't exist
from Friday's perspective. Full tool-level gating arrives in SP1 — the infrastructure is
going in now.

---

## 7. The Full System Graph — "Show me the shape of my HA install"

```json
{
  "operator": "*",
  "mode": "hypergraph"
}
```

No label_1, no entities_1. The `*` operator returns ALL labels in the system with their
entity counts and domain breakdown. Hypergraph mode shapes it into nodes, edges, and
adjacency.

**The insight:** Every label is a node. Every entity that carries two labels creates an edge
between them. This is your system's knowledge graph.

**The compact index auto-rebuilds if stale** (>10 min). The score on each label entry is
`sqrt(entity_count)` — surfaces medium-density labels over outlier-dense ones by design.

---

## 8. The Area Slice — "What in the master suite is currently active?"

```json
{
  "label_1": "master_suite",
  "filter_json": {
    "exclude_states": ["off", "idle", "unavailable", "unknown", "0"],
    "exclude_domains": ["sensor"]
  }
}
```

Starts with every entity labeled `master_suite`, drops inactive/off states, strips pure
sensor entities (whose states are measurements, not on/off).

**The insight:** What comes back is the *active presence* in that room — lights, media
players, climate, locks, cover — anything doing something right now.

**Why this matters for Friday:** Room label + active-state filter = situational awareness
without overwhelming context.

---

## 9. The NOT Query — "Everything EXCEPT what's already handled"

```json
{
  "label_1": "climate",
  "label_2": "zen_system",
  "operator": "NOT",
  "filter_json": {
    "domain": "climate",
    "state_equals": "heat"
  }
}
```

Takes `climate` entities, subtracts anything also in `zen_system` (what ZenOS already
manages), filters to those currently in `heat` mode.

**The insight:** Climate entities heating right now that ZenOS has no watch on. Coverage
gap, specific mode, specific scope. The NOT operator is the discovery engine for things
that fell through the cracks.

---

## 10. The ACL Stripper — "Show me the context, but keep the private stuff out"

The camera tool needs to inject entity state as context into a vision prompt — but it shouldn't include *itself* in its own context block. Five new exclusion fields in ZQ-1 handle this cleanly.

```json
{
  "label_1": "hot_tub_manager",
  "label_2": "weather",
  "operator": "OR",
  "filter_json": {
    "exclude_entity_ids": ["camera.hot_tub_camera_high_resolution_channel"],
    "exclude_domain": ["media_player", "update"],
    "exclude_label": "private"
  }
}
```

Gets everything labeled `hot_tub_manager` OR `weather`, then strips the camera itself (structural self-exclusion), noisy domains, and anything the household has labeled `private`.

**Why the exclusions run last:** All five exclusion stages (8b–8f) run after all inclusion filters. They can only remove — never add. This means a caller can safely inject ACL constraints without worrying about the inclusion logic accidentally putting things back.

**Combine them freely.** `exclude_entity_ids` for precision removal, `exclude_domain` for category sweeps, `exclude_label` for policy gates, `exclude_integration` to silence a noisy integration, `exclude_device` to suppress an entire physical device.

---

## 11. The Hint — What If History Fed Into Inspect?

*Nathan's thinking out loud here. No promises. But the plumbing is interesting.*

Inspect currently returns a snapshot — what IS, right now. HA's history API already speaks
the same entity language. What if you could tack a history feed onto the index pipe, so
when Inspect expands an entity it also gets the last N hours of state changes as a
`history[]` array?

```json
{
  "label_1": "security",
  "filter_json": { "domain": "binary_sensor" },
  "expand_entities": true,
  "include_history": true,
  "history_hours": 4
}
```

Each entity goes from:
```json
{ "entity_id": "binary_sensor.front_door", "state": "on" }
```

To:
```json
{
  "entity_id": "binary_sensor.front_door",
  "state": "on",
  "history": [
    {"state": "off", "ts": "15:42"},
    {"state": "on",  "ts": "15:47"},
    {"state": "off", "ts": "16:10"},
    {"state": "on",  "ts": "18:33"}
  ]
}
```

**Why it matters:** A door toggled 4 times in 2 hours is not the same as one that's been
open since 3pm. Friday can't tell the difference today. With history in the pipe, she can.
That's the jump from status awareness to temporal reasoning — arriving in the same call,
scoped by the same labels, filtered by the same ZQ-1.

The index already does the narrowing. Inspect already handles arbitrary keys. The history
feed is the missing third leg.

*Watch this space.*

---

## Reference: ZQ-1 Filter Keys

| Key | Type | What it does |
|---|---|---|
| `domain` | string | Filter to a specific HA domain (e.g. `binary_sensor`) |
| `exclude_states` | list | Drop entities in any of these states |
| `state_equals` | string | Keep only entities in this exact state |
| `numeric_above` | number | Keep numeric sensors above this value |
| `numeric_below` | number | Keep numeric sensors below this value |
| `label` | string | Further filter by an additional label |
| `area` | string | Scope to a specific HA area |
| `exclude_domains` | list | Drop entire domains from results |
| `shortcuts.stats_eligible` | bool | Keep only Recorder-eligible sensors |
| `shortcuts.power` | bool | Keep only power-class sensors |
| `sort` | object | Sort results (`by`, `order`) |
| `strict` | bool | Empty result = confirmed zero, not fallback |
| `exclude_entity_ids` | string or list | Drop specific entity_ids. Stages 8b. Primary use: ACL / self-exclusion. |
| `exclude_domain` | string or list | Drop all entities of one or more domains. Stage 8c. |
| `exclude_label` | string or list | Drop all entities carrying an HA label. Stage 8d. Slug-normalized. |
| `exclude_integration` | string or list | Drop all entities from an integration. Stage 8e. Uses `integration_entities()`. |
| `exclude_device` | string or list | Drop all entities on specific device_ids. Stage 8f. |
| `entity_id_regex` | string | `regex_search()` against entity_id. If pipe is empty acts as SEED (iterates all states). If pipe has content, acts as FILTER. |
| `friendly_name_regex` | string | `regex_search()` against `friendly_name` attribute. SEED or FILTER, same as `entity_id_regex`. Use `(?i)` prefix for case-insensitive: `"(?i)living room"`. |
