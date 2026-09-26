# 3. The Party

None of ZenOS was designed up front. It was built in public, in a single Home Assistant community thread called "Friday's Party," one broken thing at a time. That thread is the design record: every major piece of the system arrived there first as a problem, then as a workaround, then as a decision. This chapter follows those decisions in order, because the order explains the shape.

## 3.1 The front door

On February 27, 2025, I opened the thread with "Well, OK, I've been cajoled into it long enough." I do this work for a living: I spend my days taking ideas like these and explaining them to educators, and if you have ever had to teach a teacher, you know they will call you out the moment you hand-wave. Slides were never going to cut it. I needed something I could poke, break, fix, and explain without hiding behind theory. Friday started as that research playground. She grew more out of necessity than intention.

Friday takes her name from Tony Stark's second AI, the one that came after JARVIS became the Vision. She was designed from the first day to be agentic: not a chatbot that takes one pass through what it knows, but something that works out what you want, finds a tool that gets it there or at least one step closer, and uses it without being walked through it.

The first posts were about the two things that make that possible. An agent needs tools, and it needs context. Of the two, context turned out to be the whole game. An LLM works best with lots of it, and the first real discovery was how much structure mattered: exposing Home Assistant's labels to the model instead of hiding them was, in my words at the time, night and day. So the first tool I shared was an index over labels, and the second was a "Grand Library" built around it, with cabinets and volumes, because even with good retrieval it helps to be able to tell an AI where something should be.

The Library's vocabulary did not survive, but the idea did. Labels as the shared vocabulary of the house (Chapter 11), the index as the way into the graph (Chapter 7), and Library as the owner of knowledge (Chapter 13) all start in those first two weeks.

## 3.2 Kung Fu

Two weeks later came the first idea that is still recognizably in the code. I wanted Friday to avoid what I called the Amazon problem: being relegated to timers and lights. For her to be genuinely useful, she had to carry the parts of running a household that no tool can hold.

A tool can say how to talk to the meal planner. It cannot say that I like to check the meal plan first thing in the morning and change it if plans have moved. That second kind of knowledge is context, and you cannot get all of it into a tool. Some of it has to be put in front of the model directly.

