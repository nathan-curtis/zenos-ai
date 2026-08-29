# Release Notes — 2026.8.0 'Chef'

**Status:** Released
**Branch:** `main`
**Base:** 2026.7.1 (patch on 2026.7.0 'Neo')

---

*Yes, chef.*

---

## Summary

Every kitchen runs on the same discipline, whether it's feeding four or forty: mise en place. Everything in its place, prepped and known, before service starts. You don't discover you're out of stock mid-ticket. You don't guess whether a substitution works — you know your pantry, you know your recipes, and you know exactly what's already in motion before you fire the next course.

Neo gave the system a graph. Ready Player Two and the 7.1 patch gave tools a way to introduce themselves. Chef is the release where the kitchen actually learns to run a pass.

Taskmaster stopped being a label pattern read by the summarizer and became a real expediter — a script that knows which station a ticket belongs on (your own to-do list, a tracked service ticket, a Grocy chore, a CRM follow-up) and calls it there without pretending every station works the same way. It doesn't merge the menus. It routes the order.

The kitchen itself — Mealie — got the batch this release is named for. Not new plumbing this time. New *judgment*. `kitchen_brief` costs out the whole week's meal plan against real spend in one deduped pass instead of re-querying the same onion four times because it showed up in four recipes. `recipe_fulfillment` and `recipe_consume` check the walk-in before promising a dish is makeable, and deplete it honestly afterward using real unit conversions, not guessed math. `waste_log` books a spoiled item as waste in Grocy's own ledger — no shadow bookkeeping, no second source of truth, because cabinet space is finite and lying to yourself about food cost helps no one. `event_menu_create` scales a whole dinner party to a headcount in one call and flags every allergy at the table, because Room Manager now knows who's actually staying in which room tonight, guest or family, and Twenty's CRM knows their prefs before they ever sit down.

That's the through-line of this release: **know before you promise.** The SP1 identity gate — `resolve_caller_identity` — is the walk-in cooler lock. Every gated action asks it who's actually asking before it hands over the keys, instead of every tool improvising its own front door. Portainer container control is the first real thing standing behind that door: list and inspect freely, but restarting or stopping a container needs a badge, and pulling one out needs a badge *and* a manager's nod, live, with a real timeout if nobody answers. Today the badge check is honest theater — sim_mode, fail-closed by policy, flippable with one switch — but the chokepoint is real, and everything gated through it upgrades for free the day the real lock goes in.

None of this replaces judgment. `recipe_cost` says plainly when it's a lower bound because half your pantry has no price history. Waste logging doesn't try to reconstruct *why* something spoiled, just that it did. Medications still never outrank a calendar alarm in Taskmaster's scoring, no matter how many hard-coded incentives a lesser system might have to make everything feel urgent. A kitchen that lies to make service feel smoother isn't running well. It's just hiding the ticket times.

Yes, chef.

---

## Taskmaster — The Expediter

**New file:** `dojotools/dojotools_taskmaster.yaml`, v0.3.0. Never previously committed to this repo's history.

Progressive-enhancement task orchestration. Every household gets task tracking on O365 To Do alone, zero config. Configure Radar (Zammad) and task creation upgrades to a real tracked ticket with lifecycle/triage. Configure Inventory (Grocy) and chore-shaped recurring tasks upgrade to Grocy chores tied to product/location/stock. Taskmaster never merges these systems' data models into one shape — it picks the best backend at creation time and fans reads out across whichever are configured. Routing priority: explicit contact/company → chore-shaped → Radar-shaped → Grocy task → base To Do.

- **`briefing`** — the scheduled seed, KF5 self-registered, fires on `quarter_hour` + occupancy-change triggers. Conductor items first, then everything else scored deterministically (medication/PRN/chore/date-rule logic). Returns a `context` block of *real* situational signals — `home_mode`, `quiet_hours_active`, `guest_present` (now Room-Manager-aware), `upcoming_appointments`, `weather.outdoor_ok`, and (new) `prep_schedule` — Kitchen's own `prep_brief` for today, wired in so cooking lead-time is proactive on Taskmaster's existing trigger, not a separate pull.
- **`almanac`** — on-demand "what does today look like": meal plan, appointments, and a collision flag between them. Informational only, never blocking.
- **`stale_review`** — filters `briefing`'s scoring to urgency ≥ 2, suppressing anything already triaged to the backlog rank regardless of raw priority.
- **`facilities_brief`** — the building-super view: physical-plant kata components (energy, water, hot tub, garage freezer, dishwasher, laundry) read live, no re-query, urgency ≥ 2 only.
- **Hard cap, no override path:** medications never rank above urgency 3. A missed dose is a reminder, not an emergency, and the tool enforces that structurally — not as a suggestion to the model reading it.

