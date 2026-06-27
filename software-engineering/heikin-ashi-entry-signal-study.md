---
layout: default
title: "Heikin-Ashi: Anatomy of an Edge That Dissolves Under Controls"
parent: Software Engineering Notes
grand_parent: Blog
nav_order: -20260615
date: 2026-06-15
tags: [heikin-ashi, backtesting, technical-analysis, quantitative-finance, ablation, walk-forward, overfitting, wichtelm]
post_excerpt: A discovery follow-up that surfaces a clean, mechanistic-looking Heikin-Ashi signal — the bullish colour reversal — and then watches it dissolve under the right controls: a raw (non-HA) reversal does as well or better, the edge is concentrated in survivors and vanishes on delisted names, and it never beats buy-and-hold. A case study in how a carefully-validated signal can still evaporate.
description: "A role- and direction-resolved walk-forward Heikin-Ashi study where an apparent bullish-reversal edge survives turnover-matching, intraday and short-side controls — then dissolves under a raw-signal control (it isn't HA) and a survivorship correction (it vanishes on delisted names), and never beats buy-and-hold. A methodological case study in deflating an apparent edge."
---

# Heikin-Ashi: Anatomy of an Edge That Dissolves Under Controls

## Abstract

A [previous study](/software-engineering/heikin-ashi-empirical-study/) falsified one
symmetric Heikin-Ashi (HA) strategy family and found its drawdown reduction was an
exposure effect. This follow-up runs the discovery question — *does any HA rule add
value?* — as a role- and direction-resolved walk-forward factorial, and it surfaces
something that **looks** like a clean, mechanistic edge: the **bullish colour
reversal** ("green flip") beats an EMA-cross entry on equities (44/66 OOS
stock-folds, *p* ≈ 0.005), the pattern tracks the *direction* of the flip rather
than the entry/exit role (confirmed by a short-side mirror), and it survives a
**turnover-matched** control and an **intraday** replication. A tidy result — until
the right controls are applied. Then it **dissolves**: (1) a **raw, non-HA fast
reversal** does as well or better (it beats the EMA baseline 49/66 vs the green
flip's 44/66, and beats the green flip head-to-head) — so it **isn't Heikin-Ashi**;
(2) on a **point-in-time S&P universe including delisted names**, the edge is
concentrated in survivors (74/107) and **vanishes on delisted names** (19/38, *p* ≈
0.56) — so it is **partly survivorship**; and (3) it **never beats buy-and-hold**.
What survives is thin and generic: a fast-reversal entry beats a slow-EMA entry on
surviving equities, modestly (≈ +0.1 Sharpe), and Heikin-Ashi is incidental to it.
The lasting value is methodological — a worked example of how a carefully-validated,
mechanistically-plausible signal evaporates under a raw-signal control and a
survivorship correction. Everything runs on the open-source
[`wichtelm`](https://github.com/jbiscella/wichtelm-app) backtester.

---

## 1. Why a second study

The first article's verdict — "HA adds no measurable edge over an equivalent trend
rule on equities" — was correct *about the thing it tested*: a strong-candle rule
that fires HA on the way in and the way out. But "this one architecture fails" is not
"HA is useless." So this study changes the question from *falsification* to
*discovery*, with the discipline the first one earned the right to demand: a
**pre-specified factorial**, **walk-forward** selection, and a battery of controls —
turnover-matching, a short-side mirror, transfer tests, and — decisively — a
**raw-signal control** and a **survivorship correction**. The headline is not the
signal it finds; it is what the controls do to it.

## 2. Design

**The factorial.** HA and the EMA-cross are made comparable by using the HA
primitive that is, like a cross, a transition *event*: the **colour reversal**
(`ha_bullish_reversal` / `ha_bearish_reversal`). The factorial swaps the signal
independently at entry and exit:

| Family | Entry trigger | Exit trigger |
|---|---|---|
| `d-ema` (baseline) | EMA-cross up | EMA-cross down |
| `d-haentry` | HA bullish reversal | EMA-cross down |
| `d-haexit` | EMA-cross up | HA bearish reversal |
| `d-rev` | HA bullish reversal | HA bearish reversal |

All long-only on the **real OHLC** (HA is signal-only), 100%-equity sized, same
fixed stop/take and the gap-aware protective fills from `wichtelm` PR #64.

**Walk-forward.** The core universe is 22 large-cap daily equities, 2006–2026, under
a 3-fold expanding-window walk-forward (IS 2006→2012/2016/2021; disjoint OOS
2012–16 / 2016–21 / 2021–26). Within each fold a family is swept on the in-sample
window only; the cross-instrument-robust combination (best median IS Sharpe across
the 22 names) is locked and run once on the untouched OOS window. Metrics are
recomputed from `wichtelm`'s per-bar equity curve (validated to match the HTML
report to 3 dp), **net of a 2 bp-per-side fee**. The headline test is paired per
instrument and pooled across folds into 66 observations.

**The controls** (§3–§4): a turnover-matched fast-EMA baseline; the short-side
mirror; an intraday (1-hour) replication and a crypto transfer; a **raw fast-reversal
control** (a non-HA short-SMA cross); and a **frozen-hypothesis re-test on a
point-in-time S&P universe with delisted names**.

## 3. An apparent edge, carefully validated

Pooled across the three OOS folds, each HA family vs the matched `d-ema` baseline:

| HA role | OOS wins | sign-test *p* | median Δ Sharpe |
|---|--:|--:|--:|
| **HA at entry only** (`d-haentry`) | **44/66** | **0.005** | **+0.10** |
| HA at entry & exit (`d-rev`) | 30/66 | 0.81 | −0.03 |
| HA strong, both sides (`strong-lo`, article 1) | 25/66 | 0.98 | −0.11 |
| HA at exit only (`d-haexit`) | 14/66 | 1.00 | −0.21 |

![The 2x2 mechanism: an apparent edge tracking the bullish colour flip — winning at the long entry and short cover (green flip), losing at the long exit and short entry (red flip); it holds across equity timeframes but not crypto](/assets/images/ha-green-flip-mechanism.png)

Used at the **entry**, the bullish reversal beats its non-HA twin; at the exit it is
worse. The **short-side mirror flips the pattern exactly** — the value tracks the
*colour of the flip*, not the entry/exit role: the **bullish reversal** wins wherever
used (long entry 44/66, short cover 53/66) and the **bearish reversal** loses wherever
used (long exit 14/66, short entry 12/66). It is **not a turnover artefact** — against
a fast EMA matched to its trade frequency, the bullish flip still wins (long entry
46/66, short cover 58/66, *p* ≈ 0.001) — and it **holds intraday** (34/50, *p* ≈
0.008), though it does not transfer to crypto (5/16).

At this point the result looks robust: pre-specified, walk-forward, turnover-matched,
direction-confirmed, timeframe-replicated. The natural conclusion — the one an earlier
draft of this article actually drew — is "Heikin-Ashi's information lives in the green
flip." That conclusion is **wrong**, and two controls show why.

## 4. …and how it dissolves

### 4.1 It isn't Heikin-Ashi

Heikin-Ashi is a deterministic transform of OHLC — a recursive two-bar smoother. The
question a believer must answer is whether *that specific smoothing* carries
information, or whether any fast reversal detector would do. So the green-flip entry
is pitted against a **raw, non-HA fast reversal**: price crossing above a short SMA
(`price_crosses_above_sma(2)` is just "today's close ticks up after ticking down" —
the rawest possible reversal), same EMA-cross exit and structure, same walk-forward.

| Paired comparison (OOS, 66 folds) | wins | sign-test *p* | median Δ net |
|---|--:|--:|--:|
| Raw fast reversal vs EMA baseline | **49/66** | **0.000** | +18.6% |
| HA green flip vs EMA baseline | 44/66 | 0.005 | +14.5% |
| **HA green flip vs raw reversal** | **27/66** | **0.95** | −5.5% |

![Two controls dissolve the edge: a raw non-HA reversal beats the EMA baseline more than the HA green flip does (and beats the green flip head-to-head); and the green-flip edge is concentrated in survivors, dropping to a coin-flip on delisted names](/assets/images/ha-edge-dissolves.png)

The raw reversal beats the EMA baseline *more* than the HA flip does (49/66 vs 44/66),
and it beats the HA flip **head-to-head** (the green flip wins only 27 of 66). The
"green flip" was never a Heikin-Ashi phenomenon — it is a generic **fast-reversal
entry-timing** effect, and HA's smoothing is an incidental (slightly inferior) way to
compute it. The mechanistic story of §3 survives only in stripped-down form: a *fast
bullish reversal* is an earlier entry than a slow EMA-cross, in a drifting market —
nothing specific to candles.

### 4.2 It is concentrated in survivors

The 22-name core is currently-listed survivors, whose pullbacks, by selection,
resumed — exactly the bias that would manufacture a "buy the dip's turn" edge. To
test it, the green-flip rule is **frozen** (a single central config, no parameter
search, no peeking) and re-run on a **point-in-time S&P universe including delisted
and removed names**, each over its full available window:

| Subset (frozen green flip vs EMA) | wins | sign-test *p* | median Δ net |
|---|--:|--:|--:|
| All names (145) | 93/145 | 0.000 | +53% |
| Survivors (107) | 74/107 | 0.000 | +96% |
| **Delisted / removed (38)** | **19/38** | **0.56** | **+0.3%** |

The frozen rule does beat the EMA baseline on the broad universe — so the
fast-reversal-entry effect is real and not pure parameter overfit — but it is
**concentrated in survivors and collapses to a coin-flip on the delisted names**
(19/38, median +0.3%). The "pullbacks resume" mechanism works on companies that
survived; on those that died, it does not. A meaningful slice of the apparent edge is
survivorship.

### 4.3 And it never beats buy-and-hold

Even granting the thin surviving effect, it does not make money relative to doing
nothing. Three composition attempts (all walk-forward, net of cost):

- **Filters make it worse.** Bolting an EMA / weekly / RSI / MACD filter or a trailing
  stop onto the entry uniformly *cuts* return and Sharpe; the plain entry is best at
  +55% / Sharpe 0.59 and still trails equal-name buy-and-hold (+92% / 0.77), beating
  B&H's Sharpe on only 19/66 stock-folds. Each filter means more time in cash → more of
  the drift surrendered.
- **Forex doesn't rescue it, and HA hurts there.** Re-run long/short on 12 FX pairs, no
  variant beats a flat-ish buy-and-hold, and a plain EMA-cross (Sharpe 0.17) beats both
  HA variants (0.09 and −0.09). The fast-reversal edge is a drift phenomenon; strip the
  drift and it evaporates.
- **Diversification only ties it.** Pooling all 22 sleeves into an equal-weight,
  daily-rebalanced portfolio lifts the strategy's Sharpe to 1.27 — but buy-and-hold's to
  1.32. The portfolio nearly matches B&H's risk-adjusted quality at ~54% exposure and
  half the drawdown (−14% vs −33%), but does not beat it; the lower drawdown is the same
  **exposure effect** the first study identified (§4.2.2), since a passive B&H de-risked
  to 54% carries the same Sharpe.

## 5. What survives, and what it means

Strip away the over-attribution and the survivorship, and a single thin claim is left
standing: **on surviving equities, a fast bullish-reversal entry beats a slow EMA-cross
entry, modestly (≈ +0.1 Sharpe), in a drifting market.** It is real (frozen-hypothesis,
out-of-sample), but it is **not Heikin-Ashi** (a raw reversal does as well or better),
**partly survivorship** (gone on delisted names), drift-conditional (gone on crypto and
FX), and **not alpha** (never beats buy-and-hold). The green/red asymmetry of §3 is a
true *description* of the data, but its mechanism is reversal-timing-versus-drift, with
HA incidental.

The durable contribution is methodological. The §3 result had every surface marker of
rigour — pre-specified factorial, walk-forward, turnover-matched, direction-confirmed
by a genuine out-of-sample prediction (the short-side mirror), timeframe-replicated —
and it was still **misattributed and survivorship-inflated**. Two controls that retail
backtests almost never run caught it: a **raw-signal control** (does a non-HA version
do as well?) and a **survivorship correction** (does it hold on names that delisted?).
Either alone would have deflated the headline; together they reduce a "mechanistic HA
discovery" to a generic, modest, non-investable timing tilt.

## 6. Limitations

- **The surviving claim is thin and equity/drift-specific.** It is demonstrated on
  surviving equities in an up-drifting era; it does not generalise to crypto or FX, and
  it is not an alpha.
- **Correlated names, nested folds.** The 66 daily observations are 22 cross-correlated
  stocks × 3 overlapping IS folds, so sign-test *p*-values overstate significance; the
  real evidence is consistency across disjoint OOS windows, not any single *p*. (This
  cuts both ways — it tempers the apparent edge of §3 as much as the controls of §4.)
- **One raw control, one universe.** §4.1 uses a short-SMA reversal as the non-HA
  control; other smoothers (EMA-slope, smoothed typical price) are untested but would, if
  anything, further confirm HA is not special. §4.2's point-in-time set still carries the
  first study's data-quality caveats.
- **Frozen-config sensitivity.** §4.2 freezes one central parameterisation to avoid
  peeking; a different freeze could shift magnitudes, though the survivor/delisted *split*
  is the robust point.
- **No risk-scaling, no deflated Sharpe.** Fixed 100%-equity sizing; rule-vs-rule.

## 7. Conclusion

Asked whether any Heikin-Ashi rule adds value, this study found a signal that looked
like a textbook positive result — a direction-resolved, walk-forward, turnover-matched,
intraday-replicated bullish-reversal edge — and then dismantled it with its own
controls. It **isn't Heikin-Ashi** (a raw fast reversal does as well or better); it is
**partly survivorship** (a coin-flip on delisted names); and it **never beats
buy-and-hold**. What remains is a thin, generic, drift- and survivor-conditional
fast-reversal entry tilt to which Heikin-Ashi contributes nothing distinctive. The
honest headline is therefore not a discovery but a **method**: an apparent edge — even
a carefully-validated, mechanistically-plausible one — should be assumed to be a
mis-attribution or a survivorship artefact until a raw-signal control and a
point-in-time universe say otherwise. Here, they didn't.

---

### Reproducibility & disclaimer

All backtests use [`wichtelm-app`](https://github.com/jbiscella/wichtelm-app) at commit
`6ee5a1c` (the gap-aware protective-fill build), the same build pinned by the
[first study](/software-engineering/heikin-ashi-empirical-study/). Strategy families are
plain-text `.strat` files; the walk-forward harness, the raw-reversal and turnover
controls, the transfer tests and the frozen point-in-time re-test are external scripts.
Price data was snapshotted from a commercial provider and split-adjusted; the licensed
series is not redistributed, and the figures show only **derived win-rates**, never
prices. The point-in-time S&P membership (incl. delisted names) is the public
[`fja05680/sp500`](https://github.com/fja05680/sp500) dataset. Returns are net of a 2
bp-per-side fee applied post-hoc to each run's round-trip count.

*This article is for research and educational purposes only. It is not financial advice.
Past performance is not indicative of future results, and hypothetical backtested
results carry well-documented biases.*
