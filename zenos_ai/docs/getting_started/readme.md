# Getting Started — ZenOS-AI 2026.6.0 'Clue'

*From fresh install to live agent*

---

New to ZenOS-AI? This folder is your onboarding path. Read in order for a clean first-run experience.

The path is intentionally human-scale. You should understand each layer before the next one comes alive:

```text
installed but inert
  -> tools exposed to Assist
  -> rooms and labels confirmed by you
  -> devices gain meaning
  -> components can notice useful events
  -> AlertManager/Postman can ask for human judgment
```

The "wow, it noticed that" moment should arrive as a clear consent-based loop, not as a surprise. ZenOS-AI only has the tool access and labeled context you give it.

2026.6.0 is not just "install the scripts and talk to the AI." The first-run path builds a home operations graph:

```text
Room Manager map
  -> labeled entities and people
  -> perception tools like Camera
  -> actors like AutoVac, ZenLux, ZenShade, SpaMaster
  -> AlertManager attention
  -> Postman human acknowledgement
  -> clear, suppress, escalate, or remember
```

That is the point of onboarding. OOBE teaches ZenOS where things are, labels teach it what things mean, DojoTools give it safe ways to act, and Postman/AlertManager provide the human-in-the-loop layer when the system needs judgment.

---

## Documents in This Folder

### 1. `install.md` — Installation

Start here. Covers:

* File drop: `packages/zenos_ai/` and `custom_templates/zenos_ai/` into your HA config
* `configuration.yaml` setup
* Pasting the conversation agent system prompt
* Setting your conversation agent with the friendly `select.zenos_conversation_agent` dropdown once the package has loaded
* Restarting HA and verifying Flynn initializes cleanly
* Checking `sensor.zen_agent_health` for first-boot confirmation

**Time:** ~15 minutes on a clean install.

---

### 2. `first_run.md` — First Boot & OOBE

After installation. Covers:

* What Flynn does on first boot (the stepgate sequence)
* The OOBE conversation flow — naming your AI and building the Room Manager spatial map
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

**Do this next — it's the fastest way to prove ZenOS-AI can get your attention.**

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

### 5. `cabinet_placement.md` — Where Things Go and Why

After entity exposure. Covers:

* Dojo vs Kata vs System cabinet — what lives where
* Drawer vs KFC — when to use each
* The quick-reference placement table
* Common misplacement patterns and how to fix them

---

### 6. `oobe.md` — OOBE Walkthrough

The six-step first-boot configuration protocol in detail. Covers:

* What OOBE is and when it fires
* The conversational path (chatting with your AI to configure it)
* The Agent Builder path for invoking the same OOBE protocol
* How Flynn detects OOBE completion
* What to do if the OOBE notification won't dismiss

---

### 7. `troubleshooting.md` — Troubleshooting

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

### 8. `user_management.md` — User and AI User Management

For adding, removing, and moving identities after initial setup. Covers:

* Provisioning a new AI user or human user from a stacks cabinet
* Deprovisioning (removing) an identity and returning the cabinet to stacks
* Swapping an identity to a different cabinet (replace mode)
* Transferring the default label before a swap
* Targeted identity-layer repairs: single cabinet reset, label fix, manifest rebuild
* Full identity-layer nuke and restart

---

## Recommended Order

```
install.md -> first_run.md -> first_alert.md -> entity_exposure.md -> cabinet_placement.md -> oobe.md
```

Keep `troubleshooting.md` and `user_management.md` open in a tab. You might need them.

---

→ **[Full Documentation Hub](../readme.md)**
→ **[Understanding KF4 — Adding Components](../kung_fu/understanding_kf4.md)**
→ **[Health Sensor Reference](../sensors/readme.md)**
