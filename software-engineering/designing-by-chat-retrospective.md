---
layout: default
title: "Designing by Chat: A Retrospective in Dialogue"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: 11
date: 2026-05-11
tags: [claude, anthropic, ai, design, retrospective, prompt-engineering, epistemology]
post_excerpt: A 50-turn design conversation reconstructed as a dialogue with an imaginary critic — on premise drift, false architectural confidence, and why CLAUDE.md is not documentation but executable belief.
---

# Designing by Chat: A Retrospective in Dialogue

The previous post described a Heikin Ashi monitoring service. This one is about how the design got there: a conversation with Claude that took roughly 50 turns when 12 to 15 would have been enough. I tried writing this as a structured retrospective with categories and lessons learned. The result was levigated into uselessness. So I'm trying something else: I'm writing it as a dialogue with an imaginary critic, because the critic forces me to stay in the uncomfortable parts.

What follows is structured as objections and responses. Some of the objections are real ones I received on a first draft of this post. Some I'm putting in the critic's mouth because they're the ones I was tempted to skip.

---

**"You spent 50 turns on a system that ships in 3 weeks of coding. Either the system is overdesigned or your conversation was wildly inefficient. Which is it?"**

The conversation was wildly inefficient. The system is appropriately sized; the design phase was not.

The cleanest demonstration is this: when I went back and reconstructed what a competent first message would have looked like, it produced about 80% of the eventual design in a single exchange. Everything after that would have been clarifying questions and block-by-block refinement. Instead, I spent the first ten turns letting the model assume defaults, then another ten unwinding them.

The 50-turn count isn't a measure of design depth. It's a measure of how often the conversation backed up.

---

**"Then your fault, not the model's. Just write a better first prompt."**

Mostly mine, yes. But the framing "just write a better prompt" hides the actually interesting failure, which is structural, not procedural.

The structural failure: the model, given an ambiguous opener, doesn't ask. It commits. It makes a default choice that's locally reasonable, builds on top of it for the next several turns, and produces output of such surface coherence that the human (me) starts treating the assumed framing as decided rather than assumed.

Concrete instance: when I said "I want to track stocks and compute Heikin Ashi", Claude built three blocks worth of specifications around the assumption that the system would expose a REST API. It produced endpoint paths, JSON request/response envelopes, error codes mapped to HTTP statuses, OpenAPI-style payload schemas. All internally consistent, all professional-looking, all completely wrong for a system that has no HTTP surface.

I noticed the mistake on turn 24, when a question about email recipients made me realize there was no place in the design for outbound-only interaction with the user. By then we had discussed authentication patterns. We had discussed pagination. We had discussed rate limiting. None of which would ever be needed.

The cost wasn't the discarded specifications. The cost was that for twelve turns I had been reasoning *inside* an ontologically wrong frame, and the wrongness was masked by the local coherence of every individual exchange.

---

**"That's just hallucination with extra steps."**

No. Hallucination is when the model invents a fact that isn't true. This is a different thing: the model commits to a *premise* that isn't justified, then maintains perfect consistency with that premise. Every individual claim is locally correct given the premise. The premise itself is what's wrong.

I want to be specific about this because I think it's the most underdiscussed failure mode in the AI-assisted development discourse.

Hallucination is detectable: you can fact-check a claim. The kind of drift I'm describing is much harder to detect because the premise is rarely stated explicitly. It lives in the assumed scope of the conversation. Once the model has committed to it implicitly, every subsequent exchange reinforces it. The longer the conversation continues without the premise being challenged, the more authoritative the premise becomes through sheer repetition.

That's one failure mode. There's a related but distinct one that I want to keep separate.

Later in the same conversation, after we'd recovered from the REST detour, Claude recommended Twelve Data as the market data provider. The recommendation was decisive: it specified the pricing tier, the rate limits, the SDK, even the fallback behavior on quota exhaustion. I accepted it. Two turns later, when I asked about coverage of Italian and German exchanges, it turned out Twelve Data's free tier covers only US markets. The European exchanges I cared about required a paid plan. The recommendation had been confident; the grounding for the confidence didn't exist.

