# 8. Cabinets as Graph

Labels describe the house. Cabinets hold what ZenOS itself knows and remembers: identities, preferences, component definitions, summaries, certifications, configuration. This chapter describes cabinets as the second half of the graph, and FileCabinet as the only way to read and write them.

Cabinets exist because Home Assistant offered exactly one place to keep durable text: a trigger-based template sensor holding values in its attributes. Most of the design follows from that constraint. The real question was never how to store facts. It was how to store context: what a fact relates to, who it belongs to, why it matters.

Some context is small enough for a drawer. Some is a whole elephant, a person or a place with a history, and you do not stuff an elephant into a drawer. You store a picture of it, a pointer to the full thing, and let context be inherited by following links. The manifest started life as the library's card catalog.

## 8.1 What a cabinet is

A cabinet is a Home Assistant template sensor whose `variables` attribute holds a set of drawers. Each drawer is a key with a value:

```text
{ value: <anything: string, dict, list>, timestamp: <set automatically> }
```

Every cabinet carries a header drawer, `AI_Cabinet_VolumeInfo`: the cabinet's GUID, version, type flags, and a validation signature that marks the sensor as a genuine cabinet rather than any template sensor that happens to have a `variables` attribute. The header is protected. No garbage collector, summarizer, or ordinary write touches it.

Cabinets have types, and types are labels (Chapter 18): household, family, user, AI persona, plus system cabinets for the Dojo (component definitions), the Kata (summaries), and the system itself. Expansion cabinets provide overflow space.

## 8.2 Finding a cabinet: the Highlander resolvers

Code never looks up a cabinet by label at runtime. Each core cabinet (dojo, kata, system, household, family, user, AI persona) has a trigger-based resolver sensor, `sensor.zen_*_cabinet_resolved`, whose state is that cabinet's entity ID. The resolvers update on Home Assistant start, on label changes, and on an explicit `zen_resolver_refresh` event. Every piece of OS code reads cabinet entity IDs from these sensors. There can be only one source of truth per cabinet.

## 8.3 Drawers form a graph

Three things make a cabinet a graph rather than a key-value store.

**Nesting.** A drawer key may contain `/`, and FileCabinet treats it as a path. `recipes/weeknight/tacos` is a child of `recipes/weeknight`. Reading a drawer with children reports them. Moving or deleting a drawer cascades to its children, and a move verifies every child's write at the destination before deleting the source.

**Mounts.** A VirtualDrawer is a mount point: its value names a drawer in another cabinet, and reading it transparently returns the target's value. One drawer can therefore appear in many cabinets without being copied, the same way a filesystem link does.

**Live drawers.** A LiveDrawer is a mount whose target is a tool call. Its value carries `fc_args`: a tool, its parameters, and a cache policy. Reading it calls the tool and caches the result in a `cache` sub-drawer. When the cache is cold, it returns the cached value and marks the read stale. A live drawer is never empty. This is how a KFC definition can live in a tool and still appear in the Dojo cabinet (Chapter 13).

```mermaid
graph LR
  subgraph HH["Household cabinet"]
    VI1["AI_Cabinet_VolumeInfo"]
    PREF["preferences"]
    RT["room_topology"]
    MNT["shared_list (VirtualDrawer)"]
  end
  subgraph FAM["Family cabinet"]
    LIST["grocery_list"]
  end
  subgraph DOJO["Dojo cabinet"]
    KFC["alert_manager (LiveDrawer)"]
    KFCC["alert_manager/cache"]
  end
  TOOL[["zen_dojotools_alertmanager<br/>mode=kfc_manifest"]]
  MNT -. mount .-> LIST
  KFC -. fc_args .-> TOOL
  KFC --- KFCC
```

## 8.4 FileCabinet

There are exactly two licensed paths to cabinet data, and they never cross:

