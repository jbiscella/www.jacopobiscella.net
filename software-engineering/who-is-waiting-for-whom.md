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

This year I've built several projects with a coding agent doing most of the implementation, from [H-tchen Mail](https://github.com/jbiscella/H-tchen-Mail), [ha-track](https://github.com/jbiscella/ha-track) and [wichtelm-app](https://github.com/jbiscella/wichtelm-app) to, more recently, [Ferrifuge](https://github.com/jbiscella/ferrifuge), a Rust client for Centrifugo. Working this way for a while, I noticed that the question I kept coming back to wasn't so much how fast the agent was, but who was waiting for whom, and why. My bet is that, in the near future, the developers who do best with these tools will be the ones who learn to work with an agent so that neither of them ends up waiting for the other, or cleaning up after the other.

Sometimes the bottleneck was me: the agent had finished its work, and that work sat there until I found the time to look at it properly. At other times it went the other way round, and that was much harder to spot, because from the outside it didn't look like waiting at all. The agent had quietly made a decision I had never asked it to make, and I only realized it later, when I had to go through a long diff to understand why it had done what it did.

The way I've come to think about it is as a kind of symbiosis, in the working sense rather than the biological one: each side does what the other can't, and each removes the other's reason to wait. The agent needs constraints it can check and someone to decide when the specification runs out, while I need someone who can implement and verify without me watching every step.

What helped most, in my experience, were two practices that I've often seen teams set aside as a waste of time, BDD and TDD. I suspect that the reason they were set aside has changed in the meantime. They used to be expensive because people had to learn them slowly, and most projects simply didn't have that time, whereas an agent can be given the whole workflow at the start of every session: the scenarios tell it what to build, a failing test tells it what is still missing, and the move from red to green tells it when it is done. In between, it's the tests and the automated checks around them that do the supervising, which means I don't need to be watching.

I'd like to be careful about what this means, because it's easy to say more than is true. The tests don't replace supervision; together with the other automated checks around them, they are the supervision: the agent decides how to satisfy the constraints but not what those constraints should be, and whenever the specification and the references it relies on can't answer a question, that question comes back to me.

What follows is my reading of my own experience rather than the result of a controlled comparison, and the wider evidence is still mixed. In this post I look at the economics of working this way and at what they might mean for developers, QA and business analysts, while its companion, which follows this one, looks at how the mechanism works and where it breaks.

---

## Two bottlenecks, one in each direction

Once I started looking at it this way, it became clear that the two directions are quite different from each other, both in how they show up and in what it takes to deal with them:

| | The human blocks the agent | The agent blocks the human |
|---|---|---|
| **Symptom** | Every result waits for someone to look at it | The diff is plausible and wrong, and you have to work out why |
| **Cause** | No check the agent can run itself | Ambiguity the agent resolved silently |
| **Remedy** | A pass/fail signal the agent can read | A spec precise enough to expose the ambiguity, and a rule to stop on it |

