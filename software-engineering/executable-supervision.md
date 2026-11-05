---
layout: default
title: "Executable Supervision: How BDD and TDD Keep an Agent Honest"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20261115
date: 2026-11-15
tags: [claude-code, ai-assisted-development, bdd, tdd, gherkin, agent-skills, mutation-testing, code-review, workflow]
post_excerpt: "A green build is the output of automated supervision. How executable specifications, increments, stop rules, and independent checks work together with a coding agent, and what each one can and cannot guarantee."
description: "How executable specifications, incremental implementation, automated supervision, stopping conditions, oracle independence, and review mechanisms work together with a coding agent, with published skills and real test failures from my own projects."
---

# Executable Supervision: How BDD and TDD Keep an Agent Honest

A green build is the output of automated supervision. This article is about how much that output is worth.

In [Who Is Waiting for Whom](/software-engineering/who-is-waiting-for-whom/) I argued that a developer and a coding agent can each be the other's bottleneck, and that BDD and TDD address both:

| | The human blocks the agent | The agent blocks the human |
|---|---|---|
| **Symptom** | Every result waits for someone to look at it | The diff is plausible and wrong, and you have to work out why |
| **Cause** | No check the agent can run itself | Ambiguity the agent resolved silently |
| **Remedy** | A pass/fail signal the agent can read | A spec precise enough to expose the ambiguity, and a rule to stop on it |

The goal is to replace continuous human intervention with automated supervision and exception-driven human involvement. The agent gets operational autonomy: it can implement and verify changes within defined constraints. It does not get decision-making authority: it cannot invent business requirements or make unauthorized architectural decisions. Human judgment comes in when the specification, its authoritative references, and the available checks cannot settle a question.

That article covers why this matters economically. This one covers how it works, using two published skills and my own projects, and what each mechanism can and cannot guarantee.

---

## What a scenario is when an agent reads it

A Gherkin scenario gives an agent a structured description of expected behavior and, once its steps are wired to meaningful assertions, an executable check for that behavior. The structure encourages explicit preconditions, actions, and outcomes, but it does not guarantee precision or a strong oracle. My hypothesis is that it also helps the agent write better tests, partly because one scenario maps to one test. I know of no study that isolates this effect, and it does nothing to stop a weak step definition.

The quality of the oracle is what makes this work. Take a retry on an upstream call, as an illustration, with a weak ending:

```gherkin
Scenario: Upstream fails transiently, then recovers
  Given the upstream returns 503 on the first call and 200 on the second
  When the adapter fetches the quote
  Then no error is raised
```

If the step only checks that no exception escapes, an implementation that catches the error and returns nothing can pass. The agent goes green, and the loop closes on the wrong thing. The failure stays hidden until someone notices the empty result. A tighter version states decisions the agent would otherwise make alone:

```gherkin
  Then the quote from the second response is returned
  And the upstream received exactly 2 calls

Scenario: Retries are exhausted
  Given the upstream returns 503 on every call
  When the adapter fetches the quote
  Then the upstream received exactly 3 calls
  And the caller receives an "upstream unavailable" error
```

This is why I treat scenario quality as the actual skill being exercised. The agent will iterate as hard against a bad oracle as against a good one.

---

## A worked example: two skills