| Path | Who uses it | For |
|---|---|---|
| `script.zen_dojotools_filecabinet` | Scripts, automations, agents | Every read and write |
| The CABS macros in `zenos_cabinets.jinja` | Templates, and only inside `zen_sutra_filecabinet` | The terminus's own reads |

`zen_dojotools_filecabinet` is the agent-facing tool. `zen_sutra_filecabinet` is the internal terminus that does the work, and nothing outside it calls it directly. FileCabinet is also the surface for the wiki: `stack=wiki` routes to the Wiki.js sutra instead of a cabinet.

Every write goes through a health-aware pipeline. Before writing, FileCabinet checks that the cabinet is out of `init` (a cabinet with no identity cannot hold data), that the volume is healthy, that the GUID matches, that the cabinet or drawer is not read-only, and that the schema is compatible. Any failure blocks the write unless the caller passes `force_action: true`. The `init` check cannot be forced, and the system cabinet is permanently read-only.

A write reports `write_verified`: whether the value actually persisted when read back. This matters because Home Assistant can accept a write and not persist it, for example when a value exceeds the attribute size it will store. A caller that needs to know the write landed checks `write_verified`, not just `status`.

Reserved drawers (keys beginning with `_`, and the header) are protected from ordinary writes and from garbage collection. `zen_dojotools_filecabinet_gc` removes expired drawers on a schedule and leaves reserved ones alone.

> **Not yet built.** FileCabinet has no certification gate. Because every other part of ZenOS depends on it at boot, gating it needs its own security-class design rather than a bolt-on, and changes to it have to land without breaking a single running install. FileCabinet's single-exit pass, envelope, and certification gate are the scope of 2026.11.0 'This Is Spinal Tap'. That work includes two rules for the graph: following a mount confers no authority, so a target reached through soft links is checked exactly as if it were reached directly, and security is resolved before a LiveDrawer's tool call runs (Chapter 21).

Between them, labels and cabinets say what every thing is and what the system knows about it. Neither says how the rooms of the house relate to each other, which is the last piece of the graph.

<!-- where -->
## 8.5 Where to look

Every claim in this chapter can be checked in the code. These are the places to start.

* Cabinet schema, core cabinets, and volume routing: [`zenos_cabinets.yaml`](../../../packages/zenos_ai/zenos_cabinets.yaml) (`variables`). Docs: [readme.md](../cabinets/readme.md).
* Safe drawer reads, including mounted drawers: [`zenos_cabinets.jinja`](../../../custom_templates/zenos_ai/zenos_cabinets.jinja) (`macro cabinet_drawer_value_mounted`). Docs: [zenos_cabinets_jinja.md](../custom_templates/zenos_cabinets_jinja.md).
* The agent-facing cabinet tool, which checks health before a write and confirms it persisted after: [`dojotools_filecabinet.yaml`](../../../packages/zenos_ai/dojotools/dojotools_filecabinet.yaml) (`zen_dojotools_filecabinet`, `write_verified`). Docs: [zen_dojotools_filecabinet_readme.md](../scripts/zen_dojotools_filecabinet_readme.md).
* The Highlander resolvers every tool reads instead of searching: [`zenos_summarizer_system_health.yaml`](../../../packages/zenos_ai/sensors/zenos_summarizer_system_health.yaml) (`zen_default_household_cabinet_resolved`). Docs: [readme.md](../sensors/readme.md).
* The index shows a drawer's description and a 64-character preview; the full value always comes from FileCabinet: [`dojotools_index.yaml`](../../../packages/zenos_ai/dojotools/dojotools_index.yaml) (`truncated to 64 chars`, `use FileCabinet for full data`). Docs: [zen_dojotools_index_readme.md](../scripts/zen_dojotools_index_readme.md).
<!-- /where -->

<!-- nav -->
---

[← Labels and the Hypergraph](07_labels_and_the_hypergraph.md) · [Contents](00_toc.md) · [Doc hub](../readme.md) · [Spatial Topology →](09_spatial_topology.md)
<!-- /nav -->
