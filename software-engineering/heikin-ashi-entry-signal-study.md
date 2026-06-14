---
layout: default
title: "Heikin-Ashi, Role-Resolved – The Signal Lives at the Entry, Not the Exit"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20260615
date: 2026-06-14
tags: [heikin-ashi, backtesting, technical-analysis, quantitative-finance, ablation, walk-forward, wichtelm]
post_excerpt: A follow-up that asks the question the first study did not — does ANY Heikin-Ashi rule add value? Resolving HA by role (entry vs exit) over a 3-fold walk-forward, the HA colour-reversal entry beats a turnover-matched non-HA baseline out-of-sample (p ≈ 0.001), while HA at the exit destroys value.
description: "A role-resolved, walk-forward Heikin-Ashi discovery study: HA carries incremental value specifically at the entry (colour-reversal trigger), survives a turnover-matched control out-of-sample, but destroys value at the exit — which explains why a symmetric HA composite looked dead."
---

# Heikin-Ashi, Role-Resolved – The Signal Lives at the Entry, Not the Exit

## Abstract

A [previous study](/software-engineering/heikin-ashi-empirical-study/)
falsified **one** Heikin-Ashi (HA) strategy family — a strong-candle trend rule
that used HA on **both** the entry and the exit — and found its marginal
contribution over an EMA-cross was ≈ 0 on equities. A fair objection: that tests
one architecture, not the HA hypothesis space. This follow-up asks the
discovery question instead — *does any HA rule add incremental value?* — and
narrows it to the cleanest testable axis: the **role** HA plays. Using a 2×2
factorial (entry ∈ {HA colour-reversal, EMA-cross} × exit ∈ {HA, EMA}), validated
by a **3-fold expanding-window walk-forward** over 22 daily equities (2006–2026),
the result is **role-specific**: an HA bullish-reversal used **at the entry**
(EMA-cross exit held constant) beats its behaviour-matched non-HA baseline on
**44 of 66** out-of-sample stock-folds (sign-test *p* ≈ 0.005, median +0.10
Sharpe, net of cost), and the edge **survives a turnover-matched control** against
a faster EMA that trades just as often (**46/66, *p* ≈ 0.001**). The same HA
signal used **at the exit** significantly *destroys* value (14/66). This resolves
the earlier null: a symmetric HA composite looks dead because its harmful HA exit
cancels its useful HA entry. The effect is **modest and still loses to
buy-and-hold** — this is a better *rule*, not an *alpha* — but it is real,
directional, and reproducible. As before, this is exploratory research built on
the open-source [`wichtelm`](https://github.com/jbiscella/wichtelm-app)
backtester.

---

## 1. Why a second study

The first article's verdict — "HA adds no measurable edge over an equivalent
trend rule on equities" — was correct *about the thing it tested*: a strong-candle
rule that fires HA on the way in and HA on the way out, bundled with EMA filters
and stops. But "this one HA architecture fails" is not "HA is useless." HA is a
**family** of signals (colour reversals, run-length, body geometry, strong
candles) that can play several **roles** (entry trigger, exit trigger,
confirmation filter). A single failed composite cannot speak for that space.

So this study changes the question from *falsification* to *discovery*, and
imposes the discipline the first one earned the right to demand of it:

- A **pre-specified factorial**, not a grab-bag of strategies, so the result is a
  clean statement about *where* HA helps, not *whether some configuration*
  happened to win.
- **Walk-forward** parameter selection (out-of-sample by construction), not a
  single in-sample window.
- A **behaviour-matched** non-HA baseline, and then a **turnover-matched**
  control, because the first article's own §4.2.2 lesson was that an apparent
  edge often dissolves once you neutralise the obvious confound.

## 2. Design

### 2.1 The 2×2 role factorial

HA and the EMA-cross are made directly comparable by choosing the HA primitive
that is, like a cross, a **transition event** rather than a persistent state: the
**colour reversal** (`ha_bullish_reversal` / `ha_bearish_reversal`). A green-flip
is the HA analogue of price crossing above its EMA; both are one-shot,
similar-frequency triggers, which keeps turnover comparable from the outset. The
factorial swaps the signal independently at entry and exit:

| Family | Entry trigger | Exit trigger |
|---|---|---|
| `d-ema` (baseline) | EMA-cross up | EMA-cross down |
| **`d-haentry`** | **HA bullish reversal** | EMA-cross down |
| `d-haexit` | EMA-cross up | **HA bearish reversal** |
| `d-rev` | **HA bullish reversal** | **HA bearish reversal** |

All four are long-only on the **real OHLC** (HA is signal-only, never a synthetic
fill price — §3.2 of the first study), sized at 100% of equity, with the same
fixed `stop_loss`/`take_profit` structure and the gap-aware protective fills
introduced in `wichtelm` PR #64. The article-1 strong-candle family
(`strong-lo`, HA strong + EMA filter on both sides) is carried along as a
reference point. The entry family, in the `wichtelm` DSL:

```gherkin
Feature: Discovery — HA entry only (HA reversal in, EMA cross out)
  Primary timeframe: 1d
  Parameter streak default 1
  Parameter trend_period default 50
  Parameter stop_loss_pct default 8
  Parameter take_profit_pct default 25

  Scenario: Enter long on a bullish HA colour reversal
    Given no open position
    When ha_bullish_reversal(streak)
    Then long_entry
    And with stop_loss at entry_price * (1 - stop_loss_pct / 100)
    And with take_profit at entry_price * (1 + take_profit_pct / 100)
  Scenario: Exit long on a price-below-EMA cross
    Given a long position is open
    When price_crosses_below_ema(trend_period)
    Then long_exit
```

### 2.2 Walk-forward validation

The universe is 22 large-cap daily equities with continuous 2006–2026 history
(the same long-history set as the first study). Validation is a **3-fold
expanding-window walk-forward** with disjoint out-of-sample (OOS) windows:

| Fold | In-sample (parameters chosen here) | Out-of-sample (evaluated here) |
|---|---|---|
| F1 | 2006–2012 | 2012–2016 |
| F2 | 2006–2016 | 2016–2021 |
| F3 | 2006–2021 | 2021–2026 |

Within each fold, every family is swept over its parameter grid (streak ∈ {1,2,3},
EMA period ∈ {20,50,100}, stop/take ∈ {8%/25%, 15%/100%}) **on the in-sample
window only**, and the **cross-instrument-robust** combination — the one with the
best median in-sample Sharpe across the 22 names — is **locked**. That locked
configuration is then run once on the untouched OOS window. Parameters are never
chosen on data they are scored on.

### 2.3 Metrics and the incremental-value test

All metrics are recomputed from `wichtelm`'s per-bar equity curve (the
`--dump-equity` export), validated to reproduce the HTML report's Sharpe / return
/ drawdown to three decimals. Returns are taken **net of a 2 bp-per-side fee**
applied to the realised round-trip count, so a higher-turnover family is charged
for its turnover. The headline test is **paired and per-instrument**: for each
stock-fold, *(HA-family OOS net) − (matched-baseline OOS net)*, pooled across the
three folds into 66 paired observations, summarised by the median paired delta and
a sign test on the win count.

