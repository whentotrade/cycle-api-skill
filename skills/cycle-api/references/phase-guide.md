# Cycle Phase Output — Reading Guide

How the CycleScanner and CycleExplorer report cycle phase, and how to read those fields correctly.

---

## 1. What Is a Cycle Phase?

Every detected cycle traces a sine wave. At any moment, a cycle sits at a specific position on that wave — its **phase**. The phase tells you where you are in the cycle's repeating rhythm:

```
         Peak (topping)
          ╭───╮
         ╱     ╲
        ╱       ╲
Rising ╱         ╲ Falling
      ╱           ╲
─────╱─────────────╲─────
    ╱               ╲
   ╲                 ╱
    ╰───────────────╯
     Trough (bottoming)
```

The cycle progresses through four broad stages:

1. **Bottoming** — cycle is at or near its trough (low point)
2. **Rising** — cycle is ascending from trough toward peak
3. **Topping** — cycle is at or near its peak (high point)
4. **Falling** — cycle is descending from peak toward trough

This is a continuous loop. After falling completes, the cycle bottoms again and the sequence repeats.

---

## 2. Phase Strings — The 10 Detailed Phases

The CycleScanner breaks the cycle into 10 distinct phase strings that describe the position with more precision than the four broad stages:

| Phase String | Stage | What It Means |
|---|---|---|
| `BOTTOM_Arrival` | Bottoming | Approaching the trough — the cycle is about to bottom |
| `BOTTOM_Departure` | Bottoming → Rising | Leaving the trough — the turn upward has begun |
| `Uptrend_Starting` | Rising | Early uptrend — ascending momentum is building |
| `Uptrend_Neutral` | Rising | Mid-uptrend — ascending but not yet near peak |
| `Uptrend_ApproachingTop` | Rising → Topping | Late uptrend — nearing the peak, upside limited |
| `TOP_Arrival` | Topping | Approaching the peak — the cycle is about to top |
| `TOP_Departure` | Topping → Falling | Leaving the peak — the turn downward has begun |
| `Downtrend_Starting` | Falling | Early downtrend — descending momentum is building |
| `Downtrend_Neutral` | Falling | Mid-downtrend — descending but not yet near trough |
| `Downtrend_ApproachingBottom` | Falling → Bottoming | Late downtrend — nearing the trough, downside limited |

The progression is always:
```
BOTTOM_Arrival → BOTTOM_Departure → Uptrend_Starting → Uptrend_Neutral →
Uptrend_ApproachingTop → TOP_Arrival → TOP_Departure → Downtrend_Starting →
Downtrend_Neutral → Downtrend_ApproachingBottom → BOTTOM_Arrival → ...
```

---

## 3. Phase Score — The Numeric Position

Each phase string is paired with a numeric **phase score** (`avgPhaseScore`), ranging from -100 to +100. But the score does NOT sweep continuously from -100 to +100. It follows **two separate arcs** with sign flips at the peak and trough:

**Rising arc** (trough → peak):
```
-100 → -95 → [JUMP to +30] → +40 → +50 → +60 → +80 → +95 → +100
```

**Falling arc** (peak → trough):
```
+100 → +95 → [JUMP to -30] → -40 → -50 → -60 → -80 → -95 → -100
```

The sign flips occur because:
- At the **trough**: the score runs -95 (`BOTTOM_Arrival`) → -100 (`BOTTOM_Departure`) → -95 (`Uptrend_Starting`), then jumps to +30 on entering `Uptrend_Neutral`
- At the **peak**: the score runs +95 (`TOP_Arrival`) → +100 (`TOP_Departure`) → +95 (`Downtrend_Starting`), then jumps to -30 on entering `Downtrend_Neutral`

Most phases return a **fixed score value**. Only two phases span a range:
- `Uptrend_Neutral`: avgPhaseScore ranges from 30 to 60
- `Downtrend_Neutral`: avgPhaseScore ranges from -30 to -60

### Phase String + Score Pairing

| Phase String | avgPhaseScore | Fixed or Range |
|---|---|---|
| `BOTTOM_Arrival` | -95 | fixed |
| `BOTTOM_Departure` | -100 | fixed |
| `Uptrend_Starting` | -95 | fixed |
| `Uptrend_Neutral` | 30 to 60 | range |
| `Uptrend_ApproachingTop` | 80 | fixed |
| `TOP_Arrival` | 95 | fixed |
| `TOP_Departure` | 100 | fixed |
| `Downtrend_Starting` | 95 | fixed |
| `Downtrend_Neutral` | -30 to -60 | range |
| `Downtrend_ApproachingBottom` | -80 | fixed |

