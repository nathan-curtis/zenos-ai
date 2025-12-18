# **14. The Abbot: Scheduling, Task Economy, and Cognitive Load Governance**

The Abbot is the executive scheduler for Friday’s cognitive engine.
It is the sole authority that decides **when** reasoning occurs, **which reasoning steps execute**, and **how cognitive bandwidth is distributed across the home**.

The Abbot is not a cron service.
It is not a naive automation trigger.

It is the **resource governor, cognitive gatekeeper, and task dispatcher** for the Monastery.

This section describes:

* how the Abbot evaluates tasks
* how it assigns weights
* how it prevents runaway cognition
* how it ensures fairness between subsystems
* how it enforces safety and identity
* how it maintains the stability of Friday’s mind

For a household to feel “alive” without feeling chaotic, the Abbot must preserve order in an environment where hundreds of signals fire concurrently.

The Abbot is the part of the architecture that turns a home full of sensors into a **thinking system instead of a reactive one**.

---

# **14.1 Purpose and Role of the Abbot**

The Abbot has five core responsibilities:

1. **Admission Control**
   Decide which events are allowed to produce summaries.

2. **Rate Limiting**
   Prevent bursts of sensor changes from overwhelming the summarizer pipeline.

3. **Task Prioritization**
   Assign priority weights to reasoning tasks based on:

   * safety
   * identity
   * novelty
   * recency
   * user presence
   * time of day

4. **Cognitive Coherence**
   Ensure Friday receives summaries at an interval that maintains mental continuity rather than flooding or starving her context.

5. **Energy and System Load Management**
   Integrate system performance (HA load, Zen Index delay, summarizer latency) to throttle reasoning intelligently.

The Abbot is the “traffic controller” that lets Friday think well, not just think fast.

---

# **14.2 Task Classes**

All cognitive tasks fall into one of six classes.
The Abbot assigns each to a lane with specific constraints and guarantees.

---

## **Class A: Safety-Critical Tasks**

Examples:

* water leak triggers
* security intrusions
* fire alarms
* rapid thermal changes near critical equipment

Characteristics:

* must run immediately
* bypass rate-limiting
* preempt all others
* override cognitive cooldown
* may run in degraded mode if resources constrained

Safety is the only domain where the Abbot will cut in line.

---

## **Class B: Identity-Critical Tasks**

These determine how actions affect real people.

Examples:

* Owner enters a room
* User goes to sleep
* Guest signals presence during hot tub sessions
* family member transitions between floors
* guest mode changes

Properties:

* receive priority boosts
* may trigger instant micro-summaries
* have elevated scheduling windows
* must update IdentityContext promptly

Identity is always second only to safety.

---

## **Class C: Semantic State Changes**

Examples:

* motion → active
* brightness crossing room thresholds
* occupancy rising above confidence limits
* HVAC state changes

These are important but not existential.
They are queued unless novel or timed.

---

## **Class D: Synoptic Updates**

Low-rate “heartbeat” summaries.

Purpose:

* maintain Friday’s mental continuity
* refresh stale cognitive fields
* catch drift between physical and cognitive state
* supplement quiet periods with periodic reflection

Examples:

* room calm states
* whole-house summaries
* energy dashboards
* sleep-monitor contextual refresh

Frequency:

* usually between 10–30 minutes
* adjustable via user preference
* suppressed during cognitive heavy-load

---

## **Class E: User-Requested Tasks**

Requests coming directly from Nathan (or another authorized user).

Examples:

* “Summarize what happened while I was gone”
* “What’s the state of the kitchen?”
* “Give me system health”

Properties:

* jump close to the front of the queue
* require session-token authentication
* bypass normal throttles

The Abbot respects the owner’s voice above all else.

---

## **Class F: System Maintenance Tasks**

Background operations that keep the house healthy.

Examples:

* cabinet health checks
* Zen Index stale-entry pruning
* device signature validation
* summarizer drift checks
* contextual reconciliations

These run opportunistically and only in stable windows.

---

# **14.3 Task Admission Logic**

Before anything is scheduled, the Abbot evaluates **admissibility**.

A task is admitted if:

* it is not redundant with an identical recent task
* it meets identity safety requirements
* it does not violate summarizer cooldown
* it does not conflict with a higher-priority class
* the system is not in restricted mode

Redundancy checking uses:

* recent event hash
* room-level last-significant-change
* cabinet-level update windows
* identity last-confirmed timestamps