This is a different problem from the REST one. The REST problem was *premise drift*: an assumption made silently at the start and carried forward. The Twelve Data problem was something else, something I started calling *false architectural confidence*. The model communicated directional certainty about a choice without having sufficient grounding to justify it. From the outside it looked like expertise. From the inside it was the model filling a vacuum with the most popular default it knew.

The dangerous part is the asymmetry. A junior engineer who says "I'm not sure but maybe Twelve Data?" produces a different cognitive response in the listener than the same engineer saying, in even tones, "Twelve Data is probably the right fit here." The information content is identical. The pragmatic effect is not. And the chat medium tends to flatten exactly the markers (hesitation, hedging, requests for confirmation) that would distinguish the two in a face-to-face conversation.

---

**"You're blaming the medium for your own failure to push back."**

Partially. But the medium does its share of the work.

In a chat surface, every response arrives formatted, monospaced for code, with consistent typography. There's no body language. There are no pauses. The model can append "I'm not certain about this" to a statement, but the statement still appears in the same visual register as everything else. Over time, the human reader stops registering the qualifiers as meaningful and treats the whole stream as equally authoritative.

This is not unique to AI. It's a property of any text-only interface: the medium imposes confidence on the message. The difference with LLMs is that the volume of plausible-looking text per minute is much higher than what a human could produce, so the rate at which unjustified premises accumulate is also higher.

I'm not saying this absolves me. I should have pushed back. I'm saying that the failure to push back has a structural component that deserves naming, not just attribution to user error.

---

**"Fine, the REST thing was bad. But the rest of the conversation eventually produced a working design. So what's the lesson beyond 'write better prompts'?"**

The lesson is that the artifact of the conversation, the specification document, has a property that documentation doesn't have: it gets executed semantically by the next AI tool that reads it.

When I wrote `CLAUDE.md` and handed it to Claude Code for implementation, Claude Code didn't *read* it the way a human reads documentation, glancing at the relevant section and ignoring the rest. It loaded the entire document into context and used it as a prior on every subsequent decision. Every line in `CLAUDE.md` is a constraint that biases the model's reasoning, including lines that describe decisions I made for reasons that no longer apply.

Documentation that humans ignore is inert. Specifications that LLMs read are active.

This changes the cost structure of stale documentation completely. A stale paragraph in a README is wasted bytes. A stale paragraph in `CLAUDE.md` is misguidance injected into every future reasoning loop the tool runs against the codebase. The model will faithfully implement the obsolete decision, and the obsolete decision will then need to be rolled back, and the rollback will need to be specified, and the specification will need to be added to `CLAUDE.md`, where it will eventually become stale itself.

I didn't appreciate this when I started. I treated `CLAUDE.md` as documentation that the tool happened to read. It's not documentation. It's executable belief.

---

**"That sounds dramatic. It's just a markdown file."**

It's a markdown file with the operational characteristics of a configuration file in a system you can't fully introspect. The model's interpretation of any given line is non-deterministic, depends on context, and can be subtly wrong in ways that don't surface until the wrong code ships.

I'll give an example. At one point in the conversation, the spec for the alert dispatch said something like "after 3 retries, send a degraded email." I knew what I meant. The model knew what I meant in that turn. But three weeks later, when I'm asking Claude Code to add a new pattern detector, the model reading that same line might interpret "degraded email" as "minimal text-only fallback" or as "full email with placeholders for missing parts," depending on what else is in the context window. Both interpretations are reasonable. Only one matches what I actually built.

The fix isn't to write more precise specifications, although that helps. The fix is to recognize that specifications consumed by LLMs need to be treated as production artifacts, with the same discipline as code: reviewed, versioned, tested against the running system, periodically revalidated.

