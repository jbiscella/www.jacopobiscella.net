---
layout: default
title: "A watchlist sentinel that emails me only when a stock does something interesting"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20260516
date: 2026-05-16
tags: [aws, lambda, java, micronaut, trading, side-project]
post_excerpt: A serverless daily watchlist watcher that stays silent until a stock forms a meaningful Heikin Ashi pattern — then sends one email with a chart, the numbers, and AI-generated context. The inbox is the interface.
description: "A serverless daily watchlist watcher that stays silent until a stock forms a meaningful Heikin Ashi pattern — then sends one email with a chart, the numbers, and AI-generated context. The inbox is the interface."
---

# A watchlist sentinel that emails me only when a stock does something interesting

I don't want to stare at charts. I want to *not* stare at charts, and still
not miss the moment a stock I care about turns. So I built a small service
that does the staring for me and emails me — only when something is worth a
second look.

It's called **[H-tchen Mail](https://github.com/jbiscella/H-tchen-Mail)** — the Heikin Ashi Monitoring Service. You hand it a handful of
tickers; it watches them once a day and stays silent. Most days nothing
arrives. When one of them forms a meaningful candle pattern, a single email
shows up: a chart, the numbers, and a short plain-language note. **The inbox
is the interface.** No dashboard, no app, nothing to open.

It is a *notification* tool, not an advisor. Every alert is a nudge to go
look — never an instruction to trade.

## Why Heikin Ashi

Normal candlestick charts are noisy. *Heikin Ashi* — Japanese for "average
bar" — redraws each candle blended with the one before it. The result is a
smoother chart where trends and turning points stand out and day-to-day
jitter fades. It's much easier to read at a glance, which makes it a good
basis for *automated* reading too.

The service watches three patterns on those smoothed candles:

- **Colour change** — a run of same-coloured candles is broken by the
  opposite colour. Heikin Ashi trends tend to hold one colour, so the first
  flip after a streak is a classic early hint of a possible reversal.
- **Strong candle** — a big body with almost no wick: price moved one way
  decisively for the whole period. Conviction, not a half-hearted drift.
- **Doji** — a tiny body, open and close almost equal. A tug-of-war between
  buyers and sellers, which often sits right before a turn.

Each pattern is tunable per stock, and I pick which ones I care about.

## What an alert actually does

Once a day a scheduled job wakes up and, for every stock on the list, runs
four stages:

1. **Ingest** — fetch the latest closed daily and weekly bars from a market
   data API and store them.
2. **Heikin Ashi** — compute the smoothed candles from the raw prices,
   deterministically (all decimal arithmetic, no floating-point drift).
3. **Detect** — evaluate the three patterns on the freshly computed bars.
   Only the most recent bar can raise an alert, so the very first run on a
   stock — which ingests years of history at once — doesn't unleash an
   avalanche of stale alerts.
4. **Dispatch** — for each detected pattern, render a chart, ask an AI for a
   short fundamental-analysis note, compose an email and send it.

That AI note is the part I'm happiest with. A language model is given the
pattern that fired and a set of read-only tools — recent news headlines,
analyst ratings, earnings dates — and asked to write a few honest sentences:
what in the recent news *supports* this signal, what *weakens* it, and how
confident it is. It's explicitly told to admit when the data is thin rather
than invent a story. So the email isn't just "a doji formed" — it's "a doji
formed, and here's the context for and against reading anything into it".

If the chart or the AI step fails, the alert isn't lost: it's queued and
retried, and after a few attempts it degrades gracefully to a plainer email
rather than failing silently.

## The stack, and why

The whole thing runs **serverless on AWS Lambda** — no server to keep alive
for a job that fires once a day. The pieces:

| Concern        | Choice                                            |
|----------------|---------------------------------------------------|
| Language       | Java 25                                           |
| Framework      | Micronaut (compile-time DI — fast Lambda starts)  |
| Storage        | DynamoDB, single-table design                     |
| Charting       | JFreeChart, headless                              |
| AI             | AWS Bedrock + Claude, with a tool-use loop        |
| Email          | SES, multipart text + HTML with an inline chart   |
| Infrastructure | Terraform                                         |
| CI/CD          | GitHub Actions, OIDC — no static AWS keys         |

A few decisions I'd defend:

- **Decimal arithmetic everywhere.** Prices and the Heikin Ashi chain use
  `BigDecimal`, never `double`. Over a long chain of derived candles,
  floating-point error accumulates; for a tool whose whole job is reading
  candle shape, that's unacceptable.
- **Ports and adapters.** The business logic — Heikin Ashi maths, pattern
  detection — is pure functions with no I/O. Market data, AI, email and the
  database all sit behind interfaces. The core never imports an AWS SDK
  class. It makes the interesting code trivial to test.
- **The spec is the source of truth.** The project is described in one long
  specification document, with every behaviour written as Given/When/Then
  scenarios. Those scenarios are *executable* — they run as the acceptance
  test suite. The code is meant to mirror the spec, not the other way round.

## What I learned

Two things, mostly.

First: an executable spec is only as good as the discipline behind it. While
recently auditing the test suite I found two behaviours the spec clearly
described but the code never actually implemented — including a retry path
that, on a bad day, would silently drop a whole day of data. The spec said
so plainly; the gap was that no test had ever forced the issue. A scenario
that can pass *whether or not* the feature works isn't a test, it's
decoration.

Second: the boring operational glue is where the real time goes. Getting a
model enabled, getting one IAM permission right, making sure the thing that
got deployed is the thing you built — none of that is glamorous, and all of
it will happily waste an afternoon if you debug by guessing instead of by
checking.

## Where it's going

It works end to end today. The natural next steps are a proper CLI for
managing the watchlist, a few more patterns, and tightening the gap between
"the spec says" and "the code does" until there isn't one.

If you've ever wanted a quiet, opinionated little robot that watches your
stocks and respects your attention — that's the whole idea. The inbox is the
interface.

The project is [on GitHub](https://github.com/jbiscella/H-tchen-Mail). For the engineering retrospective — how the design was built in chat, the workflow that emerged, and the production failures — see the [six-part series](/software-engineering/heikin-ashi-monitoring-service/).
