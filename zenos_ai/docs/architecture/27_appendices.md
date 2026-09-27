# 27. Appendices

## A. Glossary

| Term | Meaning |
|---|---|
| Abbot | The scheduling and dispatch function: the Scheduler and the Dispatcher together (Chapter 22) |
| AdminTool | A privileged tool for setup, policy, repair, or recovery. Not agent-exposed (Chapter 13) |
| Anchor | The subject of a Lens Bus query: a label, person, area, or zone (Chapter 13) |
| Bundle | A named set of certifications granted together (Chapter 19) |
| Cabinet | A structured store backed by a Home Assistant entity's attributes, holding drawers (Chapter 8) |
| CabCeption | A cabinet mounted inside another cabinet (Chapter 8) |
| Cert, certification | A grant of a class of action at a level, with optional scope and constraints (Chapter 19) |
| Codex | Executable domain expertise or security-scoped action policy (Chapter 13) |
| DojoTool | Friday's supported, agent-callable capability (Chapter 13) |
| Drawer | One named value inside a cabinet (Chapter 8) |
| Envelope | The standard tool response: `{status, mode, tool, result, system_message, caller_token}` (Chapter 12) |
| Flynn | The boot orchestrator and the fallback persona the agent gets when boot or identity fails (Chapters 17, 23) |
| Highlander resolver | A sensor that resolves exactly one cabinet of a type, so code never searches labels at runtime (Chapter 8) |
| Kata | A summary in the fixed component schema (Chapter 15) |
| KFC, Kung Fu Component | A declared summarization contract: what a component watches and how it is summarized (Chapter 14) |
| Lens Bus | The anchor-query layer that merges evidence from registered providers (Chapter 13) |
| Live acknowledgement | A human yes or no, asked for one specific action at the moment it is attempted (Chapter 20) |
| LiveDrawer, VirtualDrawer | Drawers whose value is computed or mounted from elsewhere rather than stored (Chapter 8) |
| Monastery | The summarization pipeline: Ninja, Monk, Trapper Keeper, SuperSummary (Chapter 14) |
| Ninja | The per-component summarizer run |
| Monk | The model call inside a summarizer run, through `ai_task` |
| Ontology | The shared vocabulary (labels, contracts, declarations) that lets an agent read the graph (Part III) |
| Portal | A connection between two rooms, with light and sound transmission values (Chapter 9) |
| REFLEX | The layer that turns a room's state into scenes and effects (Chapters 16, 25) |
| Root | The lowest reusable backend transport. Never exposed (Chapter 13) |
| Scope | Per-target `allow` or `deny` entries on a certification (Chapter 19) |
| SESE | Single entry, single exit: a tool builds one response and returns it once (Chapter 12) |
| Stack | A Lens Bus knowledge provider (Chapter 13) |
| Stripe | A component's position in dependency order. Never a privilege level (Chapter 24) |
| SuperSummary | The whole-house summarizer that writes `zen_summary` (Chapter 14) |
| Sutra | An internal backend operations adapter. Never exposed (Chapter 13) |
| Trapper Keeper | The step that writes a validated Kata to its drawer (Chapter 14) |
| Twin | The model of the house an agent builds by reading the graph through the ontology (Part IV) |
| Wasp hold | Holding an enclosed room as occupied while its doors are shut and presence is live (Chapter 25) |

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

This catalog is a snapshot. The live catalog is built from each tool's `certs_required` declaration by `zen_dojotools_manifest mode=cert_audit` (Chapter 24), and that is the authority. Levels shown are the highest level the declaring tool uses, or the level each listed mode requires.

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

Chapter 11 describes the families. The live catalog is computed, not maintained: every tool declares `required_labels` and `optional_labels` in its manifest, and `zen_dojotools_manifest mode=label_audit` reports every declared label, which tools depend on it, and whether it exists. It can create missing definitions behind a confirmation.

| Family | Where described |
|---|---|
| Room labels and `zen_room_state` | Chapters 11, 16 |
| Room Manager signals, holds, and opt-outs | Chapter 25 |
| `scene_<state>`, `reflex_transition_<N>` | Chapter 25 |
| Tool role taxonomies (`zen_lm_*`, `zen_cv_*`, `zen_mm_*`, `zen_plant_*`, `autovac_*`), `primary` | Chapter 11 |
| Security (`security_manager`, `security_camera`, `alarm_panel`, `ext_lock`) | Chapter 11 |
| Cabinet type labels and tool tier labels | Chapters 8, 11 |
| `zen_kfc_provider` | Chapter 13 |
| `zen_agent_disabled` | Chapter 25 |

## E. Not yet built

