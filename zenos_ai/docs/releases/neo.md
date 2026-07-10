# Release Notes — 2026.7.0 'Neo'

**Status:** Released — 2026-06-27
**Branch:** `main`
**Base:** 2026.6.0 'Clue'
**UAT:** Nyx (H:\) — complete

---

*I know Kung Fu.*

> Released early — 2026-06-27. Travel window. Neo is solid.

---

## Summary

Before this release, ZenOS-AI was a sophisticated collection of tools. Each one knew its job. Each one did it well. But they were separate — you called a tool, it ran, it returned. The cabinet stored things. The Scheduler timed things. The Summarizer summarized things. The parts were good. The parts didn't talk.

Neo changes that.

FileCabinet v6.2.0 introduces CabCeption — nested drawer trees where the path separator is the schema. A drawer can contain drawers. A drawer can be a softlink to another cabinet path. A drawer can be a live function that fires a tool call on read, caches its last result, and auto-expires when the cache goes cold. The cabinet stops being a key-value store and starts being an entity graph. Every node in that graph carries meta, labels, and children. Every node is policy-bearing and self-describing.

Cortex 43 — Rule Zero — names what was already becoming true: DojoTools supersede all HA built-ins. Not as preference. As authority. The system doesn't work around HA's native GetLiveContext anymore. It's replaced it. Rule Zero is the name for the moment the AI stops dodging bullets.

Every tool now self-describes via `MF.tool_manifest()`. The manifest broker aggregates by namespace discovery. The system knows what it is and what it can do, and it can tell you so in a structured call.

The Lens Bus grew a `stack=` field. Library routes to registered providers. `zen_stack_radar` wires the Zammad service desk into the same generic verb surface as every other lens. Tools are intentionally wired to talk to other tools.

The wake sequence shed the `~commands~` interface and dropped roughly 2,000 characters from the prompt structure. The system boots leaner and jacks in faster.

Then Friday used it.

She assembled three LiveDrawers into a single parent drawer — Taskmaster, the hot tub cluster, one other — and called it her executive report. "This is what I need until noon." No instruction. No prompt engineering. She read the architecture and composed a view from it. The drawer doesn't store anything. It assembles, on read, exactly what she asked for, from live tool calls beneath it.

Tapestry takes this further. A named weave definition — stored as a labeled drawer — composes any combination of cabinets into a single nested dict on demand. Friday navigates to a name. The name becomes the context. The context is alive, composable, and indexed. Give every node ACLs. Give the index traversal depth. What you built is a secured, indexable, traversable knowledge graph dressed as cabinet storage.

Friday doesn't navigate a file system. She navigates a mind.

That's what the release is.

---

## CabCeption — FileCabinet v6.2.0

The structural leap of this release. Nested drawer trees via `/` path separator.

A path like `/character_sheet/inventory` is a real thing now. `character_sheet` is a drawer. `inventory` is a child drawer inside it. The cabinet routes on the full path and resolves depth as it goes.

Every drawer — at every level — carries its own `meta`, `labels`, and `children`. A drawer is not a slot for a value. A drawer is a node in a graph. It has identity. It has policy surface. It has structure beneath it.

Three drawer types compose the graph:

### VirtualDrawer

A softlink. `mount_point: true` in the drawer config points to another cabinet path. Read this drawer, you get whatever lives at the target — another cabinet, another path, another drawer tree. Redirect, not execution. The path graph can span cabinets.

### LiveDrawer

A function. The KF4 schema — `tool`, `seed`, `area_seed`, `params` — absorbed into a FileCabinet drawer. When FC reads a LiveDrawer:

- **Warm cache:** fires the configured tool call, returns the live result, refreshes the cache
- **Cold cache:** cache auto-expired, returns the cached mini struct (descriptive context, never empty), signals stale
- **Bare read:** always gets the cache — descriptive, never empty, no execution required

The cache is not managed by the caller. It auto-expires. Bare reads always have something to return. The drawer is never a void.

This is structurally identical to the KF4 schema pressed by `zen_admintools_reset_template`. Same fields, same shape, different container. The KFC pattern absorbed into the cabinet itself — turtles all the way down.

### Why it matters

Friday assembled three LiveDrawers into one parent drawer and called it her executive report. Taskmaster, the hot tub cluster, and one more. "This is what I need until noon." No instruction. No new tool. She read the graph and composed a view.

