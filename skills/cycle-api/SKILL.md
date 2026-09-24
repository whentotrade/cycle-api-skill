---
name: cycle-api
description: >
  How to use the Cycle Analysis API (api.marketzeitgeist.com) from an AI agent or code:
  loading market data, detecting dominant cycles, scanning the cycle spectrum, applying
  DSP filters (detrend, smoothing, CRSI), finding trend turning points, clustering cycle
  lengths, and getting cycle consensus scores, plus how to read every result. Use this
  skill whenever the user asks to run cycle analysis, scan for cycles, find the dominant
  cycle, detrend or smooth a series, compute CRSI, detect turning points, get a consensus
  score, or interpret cycle phase, strength, stability, or CRSI signals returned by the API.
  Trigger on phrases like "run cycle analysis", "what is the dominant cycle", "scan cycles",
  "detrend this data", "apply CRSI", "detect turning points", "smooth this series",
  "what cycles exist in", "cycle spectrum", "tops and bottoms", "consensus score",
  "bullish or bearish consensus", "cycle consensus", "CRSI signal", "cycle phase",
  or any request involving the Cycle Analysis API.
---

# Cycle Analysis API

Base URL: `https://api.marketzeitgeist.com`
Auth: the key in the `X-API-Key` header on every request (`?api_key=` in the query works too, but ends up in logs)
API key: created on the API page of the app (app.marketzeitgeist.com → Account → API; FSC members: app.cycles.org)
Live schema (Swagger): https://api.marketzeitgeist.com/specs/index.html?url=/apidocs/v1/swagger.json

This skill explains how to use the API and how to read its results. For exact parameter
types and response schemas, the live Swagger is authoritative.

| Reference | What's in it |
|---|---|
| `references/endpoints.md` | Every endpoint: parameters, request bodies, responses |
| `references/request-examples.md` | Working code in C#, JavaScript and Python |
| `references/phase-guide.md` | Cycle phase strings, phase scores, average vs current field groups |
| `references/consensus-guide.md` | Cycle Consensus score: fields, formula, how to read it |
| `references/crsi-signals.md` | How the API derives `crsiScore` / `crsiSignal` |
| `pipeline-tester.html` | Browser tool to try the full pipeline with your key |

---

## Key levels — read this first

What a key may do depends on its level. Check `GET /api/me/limits` before choosing a pipeline.

| Level | Market data (`/api/data/*`, `MarketCycles`, `LastTopsAndBottoms`, consensus `score`) | Own data (analysis routes with a body or `?datasetid=`) | Streams |
|---|---|---|---|
| **Guest** (free, FSC members) | **no** — these routes answer `403` with a message naming the feature | yes: send the values with the call, or store up to 3 datasets (`PUT /api/datasets/{name}`) and name them with `?datasetid=NAME` | no |
| Paid tiers | only with the feature `MarketDataAccess` set by the operator | yes, larger datasets | yes, by the tier's number |
| PRO-level features (`useStability`, `dominantPeakFinder`, `CycleSpectrumPeakFinder`) | need a PRO-level key; otherwise the answer's `license` text says "skipped" and the scores stay 0 | | |

**Guest pipeline:** your closes (at least 100) → `CycleScanner` / `CycleExplorer` / `CRSI` with the array as the body,
or once `PUT /api/datasets/MYSERIES` with `[{dateUnix, close}, …]` and then `POST /api/cycles/CycleScanner?datasetid=MYSERIES`.
The "Standard Pipeline" below (symbol search → market data) needs market data by key and is **not** available to Guest.

**Answers:** `401` no valid key · `403` with a message: not in your tier or features, retrying does not help · `429`
with `Retry-After`: rate limit · `429` with a quota message: the monthly cap (Guest: 2,000 key calls; streams never count).

---

## Request Format

**GET**
```http
GET /api/<group>/<endpoint>?api_key=<key>&param=value
Accept: application/json
```

**POST**
```http
POST /api/<group>/<endpoint>?api_key=<key>&param=value
Accept: application/json
Content-Type: application/json

[123.4, 124.1, 122.8, ...]
```

POST body is always a **raw JSON array of doubles** — never wrapped in an object.

---

## Endpoints

