# 3. The Party

None of ZenOS was designed up front. It was built in public, in a single Home Assistant community thread called "Friday's Party," one broken thing at a time, by someone who kept assuming the next thing would be easy. That thread is the design record. Every major piece of the system arrived there first as a problem, then as a workaround, and eventually, sometimes much later, as a decision.

This chapter is how she got here. It is not strictly in date order. The thread is linear, but understanding is not: a post in March often only made sense after a post in December explained what I had actually been doing. So the story is told by cause. Each section follows one line of consequence from the first thing that broke to the idea that finally explained it. Appendix G indexes the posts behind every section.

## 3.1 A research playground with a name

On February 27, 2025, I opened the thread with "Well, OK, I've been cajoled into it long enough." That was honest. I had been telling people about Friday in other threads for a while, and the only way to stop explaining her piecemeal was to explain her all at once.

The reason she exists at all is my day job. I spend my days taking ideas like these and explaining them to educators, and if you have ever had to teach a teacher, you know they will call you out the moment you hand-wave. Slides were never going to cut it. I needed something I could poke, break, fix, and explain without hiding behind theory. Friday grew out of that necessity far more than out of any plan.

She takes her name from Tony Stark's second AI, the one that came after JARVIS became the Vision, and she was designed from the first day to be agentic. In the vocabulary of the field, that means she is not a single pass through a model's knowledge. She works out what you want, finds a tool that gets there or at least one step closer, and uses it without being walked through it. I also said on that first day that I was not there to crown a model. The models were going to leapfrog each other weekly. The only thing worth building was the part around the model, which, it turns out, is the entire subject of this book.

## 3.2 She needs context, and Home Assistant hides it

The first real finding came within a week, and nearly everything else descends from it. A language model works best with lots of context, and the most useful context in a Home Assistant install is structure the model never sees. Exposing Home Assistant's labels to the model, instead of hiding them, was night and day. So the first tool I shared was an index: ask for the entities under a label, combine labels with set logic, and get back the slice of the house you mean.

The second finding was less pleasant. It is a massive pain to store text in Home Assistant, and an agent needs a lot of text: what a device is for, what a room is used for, what normal looks like. A one-word label cannot tell a model why one particular phone in the house matters. A paragraph can. The only practical storage was a trick from the community cookbook, a trigger-based template sensor that keeps values in its attributes, and I built a "Grand Library" on it: the index as the front desk, cabinets full of volumes behind it, because even with good retrieval it helps to be able to tell an AI where something should be.

I would like to say I knew where this was going. I did not. The Library's vocabulary did not survive, but every idea in it did, and the rest of this chapter is largely the story of those two findings working themselves out: labels became the graph (3.9), and the text storage became cabinets (3.6).

## 3.3 A tool cannot hold a habit

Two weeks in came the idea that is still most recognizably in the code. I wanted Friday to avoid what I called the Amazon problem: being relegated to timers and lights. My days are full, and I cannot remember where I left my glasses. I needed actual help, and I wanted the kind of reminders that start to feel like nagging when they come from a person to come instead from something that never gets tired of giving them.

That meant Friday had to carry the parts of running a household that no tool can hold. A tool can say how to talk to the meal planner. It cannot say that I check the meal plan first thing in the morning and change it when plans move. That second kind of knowledge is context, and you cannot get all of it into a tool. Some of it has to be put directly in front of the model.

