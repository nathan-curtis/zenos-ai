## **ZenOS-AI Whitepaper "The Book of Friday" — Table of Contents**

*(Release Target: ZenOS-AI v1.0 — Identity may slip to v1.5)*

*(Content accuracy pass 2026-08-04: chapters 01, 04, 05, 06, 07, 08, 10, 14, 15,
20, and security_model_ga corrected against current code. Fixed: stale
entity/script names, fabricated dotted event types, a wrong validation
signature constant, and several design-target claims (identity_hash
enforcement, Abbot-level ACL enforcement, ch06/ch14's task-economy/scoring
model) reframed as not-yet-implemented rather than current behavior. Redirector
(ch10) documented as shipped — the Cabinet package, `zen_dojotools_filecabinet`.
Abbot (ch06/ch10/ch14) documented as shipped — the Scheduler + Dispatcher
working jointly, not a single component. "Monk Summaries" Tier 2 (ch07)
corrected to `trapper_keeper`, its real and leaner realization; "Monk" clarified
as the inference-execution step used inside every tier, not a separate layer.)*

<img 
  src="https://github.com/user-attachments/assets/4b337f77-5c3e-4704-9d29-0749b5b7a187"
  width="512"
  height="768"
  alt="image"
/>

---

## **ZenOS-AI Whitepaper "The Book of Friday" — Table of Contents**

*(Release Target: ZenOS-AI v1.0 — Identity may slip to v1.5)*

---

## **0. Preface & Release Notes**
* **[readme.md](./readme.md)** — Executive overview, architectural intent, release scope.
* **[00_toc.md](./00_toc.md)** — This document.

---

## **1. The Monastery Core**
* **[01_the_monastery_core.md](./01_the_monastery_core.md)**

## **2. Architectural Overview**
* **[02_Architectural_Overview.md](./02_Architectural_Overview.md)**

## **3. Cognitive Architecture Foundations**
* **[03_Cognitive_Architecture_Foundations.md](./03_Cognitive_Architecture_Foundations.md)**

## **4. Cognitive Data Flow**
* **[04_Cognitive_Data_Flow.md](./04_Cognitive_Data_Flow.md)**

## **5. Reasoning and Kata Design**
* **[05_Reasoning_and_Kata_Design.md](./05_Reasoning_and_Kata_Design.md)**

## **6. Scheduler and The Abbot**
* **[06_Scheduler_and_The_Abbot.md](./06_Scheduler_and_The_Abbot.md)**

## **7. Summarizer Pipelines**
* **[07_Summarizer_Pipelines.md](./07_Summarizer_Pipelines.md)**

## **8. Kata Cabinet**
* **[08_Kata_Cabinet.md](./08_Kata_Cabinet.md)**

## **9. Identity Architecture**
* **[09_Identity_Architecture.md](./09_Identity_Architecture.md)**

## **10. Event Substrate and Home Assistant Implementation**
* **[10_Event_Substrate_and_HomeAssistant_Implementation.md](./10_Event_Substrate_and_HomeAssistant_Implementation.md)**

## **11. RoomState and Perception**
* **[11_RoomState_and_Perception.md](./11_RoomState_and_Perception.md)**

## **12. LiveState: Authoritative Environment Model**
* **[12_LiveState_Authoritative_Environment_Model.md](./12_LiveState_Authoritative_Environment_Model.md)**

## **13. Cognitive Context Construction**
* **[13_Cognitive_Context_Construction.md](./13_Cognitive_Context_Construction.md)**

## **14. Abbot Scheduler and Task Economy**
* **[14_Abbot_Scheduler_And_Task_Economy.md](./14_Abbot_Scheduler_And_Task_Economy.md)**

## **15. Identity Access Control: Person Capsules**
* **[15_Identity_AccessControl_PersonCapsules.md](./15_Identity_AccessControl_PersonCapsules.md)**

## **16. Summarizer Engine: Kata Pipeline**
* **[16_SummarizerEngine_KataPipeline.md](./16_SummarizerEngine_KataPipeline.md)**

## **17. Katas: Structure, Semantics, Validity**
* **[17_Katas_Structure_Semantics_Validity.md](./17_Katas_Structure_Semantics_Validity.md)**

## **18. Context Frame: Operational Cognitive Surface**
* **[18_Context_Frame_Operational_Cognitive_Surface.md](./18_Context_Frame_Operational_Cognitive_Surface.md)**

## **19. Resilience and Failure Modes**
* **[19_Resilience_and_Failure_Modes.md](./19_Resilience_and_Failure_Modes.md)**

## **20. Tool Invocation and Security**
* **[20_tool_invocation_and_security.md](./20_tool_invocation_and_security.md)**

## **21. Developer Taxonomy and Component Standards**
* **[21_Developer_Taxonomy_and_Component_Standards.md](./21_Developer_Taxonomy_and_Component_Standards.md)** — Canonical component classes (DojoTool, AdminTool, Root, Sutra, Stack, Codex, KFC, Boot Orchestrator), Stripes, exposure rules, packaging conventions, and the design decision guide for core developers, plugin authors, and reviewers.

## **22. Room Manager v3 & REFLEX: The Living Room-State Engine**
* **[22_Room_Manager_v3_REFLEX.md](./22_Room_Manager_v3_REFLEX.md)** — The concrete implementation of Chapter 11's RoomState theory: the per-room cascade (`room_state.yaml`), the REFLEX event bus (Signal Dispatcher → scene resolution → nightlight), opt-in constructs (Control Burnout, TV Sleep Timer, Vent Fan Auto), and the one-automation/one-script consolidation. Distinct from `zen_dojotools_room_manager` (RoomReg, spatial topology) — see [components/room_manager_v3_reflex.md](../components/room_manager_v3_reflex.md) for the practical/agent reference and [getting_started/room_manager_operators_manual.md](../getting_started/room_manager_operators_manual.md) for the plain-language operator's manual.

---

## **Appendices (Future Work: Sections 23–30)**

Reserved for:

* SSE/MCP v2 architecture  
* Persona certificates & identity proofs  
* Kata algebra research  
* Hypergraph engine v2  
* External household federation  
* Distributed Monastery workers  
* Multimodal pipelines (vision/audio)  
* Advanced safety model  
* Formal proofs of invariants