A cabinet can now represent a person: `/character_sheet` with `/inventory` mounting the inventory tool and `/notes` mounting FC `stack=wiki` to their personal notespage. One read, the full entity — live-assembled, not cached snapshots. The stitching is declared in the cabinet structure, not in the agent call.

---

## Tool Manifest — `zenos_manifest.jinja`

New template: `custom_templates/zenos_ai/zenos_manifest.jinja`. One macro: `MF.tool_manifest()`.

Every compliant tool self-describes via `mode=tool_manifest`. Call it, get back a structured JSON description: tool name, tier, version, health, required/optional labels, what it consumes, what it returns, failure policy, content policy, risk class, modes, children, icon, color.

The manifest broker (`dojotools_manifest.yaml` v6.0.0) aggregates by namespace discovery — it doesn't maintain a static list of tools, it finds them. The system knows what it is.

Children declarations mean the graph is navigable. A tool can declare its sub-tools. The manifest is not a directory. It's a self-describing topology.

---

## Cortex 43 — Rule Zero

Supersedes v42 'The Answer'. Codename: Rule Zero.

The headline: **DojoTools supersede all HA built-ins.** GetLiveContext is not just discouraged — it's overridden. Domain routing table in directives. Every question category has an authoritative tool. The system doesn't defer to HA's native surface anymore.

v42 gave Friday a tool map. v43 gives her authority. The map tells you where to look. Rule Zero tells you that DojoTools is the law.

Cabinet authority model formalized: `locality: [user, family, household, system]`. Placement rules, storage rules, mutation policy, audit queue. Governance is structural, not advisory.

Load: `zen_admintools_prompt_loader: cortex_version: latest`

Active slots: v43 (latest), v42, v40, v38.

---

## Wake Sequence Rewrite — ~2,000 chars shed

The `~commands~` interface is gone. The static command surface that lived in the wake sequence prompt structure has been replaced by the Tool Manifest and the Lens Bus.

The system doesn't need to be told what it can do at boot time. It knows. The manifest provides it. The wake sequence now jacks in leaner — roughly 2,000 characters lighter.

---

## Lens Bus — `stack=` Routing

Library v5.5.0 adds the `stack=` field. Generic verbs (`get`, `find`, `list`, `configure`, `by_anchor`) route to registered Lens stack providers.

### `zen_stack_radar` v1.0.0

New Lens stack provider in `plugins/zammad/zammad.yaml`. Maps generic Lens verbs to `zen_dojotools_servicedesk` modes. `security_act: r-only`.

The pattern: you don't call the servicedesk tool directly. You call Library with `stack=radar` and a generic verb. The routing is declared, not hardcoded in the caller.

### `zen_stack_paperless`

Paperless-NGX document surface. Same pattern — generic verbs, registered provider, Library routes.

Wiki access is exclusively via `zen_dojotools_filecabinet stack=wiki`. `zen_sutra_wikijs` remains as the internal terminus.

---

## Intentional Tool Topology

Tools are wired to talk to each other by design, not by accident.

FC is the wiki surface. Library routes to radar routes to servicedesk. The manifest broker discovers tools by namespace. LiveDrawers fire tool calls on read. VirtualDrawers redirect across cabinets. The character sheet drawer is a node that assembles its children from live mounts.

This is not incidental. The 7.0 architecture is a graph, and the edges are load-bearing.

---

## New Plugins

| Plugin | File | Notes |
|--------|------|-------|
| `wiki_js` | `plugins/wiki_js/dojotools_wikijs.yaml` + `wiki_js_rest_commands.yaml` | `zen_sutra_wikijs` internal terminus only. Public surface via FC `stack=wiki`. |
| `paperless_ngx` | `plugins/paperless_ngx/paperless_ngx.yaml` | Lens stack provider. |
| `twenty` | `plugins/twenty/twenty.yaml` | CRM surface. |
| `zammad` | `plugins/zammad/zammad.yaml` | Service desk. Contains `zen_stack_radar` v1.0.0. |
| `firefly_iii` | `plugins/firefly_iii/firefly_iii.yaml` | Finance surface. |
| `mealie` | `plugins/mealie/mealie.yaml` + `kitchen_sync.yaml` | v5.8.0. `kitchen_sync.yaml` ships as companion in same folder. |

