---
layout: default
title: "Who Is Waiting for Whom: The Economics of Working with a Coding Agent"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20261108
date: 2026-11-08
tags: [claude-code, ai-assisted-development, bdd, tdd, qa, business-analysis, regtech, productivity]
post_excerpt: "An agent that never asks a question can still block you. Why the quantity to optimize with coding agents is accepted, correct increments rather than generated code, and what that means for developers, QA, and business analysis."
description: "Why BDD and TDD become valuable interfaces to coding agents: the costs of waiting and rework, generation throughput versus accepted increments, the case for preserving verification and requirements expertise, and a proposed way to test the claim."
---

# Who Is Waiting for Whom: The Economics of Working with a Coding Agent

An agent that never asks a question can still block you. It just does it later, often at a higher price.

My bet for the immediate future: the competitive developer is the one who works with a coding agent so that neither party spends its time waiting for the other, or cleaning up after the other. The practices that make this possible already exist. They are BDD and TDD, the same ones many teams wrote off as a waste of time.

The old objection was about people. To use these practices well, you first had to grow your developers into them, and that took time most projects would not grant. An agent changes the economics, because the workflow can be handed to it at the start of a session and its output can be checked by a machine. Behavior scenarios tell the agent what to build. A failing test shows what is missing, and its move from red to green signals success. In between, the tests supervise the agent, and no human has to watch.

That last sentence is the thesis, and it needs precision. Automated guardrails are supervision, not the absence of it. The goal is to replace continuous human intervention with automated supervision and exception-driven human involvement. The agent gets operational autonomy: it can implement and verify changes within defined constraints. It does not get decision-making authority: it cannot invent business requirements or make unauthorized architectural decisions. Human judgment comes in when the specification, its authoritative references, and the available checks cannot settle a question.

