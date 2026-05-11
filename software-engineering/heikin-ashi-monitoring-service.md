---
layout: default
title: "Designing a Heikin Ashi Monitoring Service from Scratch"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: 10
date: 2026-05-10
tags: [aws, lambda, dynamodb, java, micronaut, bedrock, heikin-ashi, serverless]
post_excerpt: A serverless monitoring service for equities — computes Heikin Ashi candles, detects patterns, enriches alerts with AI-generated fundamental context, and sends them by email. Designed entirely in a long chat with Claude.
---

# Designing a Heikin Ashi Monitoring Service from Scratch

I wanted a service that watches a small list of equities, computes Heikin Ashi candles on daily and weekly timeframes, detects a few patterns I care about, and emails me when something interesting happens. Not a trading bot. Not a backtesting platform. A monitoring service for medium-to-long-term investment decisions, with enough fundamental context in the alert to know whether the technical signal is worth a second look.

This post is the architectural digest of the system, designed entirely in a long chat with Claude. The next two posts cover the retrospective on what went wrong during that conversation, and the workflow pattern that emerged for using two different Claude products together.

---

## What the System Does, in One Paragraph

A scheduled job runs every evening at 22:00 UTC. For each instrument I've registered, it fetches new OHLC bars from Yahoo Finance, persists them, derives Heikin Ashi bars from them, and runs each new HA bar through three pattern detectors. When a pattern fires, the system renders a chart with the pattern bar highlighted, asks an AI agent to fetch fundamental data and write a short corroboration/contradiction analysis, and sends me an email with everything inline. If the chart or AI step fails, the alert goes into a retry queue and is reprocessed every hour, up to three times. After that, a degraded version of the email is sent regardless, so I never miss a signal entirely.

There is no REST API. There is no web UI yet. The list of instruments is managed via a CLI script. The whole thing runs on AWS Lambda with about ten dollars a month in infrastructure costs at most.

---

## Stack at a Glance

| Layer | Choice |
|-------|--------|
| Language | Java 25 (Amazon Corretto) |
| Framework | Micronaut 4.10 |
| Compute | AWS Lambda + SnapStart, ARM64 |
| Database | Amazon DynamoDB, single-table design |
| Market data | Yahoo Finance via `de.sfuhrm:YahooFinanceAPI`, behind a `MarketDataProvider` interface |
| AI | AWS Bedrock + Claude via `BedrockRuntimeClient.converse()` with manual tool-use loop |
| Charting | JFreeChart in headless mode |
| Email | Apache Commons Email for MIME composition, sent via SES v2 with raw content |
| IaC | Terraform with S3 + DynamoDB lock backend |
| CI/CD | GitHub Actions with OIDC federation to AWS |

A few of these choices look obvious in retrospect, and a few were the result of three or four reversals during the design conversation. The retrospective post covers the reversals.

---

## Why DynamoDB Single-Table

The data model has four interrelated entities: an Instrument, its Configuration, its OHLC bars, and its Heikin Ashi bars. Plus two operational entities: a UniqueLock for ticker uniqueness, and a PendingAlert for the retry queue.

A relational database would model these as six tables with foreign keys. DynamoDB doesn't do JOINs, so the same approach there would mean N round trips for any query that touches more than one entity. Single-table design solves this by colocating related items under the same partition key. Asking for "instrument X plus its config plus its last 50 OHLC bars" becomes a single Query.

The shape ended up like this:

| Entity | `pk` | `sk` |
|--------|------|------|
| INSTRUMENT | `INSTRUMENT#<id>` | `META` |
| CONFIG | `INSTRUMENT#<id>` | `CONFIG` |
| OHLC | `INSTRUMENT#<id>` | `OHLC#<tf>#<bar_time_iso>` |
| HA | `INSTRUMENT#<id>` | `HA#<tf>#<bar_time_iso>` |
| UNIQUE_LOCK | `TICKER#<exchange>#<ticker>` | `LOCK` |
| PENDING_ALERT | `PENDING_ALERT#<event_uid>` | `META` |

Two sparse Global Secondary Indexes carry the rest:

- `gsi_status` lists active or archived instruments without scanning
- `gsi_retry_due` lets the retry poller find expired pending alerts in one Query

Storage policies on bars (`FULL_HISTORY`, `ROLLING_WINDOW`, `SNAPSHOT_ONLY`) are implemented through DynamoDB's native TTL attribute, with a small caveat for `SNAPSHOT_ONLY` where a TransactWrite atomically deletes previous bars and inserts the new one.

I evaluated TimescaleDB on managed Postgres, Aurora Serverless v2, and Amazon Timestream. DynamoDB won because it's the only AWS database with a real perpetual free tier (25 GB + 25 RCU/WCU + 200M requests/month), and the workload volumes for daily and weekly bars on a small watchlist sit comfortably inside that envelope with two orders of magnitude to spare.

---

## Where the Design Got Bent: Stock Splits

The cleanest summary of why this system is shaped the way it is comes from a single failure scenario I had to design around: stock splits and dividends.

Yahoo Finance applies *retroactive* adjustments to historical OHLC bars when a corporate action happens. If a company executes a 2-for-1 split on Monday, by Tuesday morning the historical bars going back ten years are silently rewritten with halved prices and doubled volumes. The new series is internally consistent. The old one no longer exists anywhere accessible to me.

For a system that pulls fresh data daily and stores it forever, this has three cascading consequences:

**1. OHLC bars are not immutable.** The same `(instrument, timeframe, bar_time)` key may legitimately have different values on Tuesday than it had on Monday. Any idempotency guarantee built on conditional puts with `attribute_not_exists` is wrong.

**2. The HA chain is recursive.** `ha_open[t]` depends on `ha_open[t-1]` and `ha_close[t-1]`, all the way back to the seed. If a bar five years ago gets rewritten, every HA value computed from that day onward is stale.

**3. A stale HA chain produces phantom patterns.** Bars that look like reversals because the recomputed values would have placed them on the other side of the open — but the stored values still reflect the pre-adjustment world.

The design absorbs all of this with a single rule: on ingestion, compare the incoming OHLC bar to the stored one. If they differ beyond a small tolerance, treat the incoming bar as the new truth and recompute the HA chain from that point forward. Two lines of code, three paragraphs of specification. The discovery that the rule was needed took most of an afternoon.

---

## The Heikin Ashi Math, Briefly

For each bar at chronological index *t*:

```
ha_close[t] = (open[t] + high[t] + low[t] + close[t]) / 4
ha_open[t]  = (ha_open[t-1] + ha_close[t-1]) / 2
ha_open[0]  = (open[0] + close[0]) / 2     # seed
ha_high[t]  = max(high[t], ha_open[t], ha_close[t])
ha_low[t]   = min(low[t], ha_open[t], ha_close[t])
```

The recursion makes HA chains sensitive to floating-point drift over long sequences. All arithmetic happens in `BigDecimal` with `MathContext.DECIMAL64`. A property-based test using jqwik checks that on chains of 500 bars, no value drifts more than 10⁻⁸ from a reference computation. With `double`, the same test fails by orders of magnitude on chains as short as 100.

---

## The Three Patterns

Each instrument has three patterns, individually enabled and configurable:

| Pattern | Subtype | Condition |
|---------|---------|-----------|
| `color_change` | `bullish_reversal` | current bar is GREEN, previous N bars are all RED |
| `color_change` | `bearish_reversal` | mirror |
| `strong_candle` | `bullish_strong` | bar is GREEN, lower wick under tolerance, body ratio above threshold |
| `strong_candle` | `bearish_strong` | mirror |
| `doji` | `doji` | body ratio under threshold |

A NEUTRAL bar (where `ha_close == ha_open`, rare but possible) breaks streaks. The detection function is pure: it takes the new HA bars and the relevant configuration, returns a list of `PatternEvent` objects with no side effects.

---

## The Alert Pipeline, with Best-Effort Enrichment

When patterns fire, three things have to happen:

1. Render a Heikin Ashi chart of the last 30 bars, with an annotation pointing at the triggering bar.
2. Run an AI agent that decides which fundamental data to fetch and writes a short note on whether the fundamentals corroborate or contradict the signal.
3. Compose a multipart email with both a plain-text version and an HTML version that embeds the chart inline.