Every "Not yet built" box in the book, in one place.

| Chapter | Item | Status |
|---|---|---|
| 4 | History cabinet as long-term episodic memory | In active development |
| 4 | Self model layers (drives and values, meta-awareness); trajectory and prediction in SuperSummary | Design direction |
| 8 | FileCabinet certification gate, single exit, envelope; no bypass traverse checking through mounts; security resolved before a LiveDrawer target runs | Scope of 2026.11.0 'This Is Spinal Tap' |
| 10 | Tool search: send core tools and discover the rest on demand | Depends on Home Assistant's agent integration |
| 14 | Search and vector backends behind the index, seamless to the agent | Under investigation |
| 17 | A real `session_token` in the prompt frame | Arrives with session binding |
| 18 | Partner-aware authorization | Design direction |
| 18 | Cryptographic binding of an MCP session to a persona | Design direction. Every call resolves to the default agent |
| 19 | A registered OID space, so dotted certification names become real object identifiers | Being pursued |
| 19 | Certification expiry and revocation checks at use time | Design direction |
| 19 | Signed certificates issued by a certificate authority | Design direction; the entry is already shaped for it |
| 19 | Secure enclave cabinet for certifications, accessed only through a representative token salted and hashed to the session ID and expiring with it | In progress |
| 19 | A gate that reads `waives_live_ack` | Recorded by CertAdmin, read by no tool |
| 20 | Admission: the base agent certification | In design |
| 20 | Console-admin path for bundle grants | Planned |
| 20 | Administrative competency certification | Design direction for this release line |
| 21 | Certifying a corpus (a cabinet) as an object, with hard links between cabinets enforced through cabinet ACLs | Direction; ACL structure exists, unenforced |
| 21 | Filtering at the source: FileCabinet resolves security before returning any drawer, so each principal reads its own subgraph | In progress |
| 21 | Per-principal traversal | Arrives with session binding |
| 21 | Membership edges that gate actions | Design direction |
| 21 | Claims computed from a fold over the label graph | Design direction |
| 22 | Routing inference jobs to different providers | Stated direction |

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

Two things stand out. The largest cabinet is the one to watch: each cabinet declares a storage ceiling, and the household cabinet is three quarters of the way to its own. And the prompt is dominated by the system section and the Katas, not by the house: the index that describes more than a thousand labels costs under 5,000 characters, which is what exposing tools instead of entities buys (Chapter 10).

## G. Friday's Party index

