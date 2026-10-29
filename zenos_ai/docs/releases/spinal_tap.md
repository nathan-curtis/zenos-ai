# Release Notes — 2026.11.0 'This Is Spinal Tap'

**Status:** Public beta 3
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

Backstage isn't the stage. The pass gets the agent in to see the band; every instrument still takes its own cert. The certs are new too: each one now lives under `zenos.*`, carries a date, and asks you on your phone in plain words. And the guitar?

Don't touch it. Don't look at it. Don't EVEN point at it. Once cabinet enforcement is on, a cabinet your agent has no pass for won't just refuse the read. It won't show up in the listing.

**Backstage passes for agents, locks on the cabinets, and a working screen for the show.**

---

## Planned Scope

**The goal:** no unadmitted agent reaches a protected drawer, while System bootstrap and recovery keep working.

**The chain:** Postman approves, CertAdmin admits, the enclave stores credentials, `zen_root_authentik` authenticates, Identity authorizes, and FileCabinet enforces. Where a call came from is recorded for audit and is never authority.

1. **Access-path audit.** Every way into a protected drawer is mapped before any enforcement lands: tool calls, mounts, live drawers, boot and repair paths.
2. **FileCabinet security.** One enforcement point. Read, write, protected configuration and cabinet administration are separate permissions. They cover listing, search, copies, moves, mounts, expansion and live-drawer dispatch. Authorization follows the resolved destination, and a cross-cabinet operation checks both sides. Following a mount confers no authority, so mount resolution happens before the access check. Callers see only what they may read: a hidden drawer is indistinguishable from one that does not exist. You can't touch the guitar, and you can't point at it either. Effective inherited access can be shown per drawer.
3. **Trusted entry points.** System and Default Agent authority are separated. Flynn and the prompt loader are named trusted entry points, and there is no agent-selectable "system" bypass.
4. **Admission enforcement.** `zenos.agent.basic` stops being a badge that is requested and becomes one that is checked. In 11.0 the check is the onboarding gate: the backstage pass, checked by every agent-facing tool. Participation stays distinct from administrative authority. The admission-token contract with `zen_root_authentik`, and Postman approvals bound to the enrollment they were given for, are not in 11.0: they need an OIDC plugin first, and the gate is their precursor.
5. **Bootstrap and recovery.** Flynn's cold boot, upgrade, revocation and recovery run on a trusted path, and each is proven.
6. **Audit-only rollout.** Enforcement ships first as would-deny logging, with a rollback, then turns on.
7. **Operator permission report.** Shows the resolved identity, granted capabilities, effective cabinet access and the reason an operation was denied. Secrets are excluded.

**FileCabinet interface cleanup.** The `action_type` field moves to `mode=`, with merge and replace behavior in `write_mode=`, and a compatibility shim carries existing callers. It follows the audit, because the audit maps the same callers.

**Also planned**
- Every tool guards its inputs (the contract sweep begun in 2026.10.1).
- One configure contract across plugin setup.
- Identity and Labels event-ordering hardening.
- Display, finished: the shared dashboard ships, interruption awareness covers the remaining Media Manager watch paths, and expiry and dismissal are tested against the real scheduler cadence.

**Later phase:** binding to the conversation or session. It needs its own design and acceptance criteria before it joins scope. The admission-token contract follows the OIDC plugin.

Order of work: audit, designs, audit-only mode, enforcement.

---

## Scope of the Claim

FileCabinet enforcement protects the ZenOS access boundary. Home Assistant administrators, and any path with direct access to the underlying state, remain part of the trusted platform.

Identity in this release tells Flynn, the agent and the user apart. Telling one agent or one session from another is a later phase.

The backstage pass checks calls a user started. A call no user started (boot, Flynn, an automation, the scheduler) passes, so an automation that calls a ZenOS tool is trusted as far as whatever can trigger it.

---

## Where It Stands: the Amps Are Wired, the Volume Is at Ten

Every lock on the cabinets is built, wired to its door, and running in **audit mode**. The gate decides every FileCabinet call and every admin-door write, logs what it would have refused or asked a human about, and lets the call through. Out of the box, no cabinet call is refused. The rules the gate will enforce, the principals they name, and the migration that puts them on every cabinet are all in place, and the logs show what enforcement will change before it changes anything.