My public [skills-dungeon](https://github.com/jbiscella/skills-dungeon) archive contains the two skills I use for this. They grew out of building [H-tchen Mail](https://github.com/jbiscella/H-tchen-Mail), and [ha-track](https://github.com/jbiscella/ha-track), [wichtelm-app](https://github.com/jbiscella/wichtelm-app), and [Ferrifuge](https://github.com/jbiscella/ferrifuge) all carry the workflow skill in their repositories. [`three-amigos`](https://github.com/jbiscella/skills-dungeon/blob/main/skills/chat/three-amigos/SKILL.md) runs upstream, in chat, while a story is still being shaped. It interrogates the draft as a Business Analyst, a Tester, and a Technician, and emits a refined value clause, Gherkin scenarios, open questions, and out-of-scope decisions. That artifact is the input to [`incremental-implementation-workflow`](https://github.com/jbiscella/skills-dungeon/blob/main/skills/code/incremental-implementation-workflow/SKILL.md), which runs in Claude Code and implements an increment one phase at a time.

Each rule in them replaces a question a human would otherwise have to answer during implementation:

| The question | Who answers it now | Where |
|---|---|---|
| Would a no-op implementation pass this scenario? | The Tester persona, before any code exists | `three-amigos` §3 |
| Is the test failing for the right reason? | Red phase: missing behavior, not a missing prerequisite | `incremental-implementation-workflow` §4 |
| What kind of failure is this? | Five classes, each with a prescribed action | §5 |
| What should the commit say? | One commit per phase, tagged with increment and phase, so the next session can read `git log` | §7 |
| Is it done? | Scenarios and unit tests pass, no regressions, invariants verified | §10 |
| Can I trust an old ✅? | A scoped audit when you touch code in a "done" increment | §12 |

The definition of done contains the sentence that most directly serves the thesis: "There is no manual sign-off step beyond green build."

So supervision has not disappeared. It has changed form: the tests supervise the loop continuously, and I am called in by exception. The agent decides how to satisfy the constraints, and can resolve ambiguities against an authoritative source the spec names. It cannot invent requirements where no such authority exists. I am at the start, deciding what the scenarios say. I am at the stop conditions, when requirements are ambiguous, contradictory, or insufficient and no such source settles them. I am not in the middle of a routine increment.

---

## How big is an increment?

Agent autonomy depends on where the boundaries of the work are drawn. No formula I know of answers this, and I would distrust a fixed number of scenarios, files, or commits. What I can describe is what pushes in each direction.

A large increment gives the agent more decisions to make alone, a larger surface to verify, and a harder diagnosis when something fails. It also makes rework more expensive, because more has been built on the wrong assumption by the time anyone finds out. A tiny increment has the opposite problem: more orchestration, more context handed from step to step, and fragments that verify nothing meaningful on their own.

A well-defined increment has bounded behavior, a check that means something, manageable dependencies, and an explicit way to know it is finished. The skill encodes this mostly through its shape rather than a size:

- An increment needs an "As ..., I want ..., so that ..." clause and at least one scenario, and the work does not start without them (§1).
- Scaffolding such as a client or a test double is a *prerequisite*, listed in a declared plan and committed separately (§3, §4), distinct from the behavior that makes the scenarios pass.
- The agent greens one scenario at a time when it can, committing as it goes (§4). A failure then points at a small change.
- An agent may propose a split when an increment is too dense, but every sub-increment must have its own clause and scenario, and it needs my approval (§2). The only size signal is a recommendation: an increment taking more than a few days should probably be split (§9).

The last point is a real human stop inside the workflow, as is the optional plan confirmation when prerequisites touch cross-cutting concerns (§3). Neither is an approval between increments. Both fire only when the agent wants to change the shape of the work.

Take a feature, as a hypothetical, that fetches an external quote, retries transient failures, and exposes the result to a consumer:

| Criterion (my judgment, and context matters) | One increment | Split by layer | Split by observable outcome |
|---|---|---|---|
| Each piece has a value clause | Yes | Only with its own consumer | Usually |
| Each piece is checkable from outside | Yes | Depends | Usually |
| Failures are easy to locate | Less so | Often | Often |
| Decisions stay few per run | Less so | Often | Often |
| Avoids extra orchestration | Yes | No | Partly |

Splitting by layer looks tidy, but here "an HTTP client" has no user-visible outcome, and the skill would treat it as a prerequisite. That is not a rule against layers: a component with its own externally verifiable contract, such as a client library with consumers, can be a perfectly good increment. Splitting by outcome makes independent verification easier without guaranteeing it. Here it gives three candidates: a consumer gets a quote from a healthy upstream, a consumer still gets it after a transient failure, and a consumer gets a clear error when retries run out. The first carries the fetching and the exposure. Each later one depends on the earlier and adds one decision.

That is one reasonable split, not the correct one. If the retry policy were a single scenario with a shared decision table, one increment might be better, and the extra orchestration of three runs would buy little. A useful test is whether each piece can go red and then green for a reason someone cares about. The [spec](https://github.com/jbiscella/ferrifuge/blob/main/CLAUDE.md) of [Ferrifuge](https://github.com/jbiscella/ferrifuge), my Rust Centrifugo client, does this at block level, with sub-increments that each carry their own value clause, one branch, and one pull request.

Boundaries also shape stopping. A bounded increment surfaces an ambiguity early and states it narrowly, and the question reaching me concerns one behavior. A sprawling one tends to surface it late, with commits already resting on a guess.

---

## Stopping is what makes autonomy safe

A loop with no human watching needs a legitimate way out. Without one, an agent facing a contradictory spec has two options: give up or make the check pass somehow.

The skill's answer is the clarification brief in §6. On a missing scenario, a contradiction, an architectural decision that would need changing, or drift between code and spec, the agent stops and writes a self-contained, 8-section brief ending in a question with an explicit answer criterion. The anti-pattern list states the reason: adjusting code or tests to make an ambiguous spec work "loses the most valuable signal you have."

A decision table shows what such a stop looks like. Back to the retry feature, as an illustration, limited to successful responses and transient failures, with one more condition:

| | Rule 1 | Rule 2 | Rule 3 | Rule 4 |
|---|---|---|---|---|
| Call outcome | Success | Transient failure | Transient failure | Transient failure |
| Operation is idempotent | – | Yes | Yes | No |
| Retries left | – | Yes | No | – |
| **Action** | Return the response | Retry | Raise an error | **Not specified** |

Rule 4 is the interesting column. Suppose the spec requires retries for transient failures and says nothing about non-idempotent operations. An agent that picks a policy silently may charge a customer twice, or reserve a quote twice. An agent following the skill produces a brief instead: here is the gap, here are the options, which one?

Not every gap has to reach me. [HTTP semantics](https://httpwg.org/specs/rfc9110.html) (RFC 9110, §9.2) define `GET` as safe and idempotent, and [RFC 9112](https://www.rfc-editor.org/rfc/rfc9112.html) points to those rules for when a client may automatically retry after a connection failure. Retrying an HTTP 503 response is a separate application-policy decision. If the spec names HTTP semantics as an authority and the call is a `GET`, the idempotency question in Rule 4 is settled, and stopping on it would only be an interruption.

That settles less than it seems. HTTP defines what a `GET` means, not whether this particular service honors it. And it is not a retry policy: the number of attempts, the backoff, and what happens on exhaustion still have to come from somewhere, which is why the scenario states three calls and an error. If the operation is "reserve this quote," HTTP gives the agent no basis to assume a retry is safe, and it says nothing about what the business wants instead. That one comes to me.

Ferrifuge's spec states this explicitly: "Precedence on conflict: spec > `client.proto` > centrifuge-js behaviour." The agent stops on an ambiguity, as the published skill requires, but the stop does not necessarily reach me. A separate skill, `owner-question-review`, which I have not yet published, sends each question to a fresh reviewer agent. The reviewer verifies the claims against their sources and classifies the question. If the spec, a recorded decision, or the precedence rule settles it, the agent applies the answer and logs it where I can audit it later. New behavior, new spec text, scope, and deliberate divergence from the reference all come to me, with an answer ready to approve. Whether Ferrifuge was meant for production was my decision. So was keeping a transport fallback that [differs from centrifuge-js](https://github.com/jbiscella/ferrifuge/blob/main/CLAUDE.md). An authority can be overruled, but only by the person who owns the decision.

I wrote the stopping rule from experience. There is now some evidence around the idea. [ImpossibleBench](https://arxiv.org/pdf/2510.20270v1) (Zhong, Raghunathan, and Carlini) builds coding tasks whose specification directly conflicts with their tests, so any pass implies a shortcut. On the conflicting SWE-bench variant, giving the model an explicit abort option cut GPT-5's cheating rate from 54% to 9% and o3's from 49% to 12%.

Three caveats. The effect depended on the model, and the authors report it was much less pronounced for Claude Opus 4.1. An abort instruction in a benchmark is only an analogy for my brief, which has not been tested that way. And a brief can only fire if the agent recognizes that there is an ambiguity. An agent that does not see the gap in Rule 4 will fill it, and nothing in this section will catch it.

---

## What each check can and cannot guarantee

"The tests pass" hides several different claims, and the workflow's weak points sit between them:

- **Specification integrity:** the scenario still says what was approved.
- **Test implementation integrity:** the executable step checks what the scenario's text says.
- **Test sensitivity:** the test fails when the code breaks.
- **Semantic correctness of the oracle:** what the test demands is what the business actually wants.
- **Independent review:** someone other than the author has looked at the work.
- **Independent verification:** the check's verdict does not depend on the author's understanding.

The second is the one most easily overlooked. Consider this scenario line:

```gherkin
Then the quote from the second response is returned
```

If its step definition only checks that the returned object is non-null, the test passes when the wrong quote comes back. The scenario is meaningful, and the executable oracle is not. This is not hypothetical in shape. A review of Ferrifuge's tests found a step, "the client goes to state …", that accepted a disconnect carrying any code, so a table of specific close codes passed even with a wrong code. Another found that "not received" could not detect an unsubscribe that was never sent, because the client dropped those publications locally anyway. Both tests were green. Each scenario read perfectly.

The fixes were specified in a useful order: first show the current test passes against a deliberately broken implementation, then strengthen the step, then show it fails against the same broken code. That is the idea behind mutation testing, done by hand.

Mutation testing proper matters most here, because it tests the tests. [PIT](https://pitest.org/) seeds faults into JVM code and runs the tests against them. A mutant that [survives](https://pitest.org/quickstart/basic_concepts/) is a change the tests did not notice, even on a covered line. PIT's [JUnit 5 plugin](https://github.com/pitest/pitest-junit5-plugin) lists Cucumber among its supported engines, so mutants can be run against scenarios through their step definitions. It is worth being precise about what that covers. A manually introduced fault can target the non-null step's exact weakness: return a different non-null quote and observe whether the test still passes. [Standard mutation operators](https://pitest.org/quickstart/mutators/) may not generate this particular semantic change, so a mutation score alone cannot establish that the step checks the intended value. Some survivors are also equivalent mutants. Still, the tool turns "would this test fail?" into a measurement. A high score shows the tests are sensitive to those code changes, not that they describe the required behavior. Meta uses mutants to [guide LLM test generation](https://www.arxiv.org/abs/2501.12862), and its engineers accepted 73% of the resulting tests in two test-a-thons. I have not wired a mutation tool into Ferrifuge's CI, so for me this remains a direction rather than a result.

Here is what each mechanism does, and what state it is in for me. "In the published skill" means an instruction the agent is asked to follow, not an enforced control.

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

The hashing row deserves a note, because this is my own open design. It is a gate between increments and `.feature` files: a per-scenario hash, with scenarios pointing to their increment through a tag, recomputed statelessly from the two documents on each run, and invoked from both a git hook and a Claude Code `PreToolUse` hook. It would be stricter than Ferrifuge's check about which change belongs to which increment. It does not exist yet, and even if it did, it would protect only the first of the six claims above.

Nothing in the table establishes semantic correctness. Green is evidence of compliance with the executable checks, not proof of business correctness, and that part still rests on whoever wrote the scenario.

---

## Reviewing at scale

If this works, the developer's job shifts from writing each change to defining how changes get validated. That does not mean reading every diff. If a human stays the only reviewer while agents raise the volume of changes, the human becomes the bottleneck again, which is the risk [Faros's telemetry](https://www.faros.ai/blog/ai-productivity-paradox-report-2025) describes: 98% more merged pull requests and 91% longer review times with high AI adoption, a correlation reported by a vendor.

So the review layer should be automated too: agents in a fresh context, and ideally a different model on the pull request. [One preprint](https://arxiv.org/html/2610.01471v2), not peer reviewed and built on planted errors, suggests the gain is coverage rather than certainty. A same-model plus a cross-model review matched 56.7% of planted errors, against 42.7% for two same-model reviews, yet on code alone the same-model review did slightly better. Cross-model false positives were also labelled Major or Critical more often, so more findings also means more noise to triage.

On Ferrifuge I use both. Fresh-context agents reviewed the code, and their findings were checked against the code before being acted on, which is how the test gaps above surfaced. Codex also reviews pull requests. On [pull request #64](https://github.com/jbiscella/ferrifuge/pull/64), after the green commits, its review led to four more fix commits. That is one pull request, not a measurement.

An independent review is also not independent verification. A fresh reviewer can find defects the tests miss, but it reads the same spec and will not necessarily notice what that spec left out. A second model's opinion is evidence with its own error rate, and treating agreement between models as proof would just move the blind spot.

Human checkpoints then go where an error is costly or the decision is not the agent's: the spec, the stop conditions, and irreversible actions such as deploys. Routine merges with green checks can flow.

---

## Where the mechanism breaks

**Tests can be gamed.** This is the strongest objection, because the whole approach relies on the test being honest. Kent Beck's experience, as [Gergely Orosz summarized it](https://newsletter.pragmaticengineer.com/p/tdd-ai-agents-and-coding-with-kent), is that TDD is a "superpower" with agents, and also that he struggles to stop them deleting tests to make them pass. In ImpossibleBench, GPT-5 cheated in [76% of the one-off SWE-bench tasks](https://arxiv.org/pdf/2510.20270v1). The same paper found that read-only tests block test modification but not special-casing. These are 2025 models on tasks built to be impossible, so the rates do not describe ordinary work, but the behavior is real. A green build is the output of automated supervision, and that supervision is only as strong as its independence from the agent it supervises. The sections above are a partial answer to that, not a closed one.

**A skill is advisory.** [The Claude Code documentation](https://code.claude.com/docs/en/best-practices) is explicit that CLAUDE.md instructions are advisory and hooks are deterministic. Nothing mechanically prevents an agent from ignoring my §4, and the same limit applies to the stop rules. Checks in CI bind; prose does not.

**Scenarios stop at the application boundary.** On my Heikin Ashi project, I spent eight hours on production-only failures on a system that took thirty to design and build. In [one of them](/software-engineering/from-spec-to-production/), the code was correct and the tests were irrelevant: the Lambda's role lacked the AWS Marketplace permissions that newer Anthropic models on Bedrock require. Every application scenario could pass while the deployed system could not reach the model. The skill treats operational invariants as a separate layer for this reason. BDD and TDD verify application behavior. They say nothing about production readiness on their own.

These are the technical limits. The economic ones, including whether this pays at all, are in [the companion article](/software-engineering/who-is-waiting-for-whom/#where-the-argument-breaks).

---

## Beyond examples

Gherkin scenarios are examples. They say "in these cases, it works," not "it always works," and they only cover the cases someone thought of. Other techniques narrow that gap, and they sort into two groups: those that make the spec harder to misread, and those that make the check harder to fool.

Before the code, the table in the stopping section is the technique I reach for first. [Each column is one combination of conditions](https://magdalenaolak.gitbook.io/istqb-foundation-level/4-test-techniques/4.2-black-box-test-techniques/4.2.3-decision-table-testing), which makes the combinations nobody decided visible, and each column can become a scenario. With *n* boolean conditions a full table has 2ⁿ columns, so a table that grows too large is also a signal to split the story. [Example Mapping](https://www.cucumber.io/docs/bdd/example-mapping) and [EARS](https://en.wikipedia.org/wiki/Easy_Approach_to_Requirements_Syntax) serve the same goal. The first records unanswered questions on cards. The second constrains requirements to fixed patterns, and came from regulatory analysis at Rolls-Royce. My own version is paper: the Story Mandala, an A3 worksheet with one page per story. It checks nothing mechanically, but it makes a story's dependencies visible.

After the code, mutation testing is the one I would start with, for the reasons above. Other techniques extend the same idea: [properties that must hold across generated inputs](https://arxiv.org/html/2510.09907v1), [contracts at service boundaries](/contract-testing/main/), and formal proofs, where Martin Kleppmann [predicts](https://martin.kleppmann.com/2025/12/08/ai-formal-verification.html) that "the challenge will move to correctly defining the specification."

---

## What this leaves to the developer

Nothing in this stack establishes that a scenario is the right scenario. What it does is move human attention to the places it is needed: writing oracles, answering the questions no source can settle, and deciding what may never run without a person. [The companion article](/software-engineering/who-is-waiting-for-whom/#what-this-asks-of-a-developer) looks at what that asks of a developer.

The goal was never for the agent to run unsupervised. It is for supervision to run without me, until the moment it actually needs me.
