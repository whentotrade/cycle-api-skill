# CRSI Signals — How the API Scores CRSI

## Overview

The Cyclic RSI (CRSI) is a smoothed cyclic RSI oscillator that oscillates between 0 and 100, with dynamically calculated upper band (UB) and lower band (LB) that define overbought and oversold zones. The Cycle Consensus endpoints convert CRSI's position relative to its bands into a discrete `crsiScore` (−3 to +3) and a named `crsiSignal`. This guide explains how that value is derived, so you can read it correctly.

The API applies two rules when it computes CRSI for consensus:
- **CRSI period** is set to **half the dominant cycle length** detected for the series. CRSI as a momentum oscillator should detect turning points *within* the cycle, not track the full period.
- **Band constraint**: The CRSI endpoint needs ~3 full cycle repetitions to compute valid Bollinger-style bands. Always cap the dominant cycle length at `dataLength / 3` before passing it to CRSI. See the CRSI endpoint documentation for details.

---

## The Discrete CRSI Signal (−3 to +3)

**Purpose:** Discretize CRSI momentum into 7 integer states based on position relative to Bollinger-style bands, direction, and recent band crossings. Produces a signal value from −3 (strongest bearish) to +3 (strongest bullish).

### Direction Calculation

Simple **1-bar delta**: compares `crsi[current]` vs `crsi[current − 1]`.

- Rising: `crsi[i] > crsi[i-1]`
- Falling: `crsi[i] < crsi[i-1]`

No threshold — any single-bar change determines direction.

### 7-State Signal Logic

The logic has a **temporal dimension** — it checks whether CRSI was beyond bands in the recent past (last 4 bars) to detect completed crossings:

**Step 1: Is CRSI currently within bands?**

If yes → look back up to 4 bars for a recent extreme:
- Was above UB within last 4 bars → **+3 (Bull Exit)**: completed reversal from overbought
- Was below LB within last 4 bars → **−3 (Bear Exit)**: completed reversal from oversold
- No recent extreme → **0 (Neutral)**

**Step 2: CRSI is above upper band**

- CRSI declining (crsi[i] < crsi[i−1]) → **+2 (Bull Fatigue)**: overbought and starting to turn down
- CRSI still rising (crsi[i] ≥ crsi[i−1]) → **+1 (Bull Exhaustion)**: overbought and still pushing higher

**Step 3: CRSI is below lower band**

- CRSI rising (crsi[i] > crsi[i−1]) → **−2 (Bear Fatigue)**: oversold and starting to turn up
- CRSI still falling (crsi[i] ≤ crsi[i−1]) → **−1 (Bear Exhaustion)**: oversold and still pushing lower

| Signal | Value | Condition | Meaning |
|--------|-------|-----------|---------|
| Bull Exit | +3 | Was above UB, now within bands (last 4 bars) | Overbought move completed, crossed back inside |
| Bull Fatigue | +2 | Above UB, CRSI declining | Overbought, momentum turning down |
| Bull Exhaustion | +1 | Above UB, CRSI still rising | Overbought, momentum overstretched |
| Neutral | 0 | Within bands, no recent extreme | No signal |
| Bear Exhaustion | −1 | Below LB, CRSI still falling | Oversold, momentum overstretched |
| Bear Fatigue | −2 | Below LB, CRSI rising | Oversold, momentum turning up |
| Bear Exit | −3 | Was below LB, now within bands (last 4 bars) | Oversold move completed, crossed back inside |

**Important naming convention:** The names follow the *momentum move*, not the price implication. "Bull Exhaustion" (+1) means the *bullish CRSI move* is exhausted (overbought), which is actually a cautionary signal for price. "Bear Fatigue" (−2) means the *bearish CRSI move* is fatiguing (oversold, turning up), which is actually a constructive signal for price.

When price-CRSI divergence is detected, Fatigue/Exit signals are upgraded to divergence-confirmed variants: `BullFatigueReversalConfirmed`, `BearFatigueReversalConfirmed`, `BullExitReversalConfirmed`, `BearExitReversalConfirmed`. Divergence affects the signal label only — it does not change the numeric value.

## How the signal enters the consensus score

CRSI contributes `(crsiScore / 3) × 20`, so at most ±20 of the ±100 combined score. It confirms or dampens the cycle reading. It can only change the sign when the cycle score itself is within ±20. See `consensus-guide.md` § *Score Composition Formula*.
