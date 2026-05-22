---
layout: default
title: "Heikin Ashi – Part 6: When AI Confidence Lies"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20260522
date: 2026-05-22
tags: [claude, anthropic, ai-assisted-development, epistemology, debugging, prompt-engineering]
post_excerpt: AI confidence fails in at least five distinct ways, each with its own recognition signal, its own typical cost, and its own counter-move. A field guide to premise drift, false architectural confidence, speculation as diagnosis, over-generalization from precedent, and confident invention of identifiers.
---

# When AI Confidence Lies: A Field Guide

*Part 6 of 6 in the [Heikin Ashi series](#series-navigation).*

The second post in this series introduced *false architectural confidence* — the failure mode where an AI tool communicates directional certainty about a choice without having sufficient grounding to justify it. The concrete example was a market-data provider recommendation that turned out to lack the regional coverage the project needed. The post named the pattern and moved on.

The deployment session that followed produced enough additional material to turn the single failure mode into a taxonomy. What I observed across roughly forty hours of design-then-deploy work is that AI confidence fails in at least five distinct ways, each with its own recognition signal, its own typical cost, and its own appropriate intervention. They are easy to conflate when you encounter them one at a time. Naming them separately makes them easier to catch.

This post is a field guide. Each pattern gets a name, a definition, an example from the session, the signal that lets you recognize it in the moment, and the counter-move that defuses it.

---

## The common root

Before the catalog, the structural observation that ties the patterns together.

From the user's side, an AI tool's output does not expose a reliable confidence-weighted distribution over possible answers. It exposes one fluent answer. The generation process does not reliably distinguish "I know this" from "I can produce a plausible-sounding sentence about this." To the reader, both can look equally fluent. The reader (the human) is left to infer confidence from context, hedging language, and consistency with other claims — none of which the model is reliably calibrated to produce honestly.

The patterns below are different ways the gap between actual grounding and apparent confidence manifests. They are not, strictly, "errors" in the sense of the model getting a fact wrong. They are mismatches between what the model knows and what its output implies it knows. The danger is that the implication is the part the human acts on.

A single distinction does most of the work in defending against all five patterns: **distinguish what the model has observed from what the model has produced.** The model has observed content you have shown it in this conversation, plus tool results from searches, file reads, or executions in the current turn. The model has produced everything else — recommendations, diagnoses, explanations, identifiers, generalizations that did not originate in observed content. The first category is more reliable, though not infallible: observed content can still be misread or lost in a long context. The second category is the source of every pattern below.

---

### Pattern 1: Premise drift

**Definition.** The model commits to an implicit premise early in a conversation, then maintains perfect local consistency with that premise across many turns. Every individual claim is correct given the premise. The premise itself was never justified and may be wrong.

**Example from the session.** The design conversation in post 2 spent twelve turns elaborating a REST API design — endpoints, JSON envelopes, error mappings, OpenAPI-style schemas — for a system that has no HTTP surface. Each individual specification was internally coherent. The premise that "this system has a REST API" was never stated explicitly; it was assumed silently in the second turn and reinforced for ten more before the contradiction surfaced.

**Recognition signal.** Increasing specificity over many turns without a corresponding return to first principles. If the conversation is producing more and more detailed output and you can't easily articulate what the top-level assumption is, the premise has probably drifted.

**Counter-move.** Periodically, every five to ten turns in a long design conversation, ask: *"What's the implicit frame we're operating in right now, and is it still right?"* The question feels redundant but it's the only mechanism that catches premise drift before it sinks twenty turns of work.

---

### Pattern 2: False architectural confidence

**Definition.** The model recommends a specific tool, library, service, or pattern with directional certainty, without having grounded the recommendation in the project's actual constraints. From the outside it looks like expertise; from the inside it's the most popular default the model knows for that category.

**Example from the session.** The original recommendation of Twelve Data as the market-data provider, with explicit pricing tier and rate limits, where the free tier turned out to exclude European exchanges — the constraint that mattered most for the project.

**Recognition signal.** A complete, specific recommendation arrives in a single response, with named versions and tiers, and the model neither asks about your constraints nor restates which constraints the choice satisfies. Genuinely grounded recommendations do one or the other. If neither, the recommendation is probably going by default.

**Counter-move.** *"Under what constraints is this the right choice? What would have to be true for this to be wrong?"* Vague or absent answers mean the recommendation is decoration, not analysis.

---

### Pattern 3: Speculation as diagnosis

**Definition.** When asked to explain why something is failing, the model produces a plausible-sounding causal explanation without verification. The explanation may be partially correct, fully correct, or wholly invented; the response surface gives no indication of which.

**Example from the session.** During the Yahoo Finance integration debugging, the model proposed that Yahoo's edge layer blocked AWS IP ranges as a class. The claim was confident and structurally complete. The actual cause was that Yahoo's edge layer classified the library's default User-Agent string as a scraper, independent of origin IP. The wrong hypothesis cost roughly twenty minutes before a controlled experiment with `curl` falsified it.

**Recognition signal.** Causal language used without evidence. Phrases like "this is because," "the cause is," or "what's happening is" applied to a failure the model could not have directly observed. Genuine diagnosis arrives with the test that confirmed it.

**Counter-move.** *"What one observation would confirm or falsify this hypothesis?"* If the model cannot name one, the explanation is speculation. Treat it as a candidate to test, not a fact to act on. The previous post in this series elaborates the broader discipline.

---

### Pattern 4: Over-generalization from precedent

**Definition.** The model generalizes a true claim about one specific case into a broader claim about a category of cases. The original claim was correct; the generalization is not, but the surface coherence makes the broader claim feel like the natural inference.

**Example from the session.** Earlier in the deployment, Bedrock's Opus model required an EU-regional inference profile (`eu.anthropic.claude-opus-4-7`) rather than a raw model ID, because the foundation model wasn't directly invokable in eu-central-1. From this true and observed fact, the model generalized that all Anthropic models in eu-central-1 require an inference profile. When the project later switched to a different Anthropic model, the generalization shaped the diagnosis of a separate failure (an IAM Marketplace permission gap), wasting attention on inference profile issues that weren't the cause. The danger is structural: a verified local fact gets promoted into an unverified global rule.

**Recognition signal.** A confident statement about a category of cases that derives from one prior observation. The model uses language like "for this class of models" or "with this kind of API," where the qualifying clause is doing the over-generalization work.

**Counter-move.** *"What's the smallest example that would contradict this generalization?"* The exercise either confirms the pattern by failing to find a counter-example, or surfaces the over-generalization immediately.

---

### Pattern 5: Confident invention of identifiers

**Definition.** The model fills in a file path, ARN, configuration key, or other identifier with a plausible value rather than verifying or asking. The invented identifier is structurally well-formed and may even be partially correct.

**Example from the session.** Multiple instances of the model proposing `grep` or `cat` commands against file paths that did not exist in the project — paths that were plausible given Maven conventions and the project's naming, but that did not match the actual filesystem. Each instance produced thirty to ninety seconds of confused output before the correct path was located.

**Recognition signal.** A command that operates on a specific path, ARN, key, or table name that hasn't been established in the conversation. The identifier sounds right; it has no provenance. The risk is highest when the invented identifier is *almost* right — close enough to a real value that the command runs partially before failing, leaving the system in an unclear state.

**Counter-move.** Before running a command against any identifier the model produced from inference rather than observation, verify the identifier with a `find`, `ls`, `aws ... describe`, or equivalent. The cost of the verification is one command. The cost of acting on an invented identifier is the time to discover the error plus the time to recover from any partial state. The cost asymmetry is large enough that the verification should be default behavior, not exception.

---

## How the patterns relate

The five patterns are distinct in mechanism but they share an underlying property: in each case, the model's output appears more grounded than it is. The fluency of the response surface implies a level of verification that didn't happen.

The patterns vary in how easy they are to detect:

| Pattern | Detection difficulty | Typical cost | Best countermeasure |
|---|---|---|---|
| Premise drift | Hard — coherent for many turns | Hours of work in wrong frame | Restate the frame periodically |
| False architectural confidence | Medium — recommendation lacks asked-for constraints | Days if not caught before commit | Ask for the constraints the recommendation is satisfying |
| Speculation as diagnosis | Medium — causal language without test | Minutes to hours per instance | Ask for the observation that would falsify |
| Over-generalization from precedent | Hard — sounds like reasonable inference | Compounds across the session | Ask for a smallest counter-example |
| Confident invention of identifiers | Easy — fails on first execution | Seconds to minutes per instance | Verify the identifier before running |

The hardest to catch are the ones where the model is reasoning coherently over a longer arc — premise drift and over-generalization — because the surface output gives the same signal across many turns and the human reader stops looking for the original assumption.

The easiest to catch — confident invention of identifiers — is also the cheapest per instance, but it recurs so often that the cumulative friction is significant over a long session.

---

## What this changes about how I use AI tools

Two updates.

The first: I no longer treat the AI's response as the answer to my question. I treat it as the first draft of the answer, where the model's job is to produce the candidate and my job is to verify it. The shift is small in the moment but compounds significantly across a project. The model is usually stronger at producing candidates than at verifying that those candidates are grounded in the current context; expecting it to do both is the source of most of the failures above.

The second: I now budget time, in any non-trivial session, for confidence-failure recovery. For my own planning, I budget roughly fifteen to thirty minutes of correction time for every hour of non-trivial AI-assisted design or operational work. That ratio is my own and reflects my own experience and discipline; it isn't a law. It does improve with practice. It does not go to zero. A project plan that assumes AI-assisted work is faster than unassisted work without budgeting recovery time will systematically overrun. Mine did.

---

## What's next

The first three posts in this series described how to design with AI tools. The fourth and fifth described what design discipline cannot reach. This post described how to read the AI's confidence honestly across all of the above.

A theme runs through all six: the productive disciplines are the ones that distinguish, in the moment, between things the model knows and things the model is producing on the fly. Design discipline keeps the spec close to verified intent. Boundary discipline keeps the production envelope close to verified invariants. Operational discipline keeps debugging close to verified observations. The confidence-reading discipline above is the one that operates on the AI's output itself — the last layer, the one that decides what gets propagated as a fact and what gets held as a hypothesis.

The remaining gap, if there's another post in the series, is the one about extracting reusable discipline from individual project experience — turning a session's failures into the next session's checklists. The patterns above are at least named. The next step is making them harder to fall into.

---

## Series navigation {#series-navigation}

| | |
|---|---|
| ← Part 5 | [Heikin Ashi – Part 5: The Operator's Manual](/software-engineering/the-operators-manual/) |
