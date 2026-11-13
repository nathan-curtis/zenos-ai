# Release Notes — 2026.11.0 'This Is Spinal Tap'

**Status:** Release candidate 3
**ETA:** Saturday, November 14, 2026
**Branch:** `feat/2026.11.0`
**Base:** 2026.10.1 (patch on 2026.10.0 'Tron')

---

## Summary

Security goes to eleven.

Tron made admission explicit and put a badge in every agent's hands. This release makes the badge matter, and puts locks on the cabinets behind it. FileCabinet holds identity state, component definitions, configuration, mounts and the material that becomes agent context, and its live drawers can invoke tools. Permission to change a drawer can change what the system later believes or does, so the drawer is the next boundary to enforce.

Picture it. The agent turns up at the stage door, here to see the Tap. Flynn, the roadie, is on the door with the clipboard.

"Hold on. Pass?"

No pass, no further. An agent without `zenos.agent.basic` gets as far as the door. It can say who it is, ask for its pass, and help you sort out whatever is keeping the request from reaching you, and that's all. Flynn radios the request through to your phone. You say yes.

"Yeah, let 'em in."

Backstage isn't the stage. The pass gets the agent in to see the band; every instrument still takes its own cert. The certs are new too: each one now lives under `zenos.*`, carries a date, and asks you on your phone in plain words.

The road crew needs passes too. The summarizer works when no one's on stage: it reads the house on a schedule and sometimes decides something needs doing. In 11 it works as a named agent, the one responsible for your household, and it acts on its own only with a cert you approved for the background.

And the guitar?

Don't touch it. Don't look at it. Don't EVEN point at it. In 11, a cabinet your agent has no pass for refuses the read. In 12 it won't even show up in the listing.

**Backstage passes for agents and locks on the cabinets.**

---

## What's New

