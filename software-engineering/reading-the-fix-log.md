---
layout: default
title: "Agent Symbiosis – Part 3: Reading the Fix Log"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20261115
date: 2026-11-15
tags: [claude-code, ai-assisted-development, bdd, tdd, mutation-testing, code-review, retrospective, workflow]
post_excerpt: "I went through every fix commit of a project built with a coding agent, looking for the ones a better process could have prevented. The safety net mostly held; what it cost was rounds of review spent finding, one at a time, things a table, a mutation or an invariant would have shown at once."
description: "A commit-by-commit look at the fixes in an agent-built project: what the automated supervision caught, what it found only after several rounds, and the small process changes that would move each kind of discovery upstream."
---

# Reading the Fix Log: What an Agent's Mistakes Say About the Process

*Part 3 of 3 in the [Agent Symbiosis series](#series-navigation).*

The first two parts of this series were about a way of working: in [the first](/software-engineering/who-is-waiting-for-whom/) I described why a developer and a coding agent end up waiting for each other, and in [the second](/software-engineering/executable-supervision/) how executable specifications, stop rules and independent checks can supervise the agent while I'm not looking. Both were written from the inside of the method, and I wanted to finish with something written from the outside, looking back at what actually happened rather than at what was supposed to happen.

So I went through the history of [Ferrifuge](https://github.com/jbiscella/ferrifuge), the Rust client for Centrifugo I've been building this way, and read every commit that fixed something. The question I kept asking each one was a simple one: could a change in the process, rather than more care from the agent, have prevented this, or at least found it sooner?

---

## What the numbers say, roughly

The project has 686 commits over about three weeks of calendar time, and since the workflow tags every commit with its phase, the fixes are easy to find: 185 commits are tagged as fixes, against 149 tagged green, the phase where a scenario first passes. More fixes than green commits sounds alarming at first, but I think it says less than it seems, because a single green commit can carry a whole scenario while a fix can be a one-line format correction, so I'd treat these counts as a rough picture rather than a measurement.

Two things in that picture seem worth noting, though. About half of the fix commits mention the review by a model from another vendor, and many of the others come from CI, so the great majority of what went wrong was caught before it reached the main branch, which is the net doing its job. And the ratio changed over time: in the first four days there were around three fixes for every green commit, while in the last four it was close to one. Some of that is the project getting easier, I'm sure, but some of it is the process learning, and that second part is what this post is about.

What follows are the kinds of fix I found, in the order in which they seemed to me most preventable.

---

## Rounds of review where a table would have done

The most expensive pattern wasn't a bug but a rhythm. In several areas the cross-model review came back with a finding, the agent fixed it, the next review found another case close to the first, and so on, for four, five, and in one area eight rounds.

That area was the handling of a laptop going to sleep. When the machine wakes up, timers are overdue, tokens may have expired, and whichever input arrives first, a frame from the server, a closed connection, a timer firing or a call from the application, reveals that time has jumped. Each round of review found a new combination of *which input revealed the sleep* and *what state the client was in at that moment* that the previous fix had handled in the wrong order. Every finding was real, and every fix was correct for its case, but the cases were being discovered one at a time, by an expensive reviewer, after the code was written.

Looking back, the shape of the problem was a [decision table](/software-engineering/executable-supervision/#beyond-examples): the kinds of input along one side, the client's states along the other, and in each cell the order in which the sleep should be handled. Laid out before the red phase, the table would have shown the empty cells at once, and each empty cell is exactly the kind of gap that the stop rule from part two exists for. I don't think it would have removed the review rounds entirely, because tables describe combinations and not every interleaving between threads, which is where [scenarios stop](/software-engineering/executable-supervision/#where-the-mechanism-breaks) anyway; but it would have turned eight rounds of discovery into one round of filling in a table, and most of the questions would have reached the reference systems, or me, before any code depended on the answers.

The change to the process, then, would be a small one: when an increment is about ordering, timing or state, ask for the table of inputs and states before the first failing test, and treat an empty cell as a question rather than as an implementation detail.

---

## Tests that passed for the wrong reason

The second pattern was the one part two worried about most, and here it was, in the log. Early on, the steps checking the reconnection backoff accepted an implementation capped at 19 seconds instead of 20, a "jitter" with only two possible values, and delays below the configured minimum; it took five fixes to make them strict, each prompted by the reviewer describing a wrong implementation that would still have passed. Later fixes followed the same shape elsewhere: a step that accepted a disconnect with any code, one that couldn't tell whether an unsubscribe had ever been sent, a check of the network subprotocol that passed whatever the client had asked for.

What I find encouraging is that the agent already had the right technique, from the second day: break the implementation on purpose, see the test still pass, strengthen the test, and see it fail on the same broken code. The commits say so explicitly, often with the mutation they used. What I find instructive is that this remained something the agent did when a reviewer prompted it, and it only became an acceptance criterion of each block of the spec, "each scenario is seen failing once on a mutation … recorded in the PR", near the end of the period I looked at. I wrote in part two that I hadn't wired a mutation tool into my CI; that is still true, but the manual version, done for every scenario and recorded where a reviewer can see it, turned out to be cheap enough to require, and I'd require it from the first block next time.

---

## The same bug, one instance at a time

Some fixes repeated each other. The client must never print a token, and the reviews found, over several rounds, five different places where one could leak: the debug output of the configuration, of the internal state machine, of the actions it queues, of the inputs it receives, and the text of an HTTP error. Text files read on Windows arrived with different line endings and broke a check, and two weeks later a second, unrelated check broke in the same way. The test runner ran out of stack on Windows, and later, with a larger world to carry, it ran out again on every platform.

Each fix was right, and each was local. What none of them did was turn the second occurrence into a rule that covers the third. In the case of secrets the project eventually got there, with a scenario that captures every log record and fails if any of them contains a secret, which protects every path at once, including the ones nobody has written yet.

This is where I think the human part of the symbiosis is clearest. The agent fixes the instance in front of it, as it's asked to; noticing that a class of defect is coming back, and deciding that it deserves an invariant, a shared helper or a check that runs everywhere, is a judgment about the process, and that judgment is mine to make. A simple rule would help the agent bring it to me: when a fix resembles an earlier one, say so in the commit, and propose the invariant.

---

## Timing, and failures that can explain themselves

A couple of dozen fixes had to do with timing on CI runners, which are slower and less predictable than my machine, above all on Windows and macOS. Most of them were test races rather than client bugs: a step that started pinging only after the client had armed its deadline, a measurement taken from the wrong moment, a fake server that recorded a command after answering it, so that a quick step could read the record before the command was in it.

One intermittent failure is worth telling in some detail, because of how it was solved. A scenario on Windows failed now and then with a timeout, and rather than guessing, the agent made the failure message richer, in three steps over three days, each time adding when another part of the scenario had actually run. The next time it failed, the message showed that the step meant to settle the call had run six seconds after it, past its five-second timeout, so the runner had stalled and the client had timed out exactly as it should. Guessing has a cost of its own: elsewhere, a quick fix for another intermittent failure made it worse and had to be replaced.

Three habits would have prevented most of this, and they are habits of test writing more than of implementation: wait for the event you need rather than for a fixed time, measure from the cause rather than from when the step happens to look, and write every timing assertion so that, when it fails, it says enough to tell a slow runner from a wrong client. These belong in the skill, next to the rules about red and green, because the agent writes the tests too.

---

## A check that nobody reads

Exception-driven supervision has a quiet precondition, which is that the exceptions reach someone. Some jobs in the project run only on the main branch or at night, because they are slow, while the work happens on pull requests, so when the browser job for Safari broke on a missing command-line tool, it failed on every run on main for most of a day before anyone noticed; earlier, a hang on Windows and a race on macOS had also shown up on main first.

None of these did lasting harm, but they show a gap in the loop rather than in the code: a check whose failure nobody reads is not supervising anything. The fix is plumbing, a way for failures on main and in the nightly runs to come back to the agent as work, and it's the kind of plumbing that is easy to postpone because everything looks green from where the work is happening.

---

## Small slips, and the ones nobody could prevent

Around ten fixes were formatting, lint or documentation errors that a local check would have caught before the push, and one commit message says it plainly: "The previous commit went out without its rustfmt pass." These are cheap to fix, but each one costs a CI run and a round trip, and a pre-push hook makes them disappear.

Then there were the fixes no process could have prevented: a dependency release withdrawn by its authors, security advisories published after the code was written, a tool that stopped being maintained. A daily dependency check found them, which is the right answer for that class of problem, although in one case it had been failing for four days before anyone looked, which is the previous section all over again.

It's also worth saying what didn't show up as a fix. The ambiguities in the spec, the questions about which code reconnects and when a token should be refreshed, were mostly settled by the reference systems I described in part two, and the few that weren't came to me as a brief with a proposed answer. That part of the process held up well, and the log is quieter there than I expected.

---

## The process, revised

Put together, the changes are small, and none of them adds a new kind of human checkpoint. This is the diagram from part two with them added in green, while the amber steps are still the only places where I'm involved:

[![The workflow from part two with five additions: an input-by-state decision table before the agent iterates; each scenario seen failing on a mutation; local checks before the push; failures from main and nightly runs fed back to the agent; and, at the human checkpoint, recurring fixes promoted to invariants in the specification](/assets/images/reading-the-fix-log-workflow.svg)](/assets/images/reading-the-fix-log-workflow.svg)

*Click the diagram to open it at full size.*

| Kind of fix | Caught by | Change that moves it upstream |
|---|---|---|
| New ordering case each review round | Cross-model review | Inputs-by-state table before the red phase |
| Test passes on a wrong implementation | Review, mutation by hand | Mutation check required for every scenario |
| Same defect in another place | Review, CI on another platform | Second occurrence becomes an invariant |
| Timing race in a test | CI on slower runners | Wait for events, measure from causes, diagnosable failures |
| Failure only on main or at night | Nobody, for a while | Failures fed back to the agent |
| Format and lint | CI | Pre-push hook |
| Withdrawn dependency, advisory | Daily dependency check | None needed |

What strikes me, reading this table back, is that every row on the right is a change to the process rather than to the code, and the agent couldn't have made any of them on its own initiative, because each one changes the rules it works under. It fixed every defect it was shown, often very well; deciding that a kind of defect should never need fixing again is the part of the work that stays with me.

That is perhaps the most concrete form the symbiosis from part one has taken: the agent turns specifications into working code and finds, round after round, where the specification and the checks fall short, and I read where it struggled and change the specification and the checks so that next time it struggles less. The fix log turned out to be the best place to have that conversation, because it records what both of us got wrong, in order, without either of us having to remember it.

---

## Series navigation {#series-navigation}

| | |
|---|---|
| ← Part 2 | [Agent Symbiosis – Part 2: Executable Supervision](/software-engineering/executable-supervision/) |