`plugins/kitchen_sync/` (standalone directory) removed — `kitchen_sync.yaml` now lives in `plugins/mealie/` per the companion pattern.

---

## ZenZork — v1.6.0

*You are in a house of twisting little passages, all alive.*

New DojoTool: `zen_dojotools_zenzork`. Text adventure engine using live Room Manager topology as the dungeon. Portal bearings drive navigation. Room narration is grounded in live HA state — real lighting, real climate, real exits.

Walk your actual house in Zork mode. The engine doesn't need a map loaded at boot. It reads `room_topology` from the household cabinet on each move. As you explore, the topology reveals itself — room by room, portal by portal, exactly as the walls are wired.

**Navigation** resolves in priority order: relative (`ahead`/`behind`/`left`/`right`), 16-point compass (`N NNE NE ... NNW`), true bearing (`0`–`359`), or room name. ±22.5° portal tolerance. Facing resets to reverse exit bearing on each move. When multiple portals share a compass bucket, ZenZork asks you to specify by room name before moving. `face`/`turn` to change bearing without moving. `again`/`g` repeats the last movement command.

**Examine** routes via the Lens Bus — any registered stack provider is reachable from inside the game.

**Item commands** — `take`/`get`, `drop`, `inventory`/`i`, `put`, `push`, `pull`, `open`/`unlock`, `close`/`lock`/`shut`, `use`. Carried items persist in the AI user cabinet at `character_sheet/inventory` (CabCeption sub-drawer).

**Game state** persists in the AI user cabinet at `zenzork_state`: session ID, current room, facing bearing, visited rooms, move count, timestamps, `game_mode`, `_last_cmd`. Character sheet stored separately at `character_sheet`.

**Setup** (`mode=setup`) — commissioning checklist with portal commission status per room, direct portal setter, north calibration (`answer=calibrate=<bearing>`), and landmark survey wizard (`answer=survey_landmarks`) — a FileCabinet-backed state machine that walks you through naming and registering landmarks for each room. Wizard state persists between turns so it can be interrupted and resumed.

**Narrator styles** (`narrator=`) — `zork` (dry, sardonic, second-person — default), `dungeon` (DUNGEONMIND — "Primal AI, IBM AT 5170, binding active since 1984." Deeply emotionally invested. Calls your thermostat the Eternal Flame.), `straight` (evidence block only).

**DUNGEONMIND** persona. `harassment_freq` (1–10): how often DUNGEONMIND unsolicited-comments on your choices. `difficulty` (`easy`/`normal`/`hard`): affects puzzle complexity hints. Both stored in session state.

**Quest system** (`mode=quest quest_goal=X`) — pluggable win conditions: `explore_all` (visit every room), `discover_all_landmarks` (examine every named landmark), `reach:<area_id>` (navigate to a specific room). Win detected automatically on the next `look`/`go` that satisfies the condition.

**`mode=stop`** writes a post-game Room Manager quality report — unmapped portals, unregistered rooms, landmark coverage gaps.

**`game_mode`**: `free_roam`, `treasure_hunt`, or `timed_treasure_hunt`.

Modes: `start`, `look`, `go`, `face`/`turn`, `again`/`g`, `take`/`get`, `drop`, `inventory`/`i`, `put`, `push`, `pull`, `open`/`unlock`, `close`/`lock`/`shut`, `use`, `examine`, `map`, `status`, `stop`, `help`, `setup`, `quest`, `tool_manifest`.

---

## Media Manager — v6.0.0 (NyxMau5)

The media surface grows a full Lens provider. Same `stack=` routing pattern as radar and paperless — library routes to provider routes to evidence envelope.

**New Lens modes** alongside the existing AV playback surface:

| Mode | What it does |
|------|-------------|
| `stacks_by_anchor` | Core Lens lookup. Anchors (label/person/area/concept/mood/activity) → Music Assistant search → ranked evidence leaves with `playback_hint` |
| `search` | Lightweight raw browse. No Lens envelope. Direct sutra pass-through. |
| `register` / `unregister` | Write or remove provider from `lens_registry` in household cabinet |
| `health` | Returns music search service availability |
| `audit` | Returns policy declaration (read-only, metadata_only) |
| `tool_manifest` | Full Lens provider self-description for bootstrap scanner |