- **The backstage pass.** Every agent-facing tool checks `zenos.agent.basic` first. An agent you haven't approved gets help and onboarding, and nothing else. See [The Backstage Pass](#the-backstage-pass).
- **Certs, renamed and readable.** Every cert lives under `zenos.*`, carries a date, and asks on your phone in plain words. See [Certs, Renamed and Readable](#certs-renamed-and-readable).
- **The background asks first.** The summarizer acts as your household's agent, reads private data only with a cert you approved for the background, and acts only through what a tool declares. Components follow one spec, KF6. See [The Pipeline Gets a Pass](#the-pipeline-gets-a-pass).
- **Locks on the cabinets.** One access record per cabinet, default rules for every class of cabinet, built-in locks on certs and approvals, and one gate that checks every hop of a mount. See [The Locks](#the-locks).
- **FileCabinet, rebuilt to be lockable.** One exit, honest writes, stated merges, and an admin door for structural writes. See [FileCabinet, Rebuilt to Be Lockable](#filecabinet-rebuilt-to-be-lockable).
- **Names in parts.** ZenOS follows Home Assistant's new naming model and child devices. See [Names in Parts](#names-in-parts).
- **What the new security unlocks.** Unattended camera scans, summary chains, label triggers and more, each off until you approve it. See [What the New Security Unlocks](#what-the-new-security-unlocks).

---

## Cranked to Eleven

**A fresh install is cranked to eleven:** Flynn arms FileCabinet's gate when it sets up the system cabinet, and the gate locks to enforce once ACL v2 is on every cabinet. **An upgraded install has the same knob, and you turn it:** until you do, the gate runs in audit mode. It decides every FileCabinet call, logs what it would have refused or asked a human about, and lets the call through, so the logs show what enforcement will change before it changes anything. The admin door's own gate is audit only on every install.

The knob is one drawer, `fc_acl_mode`, and one switch: `zen_admintools_prompt_loader mode=fc_acl_mode`, from the Home Assistant console. `action_type: enforce` turns an upgraded install to eleven, and `audit` turns any install back. No agent or automation can flip it.

One gate ships on everywhere: the backstage pass. See [The Backstage Pass](#the-backstage-pass).

---

## Scope of the Claim

FileCabinet enforcement protects the ZenOS access boundary. Home Assistant administrators, and any path with direct access to the underlying state, remain part of the trusted platform.

Identity in this release tells Flynn, the agent and the user apart. Telling one agent or one session from another is a later phase.

The backstage pass checks calls a user started. A call no user started (boot, Flynn, an automation, the scheduler) passes, so an automation that calls a ZenOS tool is trusted as far as whatever can trigger it.

---

## The Backstage Pass

Before 11.0, `zenos.agent.basic` was a badge an agent was given at the end of onboarding, and nothing checked it. Now it is the ticket to play.

**No pass, no tools.** Every agent-facing tool calls one gate first, `OS.entry_gate` in `zen_os_1.jinja`. An agent without the pass gets each tool's `help`, `tool_manifest` and `kfc_manifest`, and the few steps it needs to ask for the pass and help you fix whatever stops the ask from reaching you: onboarding, `whoami`, the cert request and the live-ack, alert, push and display hops it travels, and a health report trimmed to the approval path. Anything else answers `guarded: onboarding_required` with `next_step: zen_flynn_oobe mode=run`, and logs one `zen_entry_gate` warning. Calls no user started (boot, Flynn, automations, the scheduler) pass.

**Flynn hands it out first.** `zen_flynn_oobe mode=run` asks you to approve the agent (the `zenos_agent_basic` bundle, one push) before it hands out a single setup step. An agent that finished onboarding but was never approved gets one "waiting for approval" notice from Flynn, and answers as itself until you say yes.

**No onboarding, no pass, no breakage.** An unapproved agent can't do anything, so it can't break anything, and it still has enough to walk you through fixing the approval path. If the push can't reach you, the console override stands in, and now works before approval. Both routes are in the [Install Guide](../getting_started/install.md#step-35--set-up-your-mobile-notification-path) and the [Security & Certification manual](../getting_started/security_certification_manual.md#25-the-backstage-pass-zenosagentbasic).

**Home Assistant can't tell you from the agent.** A dashboard button, or your own call from Developer Tools, runs as a logged-in user, the same as the agent. Until the agent is approved, those calls are refused too.

**A refusal stops the call.** In more than 20 tools, a refusal or cert denial could be overwritten by a later branch that went on to do its side effect; a denied Autovac `pause` still paused. Every top-level branch now waits on the result. See [Fixed](#fixed).

---

## Certs, Renamed and Readable

**Every cert lives under `zenos.*`.** The 42 flat names moved to dotted ones: `lock_control` is `zenos.actuation.lock_control`, `todo_control` is `zenos.comms.todo_control`, and so on. For one release, a grant under either name satisfies a check against the other, and `cert_list` shows an old-name grant as held with `reissue_pending`. Reissuing it under the new name replaces the old grant and keeps its scope. The alias goes in 2026.12.

**Approvals read in plain words.** Every cert has a short `description` written for the phone, and a `detail` for the long form. The approval prompt reads "<agent> asks to: Lock any door and unlock interior doors (zenos.actuation.lock_control, level 1). Approve?" A bundle lists each member the same way. The catalog is in the [Security & Certification manual](../getting_started/security_certification_manual.md#3-the-certification-catalog).

**Grants are dated.** Every new grant records `not_before` and `not_after` (one year), in UTC, with the X.509 field names. Nothing enforces expiry yet.

**The vault holds the catalog.** The vault cabinet is now a required system cabinet that Flynn provisions on boot, and only the admin door can read it. `cert_list` reads the cert catalog from a cache there, so it answers in a fraction of the time, and builds it live whenever the vault isn't there. Grants are always read live.

## The Pipeline Gets a Pass

The summarizer pipeline works differently in 11.0. A Ninja run reads one component of the house, a model writes its summary, and if the summary calls for action the run can act, usually by sending a notification through Postman. SuperSummary then writes the whole-house summary from those. Until now the pipeline answered to two lists in the system cabinet: a seed whitelist for the tools a run could read from first, and an act whitelist for the acts it could take. Neither said which agent was acting, and nobody approved either on a phone.

In 11.0 a run:

1. **Acts as a named agent:** the one responsible for the household, which must be onboarded. Otherwise the run stops and summaries pause, with a notification saying why.
2. **Reads only what the tool declares:** a seed must be a mode the tool lists for the summarizer, or the run stops before the model is called.
3. **Acts only with a cert granted for the background:** the act's cert must be held with the Ninja emit stage, which a person approved. Otherwise the act doesn't happen, and the summary records what to grant.

SuperSummary writes as the same agent and builds its capsule from it, not from the persona-name helper.

A cert used to answer one question: may the agent do this when someone asks? Now a grant also says where it may be used.

**Grants name their stages.** A grant carries `st`, one letter per pipeline stage, stored the way the ACL ops are:

| Letter | Stage |
|---|---|
| `i` | interactive |
| `l` | live drawer |
| `s` | Ninja seed |
| `e` | Ninja emit (the act at the end of a run) |
| `m` | SuperSummary |
| `c` | scheduler |

A grant without `st` is interactive only, so every existing grant behaves exactly as before. Outside `i`, only an exact grant counts, never a parent cert. Each tool declares the stages its certs allow (`st_ok`), and a grant can't ask for a stage its tool doesn't allow. Adding a stage is a phone approval like any other grant, and the prompt says it in plain words: "in the background, with no one asking". `cert_req_grant` and `cert_req_revoke` take `cert_stages`, and a revoke can drop one stage and keep the rest. In 11.0 four stages are checked: interactive, the live-drawer stage (a live drawer that runs a gated mode of its tool), the Ninja seed stage (read certs and a component's minimum level) and the Ninja emit stage (acts and chains). No tool allows the live-drawer stage yet, so it can't be granted until one does. The SuperSummary and scheduler stages are reserved.

**The pipeline acts as the agent responsible for the household.** That is the prime in the household's ACL, else the default agent, and it has to be onboarded. If it isn't, summaries pause with a notification instead of running as nobody. SuperSummary builds its capsule from that agent, not from the persona-name helper.

**Seeds are declared by the tool.** A tool lists the read modes the Ninja may seed a run with in its `tool_manifest` (`seed_modes`, each `mode` or `mode:case`). An undeclared seed refuses the run before the summary model is called, naming the modes that are declared. Index, Room Manager, AutoVac, Taskmaster, Inventory and Finance declare theirs. Finance names four `run` cases, not every one.

**Acts need the emit stage.** Before the Ninja sends anything, it finds the cert on the target tool whose stages include `e`, and the responsible agent must hold that cert with stage `e`. Today that's Postman's `zenos.comms.dispatch`. A refused act is written onto the component's summary as `act_blocked`, with what to grant. A life-safety act skips the cert check, as Postman's own gate does. The kill switch and the infrastructure deny still apply to it.

**`zenos_agent_autonomous`** is the bundle for this: every cert whose tool allows the Ninja emit stage. It isn't a hand-kept list, so a tool that later allows `e` joins on its own. Granting it adds `ie` (interactive and Ninja emit) by default, so one phone approval covers it. A bundle grant never lowers a level the agent already holds.

**The 11 bridge.** Through 11, the old seed list still passes a tool that declares no `seed_modes`, and the old act list still passes an act whose cert the agent holds for interactive use only. Both are stamped legacy, and the act side raises a notice saying what to grant. `prompt_loader mode=whitelist` still manages both lists, with a deprecation note. The lists and the bridge are removed in 12.

---

## The Locks

**One record.** Identity and FileCabinet share one ACL record in each cabinet's VolumeInfo: `acls: {v: 2, owner, prime, partners, ops}`. Principals are cabinet GUIDs, never names. `ops` holds a rule for each of read, write, delete and admin: who may do it (`who`), the cert it needs, and whether a live human must say yes (`ack`). Identity writes the record, through the admin door. FileCabinet and Identity read it only through the CABS `acl_*` macros and `ID.principals`, so there is one reading of the rules, not two.

**Every cabinet has a class, and every class has defaults.** A cabinet is `system`, `vault`, `dojo`, `kata`, `history`, `household`, `family`, `user`, `ai_user` or `scratchpad`. Each class has a default row: the system cabinet is read-only to everything but the admin door, the vault takes writes only from Identity and is read only through the admin door, a person's cabinet answers to that person, their partners and the prime agent, and a scratchpad is open. A drawer may tighten or loosen its own ops; each op falls back on its own to the cabinet's rule and then the class default. The full table is in the [zenos_cabinets.jinja reference](../custom_templates/zenos_cabinets_jinja.md#acl-v2). The class is a stamp in VolumeInfo, which only the admin door can change. The gate never reads a label.

**Some drawers no owner can open.** Certs (`zen_ai_certs`), integration settings (`integrations_config`) and live-ack answers (`alert_response_*`) take writes only from tools, and the console override (`cert_admin_console_bypass`) and the enforce switch (`fc_acl_mode`) only from the admin door. These built-in locks win over every other rule, so `acl_set` can't loosen them.

**The gate knows who's asking.** The agent-facing FileCabinet tool marks every call it makes as an agent call. A tool calling the engine is internal, and can name itself with `caller_tool`, so a rule can say `@tool:zen_dojotools_identity` and mean exactly that. New principals in `who`: `@internal`, `@tool:<script>`, `@self`, `@partner`. Before this, every FileCabinet call looked like it came from the default agent. Now the audit log tells an agent from a tool.

**Mounts confer no authority.** FileCabinet resolves the real target behind a mount chain (up to 8 hops; a loop refuses) and checks every hop, none skipped: each pointer the chain passes needs read, and the drawer it ends on needs the action's op. A caller that can't read a pointer doesn't get past it. A pointer answers with its cabinet's rules, never a drawer's. An expansion answers with its host cabinet's rules, so moving a drawer into one changes where it lives, not who can reach it.

**Two gates.** FileCabinet's gate enforces on a fresh install and audits on an upgraded one. It answers `acl_audit` and logs `fc_acl_audit` with the caller's `origin` and, per check, where the rule came from and how the caller matched. The admin door checks agent and console writes against `zenos.system.cabinet_admin` and answers `gate_audit`; it audits on every install. Boot, Flynn and automations are exempt.

**One switch to enforce.** The system cabinet's `fc_acl_mode` drawer, `{mode: audit|enforce}`, sets FileCabinet's gate. Missing means audit. In enforce, a deny is refused with `guarded:`, and a rule that asks for a live ack asks the household admin every time; not granted is `denied`. Only the admin door writes the drawer, and the way to set it is `zen_admintools_prompt_loader mode=fc_acl_mode` from a console session (`action_type` `status`, `audit` or `enforce`), the same check as the console override. No agent, script or automation can flip it.

**A fresh install locks itself.** When Flynn sets up the system cabinet, which happens only on a fresh install or after `reset_all`, it writes the drawer as audit, armed. Identity's `acl_migrate` turns an armed drawer to enforce once the ACL schema is current, logs "FileCabinet locked", and answers `fc_acl_locked: true`. An upgraded install never gets an armed drawer, so it stays in audit. Setting the switch by hand drops the arm, so your choice stands. `acl_audit.mode` on every answer says which applied. The how-to is in the [FileCabinet readme](../scripts/zen_dojotools_filecabinet_readme.md#access-control-acl-v2).

**Escape hatches, with a paper trail.** A cabinet's owner changes its rules with `identity acl_set`. `identity take_ownership` takes a cabinet over: it needs `zenos.identity.acl_take_ownership` and a live human yes on every call, rewrites only the ACL, never the contents, and is audited. The console override stands in for that yes (and every other live acknowledgement) in exactly one place, Identity's `request_live_ack`, and each use is audited.

**One migration, stamped.** `identity acl_migrate` brings every cabinet to the record: it converts v1 lists to GUIDs, stamps classes, repairs bad identity data and stray VolumeInfo copies, and never drops a dangling principal. A clean run stamps `acl_schema` version 3 on the system cabinet. Flynn runs it on a fresh install and skips it while the stamp is current; `systemtools cabinet_schema_upgrade` runs it when the stamp is behind.

---

## FileCabinet, Rebuilt to Be Lockable

A gate is only as good as the engine behind it, so FileCabinet was rebuilt first.

- **One exit per action.** Get, create, update, upsert, delete and list each set a result and fall through to the engine's single exit, so the gate sits in one place. The agent-facing tool has one exit too, where it had 22: each step runs only while no earlier one has answered, so a cross-stack move never deletes its source after a failed write, and a failed engine call answers with an error naming the stack instead of nothing.
- **Honest writes.** A write the cabinet never confirmed answers `write_unverified` and logs a warning, instead of success. `update` on a missing key answers `not_found`. Delete reports `children_deleted`.
- **Every write is a stated merge or replace.**
- **One read taxonomy.** Content is in `value`; `read` is `get`.
- **The drawer envelope is `{value, meta, timestamp[, acl]}`.** `meta` is always present and carries expiry; `acl` carries a drawer's own rules. A cabinet set keeps both instead of rebuilding the drawer.
- **The admin door.** `zen_admintools_cabinet_writer` is the one writer for structural, system and vault cabinet writes, which FileCabinet refuses. It isn't MCP-exposed, every write names its caller and is logged, and it waits until the cabinet shows the write before it answers, and confirms the content. `merge` mode writes only the keys that changed. AdminTools, Flynn, GC and maint fire no raw cabinet events any more.
- **The cabinet guards hold.** The 128 KB size cap and the unknown-at-boot guard refuse the write and leave the cabinet as it was.
- **Expansions are a permanent redirect.** `expand_drawer` moves a root drawer and its whole tree into an expansion slot unchanged (value, meta and the drawer's own ACL), verified byte for byte, and leaves a pointer per key. Reads, writes and new children follow the drawer there, and `get` and `list` show the cabinet and its expansion as one tree. One expansion holds one drawer, a drawer expands once, and the pointers can't be edited. Move, copy and delete work on the real drawers wherever they live, and an operation that empties an expansion tears it down. `collapse_drawer` brings the tree back, checking first that it fits. Any failure along the way rolls back. See [Expansions](../scripts/zen_dojotools_filecabinet_readme.md#expansions--expand_drawer-and-collapse_drawer).

---

## Names in Parts

Home Assistant's names grew in layers that didn't always agree. Names first had to carry all their own context ("Kitchen coffee machine power"). Then devices and areas made that context repeat, so moving a device to another room left every one of its entity names wrong. In 2026.11 each thing is named once, for what it is: the area "Kitchen", the device "Coffee machine", its entities "Power" and "Energy". Home Assistant adds the context where it's needed. Existing entity IDs don't change, and names that start with their device's name lose that prefix. The `friendly_name` attribute, which combines the device and entity names, is kept for backward compatibility for now. It is on its way out, and Home Assistant will announce its removal ahead of time. A device made of several parts, such as a four-socket power strip, now gets a child device for each part, starting with Z-Wave, MQTT and ESPHome. A child's link to its parent (`parent_device_id`, in the registry since HA 2026.9) is separate from the hub HA reaches a device through (`via_device_id`). The Home Assistant team explains the why in [Naming things in Home Assistant, shaped by your feedback](https://community.home-assistant.io/t/naming-things-in-home-assistant-shaped-by-your-feedback/1032131).

ZenOS speaks that model in 11.0, and it took Display's place to do it. Display builds on what the agent can name; the names come first. Every entity's ID badge has to tell the truth: which entity it is, which device it belongs to, and where that device sits.

- **Nothing decides anything from `friendly_name`.** Inspect decides that an entity exists from its state object, so an entity without the attribute still returns its state and attributes. Room sensors find themselves by their room, and Room Manager finds a person's zone sensor by the person, not by a display name.
- **Names come in parts.** Outputs carry `entity_name` and `device_name` (Inspect adds `area_name`) next to the name key they already had, never one composed string. Text composes from the parts where HA has `entity_name()` (2026.4 and later), else from the legacy `friendly_name`, else the entity id.
- **`friendly_name` stays, as legacy,** while HA still sets it. ZenOS's own `friendly_name` fields (VolumeInfo, KFC, capsules) are its own schema and are unchanged.
- **The device name is the effective one** (`device_name()`), so a name you set wins over the integration's.
- **Child devices are followed.** Inspect's device tree keeps the hub (via) and the parent apart and lists child devices. A device selector in Index, Security Manager's zone walk and Spa Master's anchor device include child devices' entities, the way HA targets a parent.
- **Gated on the HA version,** read from the core update entity. Without Supervisor there is no update entity, and the code detects the feature instead.

---

## What the New Security Unlocks

Each of these was unsafe, or a dead path, before 11. Now each one ships switched off, behind a cert its tool declares, and turns on only when you approve it on a real device. The grant history shows exactly what the agent can do in the background, and why.

- **Unattended camera scans.** The camera heartbeat scans on its own schedule: exterior always, interior only when you're away. It needs `zenos.security.camera_scan` for the background.
- **Summaries that start other summaries.** Up to three in a row, only to the targets each one lists in `chain_to`, each hop gated on its own, and never in a loop. It needs `zenos.pipeline.chain`.
- **Acts any tool can declare.** A tool says what it does and what it takes, and the pipeline routes to it with no hand-written route. That's the base for future acts such as lights, covers and locks, each behind its own cert.
- **Private data in background summaries, with consent.** Finance, medication and presence summaries can run unsupervised, because the agent may only read them with a read cert you approved. Turning the cert off pulls the data out of the prompt too.
- **A medication summary of its own.** Meds no longer ride along in the household task summary.
- **Any device you can label becomes a trigger, with no YAML.** Label a leak sensor `zen_leak`, write a component that subscribes to it, and it runs when the sensor gets wet. Leak, smoke, CO and gas sensors work on day one once they're labeled. Alert Manager wakes the moment a leak sensor labeled `zen_leak` gets wet, a detector labeled `zen_smoke` senses smoke, CO or gas, or an alarm panel labeled `alarm_panel` is triggered, and a life-safety alert is never held back because another alert went out a few minutes earlier.
- **Your agent knows what it's allowed to do.** Agent health now says whether your agent is onboarded, and when something is held back by a missing permission it shows `restricted` and names it. The health report tells the agent which permissions it can ask you for, and which problems need a person to fix (so it won't keep asking).
- **The agent can manage what it exposes, safely.** Ectoplasm can expose and unexpose entities, because the admin plane can never be exposed: expose always refuses it, and Flynn removes it every hour.
- **Turning a component on asks for its own certs.** You don't have to work out which cert a summary needs.
- **One approval covers an upgrade.** `zenos_agent_parity` replaces chasing individual grants, and read certs start in audit on an upgrade, so a working install does nothing.

See [Building a KFC](../kung_fu/building_a_kfc.md#kf6-acts-chains-read-certs-and-triggers-2026110) and the [Security & Certification manual](../getting_started/security_certification_manual.md#66-read-certs-and-parity).

---

## Still to Come

Planned for 2026.12.0:
- Enforcement by default on upgraded installs, and at the admin door.
- Hiding unreadable drawers from listings and search.
- The operator permission report: who is asking, what they hold, what they can reach, and why a call was refused.
- System and Default Agent authority, separated.
- The summarizer's seed and act whitelists and the old cert names removed (both still work through 11).
- A core pack that fills Security Manager's cache slots on a stock install.
- Display finished (it stays beta in 11): the shared dashboard, and the remaining Media Manager watch paths.

Later: the admission-token contract with `zen_root_authentik` (it needs an OIDC plugin first), revocation and recovery on the trusted path, and binding to the conversation or session.

---

## Read this before upgrading

- **Approve your agent.** An agent that never received `zenos.agent.basic` loses every ZenOS tool except help and onboarding until you approve it: run `zen_flynn_oobe mode=run` (or ask the agent to) and say yes to the push. An agent you approved at the end of onboarding in 2026.10 already holds it. Until then, dashboard buttons and your own Developer Tools calls to ZenOS tools are refused too. The push goes to `notify.admin_devices` until Postman has a profile; if no push can reach you, use the console override ([Install Guide](../getting_started/install.md#if-no-push-can-reach-you)).
- **The ACL v2 migration is mandatory.** Identity setup refuses an old-schema household or family with `acl_schema_upgrade_required`. The migration also stamps each cabinet's class; until it does, a cabinet already on v2 gets the unclassified defaults. After reloading (templates, automations, template entities, then scripts), run `zen_dojotools_identity mode=acl_migrate` to see the plan (a dry run is the default). Check every proposed cabinet `class`, especially any it calls `scratchpad`. Then apply it with `dry_run: false` and `confirm_action: true`, or run `zen_dojotools_systemtools tool=cabinet_schema_upgrade`, which applies it when the schema stamp is behind. A second dry run should plan nothing, and the system cabinet then holds `acl_schema` version 3. Flynn's bootstrap runs the same migration on its own until the stamp is current: on a fresh install, and on an existing one the next time the bootstrap runs. Drawers need no migration of their own; existing ones read as before.
- **Certs are renamed, and you have until 2026.12.** Nothing to do now: an old-name grant keeps working and shows `reissue_pending` in `cert_list`. Reissue it under its `zenos.*` name before 2026.12. Re-granting `zenos_agent_basic` reissues every member at once.
- **Re-grant `zenos_agent_basic` if to-do writes fail.** `zenos.comms.todo_control` level 1 (then `todo_control`) joined the bundle in 2026.10.1, but an agent that was granted the bundle's certs before then doesn't hold it. Re-grant `zenos_agent_basic`, or grant `zenos.comms.todo_control` level 1, before to-do writes work.
- **Enforcement is opt-in on an upgrade.** An upgraded install stays in audit. Read the `fc_acl_audit` lines first (`would_deny=True` is what enforce would refuse), then lock it from the Home Assistant console with `zen_admintools_prompt_loader mode=fc_acl_mode action_type=enforce`. `action_type=audit` is the rollback. A fresh install of 11 locks on its own.
- **A live drawer of your own that runs a gated mode** logs `live_cert_audit` on an upgrade (audit), and is refused on a fresh install (enforce) until its tool allows stage `l`. The shipped live drawers are unaffected.
- **Expect audit lines.** Agents that don't hold `zenos.system.cabinet_admin` log `door_gate_audit` lines, and the FileCabinet gate logs `fc_acl_audit` wherever enforcement would have refused or asked. They are the preview of enforcement, not errors.
- **Cabinet labels need `zenos.system.cabinet_admin` level 2.** Tagging or untagging a cabinet, or creating, changing or deleting a `zen_*cabinet` label, is refused without it. Automations, the scheduler and Flynn are not affected.
- **The vault cabinet provisions itself.** `sensor.zen_cabinet_health` and the health report show warnings until the vault is online; Flynn provisions it on the first pass after warmup, within minutes. To do it straight away, trigger `automation.flynn_stepgate_sentinel`. An install whose vault never left `unknown` (every vault write was dropped before this release) gets it initialized on that pass too.
- **FileCabinet reads return content in `value`.** The `{path: value}` map that `read` put in `result` is retired; `result` is only the tool envelope, and `read` is a plain alias of `get`. Anything of your own that reads `result.<path>` needs to read `value`.
- **FileCabinet writes are honest.** A create, update, upsert or delete the cabinet never confirmed answers `status: error`, `code: write_unverified` instead of success. `update` on a missing key answers `not_found` (`force_action` creates it), and `create` on an existing key is an error unless `force_action`.
- **Every write is a merge or a replace.** `upsert` deep-merges by default and `update` replaces. A read-modify-write caller passes `write_mode: replace`; under a merge, a key you removed comes back.
- **Structural cabinet keys belong to the admin door.** VolumeInfo, `meta`, `_label_index`, `_zen_relationships`, the system cabinet and the vault are written only through `zen_admintools_cabinet_writer`, and FileCabinet refuses them. Raw `set_variable_*` events of your own aimed at those keys need to move to the door.
- **The lens and codex registries have one writer each:** `prompt_loader mode=lens_registry` and `mode=codex_registry`. Code writing those drawers directly needs to go through them.
- **`run_repair` has one action, `acl_migrate`.** The 4.5.6 repair names are refused, and `maint_4_5_6.yaml` and `repair_fam_cab_volumeinfo.yaml` are gone. The cabinet factory no longer takes `family_guid`.
- **`reset_all` wipes, and now it wipes everything.** Cabinet admin's `reset_all` did nothing while answering success. Its default now, `scope: all`, is a true fresh start: every cabinet (your user and AI user cabinets, the vault and expansions included) and every `zen_` label on the house, your own included. Flynn recreates only its own labels. `scope: core` is the old wipe of the ring-0 cabinets only, labels kept. Run it with `dry_run: true` first to see the list. `scope: all` also needs `zenos.registry.label_lifecycle_control` at level 2. A wipe clears the console override and every cert along with everything else, so throw the override from the console and re-request certs afterwards. After a `scope: all` wipe the install comes back as a fresh one, FileCabinet lock included.
- **`reset_labels` asks every time.** It needs `zenos.registry.label_lifecycle_control` at level 2 and a live household-admin yes on every call; `confirm` alone no longer runs it.
- **If `script.zen_admintools_kungfu_loader` shows unavailable,** delete its entity registry row. The loader was retired in this release.
- **Inspect's `device_tree.parent_name` is now `via_device_name`.** It always held the hub's name, not a parent's. `parent_device_name` is the child-device parent now. Anything of your own that reads `parent_name` needs the new key.
- **Summaries pause until the household's agent is onboarded.** The summarizers act as the household's prime agent, else the default agent. If that agent isn't onboarded, both refuse with `agent_not_onboarded` and post "ZenOS: summaries are paused". Onboard it through Flynn.
- **SuperSummary writes as that agent,** not as the persona named in `input_text.zenos_persona_name`.
- **Grant `zenos_agent_autonomous` for background notifications.** The Ninja's routine Postman sends need `zenos.comms.dispatch` with the Ninja emit stage. The bridge only covers an agent that already holds that cert for interactive use, and many upgraded installs don't. Until you grant the bundle, routine sends are refused and written onto the component's summary as `act_blocked`. Life-safety sends aren't affected.
- **A seed tool of your own needs `seed_modes`** in its `tool_manifest` before 2026.12. Until then the old seed list carries it, with a legacy stamp on each run.
- **The Ninja's context cap now trips.** It never could before (see Fixed), so a component with a very large prompt that used to run may now stop with `ninja_context_overflow`. Raise `max_context_tokens` in `zen_ninja_config` if your model's context window allows it, or lower `domain_inspect_limit` in that component's `index_call`.
- **Read certs start in audit, so nothing changes.** Finance, medication and presence summaries keep running and log `seed_cert_audit` where enforce would refuse. To lock them: grant `zenos_agent_parity` (one phone approval), then switch from the console with `zen_admintools_prompt_loader mode=seed_cert_mode action_type=enforce`. A fresh install starts in enforce and asks for parity at onboarding.
- **Master switches are gone.** A component turns on and off only through its drawer: Scribe `mode=enable` / `disable`. Nothing ever read a master switch, so nothing changes, but an `input_boolean` you kept for one does nothing.
- **Medications have their own summary.** The `taskmaster` summary no longer carries meds; `taskmaster_meds` does, and registers at the next `bootstrap_kfc` (boot or 00:01).
- **A summary of your own that seeds Taskmaster `briefing` keeps working.** Bare `briefing` is no longer a seed mode, so it's mapped to `briefing:household` (without meds), and a one-time notice says so. To make it permanent, update the seed in Scribe; for meds, turn on `taskmaster_meds`.
- **Admin tools leave the agent's tool list.** Flynn removes every `zen_admintools_*` script from Assist, Alexa and Google every hour. A test server that needs them sets the system cabinet's `admin_plane_guard` drawer to `listed`. The guard needs Spook.
- **The camera heartbeat and summary chains stay off until granted.** Both were dead paths before. Grant `zenos.security.camera_scan` or `zenos.pipeline.chain` with the background stage (or `zenos_agent_autonomous`) to turn them on; a chain also needs `chain_to` on the component that starts it.
- **Check what each component needs:** `zen_dojotools_manifest mode=component_certs`. A component listed `never_fires` subscribes only to triggers nothing registers.
- **Prefer `zen_door` over the group triggers.** `door_opens_or_closes`, `window_opens_or_closes`, `lock_changes` and `garage_door` watch `group.all_doors` and its siblings, and never fire without them.
- **A trigger automation of your own** can register its IDs (see the [custom scheduler example](../examples/zen_dojotools_scheduler_custom.yaml)), so its components don't show as `never_fires`.
- **Your own copy of the scheduler loop** needs the same fix as the scheduler (see Fixed): sleep the gap since the previous subscriber, not each subscriber's whole delay.

---

## Added

- **Declared acts and summary chains.** A tool declares its act (`act: {mode, args, description}`) next to its emit cert; the Ninja tells the model what it may return and checks it (`no_act_declared`, `act_args_invalid`), and the dispatcher routes any declared act with no hand-written route. Camera declares `scan` behind the new `zenos.security.camera_scan`. A summary can start another through the Ninja's `run` act, behind `zenos.pipeline.chain`, only to targets in its `chain_to`, at most 3 deep and never in a loop (`chain_loop`, `chain_depth`).
- **Read certs on seeds.** A tool tags a seed mode with the cert that reads it (`'mode@cert:level'`). Finance, Room Manager's `home_overview` and Taskmaster's medications carry `zenos.finance.read`, `zenos.presence.read` and `zenos.health.read`. The switch is `seed_cert_mode`: audit on an upgrade, enforce on a fresh install. Under enforce a missing cert refuses the run (`seed_cert_required`) and that summary's data leaves the prompt.
- **`zenos_agent_parity`,** one approval that keeps an upgraded install running as before, built from what your enabled summaries actually need.
- **Turning a component on asks for its certs.** Scribe's preflight lists every cert the run needs and requests the missing ones for the agent that will run it. `component_certs` shows every component's status, and flags an act whose tool doesn't exist (`act_broken`), so a summary can't silently notify through a retired tool.
- **The trigger registry and label triggers.** `zen_trigger_registry` lists every trigger a component can subscribe to; packs register themselves from automation context. Six label triggers (`zen_leak`, `zen_smoke`, `zen_door`, `zen_power`, `zen_appliance`, `alarm_panel`) fire on whatever carries their label. Scribe `list_triggers` reads the registry, and a component nothing can fire shows as `never_fires`.
- **The admin plane guard.** `prompt_loader` is never exposed, Flynn unexposes the admin plane every hour, and `mcp_sync` flags an exposed admin tool as critical. Ectoplasm gains `entity_expose` (refuses the admin plane) and `entity_unexpose`.
- **A component's minimum level is enforced (KF6).** A KFC's `meta.requires: {cert, level}` was stored and never read. Now the agent the pipeline acts as must hold that cert at that level, granted for the Ninja seed stage `s`, or the run stops before the model with `kfc_cert_required`, keeps the last summary, and a notification names what to grant. An unmet component stays out of the agent's prompt. Scribe sets it (`requires_cert`, `requires_level`) and refuses any write of a requirement that could never be met (`kfc_requires_invalid`), and `bootstrap_kfc` rewrites a mount when a tool changes it. Every shipped component has no requirement, so nothing changes on upgrade. See [Building a KFC](../kung_fu/building_a_kfc.md#kf6-the-security-contract-2026110).
- **Media Manager turns players on and off.** `power_on` and `power_off` need `zenos.media.playback_control` and a player that reports the power feature, with no role label; `power_on` refuses in a Paused room. Switching a source, applying a preference or starting an activity wakes a player that's off first. See [Media Manager](../components/media_manager.md#device-control).
- **Print Shop knows the whole printer.** `status` says when the paper is out and an ink is low by the printer's own threshold, and on Epson printers adds the maintenance box and page counters from the `epson_workforce` integration, matched to the same printer by model. A new `printers` mode lists every printer with its room, Bambu Lab 3D printers included: their status and a `readiness` check are read-only, and nothing is ever sent to a 3D printer. See [Print Shop](../scripts/zen_dojotools_print_readme.md).
- **Semantic roots.** The tool manifest publishes a root for each question word: who (Identity), what (Index), when (Taskmaster almanac), where (Room Manager), why (history), how (help) and knowledge (FileCabinet). Each root answers what matters now and names where to go deeper, and the boot prompt renders them as one line.
- **Taskmaster almanac, the "when" root.** One call returns a headline and the current period of the day, vacation, events from calendars carrying the `almanac` label (timed events first, at most two per calendar), tasks due by today, meals and weather. `horizon_hours`, `lookahead_days` and `max_items` tune it.
- **Per-tool roots on the core tools.** Room Manager, Index, FileCabinet, Identity and Taskmaster declare their own roots in their `tool_manifest` modes. A bare call returns them in about 350 characters.
- **Cortex 46.** Cortex 44 'Open Door' with one fix: its `core_tools` list named a log viewer that does not exist, and now names `zen_dojotools_ha_log_viewer`. Its `os_release` stamp carries only the cortex, its codename and when it was pressed, without the stale list of component versions. 46 is the new latest, and the selectable versions are 46, 45 and 44.
- **Who knows where a member belongs.** `identity resolve` returns `name`, `person_entity`, `member_type` and `memberships`, and finds a person anywhere in the household tree, not only among the household's own users.
- **The ACL record, cabinet classes, built-in locks, caller origin, both audit gates, and Identity's `acl_migrate`, `acl_set` and `take_ownership`.** See [The Locks](#the-locks).
- **FileCabinet GC `repair` mode.** Normalizes one drawer to the `{value, meta}` envelope, with `dry_run`.
- **Inspect lists the lenses that apply.** Every inspection carries `domain_context.lens_options`: each lens provider that matches one of its anchors, what it returns and whether it is available, built from the registry with no provider calls. `+lenses` fetches all of them; `lenses=<provider,...>` fetches the ones you name.
- **Scribe asks instead of refusing.** A live KFC write by a caller without `zenos.knowledge.kfc_publish` asks the household admin to approve that one write; declined or timed out, it is refused as before.
- **The lens registry can be expanded.** Lens register and unregister go through the central lens writer and read through the pointer, so `lens_registry`, one of the household cabinet's biggest drawers, can move to an expansion without being wiped.
- **FileCabinet `collapse_drawer`.** Undoes an expansion: the tree comes back to its cabinet, verified, and the slot is freed.
- **Cabinet capacity.** `sensor.zen_cabinet_health` gains a `cabinet_capacity` attribute: each cabinet's size as a percentage of the 128 KB cap, the fullest cabinet, and a warning list at 90% and over. The sensor's state is unchanged. `zen_health_report` adds a `cabinet near full` warning for each one, naming `expand_drawer` and `expansion_sitrep`.
- **The backstage pass.** Every agent-facing tool checks `zenos.agent.basic` first. See [The Backstage Pass](#the-backstage-pass).
- **The live-drawer stage.** A live drawer runs its tool as the agent responsible for the household, and a gated mode needs that agent to hold the cert with stage `l`. `seed_cert_mode` decides: audit logs `live_cert_audit` with `would_deny` and runs, enforce refuses with `live_cert_required`. No shipped tool allows `l` yet, and no shipped live drawer runs a gated mode.
- **Approval first.** `zen_flynn_oobe mode=run` asks for the `zenos_agent_basic` bundle before the setup protocol; `complete` answers `admission_pending` while the pass is still missing; `status` reports `badge_held`. Flynn's Sentinel posts one "waiting for approval" notice for an onboarded agent without the pass, and clears it once the pass is granted.
- **Views before approval.** Identity `whoami` shows an unapproved agent only itself (name, cabinet, onboarded, `has_basic: false`) and the next step. `zen_health_report` covers only the approval path; the full report names the house's areas and devices.
- **Plain cert descriptions and approval prompts, dated grants, and the cert catalog cached in the vault.** See [Certs, Renamed and Readable](#certs-renamed-and-readable). `cert_list` gains `catalog_source` and `force_refresh`; the scheduler's script reloads and Flynn keep the cache fresh. It also answers with plain `notes` when grants are still under an old name or held above a cert's `max_level`, since that's the one place the agent learns its grant state.
- **Name parts and child devices.** See [Names in Parts](#names-in-parts). Inspect adds `entity_name`, `device_name` and `area_name`, and its device tree adds `via_device_name`, `parent_device_id`, `parent_device_name` and `child_devices`. `device_info` adds `parent_device_id` and `child_devices`, and `device_list` adds `parent_device_id`. ZQ-1 adds `entity_name_regex` and `device_name_regex`.
- **Radar moves a ticket between tags.** `ticket_untag` removes a tag. `ticket_retag` swaps one for another: it adds the new tag first and removes the old one only if the add worked, so a ticket never ends up with neither.
- **Pipeline stages on cert grants, the responsible agent, and `zenos_agent_autonomous`.** See [The Pipeline Gets a Pass](#the-pipeline-gets-a-pass). Identity gains `cert_stages`, `stage_letters` and `responsible_agent`, and `ID.caller` takes a container and a stage. The cert catalog carries each cert's `st_ok`, and `my_certs` each grant's `st`. `tool_manifest` takes `seed_modes`.
- **Tool summaries and a paged directory.** Every tool's `tool_manifest` carries a `summary`: one plain line saying what it is for. `zen_dojotools_help mode=directory` lists them a page at a time (`page`, `page_size` up to 50, and `tier` for one tier), and it builds only the page you asked for, so it stays quick on a big install. The response says how many pages there are and gives the call for the next one.
- **A fresh install locks the FileCabinet.** See [The Locks](#the-locks). The switch is `zen_admintools_prompt_loader mode=fc_acl_mode` (`status`, `audit`, `enforce`), console only; `status` also says whether a fresh install armed it.
- **`reset_all` takes `scope` and `dry_run`.** `scope: all` (the default) wipes every cabinet and every `zen_` label; `scope: core` wipes the ring-0 cabinets only. `dry_run: true` lists the cabinets and labels and writes nothing. A clear the cabinet writer refuses comes back in a `failed` list with `status: partial`.
- **The vault is a required system cabinet.** Flynn labels and initializes it on boot. `sensor.zen_vault_cabinet_resolved` resolves only when exactly one cabinet carries `zen_vault_cabinet`. Its read rule is the admin door: Cabinet Writer's new `get` mode is the one way to read it.

## Changed

- **Master switches are gone.** A component turns on and off only through `meta.enabled`. Scribe ignores `master_switch` and stores it empty, and Security Manager's KFC drops its own.
- **Taskmaster's briefing takes a `case`** (`household`, `medications`, or none for everything), and meds move to their own `taskmaster_meds` summary.
- **Security Manager's three sub-parts are cache slots.** `security_perimeter`, `security_panel` and `security_cameras` subscribed to triggers nothing fires. Now they're `role: cache`: a trigger pack writes them, the rollup reads them, and the Ninja never summarizes over them (`cache_slot`).
- **Help is the default.** A `zen_*` script answers help for a missing or unknown mode, and a bare call (no mode and no input) never acts, crashes or errors. Tools that used to act on a bare call (announce, the summarizers, the prompt loader, the reset and repair tools, Camera, Provisioner and FileCabinet GC) now return help. Calls that pass input are unchanged, and every internal caller now names its mode.
- **Compact bare replies.** Inventory, Kitchen and Finance return a short mode list on a bare call, down from roughly 100k, 65k and a few thousand characters. The full catalogue is for `mode=help`.
- **Manifest publish cadence.** Publishing runs on the events a reload itself fires, writes only when content changed, and cannot loop on its own events.
- **A disabled component leaves the prompt.** A component with `meta.enabled: false` used to stay in the agent's prompt, marked off. Now it drops out, as it already did from SuperSummary; its last kata is kept and it returns when it's turned back on.
- **Kung Fu is KF6.** KF4 (the authored drawer) and KF5 (the self-registering tool) roll into one spec, with one contract for both and the pipeline's security rules written in: a component runs as the household's onboarded agent, seeds only from modes a tool declares, acts only with a background grant, and reaches the Dojo only through Scribe or `bootstrap_kfc`. A component's minimum level (`meta.requires`) is enforced (see Added). See [Building a KFC](../kung_fu/building_a_kfc.md) and [Understanding KF6](../kung_fu/understanding_kf6.md).
- **The KungFu Loader is retired.** Every component self-registers through KF6's self-registering path (each tool's own `mode=kfc_manifest`, registered by `bootstrap_kfc` on reload and daily), so a prompt load now only loads the prompt and the `ship_zen_system` option is gone. The manifest whitelists `<tool>:kfc_manifest`.
- **FileCabinet internals.** See [FileCabinet, Rebuilt to Be Lockable](#filecabinet-rebuilt-to-be-lockable). In-repo callers call `zen_sutra_filecabinet` directly, one script run instead of two; the wrapper still works. Expiry still stored inside an older value counts, and GC normalizes every drawer it moves. Writes through a mount chain land on the real target, where they used to stop at the first hop.
- **`prompt_loader` runs queued,** up to 25 at once, so a burst of registry calls waits its turn instead of being dropped. When `sim_mode_allowed` is unset, it stamps it from Flynn's `input_boolean.zenos_sim_mode_override` (on by default) instead of `false`, so a load right after `reset_all` still resolves its caller.
- **Identity `area_labels` holds only real HA areas,** so data-routing labels no longer show up in a "who" answer.
- **FileCabinet takes `mode` for the action,** like every other tool, and `write_mode` (`merge` or `replace`) for how a write lands. `action_type`, `merge` and the old `mode: merge|replace` still work, so existing callers are unchanged.
- **Every cert is under `zenos.*`,** with a one-release alias for the old flat names. See [Certs, Renamed and Readable](#certs-renamed-and-readable).
- **`expand_drawer` moves up to 200 keys** per root drawer, up from 30. A bigger tree is refused with the fix: move some children under a new root drawer (`mode=move`) and expand each root.
- **Cabinet labels are gated.** Labels needs `zenos.system.cabinet_admin` level 2 to tag or untag a cabinet, or to touch a `zen_*cabinet` label, so a second cabinet can't be tagged as the vault and knock the resolver offline.
- **Inventory area modes take `homeassistant_area_id`,** as the field always was; the description and quickstart said `area=`. A call with no area is refused with what it needs.
- **The agent, not a persona name,** in shipped help text, descriptions and cert text. Zammad's audit actor is the configured persona name, else `agent`.
- **The summarizer's seed and act whitelists become certs.** A seed is a mode the tool declares, and an act needs the target tool's emit cert held with the Ninja emit stage. The lists keep working through 11 as a bridge. See [The Pipeline Gets a Pass](#the-pipeline-gets-a-pass).
- **`friendly_name_regex` outlives `friendly_name`.** Once HA stops setting the attribute, it matches the device name + entity name. While HA sets it, matches are unchanged.
- **`reset_labels` is gated** by `zenos.registry.label_lifecycle_control` at level 2 and a live household-admin yes on every call. It had only `confirm`.
- **Tool descriptions are plain ASCII.** Every script and field description in the tools and plugins swaps its em-dashes, arrows, curly quotes and symbols for plain characters. The text means the same, and new tools follow the same rule.
- **Inspect fetches lens evidence only on request.** A drawer peek no longer calls lens providers on its own. Pass `+lenses` or `lenses=` to get evidence. Fetched evidence is merged by item across anchors and providers, and `providers` carries status and counts, not a second copy of the documents.

## Fixed

- **Scribe `enable` and `disable` write again.** Both always answered `missing_required`.
- **Event emitter refuses internal event kinds.** The agent-facing `zen_dojotools_event_emitter` passed any `kind` through to the event bus, and several listeners act on kinds: scheduled reloads, tool dispatch, approval acknowledgements, prompt injection, garbage collection, and cabinet, identity, alert and label signals. Those kinds are now reserved and refused with a `guarded:` error. Internal callers fire their own events directly. Free-form kinds and the summary and ninja forces still work.
- **The help proxy only reaches ZenOS tools.** `zen_dojotools_help` took any script name and called it with `mode=help`, which runs the default path of a script that has no help handler. It now proxies only existing `zen_dojotools_`, `zen_stack_`, `zen_sutra_` and `zen_codex_` tools and refuses anything else with a `guarded:` error.
- **Cabinet schema upgrade now runs.** SystemTools sent the cabinet admin tool an operation field it does not read, so the upgrade never started.
- **ZenZork `examine` returns lens results** instead of help.
- **Event emitter no longer fails when `severity` is omitted.**
- **Trapper Keeper has real input.** It was seeded with nothing and reported a false telemetry blackout. The Ninja summarizer also no longer calls the model when there is nothing to read, and it unwraps the index result envelope so the inspect supplement receives data.
- **A provider whose `kfc_manifest` answers `result: ok` with the definition elsewhere** handed FileCabinet's live-mount read the string "ok". `result` must carry the definition; AutoVac and Taskmaster now do.
- **Home Mode anchors must be strictly ascending.** Six equal anchors, which is what a fresh install reports, used to pass the check and hold the house in Night-Late all day. They now fall back to the default schedule and raise `home_mode_anchor_misconfigured`.
- **Security Manager refreshes after a restart.** Its reading now also runs on Home Assistant start and on home mode updates, with a 60 second settle, so a reading taken while the panel was still catching up no longer sits at the top of the summary until midnight.
- **Cabinet admin refusals hold.** A cert denial, a missing `hammer_ok`, or a refused live acknowledgement set an error and then fell through to the operation, so the operation ran anyway: a hammer without `hammer_ok` wiped the cabinet. Everything after the guards now runs only when no refusal was set, and `reset` asks for the same per-call live household-admin acknowledgement as hammer.
- **Identity remove modes find the member's real list.** A family removed without `member_type` stayed put while the call reported success.
- **Import recursion in `zen_identity.jinja`.** The header's usage example ran a real import of the file, so any import recursed. The header is now a Jinja comment.
- **Flynn log noise.** `flynn_unified_engine` defaults `value` and `action_type` in the template instead of logging `'value' is undefined` on every stepgate pass.
- **The camera heartbeat runs again.** Its old drawer matched no label, so the Ninja empty-input guard skipped it. It is now a KF5 component of the camera tool, enabled only while cameras carry the `exterior` or `interior` label.
- **FileCabinet's `write` alias meets the write guards.** A direct engine call with `action: write` skipped the system cabinet, read-only, protected drawer, reserved key and health guards. Aliases now resolve before any guard.
- **`move` onto its own key refuses** with `code: same_key` instead of writing the drawer onto itself and deleting it.
- **Identity VolumeInfo writes touch only the changed keys,** so an empty read can no longer replace the whole drawer.
- **Flynn's history mount** reads the mount pointer from the drawer's value and stops re-sending `set_mount` on every bootstrap; a failed mount logs a warning. Flynn's primary-user picker now hides the head of household.
- **Labels follow drawers.** The label index stays in the cabinet you address, so an expanded drawer keeps its labels, and move, copy and delete carry a tree's labels with it. A cross-cabinet copy no longer replaces the destination's whole label index; delete no longer leaves its children's labels behind; relabel and create answer only after the index write lands, so two label writes in a row both stick; and a label search finds nested (`a/b`) drawers.
- **Index paging covers an explicit `limit`.** `truncated` and `next_offset` report what is left past the page whatever cut it, and the last page of a big set is no longer flagged as capped with a `next_offset` past the end.
- **Lens dispatch keeps both anchors** when a provider returns one item through two of them, instead of dropping the second.
- **`expand_drawer` keeps everything it moves.** It used to overwrite the slot's VolumeInfo (losing its id, ACLs and flags), copy only a drawer's value (dropping its title, labels, meta and ACL), and leave the moved drawer under the slot's default rules. Now the slot's VolumeInfo is merged, the whole drawer moves, and the expansion answers with the host's ACL.
- **Scribe writes only its own artifacts,** and Inspect never dumps a whole cabinet; its drawer peek follows FileCabinet's read rule.
- **`query` without `filter_json` no longer crashes,** including calls that pass `domain`, `label` or `areas` at the top level.
- **Music Assistant `get_library` without a search** no longer sends `search: None` and gets a 400; a missing Music Assistant entry or a failed call answers `error`.
- **Infra survives update entities with no title,** which had taken down the tool scan, the manifest, Identity's `cert_list` and the KFC bootstrap with it.
- **Ansible's `omit` is gone from service calls.** It rendered as an empty string, so Lights sent `color_temp_kelvin: ''`, Ectoplasm's `repair_list` sent `domain: ''`, and Identity's `cert_list` sent `target: ''`.
- **Live-ack alerts clear** as soon as they are answered or time out, instead of staying active for a day.
- **Grocy by-name stock adds work** for a product whose stock unit key is absent; only a key that is present and null is refused.
- **The cert-denial hint names the real mode,** `cert_req_grant`.
- **Zammad agrees with itself about configuration:** both readers unwrap `integrations_config` stored as JSON text or wrapped in `value`.
- **CertAdmin's live-ack prompt** falls back to the cert name when the catalog description is empty.
- **Mealie `foods_audit` reads every page** instead of stopping silently at 1,000 foods, and reports `total_available`, `pages_read` and `truncated`. Its advanced GET joins an endpoint's own query string with `&`.
- **Twenty REST calls** strip a leading `/` or `/rest/` from the endpoint.
- **ZenOS no longer triggers Home Assistant's `config_entries` deprecation warning.** Inspect, Music Assistant and the target resolver read a device's `config_entry_id` (HA removes `config_entries` in 2027.10).
- **Help matches the code.** SystemTools answers an unknown mode with a `guarded:` error pointing at `mode=help` and `zen_health_report`; Labels' `tag`/`untag` need no `confirm`; Postman's family-policy recipe no longer suggests seeding `{}`.
- **A refusal stops the call.** Branches that ran after a refusal or cert denial, overwriting it and doing their side effect, now wait on the result in Room Manager, Labels (a refused `tag` used to tag anyway), Lights, Media Manager, Climate, Scribe, Display, Plant, Finance, Manifest, Covers, Service Desk, SystemTools, Timekeeper, Calendar, Todo, Library, Spa Master, Camera and Infra.
- **Autovac's cert check holds.** `pause`, `unpause`, `queue`, `dequeue`, `clear`, `skip`, `unskip`, `ready`, `unready` and `mark_cleaned` wrote their state after `zenos.autovac.operate` denied: a denied pause answered ok and paused.
- **FileCabinet GC's side effects run only on a real run.** Priority-inject eviction, the active-alert clear, Postman log aging and the kata broadcast ran on any `gc` call, a refused one included.
- **Taskmaster checks before it registers.** `register` and `unregister` ran ahead of every check.
- **The scheduler's subscriber delays no longer add up.** A delay is an offset from the trigger, but the loop slept each one in turn, holding the queued scheduler for minutes on a busy install, deferred script reloads included. Each subscriber now sleeps only the gap since the previous one.
- **Flynn's notices stay up.** `flynn_active_notification` was wiped on every Sentinel pass, so each notice was raised again every pass and Gate 4's note lasted one pass. Only the gate whose notice it is clears it now.
- **Cabinet factory waits for the GUID** (up to 10 seconds) and answers an error instead of a false "initialized".
- **Provisioner's deprovision removes every type label** as a list, where it crashed on joined label names, and keeps `zen_cabinet`, so a freed slot stays a slot.
- **Taskmaster's almanac works on a busy calendar.** It copied the whole calendar response into one variable and went past Home Assistant's 262,144-character template limit on every call. It keeps four fields per event now.
- **An entity without a `friendly_name` still exists.** Inspect used the attribute as its existence test, so an entity HA had stopped naming that way read `unknown` and lost its attributes. Existence comes from the state object now.
- **Device names are the ones you set.** Every tool read a device's name with `device_attr(..., 'name')`, which is the integration's name, so a device you renamed still showed the old one. They read `device_name()` now.
- **Inspect no longer calls a hub a parent.** The device tree's `parent_name` held the name of the hub HA reaches the device through. That is `via_device_name` now, and a real parent device is reported separately.
- **Battery health reads its inventory seed.** Its seed, Inventory's `battery_status`, wasn't on the seed list, so every run fell back to the index. Inventory declares the mode now.
- **Calendar finds a calendar by its name.** A `calendar_name` given as a display name never resolved: the match was set inside a loop and never left it, so only entity ids worked. It matches the entity name, the device + entity name, or the legacy friendly_name.
- **Calendar start and end take a space-separated datetime** (`2026-11-14 18:00:00`), not only a `T`.
- **FileCabinet takes a list `volume_entity_id`.** An MCP client sends an array as a JSON list string, which crashed the wrapper on every call, help included. The wrapper and the engine now read a list or a JSON list string as its first entry, for `destination_volume_entity_id` too.
- **Grocy reads the area you pass.** `area_id_norm` read Home Assistant's `area_id()` function when no area was passed, so every area mode called without one used it as the area. `chores_by_area` and `room_brief` read the area like every other area mode.
- **Help's tool directory lists every tool.** It read each manifest without unwrapping its envelope, and found 9 tools of 66. It also dropped any tool whose manifest didn't name `help` among its modes, which hit most stacks, though help is every `zen_*` script's default. Entries now carry `tier` and `returns`.
- **Security Manager's envelope names the mode** for an `action=` call answered before dispatch; it reported `mode: ""`.
- **zen_system stops calling a summary critically stale too early.** Its KFC told the model 15 and 30 minutes while the health check uses 25 and 45, so one skipped SuperSummary run read as critical. The KFC uses 25 and 45 now.
- **Templates seed on a fresh install.** The template reset wrote with an update, which does nothing on a drawer that doesn't exist, so a new install never got `zen_template`, `kata_template` or `kfc_template`, and Flynn retried every pass. The reset creates them now.
- **The cabinet-upgrade alert means it.** `cab_schema_upgrade_available` was raised for any cabinet warning, a vault not yet written included, with wording about a pre-RC2 schema. It is raised only for a cabinet on that schema now, and it clears when the system comes up all green.
- **Autotag reports what landed.** It counted a tool as tagged whether or not the label stuck. It reads the labels back now: what didn't land is listed in `failed`, naming any label that doesn't exist, and the run answers `partial`. `tier=` limits the run to one tier, as its help says; it used to tag every tier.
- **SuperSummary survives a model failure.** An error, an outage or an empty reply from the model ends the run with `monk_failure` and an event, as the Ninja already did, and the last good summary stays in place.
- **The Ninja's context cap works.** The guard measured the prompt's keys, not its length, so it never tripped. It measures characters now. A component over `max_context_tokens` stops with `ninja_context_overflow` and a hint: raise `max_context_tokens` in `zen_ninja_config`, or lower `domain_inspect_limit`. The summarizers' `prompt_size_chars` counts characters too.
- **Calendar makes one write.** An MS365 create also ran the generic create, and an MS365 update also deleted and recreated the event. A create or update now writes once, and the delete-and-recreate update for other calendars runs only with both start and end, after its delete.
- **Documents and tickets answer subject lookups.** Paperless and service-desk tickets were registered on the Lens Bus but never returned anything. They do now: documents by their tags, and tickets tagged with an area or label. An unconfigured source shows as unavailable instead of an empty answer, and catalog items come back with their titles and only the room they are in. See the [Lens Bus](../library/lens_bus.md).
- **Media previews and refusals change nothing.** `prefs_apply` with `dry_run=true` still sent the source and volume to the device, and `activity_apply` kept going after refusing a Paused room or a missing activity. Both stop now, and a preference skips a step the device can't do instead of failing on it.
- **A refused catalog change stops.** A catalog move, loan, return or update that was refused (no match, several matches, a missing borrower) still wrote, so an unclear loan lent the first copy and reported success. A refusal now writes nothing, and success means the inventory confirmed it.
- **Calendar keeps your dates.** An update that changed only the summary moved the event; it keeps its dates now. A date alone (`YYYY-MM-DD`) makes an all-day event. An omitted end follows the start (one day all-day, one hour timed); it was always tomorrow, so a future event without an end was refused. A date that doesn't parse is refused before anything is written.
- **Auto-arm can't be called from a conversation.** Security Manager's `auto_arm_check` is refused from a user or agent session; only the auto-arm automations call it. A caller could name an unlocked lock and disarm an armed-home panel with no cert or acknowledgement.
- **Auto-arm no longer treats a lost presence reading as an empty house.** When `zone.home` went unknown or unavailable it read as zero people, so auto-arm could run the Away countdown with people home. An unreadable count now arms nothing and runs no arrival path, and a template reload no longer looks like entering Night-Late or Away. Auto-arm is off unless you turned it on.
- **Turning a profile's tracking off sticks.** Profile Editor wrote it back on.
- **Identity finds nested family members.** `is_member` missed a member of a family inside a family.
- **The cabinet repair selector lists its options again.**
- **Spa history works.** The 7-day baseline query sends its window and types, and history metrics come back as mappings, not strings. Its climate, cover and camera name fallbacks look only in their own domain instead of walking every entity in Home Assistant.
- **Finance survives a gateway error.** A reply that isn't JSON, such as an HTML error page, is a structured error instead of a script failure. Transaction search by `tags=` keeps the tags.
- **Mealie builds its ingredient-to-recipe map.** It came back empty.
- **Media Manager's room match counts.** A search hit that matches the room's provider now raises its confidence.
- **A live drawer's digest returns only the fields it names.** Each field held the whole response.
- **FileCabinet copy is honest and complete.** A copy whose destination never confirms answers `write_unverified` and stops before the labels and children. A cross-cabinet copy writes its children into the destination; they went back into the source.
- **Copying or moving a drawer keeps its access rules and expiry,** for the drawer and every child. A copy or move never declassifies a drawer.
- **Weave can't read a drawer you can't.** `weave` and `weave_preview` check read access on every source drawer and nested definition, as a `get` does. A denied source is logged, and under enforce it is never read.
- **A live drawer names one tool.** A drawer whose `tool` and `action_type` named two different scripts could pass the whitelist as one and run as the other. It is refused now.
- **The event emitter refuses `room_control_request`.** An agent could fire it to change a room's control mode without `room_control_set`'s certs.
- **Rooms no longer lose each other's timer state when several change at once.** Each room keeps its timer class under its own key.
- **Busy fan rooms no longer crowd out other room events.** Vent fan waits run in their own automation, so a burst can't fill the dispatcher and make Home Assistant drop room signals, timers or emergency arming.
- **Releasing an ensuite no longer briefly darkens its parent room.** The 8 second child bridge never matched, and the parent now re-renders when it ends.
- **An offline child room no longer holds its parent Occupied.** An unknown, unavailable or missing child counts for nothing.
- **A borrowed scene resolves.** A room in engaged, asleep or hold that borrows another state's scene never found it, because comment text sat inside the template.
- **A fast A-to-B-to-A no longer fires the scene twice.**
- **Room state updates no longer scan every sensor in the house.** A commissioned room finds its own state sensor from its labels, and Home Overview builds its room list from the `zen_room_state` label.
- **Rooms controlled only through the assistant release Automation after the burnout time.** A stale expiry clears an override only while it is still Automation.
- **Health sensors read a malformed timestamp as stale** instead of erroring, and a timestamp without a timezone as UTC.
- **Smaller fixes.** A re-fired alert's answer replaces the cached one. Provisioner's deprovision reports an error when the cabinet doesn't end up unmounted. `flip_schema_version` flips back. Stack and lens unregisters remove their entry.

## Removed

- **`maint_4_5_6.yaml` and `repair_fam_cab_volumeinfo.yaml`.** `acl_migrate` covers both, for every cabinet, and Flynn's bootstrap runs it.
- **The 4.5.6 `run_repair` actions** and the cabinet factory's `family_guid` field.
- **The CABS `cabinet_acls` and `drawer_acl` macros.** Use `acl_record` and `acl_for`.
- **Cabinet admin's `expand_drawer`,** with its `source_cabinet`, `source_drawer` and `space_threshold` fields. It moved one drawer without making the slot an expansion. FileCabinet's `expand_drawer` and `collapse_drawer` are the only way to make and undo one.