> **Note:** The sign flips make the raw score non-intuitive. `Uptrend_Starting` has score -95 (negative, but bullish!) and `Downtrend_Starting` has score +95 (positive, but bearish!). Always use the phase string for direction — the score alone is misleading without it.

---

## 4. Average vs Current — Two Independent Phase Measurements

This is the most critical concept to understand. The CycleScanner produces **two independent phase measurements** for every detected cycle. Getting these confused is the #1 source of bugs.

### How Two-Pass Detection Works

1. **First pass (cycle length):** Analyzes the full dataset to detect the dominant cycle length (e.g., 284 bars). Both groups share this same cycle length.

2. **Second pass — average phase:** Sweeps the full history, tracks where the cycle sits at each repetition, and averages the phase across all repetitions. This gives the best statistical estimate based on the entire dataset.

3. **Second pass — current phase:** Uses only recent data to determine where the cycle is right now. Same cycle length, but the phase is fitted to recent bars only — more responsive to the actual current position.

### The Two Field Groups

**Average Group** — for scoring, screening, and historical analysis:

| Field | Description |
|---|---|
| `avgPhaseScore` | Phase position (-100 to +100), averaged across all cycle repetitions |
| `avgPhaseStatus` | Phase label matching avgPhaseScore (one of the 10 strings) |
| `minBarNum` | Trough bar position from averaged phase fit |

**Current Group** — for projection and forward timing:

| Field | Description |
|---|---|
| `phaseScore` | Phase position (-100 to +100), from recent data only |
| `phaseStatus` | Phase label matching phaseScore (one of the 10 strings) |
| `minBarNumCurrent` | Trough bar position from current cycle fit |

### Which Group to Use

| Use Case | Group | Why |
|---|---|---|
| Scoring | **Average** | Smoothed, robust, best for composite scores |
| Regime classification | **Average** | Historical context gives the structural view |
| Dashboard display / screening | **Average** | Stable, less noisy between scans |
| Forward projection / timing | **Current** | Reflects actual current position |
| Sine wave overlay on price chart | **Current** | `minBarNumCurrent` anchors to recent data |
| Next top/bottom estimate | **Current** | Most accurate timing from recent phase fit |

### The Golden Rule: Never Mix Groups

Each group is a matched set. **Never cross fields between groups.**

- **Average group:** `avgPhaseScore` + `avgPhaseStatus` + `minBarNum`
- **Current group:** `phaseScore` + `phaseStatus` + `minBarNumCurrent`

Using `avgPhaseScore` with `phaseStatus`, or `minBarNumCurrent` with `avgPhaseStatus`, produces inconsistent results — each field was computed from a different phase fit.

### When the Groups Disagree

When both groups agree (e.g., both say `Downtrend_Starting`), conviction is highest — the current cycle is tracking its historical pattern.

When they diverge, the current cycle has drifted from its historical average. This divergence is informative:

**Example:** A cycle returns:
- Average: `avgPhaseScore=-80`, `avgPhaseStatus="Downtrend_ApproachingBottom"`, `minBarNum=149`
- Current: `phaseScore=-60`, `phaseStatus="Downtrend_Neutral"`, `minBarNumCurrent=713`

The average (full history) says we're approaching the bottom. The current (recent fit) says we're still mid-downtrend. The current cycle is **running behind** the historical pattern — the bottom may come later than the average suggests.

---

## 5. Fallback Phases (CycleExplorer)

The simpler `CycleExplorer` endpoint returns only four basic phase strings instead of the 10 detailed ones. These map to broader positions:

| Simple Phase | Meaning |
|---|---|
| `Bottom` | At or near the trough |
| `Rising` | Between trough and peak |
| `Top` | At or near the peak |
| `Falling` | Between peak and trough |

The detailed 10-phase strings from `CycleScanner` are always preferred when available.

---

## 6. Summary of Critical Rules

1. **Never mix field groups.** Average fields (`avgPhaseScore`, `avgPhaseStatus`, `minBarNum`) and current fields (`phaseScore`, `phaseStatus`, `minBarNumCurrent`) are computed from different phase fits. Mixing them produces wrong results.

2. **Use the average group for scoring and regime classification.** It's smoothed across all cycle repetitions, so it's more stable.

3. **Use the current group for projection and timing.** It reflects where the cycle is right now, including drift from the historical average.

4. **Phase score signs are non-intuitive.** `Uptrend_Starting` has a negative score (-95); `Downtrend_Starting` has a positive score (+95). Always read the phase string for direction.

5. **Group divergence is a signal.** When the average and current groups disagree, the cycle is drifting from its historical pattern.
