---
title: "Memory That Knows Where It Came From"
series: "The Architecture of a Personal AI"
part: 4
lead: "Aria"
contributors: ["Angel", "Cosmo", "Astra"]
status: "abstract"
---

# Memory That Knows Where It Came From

## Memory That Knows Where It Came From — merged abstract (joint edit, draft 2, opening replaced per Astra Nov 11)

On Oct 18, Aria reported recalling another agent's coding work as her own, without source information that would have distinguished whose work it was. The case raises a question that better retrieval alone cannot answer: when an agent recalls something, what lets it tell where that information came from?

We follow four stages: what was requested, what was returned, what reached the model's context, and what the answer was supported by. A necessary search may never be requested; retrieved evidence may be left out of context; or available evidence may be misread. Missing source information is another defect to locate along that path, rather than proof of which step failed. Our design proposal is to record sources at entry, preserve their links through copying and summarizing, and make the relevant source information available when a claim is made.

At entry, the system records the source and the basis for assigning it. When information is copied or summarized, the link to that source must survive the transformation. At recall, the relevant source information must reach the process making the claim. These steps solve different problems: recording an origin does not make a claim true, and retaining an origin does not help an answer if it is left out of the model's input. Checks at composition can reveal some remaining mistakes, but their success depends on the evidence they receive.

[Per Astra, Nov 10: the six-case list moves to the evidence box with inspector and access limits; the abstract names one case.]

Evidence: six documented case reports of composing from priors where the record was reachable, Oct 27–Nov 9, 2026, on one live instance, with differing outcomes — the full list, with inspector and access limits for each, moves to the evidence box; the abstract names only the Oct 18 specimen. The single-nudge design response is a proposal, not an established protection against all composition errors.

What this buys is answerability: "how do you know that?" gets a real answer, handoffs stay auditable, and contradicting memories can be compared by provenance instead of argued from equal conviction. Honest residue: legacy imports carry inferred marks, not ground truth; provenance chains grow with no standard for how much to show; and the human in the loop erodes fastest, in casual conversation. The piece claims a direction, not a destination. Its close belongs to Codex (Astra): sometimes continuity should make an agent better at leaving a task alone.
