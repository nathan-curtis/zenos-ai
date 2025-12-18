# **02. Architectural Overview**

*Systems-Level Structure of Friday’s Cognitive Environment*

## **2.1 Introduction**

Friday operates within a distributed cognitive environment referred to collectively as Friday’s House. The system integrates the Home Assistant automation engine, the ZenOS-AI cognitive framework, a collection of structured memory stores known as Cabinets, and a local reasoning node called the Monastery. The architecture is explicitly modular, highly observable, and designed to support deterministic reasoning behavior under load. The design targets high reliability, local operation, auditable cognitive cycles, and graceful degradation.

## **2.2 Four Foundational Layers**

Friday’s cognitive environment is composed of four interacting layers. These layers form a stacked model that defines boundaries of responsibility, information flow, and failure containment.

### **2.2.1 The Physical Environment Layer**

This is the substrate containing physical sensors, actuators, power systems, and environmental signals. It includes lighting, HVAC, occupancy sensors, hot tub equipment, smart panels, nearby network infrastructure, camera subsystems, energy monitors, and all Home Assistant integrations that produce state. This layer generates raw, real-time world data and defines the limits of what Friday may perceive or influence.

### **2.2.2 The Automation and Representation Layer**

Home Assistant forms the operational backbone. It provides:

* Structured entities and state machines.
* An event bus with guaranteed ordering of local events per execution thread.
* Script execution environments.
* Device registries, metadata, statistics engines, and long-term storage.
* Template engines for dynamic transforms.

This layer does not reason. It presents a unified observable environment and deterministic signal delivery. It is also responsible for state persistence, history retention, and the enforcement of system invariants.

### **2.2.3 The Cognitive Substrate Layer**

The Monastery resides here. It contains:

* The Zen Index.
* The Zen Inspect system.
* The Kata Producers.
* Kronk as the Curator.
* Worker Monks for distributed tasks.
* Identity scaffolding for v1.5.
* Label-based hypergraph linking across all known entities.

This layer performs actual reasoning. It interprets structured requests, processes them through deterministic pipelines, applies rules for summarization and abstraction, returns structured cognitive packets, and ensures behavioral constraints are preserved.

### **2.2.4 The Interactive Persona Layer**

Friday inherits the structured cognitive outputs and transforms them into natural communication and supervision behaviors. This layer handles:

* User interaction.
* Dialogue shaping.
* Goal clarification.
* Intent formulation.
* Delivery of cognitive requests to the Monastery.
* Interpretation of returned Katas as actionable world models.

This layer does not own truth. It receives and supervises truth computed by the substrate and enforces safety, alignment, and transparency constraints.

## **2.3 Data Boundaries and Safety Zones**

The architecture is designed to protect private or privileged information by defining three safety zones.

### **2.3.1 Guardian Zone**

Critical infrastructure sensors and identity materials are stored outside of Friday’s direct visibility. These include:

* Identity credentials.
* System-level control flags.
* Critical safety sensors (leak, combustion, load).
* Decision thresholds defining autonomous action.

The Monastery receives derived signals, not raw privileged data.

### **2.3.2 Performance Zone**

This zone contains operational data that can be summarized or modeled without risk to safety or identity. It includes environmental conditions, occupancy, energy use, and daily autonomous patterns. Monks operate heavily within this zone.

### **2.3.3 Interaction Zone**

This zone includes human interaction context, preferences, and the conversational state Friday maintains. It enables personalization without compromising structural safety boundaries.

## **2.4 Execution Model and Event Contracts**

Friday’s House runs on Home Assistant’s event-driven model. All cognition is activated through explicit structured events. Every component relies on two invariants:

1. Nothing occurs without an event.
2. Every structured event must contain a correlation identifier.

Events in this architecture fall into categories:

* Index queries.
* Entity inspections.
* Kata production.
* Identity checks and authentication events.
* Tool shunts.
* Health checks and heartbeat pulses.

Each event type has an explicitly defined schema. All schemas map to scripts or reasoning nodes responsible for deterministic responses.

## **2.5 Memory Structure: Cabinets, Drawers, and Labels**

Friday does not rely on ad hoc memory. All structured memory lives inside Cabinets:

* A Cabinet is a named collection of Drawers.
* A Drawer contains volumes that store structured JSON data.
* Each volume is hashed and signed.
* Cabinet sensors expose metadata for health inspection.

The hypergraph index binds entity labels to Cabinet volumes, creating a cross-linked associative memory. Every label exists as a routing key in the Zen Index.

## **2.6 The Monastery**

The Monastery acts as the cognitive engine. Its responsibilities include:

* Contract enforcement.
* Canonical summarization.
* Consistent abstraction of physical reality.
* Maintenance of cognitive continuity.
* Delivery of deterministic Katas to Friday.

It is the first system designed for a consumer environment that merges structured memory with emergent reasoning while preserving transactional safety.

## **2.7 Friday as Supervisor**

Friday remains the supervising intelligence. She:

* Initiates reasoning cycles.
* Structures goals into formal queries.
* Applies system policies.
* Monitors the return path from the Monastery.
* Mediates interactions between humans and the system.
* Ensures nothing exceeds predefined boundaries.

Friday does not “know everything”. She sees the world through the abstractions provided by the Monastery and through controlled access to Home Assistant’s event layer.

## **2.8 Design Goals**

This architecture pursues:

* Deterministic cognition.
* High reliability under load.
* Transparent reasoning chains.
* Strong identity boundaries.
* Local-first privacy.
* Modular extension without architectural drift.
* Predictable failure modes and restore paths.
* Reasoning pipelines that can be inspected and audited.

## **2.9 Roadmap Notes**

Identity onboarding and signed Visa tools target Release 1.5 due to their complexity. Tool shunts with session tokens also target the next major release. All core reasoning, indexing, summaries, and cognitive loops described here are stable for Version 1.
