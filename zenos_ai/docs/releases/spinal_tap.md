# Release Notes — 2026.11.0 'This Is Spinal Tap'

**Status:** In development (beta)
**Branch:** `feat/2026.11.0`
**Base:** 2026.10.1 (patch on 2026.10.0 'Tron')

---

## Summary

Security goes to eleven.

Tron made admission explicit and put a badge in every agent's hands. This release makes the badge matter, and puts locks on the cabinets behind it. FileCabinet holds identity state, component definitions, configuration, mounts and the material that becomes agent context, and its live drawers can invoke tools. Permission to change a drawer can change what the system later believes or does, so the drawer is the next boundary to enforce.

The rule is the one the guitar's owner laid down: don't touch it, don't look at it, don't even point at it. A cabinet your agent has no pass for doesn't just refuse the read. It never shows up in the listing.

**Backstage passes for agents, locks on the cabinets, and a working screen for the show.**

---

## Planned Scope

**The goal:** no unadmitted agent reaches a protected drawer, while System bootstrap and recovery keep working.

**The chain:** Postman approves, CertAdmin admits, the enclave stores credentials, `zen_root_authentik` authenticates, Identity authorizes, and FileCabinet enforces. Where a call came from is recorded for audit and is never authority.

1. **Access-path audit.** Every way into a protected drawer is mapped before any enforcement lands: tool calls, mounts, live drawers, boot and repair paths.
2. **FileCabinet security.** One enforcement point. Read, write, protected configuration and cabinet administration are separate permissions. They cover listing, search, copies, moves, mounts, expansion and live-drawer dispatch. Authorization follows the resolved destination, and a cross-cabinet operation checks both sides. Following a mount confers no authority, so mount resolution happens before the access check. Callers see only what they may read: a hidden drawer is indistinguishable from one that does not exist. You can't touch the guitar, and you can't point at it either. Effective inherited access can be shown per drawer.
3. **Trusted entry points.** System and Default Agent authority are separated. Flynn and the prompt loader are named trusted entry points, and there is no agent-selectable "system" bypass.
4. **Admission enforcement.** `zenos.agent.basic` stops being a badge that is requested and becomes one that is checked, through an admission-token contract with `zen_root_authentik`. A Postman approval is bound to the enrollment it was given for and cannot be replayed. Participation stays distinct from administrative authority.
5. **Bootstrap and recovery.** Flynn's cold boot, upgrade, revocation and recovery run on a trusted path, and each is proven.
6. **Audit-only rollout.** Enforcement ships first as would-deny logging, with a rollback, then turns on.
7. **Operator permission report.** Shows the resolved identity, granted capabilities, effective cabinet access and the reason an operation was denied. Secrets are excluded.

**FileCabinet interface cleanup.** The `action_type` field moves to `mode=` (with merge and replace behavior moving to `merge:`), and a compatibility shim carries existing callers. It follows the audit, because the audit maps the same callers.

**Also planned**
- Every tool guards its inputs (the contract sweep begun in 2026.10.1).
- One configure contract across plugin setup.
- Identity and Labels event-ordering hardening.
- Display, finished: the shared dashboard ships, interruption awareness covers the remaining Media Manager watch paths, and expiry and dismissal are tested against the real scheduler cadence.

**Later phase:** binding to the conversation or session. It needs its own design and acceptance criteria before it joins scope.

Order of work: audit, designs, audit-only mode, enforcement.

---

## Scope of the Claim

FileCabinet enforcement protects the ZenOS access boundary. Home Assistant administrators, and any path with direct access to the underlying state, remain part of the trusted platform.

---

## Where It Stands: the Amps Are Wired, the Volume Is at Ten

Every lock in this release is built, wired to its door, and running in **audit mode**. The gate decides every FileCabinet call and every admin-door write, logs what it would have refused or asked a human about, and lets the call through. Nothing is refused yet. The rules it will enforce, the principals they name, and the migration that puts them on every cabinet are all in place, and the logs show what enforcement will change before it changes anything. Turning it to eleven is the next step.