FileCabinet's gate now has the knob to eleven: one admin-door drawer, `fc_acl_mode`, switches it to enforce, and removing the drawer switches it back. It ships off: 11.0 does not enforce cabinet access by default, and the admin door's gate stays audit only.

One gate ships on: the backstage pass. An agent you haven't approved gets help and onboarding, and nothing else. See [The Backstage Pass](#the-backstage-pass).

| Planned | Landed (audit mode, unless noted) | Still to come |
|---|---|---|
| Access-path audit | The audit drove the default table and caller origin below. | Finishing the map of every path in. |
| FileCabinet security | One gate, checking every hop of a mount chain. Separate read, write, delete and admin rules. Default rules for every cabinet class, and locks on credential and approval drawers. Effective access per drawer (`CABS.acl_for`). An enforce switch, off by default. | Enforcement on by default. Hiding unreadable drawers from listings and search. |
| Trusted entry points | The gate tells an agent from a tool (caller origin). Boot, Flynn and automations are exempt at the admin door. Provisioning and Flynn's cabinet setup are named lifecycle principals. | Separating System and Default Agent authority. |
| Admission enforcement | The backstage pass: every agent-facing tool checks `zenos.agent.basic` first. It ships on, not in audit. Flynn asks for it when onboarding starts. | The admission-token contract with `zen_root_authentik`, after the OIDC plugin (a later phase). |
| Bootstrap and recovery | Flynn's bootstrap is built to migrate and stamp a fresh install. An upgrade migrates and stamps when the bootstrap next runs, or on `acl_migrate` / `cabinet_schema_upgrade`. | A cold fresh-install run with the stamp. Revocation and recovery on the trusted path. |
| Audit-only rollout | Shipping now: `fc_acl_audit` and `door_gate_audit`, and FileCabinet's opt-in enforce switch. | Enforcement by default, and at the admin door. |
| Operator permission report | | The report. |

---

## The Backstage Pass

Before 11.0, `zenos.agent.basic` was a badge an agent was given at the end of onboarding, and nothing checked it. Now it is the ticket to play.

**No pass, no tools.** Every agent-facing tool calls one gate first, `OS.entry_gate` in `zen_os_1.jinja`. An agent without the pass gets each tool's `help`, `tool_manifest` and `kfc_manifest`, and the few steps it needs to ask for the pass and help you fix whatever stops the ask from reaching you: onboarding, `whoami`, the cert request and the live-ack, alert, push and display hops it travels, and a health report trimmed to the approval path. Anything else answers `guarded: onboarding_required` with `next_step: zen_flynn_oobe mode=run`, and logs one `zen_entry_gate` warning. Calls no user started (boot, Flynn, automations, the scheduler) pass.

**Flynn hands it out first.** `zen_flynn_oobe mode=run` asks you to approve the agent (the `zenos_agent_basic` bundle, one push) before it hands out a single setup step. An agent that finished onboarding but was never approved gets one "waiting for approval" notice from Flynn, and answers as itself until you say yes.

**No onboarding, no pass, no breakage.** An unapproved agent can't do anything, so it can't break anything, and it still has enough to walk you through fixing the approval path. If the push can't reach you, the console override stands in, and now works before approval. Both routes are in the [Install Guide](../getting_started/install.md#step-35--set-up-your-mobile-notification-path) and the [Security & Certification manual](../getting_started/security_certification_manual.md#25-the-backstage-pass-zenosagentbasic).

**Home Assistant can't tell you from the agent.** A dashboard button, or your own call from Developer Tools, runs as a logged-in user, the same as the agent. Until the agent is approved, those calls are refused too.

**Putting the gate in found more than 20 tools where a refusal didn't stop the call.** In a single-exit tool, a branch that checked only the mode ran after a refusal, overwrote it, and did its side effect. One was a real cert bypass: a denied Autovac `pause` still paused. Every top-level branch now waits on the result. See Fixed.

---

## Certs, Renamed and Readable

**Every cert lives under `zenos.*`.** The 42 flat names moved to dotted ones: `lock_control` is `zenos.actuation.lock_control`, `todo_control` is `zenos.comms.todo_control`, and so on. For one release, a grant under either name satisfies a check against the other, and `cert_list` shows an old-name grant as held with `reissue_pending`. Reissuing it under the new name replaces the old grant and keeps its scope. The alias goes in 2026.12.

