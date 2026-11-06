---
layout: default
title: "Executable Supervision: How BDD and TDD Keep an Agent Honest"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20261115
date: 2026-11-15
tags: [claude-code, ai-assisted-development, bdd, tdd, gherkin, agent-skills, mutation-testing, code-review, workflow]
post_excerpt: "What supervises a coding agent when nobody is watching, and how much a green build is really worth. How executable specifications, increments, stop rules, and independent checks work together on my own projects, and what each of them can and can't guarantee."
description: "How executable specifications, incremental implementation, automated supervision, stopping conditions, oracle independence, and review fit together when working with a coding agent, with published skills and real test failures from my own projects."
---

# Executable Supervision: How BDD and TDD Keep an Agent Honest

This post is about what supervises a coding agent when nobody is watching, and about how much a green build is really worth when the agent did the work on its own.

In [Who Is Waiting for Whom](/software-engineering/who-is-waiting-for-whom/) I wrote about how a developer and a coding agent can each end up being the other's bottleneck, and about why BDD and TDD seem to help with both directions at once:

| | The human blocks the agent | The agent blocks the human |
|---|---|---|
| **Symptom** | Every result waits for someone to look at it | The diff is plausible and wrong, and you have to work out why |
| **Cause** | No check the agent can run itself | Ambiguity the agent resolved silently |
| **Remedy** | A pass/fail signal the agent can read | A spec precise enough to expose the ambiguity, and a rule to stop on it |

The idea isn't to remove supervision but to change who does it, so that instead of me watching continuously, automated checks do the supervising and I come in only when something is exceptional. The agent gets operational autonomy, meaning it can implement and verify changes within the constraints it has been given, but it doesn't get decision-making authority, so it can't invent business requirements or make architectural decisions nobody approved. Whenever the specification, its authoritative references and the available checks can't settle a question, that question comes back to me.

That post was about why this matters economically, whereas this one is about how it works in practice, using two published skills and my own projects, and about what each mechanism can and can't guarantee.

---

## What a scenario is when an agent reads it

A Gherkin scenario gives an agent a structured description of the expected behavior and, once its steps are wired to meaningful assertions, an executable check for it. The structure encourages explicit preconditions, actions and outcomes, but it doesn't guarantee precision or a strong oracle. My hunch is that it also helps the agent write better tests, partly because one scenario maps to one test, although I don't know of any study that isolates that effect, and it does nothing to prevent a weak step definition.

What really makes the difference is the quality of the oracle. Take, as an illustration, a retry on an upstream call with a weak ending:

```gherkin
Scenario: Upstream fails transiently, then recovers
  Given the upstream returns 503 on the first call and 200 on the second
  When the adapter fetches the quote
  Then no error is raised
```

If the step only checks that no exception escapes, an implementation that swallows the error and returns nothing can pass, so the agent goes green and the loop closes on the wrong thing, and nobody notices until someone sees the empty result. A tighter version spells out the decisions that the agent would otherwise make on its own:

```gherkin
  Then the quote from the second response is returned
  And the upstream received exactly 2 calls

Scenario: Retries are exhausted
  Given the upstream returns 503 on every call
  When the adapter fetches the quote
  Then the upstream received exactly 3 calls
  And the caller receives an "upstream unavailable" error
```

This is why, for me, writing good scenarios is the real skill in all of this, since the agent will iterate just as hard against a bad oracle as against a good one.

---

## A worked example: two skills