## 3. Results

### 3.1 The role factorial

![Heikin-Ashi value is role-specific: HA at the entry adds median +0.10 Sharpe vs a matched EMA baseline (44/66 OOS stock-folds), while HA at the exit subtracts; the edge holds against a turnover-matched fast EMA across all three folds](/assets/images/ha-entry-role-resolved.png)

Pooled across the three OOS folds, each HA family measured against the matched
`d-ema` baseline (paired per stock):

| HA role | OOS wins | sign-test *p* | median Δ net | median Δ Sharpe |
|---|--:|--:|--:|--:|
| **HA at entry only** (`d-haentry`) | **44/66** | **0.005** | **+14.5%** | **+0.10** |
| HA at entry & exit (`d-rev`) | 30/66 | 0.81 | −5.2% | −0.03 |
| HA strong, both sides (`strong-lo`, article 1) | 25/66 | 0.98 | −9.9% | −0.11 |
| HA at exit only (`d-haexit`) | 14/66 | 1.00 | −18.8% | −0.21 |

The asymmetry is the whole story. Moving the HA signal to the **entry** is the
only configuration that beats its non-HA twin; using it at the **exit** is
significantly *worse* than an EMA-cross exit. The two symmetric families
(`d-rev`, and article 1's `strong-lo`) sit in between — exactly what you would
expect if a useful entry and a harmful exit partially cancel. That cancellation is
the mechanical explanation for the first study's null result on equities: it only
ever tested HA used on both sides at once.

The per-fold family medians show the entry edge is not a single-window artefact:

| Fold (OOS) | `d-ema` | `d-haentry` | `d-haexit` | `d-rev` | `strong-lo` |
|---|--:|--:|--:|--:|--:|
| F1 2012–16 | +24% / 0.46 | **+41% / 0.69** | +10% / 0.35 | +30% / 0.62 | +17% / 0.32 |
| F2 2016–21 | +52% / 0.58 | **+68% / 0.65** | +26% / 0.44 | +34% / 0.49 | +43% / 0.53 |
| F3 2021–26 | +53% / 0.51 | **+55% / 0.52** | +6% / 0.17 | +21% / 0.31 | +13% / 0.24 |

*(median OOS net return / median OOS Sharpe across the 22 stocks)*

### 3.2 The turnover-matched control

The obvious objection — the one the first study taught us to pre-empt — is that
the HA reversal entry simply *trades more* (it fires roughly twice as often as the
slow EMA-cross), and more participation in a rising tape flatters return even net
of a small fee. So the baseline is rebuilt to **match the turnover**: a *fast*
EMA-cross (short period → more crosses), with its period chosen **on the
in-sample window** to bring its trade count as close as possible to
`d-haentry`'s, then locked and evaluated OOS.

With turnover genuinely neutralised — the fast EMA trades as often as, or more
than, the HA entry on every fold — the HA entry **still wins**:

| Fold | `d-haentry` median round-trips | fast-EMA median round-trips | HA entry wins |
|---|--:|--:|--:|
| F1 | 17 | 20 | 14/22 |
| F2 | 30 | 29 | 17/22 |
| F3 | 24 | 23 | 15/22 |
| **Pooled** | — | — | **46/66** |

Pooled sign-test *p* ≈ **0.001**, median Δ net **+17.3%**, median Δ Sharpe
**+0.11**. The edge is therefore **not** a turnover artefact: at equal trade
frequency, *when* the HA reversal chooses to enter carries information a same-cost
EMA-cross does not. With four pre-specified roles tested, even a Bonferroni
threshold (0.0125) leaves both the walk-forward (0.005) and the turnover-control
(0.001) results standing.

## 4. Interpretation

Heikin-Ashi smoothing makes a colour flip a slightly **earlier and cleaner**
trend-onset trigger than a raw price/EMA cross — the two-bar averaging suppresses
the single-bar noise that causes EMA crosses to fire a touch late or to whipsaw at
the turn. That is a genuine, if small, **entry** advantage. The same smoothing is
a **liability at the exit**: it makes the signal *lag*, so an HA-reversal exit
gives back more of a move before it admits the trend is over, and it churns in
choppy tape. A trader's folklore instinct — "HA keeps you in the trend" — is half
right: it is a good way *in* and a poor way *out*.

This also sharpens, rather than overturns, the first study's economics. The entry
edge is real but **modest** (≈ +0.1 Sharpe, ≈ +15% net over a multi-year OOS
window), and **every** family here still loses to buy-and-hold. HA at the entry
makes a mechanical trend rule **better**; it does not make it a market-beating
strategy. The contribution is a *cleaner signal*, not an *alpha*.

## 5. Limitations

- **Equities, daily, long-only.** The factorial has not yet been run on crypto,
  on intraday bars, or with a short side. The first study found HA's behaviour is
  asset-class dependent, so the entry edge should not be assumed to transfer.
- **Correlated names, nested in-sample windows.** The 66 "observations" are 22
  cross-correlated stocks × 3 expanding (hence overlapping) in-sample folds, so
  the effective sample is smaller than 66 and the sign-test *p*-values overstate
  significance. The defence is **consistency** — the entry family wins in all
  three disjoint OOS windows and survives the turnover control — not any single
  *p*-value.
- **Still in-sample at the universe level.** Walk-forward isolates parameter
  overfit within these 22 names; it does not prove the *family choice* (colour
  reversal at entry) would have been selected ex-ante on a different universe.
- **One HA feature, one matched baseline.** This tests the colour-reversal event
  against an EMA-cross event. Other HA families the first study listed as
  untested — body-size/ATR geometry, wick ratios, HA/raw divergence, multi-
  timeframe HA — require new primitives in the backtester and remain open.
- **Modest effect, no risk-scaling.** Fixed 100%-equity sizing; the comparison
  is rule-vs-rule, not a deployable portfolio.

## 6. Conclusion

Asked the discovery question it was designed for, Heikin-Ashi is **not** valueless
on equities — but its value is **role-specific and easily hidden**. As a
colour-reversal **entry** trigger it beats a behaviour- and turnover-matched
non-HA baseline out-of-sample across three independent folds (*p* ≈ 0.001); as an
**exit** trigger it is measurably worse than the same non-HA rule. A strategy that
uses HA symmetrically — like the family the first study falsified — nets the two
out and looks dead. The broader methodological point is the reusable one: a
**role-resolved ablation** recovered a real, directional signal that the earlier
**symmetric** test had averaged into nothing. The signal is modest and still
trails buy-and-hold, so the honest framing is a sharper hypothesis, not a
deployable edge: *HA earns its keep on the way in, not on the way out.*

---

### Reproducibility & disclaimer

All backtests use [`wichtelm-app`](https://github.com/jbiscella/wichtelm-app) at
commit `6ee5a1c` (the gap-aware protective-fill build), the same build pinned by
the [first study](/software-engineering/heikin-ashi-empirical-study/).
The four strategy families are plain-text `.strat` files (the entry family is
reproduced in full in §2.1); the walk-forward harness, parameter grids and the
metric recomputation are external scripts. Price data was snapshotted from a
commercial provider and split-adjusted; the licensed series is not redistributed,
and the figure shows only **derived paired deltas**, never prices. Returns are net
of a 2 bp-per-side fee applied post-hoc to each run's round-trip count.

*This article is for research and educational purposes only. It is not financial
advice. Past performance is not indicative of future results, and hypothetical
backtested results carry well-documented biases.*