**Approvals read in plain words.** Every cert has a short `description` written for the phone, and a `detail` for the long form. The approval prompt reads "<agent> asks to: Lock any door and unlock interior doors (zenos.actuation.lock_control, level 1). Approve?" A bundle lists each member the same way. The catalog is in the [Security & Certification manual](../getting_started/security_certification_manual.md#3-the-certification-catalog).

**Grants are dated.** Every new grant records `not_before` and `not_after` (one year), in UTC, with the X.509 field names. Nothing enforces expiry yet.

**The vault holds the catalog.** The vault cabinet is now a required system cabinet that Flynn provisions on boot, and only the admin door can read it. `cert_list` reads the cert catalog from a cache there, so it answers in a fraction of the time, and builds it live whenever the vault isn't there. Grants are always read live.

---

## The Locks, in Audit Mode

**One record.** Identity and FileCabinet share one ACL record in each cabinet's VolumeInfo: `acls: {v: 2, owner, prime, partners, ops}`. Principals are cabinet GUIDs, never names. `ops` holds a rule for each of read, write, delete and admin: who may do it (`who`), the cert it needs, and whether a live human must say yes (`ack`). Identity writes the record, through the admin door. FileCabinet and Identity read it only through the CABS `acl_*` macros and `ID.principals`, so there is one reading of the rules, not two.

**Every cabinet has a class, and every class has defaults.** A cabinet is `system`, `vault`, `dojo`, `kata`, `history`, `household`, `family`, `user`, `ai_user` or `scratchpad`. Each class has a default row: the system cabinet is read-only to everything but the admin door, the vault takes writes only from Identity and is read only through the admin door, a person's cabinet answers to that person, their partners and the prime agent, and a scratchpad is open. A drawer may tighten or loosen its own ops; each op falls back on its own to the cabinet's rule and then the class default. The full table is in the [zenos_cabinets.jinja reference](../custom_templates/zenos_cabinets_jinja.md#acl-v2). The class is a stamp in VolumeInfo, which only the admin door can change. The gate never reads a label.

**Some drawers no owner can open.** Certs (`zen_ai_certs`), integration settings (`integrations_config`) and live-ack answers (`alert_response_*`) take writes only from tools, and the console override (`cert_admin_console_bypass`) and the enforce switch (`fc_acl_mode`) only from the admin door. These built-in locks win over every other rule, so `acl_set` can't loosen them.

**The gate knows who's asking.** The agent-facing FileCabinet tool marks every call it makes as an agent call. A tool calling the engine is internal, and can name itself with `caller_tool`, so a rule can say `@tool:zen_dojotools_identity` and mean exactly that. New principals in `who`: `@internal`, `@tool:<script>`, `@self`, `@partner`. Before this, every FileCabinet call looked like it came from the default agent. Now the audit log tells an agent from a tool.

**Mounts confer no authority.** FileCabinet resolves the real target behind a mount chain (up to 8 hops; a loop refuses) and checks every hop, none skipped: each pointer the chain passes needs read, and the drawer it ends on needs the action's op. A caller that can't read a pointer doesn't get past it. A pointer answers with its cabinet's rules, never a drawer's. An expansion answers with its host cabinet's rules, so moving a drawer into one changes where it lives, not who can reach it.

**Two gates, both auditing.** FileCabinet's gate answers `acl_audit` and logs `fc_acl_audit` with the caller's `origin` and, per check, where the rule came from and how the caller matched. The admin door checks agent and console writes against `zenos.system.cabinet_admin` and answers `gate_audit`. Boot, Flynn and automations are exempt.

**One switch to enforce.** The system cabinet's `fc_acl_mode` drawer, `{mode: audit|enforce}`, sets FileCabinet's gate. Missing means audit. In enforce, a deny is refused with `guarded:`, and a rule that asks for a live ack asks the household admin every time; not granted is `denied`. Only the admin door writes the drawer, so no agent can flip it, and removing it is the rollback. `acl_audit.mode` on every answer says which applied. The how-to is in the [FileCabinet readme](../scripts/zen_dojotools_filecabinet_readme.md#access-control-acl-v2).

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

## Read this before upgrading

- **Approve your agent.** An agent that never received `zenos.agent.basic` loses every ZenOS tool except help and onboarding until you approve it: run `zen_flynn_oobe mode=run` (or ask the agent to) and say yes to the push. An agent you approved at the end of onboarding in 2026.10 already holds it. Until then, dashboard buttons and your own Developer Tools calls to ZenOS tools are refused too. The push goes to `notify.admin_devices` until Postman has a profile; if no push can reach you, use the console override ([Install Guide](../getting_started/install.md#if-no-push-can-reach-you)).
- **The ACL v2 migration is mandatory.** Identity setup refuses an old-schema household or family with `acl_schema_upgrade_required`. The migration also stamps each cabinet's class; until it does, a cabinet already on v2 gets the unclassified defaults. After reloading (templates, automations, template entities, then scripts), run `zen_dojotools_identity mode=acl_migrate` to see the plan (a dry run is the default). Check every proposed cabinet `class`, especially any it calls `scratchpad`. Then apply it with `dry_run: false` and `confirm_action: true`, or run `zen_dojotools_systemtools tool=cabinet_schema_upgrade`, which applies it when the schema stamp is behind. A second dry run should plan nothing, and the system cabinet then holds `acl_schema` version 3. Flynn's bootstrap runs the same migration on its own until the stamp is current: on a fresh install, and on an existing one the next time the bootstrap runs. Drawers need no migration of their own; existing ones read as before.
- **Certs are renamed, and you have until 2026.12.** Nothing to do now: an old-name grant keeps working and shows `reissue_pending` in `cert_list`. Reissue it under its `zenos.*` name before 2026.12. Re-granting `zenos_agent_basic` reissues every member at once.
- **Re-grant `zenos_agent_basic` if to-do writes fail.** `zenos.comms.todo_control` level 1 (then `todo_control`) joined the bundle in 2026.10.1, but an agent that was granted the bundle's certs before then doesn't hold it. Re-grant `zenos_agent_basic`, or grant `zenos.comms.todo_control` level 1, before to-do writes work.
- **Enforcement is opt-in.** An upgraded install stays in audit until an admin sets `fc_acl_mode` to enforce. Read the `fc_acl_audit` lines first: they are what enforce would refuse or ask about.
- **Expect audit lines.** Agents that don't hold `zenos.system.cabinet_admin` log `door_gate_audit` lines, and the FileCabinet gate logs `fc_acl_audit` wherever enforcement would have refused or asked. They are the preview of enforcement, not errors.
- **Cabinet labels need `zenos.system.cabinet_admin` level 2.** Tagging or untagging a cabinet, or creating, changing or deleting a `zen_*cabinet` label, is refused without it. Automations, the scheduler and Flynn are not affected.
- **The vault cabinet provisions itself.** `sensor.zen_cabinet_health` warns until the vault is online; Flynn provisions it on the first pass after warmup. An install whose vault never left `unknown` (every vault write was dropped before this release) gets it initialized on that pass too.
- **FileCabinet reads return content in `value`.** The `{path: value}` map that `read` put in `result` is retired; `result` is only the tool envelope, and `read` is a plain alias of `get`. Anything of your own that reads `result.<path>` needs to read `value`.
- **FileCabinet writes are honest.** A create, update, upsert or delete the cabinet never confirmed answers `status: error`, `code: write_unverified` instead of success. `update` on a missing key answers `not_found` (`force_action` creates it), and `create` on an existing key is an error unless `force_action`.
- **Every write is a merge or a replace.** `upsert` deep-merges by default and `update` replaces. A read-modify-write caller passes `write_mode: replace`; under a merge, a key you removed comes back.
- **Structural cabinet keys belong to the admin door.** VolumeInfo, `meta`, `_label_index`, `_zen_relationships`, the system cabinet and the vault are written only through `zen_admintools_cabinet_writer`, and FileCabinet refuses them. Raw `set_variable_*` events of your own aimed at those keys need to move to the door.
- **The lens and codex registries have one writer each:** `prompt_loader mode=lens_registry` and `mode=codex_registry`. Code writing those drawers directly needs to go through them.
- **`run_repair` has one action, `acl_migrate`.** The 4.5.6 repair names are refused, and `maint_4_5_6.yaml` and `repair_fam_cab_volumeinfo.yaml` are gone. The cabinet factory no longer takes `family_guid`.
- **`reset_all` wipes.** Cabinet admin's `reset_all` did nothing while answering success; it now wipes every cabinet. A wipe clears the console override and every cert along with everything else, so throw the override from the console and re-request certs afterwards.
- **If `script.zen_admintools_kungfu_loader` shows unavailable,** delete its entity registry row. The loader was retired in this release.
- **Your own copy of the scheduler loop** needs the same fix as the scheduler (see Fixed): sleep the gap since the previous subscriber, not each subscriber's whole delay.

---

## Added

- **Semantic roots.** The tool manifest publishes a root for each question word: who (Identity), what (Index), when (Taskmaster almanac), where (Room Manager), why (history), how (help) and knowledge (FileCabinet). Each root answers what matters now and names where to go deeper, and the boot prompt renders them as one line.
- **Taskmaster almanac, the "when" root.** One call returns a headline and the current period of the day, vacation, events from calendars carrying the `almanac` label (timed events first, at most two per calendar), tasks due by today, meals and weather. `horizon_hours`, `lookahead_days` and `max_items` tune it.
- **Per-tool roots on the core tools.** Room Manager, Index, FileCabinet, Identity and Taskmaster declare their own roots in their `tool_manifest` modes. A bare call returns them in about 350 characters.
- **Expansion slots answer with their host's ACL.** A slot an earlier `expand_drawer` stamped now answers with the ACL of the cabinet that hosts it. The class stamped on the slot itself no longer applies.
- **Cortex 45.** Cortex 44 'Open Door' with one fix: its `core_tools` list named a log viewer that does not exist, and now names `zen_dojotools_ha_log_viewer`. 45 is the new latest and replaces the v42 slot, so the selectable versions are 45, 44 and 43.
- **Who knows where a member belongs.** `identity resolve` returns `name`, `person_entity`, `member_type` and `memberships`, and finds a person anywhere in the household tree, not only among the household's own users.
- **The ACL record, cabinet classes, built-in locks, caller origin, both audit gates, and Identity's `acl_migrate`, `acl_set` and `take_ownership`.** See [The Locks, in Audit Mode](#the-locks-in-audit-mode).
- **FileCabinet GC `repair` mode.** Normalizes one drawer to the `{value, meta}` envelope, with `dry_run`.
- **Inspect lists the lenses that apply.** Every inspection carries `domain_context.lens_options`: each lens provider that matches one of its anchors, what it returns and whether it is available, built from the registry with no provider calls. `+lenses` fetches all of them; `lenses=<provider,...>` fetches the ones you name.
- **Scribe asks instead of refusing.** A live KFC write by a caller without `zenos.knowledge.kfc_publish` asks the household admin to approve that one write; declined or timed out, it is refused as before.
- **The lens registry can be expanded.** Lens register and unregister go through the central lens writer and read through the pointer, so `lens_registry`, one of the household cabinet's biggest drawers, can move to an expansion without being wiped.
- **FileCabinet `collapse_drawer`.** Undoes an expansion: the tree comes back to its cabinet, verified, and the slot is freed.
- **Cabinet capacity.** `sensor.zen_cabinet_health` gains a `cabinet_capacity` attribute: each cabinet's size as a percentage of the 128 KB cap, the fullest cabinet, and a warning list at 90% and over. The sensor's state is unchanged. `zen_health_report` adds a `cabinet near full` warning for each one, naming `expand_drawer` and `expansion_sitrep`.
- **The backstage pass.** Every agent-facing tool checks `zenos.agent.basic` first. See [The Backstage Pass](#the-backstage-pass).
- **Approval first.** `zen_flynn_oobe mode=run` asks for the `zenos_agent_basic` bundle before the setup protocol; `complete` answers `admission_pending` while the pass is still missing; `status` reports `badge_held`. Flynn's Sentinel posts one "waiting for approval" notice for an onboarded agent without the pass, and clears it once the pass is granted.
- **Views before approval.** Identity `whoami` shows an unapproved agent only itself (name, cabinet, onboarded, `has_basic: false`) and the next step. `zen_health_report` covers only the approval path; the full report names the house's areas and devices.
- **Plain cert descriptions and approval prompts, dated grants, and the cert catalog cached in the vault.** See [Certs, Renamed and Readable](#certs-renamed-and-readable). `cert_list` gains `catalog_source` and `force_refresh`; the scheduler's script reloads and Flynn keep the cache fresh.
- **The vault is a required system cabinet.** Flynn labels and initializes it on boot. `sensor.zen_vault_cabinet_resolved` resolves only when exactly one cabinet carries `zen_vault_cabinet`. Its read rule is the admin door: Cabinet Writer's new `get` mode is the one way to read it.

## Changed

- **Help is the default.** A `zen_*` script answers help for a missing or unknown mode, and a bare call (no mode and no input) never acts, crashes or errors. Tools that used to act on a bare call (announce, the summarizers, the prompt loader, the reset and repair tools, Camera, Provisioner and FileCabinet GC) now return help. Calls that pass input are unchanged, and every internal caller now names its mode.
- **Compact bare replies.** Inventory, Kitchen and Finance return a short mode list on a bare call, down from roughly 100k, 65k and a few thousand characters. The full catalogue is for `mode=help`.
- **Manifest publish cadence.** Publishing runs on the events a reload itself fires, writes only when content changed, and cannot loop on its own events.
- **The KungFu Loader is retired.** Every component self-registers through KF5 (each tool's own `mode=kfc_manifest`, registered by `bootstrap_kfc` on reload and daily), so a prompt load now only loads the prompt and the `ship_zen_system` option is gone. The manifest whitelists `<tool>:kfc_manifest`.
- **FileCabinet internals.** See [FileCabinet, Rebuilt to Be Lockable](#filecabinet-rebuilt-to-be-lockable). In-repo callers call `zen_sutra_filecabinet` directly, one script run instead of two; the wrapper still works. Expiry still stored inside an older value counts, and GC normalizes every drawer it moves. Writes through a mount chain land on the real target, where they used to stop at the first hop.
- **`prompt_loader` runs queued,** up to 25 at once, so a burst of registry calls waits its turn instead of being dropped. When `sim_mode_allowed` is unset, it stamps it from Flynn's `input_boolean.zenos_sim_mode_override` (on by default) instead of `false`, so a load right after `reset_all` still resolves its caller.
- **Identity `area_labels` holds only real HA areas,** so data-routing labels no longer show up in a "who" answer.
- **FileCabinet takes `mode` for the action,** like every other tool, and `write_mode` (`merge` or `replace`) for how a write lands. `action_type`, `merge` and the old `mode: merge|replace` still work, so existing callers are unchanged.
- **Every cert is under `zenos.*`,** with a one-release alias for the old flat names. See [Certs, Renamed and Readable](#certs-renamed-and-readable).
- **`expand_drawer` moves up to 200 keys** per root drawer, up from 30. A bigger tree is refused with the fix: move some children under a new root drawer (`mode=move`) and expand each root.
- **Cabinet labels are gated.** Labels needs `zenos.system.cabinet_admin` level 2 to tag or untag a cabinet, or to touch a `zen_*cabinet` label, so a second cabinet can't be tagged as the vault and knock the resolver offline.
- **Inventory area modes take `homeassistant_area_id`,** as the field always was; the description and quickstart said `area=`. A call with no area is refused with what it needs.
- **The agent, not a persona name,** in shipped help text, descriptions and cert text. Zammad's audit actor is the configured persona name, else `agent`.
- **Inspect fetches lens evidence only on request.** A drawer peek no longer calls lens providers on its own. Pass `+lenses` or `lenses=` to get evidence. Fetched evidence is merged by item across anchors and providers, and `providers` carries status and counts, not a second copy of the documents.

## Fixed

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
- **Flynn's onboarding lands on a fresh install.** Onboarding state is written as Flynn, not through the agent's tool, and `answer_clear` empties the answers instead of keeping them.
- **The FileCabinet gate names the cabinet** on `expand_drawer` and `collapse_drawer` refusals, live-ack prompts and audit lines; it printed an empty volume. The tool manifest reports FileCabinet's real version.
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
- **Inspect reads `config_entry_id`** before the deprecated `config_entries` device field.
- **Help matches the code.** SystemTools answers an unknown mode with a `guarded:` error pointing at `mode=help` and `zen_health_report`; Labels' `tag`/`untag` need no `confirm`; Postman's family-policy recipe no longer suggests seeding `{}`.
- **A refusal stops the call.** Branches that ran after a refusal or cert denial, overwriting it and doing their side effect, now wait on the result in Room Manager, Labels (a refused `tag` used to tag anyway), Lights, Media Manager, Climate, Scribe, Display, Plant, Finance, Manifest, Covers, Service Desk, SystemTools, Timekeeper, Calendar, Todo, Library, Spa Master, Camera and Infra.
- **Autovac's cert check holds.** `pause`, `unpause`, `queue`, `dequeue`, `clear`, `skip`, `unskip`, `ready`, `unready` and `mark_cleaned` wrote their state after `zenos.autovac.operate` denied: a denied pause answered ok and paused.
- **FileCabinet GC's side effects run only on a real run.** Priority-inject eviction, the active-alert clear, Postman log aging and the kata broadcast ran on any `gc` call, a refused one included.
- **Taskmaster checks before it registers.** `register` and `unregister` ran ahead of every check.
- **The scheduler's subscriber delays no longer add up.** A delay is an offset from the trigger, but the loop slept each one in turn, holding the queued scheduler for minutes on a busy install, deferred script reloads included. Each subscriber now sleeps only the gap since the previous one.
- **Flynn's notices stay up.** `flynn_active_notification` was wiped on every Sentinel pass, so each notice was raised again every pass and Gate 4's note lasted one pass. Only the gate whose notice it is clears it now.
- **The console override works before approval.** `prompt_loader cert_bypass` read the console user from `whoami`, which shows nothing before the agent is approved, so it couldn't be switched on in the one state it exists for. It reads the session now.
- **The vault takes writes.** The volume redirector knew the vault but had no route for it, so every vault write was dropped. Flynn also inits a vault that was never written.
- **Cabinet factory waits for the GUID** (up to 10 seconds) and answers an error instead of a false "initialized".
- **Provisioner's deprovision removes every type label** as a list, where it crashed on joined label names, and keeps `zen_cabinet`, so a freed slot stays a slot.
- **Taskmaster's almanac works on a busy calendar.** It copied the whole calendar response into one variable and went past Home Assistant's 262,144-character template limit on every call. It keeps four fields per event now.
- **Calendar start and end take a space-separated datetime** (`2026-11-14 18:00:00`), not only a `T`.
- **FileCabinet takes a list `volume_entity_id`.** An MCP client sends an array as a JSON list string, which crashed the wrapper on every call, help included.
- **The cert catalog has no gaps.** `cert_audit` fills short-form cert entries with their gates, description and `max_level`, so an approval prompt no longer reads "no description on file". A grant no longer logs two "Disallowed recursion" warnings.
- **Grocy reads the area you pass.** `area_id_norm` read Home Assistant's `area_id()` function when no area was passed, so every area mode called without one used it as the area. `chores_by_area` and `room_brief` read the area like every other area mode.
- **Help's tool directory lists every tool.** It read each manifest without unwrapping its envelope, and found 9 tools of 66. Entries now carry `tier` and `returns`.
- **Security Manager's envelope names the mode** for an `action=` call answered before dispatch; it reported `mode: ""`.
- **Versions agree.** Header and manifest disagreed on Locks, Infra, Taskmaster, Kitchen, Service Desk, Finance, Postman, AlertManager, Media Manager, Rolodex, Query and Help; Camera's help named an old version.
- **Scribe's live ack names its cert in plain words,** like every other approval; it named the old flat cert.
- **Smaller fixes.** A re-fired alert's answer replaces the cached one. Provisioner's deprovision reports an error when the cabinet doesn't end up unmounted. `flip_schema_version` flips back. Stack and lens unregisters remove their entry. Grocy, Lights, Media Manager, Room Manager and cabinet inspect read drawer content from `value`.

## Removed

- **`maint_4_5_6.yaml` and `repair_fam_cab_volumeinfo.yaml`.** `acl_migrate` covers both, for every cabinet, and Flynn's bootstrap runs it.
- **The 4.5.6 `run_repair` actions** and the cabinet factory's `family_guid` field.
- **The CABS `cabinet_acls` and `drawer_acl` macros.** Use `acl_record` and `acl_for`.
- **Cabinet admin's `expand_drawer`,** with its `source_cabinet`, `source_drawer` and `space_threshold` fields. It moved one drawer without making the slot an expansion. FileCabinet's `expand_drawer` and `collapse_drawer` are the only way to make and undo one.