The first one is easy to recognize, and the [Claude Code best practices](https://code.claude.com/docs/en/best-practices) describe it well when they say that "Without a check it can run, 'looks done' is the only signal available, and you become the verification loop: every mistake waits for you to notice it."

The second took me much longer to see, because an agent that guesses doesn't look blocked at all; it looks productive, and the cost only shows up later, when you have to reconstruct from a long diff a decision it made on your behalf.

### Three costs, and why fewer interruptions is not the goal

When I try to write these down as costs rather than symptoms, I end up with three of them:

1. **The human waits for the agent.** This can be the least disruptive of the three, as long as there's something else to do in the meantime and the result can be trusted when it arrives, but it becomes expensive as soon as that result is blocking something important. Without the trust, moreover, it isn't really waiting at all, but watching. And trust depends on visibility as well: if I can't see what the agent is doing, or what it is waiting for, I end up checking on it anyway, and that is watching too.
2. **The agent waits for the human.** Every review, clarification or approval stalls the run, which is why most agent tooling, quite understandably, focuses on removing this cost first.
3. **Rework.** Someone finds out that a finished result is wrong, and then has to work out why, decide what it should have done instead, and redo it, together with whatever has been built on top of it in the meantime.

These three aren't interchangeable, and they aren't the whole bill either, since elapsed time and the cost of verification and compute are separate quantities again, and I don't know of any general ranking between them. What I have come to distrust, though, is attacking only the second one, because an agent that never stops never visibly blocks anyone, and it can keep producing code while rework quietly piles up behind it.

A hypothetical example helps to make this concrete. Imagine an agent that is asked to add retries to an upstream call without any precise acceptance criteria: it picks a retry count, decides which exceptions count as transient and chooses a fallback, all without asking anything, and the feature looks complete. Weeks later someone notices that, after a transient failure, it returns an empty result instead of the quote. That agent never waited for anyone, and yet it still blocked its developer.

The defect itself is only part of the cost, because the rest of it is the investigation that follows: why that retry count, which failures it treated as transient, why that particular fallback, and whether anyone had actually agreed to any of it. Had a scenario stated the returned value, the number of calls and what happens when retries run out, there would have been much less to reconstruct, since those decisions would have been written down before the code, and a check would have failed as soon as one of them was broken.

This is why I've stopped thinking of generation throughput as the thing to optimize, and why what I actually want more of is accepted, correct increments. That is still only a proxy, and not a measure of business value in itself, but it's a much better one than generated code, and the two can turn out to be surprisingly far apart:

- METR asked active maintainers to [review 296 AI-generated pull requests](https://metr.org/notes/2026-03-10-many-swe-bench-passing-prs-would-not-be-merged-into-main/) that had passed SWE-bench's automated tests, and found that roughly half wouldn't have been merged, even after adjusting for the human patches those same maintainers also rejected. The agents had a single attempt and no feedback from a reviewer, so this isn't a ceiling on what they can do, but it does show how much distance there can be between "passes the tests" and "acceptable."
- [Faros's telemetry](https://www.faros.ai/blog/ai-productivity-paradox-report-2025), gathered from over 10,000 developers, associates high AI adoption with 98% more merged pull requests and 91% longer review times, without a significant correlation with throughput or quality at company level. Since Faros sells engineering analytics and the data is correlational, I read it as a description of the risk rather than as proof of a cause.
- [DORA's 2025 report](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report) finds AI adoption positively related to delivery throughput and negatively related to stability, and describes AI as an amplifier, with an explanation that is close to what I'm suggesting here: "Without robust control systems, like strong automated testing, mature version control practices, and fast feedback loops, an increase in change volume leads to instability."

My working hypothesis is that a spec-first workflow moves effort around rather than removing it, asking for a bit more preparation before each increment in exchange for less review and correction afterwards. I haven't measured that trade, and I wouldn't expect it to pay off every time: on a small task, writing the scenarios can cost more than fixing whatever the agent gets wrong, and exploration works only as long as the scenarios stay thin enough to change. Automated supervision also has its own bill. Human effort, elapsed time and resource consumption are three different quantities, and in a long autonomous run they can come apart: the developer's effort can stay low while most of the elapsed time goes into waiting for CI and automated reviews, and the local machine and the CI are shared, limited resources that the agent itself can end up saturating.

Parallel work fits into the same accounting. A workflow could be worth it even if each increment takes the agent longer, as long as it needs much less of the developer's attention and leaves them free to look after other independent work. That too is a hypothesis, and it has some obvious limits, because parallel work adds coordination, shared dependencies and long CI runs get in the way, more agents running at once don't automatically mean more throughput, and the developer's own cognitive load may simply become the new bottleneck. Further down I suggest a way in which these trades could be measured.

---

## The objection was about people

The best-known industrial data on TDD comes from [a case study of four teams](https://doi.org/10.1007/s10664-008-9062-z), three at Microsoft and one at IBM, in which pre-release defect density fell by 40% to 90% compared with similar projects, while the teams reported, subjectively, a 15% to 35% increase in initial development time.

On top of that there's a cost the study doesn't capture, which is that a team has to learn to write a good test first, or a good scenario, and that kind of learning is slow and uneven. Looking at both costs together, I can understand a manager who decided it wasn't worth it.

With an agent in the loop, though, three things look different to me:

1. **The workflow transfers in seconds.** The one I use is a [317-line file](https://github.com/jbiscella/skills-dungeon/blob/main/skills/code/incremental-implementation-workflow/SKILL.md), and every session that loads it starts from the same rules, with no onboarding and no months of pairing. That doesn't mean the agent will apply them well, since judgment doesn't come with the file, and this is exactly why the checks need to be independent.
2. **The typing has moved.** Since the agent writes both the test and the implementation, the initial-time penalty no longer falls on the developer in the same way. This is my own inference, as I don't know of any study that has repeated those measurements with agents.
3. **The test has a second reader.** A test used to be mostly a safety net for people, whereas for an agent it is also the most precise definition of "finished" that it can check by itself.

---

## The mechanism, briefly

In short, a scenario states a requirement and, once its steps are wired to meaningful assertions, becomes the check for it. The agent writes a failing test, implements until it goes green, and stops with a written brief whenever the requirements are ambiguous, contradictory or incomplete and nothing it's allowed to rely on settles the question.

A check like the retry scenario above is what I mean by an oracle, that is, the thing that decides whether a result is right. A green result tells you that the code complies with those checks, but not that the business got what it needed, and since the agent will work just as hard against a weak oracle as against a strong one, a lot depends on who writes them. The companion article, which follows this one, goes through the scenarios, the skills and the ways in which all this can fail.

---

## The people being treated as overhead

Some companies seem to be drawing the opposite conclusion: if agents write both the code and the tests, then QA and business analysis look like the obvious place to save. I've heard of a company that recently cut its QA team sharply and, in practice, replaced much of it with Claude Code.

That is a single secondhand example, and I can't say how common this is. [Coverage of an Indeed report](https://www.businesstoday.in/technology/news/story/ai-engineers-in-qa-testers-out-indeed-report-tracks-sharp-shift-in-tech-job-market-489695-2025-08-16) lists QA testers among the roles falling out of favor, without giving a figure, while for business analysts the data I found points the other way, with [one tracker](https://www.extern.com/post/will-ai-replace-business-analysts-guide) showing US postings rising 17% between July and October 2026. What I'm describing, then, is a direction I think I'm seeing rather than a measured trend. For me, though, the question isn't really who holds these titles, or at what ratio, but whether anyone still holds the expertise.

If an agent's loop is only as good as its oracles, then defining those oracles becomes the scarce work, and that is precisely what these roles have been practicing for years. A good business analyst asks who the actor really is, whether the value can be falsified and what's out of scope, while a good tester asks what a check actually observes, which edge cases are missing and whether the test would pass no matter what the implementation did. Two of the three personas in [`three-amigos`](https://github.com/jbiscella/skills-dungeon/blob/main/skills/chat/three-amigos/SKILL.md), my refinement skill, are a Business Analyst and a Tester, and that's deliberate, because their questions are what the skill is made of. These are often the people who are most disciplined about exactly the preparation an agent needs, and that preparation is what these cuts risk taking away.

I'm saying this as a developer, not as someone defending another profession. I spent years in regtech, and there I often saw that the most stable products didn't come from the most technically brilliant developers, but from the ones the environment had forced to learn QA techniques, BDD and TDD. That's an observation rather than a study, and regulated environments differ in other ways too, such as audits and the pace of change, but it has shaped what I believe, which is that discipline makes up for talent more often than talent makes up for missing discipline.

It also cuts against a simple defense of job titles, because those developers show that the techniques can move to other people, and some of the work will reasonably move to agents, repetitive manual regression above all. What I don't think can be dropped is the verification and requirements expertise itself, wherever it ends up sitting, since removing the people who hold it without asking anyone else to learn it leaves you with a faster producer working against weaker oracles.

---

## Where the argument breaks

The technical objections, such as tests that can be gamed and scenarios that stop at the application boundary, are discussed in the companion article, so here I'll stay with the economic ones.

**The cost moved.** The learning curve hasn't disappeared, because writing a scenario with a real oracle, or a value clause that could actually turn out to be false, is still the hard part, and now it sits with whoever writes the spec. There's also a subtler risk, which is that the persona pass can make a weak spec feel reviewed, and the [`three-amigos` README](https://github.com/jbiscella/skills-dungeon/blob/main/skills/chat/three-amigos/README.md) says plainly that it doesn't replace stakeholder validation. Three personas played by one model don't know what your users need, and since much of BDD's original value came from conversations between real people, a simulation can't reproduce it.

**The productivity evidence is unsettled.** METR's [early-2025 trial](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) found that experienced open-source developers took 19% longer with AI, and its [February 2026 update](https://metr.org/blog/2026-02-24-uplift-update/) has point estimates leaning toward a speedup, but with confidence intervals that include zero, so that METR itself calls the new data "an unreliable signal," partly because developers increasingly declined to work without AI. Neither study tested a spec-first workflow, which means that my bet on who will be competitive is a reading of where things are going, not something anyone has measured.

**It doesn't fit everything equally.** Exploration doesn't rule it out, because the formalization happens in small increments, with a thin scenario for what you know now that changes as you learn more; on H-tchen Mail, for instance, I retrofitted Gherkin as the way to define the rules. What I warned about in the [Two Claudes pattern](/software-engineering/the-two-claudes-pattern/) still applies to formalizing a large spec too early, when it hardens a shape you haven't earned yet, and the overhead remains the wrong tool for anything judged by perception rather than propositions, or for scripts you'll delete on Friday. [The workflow skill's README](https://github.com/jbiscella/skills-dungeon/blob/main/skills/code/incremental-implementation-workflow/README.md) also excludes pure exploratory spikes with no spec to enforce, as well as codebases with no test infrastructure.

---

## How this could be tested

Everything I've cited so far sits next to the question rather than answering it, since the TDD study predates agents and METR didn't test a spec-first workflow. I haven't found anything showing that this way of working beats the alternative, and my own experience comes from my projects rather than from a controlled comparison, so what follows is an experiment I'd like to see run, not one I have run.

The idea would be to take comparable but different development tasks in an existing codebase, and two workflows using the same model:

- **Conventional:** prompts, then human review of the output.
- **Spec-first:** scenarios first, TDD, automated guardrails, and exception-driven intervention.

Each developer would use both workflows but never see the same task twice, with tasks and order randomized or counterbalanced so that neither experience nor difficulty lines up with one of the workflows, and the acceptance criteria would be fixed before anyone looked at the results. Someone outside the experiment would then judge the work in two separate ways, with hidden acceptance tests for the specified behavior and a maintainer's merge decision, as in METR's note, for whether the work is acceptable. "Passes its own tests" wouldn't count, since that is precisely the claim being tested.

| Dimension | What to measure |
|---|---|
| Human effort | Preparation time; number and duration of interventions; review and rework time |
| Elapsed time | Time until acceptance, including agent execution and time spent waiting for a person or CI |
| Compute cost | Model usage and CI time, which is where the cost of verification appears |
| Outcome | Rework required; defects found by independent evaluation |

I'd keep these measurements separate because they can point in different directions. Hypothetically, the spec-first workflow might need less human effort and still finish later, because stops and long CI runs queue up while the developer is busy with something else, or it might finish sooner while asking more of the developer's concentrated attention before each increment, and which of the two counts as better depends on what's scarce.

The limitations are real. Tasks are hard to make comparable, models change between runs, and familiarity with a workflow takes time and favors whichever one a developer already knows. Preparation may also carry over, since once you've written scenarios you may review better even without them, and the spec-first workflow may simply lose on some tasks; I'd expect that on small ones, and a good experiment should be able to show it.

---

## What this asks of a developer

If this holds, the skills that gain value are the ones at the two ends of the loop, and the table below is, in a sense, the symbiosis written down: on the left what the agent can take over, on the right what it can't do without us.

| Moves to the agent | Stays with the developer |
|---|---|
| Writing the implementation and the test wiring | Deciding what is worth building and what is out of scope |
| Iterating from red to green | Writing oracles a no-op cannot satisfy |
| Classifying failures against written rules | Making the check independent of the agent it checks |
| Recording decisions in commits | Answering the brief when the spec runs out |

In practice, I think this means getting better at a few things, even as agents write more of the code and the tests:

- **Defining verifiable behavior.** This means spotting a weak oracle, by reading a step definition and asking what would still pass, and writing requirements that could actually fail, because "so that the system is more robust" can't fail, and so it can't guide anything.
- **Managing decision boundaries.** This means finding the implicit assumptions, since whether a retry is safe for a non-idempotent operation stays invisible until someone asks, and deciding what can be delegated, where the classification I describe in the companion article is my current attempt.
- **Evaluating correctness.** This means keeping implementation correctness apart from business correctness, because the code can do exactly what the scenario says while the scenario itself is wrong, and understanding the architectural invariants that have to hold across every change, since no single scenario will show them breaking.
- **Keeping technical judgment.** This means staying deep enough in the implementation to judge the result, because you can't evaluate a review finding, or an agent's claim about a stalled websocket, without understanding the machinery.

That last point has a real counterargument, which is that delegating too much may slowly wear down the very judgment this whole approach depends on. In an [Anthropic randomized trial](https://www.anthropic.com/research/AI-assistance-coding-skills) with 52 mostly junior engineers learning a new library, the group using AI scored 50% on a follow-up quiz against 67% for those coding by hand, with the biggest gap on debugging, and in the authors' qualitative analysis delegation patterns scored lowest while patterns aimed at understanding scored higher. Anthropic sells these tools, the sample was small, it measured immediate understanding rather than long-term skill, and it didn't look at an agent workflow; still, it points at a real tension, because I'm asking developers to become better judges while removing a lot of the practice that used to build that judgment. I don't have a clean answer to this, although the better-scoring patterns in the trial suggest a partial one, which is to ask the agent to explain and not just to fix.

None of the right-hand column is new, since it's what a good Three Amigos session and a disciplined TDD practice have always asked for. I don't think the practices were ever wrong; they were simply expensive to teach to people, whereas the workflow is cheap to hand to an agent. What stays expensive is the judgment about what the scenarios should say, and that is where I'd spend my own learning time now, which is also why it seems a strange moment to let that expertise walk out of the door before anyone else has learned it.

How that supervision works in practice, and how much a green build is really worth, is what the companion article is about.
