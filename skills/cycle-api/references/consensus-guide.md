# Cycle Consensus — Practical Application Guide

The Cycle Consensus Score is a **turning point alerting system**. It synthesizes multiple detected market cycles into a single directional reading and answers one question: **Are the cyclical conditions present for a trend change right now?**

It does not predict where price will be in N days. It tells you whether the dominant cycles are converging toward a turning point — and in which direction.

---

## The endpoint

**POST** `/api/CycleConsensus/calculate` — the consensus score for your own close prices.

The body is an **object**, not a bare array:

```json
{
  "datapoints": [101.2, 101.9, 102.4, ...],
  "bartelsLimit": 10,
  "minCycleLength": 15,
  "maxCycleLength": 400,
  "savgolSmoothing": false,
  "includeCrsi": true
}
```

All fields except `datapoints` are optional (the values shown are the defaults). At least 100 closes,
oldest first. With a stored dataset, leave `datapoints` out and add `?datasetid=NAME` (plus `maxbars`,
`from`, `to` for the window); settings like `includeCrsi` can still go in the body.

Pipeline inside: CycleScanner (HP filter detrending) → stability scoring → dominant peak detection →
consensus → CRSI.

---

## Response Properties Reference

### Score Properties

| Property | Type | Range | Description |
|----------|------|-------|-------------|
| `combinedScore` | double | -100 to +100 | Final consensus score. Positive = bullish, negative = bearish. Cycles contribute ±80 max, CRSI ±20 max. |
| `bullishConsensus` | double | 0–100 | Weighted strength of cycles voting bullish (rising + bottoming). |
| `bearishConsensus` | double | 0–100 | Weighted strength of cycles voting bearish (falling + topping). |
| `bullishCycleCount` | int | | Cycles in a bullish phase (rising + bottoming). |
| `bearishCycleCount` | int | | Cycles in a bearish phase (falling + topping). |
| `breadthFactor` | double | 0–1 | Penalises a consensus carried by only a few cycles. |
| `combinedScoreReasoning` | string | | Step-by-step text of how the score was derived. |

### The four phase arrays

| Property | Cycles that are … | Imminence |
|----------|-------------------|-----------|
| `toppingCycles` | at or near their peak, turning down | **At the turn**: the reversal is happening now |
| `bottomingCycles` | at or near their trough, turning up | **At the turn**: the reversal is happening now |
| `fallingCycles` | declining from peak toward trough | **Approaching**: bearish pressure building |
| `risingCycles` | ascending from trough toward peak | **Approaching**: bullish pressure building |

The number of entries in each array is the count per phase. Each entry (`CycleContributionDto`):