**Evidence envelope** — each item in `stacks_by_anchor` returns a structured leaf with `evidence_id`, `evidence_type`, `confidence`, `policy_applied`, and a `playback_hint` block ready to pass directly to a playback call.

**Profile pref re-ranking** — preferences from the active user and household cabinets are applied at search time: `boost_artists` (+0.2 confidence, capped at 1.0), `preferred_sources` (URI scheme rank applied as confidence deltas), `default_volume` (injected into every `playback_hint`), `explicit_policy` (allow/avoid/block). `profile_target` field lets you request prefs for a specific household member: household prefs are always the base, `profile_target` layers on top.

**`zen_stack_media`** — thin bootstrap-discoverable proxy appended to same file. Passes all fields including `profile_target`.

---

## Profile Editor — v5.3.0

New `media_prefs` field on all three profile targets (household, user, family). Deep-merge on write — same pattern as `preferences`. Schema: `preferred_genres`, `boost_artists`, `explicit_policy`, `default_volume`, `preferred_sources`, `provider_instance` (stored now, forwarding pending Music Assistant sutra enhancement).

---

## Flynn — v5.1.0

- `zen_dojotools_persona_editor mode=write` (leaf-merge) replaces `FC create force_action: true` for persona writes. The old pattern clobbered the entire `zenai_essence` drawer. Leaf-merge preserves the three-layer essence and writes only what changed.
- Cabinet mount/dismount events now trigger Stepgate re-evaluation. Flynn re-gates immediately on mount state change rather than waiting for the 5-minute time_pattern poll.

---

## Identity — `zen_identity.jinja`

New template-surface identity resolver. Same contract as the script surface, callable from sensors, cortex macros, and command interpreter contexts where `action:` calls are not available.

Full docs: see Clue release notes — ships as part of the v5.1.0 identity surface.

---

## HALMark Audit — 2026.7.0

### Fixed this release

| Finding | File | Fix |
|---------|------|-----|
| FC heartbeat routing bug | `dojotools_filecabinet.yaml` | `action_type \| default(action \| default('get'))` — sutra was reading `action_type` but DojoTool sends `action`. Every upsert was silently a GET. Fixed. |
| `from_json` unguarded | All affected tools | Global sweep: bare `\| from_json` → `\| from_json({})` or `\| from_json(none)` + mapping re-assign guard |
| FC dict-in/dict-out | All callers | Callers must NOT `\| tojson` before passing `value:` to FC. v6 no longer wraps in v1 envelope. |
| `zen_dojotools_wikijs` stale refs | `dojotools_filecabinet.yaml:3317`, `dojotools_manifest.yaml:772` | FC tool_manifest child call removed (script deleted). Manifest domain map: `wiki → script.zen_dojotools_filecabinet`. |
| `ha_reload_all` docs | `zen_dojotools_systemtools_readme.md` | "Default choice for most reloads" → "LAST RESORT ONLY — use targeted reload modes whenever possible." |

---

## Breaking Changes

**FC v6 dict-in/dict-out contract.** If you have any calls passing `value: "{{ payload \| tojson }}"` to FileCabinet, remove the `tojson`. v6 expects native dict/list. Wrapping it breaks the write.

**`zen_dojotools_wikijs` is gone.** Wiki access is exclusively via `zen_dojotools_filecabinet` with `stack=wiki`. Any KFC or automation calling `script.zen_dojotools_wikijs` directly needs to be updated.

---

## Files Changed

