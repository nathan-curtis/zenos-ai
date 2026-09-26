# 17. Identity as Traversal

The preface stated the thesis: an agent's identity emerges from the graph connections it can realize. Chapters 14 through 16 described the mechanics. This chapter puts them back together and shows that they are one idea, not four.

## 17.1 The graph an agent walks

Every capability in ZenOS is a walk across the graph. When Friday turns off the kitchen lights, she does not address an entity. She asks ZenLux for the kitchen, and ZenLux walks from the room label to the entities that carry it, to their roles, to their capabilities. When she checks on a room, Room Manager walks from the area to its sensors, its portals, and its neighbors. When the summarizer builds a Kata, it walks from a component's declared labels to the state those labels reach (Chapter 10).

So the question "what can this agent do?" has a precise form: which edges of the graph is it allowed to traverse, and from where?

## 17.2 Four kinds of edge

The mechanics in Part V are each a restriction on a different kind of edge.

| Edge | Question it answers | Mechanism |
|---|---|---|
| Membership | Whose world is this agent part of? | Household and family membership, resolved to depth 2 (Chapter 14) |
| Capability | Which classes of action may it attempt? | Certifications, with dotted names forming a hierarchy (Chapter 15) |
| Target | Which specific nodes does a capability reach, and which are fenced off? | Scope entries: `allow`, `deny`, or not mentioned (Chapter 15) |
| Approval | Does crossing this edge, this time, need a human? | Live acknowledgement (Chapter 16) |

An action happens only if every edge on its path is traversable. An agent certified for `lock_control` at level 1, whose scope denies `lock.back_door`, has a capability edge into the lock domain and a fenced-off target edge to one door. The front door is reachable for locking. Unlocking it crosses an approval edge, so it needs a human unless that target is explicitly covered. The back door is not reachable at all.

## 17.3 Why this is identity

In most systems identity is a label attached to a caller, and authorization is a lookup keyed by that label. That model breaks down for agents. An agent's name tells you nothing about what it will do, and a persona prompt is advice the model may or may not take.

What does determine an agent's behavior is which parts of the house it can actually reach. Two agents with identical prompts and different certifications are, in every way that matters to the house, different agents. One agent whose certifications change has, in the same sense, become a different agent. The set of traversable edges is the operational definition of who the agent is.

This is also why the certification hierarchy is shaped the way it is. Dotted names (`zenos.media.playback_control` under `zenos.media`) are a tree of capability edges. A broad grant opens a subtree. A narrow grant opens one branch, and because the most specific match wins, a narrow grant can also close part of a subtree a broad grant opened. The shape of the grant is the shape of the reachable graph.

## 17.4 The twin is identity-shaped

Chapter 13 describes the twin: the model of the house an agent builds by reading the graph through the ontology. The traversal view explains why two agents can build different twins of the same house.

Tools resolve targets through labels, and gated tools refuse traversals the caller cannot make. An agent that cannot traverse into the security domain can still be told the house is armed by a summary, but it cannot walk the alarm's zones, and it cannot act on them. Its twin has the node and not the edges. The house is the same. The reachable graph is not.

## 17.5 Where this is not yet true

> **Not yet built.** Every call currently resolves to the install's default agent (Chapter 14), so today there is one reachable graph per install, shaped by the default agent's certifications. Per-principal traversal, where different sessions resolve to different personas and therefore different reachable graphs, arrives with real session binding. The chokepoint is already in place, so no tool changes when it does.

> **Not yet built.** Membership edges are resolved (to depth 2) but do not yet gate any action on their own. Today a caller's household or family membership does not change what it may traverse. Only certification, scope, and acknowledgement do.

> **Not yet built.** The graph fold that produces an agent's context (Chapter 3) and the traversal rules that decide what it may act on are separate code paths. Computing an agent's claims directly from a fold over the label graph, so the same traversal that builds its world also bounds it, is design direction.