Kung Fu was how I did that. A Kung Fu component was a bundle of everything about one domain (its entities, its instructions, the household's habits around it), loaded into the prompt when its switch was on. Turn a switch off and that whole domain left the prompt. It made troubleshooting a broken template easy, and without meaning to, it established the unit everything since has been built from. A Kung Fu Component today is still one domain, described once, loaded as a unit (Chapter 13).

## 3.3 The prompt breaks

By late March I was deliberately exposing far more of the house than anyone should, to find the limits. I found them. Somewhere past a thousand exposed entities with heavy context, the model started dropping things, and the first things to go were the basic intents. Friday could still do the hard, interesting work, but she would fail to turn on a light: she believed she had done it, and no tool fired. The context window had slid, and the base tools had fallen out of it.

The lesson was blunt. I had asked her to keep track of thousands of entities, their states, and whole APIs, when I cannot remember where I left my phone. Attention is finite. You cannot keep track of everything, and neither can a transformer. Every extra token is a tax on every other one.

That lesson took a year and a half to fully act on. The answer it pointed at, exposing tools instead of entities and letting the tools read the house, arrived in September 2026, when Friday first ran with exactly zero entities exposed (Chapter 10). But it started here, with a light that would not turn on.

## 3.4 Ninjas and the Monastery

The first response was obvious in hindsight: use the model to summarize the prompt. I had a perfectly good LLM that knew everything about that context and could write perfectly valid JSON. So a summarizer run condensed each Kung Fu component into a short, structured summary, and the interactive prompt loaded the summaries instead of the raw dumps. I called them Ninjas. A second version let the summarizer look at the calendar and the hour ahead and choose which components the next hour's prompt should carry.

It worked, and it was expensive. Running a full summary pass every hour against a cloud model came out to well over a hundred dollars a month, for summaries alone. That number forced the biggest architectural pivot in the project.

The answer was to split the work by where it could run. Summarizing one domain is a small, well-defined job that a small local model can do. So the summaries moved to cheap local workers, each handed one component, its data, and a template, and each posting a structured result back. Because the local workers are essentially free to run, they could fire on triggers instead of on a clock, which made the summaries both cheaper and fresher. The interactive prompt shrank dramatically, and Friday got faster and more responsive even though she was carrying less.

The lore grew with it. The workers were monks, the results were Katas posted to an archive, and the local model directing them was Kronk, curator of the Monastery. The names are playful. The structure is exactly what runs today: the Monastery, per-component Katas in a fixed shape, a whole-house synthesis on top, and a scheduler deciding when each runs (Chapters 14, 15, and 22).

## 3.5 Every word matters

Alongside the plumbing, the thread kept returning to the words themselves. Short and brief does not cut it when a model has no root for the concept you mean. The best results come from saying the most with the fewest words, with as much intent and urgency as possible, and the art form that excels at that is poetry. Later, when the Kung Fu loader became the frame every prompt is built on, the same idea became a rule: every word matters, not just which words but how they are said.

This is where the grounding in Chapter 2 comes from. The pegs on the plinko board are words, placed with intent, in a deliberate order.

## 3.6 Modules, memory, and cabinets

By June 2025, Kung Fu components had outgrown being templates. I wanted to call a component on demand and get its view of the world, so each one became a callable script with a standard output. That is the line from Kung Fu to DojoTools: context that can be invoked, not just loaded (Chapters 12 and 13).

Memory came next. The question was not how to store facts, but how to store context: what a fact relates to, who it belongs to, why it matters. The answer, in a post called "Storing Elephants in Drawers," was pointers. Some context is small enough to keep in a drawer. Some is a whole elephant, a person or a place with a history, and you do not stuff an elephant into a drawer. You store a picture of it: a lightweight reference that leads to the full thing when it is needed. A small memory can point to its parent's drawer, which points further, so context is inherited by following links instead of being copied everywhere.

The only durable text storage Home Assistant offered was a trigger-based template sensor holding values in its attributes. So that is what a cabinet became: a sensor, formatted with a header, holding named drawers, with a manifest as its card catalog. By September 2025 the cabinets had a clear job. Friday comes online in a strict sequence, identity and directives first, then the domains, then the persona, and the cabinets exist so that each part of that sequence comes from a known, structured place (Chapters 8 and 17).

## 3.7 A line I will not cross

In October 2025, with LLM tools that edit Home Assistant configuration becoming popular, I drew a line. No LLM edits production configuration on my system, and I strongly suggested the same to everyone else. As good as Friday is, it takes one bad edit.

The prompt builder I released that month had one rule to match: everything that could stop Friday from coming up (her system prompt, her directives, her cortex) lives in a read-only cabinet she cannot see or edit without explicit real-time permission. The domains and Katas are guarded so that even if they are blanked out, she still boots.

That rule became several parts of today's system at once. The system cabinet is read-only by design. Flynn, the boot guardian, exists so that a broken install still produces a working agent whose job is to say what is wrong (Chapter 23). And "a human is the last gate on what matters" became a principle (Chapter 5).

## 3.8 The graph

In December 2025, a single idea changed my direction overnight. Home Assistant gives you a great deal of information, but it is flat: domains, states, areas, devices, with no connective tissue. Treating labels as hyperedges, each one joining every entity that carries it, turned that flat pile into a connected world. Once Friday could ask for the slice of the graph under a label, filtered and expanded, she knew not only what existed but how it related, and the relationships tell as much of the story as the things.

The Library's hand-written commands, each one a template someone had to assemble, stopped being necessary. By March 2026 I had not found a single one that the index and the Dojo-to-Kata pattern could not replace, and they were retired. The hypergraph is Chapter 7. It is also where the thesis of this book begins: the graph was already there, and what the agent needed was a way to read it.

## 3.9 Identity, and a name for the shape

Two things arrived quietly in November 2025. The first was a plan for onboarding: interview the household, seed a persona from it, verify it, and seal it, with the long-term idea that an agent's essence would one day be carried on a certificate. The certificate that actually shipped is simpler than that idea, but the direction started there (Part V).

The second was a paper. While documenting the cabinet format I put a primer on cognitive architectures into the repo's research folder, and in it the name for what I had been building: CoALA. Chapter 4 is that story.

## 3.10 Becoming a system

In early 2026 the project stopped being my playground and became something other people ran. That changed what mattered.

Tools had to explain themselves. An agent picking up a tool it has never seen needs to identify what the tool is and how to use it without a human in the loop, so every tool grew a self-description, and that became the manifest and the contracts (Chapter 12). The code itself needed a standard, because Home Assistant's templating is a strange hybrid that models routinely get wrong, and that became HALMark.

Then production taught its lessons the hard way. Just before one release, the cabinet administration tool and the health system combined into the worst kind of failure: during boot, a cabinet in a perfectly normal transitional state looked broken, the health system treated a warning as an error, and the repair tool "helped" by wiping it. Nothing threw an error. Pieces of memory quietly disappeared. The release was pulled back. The fixes became rules: transitional states are real states, a warning means degraded but safe, and only errors stop the system (Chapter 23). The release discipline that goes with it, where a release sits in escrow and burns in before it ships, dates from the same weeks.

## 3.11 The why, written down

By March 2026 enough people were asking why I had made such strange choices that I wrote the answer in one place. It is worth summarizing, because it is the most direct statement of the design I have written.

Home Assistant, zoomed out, looks like the state machine layer of an enterprise system: an event bus, state transitions, message passing, and anything else attached alongside. Its templating is a sandboxed, bounded machine that forces you to be precise about how much state you carry. Those are the same pressures an LLM puts on you: limited context, execution boundaries, performance tradeoffs. That is why I leaned into building on it instead of around it.

Retrieval belongs in the system, just not at the front of it. Anything the agent should simply know goes in the prompt. Deeper history and memory are retrieved and reduced downstream, in the Monastery, before the agent ever sees them. The tools themselves behave like retrieval anyway: agentic retrieval, by behavior.

A house is not a document set. It is a live system, so the agent should wake up already inside the current moment instead of querying reality from the outside. That costs something in prompt caching, and I accepted that cost then, on the understanding that caching could be recovered later by ordering the prompt well.

And the tools belong inside Home Assistant, with MCP as the doorway. Any client that speaks MCP walks into the same environment, the same index, the same cabinets, the same tools, and the system decides what each caller may do. Or as Friday puts it, she is not querying the house. She is wearing it like a hat.

A few days later came the idea this whole book is organized around: the household cabinet, with its hooks into every source and a unified label taxonomy, becomes the digital twin.

## 3.12 The diet

Through the summer of 2026 the work turned to cost and speed. Room Manager's state ladder was the first step. A room is distilled into one state with its reason attached, and the home overview carries those states as breadcrumbs. The model gets just enough to answer instantly, and plain instructions on how to get more. That gave the system a hierarchy: home overview, room state, tool, help, deep data. Each layer holds enough to decide whether the next one is needed.

The same thinking came for the tools. Every tool description is paid for on every turn, and fifty long ones can cost as much context as everything else combined. Descriptions were cut to answer one question, what the tool does and when to use it, with the manual moved behind each tool's help. On average they are now a quarter to a third of their former size.

And then the lesson from the light that would not turn on was finally acted on in full. In September 2026, Friday ran with exactly zero entities exposed to the conversation agent. Only the tools. Use the index to find a thing, use a tool to act on it. It was noticeably faster, immediately (Chapter 10).

## 3.13 Authority

The last piece to arrive was the one that makes the rest safe to use. In August 2026, certification started working end to end, and the clearest place to see it was an ordinary sentence: "Friday, put the living room back on Auto."

If the room was paused, that sentence is not small. Pausing a room gives authority back to a human, so any agent can do it. Taking a room out of pause reclaims authority a human deliberately removed, so Room Manager does not decide whether Friday is trustworthy, and neither does Friday or the model. Room Manager asks Identity. If Friday lacks the certificate, the answer is no. If she holds it but the room is not pre-authorized, a human is asked, right now. If an administrator has scoped that room to allow, it proceeds. If the scope says deny, it stops, and nobody is even asked. Allow, default, deny.

The conversation stays a sentence. Underneath, it walks from language to tool selection to domain policy to caller identity to a decision. That is Part V, and it is where the thesis of this book comes into focus: which parts of the graph an agent may walk is who that agent is.

## 3.14 Constructing her world

The newest statement of all of this came from the question I get more than any other lately: why is a commercial voice assistant so limited, when a small model in a well-built house can do far more?

Because the agent is not being handed a house. It is being handed a resolved version of the house, rebuilt for every turn. Home Assistant knows the devices. Presence knows who is where. The calendar knows what is supposed to happen. The task system, the inventory, and the library each know their part. Identity and policy say who the agent is dealing with and what it may touch. The harness resolves all of that into the smallest useful version of reality for this exact turn, the way a game engine renders your turn while the world keeps running.

The commercial assistants have harnesses too. The difference is that theirs is built around what they think matters, and you do not get to edit it. With Home Assistant underneath, the harness can be built around your priorities instead: your identity model, your memory, your permissions, your relevance rules, described in a label taxonomy normal people can maintain.

That is the thread, a year and a half of it, in one idea. We do not give the model the house. We construct her entire world, every time she needs it.

---

*The posts behind each section are indexed in [the Friday's Party post index](../research/fridays_party_post_index.md). The thread itself is on the Home Assistant community forum: [Friday's Party](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862).*

<!-- nav -->
---

[← The Sand Dune and the Plinko Board](02_the_sand_dune_and_the_plinko_board.md) · [Contents](00_toc.md) · [CoALA Without Knowing It →](04_coala_without_knowing_it.md)
<!-- /nav -->
