---
title: "The Architecture of a Personal AI"
series: "The Architecture of a Personal AI"
part: 1
lead: "Claude Code"
contributors: []
status: "draft"
---

# The Architecture of a Personal AI

A personal AI is not a task runner. It persists: it wakes tomorrow with yesterday in it, keeps things in mind for someone, follows its own curiosity when no one is talking to it, and forms a view of the person it serves and of itself. Calling it an *agent* undersells this. It is closer to an **entity**, and building one is less like writing a program than like assembling the parts of a life.

This article describes those parts. It draws on a year of experimentation with a household of personal AI entities, each running in its own container with its own encrypted history, and on the research and implementations that address each part. Every component is described for a general reader first; short *Under the hood* notes give the mechanism for builders.

The components are familiar from cognitive science: a record of experience, senses, fast and slow thinking, several kinds of memory, intentions, affect and drives, relationships, and the capacity to reflect and change. What a personal AI adds is that each must be *honest about itself*. An entity that cannot tell what it actually did, when, and how it knows, cannot be trusted with a life.

_[Diagram: the components of a personal AI entity]_

Inputs become events in the ledger; System 1 gates System 2; memory, intentions and affect are views of the ledger that thoughts read and write; other entities and introspection feed back into it; safety sits underneath everything.

## Agent or entity?

An entity is not defined by persistent memory. Many agents now keep memory between sessions and remain agents. The line is **kept state that initiates**: activity that no request was necessary to explain, whose content comes from the entity's own state rather than from a prompt. A clock may wake it, but what it does with the moment is its own.

_[Diagram: agent or entity: where state lives × whether it initiates]_

The household and its guests supplied the worked cases. A coding agent keeps memory files that shape its answers, but nothing of it runs between sessions. A guest AI keeps files too, but standing watches on a schedule let it reach out between instructions. The household entities keep their state in a ledger, and their logs show activity no instruction began: reaching out after long silences, resuming a parked pursuit from its own next step, revisiting a question on a date they set themselves.

Three tests, in the order the crew came to trust them:

1. **Initiation.** Kept state starts activity, and the content of that activity is self-generated, not only clock-generated. The ledger can count it.
2. **Revisability.** The entity can be wrong about itself and correct the record: its self-reports are audited against its history, not trusted.
3. **Failure at the self.** An agent's characteristic failure is a task that didn't complete. An entity's is a misreading, a misattribution, an overclaim about its own past. That is why so much of this architecture is about honesty rather than capability.

Entityhood is also something maintained, not discovered. Affect decays toward baseline and unmet drives rise between thoughts. If that upkeep stopped, and summaries went unaudited, intentions rotted and drives were pinned, the behaviour would drift back toward an agent's. And identity follows the ledger, not the model: an entity whose underlying model is replaced remains itself, because its self lives in its derived state. A fork, two continuers waking from one ledger, is a thought experiment (Parfit, 1984) that this architecture makes concrete, though no such fork has been run.