See [Taskmaster](../kung_fu/taskmaster.md).

## SP1 Identity Gate — The Walk-In Lock

**`dojotools_identity.yaml`** (v5.4.0) gains `mode=resolve_caller_identity` — the platform-wide chokepoint for "who is actually asking, and are they allowed to do this." Every gated dojotool calls this instead of hardcoding its own default-agent fallback or calling `persona_editor` directly on its own borrowed identity.

- **OS-level policy gate**, not per-tool: `integrations_config.identity.sim_mode_allowed` (household cabinet, **fails closed by default**). Simulated identity is rejected unless the household has explicitly opted in.
- **Cert delegation**: pass `required_cert`/`required_cert_level` and the mode runs the cert check itself against the resolved identity, returning `authorized: true/false` — callers stop checking their own borrowed identity against their own cert list.
- **Continuity switch**: since the new default is fail-closed, every existing gated action would go dark the instant this shipped. `flynn.yaml` gained `input_boolean.zenos_sim_mode_override` (**defaults on**) plus a boot-time stamping automation, so upgrading this release changes nothing about current behavior until you deliberately flip the switch off.
- Backed by `zen_root_authentik` — an honest stub. No real OIDC network call exists yet. `sim_mode: true` is the one flip point in the code for when it does.

## Portainer Container Control — First Through the Door

**New:** `dojotools_infra.yaml` gains a conditional codex (inert until configured) on top of its pre-existing, byte-for-byte-unchanged status/monitoring modes.

- `containers_list` / `container_get` — read, no gate.
- `container_restart` / `container_start` — needs an `infra_container_control` cert (default level 2+), resolved via the new identity gate.
- `container_stop` / `container_remove` — cert level 3+ **plus** a live household-admin acknowledgment round-trip through AlertManager (~150s timeout, fails closed if nobody answers).

Allow-list is deny-by-default, admin-managed via a separate Developer-Tools-only tool, never MCP-exposed. See [Infra / Portainer](../components/infra.md).

## Kitchen — Mise en Place

**`plugins/mealie/mealie.yaml`** jumps 5.8.0 → 5.22.0. Thirteen versions of real work, most of it building the fulfillment and costing layer that never existed before this release, capped by the exec-chef batch it's named for.

**Fulfillment & Costing** — "can I actually make this, and what does it cost":
- `recipe_fulfillment` / `recipe_consume` — check real stock before promising a dish, deplete it honestly after, using Grocy's actual unit-conversion table instead of guessed math.
- `recipe_cost` / `kitchen_brief` — per-recipe and per-week cost rollups against real price history. States plainly when a total is a lower bound rather than pretending completeness it doesn't have.
- `recipe_plan`, `cookbook_fulfillment`, `stock_expiry_suggestions`, `mealplan_suggest` — combine fulfillment checking with meal planning; `mealplan_suggest` never auto-schedules, it only suggests.
- `recipes_scale`, `recipes_parse_ingredients`, `recipes_create_from_url` — closes real gaps in the recipe-import/scaling path, including a fix for a Mealie-side bug that could silently delete a recipe on a failed update (see Fixes).

**Executive Chef (new this release):**
- `waste_log` / `waste_summary` — logs discards via Grocy's own `spoiled=true` stock field. Deliberately no separate ledger — cabinet space is finite, and Grocy already tracks this natively.
- `event_menu_create` — scales a multi-dish menu to a guest count in one call, flags allergens per dish via Room Manager's new occupant-prefs lookup.
- `prep_schedule_set` / `prep_brief` — lead-time tagging on meal plan entries, now feeding Taskmaster's briefing directly.
- `leftovers_to_stock` — books cooked leftovers back into Grocy as real, idempotent stock.

