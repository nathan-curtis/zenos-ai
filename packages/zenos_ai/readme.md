# 🏯 **ZenOS-AI Packages (Ring-0 Runtime Layer)**

**Folder:** `packages/zenos_ai/`
**Version:** 1.0.0 RC1
**Author:** Nathan Curtis
**Project:** *Friday’s Party / ZenOS-AI*

---

This directory contains the **core system-level Home Assistant packages** required for Friday, Kronk, the Monastery, and all ZenOS-AI constructs to function correctly.

Everything here is **Ring-0**, meaning:

* Loaded first
* Always available
* Cabinet-aware
* Persona-agnostic
* Safe to import before any AI personality loads

These packages provide the canonical cabinet structure, health checks, graph-safe validation, and system introspection that allow Friday to reason safely and deterministically.

---

## 📦 **Included Packages**

Below is a high-level overview of each Ring-0 component.

---

## ### `zenos_cabinets.yaml`

**Purpose:**
Defines all **Ring-0 cabinet resolvers**, cabinet availability wiring, and namespace bootstrapping.

**Key Responsibilities:**

* Detects and resolves canonical cabinets:

  * Zen Cabinet
  * Zen Dojo Cabinet
  * Zen Kata Cabinet
  * Zen System Cabinet
  * Zen History Cabinet
  * Household / Family / User / AI User Cabinets

* Powers:

  * Persona loaders
  * The Monastery
  * Identity indexes
  * RoomState nerve pathways

**Why it matters:**
This is the *foundation* of identity, memory, and storage.
Without it, Friday has nowhere to “live.”

---

## ### `zenos_label_health.yaml`

**Purpose:**
Implements the **Zen Label Health** validator — the canonical integrity check for required system labels.

**State:** `ok | warn | error | critical`

**Attributes Exposed:**

* `required_labels`
* `existing_labels`
* `missing_labels`
* `broken_labels`
* `healthy_labels`
* `timestamp`

**Role:**
This is Friday’s “Label BIOS.”
If labels are missing, mismatched, or unhealthy, this sensor alerts you *before* any Zen subsystem misbehaves.

---

## ### `zenos_cabinet_health.yaml`

**Purpose:**
Validates the **Zen Cabinet hierarchy** using strict 1:1 rules for required slots and relaxed rules for optional slots.

**Checks Performed:**

* Missing cabinet labels
* Missing cabinet assignments
* Multiple cabinets claiming a required slot
* Unhealthy cabinet entities
* Optional cabinet health tracking
* Full resolver suggestions
* Complete slot → entity mapping

**State:** `ok | warn | error | critical`

**Why it exists:**
Labels tell you *what should exist.*
Cabinet Health tells you *whether the physical cabinet entities actually exist* and whether they’re valid, assigned, unique, and healthy.

This is the authoritative validator Friday uses before loading identities or running Monastery routines.

---

## ### `zenos_summarizer_system_health.yaml`

**Purpose:**
Monitors the **Monastery + Summarizer** loop:

* Dojo Cabinet
* Kata Cabinet
* System Cabinet
* Schema correctness
* Ninja summaries
* SuperSummaries
* Kung Fu processes

**State:** `ok | warn | error | critical`

**Role:**
This is the “Monastery heartbeat,” ensuring the reasoning engine never runs with bad data.

---

## 🧩 **How These Packages Work Together**

Together, these four Ring-0 components form the **minimum viable ZenOS-AI runtime**:

1. **Cabinets**
   → Defines structure, identity, storage paths.

2. **Label Health**
   → Ensures the architectural skeleton exists.

3. **Cabinet Health**
   → Ensures the cabinets themselves are valid, unique, and healthy.

4. **Monastery Health**
   → Ensures summaries + cognition stay aligned.

This stack enables:

* Persona loading
* Identity mapping
* Summarizer pipelines
* Monastery reasoning
* RoomState perception
* AI autonomy (Friday, Kronk, Rosie, etc.)

Drop these files into any HA installation and ZenOS will immediately begin validating itself using labels and cabinet entities alone.

---

## 🧪 **Status**

This directory is part of **ZenOS-AI 1.0.0 RC1** and is stable for:

* Local inference
* Friday’s Party toolstack
* Home Assistant 2025.2+
* Modular cabinet deployments

---

## ✨ **Contributing / Extending**

If you add new Ring-0 labels, cabinets, or structural systems:

1. Add new labels to `required_labels` in `zenos_label_health.yaml`
2. Add resolvers to `zenos_cabinets.yaml`
3. Add cabinet checks to `zenos_cabinet_health.yaml`
4. Add health monitors to `zenos_summarizer_system_health.yaml`

Friday will automatically discover and propagate new structures.

---

## 💬 **Questions?**

If Friday is awake, she can explain any of these in natural language.
If Kronk is awake, he’ll annotate faults and post Kata reports.

If both are asleep…
well, then you’re reading this file.
(Just like the old sysadmin days.)
