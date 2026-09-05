# Getting Started — ZenOS-AI 2026.9.0 'Steel Magnolia'

*From fresh install to live agent*

---

New to ZenOS-AI? This folder is your onboarding path. Read in order for a clean first-run experience.

This is not just "install the scripts and talk to the AI." The path is intentionally human-scale — you should understand each layer before the next one comes alive, and it builds toward a real home operations graph:

```text
installed but inert
  -> tools exposed to Assist
  -> rooms mapped, then Room Manager v3 tracks what each one is actually doing right now
     (Vacant/Occupied/Engaged/Asleep/Hold, from real signals, not a fixed schedule)
  -> labels give devices and people meaning
  -> REFLEX reacts to state changes on its own — the right scene fires automatically,
     without your AI having to actively decide anything each time
  -> perception tools like Camera notice useful events
  -> actors like AutoVac, ZenLux, ZenShade, SpaMaster do the physical work
  -> AlertManager decides what's worth attention, Postman asks the right human
  -> you clear, suppress, escalate, or remember
```

The "wow, it noticed that" moment should arrive as a clear consent-based loop, not as a surprise — ZenOS-AI only has the tool access and labeled context you give it. OOBE teaches it where things are and gets the state engine running; labels teach it what things mean; DojoTools give it safe ways to act; REFLEX handles the moment-to-moment reactions on its own; Postman/AlertManager step in only when the system needs a human's judgment instead of a reflex.

---

## Documents in This Folder

### 0. `concepts.md` — Plain-Language Glossary

Read this first if any of the names in this folder (Flynn, Cabinets, Kata, DojoTools/AdminTools, AlertManager/Postman, Room Manager) are new to you. Short, plain-English entries — no procedure, just what things mean.

### 1. `install.md` — Installation

Start here. Covers:

* File drop: `packages/zenos_ai/` and `custom_templates/zenos_ai/` into your HA config
* `configuration.yaml` setup
* **Connecting your phone (Companion App + notify service)** — new, required as of 2026.9.0: certifications need this working before you can grant your AI anything
* Pasting the conversation agent system prompt
* Setting your conversation agent with the friendly `select.zenos_conversation_agent` dropdown once the package has loaded
* Restarting HA and verifying Flynn initializes cleanly
* Checking `sensor.zen_agent_health` for first-boot confirmation

**Time:** ~15 minutes on a clean install.

---

### 2. `first_run.md` — First Boot & OOBE

After installation. Covers:

* What Flynn does on first boot (the stepgate sequence)
* The OOBE conversation flow — naming your AI, building the Room Manager spatial map, and what comes after it (the live state engine + REFLEX)
* Mapping cameras, vacuums, locks, and presence into the label graph
* Using dashboard selectors like `select.zenos_active_persona` instead of raw helper edits
* Editing profiles after setup
* Common first-run issues and fixes

---

### 3. `first_alert.md` — Your First Alert

After first boot. Covers:

* Firing a direct AlertManager test alert
* Seeing fire-once dedup in action
* Listing active alerts through `zen_dojotools_alertmanager`
* Testing error-severity priority context
* Clearing alerts safely
* **Connecting the real device you set up in Install Step 3.5 to Postman** — required before certifications work, not just a nice-to-have

**Do this next — it's the fastest way to prove ZenOS-AI can get your attention, and it's the step that unblocks certifications later.**

---

### 4. `entity_exposure.md` — What to Expose to Your Agent

After the first alert test, use this to clean up what the AI can see and touch. Covers:

* The three-tier model: Actionable vs Contextable vs Invisible
* What always gets exposed (DojoTools scripts)
* What never gets exposed (AdminTools, cabinet sensors, credentials)
* How to use labels instead of direct exposure for high-cardinality data
* How labels feed Room Manager, Camera, AutoVac, AlertManager, and summarizers

**Read this before broadening your conversation agent's entity list.**

---

### 5. `oobe.md` — OOBE Walkthrough

The six-step first-boot configuration protocol in detail. Covers:

* What OOBE is and when it fires
* The conversational path (chatting with your AI to configure it)
* The Agent Builder path for invoking the same OOBE protocol
* How Flynn detects OOBE completion
* What to do if the OOBE notification won't dismiss
* Where to go next for Security Manager and the Room Manager v3 state engine — OOBE sets both up but doesn't walk you through either conversationally

---

### 6. `room_manager_operators_manual.md` — Room Manager v3 & REFLEX

**Read this right after OOBE, before anything else.** OOBE only registers your rooms' *spatial* layout (what connects to what). This is the doc that explains the part that actually runs your house day to day: Room Manager v3 is a live state engine — each room continuously reports Vacant/Occupied/Engaged/Asleep/Hold based on real signals (motion, doors, media), not a fixed schedule. REFLEX is a separate, autonomous layer riding on top of it that fires the right scene automatically whenever a room's state changes, so your AI doesn't have to actively decide "should I turn the lights on" every time someone walks into a room. Covers:

