---
layout: default
title: "Heikin-Ashi, Resolved – It's the Bullish Flip, Not the Role"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20260615
date: 2026-06-14
tags: [heikin-ashi, backtesting, technical-analysis, quantitative-finance, ablation, walk-forward, wichtelm]
post_excerpt: A follow-up that asks the question the first study did not — does ANY Heikin-Ashi rule add value? Resolving HA by role and by direction over a walk-forward, the value turns out to live in the bullish colour reversal (the "green flip"), aligned with the equity drift — it beats a turnover-matched baseline whether used to enter longs or cover shorts (p ≈ 0.001), while the bearish flip destroys value, and none of it transfers to crypto.
description: "A role- and direction-resolved walk-forward Heikin-Ashi discovery study: the value lives in the bullish colour reversal (green flip), not in where it sits in the trade. It survives turnover-matched controls on both the long and short side, holds intraday, and aligns with the equity drift — but does not transfer to crypto."
---

# Heikin-Ashi, Resolved – It's the Bullish Flip, Not the Role

## Abstract

A [previous study](/software-engineering/heikin-ashi-empirical-study/) falsified
**one** Heikin-Ashi (HA) strategy family — a strong-candle trend rule that used HA
on **both** the entry and the exit — and found its marginal contribution over an
EMA-cross was ≈ 0 on equities. A fair objection: that tests one architecture, not
the HA hypothesis space. This follow-up runs the discovery experiment instead —
*does any HA rule add incremental value, and where?* — through a 2×2 factorial
(entry ∈ {HA colour-reversal, EMA-cross} × exit ∈ {HA, EMA}) under a
**walk-forward** with **turnover-matched** controls. The first cut looks
role-specific: an HA bullish-reversal **at the entry** beats its behaviour-matched
baseline on **44/66** out-of-sample stock-folds (*p* ≈ 0.005), while the same
signal at the **exit** destroys value. But the **short-side control flips the
pattern exactly** — and reveals the real mechanism: the value lives in the
**bullish colour reversal (the "green flip")**, *wherever* it is used. The green
flip beats a **turnover-matched** EMA whether it enters a long (**46/66, *p* ≈
0.001**) or covers a short (**58/66, *p* ≈ 0.001**); the bearish flip (the "red
flip") loses in both roles (12–14/66). The edge **holds intraday** (34/50, *p* ≈
0.008) but does **not transfer to crypto** (5/16). It aligns with the equity
upward drift, is **modest, and still loses to buy-and-hold** — a better *rule*,
not an *alpha*. A closing hunt (§5) confirms the ceiling is structural: adding
catalog filters, switching to forex, and even pooling 22 names into a diversified
portfolio never beats buy-and-hold — the portfolio only *matches* its Sharpe at
lower drawdown, which is an exposure effect. As before, everything runs on the
open-source [`wichtelm`](https://github.com/jbiscella/wichtelm-app) backtester.

---

## 1. Why a second study

The first article's verdict — "HA adds no measurable edge over an equivalent trend
rule on equities" — was correct *about the thing it tested*: a strong-candle rule
that fires HA on the way in and HA on the way out. But "this one HA architecture
fails" is not "HA is useless." HA is a **family** of signals that can play several
**roles**, and a single failed composite cannot speak for that space. So this study
changes the question from *falsification* to *discovery*, with the discipline the
first one earned the right to demand: a **pre-specified factorial** (not a
grab-bag), **walk-forward** selection (out-of-sample by construction), and a
**behaviour-matched** baseline hardened by a **turnover-matched** control — because
the first article's own lesson was that an apparent edge often dissolves once you
neutralise the obvious confound.

## 2. Design

### 2.1 The 2×2 role factorial

HA and the EMA-cross are made comparable by choosing the HA primitive that is, like
a cross, a **transition event** rather than a persistent state: the **colour
reversal** (`ha_bullish_reversal` / `ha_bearish_reversal`). A green-flip is the HA
analogue of price crossing above its EMA. The factorial swaps the signal
independently at entry and exit:

| Family | Entry trigger | Exit trigger |
|---|---|---|
| `d-ema` (baseline) | EMA-cross up | EMA-cross down |
| **`d-haentry`** | **HA bullish reversal** | EMA-cross down |
| `d-haexit` | EMA-cross up | **HA bearish reversal** |
| `d-rev` | **HA bullish reversal** | **HA bearish reversal** |

All are long-only on the **real OHLC** (HA is signal-only, never a synthetic fill
price), 100%-of-equity sized, with the same fixed `stop_loss`/`take_profit` and the
gap-aware protective fills from `wichtelm` PR #64. The article-1 strong-candle
family (`strong-lo`) is carried along as a reference. The entry family, in DSL:

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

### 2.2 Walk-forward, baseline, and the incremental-value test

The core universe is 22 large-cap daily equities, 2006–2026, validated by a
**3-fold expanding-window walk-forward** with disjoint out-of-sample (OOS) windows
(IS 2006→2012/2016/2021; OOS 2012–16 / 2016–21 / 2021–26). Within each fold a
family is swept (streak ∈ {1,2,3}, EMA ∈ {20,50,100}, stop/take ∈ {8/25, 15/100})
**on the in-sample window only**; the **cross-instrument-robust** combination — best
median in-sample Sharpe across the 22 names — is **locked** and run once on the
untouched OOS window. Metrics are recomputed from `wichtelm`'s per-bar equity curve
(validated to match the HTML report to 3 dp), **net of a 2 bp-per-side fee** on the
realised round-trip count. The headline test is **paired per instrument**:
*(HA-family OOS net) − (matched-baseline OOS net)*, pooled across folds into 66
observations, summarised by the median paired delta and a sign test.

Three hardening steps follow: a **turnover-matched control** (the baseline rebuilt
as a *fast* EMA that trades as often as the HA family, period chosen on the IS
window), and two **transfer tests** — the same factorial on **8 crypto** daily
(2021–2026, 2 folds) and on **25 intraday** 1-hour equities (2021–2025, 2 folds) —
and finally the **short-side** mirror of the whole factorial.

## 3. Results

### 3.1 The first cut: HA helps at the entry, hurts at the exit

Pooled across the three OOS folds, each HA family vs the matched `d-ema` baseline:

| HA role | OOS wins | sign-test *p* | median Δ net | median Δ Sharpe |
|---|--:|--:|--:|--:|
| **HA at entry only** (`d-haentry`) | **44/66** | **0.005** | **+14.5%** | **+0.10** |
| HA at entry & exit (`d-rev`) | 30/66 | 0.81 | −5.2% | −0.03 |
| HA strong, both sides (`strong-lo`, article 1) | 25/66 | 0.98 | −9.9% | −0.11 |
| HA at exit only (`d-haexit`) | 14/66 | 1.00 | −18.8% | −0.21 |

Read naively, this is "HA belongs at the entry." Moving the HA signal to the entry
is the only configuration that beats its non-HA twin; at the exit it is
significantly *worse*. The two symmetric families sit in between — consistent with a
useful entry and a harmful exit cancelling, which is the mechanical explanation for
the first study's null (it only ever tested HA on both sides at once).

### 3.2 It is not a turnover artefact

The HA reversal entry fires about twice as often as the slow EMA-cross, so the
obvious objection is that it simply *participates more* in a rising tape. Rebuilding
the baseline as a **fast EMA** whose period is chosen on the in-sample window to
**match `d-haentry`'s turnover**, then evaluating OOS, the HA entry **still wins**:
**46/66** (*p* ≈ 0.001), median Δ net +17.3%, Δ Sharpe +0.11, with the fast EMA
trading as often as — or more than — the HA entry on every fold. At equal trade
frequency, *when* the HA reversal enters carries information a same-cost EMA-cross
does not.

### 3.3 Does it transfer? Intraday yes, crypto no

The same factorial on **intraday (1-hour) equities** reproduces the entry result —
`d-haentry` **34/50** (*p* ≈ 0.008), consistent in both folds (18/25, 16/25) — so
it is an equity property, not a daily-bar accident. On **crypto**, it **collapses**:
`d-haentry` is the *worst* HA role (**5/16**, *p* ≈ 0.96), and crypto returns are
dominated by regime (every family makes triple-digit returns in the 2023–24 bull
fold and loses in the 2024–26 fold) rather than by where HA sits. The entry edge is
**asset-class-bound to equities** — echoing the first study's finding that HA's
behaviour is asset-class dependent.

### 3.4 The short-side control reveals the real mechanism

Running the **mirror factorial on the short side** (short on a bearish reversal /
EMA-cross-down, cover on a bullish reversal / EMA-cross-up) flips the pattern
**exactly**:

![The 2x2 mechanism: HA's value tracks the bullish colour flip, not the entry/exit role — it wins at the long entry and the short cover (both use the green flip) and loses at the long exit and short entry (both use the red flip); the green flip travels across equity timeframes but fails on crypto](/assets/images/ha-green-flip-mechanism.png)

| | HA flip used | OOS wins vs matched baseline |
|---|---|--:|
| **Long entry** | 🟢 bullish reversal | **44/66** ✓ |
| Long exit | 🔴 bearish reversal | 14/66 ✗ |
| Short entry | 🔴 bearish reversal | 12/66 ✗ |
| **Short cover** | 🟢 bullish reversal | **53/66** ✓ |

The "entry vs exit" framing was the surface. Line up *which colour flip each cell
uses* and the noise disappears: the **bullish reversal (green flip) carries the
signal wherever it is used** — entering a long *or* covering a short — and the
**bearish reversal (red flip) destroys value wherever it is used**. And the short
cover survives the same hardening: against a **turnover-matched** fast-EMA cover it
wins **58/66** (*p* ≈ 0.001, median Δ net +17.5%, Δ Sharpe +0.19). (On the short
side every family loses money in absolute terms — the equity risk premium is a
headwind both legs share — so only the *paired* delta is meaningful.)

## 4. Interpretation

Heikin-Ashi's two-bar averaging makes a **bullish** colour flip an earlier, cleaner
confirmation that a pullback has turned back up — and on equities that is a
high-base-rate event, because the asset class drifts upward. So the green flip is a
good *"the dominant uptrend has resumed"* trigger, and it pays whether you use it to
open a long or to close a short. The **bearish** flip fights that drift: it flags
tops that mostly resolve back up (bear-market rallies, V-recoveries), so acting on
red flips — exiting longs early, or shorting — is systematically punished. The
mechanism is **directional, not positional**, which is why the entry/exit reading
inverts on the short side and the crypto test (no comparable structural drift)
shows no green-flip edge at all.

This sharpens rather than overturns the first study's economics. The edge is real,
survives turnover-matching on both sides, and holds intraday — but it is **modest**
(≈ +0.1–0.2 Sharpe) and **every family here still loses to buy-and-hold**. The green
flip makes a mechanical trend rule **better**; it does not make it market-beating.
The contribution is a *cleaner signal aligned with the drift*, not an *alpha*.

## 5. Epilogue: can the green flip beat buy-and-hold?

A "better entry than an EMA-cross" is not the same as "beats holding the asset."
Three attempts to compose the green flip into a buy-and-hold-beating strategy — all
walk-forward, all net of cost — close the loop.

**Adding catalog filters makes it worse.** Bolting the obvious confirmations onto
the green-flip entry — an EMA trend filter, a weekly higher-timeframe filter, an RSI
ceiling, a MACD-momentum gate, a trailing stop — uniformly *reduces* both return and
Sharpe. The plain, unfiltered green-flip entry is the best of the lot, and it still
trails equal-name buy-and-hold:

| Variant (OOS, 3-fold, net) | median net | median Sharpe | beats B&H Sharpe |
|---|--:|--:|--:|
| **Green flip, no filter** | **+55%** | **0.59** | 19/66 |
| + MACD momentum | +25% | 0.42 | 14/66 |
| + EMA trend filter | +26% | 0.42 | 12/66 |
| + RSI ceiling | +26% | 0.41 | 11/66 |
| + trailing stop | +26% | 0.41 | 11/66 |
| + weekly filter | +17% | 0.32 | 6/66 |
| buy & hold | +92% | 0.77 | — |

Every filter makes the rule more selective → more time in cash → more of the drift
given up. The plain green flip beats B&H's Sharpe on only 19 of 66 stock-folds.

**Forex doesn't rescue it.** The natural home for a timing rule is a market with no
upward drift, where sitting in cash costs nothing — so the factorial was re-run
long/short on 12 FX pairs. It fails there too: no variant beats a (flat-ish)
buy-and-hold, and HA specifically *hurts* — a plain EMA-cross long/short (Sharpe
0.17) beats both the HA-entry (0.09) and HA-reversal (−0.09) variants. The green
flip is an **equity-drift phenomenon, not a general timing signal**; strip the drift
and its edge evaporates. (Caveat: these FX OOS windows happened to trend, which a
long benchmark captured, and 12 USD-sharing pairs over two folds is a weak probe.)

**Diversification gets tantalisingly close — and still doesn't beat it.** The
strongest version pools all 22 green-flip sleeves into one equal-weight,
daily-rebalanced portfolio (each name invested only while its rule is long, cash
otherwise). Diversification lifts the strategy's Sharpe dramatically — from ≈ 0.6
per name to **1.27** — but it lifts buy-and-hold just as much, to **1.32**:

| Portfolio (OOS, pooled) | total return | Sharpe | max drawdown | avg exposure |
|---|--:|--:|--:|--:|
| Green-flip portfolio | +417% | 1.27 | **−14%** | 54% |
| Equal-weight buy & hold | +1303% | **1.32** | −33% | 100% |

The green-flip portfolio **nearly matches** buy-and-hold's risk-adjusted quality
(Sharpe 1.27 vs 1.32) while invested only ~54% of the time and with **less than half
the worst drawdown**, and it even wins one of the three folds outright (2016–21:
Sharpe 1.63 vs 1.44 at a third of the drawdown). But it does not *beat* B&H — and,
exactly as the first study's §4.2.2 showed, the lower drawdown is the **exposure
effect**, not skill: a passive buy-and-hold de-risked to the same ~54% exposure
carries the *same* Sharpe (scaling is Sharpe-invariant) and a similarly shallow
drawdown. The diversified green-flip portfolio is, at best, an expensive way to
reproduce "hold a bit less of the index."

The verdict is consistent and structural: a rule that steps out of a drifting asset
forfeits compounding that a better entry signal cannot recover. **The green flip is a
genuinely better entry — and still not an alpha.**

## 6. Limitations

- **Equities only for the edge; crypto explicitly excluded.** The green-flip edge
  is demonstrated on equities (daily + intraday, long + short). The crypto test is
  a small, highly-correlated probe (8 coins × 2 folds) and is reported as a
  *negative*, not a measurement.
- **Correlated names, nested in-sample windows.** The 66 daily "observations" are
  22 cross-correlated stocks × 3 expanding (overlapping) IS folds, so the effective
  sample is smaller than 66 and sign-test *p*-values overstate significance. The
  defence is **consistency** — the green flip wins in every disjoint OOS fold, on
  both timeframes, on both trade directions, and survives turnover-matching — not
  any single *p*-value.
- **Drift-conditional, not a market timer.** The interpretation ties the edge to the
  equity upward drift; it has not been tested in a structurally downward-drifting
  asset, and the short side loses money in absolute terms.
- **One HA feature.** This isolates the colour reversal against an EMA-cross. Other
  HA families the first study listed as untested — body/ATR geometry, wick ratios,
  HA/raw divergence, multi-timeframe HA — need new backtester primitives and remain
  open.
- **Modest effect; not buy-and-hold-beating.** Fixed 100%-equity sizing. The §5
  epilogue tests the portfolio case directly: even diversified, the green flip only
  *matches* B&H's Sharpe (at lower drawdown, which is an exposure effect), and the
  filter and forex hunts fail outright. A better rule, not an alpha.

## 7. Conclusion

Asked the discovery question it was designed for, Heikin-Ashi is **not** valueless on
equities — but its value is narrower and stranger than "it works." It lives almost
entirely in the **bullish colour reversal**: a green flip is a cleaner trend-resumption
trigger than an EMA-cross at the same trade frequency, and it pays *wherever* it is
used — opening longs (44→46/66) and covering shorts (53→58/66) alike — because it
runs with the equity drift. The **bearish** flip, fighting that drift, loses
everywhere. A strategy that uses HA symmetrically nets the two out and looks dead —
which is exactly the family the first study falsified. The reusable methodological
point: a **role- and direction-resolved** ablation recovered a real, mechanistic
signal that the earlier **symmetric** test had averaged into nothing. The signal is
modest, equity-specific, and still trails buy-and-hold, so the honest framing is a
sharper hypothesis, not a deployable edge: *Heikin-Ashi's information is in the green
flip, aligned with the drift — not in the candles' reputation for keeping you in a
trade.* And as the §5 hunt shows, even composed at its best — diversified across 22
names — it only draws level with buy-and-hold, never ahead: the ceiling is
structural, not a matter of finding richer HA features.

---

### Reproducibility & disclaimer

All backtests use [`wichtelm-app`](https://github.com/jbiscella/wichtelm-app) at
commit `6ee5a1c` (the gap-aware protective-fill build), the same build pinned by the
[first study](/software-engineering/heikin-ashi-empirical-study/). The strategy
families are plain-text `.strat` files (the entry family is reproduced in full in
§2.1); the walk-forward harness, turnover-matched controls, transfer tests and
metric recomputation are external scripts. Price data was snapshotted from a
commercial provider and split-adjusted; the licensed series is not redistributed,
and the figure shows only **derived paired win-rates**, never prices. Returns are
net of a 2 bp-per-side fee applied post-hoc to each run's round-trip count.

*This article is for research and educational purposes only. It is not financial
advice. Past performance is not indicative of future results, and hypothetical
backtested results carry well-documented biases.*