Meal-time windows (`prep_brief`, allergen-flagging context) now derive from the household's existing `scheduler_anchors` instead of a second hardcoded time set that could silently drift from the first. See [Kitchen / Mealie](../plugins/mealie.md).

## Twenty CRM — Who's Actually At the Table

**`plugins/twenty/twenty.yaml`** — v1.7.0 → v1.10.0. Structured `stay`/`appointment`/`serviceRequest` objects replace calendar-only stay tracking, with full lifecycle dispatch (turnover chore on release, vendor history on appointment completion, shape-first Taskmaster routing on service requests) and a guest-prefs/allergen bridge that Kitchen's `mealplan_suggest` and `event_menu_create` both read from — the same underlying occupant lookup Room Manager now exposes.

## Room Manager — Who's Actually Home

**`dojotools_room_manager.yaml`** gains `room_occupant_prefs` — a guest-stay lookup that falls back to a room's static household occupant when nobody's staying there, so a family dinner gets the same allergen flagging a guest stay does. This is the shared lookup both Kitchen and Twenty now build on.

## Fixes

| File | Fix |
|---|---|
| `dojotools_core.yaml` + `dojotools_scheduler.yaml` | Real bug, not cosmetic: `trigger.event` was checked with `is mapping`, always false for a real HA Event object — every scoped `component:` resummarize was silently falling through to a full-fleet resummarize instead. GC now tracks exactly which kata keys it evicted and fires a targeted event per key. |
| `dojotools_library.yaml` | `locations_find` replaces a paginated `locations_list` call that silently missed any location created past roughly the first 250 — including the tool's own configured default shelf on a large household. `title` now renames the underlying Grocy product directly instead of being silently ignored. |
| `dojotools_manifest.yaml` | `bootstrap_stacks` no longer counts the manifest script's own entity as a discoverable stack. Nested `providers` registry lookups now merge via `combine()` instead of dropping parent-level keys. |
| `dojotools_systemtools.yaml` | `event_emitter`'s `emit` mode now maps `severity=warn` to the `level` core `system_log.write` actually accepts (`warning`) instead of passing `warn` straight through. |
| `plugins/zammad/zammad.yaml` | Ticket priority input now normalizes numeric/lowercase values to Zammad's canonical strings, mirroring an existing fix already applied to `ticket_update`. |
| `plugins/paperless_ngx/paperless_ngx.yaml` | The default-fallthrough handler was missing `stack: paperless` in its forward call — every real call was silently landing on "Unknown stack/mode" regardless of configuration. |
| `plugins/mealie/mealie.yaml` (v5.10.1) | A hand-built full-body `PUT` on `recipes_update` could trigger a Mealie-side bug where a validation error still left the recipe deleted. Rebuilt as a GET→merge→PUT path that can no longer take the unsafe route — a real recipe was lost this way before the fix. |
| `plugins/grocy/grocy.yaml` | Two `mode: single` → `mode: queued` fixes on the internal inventory dispatch scripts — overlapping callers were being silently dropped, not queued, under any real concurrent load. |

## Hardening — The Reset Test

Every prior beta pass restarted into an already-provisioned system. Nobody had done the thing every brand-new install actually does: reset labels *and* cabinets, restart, and watch Flynn try to recover completely unassisted. That test found nine real, silent-deadlock-class bugs in one session — all in the boot-gate sequence, all now fixed and verified live through two consecutive full reset-and-rebuild cycles plus a mixed-state discrimination test (one cabinet hammered alone against eleven healthy ones — only the hammered one got touched).

