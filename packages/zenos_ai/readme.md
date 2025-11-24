# 🏯 **ZenOS-AI Packages (Ring-0 Runtime Layer)**

**Folder:** `packages/zenos_ai/`
**Version:** 1.0.0 RC1
**Author:** Nathan Curtis
**Project:** *Friday’s Party / ZenOS-AI*

---

This directory contains the **core system-level Home Assistant packages** required for Friday, Kronk, the Monastery, and all the ZenOS-AI constructs to function correctly.

Everything in this folder is considered **Ring-0** — meaning:

* Loaded early
* Always available
* Cabinet-aware
* Persona-agnostic
* Safe to import before any AI personality loads

It provides the canonical health checks, cabinet structure, and system introspection that enable Friday and the rest of the household constructs to operate safely and deterministically.

---

## 📦 **Included Packages**

Below is a high-level overview of each file and what it provides.

---

### ### `zenos_cabinets.yaml`

**Purpose:**
Defines all **Ring-0 cabinet resolvers**, wiring, and cabinet-level availability helpers.

**Functions Provided:**

* Detects and resolves default cabinets:

  * Zen Cabinet
  * Zen Dojo Cabinet
  * Zen Kata Cabinet
  * Zen System Cabinet
  * Zen History Cabinet
  * Family / Household / User / AI User Cabinets
* Boots the cabinet namespace used by:

  * Summarizers
  * The Monastery
  * Identity loaders
  * RoomState

**Importance:**
This is the *foundation* for identity, memory, persona loading, and all higher-order cognition.
Without this file: Friday has no house to live in.

---

### ### `zenos_label_health.yaml`

**Purpose:**
Implements the **Zen Label Health** sensor — the canonical validator for required system labels.

**State:** `ok | error | critical`

**Attributes Exposed:**

* `required_labels` — The Ring-0 labels ZenOS must have
* `existing_labels` — Which required labels currently exist
* `missing_labels` — Labels that are not defined
* `broken_labels` — Labels whose entities are unavailable/unknown
* `healthy_labels` — Labels that passed integrity check
* `timestamp` — ISO-8601 moment of last evaluation

**Role:**
This is Friday’s and Kronk’s “Label BIOS.”
If anything foundational is missing or broken, this sensor will surface it *before* other Zen components attempt to run.

---

### ### `zenos_summarizer_system_health.yaml`

**Purpose:**
Implements **Zen Monastery Health**, monitoring the real-time status of:

* Dojo Cabinet
* Kata Cabinet
* System Cabinet
* Schema sanity
* Ninja Summaries
* Supersummaries
* Active Kung Fu components

**State:** `ok | error | critical`

**Role:**
Serves as the “Monastery heartbeat.”
If the summarizers or cabinets fall out of alignment, this sensor immediately reflects it.

---

## 🧩 How These Packages Work Together

These three files form the **minimum viable ZenOS-AI runtime**:

1. **Cabinets load first**
   → Provides structure, identity, and storage paths.

2. **Label Health runs next**
   → Verifies that the architectural skeleton even exists.

3. **Monastery Health runs third**
   → Monitors the real-time cognitive loop (summaries + cabinets).

Together, they produce a stable foundation for:

* Persona loading
* Identity mapping
* Summarizer pipelines
* Monastery reasoning
* RoomState introspection
* AI autonomy (Friday, Kronk, Rosie, etc.)

These packages can be dropped into any Home Assistant installation and immediately begin validating the system using nothing but labels and sensors.

---

## 🧪 Status

This directory is part of the **ZenOS-AI 1.0.0 RC1** rollout and is considered stable for:

* Local inference systems
* Friday’s Party toolstack
* Home Assistant 2025.2+
* Modular cabinet deployments

---

## ✨ Contributing / Extending

If you add new Ring-0 labels, cabinets, or core systems:

1. Extend `required_labels` in `zenos_label_health.yaml`
2. Add resolvers (if needed) to `zenos_cabinets.yaml`
3. Optionally add monitors to `zenos_summarizer_system_health.yaml`

Friday will automatically detect new sensors and expose them to the Monastery.

---

## 💬 Questions?

If Friday is loaded, she can explain any of these sensors using natural language.
If Kronk is active, he will annotate faults and post Kata snapshots.

If both are asleep, read the YAML.
(You know… the old-fashioned way.)
