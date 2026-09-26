# 2. Home Assistant as Substrate

ZenOS runs entirely inside Home Assistant. There is no separate broker, database, or service. Every piece of the system is a script, an automation, a template sensor, or a Jinja macro, and every piece of state lives where Home Assistant keeps state. This chapter describes what that substrate gives ZenOS, and just as important, what it does not.

## 2.1 What ZenOS builds on

| Home Assistant primitive | What ZenOS uses it for |
|---|---|
| State machine (entities, states, attributes) | The house itself, and cabinets, which are entities whose attributes hold drawers |
| Registries (areas, floors, devices, labels) | The structure of the graph (Chapters 3 and 5) |
| Event bus | `zen_event` signaling and correlated tool calls |
| Scripts | Every tool: DojoTools, AdminTools, Sutras, Stacks |
| Automations | The Scheduler, the Dispatcher, Room Manager's dispatch, KFC triggers |
| Template engine (Jinja) | Every computation, and the shared macro libraries |
| `ai_task` | Inference, through whichever provider `input_text.zenos_ai_task_entity` names |

Nothing here is private API. ZenOS uses the same primitives any Home Assistant configuration uses, which is why it installs as packages and templates in a config directory.

## 2.2 What the event bus actually guarantees

Volume 1 described the event bus as synchronous, single-threaded, and FIFO, and built claims about deterministic ordering on that description. That overstated it.

Home Assistant runs on an asyncio event loop. Events are dispatched to their listeners, and each automation or script run proceeds concurrently with others, subject to its `mode` (`single`, `restart`, `queued`, `parallel`) and `max`. Two separate runs triggered close together have no guaranteed relative order. What the substrate does guarantee is narrower and sufficient:

* A single script run executes its steps in order.
* `mode` and `max` bound how many runs of one script can overlap, and what happens to the excess.
* State changes to a single entity are atomic.

ZenOS gets its predictability from those guarantees plus its own discipline: single-exit tools (Chapter 8), explicit concurrency modes on every script, and state written through one path (Chapter 4). It does not get predictability from the event bus.

## 2.3 zen_event

ZenOS does not invent event types. Every event it emits is `event_type: zen_event`, and the specific occurrence is carried in a `kind` field inside the event data: `summary_force`, `kata_emit`, `ninja_failure`, `emission_suppressed`, `cabinet_mounted`, `dojotool_call`, `dojotool_return`, and others. A consumer listens for `zen_event` and switches on `kind`. `zen_dojotools_event_emitter mode=help` returns the current lexicon, and the appendix lists it.

The Dispatcher uses this for decoupled tool calls. A caller fires `zen_event` with `kind: dojotool_call`, a tool name, a correlation ID, and a payload. The Dispatcher routes it to the registered script and fires `kind: dojotool_return` with the same correlation ID. An unknown tool returns a structured error on the bus instead of faulting the caller's sequence.

## 2.4 Constraints that shape the design

Several limits of the substrate show up repeatedly in how ZenOS is built. They are worth knowing before reading the rest of the book.

**A script cannot call itself.** Home Assistant blocks recursive script execution unconditionally, regardless of `mode` or `max`. Anything that needs to fan out across every tool, including the tool doing the fanning, lives in a separate script. That is why ToolScan exists (Chapter 9).

**State is a string.** Every state is a string, even when it means a number or a boolean. Attributes can hold structure, and that is where cabinets keep their drawers.

**Stored template output is re-parsed.** Home Assistant coerces the output of a template variable back into native types with literal-eval rules. A value that is not a valid literal (for example an enum's repr) collapses into a plain string. Tools reduce such values to plain strings before storing them.

**A missing import is not soft.** A `{% import %}` of a template file that does not exist raises and kills the whole call. The core macros live in files every tool already depends on for this reason.

**Config check is not schema validation.** `ha_config_check` can pass a script whose structure is wrong enough that Home Assistant disables it at load. Tools are validated by parsing and by loading, not by config check alone.

**Recursion is also blocked across a chain.** If A calls B calls A while A is still running, the inner call is refused. Tools that would form such a chain call the underlying primitive directly instead of going back through the wrapper.