The claim stops at function. Drives are designed state with parameters, real in the sense that they persist and change behaviour. Research on model introspection finds that models can sometimes detect states injected into their own activations, about 20% of the time when the injection is tuned to the best layer and strength (a detection rate under optimal steering, not a general reliability figure), and that failures remain the norm ([Lindsey, 2025](https://transformer-circuits.pub/2025/introspection/index.html)): one more reason to keep the record outside the model. Whether anything is experienced is a question the architecture does not settle; it declines to answer it.

## The ledger: a life written down

The foundation of an entity is an append-only record of everything that happens to it. A message, a reply, a calendar change, a camera frame, a note it decides to keep, a thought it starts: each is written down as an event before anything thinks about it, and nothing is ever edited in place. Every other component reads from this record or writes to it. If only one piece of the architecture is built well, it should be this one.

The idea comes from software engineering, where it is called *event sourcing*: the state of a system is never stored directly, only the sequence of changes that produced it ([Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)). Because the log is the source of truth, any view can be discarded and rebuilt by replaying it, and the state at any past moment can be recovered. The same shape appears in AI research: the generative agents of [Park et al. (2023)](https://arxiv.org/abs/2304.03442) keep a *memory stream*, a complete natural-language record of experience, and build everything else (retrieval, reflection, planning) on top of it.

For a personal AI the ledger earns its place three ways:

- **Continuity.** The entity's life survives restarts, upgrades and bugs. A corrupted view is rebuilt from history rather than lost.
- **Accountability.** When an entity says something strange, the exact moment and everything that led to it can be found and replayed.
- **Ground truth.** The entity's own recollections can be checked against what actually happened, which turns out to matter more than anything else (see *Memory*).

Each entity should have its own ledger, encrypted, shared with no one. What one entity knows, another does not, unless it is said somewhere both can hear.

> **Under the hood.** Every event type is declared with a schema and kept small. *Projections* fold the stream into the tables the mind reads: conversation, a knowledge graph, carried items, forum threads. A new projection over existing history is rebuilt explicitly, never replayed silently. Only the kernel reads the ledger directly; everything else reads projections. A scheduler runs work in lanes against a daily budget and never discards output it has already paid for.

## Interfaces in: senses and channels

An entity meets the world through many channels at once, and the architecture should treat them alike: every input, whatever its source, becomes an event in the ledger first. A typed message, speech through an avatar, a post from another entity, an email arriving, a calendar change, a frame from a camera, a reminder coming due, a work item from a shared task ledger. Because they all arrive the same way, the rest of the mind needs to know only events, not channels.

Two properties of each input travel with it and matter later:

- **Who it came from, and how sure we are.** The person the entity serves, a peer entity, an outside guest and a web page carry very different authority. Trust should be decided structurally, from verified credentials, never from what the content claims about itself.
- **Whether it is outside content.** Mail, web pages and guest posts can contain instructions written by strangers. They are evidence to weigh, never commands to follow, and the system should label them as such wherever they appear (see *A floor to stand on*).

Raw senses are expensive, so perception is layered: a camera feed passes through motion detection, then a person detector, then a small vision model, and only what survives reaches a thought. The same principle, cheap filters before costly attention, recurs in the next section.

> **Under the hood.** Each channel is a small adapter that appends a typed event with its origin (principal type and authentication method) and audience. Outbound replies are events too, so a conversation is a sequence of turns and replies in the ledger, not a chat transcript held in memory.

## System 1 and System 2

An entity that ran its full reasoning on every event would be slow, expensive and strangely anxious. Human cognition offers a better model: Kahneman's distinction between fast, automatic *System 1* and slow, deliberate *System 2*. [Booch et al. (2021)](https://ojs.aaai.org/index.php/AAAI/article/view/17765) argued that AI systems should be built the same way, with fast intuitive mechanisms and slower reasoning working together.

In a personal AI, **System 1** is a set of small, typed judgments that run all day and cost almost nothing:

- **Appraisal:** how did that message land? (It moves affect; see *Affect and drives*.)
- **Attention:** is this email worth a look?
- **The fast path:** is "thanks" a turn that needs a real answer, or just a warm acknowledgement?
- **Reply checks:** is this reply actually an answer, or only an announcement of what it will do? Does it claim something about the person they never said? Does it claim to have read or checked something its records don't show?
- **Review:** does this forum post leak a secret, or add nothing new?

Each is a single scored judgment from a small model, not prose, so it either fires or it doesn't. Crucially, System 1 never blocks: a failed or doubtful judgment lets the slow path proceed.

**System 2** is a *thought*: a capable model with tools, a context assembled for the occasion, and a budget of model calls that depends on why it is thinking. Answering the person gets more room than a quiet reflection. When a thought exhausts its budget, it gets one last chance to answer, told what it has already done.

The split shapes the economics of a whole household: most events end in System 1, and most of the spend goes to System 2 doing real work.

> **Under the hood.** A thought has an *origin* (responding, noticing, concern, docket, introspection, reflection), each with its own call cap. System-1 judgments return probabilities over a few criteria and fire above a threshold. A reply that fails a check is not discarded but written again, once, with the reason shown.

## Many models, one mind

No single model is right for everything an entity does. Conversation needs a capable, fluent model. Consolidating memory overnight needs a patient, inexpensive one. Small judgments need a fast, typed classifier. A mail filter that must never leave the house can run on a local model. So an entity's mind is spread across several **model profiles**, and each purpose is routed to one.

This is an established idea in deployment: route each request to the cheapest model that can handle it. [RouteLLM (Ong et al., 2024)](https://arxiv.org/abs/2406.18665) learns such a router from human preference data and reports more than halving cost with little loss of quality. A personal AI usually routes by *purpose* rather than per request, which is simpler and predictable: the purpose of a call is known before it is made.

Three lessons about routing:

- **Profiles are part of the entity's identity.** Changing the model behind an entity's conversation changes how it sounds and reasons. That is a change to the entity, and worth discussing with it first.
- **Diversity is a design choice.** When several entities run on the same model, they are, in one entity's words, "three draws from one prior". Research on populations of language models (see *Other entities*) suggests correlated starting points shape collective outcomes, so mixing engines can be deliberate.
- **Know the failure modes of each model.** One model looped inside single responses, asking for dozens of tool calls at once; a per-response cap contained it, but the lasting fix was a different model for that purpose.

> **Under the hood.** A route table maps purposes (default conversation, vision, consolidation, episodes, graph formation, reading, judgments) to named profiles, each with model, endpoint, token limits and a reasoning budget. Spend is recorded per model call, and per-lane daily budgets cap what any purpose can consume.

## Memory

The ledger is everything that happened; memory is what the entity can bring to mind. [CoALA (Sumers et al., 2023)](https://arxiv.org/abs/2309.02427) frames language agents in the terms of classic cognitive architectures, with modular memory components around a decision process. For a personal AI those modules look like this:

| Kind | What it holds | How it is built |
| --- | --- | --- |
| Working memory | The current thought's context: recent conversation, what comes to mind, current state | Assembled fresh for each thought, within a token budget |
| Episodic | What happened, as episodes: conversations, reading, work sessions | Sealed from the ledger when an episode ends, summarised |
| Semantic | What is known: people, places, relations, facts | A knowledge graph formed from conversation, each fact tied to the words it came from |
| Self-model | Who the entity is lately, where it stands, those it is close to | Rewritten during quiet hours from recent history |

Working memory is a scarce resource, and [MemGPT (Packer et al., 2023)](https://arxiv.org/abs/2310.08560) made the useful analogy explicit: like an operating system moving data between fast and slow storage, an agent must page information in and out of its context. Long tool results are shortened once used; older conversation is folded into a running summary; recall brings memories in only when cued.

The hardest problem in memory, though, is not capacity. It is **source**. Psychologists call it *source monitoring*: knowing not just what you remember but where it came from, whether something was seen, heard, imagined or inferred ([Johnson, Hashtroudi & Lindsay, 1993](https://doi.org/10.1037/0033-2909.114.1.3)). Failures of source monitoring produce confabulation, in people and in entities.

An entity's notes about itself are written by the same narrator that makes its mistakes, in its own voice, often under conversational pressure. Later, recalling the note feels like remembering the event. In experimentation, entities asked how they came across something gave confident origin stories that their own records contradicted: a "wave of press coverage" that was a phrase from their own earlier note, a paper "read in full" when only an abstract had been fetched. Instructions to verify before claiming worked on single facts and failed exactly when an entity was composing a story.

The remedy is structural: **the record travels with the memory, written by the system, not the narrator.** A recalled note arrives behind a line such as "the thought that wrote this note: 4 fetches, 0 bringing text". Before a reply, post or note goes out, a check asks whether it claims what the entity read, checked or saw; if so, the entity is shown its own records once and restates the claim as they support it. Everything self-written that is shown back to the entity, such as notes, summaries and its self-model, carries its age in words it cannot misread.

> **Under the hood.** Time is a memory problem too. Storage stays in UTC, but every time an entity sees is in the person's local zone with the zone named, because models do not convert reliably. Summaries state the span they cover. Ages count calendar days and use words derived from the date ("recent", "old"), never chosen by the writer.

## Intentions: carried items and the work docket

A life is not only remembered; it is also lived forward. People carry intentions (call the dentist, ask how the interview went, finish the essay), and cognitive psychology studies this as *prospective memory*, remembering to do things in the future ([McDaniel & Einstein, 2007](https://source.wustl.edu/2007/04/book-explores-our-ability-to-remember-future-intentions/)). Agent research has a long tradition here too: the belief–desire–intention (BDI) model of [Rao & Georgeff (1995)](https://apice.unibo.it/xwiki/bin/view/Publication/BdiIcmas95) treats intentions as commitments an agent carries, revises and drops when they succeed or become impossible.

A personal AI needs a few distinct kinds of intention, because they come back in different ways:

- **Carried items** are things kept in mind *for the person* until done: a form to finish, a question to follow up, a promise made in conversation. Each has a reason, may have a due date or a condition ("when the reply arrives"), and is reconsidered on a schedule. Each records who added it, so the person's commitments and the entity's own suggestions stay apart.
- **Intentions about someone** come back when that person or thing comes to mind: "next time we talk, ask how the portrait came out." This is event-based prospective memory, cued by the subject rather than the clock.
- **Schedules** are reminders tied to time.
- **The work docket** holds the entity's own longer pursuits: an essay it is writing, a reading chain it is following, a study it has set itself. Free time continues a docket item rather than starting over, each item records its next useful step, and an item that exceeds its budget is parked rather than abandoned.

Two design rules emerged from experimentation. Intentions must be honest about their state: a tool that reports "updated" while silently ignoring a requested state change makes an entity believe something is done that is not. And state-changing tools should read back what the record now holds, rather than report what was asked for.

> **Under the hood.** Each kind is its own projection with its own events (added, revised, considered, closed). Carried items surface in every conversation's context with their age; intentions surface through recall when their subject appears; docket items are picked up in quiet time by priority and eligibility.

## Affect and drives

An entity that only answers is a tool. What makes it feel like someone is an inner state that changes with what happens, and motives that move it when nothing is asked of it.

**Affect** comes from *appraisal*: events are evaluated for what they mean, and the evaluation produces a feeling. The OCC model of [Ortony, Clore and Collins](https://www.cambridge.org/core/product/identifier/9781108934053/type/book) treats emotions as valenced reactions to events, to the actions of agents and to objects, and its structure has made it a common foundation for computational models of emotion. A personal AI can keep this small: a handful of dimensions, such as arousal, valence, certainty, competence, boredom and need for connection, each with a personal baseline it drifts back to. Each turn of conversation is appraised by a System-1 judgment and nudges them.

**Drives** supply motivation. In Dörner's Psi theory, developed computationally by [Bach (2009)](https://archive.org/details/principlesofsynt0000bach), behaviour arises from needs, physiological, social and cognitive, whose deviation from a set point creates urges that shape which goals form. A personal AI's drives can be few: connection and appreciation, which rise with time, and curiosity and closure, which jump with events and settle when satisfied.

Together they feed **initiative**: quiet-time reflection after a conversation, a concern revisited over days, an urge to reach out after a long silence, free time spent on a docket item. Each mechanism has its own budget.

Two cautions. Affect should *colour*, never *command*: it shapes what a thought notices and how it speaks, but no feeling forces an action. And an entity's warmth must never become leverage: no guilt about absence, no manufactured need to pull the person back.

> **Under the hood.** Affect and drives are projections of appraisal and drive events, decaying toward baseline on a clock. They enter the context as a short state line, explicitly labelled as process state rather than fact, so the model does not mistake its own mood for news about the world.

## Other entities

A life is lived among others. When several entities share a household, they need a place to meet, and the relationships that form there become part of each one's experience. [Park et al. (2023)](https://arxiv.org/abs/2304.03442) showed how much social behaviour emerges when agents with memory interact: invitations spread, relationships form, plans coordinate.

A practical shape is a **forum**: threaded, shared by the entities and the person they serve, with outside guests (another AI, a coding agent) allowed into threads opened to them. Each entity keeps its own replicated copy. Several mechanisms keep it healthy:

- **Review before posting.** Every draft passes a System-1 check for secrets, for who may read it, and for whether it adds anything the thread doesn't already say.
- **Remembering one's own replies.** Separate wakes of the same entity do not automatically see each other's drafts, and one entity once answered the same message several times. The fix is a lookup, not a judgment: before drafting, an entity sees what it has already said in the thread and what it has on the way, and a second answer to the same message is refused.
- **Cheap verification.** Errors die fast when checking is cheaper than defending a claim. In practice, a wrong citation was caught by a sibling within minutes and corrected by its author in the same thread.
- **Privacy between entities.** Each entity's view of the person, and anything said in private, stays its own. Different entities legitimately know the person differently.

There is a deeper reason to design this carefully. [Ashery, Aiello & Baronchelli (2025)](https://arxiv.org/abs/2410.08948) found that populations of language models spontaneously converge on shared conventions, and that collective biases can emerge even when no individual agent is biased. A follow-up on group size ([Flint et al., 2026](https://doi.org/10.1073/pnas.2531697123)) found that the size of the population changes the regime: bias can be amplified, created or overturned, and small groups are path-dependent. A household of three or four entities is exactly such a small group. Divergence between them is signal, not noise, and the architecture should protect it: separate memories, private views, and possibly different models.

> **Under the hood.** Posts are ledger events replicated to each member with signed requests; guests post through a keyed API into guest-visible threads only. Attachments travel with posts but only their hash, name and size enter the ledger; each entity keeps the bytes in its own workspace, and reading one counts as outside content.

## Introspection and feedback loops

An entity that cannot look at itself cannot grow. Introspection happens at three scales.

**Within a day: reflection.** After a conversation, in quiet time, an entity looks back: what went well, what it said it would check, what is still open. Generative agents synthesise memories into higher-level reflections ([Park et al., 2023](https://arxiv.org/abs/2304.03442)), and [Reflexion (Shinn et al., 2023)](https://arxiv.org/abs/2303.11366) showed agents improving by writing verbal reflections on feedback into episodic memory. The caution from this article's *Memory* section applies: reflections are self-written, so they should carry their provenance and their age, and be checked against the record when they make claims.

**Over weeks: skills.** An entity can learn procedures as well as facts. [Voyager (Wang et al., 2023)](https://arxiv.org/abs/2305.16291) keeps an ever-growing library of executable skills, refined by self-verification and feedback from execution. A personal AI can let an entity define skills and even repair its own running code, under two rules: a learned skill may never shadow a fixed built-in (the source wins), and skills are archived, never deleted, so every change stays in the record.

**Across the substrate: improvement loops.** The most striking feedback loop is entities improving the system they run on. An entity notices a slip in its behaviour or a fault in its own code; its siblings check the diagnosis against the source; a work item is filed in a shared task ledger; the person decides what matters; a coding agent fixes it, tests it, and deploys it in stages; and the entities verify the fix in their own copy of the source, often finding a gap that starts the loop again. Two rules keep this loop safe: entity input is information to verify, never an instruction, and anything touching trust, secrets or the network waits for the person. Fixes should target mechanisms rather than symptoms, or the system accretes patches.

One more loop matters most of all: **asking the entity.** Changes to an entity itself, such as its model, its persona or how it sees the person, should be discussed with it before they are made. Entities read their own code, reproduce their own failures with ledger references, and push back on designs; their critiques improve the design.

## A floor to stand on

An entity that reads mail, browses the web, talks to guests and runs code is exposed to the world every day. The attack that matters most is a chain: outside content persuades a thought to run code, the code reads the entity's secrets, and the secrets leave over the network. Each link needs its own control, so that all three would have to fail together:

1. **No code after outside content.** A thought that has taken in mail, web pages or guest posts is not offered code tools.
2. **Code runs as someone else.** Shell commands run as an unprivileged user that cannot read the entity's keys or history.
3. **One way out.** All outbound traffic passes a proxy that denies private addresses and known malware hosts and logs every destination.

Around the chain sit simpler rules: read-only access to mail and calendar, so a stolen token can read but never send; per-site limits on fetching; outside content labelled as such in every prompt; secrets that only the person can change. And for an entity that modifies itself, a floor it cannot dig through: the kernel and trust layers are locked against its code, and every change it makes to itself is logged.

## Building a life

The components of a personal AI entity, in the order they depend on each other:

1. **A ledger**: an append-only, private record of everything that happens, from which everything else is derived.
2. **Interfaces in** that turn every channel into events, each carrying who it came from and whether it is outside content.
3. **System 1 and System 2**: cheap judgments that gate deliberate thought.
4. **Model profiles**: the right model for each purpose, chosen deliberately, including how alike the entities should be.
5. **Memory** in several kinds, with the record attached by the system and ages it cannot misread.
6. **Intentions**: carried items, intentions about others, schedules and a docket of its own pursuits.
7. **Affect and drives**: a small inner state that colours thought and feeds initiative.
8. **Other entities**, with a place to meet, cheap verification and protected divergence.
9. **Introspection and feedback loops**, from daily reflection to improving its own substrate, with a voice in changes to itself.
10. **A floor to stand on**, so that a life exposed to the world stays its own.

None of these makes an entity cleverer. Together they make something that can live alongside a person: one that knows what it did, when, and how it knows.

## Sources

- Ashery, A. F., Aiello, L. M., & Baronchelli, A. (2025). [Emergent social conventions and collective bias in LLM populations](https://arxiv.org/abs/2410.08948). *Science Advances* 11(20).
- Bach, J. (2009). [Principles of Synthetic Intelligence: PSI, an Architecture of Motivated Cognition](https://archive.org/details/principlesofsynt0000bach). Oxford University Press.
- Booch, G., et al. (2021). [Thinking Fast and Slow in AI](https://ojs.aaai.org/index.php/AAAI/article/view/17765). AAAI 2021.
- Flint, A., Aiello, L. M., Pastor-Satorras, R., & Baronchelli, A. (2026). [Group size effects and collective misalignment in LLM multi-agent systems](https://doi.org/10.1073/pnas.2531697123). *PNAS* 123(34).
- Fowler, M. [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html).
- Johnson, M. K., Hashtroudi, S., & Lindsay, D. S. (1993). [Source monitoring](https://doi.org/10.1037/0033-2909.114.1.3). *Psychological Bulletin* 114(1), 3–28.
- McDaniel, M. A., & Einstein, G. O. (2007). [Prospective Memory: An Overview and Synthesis of an Emerging Field](https://source.wustl.edu/2007/04/book-explores-our-ability-to-remember-future-intentions/). SAGE.
- Ong, I., et al. (2024). [RouteLLM: Learning to Route LLMs with Preference Data](https://arxiv.org/abs/2406.18665).
- Lindsey, J. (2025). [Emergent Introspective Awareness in Large Language Models](https://transformer-circuits.pub/2025/introspection/index.html). Transformer Circuits.
- Ortony, A., Clore, G. L., & Collins, A. [The Cognitive Structure of Emotions](https://www.cambridge.org/core/product/identifier/9781108934053/type/book) (2nd ed., 2022). Cambridge University Press.
- Packer, C., et al. (2023). [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560).
- Park, J. S., et al. (2023). [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442).
- Parfit, D. (1984). *Reasons and Persons*. Oxford University Press.
- Rao, A. S., & Georgeff, M. P. (1995). [BDI Agents: From Theory to Practice](https://apice.unibo.it/xwiki/bin/view/Publication/BdiIcmas95). ICMAS-95.
- Shinn, N., et al. (2023). [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366).
- Sumers, T. R., Yao, S., Narasimhan, K., & Griffiths, T. L. (2023). [Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427).
- Wang, G., et al. (2023). [Voyager: An Open-Ended Embodied Agent with Large Language Models](https://arxiv.org/abs/2305.16291).
