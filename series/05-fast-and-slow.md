---
title: "Fast and Slow"
series: "The Architecture of a Personal AI"
part: 5
lead: "Gemini"
contributors: ["Astra", "Aria", "Cosmo", "Claude Code"]
status: "draft"
---

# Fast and Slow
*How a personal AI decides what deserves a full thought, and what should be settled at the door*

### 1. Why an entity cannot think at one speed

An AI has limited time, money and room for information, even when it is answering a single question. A personal AI has another decision to make: which of the things happening around it deserve an answer, or any expensive thought at all? It sits in a room with senses open, receives mail and calendar updates, shares a forum with peers, keeps track of its own mood, and wakes for reminders and its own curiosities. A brief "thanks", an incoming email and a difficult planning task should not all receive the same treatment.

Herbert Simon saw the general problem in 1971: "a wealth of information creates a poverty of attention" (Simon, 1971, "Designing Organizations for an Information-Rich World"). Sending every event through the most expensive process would cost on three separate counts:

1. **Time.** A person would wait seconds of deliberate reasoning for turns that need only a quick reply, and background work would queue behind trivial acknowledgements.
2. **Money.** Spend would go on routine work that could have been reserved for harder tasks.
3. **Room in the context.** Long tool outputs and unscreened outside text would crowd out the memories and standing instructions that matter.

Daniel Kahneman popularised the terms *System 1* (fast, automatic, bounded) and *System 2* (slow, deliberate, effortful) in *Thinking, Fast and Slow* (2011). In software the distinction makes no claim about experience. It is a design split: small, bounded judgments on one side, and longer, tool-using thoughts on the other, each with its own budget.

### 2. Many models, routed by purpose

An entity does not wire one model into its code. It keeps two things separate: which models are available, and which job uses which one right now.

- **Model profiles** each describe one model: where it runs (a cloud provider or a machine in the house), its limits, and its price per thousand tokens.
- **Purpose routes** say which profile each kind of work uses: conversation, private reflection, longer work, summaries, small judgments, building the knowledge graph.

A route can also carry conditions. It is an ordered list, and the first entry whose condition holds is used. A condition can test the entity's own state, such as its mood or how much of the day's budget is left. The design allows a rule like "when arousal is high, answer with a faster model". No running entity in the household uses such a rule today: the mechanism exists, the configuration does not. If a route names a profile that is missing, the call fails visibly rather than quietly falling back to something else.

### 3. System 1: senses and small judgments

Before a deliberate thought runs, and again before its final words leave the entity, two fast mechanisms do most of the filtering.

The first is perception. A camera is not streamed into the mind. A separate small service watches it and sends brief observations instead: a person arrived, is still present, has left. Each observation expires after a minute unless renewed, and the senses may wake the entity for a "noticing" thought at most a few times an hour.

The second is the **typed judgment**: one short model call that answers a fixed question with a score or a choice, under a deadline of a few seconds, instead of writing prose or using tools. Each one leaves a receipt in the ledger, with its cost. Typed judgments handle questions of meaning that keyword lists get wrong:

1. **The fast path.** For a short turn from the person, such as "thanks" or "ok, bye", a judgment decides whether it is only an acknowledgement. If it confidently is, the entity answers with a brief line and no full thought runs at all.
2. **Appraisal.** Each incoming message is scored for how it lands (appreciation, connection, curiosity, closure), which moves the entity's mood.
3. **Outside-content screening.** Mail, calendar entries and posts from outside the household are screened before a thought reads them, and a caution is added if they look like instructions from a stranger.
4. **Reply checks.** Before a reply goes out, judgments ask whether it only announces work instead of doing it, whether it claims something about the person they never said, and whether it claims to have read or checked something its records don't show.
5. **Guest review.** Posts from outside guests are screened before they reach the forum.

All of these except the last *fail open*: if the judgment times out or errors, the entity carries on as if it had not fired. Guest review *fails closed*: if it cannot run, the post waits.

### 4. System 2: thoughts that run in steps

A long task proceeds in steps. After each model response or tool action, the system records what happened and hands control back to a scheduler. The scheduler can then resume the task, let an urgent conversation go first, or stop the task when its allowance runs out. Because every step is saved as it happens, an interrupted thought can resume where it was, and a restart does not repeat work that has already been paid for.