### Cycles

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/cycles/MarketCycles/{symbol}` | Dominant cycle for a known market symbol |
| POST | `/api/cycles/CycleExplorer` | Dominant cycle for a custom data array |
| POST | `/api/cycles/CycleScanner` | Full cycle spectrum |
| POST | `/api/cycles/CycleSpectrumPeakFinder` | Spectrum peak finder *(PRO)* |
| GET | `/api/cycles/LastTopsAndBottoms` | Last cycle tops and bottoms for a symbol |
| GET | `/api/cycles/APILimits` | API usage and rate limits |

### Data

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/data/SearchSymbols` | Search for symbols — returns `symbolId` |
| GET | `/api/data/EnsureCompleteDataset` | Check if dataset needs updating |
| GET | `/api/data/WaitUntilUpdateCompleted` | Wait for a background data fetch |
| GET | `/api/data/GetDatasetSeries` | Load OHLCV time-series bars |

> **Updating data is a two-step process — there is no single `/UpdateDataset` endpoint:**
> 1. `EnsureCompleteDataset?tickerId=<id>&unixFrom=0&unixTo=<currentUnixTimestampSeconds>&lastclose=true`
>    → returns `{ isComplete: bool, trackingId: string|null }`
> 2. If `isComplete` is **false** → call `WaitUntilUpdateCompleted?requestId=<trackingId>&timeoutSeconds=30`  
>    → returns `{ status: true/false, duration: ms }`

### Cycle Consensus

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/CycleConsensus/score/{symbol}` | Consensus score for a symbol (compact production endpoint) |
| POST | `/api/CycleConsensus/calculate` | Full consensus from raw close prices (detailed response) |
| GET | `/api/CycleConsensus/validate/{symbol}` | Debug: full pipeline with intermediate values |

### DSP

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/DSP/RSDtest` | Trend turning point detection |
| POST | `/api/DSP/SavGol` | Savitzky-Golay smoothing |
| POST | `/api/DSP/Detrend` | Detrending (HP / Spline / Poly) |
| POST | `/api/DSP/SincSmoother` | Modified Sinc Smoother |
| POST | `/api/DSP/CRSI` | Cyclic Smooth RSI oscillator |
| POST | `/api/DSP/CyclePowerScanner` | Cycle power spectrum v2 |
| POST | `/api/DSP/kde` | KDE 1D clustering (custom params) |
| GET | `/api/DSP/kde-auto` | KDE clustering (auto) |
| GET | `/api/DSP/kde-auto-summary` | KDE clustering (auto + summary) |

### Stream

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/Stream/SubmitStreamData` | Submit live stream data *(tiers and memberships with streaming; 300 updates per stream and day)* |

---

## Standard Pipeline (market data by key — needs `MarketDataAccess`, not available to Guest)

```
SearchSymbols                       → symbolId  (field is symbolId, not tickerid)
    ↓
EnsureCompleteDataset               → { isComplete, trackingId }
    ↓  only if isComplete=false
WaitUntilUpdateCompleted            → { status, duration }
    ↓
GetDatasetSeries                    → OHLCV bars
    ↓  extract closes: bars.map(b => b.close ?? b.Close)
