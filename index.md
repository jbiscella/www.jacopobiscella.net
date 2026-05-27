---
layout: home
title: Home
nav_order: 1
description: "Software engineering notes on AI-assisted development with Claude Code, Java systems on AWS Lambda, and Heikin Ashi trading tools — by Jacopo Biscella, backend engineer based in Nyon, Switzerland."
---

# Jacopo Biscella

Senior Backend Engineer / Software Architect · Nyon, Switzerland

Most of my work has been on systems that have to keep running while people use them, often in high-regulatory environments — compliant archiving at Global Relay, plus pricing engines, order flows, booking platforms, and services across transport, government, banking, and insurance. That constraint shapes the practice: keep the design small, make failures legible, and notice where complexity accumulates between parts that each look reasonable in isolation. The technology stack has changed a lot over the years; that orientation hasn't.

This site is where I write down the things I want to keep. The engineering notes are the largest section, but the same instinct shows up elsewhere. I write about [tabletop miniature painting](https://www.jacopobiscella.net/blog/miniature-hobby/) the way I'd write about a tooling experiment, because the questions turn out to be similar: what should specialize, when is the popular default still the right choice, and where does a small wager beat a large one. I'm also a passionate amateur photographer; if that grows into its own section, it will probably follow the same habit of looking closely at framing, constraint, and consequence.

The recent focus has been on working seriously with AI tools — not as a productivity shortcut, but as a collaborator that fails in unfamiliar ways. The [Heikin Ashi series](https://www.jacopobiscella.net/software-engineering/heikin-ashi-monitoring-service/) is the long version of that investigation, built around an actual side project I've taken from design to production. What emerged is not a recipe, but a record of the disciplines, artifacts, and failure modes that shaped the work.

## Personal Projects

**[H-tchen Mail](https://github.com/jbiscella/H-tchen-Mail)** — A serverless daily watchlist monitor that emails me only when a stock forms a meaningful Heikin Ashi pattern. Built on AWS Lambda, Java 25, Micronaut, DynamoDB, and Bedrock. I built this deliberately as a vibe-coding experiment — using Claude as a design and implementation partner throughout — to understand in practice where AI-assisted development works well and where it breaks down. The [project overview](/software-engineering/heikin-ashi-sentinel/) describes what it does; the [six-part engineering series](/software-engineering/heikin-ashi-monitoring-service/) documents the design conversation, the workflow that emerged, and the production failures the spec couldn't prevent.

**[ha-track](https://github.com/jbiscella/ha-track)** — A modular Java toolkit for technical analysis of financial price data, centred on Heikin Ashi candles. Provides chart rendering, pattern detection, and backtesting as independently usable modules. The library that both H-tchen Mail and wichtelm-app are built on.

**[wichtelm-app](https://github.com/jbiscella/wichtelm-app)** — A backtesting tool for single-instrument trading strategies, where strategies are defined in a readable Gherkin-based DSL rather than code. Takes CSV or EODHD historical data, handles multi-timeframe logic, and produces self-contained HTML reports with equity curves, aggregate metrics, and trade analysis. Built with Java 25. Another vibe-coding experiment.

**[skills-dungeon](https://github.com/jbiscella/skills-dungeon)** — A personal container of skills I extract and accumulate through retrospectives with my AI assistants. Rather than letting the insights from each session evaporate, I feed them back into a structured repository that grows with use.

**[functional-validator](https://github.com/jbiscella/functional-validator)** — A lightweight Java validation library built around a fluent builder API: composable predicates, lambda-defined error messages, nested object support, and an `Optional`-returning result. A half-afternoon prototype exploring whether bean validation boilerplate can be replaced with something more readable.

---

[Browse the Blog](/blog/) · [View CV](/cv/) · [LinkedIn](https://www.linkedin.com/in/jacopobiscella/)