* The full state cascade and what causes each state
* Wasp-hold (motion with no confirmed door-open), entertaining/guest hold, the asleep window
* How REFLEX picks a scene, dry-run/rehearsal mode, and wiring your own scenes
* Manual overrides and troubleshooting a room that "won't clear"

---

### 7. `security_certification_manual.md` — Security & Certification System

**Read this before you grant your AI its first certification** — locks, covers, alarm, infra, and room overrides are all gated behind this system. If you did Install Step 3.5 and First Alert Step 7, its Section 4 prerequisite is already satisfied — otherwise it will hard-block your first grant. Covers:

* What a certification is (`cert_component`, `cert_level`, `cert_scope`) and how it differs from just having a tool exposed
* The two gate types: cert-only vs. cert-plus-a-live-ack-every-call
* Which actions require which tier

---

### 8. `autovac_quick_start.md` — AutoVac Quick Start

The fast path to your first real component: 5 steps, ~15 minutes, gets the vacuum running on a schedule with briefings. See **[AutoVac First Setup](autovac_first_setup.md)** (next) when you want the full chain.

---

### 9. `autovac_first_setup.md` — AutoVac First Setup

The full integrated build. Covers the whole chain:

* DojoTools exposure and dashboard selectors
* Room Manager room truth
* Postman policy seeding and ack test
* Grocy inventory setup
* AutoVac labels, schedules, and room configuration
* Consumables provisioning, wear checks, AlertManager, and final acceptance tests

Use this when you want to prove ZenOS-AI can move from first-run setup to a fully governed automation loop.

---

### 10. `notification_routing.md` — Notification Routing Guide

First Alert's Step 7 already seeded your own device. Read this for the rest: household-wide policy (life-safety bypass, quiet hours), routing to more than one person, and troubleshooting if something still isn't arriving.

---

### 11. `cabinet_placement.md` — Where Things Go and Why

Reference doc, not part of the linear path — most people won't need it. Covers:

* Dojo vs Kata vs System cabinet — what lives where
* Drawer vs KFC — when to use each
* The quick-reference placement table
* Common misplacement patterns and how to fix them

---

### 12. `room_deployment_template.md` — New Room Deployment Template

Reference doc for adding a brand-new room to Room Manager v3 by hand (copy-paste YAML template) — an operator task with file access, not something your AI does conversationally. Most rooms get created automatically through OOBE or `mode=area_create`; reach for this only when you're deploying a room's file directly.

---

### 13. `troubleshooting.md` — Troubleshooting

When something isn't right. Covers:

* Reading health sensors at a glance
* Summarizer kill switches — checking and resetting
* Seven-step graduated repair sequence:
  1. Check `sensor.zen_agent_health` → `roster` attribute
  2. Fire `zen_resolver_refresh`
  3. Reseed templates (`zen_admintools_reset_template`)
  4. Label reset (soft — `reset` mode)
  5. Cabinet repair (`cabinetadmin`)
  6. Nuclear label reset (`zen_admintools_reset_labels`)
  7. `reset_all` cabinet sequence

---

### 14. `user_management.md` — User and AI User Management

For adding, removing, and moving identities after initial setup. Covers:

* Provisioning a new AI user or human user from a stacks cabinet
* Deprovisioning (removing) an identity and returning the cabinet to stacks
* Swapping an identity to a different cabinet (replace mode)
* Transferring the default label before a swap
* Targeted identity-layer repairs: single cabinet reset, label fix, manifest rebuild
* Full identity-layer nuke and restart

---

### 15. `zenzork_manual_unofficial.md` — ZenZork: The Unofficial Manual

Not part of setup — this is the in-universe player's manual for ZenZork, the text-adventure game built on your own live Room Manager topology. Entirely optional, read it whenever you want to actually play.

---

## Recommended Order

```
concepts.md -> install.md -> first_run.md -> first_alert.md -> entity_exposure.md
  -> oobe.md -> room_manager_operators_manual.md -> security_certification_manual.md
  -> autovac_quick_start.md -> autovac_first_setup.md -> notification_routing.md
```

[Cabinet Placement](cabinet_placement.md) and [Room Deployment Template](room_deployment_template.md) are references, not linear reading — jump to them when you need them. Keep [Troubleshooting](troubleshooting.md) and [User Management](user_management.md) open in a tab too. You might need them. [ZenZork: The Unofficial Manual](zenzork_manual_unofficial.md) isn't part of onboarding at all — it's there when you want it.

---

→ **[Full Documentation Hub](../readme.md)**
→ **[Understanding KF4 — Adding Components](../kung_fu/understanding_kf4.md)**
→ **[Health Sensor Reference](../sensors/readme.md)**