| Planned | Landed (audit mode) | Still to come |
|---|---|---|
| Access-path audit | The audit drove the default table and caller origin below. | Finishing the map of every path in. |
| FileCabinet security | One gate, on the resolved mount target. Separate read, write, delete and admin rules. Default rules for every cabinet class, and locks on credential and approval drawers. Effective access per drawer (`CABS.acl_for`). | Refusing for real. Hiding unreadable drawers from listings and search. |
| Trusted entry points | The gate tells an agent from a tool (caller origin). Boot, Flynn and automations are exempt at the admin door. Provisioning and Flynn's cabinet setup are named lifecycle principals. | Separating System and Default Agent authority. |
| Admission enforcement | | The admission-token contract with `zen_root_authentik`. |
| Bootstrap and recovery | A fresh install migrates and stamps its own schema; an upgrade migrates once and stamps it. | Revocation and recovery on the trusted path. |
| Audit-only rollout | Shipping now: `fc_acl_audit` and `door_gate_audit`. | Enforcement. |
| Operator permission report | | The report. |

---

## The Locks, in Audit Mode

**One record.** Identity and FileCabinet share one ACL record in each cabinet's VolumeInfo: `acls: {v: 2, owner, prime, partners, ops}`. Principals are cabinet GUIDs, never names. `ops` holds a rule for each of read, write, delete and admin: who may do it (`who`), the cert it needs, and whether a live human must say yes (`ack`). Identity writes the record, through the admin door. FileCabinet and Identity read it only through the CABS `acl_*` macros and `ID.principals`, so there is one reading of the rules, not two.

