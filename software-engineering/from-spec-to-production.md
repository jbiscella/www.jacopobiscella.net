---
layout: default
title: "From Spec to Production: What CLAUDE.md Can't Prove"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: 14
date: 2026-05-17
tags: [claude, anthropic, claude-code, aws, lambda, iam, terraform, devops]
post_excerpt: Eight hours debugging production-only failures on a system that took thirty hours to design and build. A taxonomy of the boundary defects that CLAUDE.md can state but is structurally unable to prove.
---

# From Spec to Production: What CLAUDE.md Can't Prove

*Part 4 of 6 in the [Heikin Ashi series](#series-navigation). The project's full `CLAUDE.md` spec is [on GitHub](https://github.com/jbiscella/H-tchen-Mail/blob/main/CLAUDE.md).*

The previous three posts described a Heikin Ashi monitoring service, the design conversation that shaped it, and the workflow pattern that produced it. They all live in the design-time half of the project. This post is about what happened when the design met production: the class of failures that the specification could express in principle but had no way to enforce in practice.

The distinction matters, and it's the central claim of this post. `CLAUDE.md` is excellent at declaring what should be true about the application. It is structurally unable to *prove* that what it declares actually holds across the boundary between code and infrastructure. The eight hours I spent debugging production-only failures, on a system that took roughly thirty hours to design and build, were all spent in that gap: between an invariant being correctly stated somewhere and being silently violated somewhere else.

What follows is an attempt to characterize the gap, with concrete cases, and to articulate the discipline that fills it.

---

## Application versus production envelope

The clearest mental model I've arrived at separates two regions of the system:

| Application | Production envelope |
|---|---|
| Entities, ports, flows, errors | Build artifacts, runtime config, IAM identities |
| Domain operations, Gherkin scenarios | Library defaults, third-party API quirks |
| Pure logic, testable in-process | Cross-artifact contracts, only visible at runtime |

`CLAUDE.md` is well-suited to the left column. It can describe what entities the system manages, what error codes propagate, what the alert pipeline produces. All of that is propositional content that the specification can express and that application-level tests can verify.

The right column is what I started calling the production envelope — everything between the application boundary and the running system. The specification can make claims about the envelope ("the Lambda runtime must expose `MONITORING_BEDROCK_MODEL_ID`"), but those claims are *cross-artifact*: they hold only if multiple independent pieces of the toolchain — Maven, Terraform, GitHub Actions, AWS IAM, third-party libraries — happen to align. The application's tests don't run against the deployed artifact. They run against the IDE classpath, where everything is present by default.

The aggregate behavior of the system can be wrong because an invariant *between* artifacts was silently violated, even when every individual artifact is locally correct. That sentence is the central observation of this post.

---

## Four failures, each of a different shape

Four production failures from the Heikin Ashi project, chosen because each one points at a different kind of cross-artifact contract.

### Failure 1: The fat jar that wasn't

The application packages as a Micronaut fat jar deployed to AWS Lambda. The intent: Maven produces `target/heikin-monitor-0.1.0-SNAPSHOT-shaded.jar`, CI uploads it, Lambda runs it.

The reality, on the first deploy attempt, was that Maven produced `heikin-monitor-0.1.0-SNAPSHOT.jar` — 330 KB, the thin jar, with no dependencies. The CI workflow had a fallback `target/*.jar` glob that quietly selected the thin jar because the shaded one didn't exist. Lambda loaded a jar that could not boot. The error surfaced as `NoClassDefFoundError: io/micronaut/function/aws/MicronautRequestHandler`.

Fixing the shade plugin produced a 40 MB fat jar. Lambda loaded it. New error: `Error resolving property value [${monitoring.exchanges.supported}]. Property doesn't exist`. The `application.yml` was nominally in the jar but was being excluded from the shaded output by default filter rules. Fixing that produced a fat jar with `application.yml` at classpath root. New error: same one. `application.yml` was now in the jar, but the `META-INF/services` file that registers Micronaut's YAML loader had been overwritten during the shade merge instead of concatenated. The final cause, after fixing the merge, was that a transitively required runtime dependency had been excluded by an inherited filter elsewhere in the parent POM.

Three or four cascading failures, each one masking the next. None of them appeared in `mvn verify` because `mvn verify` runs tests against the IDE classpath, where everything is present by default. The fat jar is a derived artifact whose contents are not validated by any standard Maven phase.

The specification could have asserted: "the application must boot from a fat jar in a clean JVM with no compile-time classpath." That assertion is propositional content. What it could not do is *verify* the assertion against the actual artifact Maven produced on a given build. The verification has to happen elsewhere — as a CI step that runs the jar in a separate JVM and observes whether it boots.

### Failure 2: The environment variable that traveled through five layers and arrived nowhere

The application reads its Bedrock model ID from `monitoring.bedrock.model-id`, with a default in `application.yml` overridden per-deploy via the environment. The deploy chain looked like:

| Layer | What it does | What it had |
|---|---|---|
| GitHub Variable `BEDROCK_MODEL_ID` | Set per repo | Configured |
| Workflow `env.TF_VAR_bedrock_model_id` | Picked up by Terraform | Configured |
| Terraform `var.bedrock_model_id` | Used in IAM and SSM | Used correctly |
| Terraform `aws_lambda_function.environment.variables` | Should inject into runtime | **Missing the key** |
| Lambda runtime | Reads env, falls back to YAML | Read the YAML default |

The variable existed at every layer except the one that mattered. Four out of five layers were correctly wired. The fifth — the actual injection into the Lambda's environment block — was simply absent from `lambda.tf`. Terraform happily applied. CI happily deployed. The Lambda ran, read its environment, found no `MONITORING_BEDROCK_MODEL_ID`, fell back to whatever model ID was hard-coded in `application.yml`, and called Bedrock against that.

The model in the YAML default had not been requested in the AWS account. In Bedrock, model access is account-and-region-specific, so a syntactically valid model ID can still fail at runtime if the account hasn't been granted access in that region. The model the deploy thought it was using was Opus. The model that ran was an older Haiku. The IAM policy granted access to the Opus inference profile. Everything was internally consistent and globally wrong.

This is the most demoralizing class of failure because almost nothing a normal review process is likely to catch. The defect was visible only through cross-artifact comparison, not through ordinary local review. The Terraform diff looks correct. The SSM parameter is set. The IAM policy is right. The CI logs show the variable being passed. Only the Lambda runtime knows the env var never arrived, and it doesn't tell you — it just falls back silently.

The specification could have stated: "every property the application reads from the environment must be set by the deployment module." The statement is unambiguous. What's missing is anything that compares the application's `@Value` annotations against the Terraform module's `environment.variables` block and fails the build when they diverge. That comparison is a boundary check; nothing in the standard toolchain runs it.

### Failure 3: The IAM policy that worked for the user but not for the role

Bedrock's Converse API returned `AccessDeniedException` when called from the Lambda. The exception body included the actual diagnosis: `the IAM user or service role is not authorized to perform the required AWS Marketplace actions (aws-marketplace:ViewSubscriptions, aws-marketplace:Subscribe) to enable access to this model`.

The Lambda role had `bedrock:InvokeModel`. It had `bedrock:Converse`. It had the resource ARN for the inference profile and for the underlying foundation model. What it didn't have were the AWS Marketplace permissions that newer Anthropic models require because they are technically sold through Marketplace listings rather than as native AWS service offerings. That requirement is not visible from the Bedrock console, the Bedrock IAM examples, or any of the standard tutorials.

I discovered the requirement by calling the same Converse API from my own admin SSO credentials and observing that it worked. The diff between "works from my session" and "fails from the Lambda" was the Marketplace permission, which my admin role had implicitly via `AdministratorAccess` and the Lambda role did not.

This is the third shape of production failure: the cross-identity gap. A capability works in one identity context and fails in another. The fix is mechanical once diagnosed: add the missing actions to the role. The hard part is diagnosing it without speculating — the exception message points at the answer, but only if you read it carefully and resist the urge to assume the problem is in the more familiar Bedrock policy.

The specification could have said: "the Lambda's role must hold every permission required to invoke the configured Bedrock model." Again, true. What was missing was an `aws iam simulate-principal-policy` check, run before deploy, that would have surfaced the gap mechanically.

### Failure 4: The library that lied about its User-Agent

The first version of the market data adapter used an unofficial Yahoo Finance client. Locally it worked. From Lambda it returned `HTTP 429 Too Many Requests` immediately, with a log warning that the session-establishing cookie call had failed.

A controlled test from my own terminal showed the cause. `curl` against the same Yahoo endpoint returned 429 with the default `curl/X.Y.Z` User-Agent, and returned a normal 301 redirect with a browser-like User-Agent. Yahoo's edge layer classifies User-Agent strings and rejects ones that look like scrapers, regardless of origin IP. The library exposed a `userAgent` configuration parameter but only applied it to some of its internal HTTP calls, not to the one that mattered most.

The fix was to abandon the library and switch the market data provider, behind the existing `MarketDataProvider` abstraction, to direct HTTP calls against EODHD. The architectural abstraction paid for itself: the swap touched one adapter class and a configuration property, with no changes elsewhere.

The lesson is about the boundary, not Yahoo specifically. A specification can state that the third-party library must work in the runtime environment, against the upstream's current expectations, with the library's exact runtime configuration. What it cannot do is keep that claim true as the upstream API, edge filtering, and library behaviour change. That information lives in the integration's empirical reality and degrades over time without warning.

---

## A taxonomy of boundaries

Each of the four failures occurs at a different boundary, and each boundary admits a different kind of verification:

| Boundary | Example invariant | Enforcement |
|---|---|---|
| Build artifact | shaded jar exists, contains config, services merged | post-build artifact inspection |
| Config | every property the app reads is set in the deployment module | static cross-reference of `@Value` annotations and Terraform |
| Identity | the runtime role can call every API the application invokes | `aws iam simulate-principal-policy` against expected actions |
| Provider | the external API responds correctly from the runtime environment | scheduled CI smoke call with the production library configuration |
| Runtime | the deployed function version matches the artifact CI uploaded | post-deploy SHA verification |

Each row is a propositional fact, declarable in `CLAUDE.md`. None of them is observable from inside the application. All of them require an external check, run at a specific point in the build-deploy-run cycle, against an actual artifact rather than against a model of one.

The point of the taxonomy is not to be exhaustive. It is to make the boundary itself the unit of analysis — instead of asking "does the application work," ask "which boundaries does this deploy cross, and what verifies each?"

---

## The rule I would now apply

Every design-time assumption that crosses an artifact boundary needs one of three things:

1. a build-time check (the artifact contains what the spec says it contains),
2. a deploy-time check (the deployment satisfies the invariants the application requires),
3. or a runtime smoke check (the running system, exercised end-to-end, produces the expected result).

If an assumption has none of the three, it is not an invariant. It is a hope. The four failures above were all hopes that I had mistaken for invariants because the spec stated them clearly and I assumed the toolchain would honour them.

---

## What this changes about `CLAUDE.md`

Two updates to my mental model.

The first: `CLAUDE.md` is now explicitly scoped to the *application*. The production envelope gets its own artifacts — a deployment runbook, a CI verification script, a set of boundary tests. They are versioned together with the spec but are not part of it. I had been treating `CLAUDE.md` as the single source of truth for the system; it is now the single source of truth for the application.

The second: I now distinguish, in my own thinking, between content the model should reason from and content the model should be aware of but prevented from modifying without explicit instruction. The application's data model is the first kind. The Lambda's IAM policy structure is the second kind: I want the model to know it exists, but I do not want it to propose changes to it without prompting, because the model's prior over IAM policies is shaped by training data that doesn't match my specific deployment context. The distinction matters because including production-envelope details in `CLAUDE.md` actively widens the model's sense of what it can change.

---

## Practical recommendations

Five changes I'm adopting in response to the experience, in order of decreasing priority:

1. **Boundary tests in CI**, one per row of the taxonomy above. The build artifact check is the cheapest and most generally applicable; the identity check pays back any time the deployment uses a service with non-obvious permission requirements (Bedrock + Marketplace being the canonical example).

2. **A `DEPLOYMENT.md` next to `CLAUDE.md`**, capturing envelope invariants explicitly. Not as a tutorial — as a list of contracts that must hold, each with a pointer to the check that verifies it. Stale entries are easier to spot than in `CLAUDE.md` because the file is shorter and exists for one purpose.

3. **A post-deploy verification step** that compares the running Lambda's published version with the artifact CI just uploaded. Two-line check, would have surfaced one of the more frustrating failures (alias pointing at the wrong version after a partial deploy) in seconds rather than half an hour.

4. **An explicit "envelope mode" in the chat workflow**, distinct from design mode. When the conversation moves from "should the system do X" to "why isn't it doing X in production," the operating discipline shifts: one command per turn, verified observations only, no speculation about causes until the data is in.

5. **A short failure log** in the repo, growing over time. Each entry: the symptom, the root cause, the boundary it sits on, the check that would have prevented it. Treating production failures as data, not stories.

---

## What's next

I'm still in the middle of writing the production-envelope discipline. The list above is partial; the boundary tests are partly written; the failure log has three entries. The honest current state is that I'm doing this work by hand, after each failure, with the discipline I should have applied before. That's not a workflow. It's pattern recognition turning into method, slowly.

The first three posts described how to think about design with AI tools. This post described what the design can't reach. If a fifth post comes out of the series, it will be the one that tries to write the boundary discipline down properly — probably the `DEPLOYMENT.md` referenced above, with the boundary tests treated as production code rather than as scaffolding around them.

For now, the takeaway I'd offer to anyone applying the Two Claudes pattern to a real deploy: the spec can make the intended system legible. The boundary checks are what make the deployed system accountable.

---

## Series navigation {#series-navigation}

| | |
|---|---|
| ← Part 3 | [The Two Claudes Pattern: Chat for Thinking, Code for Building](/software-engineering/the-two-claudes-pattern/) |
| → Part 5 | *Coming soon* |