CycleExplorer / CycleScanner / CRSI → pass closes[] as raw JSON array body
```

**Parameter naming differences to watch:**
- `EnsureCompleteDataset` uses `tickerId` (capital I)
- `GetDatasetSeries` uses `tickerid` (all lowercase)
- `GetDatasetSeries` response is mixed text + JSON — parse the `[...]` array from the body

---

## Response Schemas

### `DominantCycleAnalysisSet` — MarketCycles, CycleExplorer
```json
{
  "symbol": "SP500",
  "length": 42.5,
  "amplitude": 18.3,
  "phase": 0.72,
  "phase_status": "Rising",
  "phase_score": 85,
  "nexttop": 12,
  "nextlow": 33,
  "lasttop": -10,
  "lastlow": -31,
  "phasingScore": 78,
  "cycleProfitability": 0.62,
  "barsused": 290,
  "statusCode": "OK",
  "timeSeries": []
}
```

### `CycleScannerResults` — CycleScanner
```json
{
  "peaks": [
    { "cycleLength": 42.0, "amplitude": 18.3, "bartelsValue": 72.4,
      "strength": 0.88, "dominantRank": 1, "minBarNum": 215, "stabilityScore": 0.68,
      "phaseStatus": "Rising", "phaseScore": 40,
      "avgPhaseStatus": "Uptrend_Neutral", "avgPhaseScore": 30 }
  ],
  "spectrum": [0.1, 0.3, 0.8, ...],
  "datapoints": 500,
  "statusCode": "OK"
}
```

**Peak field reference:**
- `cycleLength` — period in bars (e.g. 160). **Filter out < 30 bars as noise**
- `amplitude` — cycle amplitude (e.g. 25.0)
- `minBarNum` — bar index anchoring the cycle trough (used for sine wave plotting)
- `strength` — raw score (e.g. 2.8), **NOT a percentage**. Use gap-based clustering to find dominant cycles: relative strength differences between peaks matter more than absolute values
- `stabilityScore` — 0 to 1 range, display as `(score * 100)%`. Values >= 0.5 = good for projection. **Filter out < 0.4 as noise** (PRO-level keys only: without the PRO level every score is 0, see Key levels)
- `dominantRank` — 1 = most dominant, 0 = not ranked
- `bartelsValue` — Bartels significance score
- `spectrum` — full amplitude array for spectrum visualization (requires `includeSpectrum: true`)

**Phase Output — Two-Pass Cycle Detection:**

The CycleScanner uses a **two-pass detection circuit**. The first pass detects the **cycle length** from the full data series using spectral analysis. The second pass then determines the **phase** — where we are within that detected cycle. This phase detection runs twice, producing two independent measurements:

**How it works:**
1. **First pass (cycle length):** Analyses the full historical dataset to find the dominant cycle length (e.g., C284 = 284 bars). This is a single value — both groups share the same detected cycle length.
2. **Second pass (phase — average):** Sweeps the full history, tracking where the detected cycle sits at each repetition, and averages the phase across all repetitions. This gives the best statistical estimate of the cycle's position based on the entire dataset.
3. **Second pass (phase — current):** Uses only the recent past to determine where the cycle is right now. Same cycle length, but the phase is fitted to recent data only, making it more responsive to the actual current position.

**Average Group — for scoring and historical analysis:**

| Field | Description |
|-------|-------------|
| `avgPhaseScore` | Phase position (-100 to +100), averaged across all cycle repetitions |
| `avgPhaseStatus` | Phase label matching avgPhaseScore |
| `minBarNum` | Trough bar position from averaged phase |

Use the average group when you want the **best overall reading** based on the full history — the most statistically robust answer to "where does this cycle typically sit at this point?" This is the preferred group for **scoring** because it smooths out noise from individual cycle repetitions. The average gives you the long-term structural view of the cycle's position, which is most relevant for regime classification and composite score computation.

**Current Group — for projection and forward timing:**

| Field | Description |
|-------|-------------|
| `phaseScore` | Phase position (-100 to +100), from recent data only |
| `phaseStatus` | Phase label matching phaseScore |
| `minBarNumCurrent` | Trough bar position from current cycle |

Use the current group when you need to know **where we are right now** with precision — the most accurate answer to "when is the next top or bottom?" This group fits the phase to recent history only, so it captures any drift or acceleration in the current cycle relative to the historical average. This is the preferred group for **forward projection** because if the current cycle is running ahead of or behind the average, the projection should reflect the actual current position, not the historical average.

**When the two groups agree** (e.g., both say "Downtrend_Starting"), conviction is highest — the current cycle is tracking its historical pattern. **When they diverge** (e.g., average says "Downtrend_ApproachingBottom" but current says "Downtrend_Neutral"), the current cycle is running behind schedule relative to history. The divergence itself is informative: it tells you whether the cycle is accelerating or decelerating versus its typical rhythm.

**Summary — which group to use:**

| Use case | Group | Why |
|----------|-------|-----|
| Scoring | **Average** | Smoothed, robust, best for composite scores |
| Regime classification | **Average** | Historical context gives the structural view |
| Forward projection / timing | **Current** | Reflects actual current position, not historical average |
| Sine wave plot for projection | **Current** | `minBarNumCurrent` anchors the plot to recent data |
| Sine wave plot for historical | **Average** | `minBarNum` anchors the plot to averaged cycle |
| Next top/bottom estimate | **Current** | Most accurate timing from recent phase fit |

**Three matched groups — never cross them:**

- **Average group:** `avgPhaseScore` + `avgPhaseStatus` + `minBarNum`
- **Current group:** `phaseScore` + `phaseStatus` + `minBarNumCurrent`
- **Never mix** fields between groups. Using `avgPhaseScore` with `phaseStatus` or `minBarNumCurrent` with `avgPhaseStatus` produces inconsistent results — each field is computed from a different phase fit.

**Example:** A peak returns `avgPhaseScore=-80` with `avgPhaseStatus="Downtrend_ApproachingBottom"` and `minBarNum=149`. Simultaneously, it returns `phaseScore=-60` with `phaseStatus="Downtrend_Neutral"` and `minBarNumCurrent=713`. The average (full history) says we're approaching the bottom; the current (recent fit) says we're still mid-downtrend. The current cycle is running behind the historical pattern — the bottom may come later than the average suggests.

See **Phase strings and scores** below and `references/phase-guide.md` for every phase string and its score.

**Filtering guidelines:**
- Discard peaks with `cycleLength < 30` — too short, dominated by noise
- Discard peaks with `stabilityScore < 0.4` — unstable, unreliable for projection (only when the key has the PRO level; otherwise every score is 0)
- **Cap cycle length at `dataLength / 3`** — the CRSI endpoint requires approximately 3 full cycle repetitions to compute valid Bollinger-style upper/lower bands. Cycles longer than one-third of the data length will produce `NaN` for the `ub` and `lb` arrays, making band-relative scoring impossible. For example, a 205-bar cycle in a 583-bar dataset (ratio 2.8) returns all-NaN bands. Filter these out before selecting the dominant cycle for CRSI tuning.
- When selecting dominant cycles, look for natural strength gaps between groups rather than using fixed cutoffs

**Sine wave formula** (plot any cycle as an oscillator):
```
angle = 2 * PI * (barIndex - minBarNum) / cycleLength - PI/2
value = amplitude * sin(angle)
```
The `-PI/2` offset places a trough at `minBarNum`. Project forward by generating future business days (skip weekends).

### `ConsensusScoreResponse` — CycleConsensus/score
```json
{
  "symbol": "SPY",
  "combinedScore": 42.3,
  "bullishConsensus": 68.5,
  "bearishConsensus": 31.2,
  "bullishCycleCount": 5,
  "bearishCycleCount": 3,
  "toppingCycleCount": 2,
  "bottomingCycleCount": 3,
  "risingCycleCount": 2,
  "fallingCycleCount": 1,
  "bullishCycles": [22, 47, 89, 144, 215],
  "bearishCycles": [33, 60, 110],
  "crsiScore": 1,
  "crsiSignal": "BullExhaustion",
  "crsiLength": 10,
  "crsiSourceCycleLength": 40,
  "analysisDate": "2026-04-01",
  "barsAnalyzed": 1150,
  "reasoning": "Bullish: 68.5 | Bearish: 31.2\nRaw combined: -37.3, capped +/-80: -37.3\nCRSI (period 10, from 40-bar cycle):\n  Score: +1 -- Above upper band, still rising (overbought)\n  Contribution: +6.7\nSignal: BullExhaustion\nFinal: -37.3 + (+6.7) = -30.6, display: +31"
}
```

`bullishCycles` and `bearishCycles` are distinct cycle lengths (in bars), deduplicated and sorted ascending, projected from the contribution buckets. Useful to inspect *which* cycles drive each side without calling the heavier `/calculate` endpoint.

**Score composition:**
- `combinedScore` ranges from **-100** (extreme bearish) to **+100** (extreme bullish)
- Cycle consensus contributes up to **±80** (CycleScoreCap), weighted by cycle quality (strength × stability × dominant rank)
- CRSI contributes up to **±20** (CrsiScoreRange), scaled as `(crsiScore / 3) × 20`
- Positive = bullish, Negative = bearish

**Cycle phase counts:**
- `toppingCycleCount` — cycles at or near peak (bearish: turning down)
- `bottomingCycleCount` — cycles at or near trough (bullish: turning up)
- `risingCycleCount` — cycles ascending from trough toward peak (bullish)
- `fallingCycleCount` — cycles declining from peak toward trough (bearish)

**CRSI Signal States** (named from the exhausting trend's perspective):

| crsiScore | Signal State | Contribution | Meaning |
|-----------|-------------|-------------|---------|
| +1 | BullExhaustion | +6.7 | Overbought, momentum still rising — bull move approaching exhaustion |
| -1 | BearExhaustion | -6.7 | Oversold, momentum still falling — bear move approaching exhaustion |
| +2 | BullFatigue | +13.3 | Overbought, turning down — bull momentum fatiguing |
| -2 | BearFatigue | -13.3 | Oversold, turning up — bear momentum fatiguing |
| +3 | BullExit | +20.0 | Crossed below upper band — bull move exiting |
| -3 | BearExit | -20.0 | Crossed above lower band — bear move exiting |
| 0 | Neutral | 0 | Between bands |

When price-CRSI divergence is detected, Fatigue/Exit signals are upgraded: `BullFatigueReversalConfirmed`, `BearFatigueReversalConfirmed`, `BullExitReversalConfirmed`, `BearExitReversalConfirmed`. Divergence affects the signal label only — it does not change the numeric contribution (pilot stage).

For how the API derives `crsiScore` from the CRSI bands, see `references/crsi-signals.md`.

### `CRSI` — CRSI
```json
{ "crsi": [45.2, 48.1], "ub": [70.0, 70.1], "lb": [30.0, 29.9] }
```

### `EnsureCompleteDataset`
```json
{ "status": "string", "isComplete": true, "trackingId": "string|null" }
```

### `WaitUntilUpdateCompleted`
```json
{ "status": true, "duration": 1234 }
```

---

## Phase Strings and Scores

The CycleScanner reports cycle position as a phase string plus a phase score. The score does
not sweep continuously from -100 to +100. It follows **two arcs** with sign flips:

- **Trough:** -95 (`BOTTOM_Arrival`) → -100 (`BOTTOM_Departure`) → -95 (`Uptrend_Starting`), then a jump to +30 on entering `Uptrend_Neutral`
- **Peak:** +95 (`TOP_Arrival`) → +100 (`TOP_Departure`) → +95 (`Downtrend_Starting`), then a jump to -30 on entering `Downtrend_Neutral`

| Cycle position | Phase string | Score | Type |
|---|---|---|---|
| Late downtrend | `Downtrend_ApproachingBottom` | -80 | fixed |
| At trough | `BOTTOM_Arrival` | -95 | fixed |
| Leaving trough | `BOTTOM_Departure` | -100 | fixed |
| Early uptrend | `Uptrend_Starting` | -95 | fixed |
| Mid uptrend | `Uptrend_Neutral` | 30 to 60 | range |
| Late uptrend | `Uptrend_ApproachingTop` | 80 | fixed |
| At peak | `TOP_Arrival` | 95 | fixed |
| Leaving peak | `TOP_Departure` | 100 | fixed |
| Early downtrend | `Downtrend_Starting` | 95 | fixed |
| Mid downtrend | `Downtrend_Neutral` | -30 to -60 | range |

**Read the phase string for direction.** `Uptrend_Starting` scores -95 and `Downtrend_Starting`
scores +95, so the sign alone is misleading. Use the average pair (`avgPhaseStatus` +
`avgPhaseScore`) or the current pair (`phaseStatus` + `phaseScore`), never one field from each.
Full detail: `references/phase-guide.md`.

---

## Constraints

- Minimum **100 data points** required for all analysis endpoints
- `minCycleLength` ≥ 20 · `maxCycleLength` ≤ 400
- Bartels threshold 0–99, default 49 (lower = include weaker cycles)
- PRO level required: `CycleSpectrumPeakFinder`, `dominantPeakFinder`, `useStability`. Without a PRO-level key the
  answer's `license` text says "Stability scoring skipped: PRO level required" and every `stabilityScore` is 0 —
  **do not filter by `stabilityScore` in that case**, or every cycle is discarded; rank by `strength` instead.
- With a PRO-level key set `dominantPeakFinder: true` and `useStability: true` when calling CycleScanner.
- Rate limits are per tier and endpoint group (`GET /api/me/limits`); the Guest cycles limit is 1 call per second,
  so wait a second between analysis calls.
- Stream endpoint: 300 updates per allowed stream and day; a stream beyond your number of streams is refused.

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Daily-quota errors return **HTTP 200** with text "API calls quota exceeded!" | Regex-check response text before JSON parsing. **Match the explicit string only** — a generic substring like `"quota exceeded"` will also fire on transient 429 throttle bodies and mistakenly surface concurrency limits as billing errors. |
| **HTTP 429 "Too Many Requests" under burst load — separate from daily quota** | The API enforces a per-key concurrent-connection ceiling that is **independent of the daily call quota** (quota headers can show 99,000+ remaining while 429s are firing). Triggered most easily by `WaitUntilUpdateCompleted` calls fired in parallel — each blocks the server for up to 30 seconds. Symptoms: 429 status, `Retry-After: 1` header, body text often containing "quota exceeded" (the misleading bit). **Fix:** wrap `fetch` in a helper that honors `Retry-After` and retries with exponential multiplier + jitter (e.g., 1×, 2×, 4×… up to 5 attempts). Reading the `Retry-After` header is essential — naive blanket retries without honoring the hint will keep getting throttled. |
| **API parameter casing: `dtype` lowercase only** | The CycleScanner and Detrend endpoints expect lowercase `dtype` in query parameters. Capitalized `dType` (capital T) may silently fall through to a different default detrending mode without an error, producing different strength values, stability scores, and phase readings than expected. Always lowercase. |
| `GetDatasetSeries` returns mixed text + JSON in one body | `JSON.parse` first, fallback to `text.match(/\[[\s\S]*\]/)` |
| `EnsureCompleteDataset` uses `tickerId` (capital I), `GetDatasetSeries` uses `tickerid` (lowercase) | Copy param names exactly |
| There is **no single UpdateDataset endpoint** | Two-step: `EnsureCompleteDataset` → `WaitUntilUpdateCompleted` |
| Using `unixTo=0` in `EnsureCompleteDataset` silently fails to fetch data for new/unfamiliar datasets | Always pass the current Unix timestamp (seconds) for `unixTo`: `Math.floor(Date.now() / 1000)` in JS, `int(time.time())` in Python, `DateTimeOffset.UtcNow.ToUnixTimeSeconds()` in C# |
| Index datasets (SP500, NASDAQCOM) return close-only data — no OHLC | Detect with `bars.some(b => b.open != null)`, fall back to line chart |
| `strength` is a raw number (e.g. 3.2), not a percentage | Display as-is with `.toFixed(1)` |
| `stabilityScore` is 0–1 range | Multiply by 100 for display: `(score * 100).toFixed(0) + '%'` |
| **CycleScanner results don't match the WhenToTrade UI application** | The UI application defaults to `dType=0` (HP filter detrending), while API examples often use `dType=9` (no detrending). This causes **completely different strength values, stability scores, and dominant rankings** for the same input data. For example, the same 241-bar dataset produces strength values of 40–99 with `dType=0` vs 1–3 with `dType=9`, and the dominant cycle rank can shift (e.g., C67 appears as rank=1 with `dType=0` but rank=3 with `dType=9`). **Always check `dType` first when API results don't match the UI.** Use `dType=0` to reproduce UI behavior. Use `dType=4` (One-Sided HP / Kalman) when end-point accuracy matters — it avoids the end-of-sample bias of the standard HP filter by only using past observations, producing more reliable phase/status readings at the most recent bars. |
| **CRSI returns NaN for upper/lower bands** | The CRSI endpoint needs ~3 full cycle repetitions to compute valid Bollinger-style bands. If the selected cycle length is longer than `dataLength / 3`, the `ub` and `lb` arrays will contain all NaN values. **Always cap the dominant cycle length at `dataLength / 3` before passing it to the CRSI endpoint.** For example, with 583 data points, max usable cycle length is 194 bars. A C205 cycle will produce all-NaN bands. See `references/crsi-signals.md` for how the API reads the bands. |


---

## Building cycle applications

This skill covers the API itself. For building charts, dashboards and complete cycle
applications on top of it (composite cycle reconstruction, charting integration, phase
visualization, continuous 0-100 scoring methods), see the companion **cycle-tools** skill set.
