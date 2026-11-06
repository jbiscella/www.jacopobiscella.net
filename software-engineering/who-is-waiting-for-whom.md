---
layout: default
title: "Who Is Waiting for Whom: The Economics of Working with a Coding Agent"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20261108
date: 2026-11-08
tags: [claude-code, ai-assisted-development, bdd, tdd, qa, business-analysis, regtech, productivity]
post_excerpt: "An agent that never asks a question can still block you, just later. Notes from my own projects on waiting, rework, and why accepted increments seem a better thing to optimize than generated code, and what that might mean for developers, QA, and business analysts."
description: "Why BDD and TDD may be worth more with coding agents than they were with people: the costs of waiting and rework, generated code versus accepted increments, why verification and requirements expertise still matters, and a proposed way to test the idea."
---

# Who Is Waiting for Whom: The Economics of Working with a Coding Agent

This year I've built several projects with a coding agent doing most of the implementation: [H-tchen Mail](https://github.com/jbiscella/H-tchen-Mail), [ha-track](https://github.com/jbiscella/ha-track), [wichtelm-app](https://github.com/jbiscella/wichtelm-app), and more recently [Ferrifuge](https://github.com/jbiscella/ferrifuge), a Rust client for Centrifugo. Somewhere along the way I noticed that the question I kept asking myself wasn't how fast the agent was. It was who was waiting for whom.

Sometimes I was the bottleneck: the agent had finished, and the work sat there until I found time to look at it. Sometimes it was the other way around, and that was harder to spot, because it didn't look like waiting at all. The agent had made a decision I never asked for, quietly, and I only found out later, when I had to work out from a long diff why it had done what it did.

What helped most were two practices I'd seen many teams dismiss as a waste of time: BDD and TDD. My suspicion is that the reason they were dismissed has changed. They were expensive because people had to learn them, slowly, and most projects didn't have the time. An agent can be handed the workflow at the start of every session. Scenarios tell it what to build, a failing test tells it what's missing, and the move from red to green tells it when it's done. In between, the tests do the supervising, and I don't need to watch.

I want to be careful about what that means, because it's easy to overstate. The tests don't replace supervision; they are the supervision. The agent decides how to satisfy the constraints, not what they should be, and when the specification and its references can't answer a question, the question comes back to me.

This is a reading of my own experience rather than the result of a controlled comparison, and the wider evidence is mixed. This post looks at the economics, and at what they mean for developers, QA, and business analysts. Its companion, [Executable Supervision](/software-engineering/executable-supervision/), looks at how the mechanism works and where it breaks.

---

## Two bottlenecks, one in each direction

Once I started looking at it this way, the two directions turned out to be quite different:

| | The human blocks the agent | The agent blocks the human |
|---|---|---|
| **Symptom** | Every result waits for someone to look at it | The diff is plausible and wrong, and you have to work out why |
| **Cause** | No check the agent can run itself | Ambiguity the agent resolved silently |
| **Remedy** | A pass/fail signal the agent can read | A spec precise enough to expose the ambiguity, and a rule to stop on it |