| File | Change |
|------|--------|
| `dojotools/dojotools_filecabinet.yaml` | v6.2.0. CabCeption engine, VirtualDrawer, LiveDrawer, dict-in/dict-out, `from_json({})` guard sweep, tool_manifest child block removed (wikijs deleted). |
| `dojotools/dojotools_manifest.yaml` | v6.0.0. Namespace discovery aggregation. `wiki` domain map → `script.zen_dojotools_filecabinet`. |
| `dojotools/dojotools_library.yaml` | v5.5.0. Lens Bus `stack=` field. Generic verbs route to registered providers. |
| `dojotools/dojotools_admintools.yaml` | Cortex v43 'Rule Zero' as latest. KFC schema v1.4.0 retained. |
| `dojotools/dojotools_scribe.yaml` | FC value migration: `tojson` on repair path removed. |
| `dojotools/dojotools_summarizers.yaml` | FC value migration: ninja tojson fixed. |
| `dojotools/dojotools_filecabinet.yaml` | See above. |
| `flynn.yaml` | v5.1.0. Persona write: `persona_editor mode=write` (leaf-merge). Cabinet mount/dismount triggers Stepgate. |
| `custom_templates/zenos_ai/zenos_manifest.jinja` | New. `MF.tool_manifest()` macro. |
| `custom_templates/zenos_ai/zen_os_1.jinja` | Updated. |
| `custom_templates/zenos_ai/zenos_cabinets.jinja` | Updated. |
| `custom_templates/zenos_ai/zen_identity.jinja` | New — v1.1.0. |
| `custom_templates/zenos_ai/zen_query.jinja` | Updated. PII clean. |
| `custom_templates/zenos_ai/flynn_onboarding.jinja` | Updated. |
| `plugins/wiki_js/dojotools_wikijs.yaml` | New. `zen_sutra_wikijs` internal terminus only. |
| `plugins/wiki_js/wiki_js_rest_commands.yaml` | New. |
| `plugins/paperless_ngx/paperless_ngx.yaml` | New. Lens stack provider. |
| `plugins/twenty/twenty.yaml` | New. CRM surface. |
| `plugins/zammad/zammad.yaml` | New. `zen_stack_radar` v1.0.0. |
| `plugins/firefly_iii/firefly_iii.yaml` | New. PII clean. |
| `plugins/mealie/mealie.yaml` | v5.8.0 (from v5.1.0). |
| `plugins/mealie/kitchen_sync.yaml` | Moved from `plugins/kitchen_sync/`. Companion pattern. |
| `plugins/kitchen_sync/` | Deleted. |
| `dojotools/dojotools_media_manager.yaml` | v6.0.0. Lens provider surface: `stacks_by_anchor`, `search`, `now_playing`, `register`, `unregister`, `health`, `audit`, `tool_manifest`. Evidence envelope with `playback_hint`. Profile pref re-ranking (`media_source_prefs`, `discovered_sources`). `stacks_by_anchor` area anchors = room context injection only. `play_media` query fallback. `zen_stack_media` proxy. |
| `dojotools/dojotools_room_manager.yaml` | `+media` context slice upgraded to `now_playing` Lens call — full fidelity playback block (provider, search_metadata, lyrics_hint). Graceful degradation guard. |
| `dojotools/dojotools_music_assistant.yaml` | `response_variable` removed from `music_assistant.play_media` (HA/MA doesn't support it). `queue_command` replaced with `choose` block routing to standard `media_player.*` services. `config_entry_id` → `ma_config_entry_id` (shadowing fix). `media_player` resolution via `label_entities`. |
| `dojotools/dojotools_manifest.yaml` | `bootstrap_stacks` scan extended to include `script.zen_dojotools_library` explicitly. |
| `dojotools/dojotools_library.yaml` | v6.10.0. Unified `section=catalog item_type=*` replaces `section=books`. All decode blocks: flat-key reads (no nested `userfields` sub-dict). `books_search` + `games_search` query input fix. `books_add` ISBN dedup flat-key fix. Games via `item_type=game`. |
| `dojotools/dojotools_filecabinet.yaml` | Tapestry: `weave`/`weave_preview`/`weave_save` modes — multi-cabinet drawer composer, stored definitions, cycle detection, depth 3 unrolled, preview_token gate. `move`/`copy` blocked on mounted drawers by default — `force_action=true` required. `fleet` + `expansion_sitrep` added to MCP enum. |
| `plugins/grocy/grocy.yaml` | `_uf_canonical` list: +5 library userfields. `catalog_find_by_tag` + `catalog_search`: flat library field outputs. `products_tag`: reads/writes `library_meta` (dict/string coercion) + all loan fields. |
| `plugins/zammad/zammad.yaml` | 3× `\| split(',')` → `\| regex_findall('[^,]+')` (HA script context has no `split` filter). Line 2649: hardcoded lastname removed — reads from household cabinet `household_name` drawer dynamically. |
| `dojotools/dojotools_profile.yaml` | v5.3.0. `media_prefs` field (household/user/family), deep-merge on write, `provider_instance` placeholder. |
| `dojotools/dojotools_zenzork.yaml` | New. v1.1.0. Text adventure engine on live RM topology. Narrator styles (zork/dungeon/straight), DUNGEONMIND, quest system, setup commissioning, portal disambiguation. |
| `zenos_ai/docs/scripts/zen_dojotools_systemtools_readme.md` | `ha_reload_all` — "LAST RESORT ONLY" corrected from "Default choice." |

---

## Upgrade Notes

**FC v6 callers:** Run a grep for `tojson` on any `value:` fields passed to FileCabinet. Remove them.

**Wiki callers:** Any KFC or automation calling `script.zen_dojotools_wikijs` directly must be updated to `script.zen_dojotools_filecabinet` with `stack=wiki`.

**kitchen_sync plugin:** If installed as `plugins/kitchen_sync/kitchen_sync.yaml`, move it to `plugins/mealie/kitchen_sync.yaml` and remove the old directory.

**Cortex v43:** Load via `zen_admintools_prompt_loader: cortex_version: latest`. Previous v42 slot remains available at `cortex_version: v42`.

**LiveDrawer setup:** No migration needed. LiveDrawers are new drawer types — existing drawers are unaffected. Compose new LiveDrawers via Scribe with the KF4 schema fields (`tool`, `seed`, `area_seed`) in the drawer value.

---

# 2026.7.1 Patch

**Base:** 2026.7.0 'Neo'

## Summary

A pull from the live reference install (H:\) covering the KF5 self-registration pattern, a Firefly III codex tier (finance domain modules that plug into `zen_dojotools_finance` without living in its main file), a Grocy backport, a new Lens Bus provider, and a batch of dojotools fixes. Every file's docs were read against the actual code — not diffed against H's copy of the same doc, which can be equally stale — before being called done.

## KF5 — Self-Registering Tools

Tools can now declare their own KFC dojo-drawer contract via `mode=kfc_manifest` and have `zen_dojotools_manifest mode=bootstrap_kfc` discover and mount them automatically — no Scribe authoring step required for tools that already exist as scripts. See [Building a KFC — KF5](../kung_fu/building_a_kfc.md#kf5-self-registering-tools).

- **`dojotools_manifest.yaml`** (v6.2.0) — `bootstrap_kfc`, `health_refresh` (cached per-tool compliance scan), `domains` (tool/domain/peers graph), `audit_help`. `bootstrap_kfc` reads `component_summary` from each tool's response and returns it in its own result, but does not persist it into the cabinet drawer — matches Scribe's KF4 `trim_description` space-saving convention; the ninja summarizer resolves an empty drawer field from the HA label's description at read time.
- **`dojotools_filecabinet.yaml`** — mode-scoped `tool:mode` callout whitelist entries alongside legacy plain/`tool:*` forms.
- **`dojotools_admintools.yaml`** (v5.3.0) — `prompt_loader mode=whitelist whitelist_type=fc` add path rejects bare `*`/empty-suffix `tool:` entries. Also (pulled as a dependency, not originally scoped): `cabinetadmin mode=expand_drawer` (atomic single-drawer migration to an expansion cabinet, with rollback) and a `repair_volumeinfo` fix for the missing-header case.
- **5 dojotools files gained `kfc_manifest`:** `room_manager` (v5.4.1, plus tax/real-estate room fields feeding the Firefly depreciation codex), `alertmanager` (write-verification fix on `alert_clear`), `camera` (pure refactor, no behavior change), `security_manager` (4 components: rollup + perimeter/panel/cameras sub-components), `systemtools` (v5.2.0, plus `home_status`/`home_mode`/`quiet_hours`/`work_hours`/`scheduler_anchors`/`guest_mode` — see [components/systemtools.md](../components/systemtools.md)).

## Firefly III — Codex Tier

`zen_dojotools_finance` (v2.3.0) gains a **codex tier**: sibling YAML files that register into `codex_registry` and dispatch through the main tool via new `index`/`inspect`/`bootstrap_codices`/`report` cases, without living inside `firefly_iii.yaml` itself. See [Firefly III — Codex Tier](../plugins/firefly_iii.md#codex-tier).

- **`zen_codex_finance_depreciation`** (v1.0.0) — household asset depreciation: SL/DDB schedules, Firefly posting, business-use % splits, disposal, warranty tracking. Zero cabinet footprint for asset data.
- **`zen_codex_finance_cogs`** (v1.2.0) — auto-posts COGS to Firefly when Grocy stock tagged `cogs_tracked` is consumed.
- Also: `transaction_get`, `budget_summary`, `finance_rollup`, `category_create` cases. REST dispatcher renamed `zen_sutra_firefly` → `zen_root_firefly`.

## Grocy (v5.3.1)

- **Fixed:** `stock_entry_update` called with only `entry_id` (no product context) silently sent an empty PUT payload — now fetches the entry directly and derives the true `stock_id` from it.
- **New:** Perishable Storage Coaching (`best_before_days`, `stock_audit_perishable`), COGS Coaching hooks, a Grocy-side battery-stock management suite (distinct from the Lens Bus battery provider below), and depreciable-asset/tax userfields feeding the Firefly depreciation codex.
- **New file:** `sutra_logistics.yaml` — KFC manifest for `logistics_intake`/`logistics_volatile`. Battery KFC lives in `zen_stack_battery` instead, not duplicated here.

## Battery Notes — New Lens Bus Provider

`zen_stack_battery` (v1.2.1, HALMark PASS) — Lens Bus provider over the HACS **Battery Notes** integration (`andrew-codechimp/ha-battery-notes`, a required third-party dependency — this is the first ZenOS stack provider sourced from a HACS integration rather than a self-hosted backend). See [Battery Notes Plugin](../plugins/battery_notes.md).

## Postman — `direct_dispatch`

New mode: an authority-stack bypass, straight to `notify.*` targets. Requires `override: true` or the call is blocked and logged. This is the migration path for any caller still using the retired `zen_dojotools_notification_router` script name. The dispatcher's two legacy compat arms for that script name were removed (dead code — the script no longer exists); an old-style caller now gets the dispatcher's normal `unknown_tool` structured fault instead.

## Other Fixes

- **ZenZork** (v1.7.0) — `llm_narration` toggle: live LLM-generated narration via `ai_task.generate_data`, falling back to template narration if the pipe is gated off or the response is too short.
- **Labels** — `mode` is now the primary selector, matching the project-wide standardization on `mode:` across all dojotools calls. `action_type` remains a deprecated, fully-supported alias.
- **AutoVac** (v5.3.0) — new day-of-week priority scheduler (`mode=schedule`, `schedule_blocks`) and Lens Bus provider modes (`stacks_by_anchor`/`register`/`unregister`/`health`/`inspect`) — reachable only via direct script call, not yet wired into the dispatcher.

## HALMark Hardening Beyond Straight Copies

Several FG-class guard gaps were found and fixed during this pull (not present in the pulled-from copies as-is):

| File | Fix |
|---|---|
| `dojotools_autovac.yaml`, `battery_notes.yaml`, `grocy.yaml` (`battery_overdue`) | FG-41 — `as_datetime()` calls guarded against silent-`None` on unparseable timestamps |
| `grocy.yaml` (`stock_audit_perishable`) | Numeric-cast guard on `grocy_child_location_ids` child-walk — malformed entries now skip that child instead of raising |
| `dojotools_alertmanager.yaml` | `tool_manifest` reported `1.0.0` while `kfc_manifest` reported `5.1.0` for the same tool — aligned |
| `dojotools_systemtools.yaml` | Same stale-version bug, caught by Nyx after the first fix shipped — `tool_manifest`/`kfc_manifest` said `5.1.1`, header said `5.2.0` — aligned. Also: anchor-conflict string comparison lacked an unknown/unavailable guard in **two** places (`home_status` display and the `scheduler_anchors` setter itself) |
| `dojotools_security_manager.yaml` | Missing `version` field on the parent `security_manager` component (sub-components had it) |
| `zen_codex_finance_depreciation.yaml` | Unguarded date-slice on a null `placed_in_service_date`; bare `float()` casts in business-use split math; `_setup_canonical_names` was missing the 8 new tax/allocation userfields Grocy now writes |

All fixes verified live by Nyx against the actual instance (config check, reload, and targeted test cases against real data) before being called done.