| Field | Meaning |
|---|---|
| `cycleLength` | Cycle length in bars |
| `phaseScore` | −2 topping, −1 falling, +1 rising, +2 bottoming (not the scanner's −100…+100 phase score) |
| `strength` | Bartels-based strength, 0–100 |
| `stabilityScore` | 0–1, consistency of amplitude and phase |
| `rank` | Dominant peak rank (1 = strongest), 0 = unranked |
| `cycleQuality` | Strength × stability × rank bonus; drives the weighting |
| `contribution` | What this cycle adds to the bullish or bearish consensus |
| `reason` | Text: how the contribution was calculated |
| `amplitude`, `phase`, `avgPhase`, `minBarNum`, `minBarNumCurrent` | The cycle's wave and phase data from the scanner |

To see *which* cycle lengths vote on each side, collect `cycleLength` from `bottomingCycles` +
`risingCycles` (bullish) and `toppingCycles` + `fallingCycles` (bearish). A bullish vote carried by
short cycles is a different signal from one carried by long, structural cycles.

### CRSI Properties

| Property | Type | Description |
|----------|------|-------------|
| `crsiScore` | int | Raw CRSI score (−3 to +3). Drives the signal state and the score contribution. |
| `crsiSignal` | string | Named signal state, see below. |
| `crsiLength` | int | CRSI period (half the dominant cycle length, clamped 5–50). |
| `crsiSourceCycleLength` | int | The cycle length the CRSI period was derived from. |
| `hasBullishDivergence`, `hasBearishDivergence` | bool | Price/CRSI divergence near the last bar (pilot: label only, no effect on the score). |

### CRSI Signal States

Signal states are named from the perspective of the **exhausting trend**. The progression Exhaustion → Fatigue → Exit describes the lifecycle of momentum breakdown.

| crsiScore | Signal State | Contribution | Meaning |
|-----------|-------------|-------------|---------|
| +1 | `BullExhaustion` | +6.7 | Overbought, momentum still rising — bull move approaching exhaustion |
| -1 | `BearExhaustion` | -6.7 | Oversold, momentum still falling — bear move approaching exhaustion |
| +2 | `BullFatigue` | +13.3 | Overbought, turning down — bull momentum fatiguing |
| -2 | `BearFatigue` | -13.3 | Oversold, turning up — bear momentum fatiguing |
| +3 | `BullExit` | +20.0 | Crossed below upper band — bull move exiting overbought |
| -3 | `BearExit` | -20.0 | Crossed above lower band — bear move exiting oversold |
| 0 | `Neutral` | 0 | Between bands — no overbought/oversold condition |

**Divergence-confirmed variants** (price-CRSI divergence alongside Fatigue or Exit):
`BullFatigueReversalConfirmed`, `BearFatigueReversalConfirmed`, `BullExitReversalConfirmed`,
`BearExitReversalConfirmed`. Divergence affects the label only; the numeric contribution is unchanged.

> **See also:** `crsi-signals.md` explains how the API derives `crsiScore` and `crsiSignal` from the CRSI bands.

### Score Composition Formula

```
CycleScore  = Direction × Confidence × 100, capped at ±80
CrsiContrib = (crsiScore / 3) × 20
FinalScore  = CycleScore + CrsiContrib, clamped to ±100
```

- Cycles alone max out at **±80**; full conviction needs CRSI agreement.
- CRSI adds at most **±20**. It can only change the sign when the cycle score itself is within ±20.
- `combinedScoreReasoning` shows each step.

### PRO features inside the consensus

As of September 2026 the consensus runs stability scoring and dominant peak detection for every
caller, so `stabilityScore` and `rank` in the phase arrays can be filled even without the PRO level
(unlike CycleScanner). This may change; if they come back as 0, rank by `strength` and `contribution`.

### App UI parity (Cycle Scanner gauge)

The Cycle Consensus gauge in the Cycle Scanner app runs the **same pipeline with CRSI integration
enabled** (`includeCrsi=true`). The app has no setting to switch the CRSI term off; its `crsi=true`
URL parameter only toggles the CRSI chart panel. The app falls back to the pure cycle consensus only
when CRSI cannot be computed: fewer than about 50 bars, or a CRSI length above data length / 3
(NaN bands).

**If app gauge and API score differ,** the cause is almost always different scan inputs: the app uses
the scanner's band, Bartels limit, detrending and window, while `calculate` uses the body's settings
(defaults: Bartels 10, band 15–400) on exactly the values you send.

### App gauge vs. API score (verified 2026-08-24)

The Cycle Scanner app's "Cycle Consensus Index" gauge computes the same two-phase score: pure cycle consensus first, then a best-effort `ApplyCrsiScore` on top. Three cases silently fall back to the pure cycle consensus (series ≤ 50 bars, CRSI bands not computable, any exception in the CRSI step). Two further differences can move the number relative to an API call on "the same" data:

1. **Inputs and window.** The gauge scores the scanner's peak list under the UI's current settings (band, Bartels, dType, maxbars). Note the app loads `maxbars` up to today and then cuts at `inSampleTimeEnd`, so the in-sample window is shorter than an API series truncated at the cutoff (e.g. 1138 vs. 1150 bars).
2. **CRSI source-cycle derivation.** App and API can derive the CRSI period from different cycles. Verified example (silver `SI=F:YFI`, in-sample cut 2026-08-05, band 15–400, Bartels 10): the API derived CRSI from the 341-bar cycle (period 170), which sat below the lower band turning up — BearFatigue, contribution 13.3, score 93. The app derived CRSI length 77, which contributed nothing, so the gauge showed 80: exactly the pure cycle consensus. Same engine, same formula, different CRSI source cycle.

When gauge and API disagree, check in this order: in-sample bar count, scanner parameters, then the CRSI length the app shows (the "cRSI: n" chart label) against `crsiSourceCycleLength`/`crsiLength` in the API response. A gauge equal to the API's `includeCrsi=false` value means the app's CRSI step did not contribute.

---

## The Predictive Edge

Unlike classical indicators (RSI, MACD, moving averages) that react to price changes after the fact, the consensus score is **predictive by nature**. Cycles project forward — the phase calculation knows where each cycle is heading, not just where it has been.

Key validated findings (walk-forward on DJA daily, 1982–2026, 11,402 bars, 284 intermediate CITs):

- **97% of intermediate trend changes** had a strong consensus signal within ±15 bars
- **70% of signals arrive BEFORE** or exactly AT the turning point — on average 4 bars ahead
- **When consensus is quiet** (|score| < 10), 62% of the time no trend change occurs within 20 bars
- **Bullish signals lead by 5.1 bars** (76% direction accuracy); bearish signals lead by 2.8 bars (52%)

---

## How to Read the Score

| Combined Score | Interpretation | Action |
|---------------|----------------|--------|
| **> +50** | Strong bullish consensus | High alert for bottoming. Look for price confirmation. |
| +30 to +50 | Moderate bullish | Cycles lean bullish. Monitor for entry opportunities. |
| -30 to +30 | Neutral / mixed | No clear cyclical bias. Trend likely to continue. |
| -30 to -50 | Moderate bearish | Cycles lean bearish. Monitor for exit or hedging. |
| **< -50** | Strong bearish consensus | High alert for topping. Look for price confirmation. |

---

## Reading the Bullish and Bearish Consensus Values

The `bullishConsensus` and `bearishConsensus` values carry independent information beyond the combined score:

- **One side dominant, other absent** → Unambiguous cyclical picture. All cycles agree. Highest-conviction signals.
- **Both sides present** → Conflicted cycle landscape. Some cycles topping while others bottoming. Moves may be shallow or short-lived.
- **Both sides quiet** → Dominant cycles in neutral phases. No turn imminent. Clearest "trend continuation" reading.
- **Watching transitions** → One side building while the other fades shows cycles progressively rolling over — early alert before the combined score formally triggers.

---

## Reading the Phase Breakdown

The four phase arrays (their lengths and the `cycleLength` values inside) reveal **how imminent** the
expected turn is:

- **Topping + Falling together** → Bearish with high conviction. Some cycles already turned, others following.
- **Only Falling, no Topping** → Bearish building but inflection hasn't arrived yet.
- **Only Topping, no Falling** → Turn happening now at longer cycles, shorter haven't confirmed.
- Same logic applies symmetrically to the bullish side (Bottoming + Rising).

---

## Use Cases by Analyst Type

**Trend-Following Trader:**
When |score| < 20, the cyclical environment favors trend continuation. This is when trend-following strategies have their highest edge. No cyclical opposition to the current move.

**Swing Trader:**
A strong score (|combined| > 20) is the alert to start looking for entries. 97% of intermediate trend changes (5%+ swings) had a consensus signal within ±15 bars. If you only look for turns when the consensus flags one, you will catch nearly all of them.

**Risk Manager:**
The score quantifies cyclical risk. When strong AND diverging from the current price direction, this represents the highest-risk environment for trend continuation. Position sizing and hedging benefit from this awareness.

---

## Recommended Usage

1. **Use as an alert system** — the score tells you WHEN to pay attention, not WHEN to act. A strong score is a necessary condition for most turns but not sufficient on its own.
2. **Combine with price analysis** — highest conviction when consensus diverges from current price trend.
3. **Check the weekly score too** — call the same endpoint on the weekly dataset; agreement across timeframes is more reliable than daily alone.
4. **Not standalone** — combine with trend analysis, support/resistance, and risk management.
5. **Monitor transitions** — direction changes (flipping bullish to bearish) are more significant than the absolute level persisting.

---

## Validation Summary

| Metric | Value |
|--------|-------|
| CIT recall | **97%** — almost no turns happen without cyclical warning |
| CIT precision | **51%** — half of strong signals land near an actual turn |
| Direction accuracy | **59%** — of signals near a CIT, 59% correctly identify direction |
| Trend continuation | **62%** — quiet consensus → no turn within 20 bars |
| Robustness | **CV 2.1%** — stable across 500–1,300 bar windows |

The score does NOT predict fixed-horizon returns (50/100-day accuracy ≈ 50%). It identifies cyclical turning conditions, which is fundamentally different from predicting future returns.

*Based on walk-forward analysis, March 2026. 19 rounds of systematic testing on DJA daily/weekly data (1982–2026, 11,402 bars, 284 intermediate CITs at 5% threshold).*