The first one is easy to recognize, and the [Claude Code best practices](https://code.claude.com/docs/en/best-practices) describe it well: "Without a check it can run, 'looks done' is the only signal available, and you become the verification loop: every mistake waits for you to notice it."

The second took me longer to see. An agent that guesses doesn't look blocked. It looks productive. The cost shows up later, when you have to reconstruct, from a long diff, a decision it made on your behalf.

### Three costs, and why fewer interruptions is not the goal

If I try to write these down as costs rather than symptoms, I end up with three:

1. **The human waits for the agent.** This can be the least disruptive of the three, as long as there's something else to do in the meantime and the result can be trusted when it arrives. It gets expensive when that result is blocking something important. And without the trust, it isn't really waiting: it's watching.
2. **The agent waits for the human.** Every review, clarification, or approval stalls the run. Most agent tooling, understandably, focuses on removing this one.
3. **Rework.** Someone finds out that a finished result is wrong, has to work out why, decide what it should have done, and redo it, together with whatever has been built on top in the meantime.

They aren't interchangeable, and they aren't the whole bill either: elapsed time and the cost of verification and compute are separate again, and I don't know of any general ranking between them. What I've come to distrust is attacking only the second one. An agent that never stops never visibly blocks anyone, and it can keep producing code while rework quietly piles up.

A hypothetical example makes this concrete. An agent is asked to add retries to an upstream call. There are no precise acceptance criteria, so it picks a retry count, decides which exceptions count as transient, and chooses a fallback, all without asking. The feature looks complete. Weeks later someone notices that after a transient failure it returns an empty result instead of the quote. That agent never waited for anyone, and it still blocked its developer.

The defect is only part of the cost. The rest is the investigation: why that retry count, which failures it treated as transient, why that fallback, and whether anyone actually agreed to any of it. If a scenario had stated the returned value, the number of calls, and what happens when retries run out, there would be much less to reconstruct. Those decisions would have been written down before the code, and a check would fail as soon as one was broken.

This is why I've stopped thinking of generation throughput as the thing to optimize. What I actually want more of is accepted, correct increments. That's still only a proxy, not a measure of business value, but it's a much better one than generated code. And the two can be surprisingly far apart:

- METR asked active maintainers to [review 296 AI-generated pull requests](https://metr.org/notes/2026-03-10-many-swe-bench-passing-prs-would-not-be-merged-into-main/) that had passed SWE-bench's automated tests. Roughly half wouldn't have been merged, even after adjusting for the human patches those same maintainers also rejected. The agents had a single attempt and no reviewer feedback, so this isn't a ceiling on what they can do, but it shows how much distance there can be between "passes the tests" and "acceptable."
- [Faros's telemetry](https://www.faros.ai/blog/ai-productivity-paradox-report-2025), from over 10,000 developers, associates high AI adoption with 98% more merged pull requests and 91% longer review times, without a significant correlation with company-level throughput or quality. Faros sells engineering analytics, and the data is correlational, so I read it as a description of the risk rather than as proof of a cause.
- [DORA's 2025 report](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report) finds AI adoption positively related to delivery throughput and negatively related to stability, and describes AI as an amplifier. Its explanation is close to what I'm suggesting here: "Without robust control systems, like strong automated testing, mature version control practices, and fast feedback loops, an increase in change volume leads to instability."

My working hypothesis is that a spec-first workflow moves effort around rather than removing it: a bit more preparation before each increment, in exchange for less review and correction afterwards. I haven't measured that trade, and I wouldn't expect it to pay off every time. On a small task, writing the scenarios can cost more than fixing whatever the agent gets wrong. Exploration works too, but only if the scenarios stay thin enough to change. And automated supervision has its own bill: on Ferrifuge, a full cross-platform CI run reached about two hours, and that's waiting as well.

Parallel work fits the same accounting. A workflow could be worth it even if each increment takes the agent longer, as long as it needs much less of the developer's attention and leaves them free to look after other independent work. That's a hypothesis too, and it has obvious limits. Parallel work adds coordination, shared dependencies and long CI runs get in the way, more agents running at once don't automatically mean more throughput, and the developer's own cognitive load may simply become the new bottleneck. Further down I suggest a way these trades could be measured.

---

## The objection was about people

The best-known industrial data on TDD comes from [a case study of four teams](https://doi.org/10.1007/s10664-008-9062-z), three at Microsoft and one at IBM. Compared with similar projects, pre-release defect density fell by 40% to 90%. The teams also reported, subjectively, a 15% to 35% increase in initial development time.

There's a cost the study doesn't capture, too: a team has to learn to write a good test first, or a good scenario, and that learning is slow and uneven. Looking at both costs, I can understand a manager who said no.

With an agent in the loop, three things look different to me:

1. **The workflow transfers in seconds.** The one I use is a [317-line file](https://github.com/jbiscella/skills-dungeon/blob/main/skills/code/incremental-implementation-workflow/SKILL.md). Every session that loads it starts from the same rules, with no onboarding and no months of pairing. That doesn't mean the agent applies them well. Judgment doesn't come with the file, which is why the checks [need to be independent](/software-engineering/executable-supervision/#what-each-check-can-and-cannot-guarantee).
2. **The typing has moved.** The agent writes both the test and the implementation, so the initial-time penalty doesn't fall on the developer in the same way. That's my inference; I don't know of a study that has repeated those measurements with agents.
3. **The test has a second reader.** A test used to be mostly a safety net for people. For an agent, it's also the most precise definition of "finished" it can check by itself.

---

## The mechanism, briefly

Here's the short version. A scenario states a requirement and, once its steps are wired to meaningful assertions, becomes the check for it. The agent writes a failing test, implements until it goes green, and stops with a written brief when the requirements are ambiguous, contradictory, or incomplete and nothing it's allowed to rely on settles the question.

A check like the retry scenario above is what I mean by an oracle: the thing that decides whether a result is right. A green result tells you the code complies with those checks; it doesn't tell you the business got what it needed. And the agent will work just as hard against a weak oracle as against a strong one, so a lot depends on who writes them. The [companion article](/software-engineering/executable-supervision/) goes through the scenarios, the skills, and the ways it fails.

---

## The people being treated as overhead

Some companies seem to be drawing the opposite conclusion: if agents write the code and the tests, QA and business analysis look like the obvious place to save. I've heard of a company that recently cut its QA team in half, going from one QA engineer for every two developers to one for every four, and in practice replacing them with Claude Code.

That's one secondhand example, and I can't say how common this is. [Coverage of an Indeed report](https://www.businesstoday.in/technology/news/story/ai-engineers-in-qa-testers-out-indeed-report-tracks-sharp-shift-in-tech-job-market-489695-2025-08-16) lists QA testers among the roles falling out of favor, without giving a figure. For business analysts, the data I found points the other way: [one tracker](https://www.extern.com/post/will-ai-replace-business-analysts-guide) shows US postings rising 17% between July and October 2026. So this is a direction I think I'm seeing, not a measured trend. For me, though, the question isn't really who holds these titles, or at what ratio. It's whether anyone still holds the expertise.

If an agent's loop is only as good as its oracles, then defining the oracles becomes the scarce work. And that's exactly what these roles have been practicing for years. A good business analyst asks who the actor really is, whether the value can be falsified, and what's out of scope. A good tester asks what a check actually observes, which edge cases are missing, and whether the test would pass no matter what the implementation did. Two of the three personas in [`three-amigos`](https://github.com/jbiscella/skills-dungeon/blob/main/skills/chat/three-amigos/SKILL.md), my refinement skill, are a Business Analyst and a Tester, and that's deliberate: their questions are what the skill is made of. These are often the people most disciplined about exactly the preparation an agent needs, and that preparation is what these cuts risk taking away.

I'm saying this as a developer, not as someone defending another profession. I spent years in regtech, and I often saw developers who were less technical, and less naturally talented, deliver more stable products, because the environment forced them to learn QA techniques, BDD, and TDD. That's an observation, not a study, and regulated environments differ in other ways too, like audits and the pace of change. But it shaped what I believe: discipline makes up for talent more often than talent makes up for missing discipline.

It also cuts against a simple defense of job titles. Those developers show that the techniques can move to other people, and some of the work will reasonably move to agents, repetitive manual regression above all. What I don't think can be dropped is the verification and requirements expertise itself, wherever it ends up sitting. Remove the people who hold it without asking anyone else to learn it, and you get a faster producer working against weaker oracles.

---

## Where the argument breaks

The technical objections, like tests that can be gamed and scenarios that stop at the application boundary, are in the [companion article](/software-engineering/executable-supervision/#where-the-mechanism-breaks). These are the economic ones.

**The cost moved.** The learning curve didn't disappear. Writing a scenario with a real oracle, or a value clause that could actually turn out to be false, is still the hard part, and now it sits with whoever writes the spec. There's a subtler risk as well: the persona pass can make a weak spec feel reviewed. The [`three-amigos` README](https://github.com/jbiscella/skills-dungeon/blob/main/skills/chat/three-amigos/README.md) says it plainly: it doesn't replace stakeholder validation. Three personas played by one model don't know what your users need. Much of BDD's original value came from conversations between real people, and a simulation doesn't reproduce that.

**The productivity evidence is unsettled.** METR's [early-2025 trial](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) found that experienced open-source developers took 19% longer with AI. Its [February 2026 update](https://metr.org/blog/2026-02-24-uplift-update/) has point estimates leaning toward a speedup, but with confidence intervals that include zero, and METR itself calls the new data "an unreliable signal," partly because developers increasingly declined to work without AI. Neither study tested a spec-first workflow. So "competitive" is my reading of where things are going, not something anyone has measured.

**It doesn't fit everything equally.** Exploration doesn't rule it out, because the formalization happens in small increments: a thin scenario for what you know now, changed as you learn more. On H-tchen Mail I retrofitted Gherkin as the way to define the rules. What I warned about in the [Two Claudes pattern](/software-engineering/the-two-claudes-pattern/) still applies to formalizing a large spec too early, when it hardens a shape you haven't earned yet. And the overhead is still the wrong tool for anything judged by perception rather than propositions, or for scripts you'll delete on Friday. [The workflow skill's README](https://github.com/jbiscella/skills-dungeon/blob/main/skills/code/incremental-implementation-workflow/README.md) excludes pure exploratory spikes with no spec to enforce, and codebases with no test infrastructure.

---

## How this could be tested

Everything I've cited sits next to the question rather than answering it. The TDD study predates agents, and METR didn't test a spec-first workflow. I haven't found anything showing that this way of working beats the alternative, and my own experience comes from my projects, not from a controlled comparison. What follows is an experiment I'd like to see run; I haven't run it.

Take comparable but different development tasks in an existing codebase, and two workflows using the same model:

- **Conventional:** prompts, then human review of the output.
- **Spec-first:** scenarios first, TDD, automated guardrails, and exception-driven intervention.

Each developer would use both workflows but never see the same task twice, with tasks and order randomized or counterbalanced so that neither experience nor difficulty lines up with one workflow. Acceptance criteria would be fixed before anyone looked at results. Then someone outside the experiment would judge the work in two separate ways: hidden acceptance tests for the specified behavior, and a maintainer's merge decision, as in METR's note, for whether the work is acceptable. "Passes its own tests" wouldn't count, since that's precisely the claim being tested.

| Dimension | What to measure |
|---|---|
| Human effort | Preparation time; number and duration of interventions; review and rework time |
| Elapsed time | Time until acceptance, including agent execution and time spent waiting for a person or CI |
| Compute cost | Model usage and CI time, which is where the cost of verification appears |
| Outcome | Rework required; defects found by independent evaluation |

I'd keep these separate because they can point in different directions. Hypothetically, the spec-first workflow might need less human effort and still finish later, because stops and long CI runs queue up while the developer is busy with something else. Or it might finish sooner while asking more of the developer's concentrated attention before each increment. Which is better depends on what's scarce.

The limitations are real. Tasks are hard to make comparable, models change between runs, and familiarity with a workflow takes time and favors whichever one a developer already knows. Preparation may also carry over: once you've written scenarios, you may review better even without them. And the spec-first workflow may simply lose on some tasks. I'd expect that on small ones, and a good experiment should be able to show it.

---

## What this asks of a developer

If this holds, the skills that gain value are the ones at the two ends of the loop:

| Moves to the agent | Stays with the developer |
|---|---|
| Writing the implementation and the test wiring | Deciding what is worth building and what is out of scope |
| Iterating from red to green | Writing oracles a no-op cannot satisfy |
| Classifying failures against written rules | Making the check independent of the agent it checks |
| Recording decisions in commits | Answering the brief when the spec runs out |

In practice, I think that means getting better at a few things, even as agents write more of the code and the tests:

- **Defining verifiable behavior.** Spotting a weak oracle, by reading a step definition and asking what would still pass, and writing requirements that could actually fail. "So that the system is more robust" can't fail, so it can't guide anything.
- **Managing decision boundaries.** Finding the implicit assumptions: whether a retry is safe for a non-idempotent operation stays invisible until someone asks. And deciding what can be delegated, where the [classification I describe in the companion article](/software-engineering/executable-supervision/#stopping-is-what-makes-autonomy-safe) is my current attempt.
- **Evaluating correctness.** Keeping implementation correctness apart from business correctness, because the code can do exactly what the scenario says while the scenario is wrong. And understanding the architectural invariants that have to hold across every change, since no single scenario will show them breaking.
- **Keeping technical judgment.** Staying deep enough in the implementation to judge the result. You can't evaluate a review finding, or an agent's claim about a stalled websocket, without understanding the machinery.

That last point has a real counterargument. Delegating too much may slowly wear down the judgment this whole approach depends on. In an [Anthropic randomized trial](https://www.anthropic.com/research/AI-assistance-coding-skills) with 52 mostly junior engineers learning a new library, the group using AI scored 50% on a follow-up quiz, against 67% for those coding by hand, with the biggest gap on debugging. In the authors' qualitative analysis, delegation patterns scored lowest and patterns aimed at understanding scored higher. Anthropic sells these tools, the sample was small, it measured immediate understanding rather than long-term skill, and it didn't look at an agent workflow. Still, it points at a real tension: I'm asking developers to become better judges while removing a lot of the practice that used to build that judgment. I don't have a clean answer. The better-scoring patterns in the trial suggest a partial one: ask the agent to explain, not just to fix.

None of the right-hand column is new. It's what a good Three Amigos session and a disciplined TDD practice have always asked for. I don't think the practices were wrong; they were expensive to teach to people, and the workflow is cheap to hand to an agent. What stays expensive is the judgment about what the scenarios should say. That's where I'd spend my own learning time now, and it seems a strange moment to let that expertise walk out of the door before anyone else has learned it.

How that supervision works, and how much a green build is really worth, is what the companion article, [Executable Supervision](/software-engineering/executable-supervision/), is about.