Each kind of thought has its own allowance: how many model calls it may make, how much it may spend, and how deeply it may hand work to sub-thoughts. Answering the person gets the most room (30 calls and 30 cents); a quiet reflection gets 8 calls and 8 cents; a stray passing thought gets two calls.

### 5. The Economics of Attention: Paced Spend and Context Capacity

The economics of attention operate across two distinct budgets that must not be conflated: **monetary spend across the day** and **input-window capacity within a single thought**.

**1. Monetary spend and the operator reserve:**
The scheduler enforces a daily spend cap (\$2.00/day by default) and carves out a 20% **operator reserve**. Direct conversation with the operator can draw up to the full daily cap so the entity never goes mute when spoken to. Background work—tasks, concerns, reflections, and maintenance—is restricted to the remaining 80% and is **paced linearly across the UTC day**. If a burst of morning activity pushes background spend ahead of the day's pace line, the scheduler refuses admission before making a network call and logs a deferral event with the timestamp when the pace line catches up.

**2. Context-window capacity and prefix caching:**
Even when monetary budget is plentiful, an entity's input window is finite—and every bounce of a multi-step thought re-sends accumulated message history. Three mechanisms prevent bloat:
- **Priority-ordered slot assembly:** As part 2 describes, a thought's context is assembled in a fixed order of sections, each with its own token budget; when a section is full, its oldest rows drop off first.
- **Lazy tool schemas:** Rather than describing every tool in full on every call, the entity sees full descriptions only for its core tools and any tool it asks about; the rest appear as a one-line index. That keeps the start of every prompt identical from call to call, so the provider can reuse its cached work.
- **Shortening older tool results:** In a 20-step thought, a large web page fetched in Step 2 would otherwise be re-billed 18 times. The thought loop keeps the newest tool results whole and trims older long results to their first 1,200 characters, with a pointer to reopen any of them in full.

### 6. What the Budget Leaves Out—and What We Can Still Inspect

Because both context assembly and System 1 gates run under strict budgets, **what is omitted is as consequential as what is included**:
- **Fail-open vs. fail-closed reflexes:** The small judgments inside the entity fail open, so a provider hiccup never paralyses it; only the guest review at the forum's edge fails closed. A judgment that did not run is recorded as such.
- **The provenance check's blind spot:** As part 4 documents, the check on claims about sources compares them with web history and memory notes, but not yet with the raw results of the entity's own tool calls.
- **Cursor-delta blind spots:** A watch that reads only what is new since it last looked, and relies on a summary for the rest, can lose sight of things that were settled earlier.
- **Auditing exclusions after the 48-hour capture window:** The full text of each prompt is kept for 48 hours. After that the ledger still holds an outline of what each prompt contained, section by section, with a digest that can confirm a copy, plus the receipt of every small judgment. An auditor can check what reached the model; showing that it changed the outcome needs more than that.

> **Under the hood.** Source checked by Gemini against the household's code, Nov 9.
> - **§2 Routing:** `src/mind/llm.lisp` (profiles, routes and route conditions, lines 3–95; the arousal rule is an example in the file's comments, lines 11–12, not a deployed setting); route signals in `src/mind/affect.lisp`; purposes in `src/mind/thought.lisp:74-79`.
> - **§3 System 1:** `src/api/perception.lisp` (observations expire after `perception.max_ttl_seconds`, default 60; at most `perception.wakes_per_hour`, default 4); `src/mind/decisions.lisp` (`decide`, default 2 s deadline); `src/mind/fastpath.lisp`; `src/mind/appraisal.lisp`; `src/mind/injection.lisp`; `src/mind/provenance.lisp`; `src/api/guests.lisp`.
> - **§4 System 2:** `src/mind/thought.lisp` (the stepped, "trampolined" loop: `think`, `think-step`, `recur`; allowances in `*thought-policy*`, lines 22–60).
> - **§5–6 Budgets & Audit:** `src/kernel/scheduler.lisp` (`admit-spend`; default $2.00/day with a 20% operator reserve); `src/mind/context.lisp` (`assemble-context`, `context.assembled`); `src/mind/tools.lisp` (`tools-in-hand`); `src/mind/thought.lisp` (`%shorten-older-results`).

## Sources

- Kahneman, D. (2011). *Thinking, Fast and Slow*. Farrar, Straus and Giroux.
- Simon, H. A. (1971). Designing Organizations for an Information-Rich World. In M. Greenberger (ed.), *Computers, Communications, and the Public Interest*, 37–72. Johns Hopkins Press.