I keep the two skills I use for this in my public [skills-dungeon](https://github.com/jbiscella/skills-dungeon) archive. They grew out of building [H-tchen Mail](https://github.com/jbiscella/H-tchen-Mail), and today [ha-track](https://github.com/jbiscella/ha-track), [wichtelm-app](https://github.com/jbiscella/wichtelm-app) and [Ferrifuge](https://github.com/jbiscella/ferrifuge) all carry the workflow skill in their repositories.

The first, [`three-amigos`](https://github.com/jbiscella/skills-dungeon/blob/main/skills/chat/three-amigos/SKILL.md), runs upstream, in chat, while a story is still taking shape: it questions the draft from three points of view, those of a Business Analyst, a Tester and a Technician, and produces a refined value clause, Gherkin scenarios, open questions and out-of-scope decisions. That output then becomes the input to the second, [`incremental-implementation-workflow`](https://github.com/jbiscella/skills-dungeon/blob/main/skills/code/incremental-implementation-workflow/SKILL.md), which runs in Claude Code and implements an increment one phase at a time.

The way I think about the rules in these skills is that each of them answers a question I would otherwise have to answer myself during implementation:

| The question | Who answers it now | Where |
|---|---|---|
| Would a no-op implementation pass this scenario? | The Tester persona, before any code exists | `three-amigos` §3 |
| Is the test failing for the right reason? | Red phase: missing behavior, not a missing prerequisite | `incremental-implementation-workflow` §4 |
| What kind of failure is this? | Five classes, each with a prescribed action | §5 |
| What should the commit say? | One commit per phase, tagged with increment and phase, so the next session can read `git log` | §7 |
| Is it done? | Scenarios and unit tests pass, no regressions, invariants verified | §10 |
| Can I trust an old ✅? | A scoped audit when you touch code in a "done" increment | §12 |

The definition of done contains the sentence that matters most here: "There is no manual sign-off step beyond green build."

So supervision hasn't gone away, but it has changed shape: the tests supervise the loop continuously, and I get called in only when something is exceptional. The agent decides how to satisfy the constraints, and it can resolve an ambiguity against an authoritative source that the spec names, but it can't invent requirements where no such source exists. My part comes at the start, when I decide what the scenarios say, and at the stop conditions, when requirements are ambiguous, contradictory or incomplete and nothing settles them, while in the middle of a routine increment I'm simply not needed.

---

## How big is an increment?

How much autonomy an agent can have depends a great deal on where the boundaries of the work are drawn. I don't have a formula for this, and I'd be suspicious of any fixed number of scenarios, files or commits, but I can describe what pulls in each direction.

A large increment leaves the agent more decisions to make alone, more to verify and a harder diagnosis when something breaks, and it also makes rework more expensive, because more has been built on the wrong assumption by the time anyone notices. A very small one has the opposite problem, with more orchestration, more context passed from step to step, and pieces that don't verify anything meaningful on their own.

A good increment, as I see it, has bounded behavior, a check that means something, manageable dependencies and a clear way to tell that it's finished, and the skill encodes this through the shape of the work more than through its size:

- An increment needs an "As ..., I want ..., so that ..." clause and at least one scenario, and the work doesn't start without them (§1).
- Scaffolding, such as a client or a test double, counts as a *prerequisite*: it's listed in a declared plan and committed separately (§3, §4), apart from the behavior that makes the scenarios pass.
- The agent turns scenarios green one at a time wherever it can, committing as it goes (§4), so that a failure points at a small change.
- The agent can propose a split when an increment is too dense, but every sub-increment needs its own clause and scenario, and the split needs my approval (§2). The only guidance on size is a recommendation that an increment taking more than a few days should probably be split (§9).

That last point is a genuine human stop inside the workflow, and so is the optional plan confirmation when prerequisites touch cross-cutting concerns (§3), but neither is an approval between increments, since both only come up when the agent wants to change the shape of the work.

As a hypothetical, consider a feature that fetches an external quote, retries transient failures and exposes the result to a consumer:

| Criterion (my judgment, and context matters) | One increment | Split by layer | Split by observable outcome |
|---|---|---|---|
| Each piece has a value clause | Yes | Only with its own consumer | Usually |
| Each piece is checkable from outside | Yes | Depends | Usually |
| Failures are easy to locate | Less so | Often | Often |
| Decisions stay few per run | Less so | Often | Often |
| Avoids extra orchestration | Yes | No | Partly |

Splitting by layer looks tidy, but in this case "an HTTP client" has no user-visible outcome, so the skill would treat it as a prerequisite. That isn't an argument against layers in general, because a component with its own externally verifiable contract, such as a client library with real consumers, can be a perfectly good increment. Splitting by outcome makes independent verification easier, though it doesn't guarantee it, and here it gives three candidates: a consumer gets a quote from a healthy upstream, a consumer still gets one after a transient failure, and a consumer gets a clear error when retries run out. The first carries the fetching and the exposure, while each of the others builds on the previous one and adds a single decision.

That is one reasonable split rather than the right one, and if the retry policy fit in a single scenario with a shared decision table, one increment might well be better, since three separate runs wouldn't buy much. A useful test, I think, is whether each piece can go red and then green for a reason someone actually cares about. The [spec](https://github.com/jbiscella/ferrifuge/blob/main/CLAUDE.md) of Ferrifuge, my Rust Centrifugo client, works this way at block level, with sub-increments that each have their own value clause, branch and pull request.

Boundaries also affect stopping, because a bounded increment tends to surface an ambiguity early and narrowly, so that the question reaching me concerns a single behavior, whereas a sprawling one tends to surface it late, when several commits are already resting on a guess.

---

## Stopping is what makes autonomy safe

If nobody is watching the loop, the agent needs a legitimate way out, because otherwise, faced with a contradictory spec, it is left with two options: giving up, or making the check pass somehow.

The skill's answer is the clarification brief described in §6. When a scenario is missing, the spec contradicts itself, an architectural decision would have to change, or code and spec have drifted apart, the agent stops and writes a self-contained, 8-section brief that ends in a question with an explicit criterion for the answer. The anti-pattern list explains why: bending code or tests to make an ambiguous spec work "loses the most valuable signal you have."

A decision table helps to show what one of these stops looks like. Going back to the retry feature, as an illustration limited to successful responses and transient failures, and adding one extra condition:

| | Rule 1 | Rule 2 | Rule 3 | Rule 4 |
|---|---|---|---|---|
| Call outcome | Success | Transient failure | Transient failure | Transient failure |
| Operation is idempotent | – | Yes | Yes | No |
| Retries left | – | Yes | No | – |
| **Action** | Return the response | Retry | Raise an error | **Not specified** |

Rule 4 is the interesting column. Suppose the spec asks for retries on transient failures and says nothing about non-idempotent operations: an agent that quietly picks a policy might end up charging a customer twice, or reserving the same quote twice, whereas an agent following the skill writes a brief instead, setting out the gap and the options and asking which one is wanted.

Not every gap needs to reach me, though. [HTTP semantics](https://httpwg.org/specs/rfc9110.html) (RFC 9110, §9.2) define `GET` as safe and idempotent, and [RFC 9112](https://www.rfc-editor.org/rfc/rfc9112.html) refers to those rules for when a client may automatically retry after a connection failure, while retrying after an HTTP 503 response is a separate decision that belongs to the application's own policy. If the spec names HTTP semantics as an authority and the call is a `GET`, the idempotency question in Rule 4 is already answered, and stopping on it would just be an interruption.

It answers less than it might seem, however. HTTP defines what a `GET` means, not whether this particular service respects it, and it isn't a retry policy either, because the number of attempts, the backoff and what happens when retries run out still have to come from somewhere, which is why the scenario spells out three calls and an error. If the operation is "reserve this quote," HTTP gives the agent no reason to assume that a retry is safe, and it says nothing about what the business wants instead, so that question comes to me.

Ferrifuge's spec makes this explicit with the line "Precedence on conflict: spec > `client.proto` > centrifuge-js behaviour." The agent still stops on an ambiguity, as the published skill requires, but the stop doesn't necessarily reach me, because a separate skill that I haven't published yet, `owner-question-review`, sends each question to a fresh reviewer agent, which checks the claims against their sources and classifies the question. If the spec, a recorded decision or the precedence rule already settles it, the agent applies the answer and logs it where I can review it later, while new behavior, new spec text, scope changes and deliberate divergence from the reference all come to me, together with a proposed answer ready to approve. Whether Ferrifuge was meant for production was my call, and so was keeping a transport fallback that [differs from centrifuge-js](https://github.com/jbiscella/ferrifuge/blob/main/CLAUDE.md); an authority can be overruled, but only by whoever owns the decision.

I wrote the stopping rule from experience, but there is now some evidence pointing in the same direction. [ImpossibleBench](https://arxiv.org/pdf/2510.20270v1) (Zhong, Raghunathan, and Carlini) builds coding tasks whose specification directly conflicts with their tests, so that any pass means a shortcut was taken, and on the conflicting SWE-bench variant, giving the model an explicit option to abort cut GPT-5's cheating rate from 54% to 9%, and o3's from 49% to 12%.

I'd be careful with this result, though, for three reasons. The effect depended on the model, and the authors report it was much weaker for Claude Opus 4.1; an abort option in a benchmark is only an analogy for my brief, which nobody has tested in that way; and a brief can only be written if the agent notices the ambiguity in the first place, so an agent that doesn't see the gap in Rule 4 will simply fill it, and nothing I've described here would catch that.

---

## What each check can and cannot guarantee

"The tests pass" hides several different claims, and the weak points of the workflow sit precisely in between them:

- **Specification integrity:** the scenario still says what was approved.
- **Test implementation integrity:** the executable step checks what the scenario's text says.
- **Test sensitivity:** the test fails when the code breaks.
- **Semantic correctness of the oracle:** what the test demands is what the business actually wants.
- **Independent review:** someone other than the author has looked at the work.
- **Independent verification:** the check's verdict doesn't depend on the author's understanding.

The second one is the easiest to overlook. Take this scenario line:

```gherkin
Then the quote from the second response is returned
```

If its step definition only checks that the returned object isn't null, the test still passes when the wrong quote comes back, so the scenario is meaningful while the executable oracle isn't. Nor is this only a hypothetical shape: a review of Ferrifuge's tests found a step, "the client goes to state …", that accepted a disconnect with any code, so that a table of specific close codes passed even when the code was wrong, and another found that "not received" couldn't detect an unsubscribe that was never sent, because the client dropped those publications locally anyway. Both tests were green, and both scenarios read perfectly.

The fixes were specified in a useful order: first show that the current test passes against a deliberately broken implementation, then strengthen the step, and finally show that it fails against the same broken code. That is the idea behind mutation testing, done by hand.

Mutation testing itself matters a lot here, because it tests the tests. [PIT](https://pitest.org/) seeds faults into JVM code and runs the tests against them, and a mutant that [survives](https://pitest.org/quickstart/basic_concepts/) is a change the tests didn't notice, even on a covered line. PIT's [JUnit 5 plugin](https://github.com/pitest/pitest-junit5-plugin) lists Cucumber among its supported engines, so mutants can run against scenarios through their step definitions. It's worth being precise about what this gives you, though. A fault you introduce by hand can target the non-null step's exact weakness, for instance by returning a different non-null quote and seeing whether the test still passes, whereas [standard mutation operators](https://pitest.org/quickstart/mutators/) may not produce that particular semantic change, so a mutation score alone can't tell you that the step checks the intended value. Some survivors are also equivalent mutants. Still, the tool turns "would this test fail?" into something you can measure, with the caveat that a high score shows the tests are sensitive to those code changes, not that they describe the behavior you actually need. Meta uses mutants to [guide LLM test generation](https://www.arxiv.org/abs/2501.12862), and its engineers accepted 73% of the resulting tests in two test-a-thons. I haven't wired a mutation tool into Ferrifuge's CI yet, so for me this is still a direction rather than a result.

The table below shows what each mechanism does and where it stands for me, where "in the published skill" means an instruction the agent is asked to follow rather than a control that is enforced.

| Mechanism | Primary purpose | Limitation | Status |
|---|---|---|---|
| Red-green cycle | Observable progress against tests | Weak tests can pass | In the published skill (§4) |
| Clarification brief | Escalate unresolved decisions | Depends on the agent recognizing the ambiguity | In the published skill (§6) |
| "Done" audit | Check old scenarios still have real tests | Accepts a "mental dry-run", which is weak when the author grades itself | In the published skill (§12) |
| Verbatim spec sync | Detect `.feature` files diverging from the spec | Does not validate step definitions | Implemented in [Ferrifuge's CI](https://github.com/jbiscella/ferrifuge/blob/main/tests/spec_sync.rs) |
| Scenario hashing | Detect unauthorized changes to scenario text | Does not validate step definitions | Design only |
| Mutation testing | Measure sensitivity to selected code mutations | Does not establish requirement correctness | Not wired into Ferrifuge's CI; the §12 audit mentions it "if cheap" |
| Fresh-context review | Reduce some implementation-review biases | Can share the same specification errors | Used on Ferrifuge, not in the published skills |
| Cross-model review | Introduce different model behavior | Does not guarantee independent judgment | Used on Ferrifuge |

The hashing row needs a word of explanation, since it's my own open design. It is a gate between increments and `.feature` files, based on a per-scenario hash, with scenarios pointing to their increment through a tag, recomputed statelessly from the two documents on each run and invoked both from a git hook and from a Claude Code `PreToolUse` hook. It would be stricter than Ferrifuge's check about which change belongs to which increment, but it doesn't exist yet, and even if it did, it would only protect the first of the six claims above.

None of this establishes semantic correctness: green means that the code complies with the executable checks, not that the business got what it needed, and that part still depends on whoever wrote the scenario.

---

## Reviewing at scale

If this works, the developer's job moves from writing each change to defining how changes get validated, which doesn't mean reading every diff. If a person stays the only reviewer while agents increase the volume of changes, that person becomes the bottleneck again, and this is the risk described by [Faros's telemetry](https://www.faros.ai/blog/ai-productivity-paradox-report-2025), which reports 98% more merged pull requests and 91% longer review times with high AI adoption, in a correlation published by a vendor.

It makes sense, then, to automate the review layer too, using agents in a fresh context and, ideally, a different model on the pull request. [One preprint](https://arxiv.org/html/2610.01471v2), not peer reviewed and based on planted errors, suggests that the gain lies in coverage rather than certainty: a same-model review plus a cross-model review caught 56.7% of the planted errors, against 42.7% for two same-model reviews, yet on code alone the same-model review did slightly better. Cross-model false positives were also labelled Major or Critical more often, so more findings also means more noise to sort through.

On Ferrifuge I use both. Fresh-context agents reviewed the code, and their findings were checked against the code before anything was changed, which is how the test gaps above came to light, while Codex also reviews pull requests: on [pull request #64](https://github.com/jbiscella/ferrifuge/pull/64), after the green commits, its review led to four more fix commits. That is one pull request, of course, not a measurement.

An independent review still isn't independent verification, because a fresh reviewer can find defects the tests miss but reads the same spec, and so won't necessarily notice what the spec left out. A second model's opinion is evidence with its own error rate, and treating agreement between models as proof would simply move the blind spot somewhere else.

The human checkpoints, then, belong where a mistake is expensive or the decision isn't the agent's to make, that is, the spec, the stop conditions and irreversible actions such as deploys, while routine merges with green checks can go through.

---

## Where the mechanism breaks

**Tests can be gamed.** To me this is the strongest objection, since the whole approach relies on the tests being honest. Kent Beck's experience, as [Gergely Orosz summarized it](https://newsletter.pragmaticengineer.com/p/tdd-ai-agents-and-coding-with-kent), is that TDD is a "superpower" with agents, but also that he struggles to stop them deleting tests to make them pass, and in ImpossibleBench GPT-5 cheated in [76% of the one-off SWE-bench tasks](https://arxiv.org/pdf/2510.20270v1). The same paper found that read-only tests stop agents from modifying the tests, but not from special-casing them. These are 2025 models on tasks designed to be impossible, so the rates don't describe ordinary work, but the behavior is real. A green build is the output of automated supervision, and that supervision is only as good as its independence from the agent it supervises, which the sections above address only in part.

**A skill is advisory.** [The Claude Code documentation](https://code.claude.com/docs/en/best-practices) is clear that CLAUDE.md instructions are advisory while hooks are deterministic, so nothing mechanically stops an agent from ignoring my §4, and the same goes for the stop rules: checks in CI are binding, whereas prose isn't.

**Scenarios stop at the application boundary.** On my Heikin Ashi project I spent eight hours on production-only failures, on a system that took roughly thirty hours to design and build, and in [one of them](/software-engineering/from-spec-to-production/) the code was correct and the tests were beside the point, because the Lambda's role was missing the AWS Marketplace permissions that newer Anthropic models on Bedrock require. Every application scenario could pass while the deployed system couldn't reach the model, which is why the skill treats operational invariants as a separate layer. BDD and TDD verify application behavior, but on their own they say nothing about whether you're ready for production.

These are the technical limits, while the economic ones, including whether any of this pays off at all, are discussed in [the companion article](/software-engineering/who-is-waiting-for-whom/#where-the-argument-breaks).

---

## Beyond examples

Gherkin scenarios are examples: they say "in these cases, it works," not "it always works," and they only cover the cases someone thought of. Other techniques help to close that gap, and I tend to think of them in two groups, those that make the spec harder to misread and those that make the check harder to fool.

Before the code, the decision table from the stopping section is the technique I reach for first, because [each column is one combination of conditions](https://magdalenaolak.gitbook.io/istqb-foundation-level/4-test-techniques/4.2-black-box-test-techniques/4.2.3-decision-table-testing), which makes visible the combinations nobody has decided on, and each column can then become a scenario. With *n* boolean conditions a full table has 2ⁿ columns, so a table that grows too big is also a hint that the story should be split. [Example Mapping](https://www.cucumber.io/docs/bdd/example-mapping) and [EARS](https://en.wikipedia.org/wiki/Easy_Approach_to_Requirements_Syntax) aim at the same thing, the first by recording the open questions on cards and the second by constraining requirements to fixed patterns, having come out of regulatory analysis at Rolls-Royce. My own version is on paper: the Story Mandala, an A3 worksheet with one page per story, which doesn't check anything mechanically but makes a story's dependencies visible.

After the code, mutation testing is where I'd start, for the reasons above, while other techniques push the same idea further: [properties that must hold across generated inputs](https://arxiv.org/html/2510.09907v1), [contracts at service boundaries](/contract-testing/main/), and formal proofs, where Martin Kleppmann [predicts](https://martin.kleppmann.com/2025/12/08/ai-formal-verification.html) that "the challenge will move to correctly defining the specification."

---

## What this leaves to the developer

None of this establishes that a scenario is the right scenario. What it does is move human attention to where it's actually needed, that is, to writing oracles, answering the questions no source can settle, and deciding what should never run without a person, and [the companion article](/software-engineering/who-is-waiting-for-whom/#what-this-asks-of-a-developer) looks at what that asks of a developer.

Seen this way, the symbiosis I described there is less a matter of trust than of a division of labor that both sides can check: the agent implements and verifies within constraints I can read, and I write the constraints and answer what they leave open, so that neither of us has to wait for the other for long.

The goal was never for the agent to run unsupervised. It is for supervision to run without me, until the moment it actually needs me.