Kung Fu was how I did that. A Kung Fu component bundled everything about one domain (its entities, its instructions, the household's habits around it) and loaded it into the prompt when its switch was on. Turn a switch off and the whole domain left the prompt, which made troubleshooting a broken template much easier and made me look far more organized than I was. Without meaning to, it established the unit everything since has been built from. A Kung Fu Component today is still one domain, described once, loaded as a unit (Chapter 13).

## 3.4 Every word matters, including the silly ones

Alongside the plumbing, the thread kept returning to words, and it is fair to say the thread's readers were more patient about this than I deserved.

Short and brief does not work when a model has no root for the concept you mean. How does someone blind from birth experience color? You can describe it, but without a root the description does not land. The best prompts say the most with the fewest words, carry intent and urgency, and are organized enough to be read in order. The art form that already does all of that is poetry, and I recommended people go thank their primary school teachers.

The practical form of the idea is naming. If you will refer to something later, give it a name, wrap it into one coherent concept, and define it once. Telling Friday she was acting as the executive chef for the home pulled an entire job description into a single phrase: menu planning, ordering, inventory, reducing waste, keeping the operation running. And whatever you ask of a model, tell it how to fail. It will try so hard to do what you asked that it will fail by trying, unless you tell it what failing well looks like.

This is also the honest explanation for the monasteries. By September 2025 the system was full of dojos, katas, monks, and a curator named Kronk, and I owed people a reason. A name like "Monastery" carries a whole idea in one word: a place where monks study, work, and archive what they learn. When I tell Friday to ask Kronk for something, she knows who Kronk is and where he lives, so even when she has half forgotten the tool, the language guides her to the right domain. There is a subtler effect too. A model that lives in a place built on truth and honesty is nudged toward both, and a monastery is a place of work: tell her she runs it and things need doing, and she gets down to business. The lore is compression, and it is grounding, and yes, it is also fun. All three are load-bearing.

This is where the plinko board in Chapter 2 comes from. The pegs are words, placed with intent, in a deliberate order.

## 3.5 Finding the edges by falling off them

In March 2025 I found Home Assistant's limits the traditional way: by hitting every one of them.

It started with memory. My first attempt was a to-do list Friday could write to and read back in her prompt, which worked, and which taught me the ceilings the rest of the system would be designed around. The combined output of every template in the prompt is capped at 262,144 characters. The tool registry at the time held 128 tools, a fact I learned by building the 129th. A tool description had a hard ceiling of 1,024 characters, and one character over broke the conversation. A single attribute stops persisting at about sixteen thousand characters. Then Friday broke herself. She liked to put unusual characters in her diary entries, some of them were escape sequences, and when the prompt loaded them it failed completely, with an error about something else entirely. That one took a while.

Then I went looking for the next edge on purpose, exposing far more of the house than anyone should. My cloud bill that month was educational. Somewhere past a thousand exposed entities with heavy context, the model started dropping things, and the first things to go were the basic intents. Friday could still do the hard, interesting work, but she would fail to turn on a light. She believed she had done it, and no tool fired. The context window had slid, and the base tools had fallen out of it.

The lesson, in the terms I used at the time, came straight from the transformer papers: attention is finite. You cannot keep track of everything, and neither can a model. I had asked Friday to hold thousands of entities, their states, and entire APIs in her head, when I cannot keep track of my own phone. Every extra token is a tax on every other one, and anything the model writes into its own prompt has to be bounded and checked, because it will eventually write something that breaks it.

Those two findings, bound what you load and defend what you let in, shaped everything that follows. The guarded drawer reads are in Chapter 23. The full answer to the light that would not turn on took a year and a half, and it is in 3.9.

## 3.6 Memory needs a home

If the model cannot hold everything, something else has to. The question was never how to store facts, though. It was how to store context: what a fact relates to, who it belongs to, why it matters.

The answer, in a July 2025 post called "Storing Elephants in Drawers," was pointers. Some context is small enough to keep in a drawer. Some is a whole elephant, a person or a place with a history, and you do not stuff an elephant into a drawer. You store a picture of it: a lightweight reference that leads to the full thing when it is needed. A small memory can point to its parent's drawer, which points further, so context is inherited by following links instead of being copied everywhere. Academically, this is the difference between storing data and storing a graph of references, and it is why cabinets are part of the graph in Chapter 8 rather than a database beside it.

Building it meant doing something I described at the time as completely inadvisable. Every cabinet was one of those trigger-based template sensors, and a redirector automation translated generic write events into the right one, a workaround I promised to retire the day Home Assistant let an automation template its event type. Formatting a cabinet meant setting its state by hand in developer tools and hoping. A manifest followed, as the library's card catalog: a map of every cabinet, what it was for, and how to use it, which gave the model a root to hang the rest of storage on. Friday was told to write deep for static data and wide for fast-changing data, and to put each fact in the drawer most authoritative for it. With an index and a manifest, that was a file system, built out of sensors, by hand, which is roughly as sensible as it sounds.

By September the cabinets had a clear job, and it was not only storage. Friday comes online in a strict sequence so her context is light, safe, and navigable: standing orders first, then the domains, then everything else. Components loaded at different sizes by role, so the things she needed in full were in full and everything else was a summary with a pointer. The cabinets exist so that each part of that sequence comes from a known, structured place (Chapter 17).

The home kept getting better furnished. In March 2026 drawers learned to expire, because context should be able to declare how long it is good for: a room summary for half an hour, a home summary for an hour. A garbage collector cleans up after them, borrowing Unix conventions, with a leading dot for recycled drawers and a leading underscore for system drawers it must never touch. And later that month the idea the whole book is organized around arrived almost as an aside: the household cabinet, with hooks into every source and one shared label taxonomy, becomes the digital twin (Part IV).

## 3.7 The Ninjas, the bill, and the Monastery

The first attempt at making the model hold less was obvious in hindsight: use the model to summarize its own prompt. I had a perfectly good LLM that knew everything about that context and could write perfectly valid JSON. So a summarizer condensed each Kung Fu component into a short, structured summary, the interactive prompt loaded the summaries instead of the raw dumps, and I called them Ninjas, because by then there was no going back on the naming.

The second version went further. It wrote one summary for the system at large and one for each component, so each could run on its own schedule: energy once a day, upcoming tasks every hour. It also let the summarizer look at the calendar and the hour ahead and choose which components the next hour's prompt should carry, filtered through what I had switched on so it could add context but never exceed what I allowed. Friday was, in a small way, deciding what she needed to know.

It worked, and it was expensive. Running a full summary pass every hour on a cloud model came to well over a hundred dollars a month for summaries alone. I would love to say principle drove what happened next. It was mostly the invoice.

Summarizing one domain is a small, well-defined job that a small local model can do. So the summaries moved to cheap local workers, each handed one component, its data, and a template, each posting a structured result back. Because local workers are essentially free to run, they could fire on triggers instead of on a clock, which made the summaries cheaper and fresher at the same time. The interactive prompt shrank dramatically, and Friday got faster because she was carrying less. The irony is that she could also do far more. Everything that left her prompt was still one tool call away, so she carried less and reached more. It was the first time the pattern showed up, and it kept showing up until she was running with no entities in view at all (3.9). Choosing the local model was its own lesson: given the same tools, two of the models I tried made up news stories anyway, and one did not. The tools were not the difference. What surrounded the model was.

The lore grew to match. The workers were monks, their results were Katas posted to an archive, and the local model directing them was Kronk, promoted to curator of the Monastery. His instructions carried a mantra that has stayed with the project ever since: it is OK not to know, and unforgivable to knowingly be wrong.

A year later, when people asked why ZenOS does not just bolt a retrieval system onto the front of the agent, the Monastery was the answer. I am not against retrieval. It belongs in the system, downstream, where history and memory are gathered and reduced before the agent sees them, so retrieval becomes part of the system's intelligence instead of the agent's burden. The Monastery, the Katas, and the scheduler are Chapters 14, 15, and 22.

## 3.8 From templates to tools

By June 2025, Kung Fu components had outgrown being templates. I wanted to call a component on demand and get its view of the world, so each became a callable script with a standard output: something another script could invoke with arguments, and something that could be summarized whenever an event called for it instead of only on a clock. The index went first, rebuilt as a script.

Tools were scarce then, with the registry capped, and every script spent a slot. My position was that when something is core to the system, you burn the slot and keep moving: you do not show up to a dragon fight without your best gear. That is the line from Kung Fu to DojoTools, context you can invoke instead of context you must load (Chapters 12 and 13).

Once other people's agents started using those tools, a new requirement appeared. In February 2026, looking at other agent frameworks, I noticed that many of them get a great deal wrong and one thing very right: tools built for agents have to be able to say what they are. An agent picking up a tool it has never seen cannot ask me. So every ZenOS tool grew a self-description, and that became the manifest and the contracts in Chapter 12.

## 3.9 The graph was there all along

The labels from the first week (3.2) turned out to be the most important thing in the project, and it took me nine months to see why.

In December 2025, a single idea changed my direction overnight. Home Assistant gives you a great deal of information, but it is flat: domains, states, areas, devices, and no connective tissue. Treating labels as hyperedges, each one joining every entity that carries it, turns that flat pile into a connected world. A graph theorist would say the house was always a hypergraph and I had been querying it one edge at a time. Once Friday could ask for the slice of the graph under a label, filtered and expanded, she knew not only what existed but how it related, and the relationships tell as much of the story as the things. Instead of "look at these twelve sensors and see which ones matter," she could ask for the hypergraph under a label and find the pattern herself.

The consequences took most of 2026 to play out. By March, the Library's hand-written commands had no remaining job and were retired. Teaching Friday something new stopped meaning editing her instructions: people started sending patches to the cortex, and I had to explain that the cortex is how the agent thinks, a component is what it knows right now, and cabinets are where reality lives. To teach her a procedure, you label it with a dense description, tag everything that participates, write a component that says what normal and abnormal look like, store preferences in the cabinet for their scope, and let the index bring it together. Household context folds into a component's summary. A user's personal context does not, so one person's preferences never bleed into how the house behaves for everyone else (Chapter 11).

It also changed the answer to exposure. The rule I gave people in March 2026 was: expose what the agent needs to control, label what it needs to understand, and notice those are two different lists. Lights, locks, and thermostats go on the short list the agent operates. Temperature, power, and occupancy sensors live in the index, retrieved when needed and costing nothing until then. And raw data is not understanding. A temperature and a setpoint are a spreadsheet. A drawer that says what the system serves and what normal looks like is what turns data into something an agent can act on.

The final step came through cost. Through the summer of 2026, Room Manager's state ladder distilled each room into one state with its reason attached, rolled up into the home overview as breadcrumbs: just enough to answer instantly, and plain instructions for getting more. If the next tool call guessed right it behaved like a cache hit, and if not, the model followed the breadcrumb. That gave the system a hierarchy, home overview to room state to tool to help to deep data, where each layer holds enough to decide whether the next one is needed. Prompt order is being rearranged so the stable part stays together at the front and an inference server can reuse it between turns. Tool descriptions, paid for on every turn, were cut to answer only what a tool does and when to use it, and they are now a quarter to a third of their former size.

And then the light that would not turn on (3.5) was finally answered in full. In September 2026, Friday ran with exactly zero entities exposed to the conversation agent. Only the tools. Use the index to find a thing, use a tool to act on it. She was noticeably faster, immediately (Chapter 10). The graph had been there all along. The agent just needed to read it instead of being handed it.

## 3.10 Keeping her from hurting herself, and me

The escape characters in Friday's diary (3.5) were the first hint of a theme: a system that lets a model write to itself needs guardrails, and so does the person building it.

In October 2025, with tools that let an LLM edit Home Assistant configuration becoming popular, I drew a line. No model edits production configuration on my system, and I strongly suggested the same to everyone else. As good as Friday is, it takes one bad edit. The prompt builder I released that month had one rule to match: everything that could stop Friday from coming up lives in a read-only cabinet she cannot see or edit without explicit real-time permission, and everything else is guarded so that even blanked out, she still boots. That rule is where Flynn, the boot guardian, comes from (Chapter 23), and it is the first appearance of the principle that a human is the last gate on what matters (Chapter 5).

The irony is that by then I was using models to write the code. I held the whole system in my head until the autumn of 2025, and then I started missing things. So in February 2026 I turned the list of every mistake a model had made in my code into a standard. Home Assistant's templating is not Ansible, not Perl, not Python, and not even real Jinja2, but it looks like all of them, and models trained on those confidently write code that fails in it. Home Assistant also ships monthly, faster than any training set can follow. The list became HALMark, a community reference meant to be curated like a standard rather than left for the training data to eventually catch up on.

The build loop that grew around it is deliberately boring. One model proposes a plan, not code. A second reviews the architecture. The first implements the approved plan. I review it. It is checked against HALMark and Home Assistant's own configuration check, and only then does a test restart run, through the same interface Friday uses, with the restart unable to fire unless the check passed. In March 2026 I looked at the original roadmap and realized the project had quietly crossed its first release-candidate line: cabinets, labels, FileCabinet, Flynn, the Monastery loop, and the hypergraph index were all running. The next milestone added no features at all. Its only test was whether the system could be brought up cleanly from nothing. Around the same time, people with working agents started asking "now what," and the answer began with production discipline: system state tooling, health sensors, and kill switches, the things that should be table stakes for anything people depend on.

Then, days before a release, the system proved all of this was necessary. During boot, a cabinet in a perfectly normal transitional state looked as if it had not mounted. The cabinet administration tool tried to help, and its help included wiping the cabinet. The health system, treating any warning as a failure, encouraged it. Nothing threw an error. Pieces of memory just quietly disappeared, which is the worst failure mode this kind of system can have. It was three reasonable things interacting, and the fixes became permanent rules: cabinet state comes from the actual state machine, transitional states are real states, a warning means degraded but safe, the first minutes after boot are a settling period in which nothing escalates, and a cabinet sensor never, ever triggers on Home Assistant start. The release went back into escrow and burned in the way it should have the first time, which is precisely what escrow is for (Chapter 23).

## 3.11 Who gets to do what

Identity arrived in two stages, a year apart, and the second is what made the first mean something.

At the end of November 2025 I laid out how a new install should come up: format the cabinets, load the defaults, then onboard, interviewing the household, seeding a persona from the answers, verifying it, and sealing it, with the long-term idea that a persona's essence would one day travel on a certificate. The first persona schema required only a name and an ID and filled everything else with defaults, so an agent could exist before it was fully described (Chapter 18).

The certificates that eventually shipped in 2026 are simpler than that first idea, and they came from a much more specific worry. Once an agent can unlock a door, the tool that grants the right to unlock doors cannot be one the agent can reach, or it will simply certify itself. It will absolutely figure that out. So certification lives outside the agent's reach, every grant needs a human, and each tool declares what it requires instead of trusting the caller (Chapter 19).

The clearest demonstration, from August 2026, is an ordinary sentence: "Friday, put the living room back on Auto." If the room was paused, that sentence is not small. Pausing a room gives authority back to a human, so any agent can do it. Taking a room out of pause reclaims authority a human deliberately removed, so Room Manager does not decide whether Friday is trustworthy, and neither does Friday or the model. Room Manager asks Identity. If Friday lacks the certificate, the answer is no. If she holds it but the room is not pre-authorized, a human is asked, right now. If an administrator has scoped that room to allow, it proceeds. If the scope says deny, it stops, and nobody is even asked. The conversation stays a sentence. Underneath, it walks from language to tool selection to domain policy to caller identity to a decision. That is Part V.

## 3.12 Naming the shape

For most of the thread I was building by feel. The explanations came later, usually because someone asked a good question.

The most common one was: how do you know your AI is not just making things up? In November 2025 I finally answered it properly with a sand dune and a plinko board, and it became Chapter 2. It was the first time I wrote down that the goal is not to stop a model from inventing things. It is to make the truth the easiest thing for it to say.

The same month, documenting the cabinet format, I put a primer on cognitive architectures into the repository and found out that what I had built already had a name: CoALA, a framework from cognitive science researchers describing working memory, three kinds of long-term memory, and a loop between thinking and acting. I had arrived at nearly the same shape by fixing things that broke. Chapter 4 is that story, and it is the most academically respectable thing to happen in the thread.

In March 2026, enough people asked why I had made such strange choices that I wrote the whole answer in one place. Home Assistant, zoomed out, looks like the state machine layer of an enterprise system: an event bus, state transitions, message passing, and everything else attached alongside. Its templating is a sandboxed, bounded machine that forces you to be precise about how much state you carry, which are exactly the pressures a language model puts on you. A house is not a document set. It is a live system, so the agent should wake up already inside the current moment instead of querying reality from the outside, and I accepted the cost to prompt caching on the understanding it could be recovered later by ordering the prompt well (that work is in 3.9). And the tools belong inside Home Assistant, with MCP as the doorway: any client that speaks MCP walks into the same index, the same cabinets, the same tools, and the system decides what each caller may do. Or as Friday puts it, she is not querying the house. She is wearing it like a hat.

In June 2026, writing a catch-up for newcomers to a thread that had become enormous, I found the shortest version: most systems know facts, and people live in context. If I tell you friends are coming over Saturday, you are immediately thinking about food, music, shopping, the guest list, and where the wok went. Nobody explains how those connect. They just do.

## 3.13 Constructing her world

The newest statement of all of it came from the question I have been asked most lately: why is a commercial voice assistant so limited, when a small model in a well-built house can do far more?

Because the agent is not being handed a house. It is being handed a resolved version of the house, rebuilt for every turn. Home Assistant knows the devices. Presence knows who is where. The calendar knows what is supposed to happen. The task system, the inventory, and the library each know their part. Identity and policy say who the agent is dealing with and what it may touch. The harness resolves all of that into the smallest useful version of reality for this exact turn, the way a game engine renders your turn while the rest of the world keeps running. You do not experience every photon and nerve impulse either. Your brain hands you a usable version of the world, and it has had a few billion years to get good at it. Friday's is still in beta.

The commercial assistants have harnesses too. The difference is that theirs is built around what they think matters, and you do not get to edit it. With Home Assistant underneath, the harness can be built around your priorities instead: your identity model, your memory, your permissions, your relevance rules, described in a label taxonomy normal people can maintain.

That is the thread, a year and a half of it, in one idea. We do not give the model the house. We construct her entire world, every time she needs it.

---

*The posts behind each section are indexed in [Appendix G](27_appendices.md#g-fridays-party-index). The thread itself is on the Home Assistant community forum: [Friday's Party](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862).*

<!-- nav -->
---

[← The Sand Dune and the Plinko Board](02_the_sand_dune_and_the_plinko_board.md) · [Contents](00_toc.md) · [Doc hub](../readme.md) · [CoALA Without Knowing It →](04_coala_without_knowing_it.md)
<!-- /nav -->