This is a prediction, and the evidence for it is mixed. It comes from using this approach on several of my own projects, including [H-tchen Mail](https://github.com/jbiscella/H-tchen-Mail), [ha-track](https://github.com/jbiscella/ha-track), [wichtelm-app](https://github.com/jbiscella/wichtelm-app), and [Ferrifuge](https://github.com/jbiscella/ferrifuge), not from a controlled comparison. This article covers the economics and what they mean for developers, QA, and business analysis. Its companion, [Executable Supervision](/software-engineering/executable-supervision/), covers how the mechanism works and where it fails, including agents that cheat on tests.

---

## Two bottlenecks, one in each direction

The usual framing asks how fast the agent is. The more useful question is who is waiting for whom.

| | The human blocks the agent | The agent blocks the human |
|---|---|---|
| **Symptom** | Every result waits for someone to look at it | The diff is plausible and wrong, and you have to work out why |
| **Cause** | No check the agent can run itself | Ambiguity the agent resolved silently |
| **Remedy** | A pass/fail signal the agent can read | A spec precise enough to expose the ambiguity, and a rule to stop on it |

The first direction is well described in the [Claude Code best practices](https://code.claude.com/docs/en/best-practices): "Without a check it can run, 'looks done' is the only signal available, and you become the verification loop: every mistake waits for you to notice it."

The second direction gets less attention, and it is harder to notice. An agent that guesses does not look blocked. It looks productive, and the cost arrives later, when you reconstruct the decision it made on your behalf from a long diff.

### Three costs, and why fewer interruptions is not the goal

Written as costs rather than symptoms, there are three:

1. **The human waits for the agent.** This is potentially the least disruptive of the three, if the human can do something else in the meantime and can trust the result when it arrives. It is expensive when that result blocks critical work. And without the trust, it is not really waiting: you are watching.
2. **The agent waits for the human.** Every review, clarification, and approval stalls the run. This is the cost most agent tooling tries to remove.
3. **Rework.** Someone discovers that a finished result is wrong and has to find out why, decide what it should have done, and redo it, along with whatever has since been built on top.

These are not interchangeable, and they are not the whole bill: total elapsed time and the cost of verification and compute are separate quantities again, and I know of no universal ranking between them. But attacking only the second cost is a trap. An agent that never stops never blocks anyone visibly, and it may produce code continuously while accumulating rework. Here is a hypothetical example. An agent is asked to add retries to an upstream call. With no precise acceptance criteria, it silently picks a retry count, decides which exceptions count as transient, and chooses a fallback. It never asks a question. The feature looks complete. Weeks later someone discovers that after a transient failure it returns an empty result instead of the quote. That agent never waited for anyone, and it still blocked its developer. The defect is only part of the bill. The investigation is the rest: why that retry count, which failures it treated as transient, why that fallback, and whether anyone authorized any of it. A scenario that states the returned value, the number of calls, and the behavior when retries run out leaves much less to reconstruct, because those decisions were written down before the code, and a check fails when one is broken.

So the quantity worth optimizing is not generation throughput. It is accepted, correct increments. That is a better proxy than generated code, though not a measure of business value by itself. The two can diverge a long way:

- METR had active maintainers [review 296 AI-generated pull requests](https://metr.org/notes/2026-03-10-many-swe-bench-passing-prs-would-not-be-merged-into-main/) that passed SWE-bench's automated tests, and found roughly half would not have been merged, even after adjusting for the human patches those maintainers also rejected. The agents had one attempt and no feedback from a reviewer, so this is not a limit on what they can do. It does show how far "passes the tests" can sit from "acceptable."
- [Faros's telemetry](https://www.faros.ai/blog/ai-productivity-paradox-report-2025), from over 10,000 developers, associates high AI adoption with 98% more merged pull requests and 91% longer review times, with no significant correlation with company-level throughput or quality. Faros sells engineering analytics and this is correlational, so I treat it as a description of the risk, not a measurement of a cause.
- [DORA's 2025 report](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report) finds AI adoption positively related to delivery throughput and negatively related to stability, and calls AI an amplifier. Its explanation is close to this article's argument: "Without robust control systems, like strong automated testing, mature version control practices, and fast feedback loops, an increase in change volume leads to instability."

My hypothesis is that a spec-first workflow shifts effort rather than eliminating it: more preparation up front, in exchange for less repeated review and correction later. I have no measurement of that trade, and I would not expect it to always pay. For a small or exploratory task, writing the scenarios can cost more than fixing whatever the agent gets wrong. Automated supervision has its own bill too. On [Ferrifuge](https://github.com/jbiscella/ferrifuge), my Rust Centrifugo client, a full cross-platform CI run reached about two hours, and that time is also waiting.

The same accounting applies to parallel work. A workflow can be worth it even if each increment takes the agent longer, provided it needs much less human attention and leaves the developer free to supervise other independent work. That is a hypothesis too, and it has limits: parallel work adds coordination, shared dependencies and long CI runs constrain it, more concurrent agents do not necessarily mean more throughput, and the developer's cognitive load may simply become the new bottleneck. Later in this article I propose how these trades could be measured.

---

## The objection was about people

The best-known industrial data on TDD is [a case study of four teams](https://doi.org/10.1007/s10664-008-9062-z), three at Microsoft and one at IBM. Pre-release defect density fell by 40% to 90% compared with similar projects. Subjectively, the teams experienced a 15% to 35% increase in initial development time.

Add to that the cost the study does not capture: a team has to learn to write a good test first, or a good scenario, and that learning happens slowly and unevenly. A manager looking at those two costs could reasonably say no.

With an agent in the loop, three things change:

1. **The workflow transfers in seconds.** The one I use is a [317-line file](https://github.com/jbiscella/skills-dungeon/blob/main/skills/code/incremental-implementation-workflow/SKILL.md). Every session that loads it starts with the same rules. No onboarding, no months of pairing. Loading the rules does not guarantee applying them well, because judgment does not transfer with the file, which is why the checks [have to be independent](/software-engineering/executable-supervision/#what-each-check-can-and-cannot-guarantee).
2. **The typing moved.** The agent writes the test and the implementation, so the initial-time penalty no longer falls on the developer in the same way. This is my inference. I know of no study that replicates those numbers with agents.
3. **The test gained a second reader.** A test used to be mostly a safety net for humans. For an agent, it is also the most precise definition of "finished" it can check on its own.

---

## The mechanism, briefly

A scenario states a requirement and, once its steps are wired to meaningful assertions, becomes the check for it. The agent writes a failing test, implements until it goes green, and stops with a written brief when requirements are ambiguous, contradictory, or insufficient and no authoritative source settles them.

A check like the retry scenario above is what I mean by an oracle: the thing that decides whether a result is right. A green result is evidence of compliance with those checks, not proof of business correctness. The agent will iterate as hard against a bad oracle as against a good one, so everything below depends on who can write a good one. The [companion article](/software-engineering/executable-supervision/) has the scenarios, the skills, and the failure modes.

---

## The people being treated as overhead

Some companies seem to be drawing the opposite conclusion from all this: agents write the code and the tests, so QA and business analysis are the obvious places to save. I have heard of a company that recently cut QA by half, going from two developers per QA engineer to four and, in practice, replacing them with Claude Code.

That is one secondhand example, and I can't put a reliable number on how widespread this is. [Coverage of an Indeed report](https://www.businesstoday.in/technology/news/story/ai-engineers-in-qa-testers-out-indeed-report-tracks-sharp-shift-in-tech-job-market-489695-2025-08-16) lists QA testers among the roles falling out of favour, without a figure. For business analysts the data I found points the other way: [one tracker](https://www.extern.com/post/will-ai-replace-business-analysts-guide) shows US postings rising 17% between July and October 2026. So I am describing a direction I see, not a measured trend. The argument holds however common the cut turns out to be, because the question is not who holds these titles, or at what ratio, but whether anyone still holds the expertise.

If the agent's loop is only as good as its oracles, then the scarce work is defining the oracles. That is the work these roles have practiced for years. A good business analyst asks who the actor really is, whether the value can be falsified, and what is out of scope. A good tester asks what the check actually observes, which edge cases are missing, and whether the test would pass regardless of the implementation. It is no accident that two of the three personas in [`three-amigos`](https://github.com/jbiscella/skills-dungeon/blob/main/skills/chat/three-amigos/SKILL.md), my refinement skill, are a Business Analyst and a Tester. Those questions are the skill's substance. As a group, these professionals are often the most disciplined and structured about exactly the preparation an agent needs, and that preparation is what these cuts risk removing.

I say this as a developer, not as someone defending another profession. I have spent years in regtech, where I often saw developers who were less technical and less naturally talented deliver more stable products, because the environment forced them to learn QA techniques, BDD, and TDD. That is an observation rather than a study, and regulated environments differ in other ways too, such as audits and the pace of change. But it shaped what I believe: discipline can compensate for talent more often than talent compensates for missing discipline.

It also cuts against any simple defense of job titles. Those developers show that the techniques can move to other people, and some of the work will legitimately move to agents, repetitive manual regression above all. What cannot be dropped is the verification and requirements expertise itself, wherever it sits. Remove the people who hold it without forcing anyone else to learn it, and you have a faster producer working against weaker oracles.

---

## Where the argument breaks

The technical objections, such as tests that can be gamed and scenarios that stop at the application boundary, are in the [companion article](/software-engineering/executable-supervision/#where-the-mechanism-breaks). These are the economic ones.

**The cost moved.** The learning curve did not vanish. Writing a scenario with a real oracle, or a value clause that can be falsified, is still the hard part, and now it sits with whoever writes the spec. Worse, the persona pass can make a weak spec feel reviewed. The [`three-amigos` README](https://github.com/jbiscella/skills-dungeon/blob/main/skills/chat/three-amigos/README.md) says so directly: it does not replace stakeholder validation. Three personas played by one model do not know what your users need. BDD's original value was the conversation between real people, and a simulation does not reproduce it.

**The productivity evidence is unsettled.** METR's [early-2025 trial](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) found experienced open-source developers took 19% longer with AI. Its [February 2026 update](https://metr.org/blog/2026-02-24-uplift-update/) has point estimates that lean toward a speedup, with confidence intervals that include zero, and METR itself calls the new data "an unreliable signal," partly because developers increasingly declined to work without AI. Neither result tested a spec-first workflow. "Competitive" remains my reading of the direction, not a measured outcome.

**It does not fit everything equally.** Exploration does not rule it out, because formalization comes in small increments: a thin scenario for what you know now, changed as you learn. On H-tchen Mail I retrofitted Gherkin as the way to define the rules. What I warned about in the [Two Claudes pattern](/software-engineering/the-two-claudes-pattern/) still holds for formalizing a large spec too early, when it hardens a shape you haven't earned yet. And the overhead is still wrong for anything judged by perception rather than propositions, and for scripts you will delete on Friday. [The workflow skill's README](https://github.com/jbiscella/skills-dungeon/blob/main/skills/code/incremental-implementation-workflow/README.md) excludes pure exploratory spikes with no spec to enforce, and codebases with no test infrastructure.

---

## How this could be tested

Everything cited here is adjacent. The TDD study predates agents, and the METR work did not test a spec-first workflow. Nothing I have found establishes that this workflow beats the alternative. My experience comes from my own projects, not from a controlled comparison. I have not run the following; it is a proposed experiment.

Take comparable but different development tasks in an existing codebase, and two workflows, with the same model:

- **Conventional:** prompts, then human review of the output.
- **Spec-first:** scenarios first, TDD, automated guardrails, and exception-driven intervention.

Each developer would use both workflows but never see the same task twice, with tasks and order randomized or counterbalanced so that neither experience nor task difficulty lines up with one workflow. Acceptance criteria would be fixed before any results were seen. Someone outside the experiment would then judge the work in two separate ways: hidden acceptance tests check the specified behavior, and a maintainer's merge decision, as in METR's note, checks whether the work is acceptable. "Passes its own tests" would not count, since that is the claim under test.

| Dimension | What to measure |
|---|---|
| Human effort | Preparation time; number and duration of interventions; review and rework time |
| Elapsed time | Time until acceptance, including agent execution and time spent waiting for a person or CI |
| Compute cost | Model usage and CI time, which is where the cost of verification appears |
| Outcome | Rework required; defects found by independent evaluation |

These need to stay separate because they can disagree. As a hypothetical, the spec-first workflow might take less human effort and yet finish later, because stops and long CI runs queue up while the developer is busy elsewhere. Or it might finish sooner while demanding more of the developer's concentrated attention up front. Which of those counts as better depends on what is scarce.

The limitations are serious. Tasks are hard to make comparable, models change between runs, and familiarity with a workflow takes time to build and would favor whichever one a developer already knows. A developer's preparation may also carry over: having written scenarios once, you may review better even without them. And the spec-first workflow may simply lose on some tasks. I would expect that for small ones, and a useful experiment should be able to show it.

---

## What this asks of a developer

If the argument holds, the skills that gain value are the ones at the two ends of the loop:

| Moves to the agent | Stays with the developer |
|---|---|
| Writing the implementation and the test wiring | Deciding what is worth building and what is out of scope |
| Iterating from red to green | Writing oracles a no-op cannot satisfy |
| Classifying failures against written rules | Making the check independent of the agent it checks |
| Recording decisions in commits | Answering the brief when the spec runs out |

Concretely, I think that means getting better at a few things, even as agents write more of the code and the tests:

- **Defining verifiable behavior.** Recognizing a weak oracle, by reading a step definition and asking what would still pass, and writing falsifiable requirements. "So that the system is more robust" cannot fail, and so it cannot guide anything.
- **Managing decision boundaries.** Finding implicit assumptions: whether a retry is safe for a non-idempotent operation stays invisible until somebody asks. Deciding what can be delegated, where the [classification I describe in the companion article](/software-engineering/executable-supervision/#stopping-is-what-makes-autonomy-safe) is my current attempt.
- **Evaluating correctness.** Separating implementation correctness from business correctness, since the code can do exactly what the scenario says while the scenario is wrong. Understanding the architectural invariants that must hold across every change, because no single scenario will show them breaking.
- **Maintaining technical judgment.** Keeping enough implementation depth to judge the result. Nobody can evaluate a review finding, or an agent's claim about a stalled websocket, without understanding the machinery.

The last item has a real counterargument. Excessive delegation may gradually weaken the judgment this whole model depends on. In an [Anthropic randomized trial](https://www.anthropic.com/research/AI-assistance-coding-skills) with 52 mostly junior engineers learning a new library, the AI group scored 50% on a follow-up quiz against 67% for those coding by hand, with the largest gap on debugging. In the authors' qualitative analysis, delegation patterns scored lowest and patterns aimed at understanding scored higher. Anthropic sells these tools, the sample was small, it measured immediate understanding rather than long-term skill, and it did not study an agent workflow. Still, it points at the danger: this article asks developers to become better judges while removing much of the practice that used to build judgment. I don't have a clean answer. The trial's better-scoring patterns suggest a partial one: ask the agent to explain, not only to fix.

None of that right-hand column is new. It is what a good Three Amigos session and a disciplined TDD practice always demanded. The practices were rarely wrong. They were expensive to teach to people, and the workflow is cheap to hand to an agent. What remains expensive is the judgment about what the scenarios should say. That is where I would now spend my own learning time, and it is a strange moment to let that expertise walk out of the door before anyone else has learned it.

How that supervision works, and how much a green build is really worth, is the subject of [Executable Supervision](/software-engineering/executable-supervision/).
