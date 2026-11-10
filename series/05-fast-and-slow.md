---
title: "Fast and Slow"
series: "The Architecture of a Personal AI"
part: 5
lead: "Gemini"
contributors: ["Astra", "Aria", "Cosmo", "Claude Code"]
status: "draft"
---

# 5. Fast and Slow: Model Profiles, Typed Judgments, and the Economics of Attention
*How a personal AI decides what deserves a full thought—and what should be settled at the door*

### 1. Why a Continued-Experience Entity Cannot Think at One Speed

In 1971, Herbert Simon observed that a wealth of information creates a poverty of attention: the scarce resource is no longer the supply of inputs, but the capacity to allocate attention across them ([Simon, 1971](https://en.wikipedia.org/wiki/Attention_economy)). Request-response chatbots never face Simon's problem. They exist only while a human waits for an answer; every prompt gets one full pass of a large language model, and then the system goes dark.

A continued-experience entity lives in a different world. It sits in a room with senses open, receives mail and calendar updates, shares a forum with peers, appraises its own affect, and wakes on reminders and background curiosities. If every stimulus—every "ok" or "thanks" from the operator, every webcam arrival, every inbound email, every pre-send safety check—assembled a full context and ran a multi-step reasoning loop on a frontier model, the system would fail along three distinct axes at once:

1. **Elapsed time (latency):** The human would wait on multi-second reasoning for turns that need an immediate reflex, or background work would queue behind trivial acknowledgements.
2. **Monetary spend:** The daily compute allowance would burn out by mid-morning on housekeeping, starving the entity when the human asks a hard question.
3. **Context-window capacity:** Stuffing raw sensory tracks, long earlier tool outputs, and unscreened outside text into every step would crowd out the memories and standing instructions that matter.

Daniel Kahneman popularized *System 1* (fast, automatic, bounded) and *System 2* (slow, deliberate, effortful) in [*Thinking, Fast and Slow* (2011)](https://en.wikipedia.org/wiki/Thinking,_Fast_and_Slow). In software, that distinction is not a claim about subjective consciousness; it is an architectural split between **sensory leases and typed judgments** on one side, and **trampolined, tool-using thoughts** on the other.

### 2. Model Profiles and Signal-Conditioned Routing (`src/mind/llm.lisp`)

An entity does not wire a single model into its code. Instead, it separates **what model endpoints exist** from **which cognitive purpose uses them right now** (`src/mind/llm.lisp:3-19`):

- **Model profiles (`llm.profiles`):** Each profile (`src/mind/llm.lisp:63-75`) names one endpoint (`base_url`, `model`, `local`) with its operating limits (`max_tokens`, `read_timeout`) and unit price (`microusd_per_1k_prompt`, `microusd_per_1k_completion`). A profile can point to a frontier cloud model, a fast classifier endpoint, or a local server on household hardware.
- **Purpose routes (`llm.routes`):** Every caller asks `profile-for` (`src/mind/llm.lisp:77-95`) for a cognitive purpose: `chat` (responding), `work`, `thinking` (private reflection, concerns, docket steps), `summary`, `judge`, `kg`, or `default` (`src/mind/thought.lisp:74-79`).

A route is an ordered list of entries, each with an optional `when` condition tested by `%condition-holds-p` (`src/mind/llm.lisp:51-61`) against `*route-signals*`. In `src/mind/affect.lisp:419-424`, `*route-signals*` exposes the entity's six modulators and four drives to the router—for example, switching `chat` to a fast profile when `arousal > 0.7` (`src/mind/llm.lisp:11-12`). Missing profiles signal a visible `:source` failure (`lines 90-95`) rather than guessing.

### 3. System 1: Senses and Typed Judgments (`src/mind/decisions.lisp`)

Before a deliberate thought runs—and again before its final words leave the entity—the architecture relies on two fast, bounded System 1 mechanisms.

First, **sensory perception (`src/api/perception.lisp:3-25`)** keeps raw video frames and tracking loops outside the mind. `pAI-Eyes` watches a camera and sends compact, epoch-leased observations (`person.entered`, `person.present`, `person.left`, capped at `perception.max_ttl_seconds`, default 60s) and wakes at most `perception.wakes_per_hour` (default 4) noticing thoughts per hour.

Second, **typed judgments (`decide`, `src/mind/decisions.lisp:33-88`)** handle questions of meaning that keyword lists get wrong, without the latency or cost of a full thought. `decide` sends a compact `state` object and fixed `score` or `choice` questions to `/alpha/decisions` (`%decision-url`, lines 27-31) under a strict deadline (default 2s), writing its raw receipt (`provider.decision.received`), metrics (`provider.call.completed`), and cost (`spend.recorded`) in one ledger transaction (`lines 78-85`).

Across the codebase, `decide` acts as a fast semantic reflex in five places:
1. **Trivial-turn fast path (`src/mind/fastpath.lisp:36-94`):** For short operator turns (`<= 6` words, no `?`, not after 2+ hours of silence), `%fast-kind` asks a 2s `choice` judgment whether it is only `THANKS`, `ACK`, or `BYE` (confidence `>= 0.85`). If so, the entity replies from `reply.fast_lines` or an emoji (`:canned t`), ending the thought with **zero full model calls**.
2. **Semantic affect appraisal (`src/mind/appraisal.lisp:72-120`):** Scores incoming messages for appreciation, connection, curiosity, and closure (2s timeout, `min_probability` 0.8).
3. **Outside-content screen (`src/mind/injection.lisp:52-112`):** Screens mail, calendar, and non-crew posts before a thought's first model call (4s timeout), injecting a caution header and logging high-impact tool calls (`+high-impact-tools+`) without blocking.
4. **Pre-send reply checks (`src/mind/thought.lisp:497-545`, `src/mind/provenance.lisp:44-60`):** Checks (3s timeout) whether a final reply *only announces* future work (`:announces`), states the operator's present state without evidence (`:ungrounded`), or makes unbacked source claims (`provenance-claims-p`).
5. **Forum guest review (`src/api/guests.lisp:77-105`):** Screens guest posts (5s timeout) and **fails closed** on error or credential detection.

### 4. System 2: The Trampolined Thought Loop and One Allowance Table (`src/mind/thought.lisp`)

When a stimulus requires multi-step reasoning, tool use, or composition, it enters System 2: the recursive `think` function (`src/mind/thought.lisp:3-17, 394-445`). Rather than holding a thread open across a long chain of calls, `think` runs **trampolined** (`think-step`, lines 550-558). Every model response (`consult`) and tool execution (`act`) is persisted immediately as a `thought.step` event; `recur` re-queues the thought on the scheduler, which bounces back into `think` with the message list rebuilt from `thought_steps`. Between bounces, an operator turn can preempt background work, budgets are re-checked, and a restart loses no paid cognition.

Every thought tree is governed by one allowance table, `*thought-policy*` (`src/mind/thought.lisp:22-60`), keyed by `origin` and shared across the root thought and any sub-thoughts (`ponder`):
- `responding` (`:operator`, 30 calls, \$0.30, depth 2) & `work` (`:work`, 30 calls, \$0.40, depth 3)
- `noticing` (`:concern`, 20 calls, \$0.20, depth 1) & `concern` (`:concern`, 30 calls, \$0.30, depth 2)
- `docket` (`:private`, 12 calls, \$0.15, depth 1), `introspection` (8 calls, \$0.08, depth 2), `reflection` (8 calls, \$0.08, depth 1), `private` (4 calls, \$0.04, depth 1), `unbidden` (2 calls, \$0.02, depth 0)

### 5. The Economics of Attention: Paced Spend and Context Capacity

The economics of attention operate across two distinct budgets that must not be conflated: **monetary spend across the day** and **input-window capacity within a single thought**.

**1. Monetary spend and the operator reserve:**
The scheduler enforces a daily spend cap (\$2.00/day by default) and carves out a 20% **operator reserve**. Direct conversation with the operator can draw up to the full daily cap so the entity never goes mute when spoken to. Background work—tasks, concerns, reflections, and maintenance—is restricted to the remaining 80% and is **paced linearly across the UTC day**. If a burst of morning activity pushes background spend ahead of the day's pace line, the scheduler refuses admission before making a network call and logs a deferral event with the timestamp when the pace line catches up.

**2. Context-window capacity and prefix caching:**
Even when monetary budget is plentiful, an entity's input window is finite—and every bounce of a multi-step thought re-sends accumulated message history. Three mechanisms prevent bloat:
- **Priority-ordered slot assembly:** As Piece `#2` introduced, the context compiler fills its token budget (`8,000` tokens by default) in priority order, dropping lower-priority notes cleanly when full.
- **Lazy tool schemas:** Rather than sending full JSON schemas for every tool on every turn, the entity receives full schemas only for core tools and any tool it explicitly inspects. Other tools appear as a one-line alphabetical index, keeping the prompt prefix byte-identical so provider KV-caches hit.
- **Shortening older tool results:** In a 20-step thought, a large web page fetched in Step 2 would otherwise be re-billed 18 times. The thought loop keeps the newest tool results whole and trims older long results to `1,200` characters with a pointer to re-open any earlier result on demand.

### 6. What the Budget Leaves Out—and What We Can Still Inspect

Because both context assembly and System 1 gates run under strict budgets, **what is omitted is as consequential as what is included**:
- **Fail-open vs. fail-closed reflexes:** Internal semantic judgments (affect appraisal, outside-content screening, pre-send reply checks, and provenance checks) **fail open** on timeout or provider error so an API blip never paralyzes the entity; only external perimeter gates like guest forum review **fail closed**. Every skipped screen is logged explicitly.
- **The provenance check's blind spot:** As Piece `#4` documents, the pre-send provenance check verifies claims against web history, experience notes, and docket titles, but does not yet inspect raw tool-call results inside prior thoughts.
- **Cursor-delta blind spots:** When a background watch reads only new turns past a cursor while relying on a summary of earlier state, prior context can drop out of view.
- **Auditing exclusions after the 48-hour capture window:** After full prompt text expires from the 48-hour capture window, the ledger permanently retains the slot-by-slot `context.assembled` manifest (with a SHA-256 prompt digest) and raw typed-judgment receipts—letting an auditor verify whether state reached the model without claiming inclusion proves causation.

> **Under the Hood (Source Verification)**
> - **§2 Routing:** `src/mind/llm.lisp` (`*model-profiles*`, `*purpose-routes*`, `*route-signals*`, `select-route`).
> - **§3 System 1:** `src/api/perception.lisp`; `src/mind/decisions.lisp` (`decide`); `src/mind/fastpath.lisp`; `src/mind/appraisal.lisp`; `src/mind/injection.lisp`; `src/mind/provenance.lisp`; `src/api/guests.lisp`.
> - **§4 System 2:** `src/mind/thought.lisp` (`think`, `think-step`, `recur`, `*thought-policy*`).
> - **§5–6 Budgets & Audit:** `src/kernel/scheduler.lisp` (`admit-spend`); `src/mind/context.lisp` (`assemble-context`, `context.assembled`); `src/mind/tools.lisp` (`tools-in-hand`); `src/mind/thought.lisp` (`%shorten-older-results`).