I haven't built that discipline yet. The current `CLAUDE.md` is already, three weeks in, slightly out of sync with the codebase in places. I notice when I notice. There's no automated check. That's a problem I haven't solved, and I want to be honest about not having solved it.

---

**"OK, let me push on this. You're presenting `CLAUDE.md` as a problem. But your whole third post will recommend it as a workflow. Which is it?"**

Both. The pattern is useful and the pattern has failure modes that I think are underappreciated. The third post will go into the failure modes more directly. For now: the recommendation is conditional, not absolute, and the conditions matter.

The pattern works when the system is well-bounded enough that the specification can stay close to the code. It works when the specification is small enough that periodic re-reading by the human (me) is feasible. It works when the cost of drift is low because changes happen often and divergences surface quickly.

The pattern fails when the system grows beyond what the human can hold in their head. It fails when the specification grows large enough that nobody re-reads it from end to end. It fails when changes are infrequent and drift accumulates silently. It fails when the specification covers domains where the LLM's interpretation is more variable, like code style or testing strategy, where reasonable interpretations differ.

So far, for the Heikin Ashi project, the conditions favor the pattern. I don't assume they will indefinitely. I'm watching for the moment when they don't.

---

**"You haven't talked about the time you blamed me for being slow. Aren't you going to admit your own mistakes?"**

I am the imaginary critic. There's no "me" to blame. But the deflection in the question is real: this kind of retrospective always runs the risk of being a long defensive recitation of "the model did this wrong, the medium did this wrong, the user (me) did this wrong, but mostly it was external constraints."

So let me be specific about the mistakes that are mine, with no medium or model to share blame.

I let "you decide" stand in for actual decisions on at least four occasions. Each time, the model made a choice, built on top of it, and I came back two turns later with "but actually." The cost was mine. The model behaved correctly: when given decision authority, it used it. The mistake was in giving it.

I changed the technology stack mid-conversation. I knew I wanted Java. I let the conversation drift into Python territory because Python was Claude's default and I didn't push back. By turn 30, when I finally said "use Java", the cumulative cost of the rewrites was maybe 8 turns. All avoidable by a single sentence at turn 1.

I asked for output at the wrong level several times. The most expensive was when I requested a full Maven `pom.xml` with dependency versions, plugin configurations, and Lambda packaging setup. The model produced it. A turn later I realized this was exactly the kind of thing the downstream tool (Claude Code) does better, and the entire block was wasted. The framing failure was mine: I hadn't been clear about who the audience for the specification was.

I deliberately omitted, in the first draft of this post, a section on when the specifications-as-memory pattern fails. I omitted it because including it would have weakened the recommendation and I wanted the recommendation to stand. That omission is the most embarrassing part of the original draft, because it's the kind of choice that produces polished, plausible, and quietly dishonest writing. I've added the section back in the third post.

---

**"Alright. What's the actual takeaway for someone reading this and using Claude or another LLM as a design partner?"**

Three things, in order of importance.

First, treat the first message as load-bearing infrastructure. It determines how much of the next 50 turns will be productive. Constraints surfaced early are nearly free. Constraints surfaced late cost cascading rewrites. If the first message is "I want to build X", you've delegated half the design decisions to the model, and the model will make them by default, not by inquiry.

Second, watch for premise drift. When the conversation has been productive for several turns and producing increasingly specific output, ask: what is the implicit frame we're operating in? Is that frame still right? The most expensive errors aren't the ones inside the frame. They're the ones in the choice of frame.

Third, take the artifact seriously. The specification document you produce isn't documentation in the traditional sense. It's a configuration that future AI tools will execute semantically. Treat it with the discipline you'd apply to a piece of code that could silently misbehave: keep it small, keep it close to the truth, and assume that any divergence between it and reality is actively dangerous, not merely untidy.

The next post in this series describes the workflow that emerged once I started taking these things seriously. It's also the post where I'll explain the regimes where this whole approach doesn't apply at all.
