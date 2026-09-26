# 25. Appendices

## A. Glossary

| Term | Meaning |
|---|---|
| Abbot | The scheduling and dispatch function: the Scheduler and the Dispatcher together (Chapter 21) |
| AdminTool | A privileged tool for setup, policy, repair, or recovery. Not agent-exposed (Chapter 12) |
| Anchor | The subject of a Lens Bus query: a label, person, area, or zone (Chapter 12) |
| Bundle | A named set of certifications granted together (Chapter 18) |
| Cabinet | A structured store backed by a Home Assistant entity's attributes, holding drawers (Chapter 7) |
| CabCeption | A cabinet mounted inside another cabinet (Chapter 7) |
| Cert, certification | A grant of a class of action at a level, with optional scope and constraints (Chapter 18) |
| Codex | Executable domain expertise or security-scoped action policy (Chapter 12) |
| DojoTool | Friday's supported, agent-callable capability (Chapter 12) |
| Drawer | One named value inside a cabinet (Chapter 7) |
| Envelope | The standard tool response: `{status, mode, tool, result, system_message, caller_token}` (Chapter 11) |
| Flynn | The boot orchestrator and the fallback persona the agent gets when boot or identity fails (Chapters 16, 22) |
| Highlander resolver | A sensor that resolves exactly one cabinet of a type, so code never searches labels at runtime (Chapter 7) |
| Kata | A summary in the fixed component schema (Chapter 14) |
| KFC, Kung Fu Component | A declared summarization contract: what a component watches and how it is summarized (Chapter 13) |
| Lens Bus | The anchor-query layer that merges evidence from registered providers (Chapter 12) |
| Live acknowledgement | A human yes or no, asked for one specific action at the moment it is attempted (Chapter 19) |
| LiveDrawer, VirtualDrawer | Drawers whose value is computed or mounted from elsewhere rather than stored (Chapter 7) |
| Monastery | The summarization pipeline: Ninja, Monk, Trapper Keeper, SuperSummary (Chapter 13) |
| Ninja | The per-component summarizer run |
| Monk | The model call inside a summarizer run, through `ai_task` |
| Ontology | The shared vocabulary (labels, contracts, declarations) that lets an agent read the graph (Part III) |
| Portal | A connection between two rooms, with light and sound transmission values (Chapter 8) |
| REFLEX | The layer that turns a room's state into scenes and effects (Chapters 15, 24) |
| Root | The lowest reusable backend transport. Never exposed (Chapter 12) |
| Scope | Per-target `allow` or `deny` entries on a certification (Chapter 18) |
| SESE | Single entry, single exit: a tool builds one response and returns it once (Chapter 11) |
| Stack | A Lens Bus knowledge provider (Chapter 12) |
| Stripe | A component's position in dependency order. Never a privilege level (Chapter 23) |
| SuperSummary | The whole-house summarizer that writes `zen_summary` (Chapter 13) |
| Sutra | An internal backend operations adapter. Never exposed (Chapter 12) |
| Trapper Keeper | The step that writes a validated Kata to its drawer (Chapter 13) |
| Twin | The model of the house an agent builds by reading the graph through the ontology (Part IV) |
| Wasp hold | Holding an enclosed room as occupied while its doors are shut and presence is live (Chapter 24) |

## B. Event kinds

ZenOS signals over one Home Assistant event, `zen_event`, distinguished by `kind`. These are the kinds the book refers to, grouped by what emits them. The code emits more, mostly per-operation audit kinds from registry and device tools.

| Group | Kinds |
|---|---|
| Dispatch | `dojotool_call`, `dojotool_return`, `dojotool_dispatch_error` |
| Monastery | `summary_force`, `ninja_force`, `supersummary_force`, `gc_force`, `summarizer_start`, `summarizer_skip`, `summarizer_run_blocked`, `supersummary_run_blocked`, `summarizer_size_exceeded`, `ninja_failure`, `ninja_context_overflow`, `monk_failure`, `kata_emit`, `kata_gc_complete` |
| Escalation | `emission_suppressed`, `action_emission_blocked` |
| Room state | `room_state_changed`, `room_control_request`, `reflex_dry_run`, `watchdog_kill` |
| Graph upkeep | `label_mutation`, `entity_area_mutation`, `identity_manifest_rebuild`, `deferred_script_reload`, `deferred_reload_all` |
| Cabinets | `cabinet_mounted`, `cabinet_dismounted`, `cabinet_boot_touch`, `cabinet_vi_degraded`, `cabinet_vi_repaired` |
| Alerts and comms | `alert_fire`, `alert_clear`, `alert_response`, `alert_manager_error`, `postman_dispatched`, `postman_response`, `postman_ack`, `postman_unknown_channel`, `priority_inject_write`, `priority_inject_clear` |
| Identity | `principal_changed`, `household_member_joined`, `household_member_left`, `family_member_joined`, `family_member_left`, `partner_linked`, `partner_unlinked`, `agent_auth_flagged`, `essence_edited`, `essence_signed`, `essence_backed_up`, `essence_restored` |
| Actuation audit | `lock_actuation`, `cover_actuation`, `cover_scene_applied`, `panel_armed`, `panel_disarmed`, `container_action_denied`, `infra_escalation_attempt` |
| Scoped acknowledgement skips | `lock_ack_skipped_scoped_override`, `cover_ack_skipped_scoped_override`, `disarm_ack_skipped_scoped_override`, `container_ack_skipped_scoped_override` |

