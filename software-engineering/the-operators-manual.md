---
layout: default
title: "Heikin Ashi – Part 5: The Operator's Manual"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20260520
date: 2026-05-20
tags: [claude, anthropic, ai-assisted-development, debugging, operations, runbook]
post_excerpt: "Debugging a running serverless system is not design chat. A third mode — operational chat — has its own rules: one command per turn, hypotheses labelled as hypotheses, file paths verified not guessed. A taxonomy of where my own deployment sessions broke those rules and why it mattered."
---

# The Operator's Manual: One Command at a Time

*Part 5 of 6 in the [Heikin Ashi series](#series-navigation).*

The Two Claudes pattern describes two modes: chat for thinking, code for building. After deploying the Heikin Ashi service and spending an unexpectedly large fraction of the project debugging it, I realized the pattern was missing a third mode that I had been doing badly without naming. The previous post dealt with preventing boundary failures through verification at build, deploy, and runtime. This one deals with what happens after prevention is exhausted — when the system is running, something is wrong, and the loop between the human and the AI has to find the cause.

The third mode is *operational*: chat used not for design and not for implementation, but for guiding the human through a live debugging session against a running system. It looks superficially like design chat — same surface, same model, same conversational rhythm — but it has different rules, different failure modes, and a different definition of "useful response." Treating it as design chat is a categorical error that costs hours.

This post is an attempt to name the mode, articulate its discipline, and describe where my own session repeatedly violated it.

---

## What operational chat actually is

A concrete example. The Lambda is deployed. It returns `Runtime.BadFunctionCode`. I paste the error into chat. The AI proposes a hypothesis. I run a command. I paste the output. The AI proposes the next step. Repeat until the system runs.

That loop is not a conversation about how the system should be designed. It is a structured exchange where:

- The human's terminal is the only source of ground truth.
- The AI's role is to direct attention, not to produce solutions.
- Every claim must be reducible to the next observable check.
- Each command's output may invalidate the next step the AI was about to propose.

The third bullet is the one most often violated. In design chat, the AI's value is partly proportional to how much synthesis it produces per turn — comparison tables, considered options, weighed trade-offs. In operational chat, that same density is actively harmful, because it commits to a chain of reasoning before the data is in.

The shift is from "produce the best answer" to "produce the next observable step." Design chat optimizes for fluency. Operational chat optimizes for falsifiability.

---

## The one rule that does most of the work

In design chat, bundling multiple steps in one response is efficient: "first consider X, then Y, then Z, settle on Y because Z fails on edge case W." The human reads, follows the chain, picks up where it lands.

In operational chat, the same bundling is destructive. The output of step 1 may be that the system is in a state the AI did not anticipate, in which case steps 2 and 3 are wrong and have to be re-issued. The human now has to scroll back to find the right command, the AI has to re-explain, and both parties have wasted a turn.

The discipline is simple in statement and hard in practice: **one command per response. Wait for the output. Then propose the next.**

More precisely: one observable step per turn. In most terminal-driven debugging that means one command, but the rule generalizes — if a step is sometimes "read this file" or "open this URL," that still counts as one observation. The single-command framing is a useful proxy because it removes ambiguity about what "one step" means in the moment.

My own deployment session violated this rule dozens of times despite explicit instruction. Each violation followed the same pattern: the AI would comply for two or three turns, build conviction about what was happening, and then revert to bundling because the multi-step plan felt locally efficient. The user — me — would then have to interrupt mid-execution to ask which command to run, or would execute the first and discover the second made no sense in light of the result.

There is no good reason for the regression except that the AI's defaults favor synthesis over patience. In operational mode, the human is the rate limiter, and the AI's job is to keep up with the human's actual pace, not the pace it could sustain alone.

---

## The other rule: never speculate as diagnosis

The second discipline of operational chat is the discipline against false confidence. When the AI doesn't know why something is failing but has to respond, it pattern-matches to a plausible explanation. Plausibility is a signal of training-data fluency, not of factual correctness.

During the deployment session, examples included:

- "Yahoo blocks AWS IP ranges" (false — was User-Agent classification)
- "Bedrock requires cross-region inference profile for newer models" (partially true, but not the cause of the failure being investigated)
- "Google News RSS works from Lambda" (untested speculation)
- "The 476 alerts failed because Bedrock" (insufficient evidence — could have been chart rendering, prompt construction, or any other step in the pipeline)

Each of these sounded plausible. Each was either wrong or unsupported. In every case, the human (me) pushed back, the AI retreated, and the next response was more accurate. The cost was not the wrong claim itself — wrong claims are recoverable in a few turns. The cost was the trust degradation across the rest of the session: once the AI has guessed and been caught, every subsequent confident claim has to be re-evaluated by the human, which is slow.

The discipline is to explicitly mark uncertainty. "This is a hypothesis; the way to test it is X" produces the same useful output as "the cause is Y" but does not commit either party to a frame that may be wrong. The verb matters. *Hypothesizing* invites verification. *Diagnosing* invites action.

---

## The third rule: verify, don't infer, file paths

The smallest of the three rules, and the easiest to ignore. When operational chat involves commands against the filesystem or against AWS resources, the AI is tempted to invent paths or identifiers based on plausibility. "The IAM policy is in `terraform/main/iam.tf`" sounds reasonable for a project with the expected structure, but if the project has restructured the file is somewhere else, and the human's `cat` against a non-existent path produces 30 seconds of confusion before the AI corrects itself.

Two practices remove the friction:

1. When in doubt, propose a `find` or `ls` first, then act on the discovered path.
2. When committing to a path, frame it conditionally: "assuming the standard layout, the file is at X — if not, find it with..."

These are cheap. The cost of not doing them is recurring small frictions that compound over a long session.

---

## What operational chat looks like when it works

The best stretch of my deployment session was the diagnosis of the fat jar packaging chain. The AI proposed a single hypothesis (the jar is the thin jar, not the shaded one), asked for one piece of evidence (the file size from `aws s3 ls`), got the answer, and moved to the next hypothesis based on the data. Each turn produced one new fact. Four cascading failures got diagnosed and fixed in roughly an hour of operational chat, which is fast for that class of problem.

The worst stretch was the IAM Marketplace investigation. The AI proposed multiple causes in parallel, suggested two unrelated fixes, and built reasoning on top of an incorrect initial hypothesis. The human (me) eventually pasted the full exception body and the diagnosis became immediate. The intervening half-hour was spent reasoning inside a frame the data did not support.

The difference between the two stretches was not the AI's underlying capability. It was the operational discipline. The fat jar diagnosis stayed within one hypothesis per turn. The IAM diagnosis sprawled across multiple hypotheses before any of them was tested. The same model, the same human, the same kind of problem, two different outcomes determined entirely by the chat's rhythm.

The failure mode was not lack of intelligence. It was mode mismatch. The model did not need to be smarter. It needed to be slower in the right way.

---

## How operational chat relates to the rest of the workflow

The Two Claudes pattern, as originally described, gave one job to chat (think) and one to code (build). Operational chat is a third surface that doesn't fit cleanly into either. It happens in chat, but it's not design. It's debugging, but it doesn't happen in Claude Code, because Claude Code doesn't have AWS credentials in my setup and isn't supposed to.

Where it actually lives:

| Surface | Job |
|---|---|
| Chat (design mode) | Architecture, decisions, prose deliverables |
| Claude Code (implementation mode) | Reading and writing files, running tests, iterating on the codebase |
| Chat (operational mode) | Live debugging against the running system from my terminal |

The same chat surface serves both design and operational modes, but the disciplines are different enough that the AI should signal — and the human should signal — which mode the conversation is in. In my own session this never happened explicitly. The mode transitions were implicit, and the AI's behavior didn't always shift to match. The cost was the bundling regression, the speculation-as-diagnosis episodes, and the occasional invented file path.

A small additional discipline would help: at the start of an operational thread, name it. "I'm debugging a runtime failure now, not designing — please give me one command per turn and label speculation." That sentence costs nothing and resets the mode explicitly. I have not been disciplined about saying it, and the result is that operational threads have drifted back into design-style responses several times per session.

---

## The runbook angle

Operational chat is, in effect, a runbook being assembled in real time. The output of a successful operational thread is not just a fixed system; it is a sequence of commands that, in retrospect, would have diagnosed and fixed the issue if run in order. That sequence is reusable. The next time the same class of failure happens — a missing env var, a misconfigured IAM action, a packaging defect — the prior session's commands form the skeleton of the diagnosis.

I have not yet been disciplined about extracting these. The session has surfaced at least three reusable diagnostic sequences:

| Failure class | Reusable diagnostic sequence |
|---|---|
| Lambda fails to boot | check alias's published version → compare against latest version → fetch artifact → verify size and contents |
| Bedrock returns `AccessDeniedException` | read exception body for exact denied action → compare role policy against the action → simulate with `aws iam simulate-principal-policy` |
| Third-party API behaves differently from Lambda than from the terminal | reproduce locally with the library's exact request configuration → vary one header at a time → compare against a working baseline |

Each of these is the skeleton of a short command checklist. None of them is currently written down anywhere except buried in chat history. Promoting them out of chat and into a `RUNBOOK.md` is the next discipline step I haven't yet taken.

---

## What this means for the design-time discipline

The Two Claudes pattern is good. The production-envelope problem from the previous post is real and partially addressable through boundary checks. Operational chat is the third leg, the one that handles the inevitable cases where the design was correct, the envelope held, and the system still failed for some local, contingent, hard-to-anticipate reason.

The honest sequence of disciplines, as I now see them:

1. **Design discipline** (`CLAUDE.md` + design chat) gives you a system that should work.
2. **Envelope discipline** (boundary checks) gives you confidence that what you deployed is what you specified.
3. **Operational discipline** (one command at a time, no speculation, verify paths) gives you a way to recover when (1) and (2) prove insufficient, as they will.

The third discipline doesn't replace the first two. It catches what they miss. The fact that it's necessary doesn't mean the first two failed; it means the system is sufficiently complex that no design or envelope check can cover everything, and the human-plus-AI debugging loop is the residual mechanism that resolves the rest.

---

## What I'd tell the AI on the first operational turn

If I had to write a single instruction to paste at the top of any operational chat, it would be this:

> We are debugging a running system. Give me one command per turn, wait for the output before proposing the next step, and explicitly label hypotheses versus diagnoses. If you need a file path or identifier, ask me to find it rather than guess. Trust my pushback if I contradict a claim.

That sentence does not solve operational chat. It just keeps both parties pointed in the right direction. The discipline beyond that — the actual debugging skill — is what the AI brings on top, and the quality of that skill is the only reason this mode is worth having at all.

The next post in the series, if there is one, will probably be the one I've been postponing: the `DEPLOYMENT.md` companion to `CLAUDE.md`, with the boundary checks and the operational runbooks treated as production artifacts. The design half is documented. The operational half still lives in muscle memory. That is the next thing to fix.

---

## Series navigation {#series-navigation}

| | |
|---|---|
| ← Part 4 | [Heikin Ashi – Part 4: From Spec to Production](/software-engineering/from-spec-to-production/) |
| → Part 6 | *Coming soon* |