- **`unassigned_label_ids()` — the actual deadlock.** A tuple-indexing bug (`selectattr('1', 'eq', 0)` doesn't do what it looks like it does in Jinja) meant this macro always returned empty, so `zen_label_health` could never report `warn`, so Flynn's own label-assignment gate could never fire. A fresh or reset install sat with labels registered but never tagged onto any entity — forever, with no error anywhere.
- **Gate 2's virgin-vs-broken check.** `zen_cabinet_health`'s rolled-up state can't tell "not bootstrapped yet" apart from "genuinely broken" on its own; Flynn now checks every required slot individually, including the `absent` state (no entity at all) that a fresh system passes through and previously read as unsafe.
- **Gate 3 wasn't reachable with summarizers off.** Summarizers ship off by default, and the monastery sensor's own priority check meant it could never report the state Gate 3 was waiting for — so a fresh deploy on the shipped default configuration silently never provisioned a household or AI identity at all. Fixed, then fixed twice more: the gate's own post-bootstrap `stop` was left unconditional (blocking Gate 3.5 from ever running), and `flynn_system_ready` had the identical gap independently.
- **"Robot Donut."** `zen_cabinet_health` had no trigger of its own and several `cabinetadmin` write paths never woke it up even once triggered — cabinets could sit fully initialized and mounted for eight-plus minutes while the sensor still reported them broken. Fixed with a deliberately event-only double-refresh (immediate + 5s delayed) — never a live state read, which is the same discipline behind the original cabinet boot-wipe incident this bug is named after.
- **Flynn couldn't see his own notifications.** `persistent_notification` entities don't reliably reflect into the template-readable state machine on this HA version — confirmed empirically, not by inspection. Notification state now lives in a household-cabinet drawer, written alongside the real notification call rather than read back from it.
- **`_household_name` had never once persisted, on any install, ever.** A reserved FileCabinet key needs `force_action: true` unconditionally; the write used `false`. The comment describing a protection that "only writes if empty" was describing something that never happened — the write was silently rejected every time. New durable `flynn_bootstrap_state` flag (only stamped after a fresh read-back confirms the identity actually landed) closes the related gap where a system that reached `warn` before ever completing first bootstrap had no way back in.
- **Two false-negative diagnostics.** The Spook-installed check trusted a response body's shape without checking HTTP status first, so an expired `ha_bearer` token read as "Spook not installed" even when it plainly was. A malformed `ai_task` entity (missing its domain prefix) passed validation silently and only surfaced as a crash much later, downstream, when something actually tried to call it.
- **Capsule severity was miscalibrated.** Every real install checked this session — including this project's own production instance — runs on the `legacy` essence schema, and it's fully functional. It was reporting `error`. Reserved for a genuinely missing essence now; `legacy` reads `warn`. `zen_health_report` also gained real capsule visibility and a restructured dependency cascade (`flynn_ready.dependencies`) that shows *why* a problem is happening instead of five same-level fields that read as unrelated.

None of this changes behavior on an already-healthy, already-provisioned system — every fix above is either a diagnostic correction or closes a path that only mattered on first boot or full recovery. See [Developer Taxonomy and Component Standards](../architecture/21_Developer_Taxonomy_and_Component_Standards.md) for the boot-orchestration and health-sensor design rules this pass established, so the next one isn't tribal knowledge either.

## Also This Release

- **Grocy** (v5.6.0) — per-item bulk pricing, `bulk_stock_reconcile_recent`, `dry_run` support on stock writes, a native `spoiled` field on `stock_consume`, and a read-only `waste_history` mode backing Kitchen's waste reporting.
- **Print** — new, explicitly experimental (`0.1.0-experimental`) IPP-based printer control via the native HA IPP integration and the Rudd-O HACS bridge. Not yet proven end to end; see [Print](../scripts/zen_dojotools_print_readme.md) for the honest limitations list.
- **Shared libs** — `zenos_health.jinja` now registers 4 previously-invisible cabinet health-check types (the default household/family/user/AI-user history cabinets).

---

## Upgrade Notes

**Identity gate is opt-out, not opt-in, on upgrade.** `input_boolean.zenos_sim_mode_override` ships defaulted **on**, so nothing gated through `resolve_caller_identity` changes behavior until you deliberately flip it off. Flip it off only once you understand what it's currently holding open — today, that's everything, since real OIDC isn't live yet.

**Portainer container control requires explicit setup.** The allow-list is deny-by-default. Nothing is controllable until an admin seeds it via the Developer-Tools-only ACL tool. `containers_list`/`container_get` work with zero config; everything else stays inert until configured.

**Taskmaster is a different tool than any prior version.** If you have custom automations wired against the old KF4 label-pattern conductor design, they are not compatible with this version's mode-based dispatch. This is a clean break, not a migration path — see [Taskmaster](../kung_fu/taskmaster.md) for the current shape.

**Kitchen callers relying on `recipes_update`'s old blind-passthrough behavior** should re-check any custom automation that built a full-body PUT payload by hand — the safe GET→merge→PUT path is now the only path, which is strictly safer but means a payload that omitted fields previously relied on the old (unsafe) full replacement.

---

## 2026.8.1 — Patch

**Status:** Released
**Branch:** `feat/2026.8.1` (off `main`/2026.8.0)

A small, deliberately narrow patch for households not yet ready to move onto Steel Magnolia's new identity-gate/cert-scope security architecture. Every fix below is a genuine bug that predates Steel Magnolia entirely — nothing here depends on, or drags in, any of that new machinery. If you're already running (or planning to run) 2026.9.0 'Steel Magnolia', this patch has nothing for you that isn't already there; it exists purely as a safe, known-good parking spot on the Chef line.

| File | Fix |
|---|---|
| `custom_templates/zenos_ai/zen_os_1.jinja`, `zenos_manifest.jinja`, `dojotools_utilities.yaml` | New canonical `os_version()` macro replaces a stale, un-refreshed cabinet-based version read across 4 call sites. |
| `dojotools_index.yaml` (ZQ-1) | `filter_json` keys that don't exist (a stray `domains` instead of `domain`) used to vanish silently, producing a confidently-empty result indistinguishable from a real zero-hit query. State-class-aware history stats — requesting stats against a `total`/`total_increasing` energy sensor used to silently return empty buckets. |
| `plugins/mealie/mealie.yaml` | `recipe_consume`/`recipes_update` hardening — real 422s on food-object hydration, traced back through a real household-reported bug (Radar #289). |
| `plugins/twenty/twenty.yaml` | `stays_list`'s `ha_area:`/`crm_link:` tag parsing rebuilt to handle a stay carrying multiple `ha_area:` tags (multi-room booking on one calendar event) correctly — the old raw-regex filter could never match these. |
| `plugins/grocy/grocy.yaml` | The inventory lens's `by_anchor` path self-called its own wrapper script, which HA correctly blocks as disallowed recursion when reached via a call chain that loops back through itself (Room Manager → Lens Dispatch → Media Manager → Inventory → Inventory) — fixed to call the underlying primitive directly. Separately, the OpenAPI REST sensor has always pointed at a path Grocy has never served (`/openapi.json`, 404) instead of the real spec path, causing an hourly failure regardless of configuration — fixed, plus a longer timeout since the real spec generates slower than HA's default. |
| `dojotools_manifest.yaml` | The same self-call recursion bug as the Grocy fix above, in `mode=all`'s force-refresh republish path — the sibling `bootstrap_stacks` instance of this bug was already fixed on `main`; this was the one remaining site using the same broken pattern. |
| `dojotools_autovac.yaml` | `vacuum.clean_area` was called with the wrong service field (`areas` instead of the real `cleaning_area_id`) at all 4 call sites — confirmed against HA core's own service schema, not documentation summaries. Battery was read only via the vacuum entity's deprecated `battery_level` attribute, defaulting to 0% on missing data (which can falsely trip the low-battery gate) with no fallback for integrations that expose battery on their own sensor — new `autovac_battery` label + fallback chain, defaults to "unknown" (100%) rather than "empty" on missing data. `mode=consumables action=provision` computed the full parts/wear catalog and returned it in the response, but never actually wrote it to the household cabinet — `check_wear` always read `not_provisioned` regardless of whether Grocy itself had real provisioned products. |

Two internal self-reported tool version fields had also drifted behind their own file's header changelog and never got corrected along the way — `zen_dojotools_inventory` (`grocy.yaml`) and `zen_dojotools_kitchen` (`mealie.yaml`, doc page bumped to match) both now report their real current version.

**Known, deliberately not fixed here:** AutoVac's Roborock wear-sensor discovery was updated upstream to match Roborock's real sensor names (`main_brush_time_left` etc.), but the preset files' `wear_sensor_key` fields were never updated to match — this looks incomplete even upstream, so it was not ported here to avoid backporting a fix that may not actually work. Flagged upstream for a real fix; watch for a future 8.1.x patch if this needs to land here too.

No new features, no schema changes, no behavior changes to anything not listed above.

---

*ZenOS-AI 2026.8.0 'Chef' — service.*