A new kind should fit an existing group. Event names themselves are never templated: Home Assistant ignores a templated event name without an error.

## C. Certification catalog

This catalog is a snapshot. The live catalog is built from each tool's `certs_required` declaration by `zen_dojotools_manifest mode=cert_audit` (Chapter 23), and that is the authority. Levels shown are the highest level the declaring tool uses, or the level each listed mode requires.

| Cert | Declared by | Covers |
|---|---|---|
| `alert_policy_edit` | alertmanager | `set_policy` (L1) |
| `autovac_operate` | autovac | Vacuum dispatch and per-room queue state |
| `autovac_setup` | autovac | Room definitions and system settings |
| `cabinet_lifecycle_control` | admintools, provisioner | Cabinet restore, repair, reset, init, mount, schema, expand; provision and deprovision (L1) |
| `calendar_control` | calendar | `delete` (L2) |
| `camera_alert_policy_edit` | camera | `set_alert_policy` (L1) |
| `climate_control` | utilities (climate) | Setpoint, mode, and fan speed |
| `cover_control` | covers | Cover position and scenes. Opening a barrier-class cover needs a live acknowledgement every call |
| `display_control` | display | Casting a view to a house display (max L1) |
| `finance_account_control` | Firefly III plugin | `account_delete` (L2) |
| `helper_boolean_edit`, `helper_datetime_edit`, `helper_number_edit`, `helper_select_edit`, `helper_text_edit`, `helper_timer_control`, `helper_zones_edit` | utilities | Writes to each helper domain |
| `household_spatial_config_edit` | zenzork | Spatial configuration writes (north calibration and related) |
| `infra_container_control` | infra | Container start, restart, stop, remove |
| `infra_zwave_control` | infra | Enabling a Z-Wave device's diagnostic entities (max L2) |
| `library_catalog_edit` | library | Physical item catalog writes |
| `library_stacks_edit` | library | Document store writes |
| `lighting_control` | lights | Brightness, color, and scene actuation |
| `lock_control` | locks | Lock and unlock. Unlocking an `ext_lock` needs a live acknowledgement every call |
| `media_prefs_edit` | media manager | Room and user AV configuration |
| `pii_disclosure_control` | calendar, office, taskmaster, todo | Creating or sending content that can disclose what the agent has read (L1); mail whitelist changes (L2) |
| `plant_auto_shutoff_config_edit` | plant | Enabling leak auto-shutoff (L1) |
| `registry_lifecycle_control` | ectoplasm, labels | Reversible registry lifecycle changes; label create, update, delete, and area assignment (L1) |
| `registry_purge` | ectoplasm | Irreversible cleanup of orphaned registry entries |
| `room_behavior_control` | room manager | Room and house behavior toggles, and override writes other than setting `Paused` |
| `room_control_override` | room manager | Clearing a room out of `Paused`. Live acknowledgement every call unless scoped to that area |
| `room_topology_edit` | room manager, ectoplasm | Structural edits to the house model |
| `scene_handling` | lights | Scene creation and staging |
| `scribe_kfc_publish` | scribe | Writing a published KFC into live procedural memory |
| `security_control` | security manager | Alarm arm and disarm, alert policy |
| `spa_control` | spa manager | Spa jets, lights, water temperature, cover |
| `teams_control` | office | Setting Teams presence (L1) |
| `todo_control` | todo | `delete` (L2) |
| `water_heater_control` | utilities | Water heater power, temperature, and mode |
| `zenos.comms.dispatch` | postman | Message dispatch and policy. Life-safety dispatches are exempt (max L1) |
| `zenos.identity.profile_write` | profile | Household, user, and family profile writes (max L1) |
| `zenos.media.generate` | image generator | Image generation, which costs money per call (max L1) |
| `zenos.media.playback_control` | media manager | AV actuation |
| `zenos.media.print_control` | print | Print actuation |
| `zenos.system.home_config_write` | systemtools | Durable household configuration: home mode, quiet and work hours, anchors, guest mode (max L1) |
| `zenos.system.reload_restart` | systemtools | Reload and restart, on top of `confirm_action` (max L1) |