The posts in the Home Assistant community thread [Friday's Party](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862) that anchor the history in Chapter 3 and the origin of each chapter. The last column is where each one lands in the book, by section or chapter.

| Post | Date | Topic | What it anchors | Book |
|---|---|---|---|---|
| [#1](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/1) | 2025-02-27 | The front door | What agentic means here; tools plus context are the whole game; Friday's origin | 3.1 |
| [#2](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/2) | 2025-03-04 | The Library Grand Index | LLMs need lots of context; exposing labels is night and day | 3.2, Ch. 7, 11 |
| [#3](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/3) | 2025-03-07 | The Grand Library | Tell the AI where something should be; cabinets and volumes as a construct | 3.2, Ch. 13 |
| [#7](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/7) | 2025-03-11 | Bringing it together, part 1 | Don't drop a box at grandma's feet and expect her to know what she has | Ch. 1 |
| [#8](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/8) | 2025-03-11 | Kung Fu | Avoid the Amazon problem; a tool cannot hold all the context; domain bundles loaded as units | 3.3, Ch. 13 |
| [#15](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/15) | 2025-03-13 | Poetry in motion | Say the most with the fewest words; intent and urgency; prompt craft as poetry | 3.4, Ch. 2 |
| [#17](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/17), [#18](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/18) | 2025-03-14, 17 | Local memory | First memory experiments; not RAG; template and storage limits | 3.5 |
| [#25](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/25) | 2025-03-19 | Let's review | Context needs; prompt length limits; teach the model how to fail | 3.4, 3.5 |
| [#34](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/34) | 2025-03-23 | Ninjas | Context sliding drops base intents; summarize the prompt with the model | 3.5, 3.7, Ch. 14 |
| [#42](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/42) | 2025-03-25 | Ninja 2 | Attention is finite; per-component summaries; the model chooses the next hour's components | 3.5, 3.7, Ch. 22 |
| [#49](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/49) | 2025-03-31 | Limits | Measured context load from exposure; why "expose everything" falls apart | 3.5, Ch. 10 |
| [#51](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/51) | 2025-03-31 | Special-purpose pipelines | Local workers take one component each; trigger-fired summaries; interactive prompt clears out | 3.7, Ch. 14 |
| [#53](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/53) | 2025-03-31 | The cost of summaries | Cloud summaries too expensive; offline local summarization; monks, Katas, the archive | 3.7, Ch. 14, 15 |
| [#55](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/55) | 2025-04-02 | Meet Kronk | The Monastery and its curator; specialist local models for queued jobs | 3.7, Ch. 14 |
| [#113](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/113) | 2025-06-11 | Friday's Zen | Kung Fu as callable script modules; the start of DojoTools | 3.8, Ch. 12, 13 |
| [#120](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/120) | 2025-07-03 | Storing elephants in drawers | Context-first memory; pointers instead of copies; inherited context; manifest as card catalog | 3.6, Ch. 8 |
| [#158](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/158) | 2025-08-24 | Files: the workaround | Trigger-based template sensors as cabinet storage | 3.6, Ch. 8 |
| [#163](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/163) | 2025-09-04 | The Manifest | Knowing what is on the shelves; cabinets contain drawers | 3.6, Ch. 8, 12 |
| [#167](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/167) | 2025-09-07 | Grandma's box o' junk | Raw live state as a box of junk; labels and the index to make sense of it | Ch. 1 |
| [#168](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/168) | 2025-09-11 | Why the cabinets exist | Strict boot sequence; directives from a cabinet; load sizes by component | 3.6, Ch. 17 |
| [#181](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/181) | 2025-09-23 | The Kung Fu Loader | Every word matters; stances distill essence instead of dumping state | 3.4, Ch. 17 |
| [#195](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/195) | 2025-10-21 | The Ninja Summarizer | Condensed components with pointers back; she understands the box and knows what is in it | 3.7, Ch. 1 |
| [#200](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/200) | 2025-10-29 | No LLM edits production | System prompt in a read-only cabinet; boots even when blanked; Flynn's origin | 3.10, Ch. 5, 23 |
| [#221](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/221) | 2025-11-13 | Squishing bugs | CoALA primer added to the research folder | 3.12, Ch. 4 |
| [#234](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/234) | 2025-11-19 | Sand dune plinko | Grounding; pegs; truth as the path of least resistance | 3.12, Ch. 2 |
| [#254](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/254) | 2025-11-28 | Onboarding plan | Interview, seed, verify, seal a persona; certificate direction | 3.11, Part V |
| [#261](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/261) | 2025-12-01 | Hyper-huh? | Labels as hyperedges; the flat pile becomes a connected world; changed direction overnight | 3.9, Ch. 7 |
| [#309](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/309) | 2026-02-23 | Self-documentation | Tools built for agent use must identify themselves | 3.8, Ch. 12 |
| [#315](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/315) | 2026-02-27 | Welcome to class | HALMark: a standard for LLM-written Home Assistant code | 3.10 |
| [#320](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/320) | 2026-03-04 | RC1 crossed | Cabinets, labels, FileCabinet, Flynn, the Monastery loop, HyperIndex all running | 3.10 |
| [#326](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/326) | 2026-03-07 | What changed and why | Library commands retired for index plus Dojo-to-Kata; expiring context; packages | 3.6, 3.9 |
| [#392](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/392) | 2026-03-18 | The long walk | Production table stakes: health sensors, kill switches; the agent-assisted build loop | 3.9, 3.10, Ch. 23, Ch. 1 |
| [#423](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/423) | 2026-03-21 | The why | Home Assistant as a state machine layer; retrieval downstream; wake inside the moment; tools inside HA, MCP as the doorway | 3.7, 3.12 |
| [#427](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/427) | 2026-03-21 | The twin | The household cabinet becomes the digital twin | 3.6, Part IV |
| [#443](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/443) | 2026-03-24 | About that release | Transitional states are real; warn is not error; escrow | 3.10, Ch. 23 |
| [#456](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/456) | 2026-03-25 | Stop putting SOPs in your cortex | Label it, write a KFC, store it in the right cabinet, index it | 3.9, Ch. 11, 13 |
| [#565](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/565) | 2026-06-04 | Catch-up | People live in context; the thread's own landmark list | 3.12 |
| [#689](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/689) | 2026-08-10 | The room ladder | Room state as breadcrumbs; overview, room, tool, help, deep data; tool descriptions to 1,024 characters | 3.9, Ch. 16 |
| [#719](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/719) | 2026-08-19 | The security model | The conversation maps onto authorization; allow, default, deny | 3.11, Part V |
| [#788](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/788) | 2026-09-20 | Construct her world | A resolved version of the house, rebuilt every turn; the harness around your priorities; CoALA | 3.13 |
| [#799](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/799) | 2026-09-23 | Prompt pressure | Prefix caching; the zero-entity experiment | 3.9, Ch. 10 |
| [#802](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/802) | 2026-09-26 | 2026.10.0 weekly | Zero entities exposed; tool descriptions down 66 to 75 percent; first cut of the full security model | 3.9, 3.11, Part V |

Related sources:

* "Issues getting a local LLM to tell me temperatures, humidity, etc.", post 3 ([post](https://community.home-assistant.io/t/issues-getting-a-local-llm-to-tell-me-temperatures-humidity-etc/801976/3)): the original grandma thought experiment, November 2024. Chapter 1.
* "So You Want to AI in Home Assistant", chapters 1 through 3 ([thread](https://community.home-assistant.io/t/so-you-want-to-ai-in-home-assistant/1021777)): the box, context versus memory, the harness as the walls. Chapter 1.
* "A general problem with AI", post 6 ([post](https://community.home-assistant.io/t/a-general-problem-with-ai/1026421/6)): build a box and a scrapbook. Chapter 1.
* [Cognitive architectures whitepaper](../research/whitepaper_cognitive_architectures.md): the first CoALA mapping. Chapter 4.

## H. Research cited

Every study the book cites, with a link to the paper. The year is the first public version.

* Brown, T. B., et al. (2020). [Language Models are Few-Shot Learners](https://arxiv.org/abs/2005.14165).
* Holtzman, A., et al. (2019). [The Curious Case of Neural Text Degeneration](https://arxiv.org/abs/1904.09751).
* Hong, K., Troynikov, A., and Huber, J. (2025). [Context Rot: How Increasing Input Tokens Impacts LLM Performance. Chroma](https://www.trychroma.com/research/context-rot).
* Hsieh, C.-P., et al. (2024). [RULER: What's the Real Context Size of Your Long-Context Language Models?](https://arxiv.org/abs/2404.06654).
* Huang, L., et al. (2023). [A Survey on Hallucination in Large Language Models: Principles, Taxonomy, Challenges, and Open Questions](https://arxiv.org/abs/2311.05232).
* Ji, Z., et al. (2022). [Survey of Hallucination in Natural Language Generation](https://arxiv.org/abs/2202.03629).
* Kadavath, S., et al. (2022). [Language Models (Mostly) Know What They Know](https://arxiv.org/abs/2207.05221).
* Kalai, A. T., et al. (2025). [Why Language Models Hallucinate](https://arxiv.org/abs/2509.04664).
* Kong, A., et al. (2023). [Better Zero-Shot Reasoning with Role-Play Prompting](https://arxiv.org/abs/2308.07702).
* Lewis, P., et al. (2020). [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401).
* Li, C., et al. (2023). [Large Language Models Understand and Can be Enhanced by Emotional Stimuli](https://arxiv.org/abs/2307.11760).
* Liu, N. F., et al. (2023). [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172).
* Ouyang, L., et al. (2022). [Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155).
* Sclar, M., et al. (2023). [Quantifying Language Models' Sensitivity to Spurious Features in Prompt Design](https://arxiv.org/abs/2310.11324).
* Shi, F., et al. (2023). [Large Language Models Can Be Easily Distracted by Irrelevant Context](https://arxiv.org/abs/2302.00093).
* Shuster, K., et al. (2021). [Retrieval Augmentation Reduces Hallucination in Conversation](https://arxiv.org/abs/2104.07567).
* Sumers, T. R., et al. (2023). [Cognitive Architectures for Language Agents. Transactions on Machine Learning Research, 2024](https://arxiv.org/abs/2309.02427).
* Vaswani, A., et al. (2017). [Attention Is All You Need](https://arxiv.org/abs/1706.03762).
* Wen, B., et al. (2024). [Know Your Limits: A Survey of Abstention in Large Language Models](https://arxiv.org/abs/2407.18418).
* Xu, B., et al. (2023). [ExpertPrompting: Instructing Large Language Models to be Distinguished Experts](https://arxiv.org/abs/2305.14688).
* Xu, Z., et al. (2024). [Hallucination is Inevitable: An Innate Limitation of Large Language Models](https://arxiv.org/abs/2401.11817).
* Yao, S., et al. (2022). [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629).
* Zheng, M., et al. (2023). [When "A Helpful Assistant" Is Not Really Helpful: Personas in System Prompts Do Not Improve Performances of Large Language Models](https://arxiv.org/abs/2311.10054).

<!-- nav -->
---

[← Findings](26_findings.md) · [Contents](00_toc.md) · [Doc hub](../readme.md)
<!-- /nav -->
