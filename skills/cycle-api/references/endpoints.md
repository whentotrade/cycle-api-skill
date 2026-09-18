# Cycle Tools API — Endpoint Reference

Base URL: `https://api.marketzeitgeist.com`  
Auth: append `?api_key=<key>` to every request URL.

---

## Table of Contents
1. [MarketCycles](#1-marketcycles)
2. [CycleExplorer](#2-cycleexplorer)
3. [CycleScanner](#3-cyclescanner)
4. [CycleSpectrumPeakFinder](#4-cyclespectrumpeakfinder)
5. [LastTopsAndBottoms](#5-lasttopsandbottoms)
6. [APILimits](#6-apilimits)
7. [SearchSymbols](#7-searchsymbols)
8. [UpdateDataset](#8-updatedataset)
9. [GetDatasetSeries](#9-getdatasetseries)
10. [RSDtest](#10-rsdtest)
11. [SavGol](#11-savgol)
12. [Detrend](#12-detrend)
13. [SincSmoother](#13-sincsmoother)
14. [CRSI](#14-crsi)
15. [CyclePowerScanner](#15-cyclepowerscanner)
16. [KDE Clustering](#16-kde-clustering)
17. [SubmitStreamData](#17-submitstreamdata)
18. [CycleConsensus Score](#18-cycleconsensus-score)
19. [CycleConsensus Calculate](#19-cycleconsensus-calculate)
20. [CycleConsensus Validate](#20-cycleconsensus-validate)

---

## 1. MarketCycles

**GET** `/api/cycles/MarketCycles/{symbol}`

End-of-day dominant cycle analysis for known market symbols (stocks, indices, forex, crypto).

### Path Parameter
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| symbol | string | ✅ | Symbol ID e.g. `SP500`, `NASDAQCOM`, `AAPL`, `BTCUSD` |

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| marketType | string | `FDS` | Data source: `FDS`, `YFI`, `QDS`, `WCT` |
| analysisDate | string | `today` | Format: `yyyyMMdd`, max last 5 days |
| currentPhase | bool | `true` | Use most current phase |
| minCycleLength | int | `30` | Min cycle to detect (≥20) |
| maxCycleLength | int | `290` | Max cycle to detect (≤400) |
| bartelsLimit | int | `49` | Min Bartels reliability score (0–99) |
| plotForward | int | `100` | Bars to project into future |
| fillMissingWeekdays | bool | `false` | Fill market holiday gaps |
| savgolSmoothing | bool | `false` | Pre-process with Savitzky-Golay |
| includeTimeseries | bool | `false` | Include bar-by-bar signal data |

### Response: `DominantCycleAnalysisSet`
See key schema in SKILL.md. Status 404 if symbol not found.

---

## 2. CycleExplorer

**POST** `/api/cycles/CycleExplorer`

Dominant cycle analysis for a custom data array you supply.

### Request Body
Raw JSON array of doubles (minimum 100 values):
```json
[100.5, 101.2, 99.8, 102.1, ...]
```

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| minCycleLength | int | `30` | Min cycle to detect (≥20) |
| maxCycleLength | int | `290` | Max cycle to detect (≤400) |
| bartelsLimit | int | `49` | Min Bartels score (0–99) |
| dynamicInSampleMethod | bool | `false` | Dynamic in-sample range based on cycle length |
| plotForward | int | `100` | Bars to project into future |
| savgolSmoothing | bool | `false` | Pre-process with Savitzky-Golay |
| includeTimeseries | bool | `false` | Include bar-by-bar signal time-series |

### Response: `DominantCycleAnalysisSet`

---

## 3. CycleScanner

**POST** `/api/cycles/CycleScanner`

Full cycle power spectrum analysis. Returns all significant cycle peaks ranked by strength.

### Request Body
Raw JSON array of doubles (minimum 100 values).

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| amplitudeMulti | double | `1.0` | Multiplier for small-value data (e.g. Forex) |
| bartelsLimit | int | `49` | Min Bartels score to include a peak |
| minCycleLength | int | `5` | Spectrum range min (≥5) |
| maxCycleLength | int | `400` | Spectrum range max (≤400) |
| cycleResolution | decimal | `1.0` | Step size: `1.0` or `0.1` |
| dType | int | `0` | Detrend: 0=HP Filter, 1=Boosted HP, 2=Spline, 3=Poly, 4=One-Sided HP (Kalman), 9=None. **Important: The WhenToTrade UI application defaults to `dType=0` (HP filter). Using `dType=9` produces completely different strength scales and rankings. Always use `dType=0` when results should match the UI. Use `dType=4` for the one-sided (Kalman) HP filter which avoids end-point bias — recommended when the most recent bars matter most.** |
| epf | bool | `false` | Endpoint flattening |
| savgolSmoothing | bool | `false` | Pre-process smoothing |
| savgolSmoothingSpectrum | bool | `false` | Post-process spectrum smoothing |
| sortByStrength | bool | `true` | Sort by strength (true) or amplitude (false) |
| dominantPeakFinder | bool | `false` | AI peak finder — **always set to `true`** for meaningful `dominantRank` values. Without this, all peaks return `dominantRank: 0`. Requires a valid PRO API key. |
| useStability | bool | `false` | Stability scoring — **always set to `true`** for meaningful `stabilityScore` values. Without this, all peaks return `stabilityScore: 0`, which breaks stability-based filtering. Requires a valid PRO API key. |
| includeSpectrum | bool | `false` | Include full spectrum plot data |
| humanReadableText | bool | `false` | Return plain text summary instead of JSON |

### Response: `CycleScannerResults`
```json
{
  "peaks": [
    {
      "cycleLength": 42.0,
      "amplitude": 18.3,
      "bartelsValue": 72.4,
      "strength": 0.88,
      "avgPhase": 0.5,
      "avgPhaseScore": 30,
      "avgPhaseStatus": "Uptrend_Neutral",
      "phase": 0.6,
      "phaseScore": 40,
      "phaseStatus": "Rising",
      "dominantRank": 1,
      "stabilityScore": 0.75,
      "minBarNum": 215,
      "minBarNumCurrent": 480
    }
  ],
  "datapoints": 500,
  "bartelsLimit": 49,
  "sortedBy": "strength",
  "range": "5-400",
  "spectrum": [...],   // only if includeSpectrum=true
  "statusCode": "OK"
}
```

**Peak phase fields — two-pass detection, three matched groups:**

The CycleScanner uses a two-pass detection circuit. The first pass detects the **cycle length** from the full dataset. The second pass determines the **phase** — and runs twice: once averaging across all cycle repetitions (average group), once fitting only to recent data (current group). Both use the same detected cycle length but produce independent phase readings.

**Average group** — use for scoring and historical analysis:

| Field | Description |
|-------|-------------|
| `avgPhaseScore` | Phase position (-100 to +100), averaged across all cycle repetitions |
| `avgPhaseStatus` | Phase label matching avgPhaseScore |
| `avgPhase` | Raw phase angle in radians (averaged) |
| `minBarNum` | Trough bar position from averaged phase |

The average group gives the best statistical reading based on the full history — smoothed across all cycle repetitions. Use for **scoring** (0-100 phase score mapping) and **regime classification** where robustness matters more than precision.

**Current group** — use for projection and forward timing:

| Field | Description |
|-------|-------------|
| `phaseScore` | Phase position (-100 to +100), fitted to recent data only |
| `phaseStatus` | Phase label matching phaseScore |
| `phase` | Raw phase angle in radians (current) |
| `minBarNumCurrent` | Trough bar position from current cycle fit |

The current group reflects where the cycle is right now based on recent data. Use for **forward projection**, **next top/bottom timing**, and **sine wave plotting** where you need the actual current position, not the historical average. If the current cycle is running ahead of or behind the average, this group captures that drift.

**Never cross fields between groups.** Each field is computed from a different phase fit — mixing them produces inconsistent results.

| Use case | Group | Why |
|----------|-------|-----|
| Scoring (0-100 phase score) | Average | Smoothed, robust for composite scores |
| Forward projection / timing | Current | Reflects actual current position |
| Sine wave for projection | Current | `minBarNumCurrent` anchors to recent data |
| Next top/bottom estimate | Current | Most accurate timing |

Additional field:
- `avgPhaseTotalBars` — number of bars used in the average phase calculation

**For every phase string and its score, see the "Phase Strings and Scores" section in SKILL.md and `phase-guide.md`.**

---

## 4. CycleSpectrumPeakFinder

**POST** `/api/cycles/CycleSpectrumPeakFinder`

Identifies dominant peaks from a spectrum using visual criteria. **PRO license required.**  
Typically called after `CycleScanner` with `includeSpectrum=true`.

### Request Body: `FindPeaksSend`
```json
{
  "spectrum": [0.1, 0.4, 1.2, ...],
  "cycleStart": 5,
  "cycleEnd": 400,
  "cycleResolution": 1.0
}
```

### Response: `PeaksFinderResults`
```json
{
  "cyclingScore": 0.82,
  "cyclingRating": "High",
  "peaks_dominant": [42.0, 84.0],
  "peaks": [42, 84, 120]
}
```

---

## 5. LastTopsAndBottoms

**GET** `/api/cycles/LastTopsAndBottoms`

Retrieves last cycle tops and bottoms for a symbol.

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| symbol | string | | Symbol identifier |
| ensureComplete | bool | | Ensure dataset is complete |
| minCycleLength | double | | Min cycle length filter |
| maxCycleLength | double | | Max cycle length filter |
| barCount | int | `850` | Number of bars to use |
| bartelsLimit | int | `10` | Min Bartels score |
| timeframe | long | `0` | Timeframe in minutes (0=daily) |
| exchange | string | `""` | Exchange filter |

### Response: `CycleLastTopsAndBottoms`
```json
{
  "length": 42.0,
  "phase": 0.72,
  "nextTop": 12.0,
  "nextTopTimestamp": "2025-04-15T00:00:00Z",
  "nextBottom": 33.0,
  "nextBottomTimestamp": "2025-05-06T00:00:00Z",
  "lastBarDate": "2025-03-11T00:00:00Z"
}
```

---

## 6. APILimits

**GET** `/api/cycles/APILimits?api_key=<key>`

Returns current API usage and limits for the authenticated user.
Response: plain string summary.

> **Quota Exceeded Error:** When you see `"API calls quota exceeded!"` on any endpoint, it means
> your key's per-plan rate limit has been reached — the API key **was** sent correctly.
> This error can arrive as HTTP 200 with plain-text body or as an HTTP error status.
> Wait a moment and retry, or call this endpoint to check your remaining quota.

---


## 7. SearchSymbols

**GET** `/api/data/SearchSymbols`

Search for valid ticker IDs by symbol or name. Always run this first when the user provides a name or partial ticker — the returned `tickerid` is required by data and analysis endpoints.

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| search | string | ✅ | Ticker symbol (e.g. `AAPL`, `SPY`) or partial name (e.g. `Apple`, `S&P`) |
| limit | int | `20` | Max results to return |

### Response
Array of matching symbols:
```json
[
  { "symbolId": "AAPL.US-D-1:FSC1", "symbol": "AAPL", "shortName": "Apple Inc.", "exchange": "NASDAQ", "currency": "USD", "datafeed": "FSC1", "type": "STOCK" },
  ...
]
```
Use the **`symbolId`** value (not `tickerid`) in subsequent calls to `UpdateDataset`, `GetDatasetSeries`, `MarketCycles`, etc.

---

## 8. UpdateDataset (Two-Step Orchestration)

> ⚠️ **There is no single `/UpdateDataset` endpoint.** This is an MCP-level function that internally calls two separate REST endpoints in sequence. Implement both steps when calling the REST API directly.

### Step 8a — EnsureCompleteDataset

**GET** `/api/data/EnsureCompleteDataset`

Checks if the dataset is current. If a background fetch is needed, starts it and returns a `trackingId`.

Supported feeds: `FSC1` (EODHD), `YFI` (Yahoo), `FDS` (FRED), `CDS` (Crypto), `WCT`.

#### Query Parameters
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| api_key | string | ✅ | Authentication |
| tickerId | string | ✅ | Ticker ID e.g. `AAPL.US-D-1:FSC1` |
| unixFrom | int | — | `0` (fetch from earliest available) |
| unixTo | int | — | Current Unix timestamp in seconds (e.g. `Math.floor(Date.now() / 1000)`). **Do not use `0`** — it silently fails to fetch data for datasets not yet loaded on the server. |
| lastclose | bool | — | `true` |

#### Response
```json
{ "status": "string", "isComplete": true, "trackingId": "string-or-null" }
```
- If `isComplete === true` → dataset is already current, skip Step 8b.
- If `isComplete === false` and `trackingId` is set → proceed to Step 8b.
- If `isComplete === false` and `trackingId` is null → no update was triggered (tick may be unsupported).

### Step 8b — WaitUntilUpdateCompleted

**GET** `/api/data/WaitUntilUpdateCompleted`

Polls until the background fetch finishes or times out (max 30 seconds).

#### Query Parameters
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| api_key | string | ✅ | Authentication |
| requestId | string | ✅ | `trackingId` from Step 8a |
| timeoutSeconds | int | — | `30` (recommended) |

#### Response
```json
{ "status": true, "duration": 1234 }
```
- `status: true` → update succeeded, `duration` is ms elapsed.
- `status: false` → update failed or timed out.

### Typical Pipeline
```
SearchSymbols → EnsureCompleteDataset [→ WaitUntilUpdateCompleted if needed] → GetDatasetSeries → [analysis endpoint]
```

---

## 9. GetDatasetSeries

**GET** `/api/data/GetDatasetSeries`

Returns OHLCV time-series data for a given ticker. The primary endpoint for fetching price data to feed into analysis endpoints.

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| tickerid | string | ✅ | Ticker ID e.g. `SP500:FDS`, `AAPL.US-D-1:FSC1` |
| from | string | `0001-01-01 00:00:00Z` | Start date: `yyyy-MM-dd HH:mm:ssZ` |
| to | string | `2150-01-01 00:00:00Z` | End date: `yyyy-MM-dd HH:mm:ssZ` |
| maxbars | int | `1000` | Max bars to return (0 = all). Trims to most recent N bars. |
| summaryOnly | bool | `false` | Return only metadata header, no price series |

### Response
⚠️ **Mixed format**: The response starts with a plain-text summary header, followed by a JSON array. Do NOT pass the raw response to `JSON.parse()` directly — strip everything before the first `[` character first.

Example raw response:
```
Dataset Series: GSPC.INDX-D-1:FSC1
Bars: 500
...
[{"close":5165.31,"date":"2024-03-13T00:00:00",...}]
```

Parsing pattern:
```javascript
const jsonStart = text.indexOf("[");
const bars = JSON.parse(text.slice(jsonStart));
```

Parsed JSON array of OHLCV bars:
```json
[
  { "date": "2025-03-11T00:00:00Z", "open": 5600.1, "high": 5650.3, "low": 5580.2, "close": 5630.5, "volume": 3200000 },
  ...
]
```
To extract close prices for analysis endpoints: `bars.map(b => b.close)`

---
## 10. RSDtest

**POST** `/api/DSP/RSDtest`

RSD t-Test — detects multiple trend turning points (change-in-trend / CIT) in a time series.

### Request Body
Raw JSON array of doubles.

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| denoise | bool | `true` | Pre-process: de-noise the series |
| window | int | `50` | Sliding window size (bars before/after) |
| conf | int | `80` | Confidence level: `68`, `80`, `90`, `95` |
| border | bool | `false` | Include series border points |
| ftrend | bool | `false` | Return final trend slope |
| ret_type | int | `3` | Return type: 1=full, 2=CIT indices only, 3=CIT indices + result series |

### Response: `RSD_Series`
```json
{
  "cit_index": [45, 112, 198],
  "cit_confidence_level": 80,
  "final_trend_slope": 0.023,
  "values": [
    { "x": 0, "y": 102.5, "y2": 101.8, "ts": 1.4, "cit": 0, "cit_type": 0, "confidence_level": false },
    ...
  ],
  "yin": [...],
  "yout": [...],
  "ts": [...]
}
```
`cit_index` = bar indices where trend turns were detected.

---

## 11. SavGol

**POST** `/api/DSP/SavGol`

Savitzky-Golay smoothing filter. Returns smoothed double array.

### Request Body
Raw JSON array of doubles.

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| sidelength | int | `16` | Half-window: total window = `(2*sidelength)+1` |
| degree | int | `4` | Polynomial degree |

### Response
`double[]` — smoothed values, same length as input.

---

## 12. Detrend

**POST** `/api/DSP/Detrend`

Removes trend component to isolate cyclical behavior. Returns double array.

### Request Body
Raw JSON array of doubles.

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| dtype | int | `0` | 0=HP Filter, 1=Boosted HP, 2=Spline (Cubic), 3=Polynomial, 4=One-Sided HP (Kalman), 9=No detrending |
| lbda | int | `0` | HP lambda value (0 = auto) |
| ret | bool | `false` | `false`=return cyclic component, `true`=return trend component |

### Response
`double[]` — cyclic or trend component depending on `ret`.

---

## 13. SincSmoother

**POST** `/api/DSP/SincSmoother`

Modified Sinc Smoother — improved alternative to Savitzky-Golay. Returns smoothed double array.

### Request Body
Raw JSON array of doubles.

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| m | int | `16` | Kernel half-width |
| degree | int | `4` | Polynomial degree |
| isMS1 | bool | `false` | Use MS1 variant (true) vs MS variant (false) |

### Response
`double[]` — smoothed values.

---

## 14. CRSI

**POST** `/api/DSP/CRSI`

Cyclic Smooth RSI (CRSI) — Lars von Thienen's optimized cyclic RSI oscillator.  
Returns CRSI line plus upper and lower Bollinger-style bands.

### Request Body
Raw JSON array of doubles (typically close prices).

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| length | int | `30` | RSI period length |

### Response: `Crsi_indicator`
```json
{
  "crsi": [45.2, 48.1, 52.3, ...],
  "ub": [70.0, 70.1, 70.2, ...],
  "lb": [30.0, 29.9, 29.8, ...]
}
```
Interpretation: CRSI above `ub` = overbought, below `lb` = oversold.

**CRSI period:** Set to **half the dominant cycle length** detected for the series. CRSI as a momentum oscillator should detect turning points *within* the cycle, not track the full period.

**CRITICAL — Maximum cycle length for valid bands:** The `ub` and `lb` Bollinger-style bands require approximately 3 full cycle repetitions to compute. If `length` exceeds `dataLength / 3`, the band arrays will contain all `NaN` values while the `crsi` array remains valid. Always ensure the `length` parameter does not exceed one-third of the input data length. For example, with 583 data points, maximum usable `length` is 194. Passing `length=205` returns valid CRSI values but all-NaN bands.

**Reading the CRSI output:** For how the Cycle Consensus endpoints turn CRSI's position relative to its bands into the discrete −3 to +3 `crsiScore`, see `crsi-signals.md`.

---

## 15. CyclePowerScanner

**POST** `/api/DSP/CyclePowerScanner`

Cycle Scanner v2 — alternative spectrum analysis. Returns JSON string.

### Request Body
Raw JSON array of doubles (minimum 100 values).

### Query Parameters
Same as `CycleScanner` except no `dType`, `epf`, `dominantPeakFinder`, `useStability` params.

| Param | Type | Default |
|-------|------|---------|
| api_key | string | ✅ |
| amplitudeMulti | double | `1.0` |
| bartelsLimit | int | `49` |
| minCycleLength | int | `5` |
| maxCycleLength | int | `400` |
| cycleResolution | decimal | `1.0` |
| savgolSmoothing | bool | `false` |
| savgolSmoothingSpectrum | bool | `false` |
| sortByStrength | bool | `true` |
| includeSpectrum | bool | `false` |
| humanReadableText | bool | `false` |

### Response
JSON string (parse as `CycleScannerResults` schema).

---

## 16. KDE Clustering

Three endpoints for KDE-based 1D clustering of cycle lengths.

### POST `/api/DSP/kde` — Custom params
```json
{
  "values": [42.0, 43.5, 84.0, 41.0, 85.5],
  "gridSize": null,
  "minDensityDrop": null
}
```
Response: `double[][]` — array of clusters, each containing the values in that cluster.

### GET `/api/DSP/kde-auto?api_key=<key>&values=42,43,84,41,85`
Auto-clusters values in range 30–400. Response: `double[][]`

### GET `/api/DSP/kde-auto-summary?api_key=<key>&values=42,43,84,41,85`
Same as above but returns summary per cluster.
```json
[
  { "median": 42.5, "count": 3, "center": 42.3, "values": [41.0, 42.0, 43.5], "distinctValues": [...] },
  { "median": 84.0, "count": 2, "center": 84.75, "values": [84.0, 85.5], "distinctValues": [...] }
]
```

---

## 17. SubmitStreamData

**POST** `/api/Stream/SubmitStreamData`

Push live bar data to a named stream. Throttled at **60 calls/minute**.

### Request Body: `StreamData`
```json
{
  "streamid": "MY-STREAM-1D",
  "symbolname": "AAPL",
  "messagetype": "UPSERT",
  "dates": ["2025-03-10T00:00:00", "2025-03-11T00:00:00"],
  "values": [215.5, 218.3]
}
```

| Field | Description |
|-------|-------------|
| streamid | Unique stream identifier |
| symbolname | Human-readable symbol name |
| messagetype | `HISTORY` (seed), `CREATE` (empty init), `UPSERT` (add bars) |
| dates | ISO format: `yyyy-MM-ddTHH:mm:ss` |
| values | Matching array of price values |

**Rules:**
- `HISTORY` and `CREATE` used once to initialize a stream
- `UPSERT` always includes the current + previous bar for sync
- `dates` and `values` arrays must be the same length

---

## 18. CycleConsensus Score

**GET** `/api/CycleConsensus/score/{symbol}`

Production endpoint: returns the Cycle Consensus Score for a given symbol. Runs the full pipeline internally — data load, cycle scanning (HP filter detrend), stability scoring, dominant peak detection, consensus calculation, and CRSI integration — and returns a compact result.

The Cycle Consensus Score is a **turning point alerting system** that synthesizes multiple detected market cycles into a single directional reading. It answers: **Are the cyclical conditions present for a trend change right now?**

### Path Parameter
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| symbol | string | ✅ | Symbol identifier (e.g. `SPY`, `AAPL`, `EURUSD`) |

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| barCount | int | `1150` | Number of price bars to load |
| bartelsLimit | int | `10` | Minimum Bartels significance score (0-99) |
| minCycleLength | int | `15` | Minimum cycle length to detect in bars |
| maxCycleLength | int | `400` | Maximum cycle length to detect in bars (max 400) |

### Response: `ConsensusScoreResponse`
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

### Response Field Reference

**Score fields:**
- `combinedScore` — Final score from **-100** (extreme bearish) to **+100** (extreme bullish). Positive = bullish, negative = bearish.
- `bullishConsensus` — Weighted bullish consensus (0–100). Sum of quality-weighted contributions from rising + bottoming cycles.
- `bearishConsensus` — Weighted bearish consensus (0–100). Sum of quality-weighted contributions from falling + topping cycles.

**Score composition:**
- Cycle consensus contributes up to **±80** (CycleScoreCap), weighted by cycle quality = strength × stability × dominant rank.
- CRSI contributes up to **±20** (CrsiScoreRange), scaled as `(crsiScore / 3) × 20`.
- Full ±100 requires both cycles and CRSI to agree. CRSI cannot overpower or flip the cycle signal.

**Cycle counts:**
- `bullishCycleCount` / `bearishCycleCount` — Total cycles in bullish (rising + bottoming) vs bearish (falling + topping) phases.
- `toppingCycleCount` — Cycles at or near their peak. The turn is happening now (bearish).
- `bottomingCycleCount` — Cycles at or near their trough. The turn is happening now (bullish).
- `risingCycleCount` — Cycles ascending from trough toward peak. Bullish pressure building.
- `fallingCycleCount` — Cycles declining from peak toward trough. Bearish pressure building.

**Cycle length arrays:**
- `bullishCycles` — Distinct cycle lengths (in bars) contributing to the bullish side (bottoming + rising). Deduplicated and sorted ascending. Example: `[22, 47, 89, 144, 215]`.
- `bearishCycles` — Distinct cycle lengths (in bars) contributing to the bearish side (topping + falling). Deduplicated and sorted ascending. Example: `[33, 60, 110]`.

Use these when you need to know *which* cycles are voting on each side (e.g. "is the bullish vote driven by short-cycle noise or by a 200-bar structural cycle?") without making a second call to the heavier `/calculate` endpoint.

**CRSI fields:**
- `crsiScore` — Raw CRSI score (-3 to +3). Drives both the signal state and the numeric contribution.
- `crsiSignal` — Named signal state (see table below).
- `crsiLength` — CRSI oscillator period (derived as half the dominant cycle length, clamped 5–50).
- `crsiSourceCycleLength` — Full cycle length from which the CRSI period was derived.
- `reasoning` — Human-readable step-by-step breakdown of how the CombinedScore was derived.

### CRSI Signal States

Signal states are named from the perspective of the **exhausting trend** — not the emerging one. The progression Exhaustion → Fatigue → Exit describes the lifecycle of momentum breakdown.

| crsiScore | Signal State | Contribution | Condition |
|-----------|-------------|-------------|-----------|
| +1 | `BullExhaustion` | +6.7 | Above upper band, still rising — bull momentum approaching exhaustion |
| -1 | `BearExhaustion` | -6.7 | Below lower band, still falling — bear momentum approaching exhaustion |
| +2 | `BullFatigue` | +13.3 | Above upper band, turning down — bull momentum fatiguing |
| -2 | `BearFatigue` | -13.3 | Below lower band, turning up — bear momentum fatiguing |
| +3 | `BullExit` | +20.0 | Crossed below upper band — bull move exiting overbought |
| -3 | `BearExit` | -20.0 | Crossed above lower band — bear move exiting oversold |
| 0 | `Neutral` | 0 | Between bands — no overbought/oversold condition |

**Divergence-confirmed variants:** When price-CRSI divergence is detected alongside a Fatigue or Exit signal, the state is upgraded:
- `BullFatigueReversalConfirmed` / `BearFatigueReversalConfirmed` (±2 with divergence)
- `BullExitReversalConfirmed` / `BearExitReversalConfirmed` (±3 with divergence)

Divergence affects the signal label only — the numeric contribution remains unchanged (pilot stage).

> **App parity:** the Cycle Scanner app's consensus gauge computes this same CRSI-integrated score (no app-side toggle exists to exclude CRSI). Score differences vs the app come from scan parameters and data window, not CRSI. See `consensus-guide.md` -> "App UI parity".

### Interpretation Guide

| Combined Score | Interpretation | Action |
|---------------|----------------|--------|
| **> +50** | Strong bullish consensus | High alert for bottoming. Look for price confirmation. |
| +30 to +50 | Moderate bullish | Cycles lean bullish. Monitor for entry opportunities. |
| -30 to +30 | Neutral / mixed | No clear cyclical bias. Trend likely to continue. |
| -30 to -50 | Moderate bearish | Cycles lean bearish. Monitor for exit or hedging. |
| **< -50** | Strong bearish consensus | High alert for topping. Look for price confirmation. |

---

## 19. CycleConsensus Calculate

**POST** `/api/CycleConsensus/calculate`

Full consensus calculation from raw close prices you supply. Returns detailed response with individual cycle contributions per phase category, CRSI metadata, divergence flags, and step-by-step reasoning.

Pipeline: CycleScanner (HP filter detrend) → Stability Scoring → Dominant Peak Detection → Consensus Calculation → CRSI Integration.

### Request Body: `ConsensusRequest`
```json
{
  "datapoints": [100.5, 101.2, 99.8, ...],
  "bartelsLimit": 10,
  "minCycleLength": 15,
  "maxCycleLength": 400,
  "savgolSmoothing": false,
  "includeCrsi": true
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| datapoints | double[] | ✅ | Array of close prices. Minimum 100 values required. |
| bartelsLimit | int | `10` | Minimum Bartels significance score (0-99) |
| minCycleLength | int | `15` | Minimum cycle length to detect |
| maxCycleLength | int | `400` | Maximum cycle length to detect (max 400) |
| savgolSmoothing | bool | `false` | Pre-processing: apply Savitzky-Golay smoothing |
| includeCrsi | bool | `true` | Include CRSI integration in the combined score |

### Response: `ConsensusResponse`
```json
{
  "bullishConsensus": 68.5,
  "bearishConsensus": 31.2,
  "combinedScore": 42.3,
  "bullishCycleCount": 5,
  "bearishCycleCount": 3,
  "crsiScore": 1,
  "crsiSignal": "BullExhaustion",
  "crsiLength": 10,
  "crsiSourceCycleLength": 40,
  "hasBearishDivergence": false,
  "hasBullishDivergence": false,
  "combinedScoreReasoning": "Bullish: 68.5 | Bearish: 31.2\n...",
  "toppingCycles": [
    { "cycleLength": 160, "minBarNum": 215, "phaseScore": -2, "strength": 2.8, "stabilityScore": 0.72, "rank": 1, "cycleQuality": 0.83, "contribution": 2.32, "reason": "..." }
  ],
  "bottomingCycles": [...],
  "fallingCycles": [...],
  "risingCycles": [...]
}
```

### CycleContribution Field Reference

Each cycle in the `toppingCycles`, `bottomingCycles`, `fallingCycles`, and `risingCycles` arrays contains:

| Field | Description |
|-------|-------------|
| `cycleLength` | Detected cycle length in bars |
| `minBarNum` | Bar index anchoring the cycle trough |
| `phaseScore` | Phase score: -2=topping, -1=falling, +1=rising, +2=bottoming |
| `strength` | Bartels-based cycle strength (0–100) |
| `stabilityScore` | Amplitude/phase consistency (0–1). 1.0 = highly stable |
| `rank` | Dominant peak rank from spectrum (1 = strongest). 0 = not ranked |
| `cycleQuality` | Composite quality = 0.6×Stability + 0.4×NormalizedRank. Drives contribution weighting |
| `contribution` | Weighted contribution to the bullish or bearish consensus percentage |
| `reason` | Human-readable explanation of how this cycle's contribution was calculated |

### Additional Response Fields (vs Score endpoint)
- `hasBearishDivergence` / `hasBullishDivergence` — Whether price-CRSI divergence was detected near the last bar (pilot — metadata only).
- `toppingCycles` / `bottomingCycles` / `fallingCycles` / `risingCycles` — Full cycle-by-cycle breakdown per phase category.

---

## 20. CycleConsensus Validate

**GET** `/api/CycleConsensus/validate/{symbol}`

Debug/validation endpoint: runs the full consensus pipeline for a symbol and returns detailed intermediate values at every stage. Useful for verifying that the pipeline produces expected results and for comparing API output against the UI.

### Path Parameter
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| symbol | string | ✅ | Symbol identifier |

### Query Parameters
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| api_key | string | ✅ | Authentication |
| barCount | int | `1150` | Number of price bars to load |
| bartelsLimit | int | `10` | Minimum Bartels significance score (0-99) |
| minCycleLength | int | `15` | Minimum cycle length to detect |
| maxCycleLength | int | `400` | Maximum cycle length to detect |

### Response: `DebugValidationResult`
```json
{
  "symbol": "SPY",
  "totalBars": 1150,
  "detectedPeaks": 12,
  "bullishConsensus": 68.5,
  "bearishConsensus": 31.2,
  "combinedScoreNoCrsi": 37.3,
  "bullishCycleCount": 5,
  "bearishCycleCount": 3,
  "crsiLength": 10,
  "crsiSourceCycleLength": 40,
  "crsiDominantSide": "Bullish",
  "crsiValue": 72.5,
  "crsiUpperBand": 70.1,
  "crsiLowerBand": 29.8,
  "crsiRawScore": 1,
  "totalDivergences": 3,
  "hasBearishDivergence": false,
  "hasBullishDivergence": false,
  "combinedScoreWithCrsi": 42.3,
  "crsiSignalState": "BullExhaustion",
  "combinedScoreReasoning": "...",
  "peaks": [
    { "cycleLength": 42, "strength": 0.88, "amplitude": 18.3, "stabilityScore": 0.75, "dominantRank": 1, "avgPhaseScore": 85, "phaseStatus": "Rising" }
  ],
  "debugLog": "=== STAGE 1: DATA LOADING ===\n..."
}
```

Stages in the debug log:
1. **Data Loading** — symbol, bar count, price range
2. **Cycle Scanner** — detected peaks with strength, stability, dominant rank, phase
3. **Consensus (without CRSI)** — bullish/bearish values, cycle contributions per phase
4. **CRSI Length** — dominant side, source cycle, derived CRSI period
5. **CRSI Score** — CRSI value vs bands, raw score, last 5 bars detail
6. **Divergence** — total divergences detected, bearish/bullish near last bar
7. **Final Score** — combined with and without CRSI, delta, signal state, full reasoning