Forty-six certificate types in all, counting each helper cert separately.

## D. Labels

Chapter 10 describes the families. The live catalog is computed, not maintained: every tool declares `required_labels` and `optional_labels` in its manifest, and `zen_dojotools_manifest mode=label_audit` reports every declared label, which tools depend on it, and whether it exists. It can create missing definitions behind a confirmation.

| Family | Where described |
|---|---|
| Room labels and `zen_room_state` | Chapters 10, 15 |
| Room Manager signals, holds, and opt-outs | Chapter 24 |
| `scene_<state>`, `reflex_transition_<N>` | Chapter 24 |
| Tool role taxonomies (`zen_lm_*`, `zen_cv_*`, `zen_mm_*`, `zen_plant_*`, `autovac_*`), `primary` | Chapter 10 |
| Security (`security_manager`, `security_camera`, `alarm_panel`, `ext_lock`) | Chapter 10 |
| Cabinet type labels and tool tier labels | Chapters 7, 10 |
| `zen_kfc_provider` | Chapter 12 |
| `zen_agent_disabled` | Chapter 24 |

## E. Not yet built

Every "Not yet built" box in the book, in one place.

| Chapter | Item | Status |
|---|---|---|
| 3 | History cabinet as long-term episodic memory | In active development |
| 3 | Self model layers (drives and values, meta-awareness); trajectory and prediction in SuperSummary | Design direction |
| 7 | FileCabinet certification gate, single exit, envelope | Scope of 2026.11.0 'This Is Spinal Tap' |
| 9 | Tool search: send core tools and discover the rest on demand | Depends on Home Assistant's agent integration |
| 16 | A real `session_token` in the prompt frame | Arrives with session binding |
| 17 | Partner-aware authorization | Design direction |
| 17 | Cryptographic binding of an MCP session to a persona | Design direction. Every call resolves to the default agent |
| 18 | Certification expiry and revocation checks at use time | Design direction |
| 18 | A gate that reads `waives_live_ack` | Recorded by CertAdmin, read by no tool |
| 19 | Admission: the base agent certification | In progress for the 2026.10.0 final |
| 19 | Console-admin path for bundle grants | Planned |
| 19 | Administrative competency certification | Design direction for this release line |
| 20 | Per-principal traversal | Arrives with session binding |
| 20 | Membership edges that gate actions | Design direction |
| 20 | Claims computed from a fold over the label graph | Design direction |
| 21 | Routing inference jobs to different providers | Stated direction |

## F. Reference household

The numbers in this book come from one real ZenOS install, measured on 2026-09-26. They are counts only. Treat them as one data point about scale, not as limits or targets.

| Measure | Value |
|---|---|
| Areas, floors | 36, 4 |
| Rooms registered in room topology, portals | 27, about 53 |
| Labels defined, labels in use, `zen_` labels | 1,069, 784, about 122 |
| Entities carrying at least one label | about 4,969 |
| Label to entity associations | 10,667 (about 2.1 per labeled entity) |
| Entities exposed to the conversation agent | 0 |
| Cabinets, drawers | 13, about 289 |
| Largest cabinet | 116 drawers, about 76% of its declared 131,072 character ceiling |
| Tools scanned (DojoTools, AdminTools, Stacks, Sutras) | 103 |
| Lens Bus providers registered | 17 |
| KFC components | 9 |
| Kata drawers | 49, averaging about 1,000 characters |
| A `zen_summary` record | about 3.2 to 3.5 KB |
| `render_prompt()` output, one sample | 49,868 characters |
| Principals | 2 (1 human, 1 AI persona) |
| Certificate types, held by the default agent, scoped | 46, 19, 1 |

The prompt sample by section: system 22,097, kata 16,566, index 4,578, capsule 2,366, wake 1,626, manifest 1,143, overview 797, header 497, id_manifest 198. The work queue, priority notices, and console line were empty and cost nothing.

Two things stand out. The largest cabinet is the one to watch: each cabinet declares a storage ceiling, and the household cabinet is three quarters of the way to its own. And the prompt is dominated by the system section and the Katas, not by the house: the index that describes more than a thousand labels costs under 5,000 characters, which is what exposing tools instead of entities buys (Chapter 9).

<!-- nav -->
---

[← Room Manager v3 Reference](24_room_manager_v3.md) · [Contents](00_toc.md)
<!-- /nav -->