**Every cabinet has a class, and every class has defaults.** A cabinet is `system`, `vault`, `dojo`, `kata`, `history`, `household`, `family`, `user`, `ai_user` or `scratchpad`. Each class has a default row: the system cabinet is read-only to everything but the admin door, the vault answers only to Identity, a person's cabinet answers to that person, their partners and the prime agent, and a scratchpad is open. A drawer may tighten or loosen its own ops; each op falls back on its own to the cabinet's rule and then the class default. The full table is in the [zenos_cabinets.jinja reference](../custom_templates/zenos_cabinets_jinja.md#acl-v2). The class is a stamp in VolumeInfo, which only the admin door can change. The gate never reads a label.

**Some drawers no owner can open.** Certs (`zen_ai_certs`), integration settings (`integrations_config`) and live-ack answers (`alert_response_*`) take writes only from tools, and the console override (`cert_admin_console_bypass`) only from the admin door. These built-in locks win over every other rule, so `acl_set` can't loosen them.

**The gate knows who's asking.** The agent-facing FileCabinet tool marks every call it makes as an agent call. A tool calling the engine is internal, and can name itself with `caller_tool`, so a rule can say `@tool:zen_dojotools_identity` and mean exactly that. New principals in `who`: `@internal`, `@tool:<script>`, `@self`, `@partner`. Before this, every FileCabinet call looked like it came from the default agent. Now the audit log tells an agent from a tool.

**Mounts confer no authority.** FileCabinet resolves the real target behind a mount chain (up to 8 hops; a loop refuses) before it checks access. Reads and writes are checked against the target's rules; delete, move, copy and the mount commands against the pointer's own.

**Two gates, both auditing.** FileCabinet's gate answers `acl_audit` and logs `fc_acl_audit` with the caller's `origin` and, per check, where the rule came from and how the caller matched. The admin door checks agent and console writes against `zenos.system.cabinet_admin` and answers `gate_audit`. Boot, Flynn and automations are exempt.

**Escape hatches, with a paper trail.** A cabinet's owner changes its rules with `identity acl_set`. `identity take_ownership` takes a cabinet over: it needs `zenos.identity.acl_take_ownership` and a live human yes on every call, rewrites only the ACL, never the contents, and is audited. The console override stands in for that yes (and every other live acknowledgement) in exactly one place, Identity's `request_live_ack`, and each use is audited.

**One migration, stamped.** `identity acl_migrate` brings every cabinet to the record: it converts v1 lists to GUIDs, stamps classes, repairs bad identity data and stray VolumeInfo copies, and never drops a dangling principal. A clean run stamps `acl_schema` version 3 on the system cabinet. Flynn runs it on a fresh install and skips it while the stamp is current; `systemtools cabinet_schema_upgrade` runs it when the stamp is behind.

---

## FileCabinet, Rebuilt to Be Lockable

A gate is only as good as the engine behind it, so FileCabinet was rebuilt first.

- **One exit per action.** Get, create, update, upsert, delete and list each set a result and fall through to the engine's single exit, so the gate sits in one place.
- **Honest writes.** A write the cabinet never confirmed answers `write_unverified` and logs a warning, instead of success. `update` on a missing key answers `not_found`. Delete reports `children_deleted`.
- **Every write is a stated merge or replace.**
- **One read taxonomy.** Content is in `value`; `read` is `get`.
- **The drawer envelope is `{value, meta, timestamp[, acl]}`.** `meta` is always present and carries expiry; `acl` carries a drawer's own rules. A cabinet set keeps both instead of rebuilding the drawer.
- **The admin door.** `zen_admintools_cabinet_writer` is the one writer for structural, system and vault cabinet writes, which FileCabinet refuses. It isn't MCP-exposed, every write names its caller and is logged, and it waits until the cabinet shows the write before it answers, and confirms the content. `merge` mode writes only the keys that changed. AdminTools, Flynn, GC and maint fire no raw cabinet events any more.
- **The cabinet guards hold.** The 128 KB size cap and the unknown-at-boot guard refuse the write and leave the cabinet as it was.

---

## Read this before upgrading

- **The ACL v2 migration is mandatory.** Identity setup refuses an old-schema household or family with `acl_schema_upgrade_required`. After reloading (templates, automations, template entities, then scripts), run `zen_dojotools_identity mode=acl_migrate` to see the plan (a dry run is the default). Check every proposed cabinet `class`, especially any it calls `scratchpad`. Then apply it with `dry_run: false` and `confirm_action: true`, or run `zen_dojotools_systemtools tool=cabinet_schema_upgrade`, which applies it when the schema stamp is behind. A second dry run should plan nothing, and the system cabinet then holds `acl_schema` version 3. Flynn's bootstrap runs the same migration on its own until the stamp is current: on a fresh install, and on an existing one the next time the bootstrap runs. Drawers need no migration of their own; existing ones read as before.
- **Expect audit lines.** Agents that don't hold `zenos.system.cabinet_admin` log `door_gate_audit` lines, and the FileCabinet gate logs `fc_acl_audit` wherever enforcement would have refused or asked. They are the preview of enforcement, not errors.
- **FileCabinet reads return content in `value`.** The `{path: value}` map that `read` put in `result` is retired; `result` is only the tool envelope, and `read` is a plain alias of `get`. Anything of your own that reads `result.<path>` needs to read `value`.
- **FileCabinet writes are honest.** A create, update, upsert or delete the cabinet never confirmed answers `status: error`, `code: write_unverified` instead of success. `update` on a missing key answers `not_found` (`force_action` creates it), and `create` on an existing key is an error unless `force_action`.
- **Every write is a merge or a replace.** `upsert` deep-merges by default and `update` replaces. A read-modify-write caller passes `mode: replace`; under a merge, a key you removed comes back.
- **Structural cabinet keys belong to the admin door.** VolumeInfo, `meta`, `_label_index`, `_zen_relationships`, the system cabinet and the vault are written only through `zen_admintools_cabinet_writer`, and FileCabinet refuses them. Raw `set_variable_*` events of your own aimed at those keys need to move to the door.
- **The lens and codex registries have one writer each:** `prompt_loader mode=lens_registry` and `mode=codex_registry`. Code writing those drawers directly needs to go through them.
- **`run_repair` has one action, `acl_migrate`.** The 4.5.6 repair names are refused, and `maint_4_5_6.yaml` and `repair_fam_cab_volumeinfo.yaml` are gone. The cabinet factory no longer takes `family_guid`.
- **`reset_all` wipes.** Cabinet admin's `reset_all` did nothing while answering success; it now wipes every cabinet. A wipe clears the console override and every cert along with everything else, so throw the override from the console and re-request certs afterwards.

---

## Added

- **Semantic roots.** The tool manifest publishes a root for each question word: who (Identity), what (Index), when (Taskmaster almanac), where (Room Manager), why (history), how (help) and knowledge (FileCabinet). Each root answers what matters now and names where to go deeper, and the boot prompt renders them as one line.
- **Taskmaster almanac, the "when" root.** One call returns a headline and the current period of the day, vacation, events from calendars carrying the `almanac` label (timed events first, at most two per calendar), tasks due by today, meals and weather. `horizon_hours`, `lookahead_days` and `max_items` tune it.
- **Per-tool roots on the core tools.** Room Manager, Index, FileCabinet, Identity and Taskmaster declare their own roots in their `tool_manifest` modes. A bare call returns them in about 350 characters.
- **Cortex 45.** Cortex 44 'Open Door' with one fix: its `core_tools` list named a log viewer that does not exist, and now names `zen_dojotools_ha_log_viewer`. 45 is the new latest and replaces the v42 slot, so the selectable versions are 45, 44 and 43.
- **Who knows where a member belongs.** `identity resolve` returns `name`, `person_entity`, `member_type` and `memberships`, and finds a person anywhere in the household tree, not only among the household's own users.
- **The ACL record, cabinet classes, built-in locks, caller origin, both audit gates, and Identity's `acl_migrate`, `acl_set` and `take_ownership`.** See [The Locks, in Audit Mode](#the-locks-in-audit-mode).
- **FileCabinet GC `repair` mode.** Normalizes one drawer to the `{value, meta}` envelope, with `dry_run`.

## Changed

- **Help is the default.** A `zen_*` script answers help for a missing or unknown mode, and a bare call (no mode and no input) never acts, crashes or errors. Tools that used to act on a bare call (announce, the summarizers, the prompt loader, the reset and repair tools, Camera, Provisioner and FileCabinet GC) now return help. Calls that pass input are unchanged, and every internal caller now names its mode.
- **Compact bare replies.** Inventory, Kitchen and Finance return a short mode list on a bare call, down from roughly 100k, 65k and a few thousand characters. The full catalogue is for `mode=help`.
- **Manifest publish cadence.** Publishing runs on the events a reload itself fires, writes only when content changed, and cannot loop on its own events.
- **The KungFu Loader is retired.** Every component self-registers through KF5 (each tool's own `mode=kfc_manifest`, registered by `bootstrap_kfc` on reload and daily), so a prompt load now only loads the prompt and the `ship_zen_system` option is gone. The manifest whitelists `<tool>:kfc_manifest`.
- **FileCabinet internals.** See [FileCabinet, Rebuilt to Be Lockable](#filecabinet-rebuilt-to-be-lockable). In-repo callers call `zen_sutra_filecabinet` directly, one script run instead of two; the wrapper still works. Expiry still stored inside an older value counts, and GC normalizes every drawer it moves. Writes through a mount chain land on the real target, where they used to stop at the first hop.
- **`prompt_loader` runs queued,** up to 25 at once, so a burst of registry calls waits its turn instead of being dropped. When `sim_mode_allowed` is unset, it stamps it from Flynn's `input_boolean.zenos_sim_mode_override` (on by default) instead of `false`, so a load right after `reset_all` still resolves its caller.
- **Identity `area_labels` holds only real HA areas,** so data-routing labels no longer show up in a "who" answer.

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
- **Smaller fixes.** A re-fired alert's answer replaces the cached one. Provisioner's deprovision reports an error when the cabinet doesn't end up unmounted. `flip_schema_version` flips back. Stack and lens unregisters remove their entry. Grocy, Lights, Media Manager, Room Manager and cabinet inspect read drawer content from `value`.

## Removed

- **`maint_4_5_6.yaml` and `repair_fam_cab_volumeinfo.yaml`.** `acl_migrate` covers both, for every cabinet, and Flynn's bootstrap runs it.
- **The 4.5.6 `run_repair` actions** and the cabinet factory's `family_guid` field.
- **The CABS `cabinet_acls` and `drawer_acl` macros.** Use `acl_record` and `acl_for`.
