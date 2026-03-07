# ✅ **Kung Fu Components — README (Tight Edition)**

### *Subsystem Definitions for the Dojo Cabinet*

```markdown
# ZenOS-AI Kung Fu Components  
### Structured Subsystem Logic for the Dojo Cabinet

Kung Fu Components define *how each major subsystem in your home works*.  
They live as drawers inside the **Dojo Cabinet** and provide Friday, Kronk, the Monastery, and the Summarizer with operational guidance.

If the Summarizer explains *what* happened,  
Kung Fu explains *what it means* and *how Friday should think about it.*

---

## 🥋 What a Kung Fu Component Is

A Kung Fu Component is a structured JSON drawer describing:

- what the subsystem does  
- what sensors/entities it depends on  
- what “normal” and “failure” look like  
- what inferences Friday should draw  
- what actions are safe  
- how to coordinate with other managers  
- which master switch controls it  
- which Library command activates it  

Each drawer is **both documentation and logic** — declarative, readable, and prompt-friendly.

---

## 🧩 Why Kung Fu Exists

Kung Fu Components give ZenOS:

- subsystem definitions without hardcoding  
- human-readable reasoning rules  
- safety boundaries  
- shared vocabulary across Index, Library, and Monastery  
- consistent interpretations of sensor events  
- a stable foundation for summaries and prompt assembly  

They're the “knowledge modules” that let the Summarizer and Friday make sense of raw data.

---

## ⚙️ How Kung Fu Fits Into the System

```

Dojo Cabinet → Dojo Loader → Live Prompt
↓
Summarizer
↓
Monastery
↓
Friday’s reasoning

````

- Dojo Cabinet stores the definitions  
- Dojo Loader injects them into Friday’s active cognition  
- Summarizer uses them to contextualize events  
- Monastery uses them to classify, correlate, and evaluate  
- Friday uses them to answer, diagnose, and act  

Kung Fu provides the **semantic grounding** for the whole ZenOS pipeline.

---

## 🛠 Creating Components (KungFu Writer)

Use the `Zen AdminTools KungFu Writer` script to create drawers.

Required fields:

- `friendly_name`  
- `command`  
- `label`  
- `priority` (reflex / core / beta / experimental)  
- `master_switch`  
- `component_summary`  
- `component_instructions`  
- `required_context`  
- `more_info`  
- `version`  
- `drawer`  

The Writer validates metadata, timestamps updates, and writes the drawer into the Dojo Cabinet through FileCabinet.

---

## 📜 Anatomy of a Good Component

A solid Kung Fu drawer includes:

- **Clear narrative** (10–40 lines) describing how the subsystem works  
- **Failure logic** (“If connectivity is down, treat flow as invalid”)  
- **Inference rules** (“High flow during vacancy implies leak”)  
- **Safety conditions** (“Never close valve unless X and Y are true”)  
- **Dependencies** spelled out  
- **One-sentence purpose summary**  
- **Semantic versioning**  

Keep language clear — the monks summarize and Friday interprets.

---

## ⚡ Example (Shortened)

```yaml
friendly_name: Water Manager
command: "~WATER~"
priority: reflex
master_switch: input_boolean.water_manager_master_switch
component_summary: Manages household water flow and leak detection.
required_context: Requires Flume, salt tank, leak sensors.
version: 3.1-draft

component_instructions: >-
  Track flow, leak sensors, salt level, and connectivity.
  Flow is valid only when all connectivity and battery sensors are online.
  Infer leaks from unexpected high flow during vacancy.
  Suppress alerts if washing machine is active (RoomState).
  Mark flow_data_valid=false if connectivity drops; do not auto-shutoff.

more_info: >-
  Protects home from leaks and high-flow anomalies. Works with Security and
  Energy Managers during sleep/away modes.
````

---

## 🤝 Contributing

Add new components when:

* a subsystem has rules
* a domain needs interpretation
* a behavior needs to be taught to Friday
* summarization needs better context
* a safety boundary needs defining

Write clearly.
The Monastery will handle the rest.

```
