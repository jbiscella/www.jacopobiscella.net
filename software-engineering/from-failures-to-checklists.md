---
layout: default
title: "Heikin Ashi – Part 7: From Failures to Checklists"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20260525
date: 2026-05-25
tags: [claude, anthropic, ai-assisted-development, checklists, retrospective, discipline]
post_excerpt: "Every AI-assisted project session generates failures worth keeping. Almost none of them get kept. The final discipline in the series: extracting what went wrong into artifacts the next session can actually use."
description: "Every AI-assisted project session generates failures worth keeping. Almost none of them get kept. The final discipline in the series: extracting what went wrong into artifacts the next session can actually use."
---

# From Failures to Checklists: Closing the Loop

*Part 7 of 7 in the [Heikin Ashi series](#series-navigation).*

The previous post catalogued five patterns through which AI confidence fails. The post before that catalogued three disciplines for operational chat. The posts before those catalogued the Two Claudes pattern, the production envelope problem, and the failures of the design conversation that started the project.

Six posts of named failure modes. A single question I have been avoiding: what does naming accomplish, if the names live in chat history and nowhere else?

The discipline I have been worst at across the entire Heikin Ashi project is the one I have been postponing writing about. Not the operational discipline, which I violated dozens of times but at least understood while violating it. Not the confidence-reading discipline, which I have applied inconsistently but can articulate. The worst one is simpler: I have not, in any systematic way, extracted the project's failures into artifacts that would make them harder to repeat.

This post is an attempt to do that extraction, and to describe the practice of doing it.

---

## Why the extraction fails by default

A design conversation ends. The system is deployed. Something failed; we diagnosed it; it's fixed. The chat window sits open, containing, somewhere in its forty-hour scroll of context, a sequence of commands that would have found the bug in twenty minutes if run in the right order. A hypothesis that was false. A false hypothesis that cost an hour. The correct hypothesis and the commands that confirmed it.

None of that goes anywhere unless someone actively moves it.

The default outcome is that the failure pattern stays in the chat window, associated with the specific session, invisible to the next session, and completely useless the second time the same class of failure occurs. The second time, you start from zero. The AI starts from zero. The thirty-minute detour is lived again in approximately the same form.

This happens even when both parties are trying to avoid it. The AI doesn't maintain state across sessions. The human remembers in rough outline but not in command-sequence detail. The discipline that would prevent the repetition is the discipline of writing things down — specifically, of writing them down in the right form and in the right place.

---

## What the right form looks like

The right form is not a narrative retrospective. A narrative retrospective is valuable for understanding the *why* of a failure — which is why this series is structured as one. But a narrative is not what you want during the next operational session, when you are staring at a `Runtime.BadFunctionCode` error and need to remember which three commands will tell you whether the artifact is the wrong version.

The right form for operational reuse is a checklist.

A checklist for AI-assisted operational work has a specific structure that differs from a general checklist. It has three components:

1. **The trigger.** A concrete description of the state that activates the checklist. Not "Lambda fails" but "Lambda returns `Runtime.BadFunctionCode` or `Runtime.ImportModuleError`." Specific enough that the human knows without judgment whether they're in this scenario.

2. **The sequence.** An ordered list of commands or observations, each of which confirms or rules out one hypothesis. The sequence is ordered by cheapness: the observations that cost least and eliminate most go first. The sequence ends when the failure is isolated or when the checklist is exhausted.

3. **The escalation.** What to do when the sequence doesn't find the cause. Usually: widen the search. Sometimes: escalate to a different diagnostic surface.

These three components reflect what the operational-mode conversation actually needs to do well: trigger recognition (so the right checklist gets activated), sequence discipline (so the human doesn't run the wrong command at the wrong moment), and escalation (so a negative result has a defined next action rather than an open-ended re-diagnosis).

---

## What the Heikin Ashi checklists look like

Part 5 of this series gestured at three reusable diagnostic sequences. Until now, they have not been written in a form that survives the session. Here they are.

**Checklist: Lambda boot failure**

*Trigger:* Lambda invocation returns `Runtime.BadFunctionCode`, `Runtime.ImportModuleError`, or `Runtime.UserCodeSyntaxError`.

1. `aws lambda get-alias --function-name <fn> --name live` — confirm which version the alias points to.
2. `aws lambda get-function-configuration --function-name <fn> --qualifier <version>` — confirm the code SHA.
3. `aws s3 ls s3://<bucket>/<key>` — check artifact size. Below 1 MB is the thin jar, not the shaded jar.
4. If thin jar: check Maven build configuration. The shaded plugin must be bound to the `package` phase, not `install`. Re-build, re-upload, re-publish.
5. If size is correct: download and inspect the manifest. `jar tf <file>.jar | grep Handler` — confirm the handler class is present at the expected path.

*Escalation:* If the handler is present and correctly named, check environment variables: `aws lambda get-function-configuration --function-name <fn> --query 'Environment'`.

---

**Checklist: Bedrock AccessDeniedException**

*Trigger:* Lambda invocation fails with `AccessDeniedException` from the Bedrock service.

1. Read the full exception body. Extract the exact denied action (e.g., `bedrock:InvokeModel` vs `bedrock:InvokeModelWithResponseStream`).
2. `aws iam list-role-policies --role-name <lambda-role>` — list inline policies.
3. `aws iam get-role-policy --role-name <lambda-role> --policy-name <policy>` — inspect the relevant policy.
4. Confirm the denied action is present in the policy. If not, add it.
5. If the action is present: confirm the resource ARN. For Bedrock, `arn:aws:bedrock:<region>::foundation-model/<model-id>` requires the exact model ID including inference profile variants.
6. `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names bedrock:InvokeModel --resource-arns <model-arn>` — simulate before re-deploy.

*Escalation:* If simulation passes but Lambda still fails, check whether the model requires an AWS Marketplace subscription. Model access requirements are per-region and may differ from what the console shows.

---

**Checklist: Third-party API behaves differently from Lambda than from terminal**

*Trigger:* A third-party HTTP API call succeeds locally but fails or returns unexpected results from Lambda.

1. Reproduce locally using the *exact* library and configuration the Lambda uses — not `curl`, not a different library, not different defaults.
2. Capture the outgoing request headers. Log `request.headers` before the call.
3. Compare User-Agent between local and Lambda execution. Lambda has no default User-Agent for most HTTP libraries; many APIs classify requests by User-Agent.
4. Reproduce locally with the Lambda's User-Agent to confirm the classification hypothesis.
5. If confirmed: set an explicit User-Agent string that matches a known-good pattern. For Yahoo Finance, a browser-like User-Agent passes classification; the library's default does not.

*Escalation:* If User-Agent is correct and failure persists, check IP-based rate limiting. Add a delay between calls. Test with a single-item request.

---

## The design-time counterpart

Checklists for operational sessions address failures that happen after deployment. The design-time counterpart addresses failures that happen during the design conversation — the premise drift and false architectural confidence patterns from posts 2 and 6.

The design-time checklist is simpler because design failures are recoverable without urgency. It just needs to interrupt the conversation periodically and ask the right questions.

**Checklist: Design session health check** *(run every 5–10 turns)*

1. What is the implicit frame we're operating in? Can I state it in one sentence?
2. Is that frame still consistent with the constraints I stated at the start?
3. In the last five turns, has any recommendation arrived with named specifics — versions, tiers, limits — but without the constraints that justify them? If so, ask: *"Under what constraints is this the right choice?"*
4. Has any claim been stated as diagnosis rather than hypothesis? If so, ask: *"What observation would confirm or falsify this?"*
5. Is the specification being built here the right audience for this output? Or should Claude Code be producing this, not the design chat?

This is not something to run on a timer. It's something to run when the conversation has been flowing smoothly for several turns — because that's exactly when the failures above are accumulating most invisibly.

---

## The meta-discipline: extraction as a project milestone

The checklists above took longer to write than the operational sessions that generated them. That ratio is wrong. The extraction should cost less than the session, not more — otherwise the amortization math never works out.

The right time to do the extraction is at the end of each non-trivial session, while the failure is fresh. Not in a separate document, not days later — in a five-minute pass through the chat before closing the window. The questions:

1. What hypothesis was wrong? Write the correct one down in trigger-sequence form.
2. What identifier did the AI invent that turned out to be wrong? Add it to the relevant checklist.
3. What premise drifted? Write the frame-check question that would have caught it.

Five minutes, while the session is still open. The alternative is writing these posts six weeks later from memory and getting the details approximately right but not exactly right — which is what I have been doing.

---

## The honest state of this project's checklists

Three checklists, above. There should be more. There are at least two others I know I need and have not written:

- A checklist for "the alert rate is wrong" — where the Lambda is running, Bedrock is answering, but the signal threshold or the Heikin Ashi calculation is producing alerts at the wrong frequency.
- A checklist for "the email is not arriving" — where the alert is generated in the logs but the SES dispatch fails silently, which happened once.

Both failures have been debugged and resolved. Neither has been written down. They live in the chat history. If either failure recurs, I will probably spend the same time diagnosing it.

The checklist habit is not yet a habit. It is a discipline I understand clearly and practice inconsistently. That gap — between understanding a discipline and having internalized it — is the honest ending of this series.

Understanding comes from naming. Naming came from writing these posts. The next step is the boring one: running the extraction at the end of every session, not as a retrospective after six weeks, but as a five-minute practice before closing the window.

---

## What the series amounts to

Six posts of analysis and one post of practice. The practice post is the one that matters, which is why it should have come earlier and will come earlier in the next project.

The disciplines, in the order they apply:

| When | Discipline | Artifact |
|---|---|---|
| Before the design conversation | Start with a complete first message | `CLAUDE.md` |
| During the design conversation | Check the frame every 5–10 turns | Updated `CLAUDE.md` |
| Before deployment | Run boundary checks | `DEPLOYMENT.md` |
| During operational debugging | One command per turn; hypothesis not diagnosis | Operational checklists |
| After any AI-assisted session | Extract failures into checklists | `RUNBOOK.md` |
| At any point | Read AI output as candidate, not answer | Confidence-reading practice |

The common structure: a trigger, a practice, an artifact. The artifact is what survives the session. The practice is what produces the artifact. The trigger is what makes you run the practice rather than closing the window and moving on.

The Heikin Ashi service is running. The disciplines are partially in place. The artifacts are partially written. The gap between "partially" and "fully" is the work that remains.

---

## Series navigation {#series-navigation}

| | |
|---|---|
| ← Part 6 | [Heikin Ashi – Part 6: When AI Confidence Lies](/software-engineering/when-ai-confidence-lies/) |