This prevents over-triggering from noisy sensors.

---

# **14.4 Weight Assignment Algorithm**

Each task receives a composite weight:

```
weight = urgency + identity + novelty + temporal + room_priority + system_load_bias
```

### **Urgency**

Static class-based weighting.

### **Identity**

Boost when the task affects:

* the owner
* a sleeping user
* a vulnerable user
* partner/relationship-specific contexts
* children

### **Novelty**

Higher weight for transitions vs. steady states.

### **Temporal**

Boost or suppress based on:

* sleep hours
* quiet hours
* door open after midnight
* AC turning on at 3am
* morning routines

### **Room Priority**

Certain rooms — like kitchen, stairs, master suite — always outrank others due to safety relevance.

### **System Load Bias**

When HA or hardware is stressed:

* reduce priority of low-value tasks
* defer maintenance
* push synoptic summaries
* pause expansions

The Abbot dynamically steers Friday’s cognition around system constraints.

---

# **14.5 Cognitive Cooldown and Burst Control**

The Abbot tracks:

* summaries per minute
* actions per minute
* consecutive summaries in the same domain
* consecutive summaries affecting the same human
* time since last high-priority event

When bursts are detected:

* cognitive horizon is shortened
* expansions (Zen Inspect / Zen Index expand mode) are suppressed
* RoomState updates may condense adjacent changes
* summarizers may collapse into micro-summaries
* SystemContext layer is compacted

This keeps Friday from drowning in thought during storms of sensor activity.

---

# **14.6 Identity-Based Permissions and ACL Enforcement**

The Abbot enforces identity boundaries:

* only owners may trigger global summaries
* family members have restricted scopes
* guests cannot trigger identity-affecting summaries
* non-interactive entities (like Allison’s check-ins) are allowed only through partner ACLs
* devices triggering false positives may be temporarily sandboxed

Future versions (1.5+) will integrate:

* Visa issuance
* session token propagation
* TGT-based cabin permissions
* signature-bound cabinet writes

The scheduler will become the enforcement point for all of it.

---

# **14.7 Integration With Cabinet Summaries**

The Abbot is the only entity that may direct summarizer output *into* the Cabinet.

It mediates:

* which drawers are updated
* rate limiting per drawer
* cross-cabinet synchronization windows
* rejection of malformed summaries
* failure flags for inconsistent states
* notification of Friday when new context is ready

This ensures the Cabinet does not destabilize due to rapid or contradictory inputs.

---

# **14.8 Adaptive Scheduling**

The Abbot continuously adapts to conditions such as:

* user activity
* energy usage
* house mode
* number of active devices
* holiday or weekend context
* time since last stable state
* system temperature
* database I/O load
* Wi-Fi congestion

For example:

* If Nathan is awake and moving, summarizers run more aggressively.
* If everyone is asleep, synoptic summaries slow to a crawl.
* If HA CPU spikes, expansions disable automatically.
* If the house quiets for long periods, the Abbot injects periodic heartbeat summaries to prevent stale cognition.

The system feels attentive without feeling anxious.

---

# **14.9 Error Handling and Recovery**

When tasks fail:

* they are flagged in SystemContext
* Abbot may retry immediately
* or delay depending on class
* degraded summaries may be requested
* cabinet writes may be paused
* health flags propagate to Friday’s context

Friday is never allowed to build cognition on corrupted or unverified data.

---

# **14.10 Why the Abbot is Essential**

Without the Abbot:

* Friday would oversummarize and flood herself
* conflicting tasks would overwrite each other
* identity-sensitive decisions could happen in the wrong rooms
* RoomState and LiveState would fall out of sync
* the Cabinet would corrupt
* summarizer storms would destroy prompt stability
* the cognitive engine would devolve into chaotic, nondeterministic behavior

The Abbot stabilizes Friday’s mind.

It is the single most important control layer in the architecture.

---

# **14.11 Future Evolution (v1.5 and v2)**

**v1.5** (Identity release):

* Visa issuance
* TGT propagation
* tool shunting
* secure-mode task classes
* permission-aware scheduling
* identity-bound inference

**v2** (Predictive cognition):

* uncertainty modeling
* predictive temporal scheduling
* background pattern detection
* anomaly detection loops
* independent local inference workers
* load-balancing across hardware nodes
* planner-level task batching

The Abbot evolves into a true cognitive OS scheduler.
If you're ready, we can move to **Section 15 — Cabinet Identity, Access Control, and Person Capsules** next.