Each step can fail independently. The compromise:

- On first failure, write a `PENDING_ALERT` to DynamoDB with `retry_count=0` and `retry_at=now+1h`.
- A separate Lambda (`retry-poller`), triggered by EventBridge every 15 minutes, queries the sparse GSI for items where `retry_at <= now` and reprocesses them.
- After three failed attempts, send a degraded email: if the chart failed, replace the image with a textual placeholder; if the AI failed, leave the analysis section empty with an "(unavailable)" header.

The user always receives an email. The cost of no deduplication is that if a Lambda timeout interrupts dispatch midway, some recipients may receive duplicates. I accepted that trade-off explicitly: I'd rather get a duplicate alert than miss one.

---

## The AI Agent, in the Smallest Viable Shape

Bedrock's Converse API supports tool use natively. The loop:

1. Send a request with system prompt, user prompt, the available tools (six methods on `MarketDataProvider`), and an inference budget.
2. If `stopReason` is `END_TURN`, parse the final JSON block and we're done.
3. If `stopReason` is `TOOL_USE`, execute each requested tool locally, append `toolResult` blocks to the message history, and re-send.
4. Cap iterations at 8. If the cap is hit, send one final request without `toolConfig` to force a wrap-up.
5. If the final output isn't valid JSON matching the expected schema, raise `LLMException`.

The output schema:

```json
{
  "corroborating": "string, optional, max ~80 words",
  "contradicting":  "string, optional, max ~80 words",
  "confidence":     "LOW | MEDIUM | HIGH",
  "data_sources":  ["quote_info", "news", "..."]
}
```

The system prompt explicitly tells the agent to be honest about limited information rather than invent it. Yahoo's fundamental data is patchy for non-US tickers, and I'd rather see "fundamentals unavailable" than a confident-sounding hallucination.

---

## Scheduling and Observability

| Function | Trigger | Memory | Timeout |
|----------|---------|--------|---------|
| `monitoring-main` | EventBridge cron `0 22 * * ? *` | 1024 MB | 900 s |
| `retry-poller` | EventBridge cron `*/15 * * * ? *` | 1024 MB | 300 s |

Both have reserved concurrency of 1. SnapStart drops cold start from 4–8 seconds to ~500ms–1s.

Eight custom CloudWatch metrics in the `Monitoring/HeikinAshi` namespace cover instruments processed, failures, bars inserted, HA bars computed, patterns detected by type, alerts sent (full vs degraded), retry queue depth, and Bedrock tool iterations. Six alarms are wired to an ops topic.

Logs are JSON via Logback's Logstash encoder. The MDC carries `trace_id`, `instrument_id`, `timeframe`, and `bar_time` through the pipeline.

---

## A Note on the Specification Document Itself

The whole system is described in a single `CLAUDE.md` at the root of the repo. That document is more than documentation: when a language model reads it, the document is executed semantically as part of the prior the model uses to reason about every subsequent change. A stale section of `CLAUDE.md` is not stale documentation; it is active misguidance injected into future reasoning loops.

This has a structural consequence. The natural temptation is to organise by domain: data model section, ingestion section, dispatch section. The better organisation, I now think, is by *frequency of expected access*. Decisions that are stable and read once at project bootstrap belong at the bottom. Decisions and conventions that the tool consults on every change belong at the top. The volatile parts of the document carry more drift risk per byte than the stable parts.

---

## What Got Cut

- **Bounce and complaint handling via SES SNS topic.** Useful but not v1.
- **Optimistic locking on configuration.** I'm the only writer.
- **Multi-tenancy.** Same reason.
- **Backtesting.** This is a monitoring service, not a research tool.
- **Web UI.** CLI script + DynamoDB items is enough for now.

---

## What's Next

The next post covers the retrospective: how the design conversation went, which decisions reversed three times, and what went wrong epistemically. The third post covers the workflow pattern I ended up adopting, including the regimes where it doesn't apply.

The full `CLAUDE.md` specification will go up alongside the code once the first version ships.
