# Cycle Consensus — Practical Application Guide

The Cycle Consensus Score is a **turning point alerting system**. It synthesizes multiple detected market cycles into a single directional reading and answers one question: **Are the cyclical conditions present for a trend change right now?**

It does not predict where price will be in N days. It tells you whether the dominant cycles are converging toward a turning point — and in which direction.

---

## API Endpoints

| Method | Path | Use Case |
|--------|------|----------|
| GET | `/api/CycleConsensus/score/{symbol}` | Production endpoint — compact score with reasoning for a known symbol |
| POST | `/api/CycleConsensus/calculate` | Full analysis from raw close prices — includes individual cycle contributions |
| GET | `/api/CycleConsensus/validate/{symbol}` | Debug — all intermediate pipeline values for verification |

---

## Response Properties Reference

### Score Properties

| Property | Type | Range | Description |
|----------|------|-------|-------------|
| `combinedScore` | double | -100 to +100 | Final consensus score. Positive = bullish, negative = bearish. Cycles contribute ±80 max, CRSI ±20 max. |
| `bullishConsensus` | double | 0–100 | Total weighted strength of cycles voting bullish (rising + bottoming phases). |
| `bearishConsensus` | double | 0–100 | Total weighted strength of cycles voting bearish (falling + topping phases). |

### Cycle Phase Counts

| Property | Description | Imminence |
|----------|-------------|-----------|
| `toppingCycleCount` | Cycles at or near their peak, turning down | **At the turn** — the reversal is happening now |
| `bottomingCycleCount` | Cycles at or near their trough, turning up | **At the turn** — the reversal is happening now |
| `fallingCycleCount` | Cycles declining from peak toward trough | **Approaching** — bearish pressure building |
| `risingCycleCount` | Cycles ascending from trough toward peak | **Approaching** — bullish pressure building |
| `bullishCycleCount` | Total bullish = rising + bottoming | Overall bullish cycle count |
| `bearishCycleCount` | Total bearish = falling + topping | Overall bearish cycle count |

### Contributing Cycle Lengths

| Property | Type | Description |
|----------|------|-------------|
| `bullishCycles` | int[] | Distinct cycle lengths (in bars) contributing to the bullish side (bottoming + rising). Deduplicated, sorted ascending. |
| `bearishCycles` | int[] | Distinct cycle lengths (in bars) contributing to the bearish side (topping + falling). Deduplicated, sorted ascending. |

These arrays let you inspect *which* cycles vote on each side without calling the heavier `/calculate` endpoint. Useful for filtering ("is the bullish vote driven by a short-cycle cluster or a long-cycle structural signal?") and for charting cycle composition alongside the score.

### App UI parity (Cycle Scanner gauge)

The Cycle Consensus gauge in the Cycle Scanner app UI (app.cycles.org/cyclescanner)
runs the **same pipeline as the API with CRSI integration enabled** — equivalent to
`includeCrsi=true` on `/api/CycleConsensus/calculate` (the `score/{symbol}` endpoint
always integrates CRSI). There is no app setting or URL parameter to disable the CRSI
term; the app's `crsi=true` URL parameter only toggles the cRSI *chart indicator
panel*, not the consensus integration. The app falls back to the pure cycle consensus
(no CRSI term) only when CRSI cannot be computed: fewer than ~50 bars loaded, or the
derived CRSI length exceeds dataLength/3 (NaN bands).

**If app gauge and API score differ**, the cause is almost always different scan
inputs — the app uses the scanner's configured band/Bartels/detrend/maxbars and
closes up to `inSampleTimeEnd`, while the API defaults to `barCount=1150`,
`bartelsLimit=10`, band 15-400 — not a CRSI on/off difference.

### CRSI Properties

| Property | Type | Description |
|----------|------|-------------|
| `crsiScore` | int | Raw CRSI score (-3 to +3). Drives signal state and score contribution. |
| `crsiSignal` | string | Named signal state — see Signal States table below. |
| `crsiLength` | int | CRSI oscillator period (half the dominant cycle length, clamped 5–50). |
| `crsiSourceCycleLength` | int | Full cycle length from which CRSI period was derived. |

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

**Divergence-confirmed variants** (when price-CRSI divergence is detected alongside Fatigue or Exit):
- `BullFatigueReversalConfirmed` / `BearFatigueReversalConfirmed`
- `BullExitReversalConfirmed` / `BearExitReversalConfirmed`

Divergence affects the signal label only — the numeric contribution remains unchanged (pilot stage).

> **See also:** `crsi-signals.md` explains how the API derives `crsiScore` and `crsiSignal` from the CRSI bands.

### Metadata Properties

| Property | Type | Description |
|----------|------|-------------|
| `symbol` | string | Symbol identifier (score endpoint only) |
| `analysisDate` | string | Date of last price bar (yyyy-MM-dd, score endpoint only) |
| `barsAnalyzed` | int | Total price bars used in the analysis |
| `reasoning` | string | Human-readable step-by-step breakdown of how the score was derived |

### Additional Properties (calculate endpoint only)

| Property | Type | Description |
|----------|------|-------------|
| `hasBearishDivergence` | bool | Price-CRSI bearish divergence detected near last bar. |
| `hasBullishDivergence` | bool | Price-CRSI bullish divergence detected near last bar. |
| `toppingCycles` | array | Individual cycle contributions in topping phase. |
| `bottomingCycles` | array | Individual cycle contributions in bottoming phase. |
| `fallingCycles` | array | Individual cycle contributions in falling phase. |
| `risingCycles` | array | Individual cycle contributions in rising phase. |

Each cycle contribution contains: `cycleLength`, `phaseScore` (-2=topping, -1=falling, +1=rising, +2=bottoming), `strength`, `stabilityScore` (0–1), `rank` (dominant peak rank, 0=unranked), `cycleQuality` (composite weight), `contribution` (weighted value added to consensus), `reason` (human-readable explanation).

### Score Composition Formula

```
CycleScore  = Direction × Confidence × 100, capped at ±80
CrsiContrib = (crsiScore / 3) × 20
FinalScore  = CycleScore + CrsiContrib, clamped to ±100
```

- Cycles alone max out at **±80** — full conviction requires CRSI agreement
- CRSI alone can add at most **±20** — it cannot overpower or flip the cycle signal
- The `reasoning` field shows each step of this calculation

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

The `toppingCycleCount`, `bottomingCycleCount`, `risingCycleCount`, and `fallingCycleCount` fields — together with the `bullishCycles` / `bearishCycles` length arrays (or the full cycle arrays in the `calculate` endpoint) — reveal **how imminent** the expected turn is:

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
