# Cycles IQ API — Endpoint Reference

Base URL: `https://api.cyclesiq.com` · Auth: `X-API-Key: <key>` header on every call (every call needs an account).
Checked against the live Swagger document on 2026-09-30. For exact types the live schema is
authoritative: https://api.cyclesiq.com/specs/index.html?url=/apidocs/v1/swagger.json

## Contents

1. [Supplying data: body or `datasetid`](#1-supplying-data-body-or-datasetid)
2. [CycleScanner](#2-cyclescanner)
3. [CycleExplorer](#3-cycleexplorer)
4. [CycleSpectrumPeakFinder](#4-cyclespectrumpeakfinder-pro)
5. [CycleConsensus/calculate](#5-cycleconsensuscalculate)
6. [CRSI](#6-crsi)
7. [Detrend](#7-detrend)
8. [SavGol](#8-savgol)
9. [SincSmoother](#9-sincsmoother)
10. [RSDtest](#10-rsdtest)
11. [CyclePowerScanner](#11-cyclepowerscanner)
12. [KDE clustering](#12-kde-clustering)
13. [Datasets](#13-datasets)
14. [Limits and usage](#14-limits-and-usage)
15. [SubmitStreamData](#15-submitstreamdata-streaming-tiers)
16. [Status codes](#16-status-codes)
17. [SearchSymbols and market ids](#17-searchsymbols-and-market-ids)

---

## 1. Supplying data: body or `datasetid`

Every route in sections 2, 3 and 5–11 takes its series in one of two ways:

- **Body:** a JSON array of numbers, oldest first: at least 101 values for sections 2, 3, 5 and 11, at
  least 11 for sections 6 to 9, more than twice the `window` for section 10 (101 with the default)
  (`CycleConsensus/calculate`: an object with `datapoints`, see section 5).
- **`?datasetid=NAME`:** the name of one of your own datasets (stored with `PUT /api/datasets/{name}`,
  uploaded in the app, or one of your streams). Leave the body out.

Window parameters, only with `datasetid`:

| Param | Type | Meaning |
|---|---|---|
| `maxbars` | int | Only the last n bars |
| `from` | string | First bar of the window (UTC date or date-time) |
| `to` | string | Last bar of the window (UTC date or date-time) |

With `datasetid` the answer carries four headers: `X-Dataset-Id`, `X-Dataset-Bars`,
`X-Dataset-First`, `X-Dataset-Last`. Body and `datasetid` together: `400`. Unknown dataset or empty
window: `404`.

---

## 2. CycleScanner

**POST** `/api/cycles/CycleScanner`

The full cycle spectrum of a series: every cycle that passes the Bartels significance test, with
length, amplitude, strength and phase.

**Use it when** you want to know which cycles are present, compare them, or pick the dominant ones
for projection, CRSI tuning or a composite.

| Param | Type | Default | Meaning |
|---|---|---|---|
| `minCycleLength` | int | `5` | Shortest cycle to scan (min 5) |
| `maxCycleLength` | int | `400` | Longest cycle to scan (max 400) |
| `bartelsLimit` | int | `49` | Minimum Bartels score for a cycle to be listed (0–99) |
| `dType` | int | `0` | Detrending before the scan: 0 HP filter, 1 boosted HP, 2 spline, 3 polynomial, 4 one-sided HP (Kalman), 9 none |
| `epf` | bool | `false` | Endpoint flattening |
| `savgolSmoothing` | bool | `false` | Savitzky-Golay smoothing before the scan |
| `savgolSmoothingSpectrum` | bool | `false` | Smooth the spectrum after the scan |
| `sortByStrength` | bool | `true` | Sort peaks by strength (`false`: by amplitude) |
| `cycleResolution` | number | `1.0` | Spectrum step: `1.0` or `0.1` |
| `amplitudeMulti` | number | `1.0` | Multiplier for series with very small values (e.g. forex) |
| `useStability` | bool | `false` | Stability score per cycle, 0–1 *(PRO-level key)* |
| `dominantPeakFinder` | bool | `false` | Rank peaks by the shape of the spectrum *(PRO-level key)* |
| `includeSpectrum` | bool | `false` | Return the full spectrum array |
| `humanReadableText` | bool | `false` | Return plain text instead of JSON |

**Answer: `CycleScannerResults`**

| Field | Meaning |
|---|---|
| `peaks` | The cycles found (`CycleData`, below) |
| `datapoints` | Number of values analysed |
| `bartelsLimit`, `range`, `sortedBy`, `cycleStart`, `cycleEnd`, `cycleResolution`, `usedAmplitudeMulti` | Settings used |
| `spectrum` | Spectrum amplitudes (with `includeSpectrum=true`) |
| `peaksString` | Short text summary of the peaks |
| `statusCode` | `OK` or an error status |
| `license` | Notes on skipped PRO steps, e.g. "Stability scoring skipped: PRO level required" |

**`CycleData` (one peak)**

| Field | Meaning |
|---|---|
| `cycleLength` | Period in bars |
| `amplitude` | Size of the wave in the units of your values |
| `strength` | Raw spectral weight, **not a percentage**. Compare peaks with each other |
| `bartelsValue` | Statistical significance of the cycle (higher = more significant) |
| `dominantRank` | 1 = most dominant, 0 = unranked. Needs `dominantPeakFinder` and a PRO-level key, otherwise 0 |
| `stabilityScore` | 0–1, consistency of amplitude and phase over time. Needs `useStability` and a PRO-level key, otherwise 0 |
| `avgPhaseStatus`, `avgPhaseScore`, `avgPhase`, `minBarNum` | Average phase group: phase fitted across all repetitions. `minBarNum` = bar index of a trough |
| `phaseStatus`, `phaseScore`, `phase`, `minBarNumCurrent` | Current phase group: phase fitted to the recent bars only |
| `cyclesBack`, `avgPhaseTotalBars` | How much history the phase fit used |
| `enabled`, `plot` | Display flags used by the app |

Never mix the two phase groups; see `phase-guide.md`.

**Reading it:** drop cycles shorter than about 30 bars as noise; look for a clear gap in `strength`
between the leading peaks and the rest; with a PRO-level key prefer `dominantRank > 0` and
`stabilityScore >= 0.5` for projection. Without PRO, never filter on those two fields (they are 0).

---

## 3. CycleExplorer

**POST** `/api/cycles/CycleExplorer`

The single dominant cycle within a length window, with its current phase and a forward projection.

**Use it when** you need one answer: "which cycle drives this series, where is it now, and when is
the next top or low?"

| Param | Type | Default | Meaning |
|---|---|---|---|
| `minCycleLength` | int | `30` | Shortest cycle to consider (min 20) |
| `maxCycleLength` | int | `290` | Longest cycle to consider (max 400) |
| `bartelsLimit` | int | `49` | Minimum Bartels score (0–99) |
| `plotForward` | int | `100` | Bars to project the cycle into the future |
| `dynamicInSampleMethod` | bool | `false` | In-sample range based on the cycle length |
| `savgolSmoothing` | bool | `false` | Savitzky-Golay smoothing before the analysis |
| `includeTimeseries` | bool | `false` | Add bar-by-bar values (`timeSeries`) |

**Answer: `DominantCycleAnalysisSet`**

| Field | Meaning |
|---|---|
| `length`, `amplitude` | The dominant cycle |
| `phase_status`, `phase_score` | Simple phase: `Bottom`, `Rising`, `Top`, `Falling` (see `phase-guide.md` §5) |
| `phase`, `minbaroffset` | Phase position and offset of the cycle low |
| `nexttop`, `nextlow` | Bars from the last value to the next projected top / low |
| `lasttop`, `lastlow` | Bars back to the last top / low |
| `phasingScore`, `cycleProfitability` | Quality scores of the detected cycle (higher is better) |
| `barsused`, `barsAvailable`, `analysisStartDate`, `analysisEndDate`, `currentPrice` | Window and last value |
| `timeSeries` | With `includeTimeseries=true`: `{ price, smoothedPrice, date, dateUnix, dominantCycle, cycleHighlighter }` per bar |
| `statusCode`, `license` | Status and notes on skipped PRO steps |

The answer also has `stabilityScore`, `bullishConsensus`, `bearishConsensus`, `combinedScore` and
`usedDominantRank`; treat them as informational (they depend on PRO steps).

---

## 4. CycleSpectrumPeakFinder *(PRO)*

**POST** `/api/cycles/CycleSpectrumPeakFinder`

Ranks the peaks of a spectrum you already have by their visual shape. Needs a PRO-level key.
Takes no `datasetid`.

**Use it when** you ran `CycleScanner` with `includeSpectrum=true` and want the dominant peaks judged
the way an analyst reads a spectrum plot.

Body (`FindPeaksSend`):

```json
{ "spectrum": [0.1, 0.4, 1.2, ...], "cycleStart": 5, "cycleEnd": 400, "cycleResolution": 1.0 }
```

Answer (`PeaksFinderResults`): `cyclingScore`, `cyclingRating`, `peaks_dominant` (the dominant cycle
lengths), `peaks` (all peak lengths).

---

## 5. CycleConsensus/calculate

**POST** `/api/CycleConsensus/calculate`

One score from −100 (strongly bearish) to +100 (strongly bullish) that sums up where all significant
cycles in the series stand, plus a CRSI confirmation.

**Use it when** you want to know whether cycles are lining up for a turn, and in which direction.
Full interpretation: `consensus-guide.md`.

Body (`ConsensusRequest`), **an object, not a bare array**:

| Field | Type | Default | Meaning |
|---|---|---|---|
| `datapoints` | number[] | — | Closes, oldest first, at least 101 (the consensus scans with CycleScanner, which refuses 100). Leave out when you use `?datasetid=` |
| `bartelsLimit` | int | `10` | Minimum Bartels score |
| `minCycleLength` | int | `15` | Shortest cycle |
| `maxCycleLength` | int | `400` | Longest cycle (max 400) |
| `savgolSmoothing` | bool | `false` | Smoothing before the scan |
| `includeCrsi` | bool | `true` | Add the CRSI term to the score |

Pipeline inside: CycleScanner (HP filter detrending) → stability scoring → dominant peak detection →
consensus → CRSI.

Answer (`ConsensusResponse`): `combinedScore`, `bullishConsensus`, `bearishConsensus`,
`bullishCycleCount`, `bearishCycleCount`, `breadthFactor`, `crsiScore`, `crsiSignal`, `crsiLength`,
`crsiSourceCycleLength`, `hasBullishDivergence`, `hasBearishDivergence`, `combinedScoreReasoning`,
and four arrays of `CycleContributionDto`: `toppingCycles`, `bottomingCycles`, `risingCycles`,
`fallingCycles`. Field meanings: `consensus-guide.md`.

---

## 6. CRSI

**POST** `/api/DSP/CRSI`

Cyclic Smoothed RSI: an RSI tuned to a cycle, with dynamic upper and lower bands.

**Use it when** you want to know whether momentum within the current cycle is stretched
(overbought/oversold) or turning.

| Param | Type | Default | Meaning |
|---|---|---|---|
| `length` | int | `30` | Oscillator period. Use **half the dominant cycle length** |

Answer (`Crsi_indicator`): `{ "crsi": [...], "ub": [...], "lb": [...] }`, one value per input bar.
CRSI above `ub` = overbought, below `lb` = oversold.

The bands need about three repetitions of the underlying cycle. If the cycle length is more than
data length / 3, `ub` and `lb` are all NaN while `crsi` stays valid. Example: with 583 values the
longest usable cycle is 194 bars.

---

## 7. Detrend

**POST** `/api/DSP/Detrend`

Separates trend and cycle. Returns the cyclic component (default) or the trend.

**Use it when** you want to see or chart the cycles without the trend, or feed a detrended series
into your own analysis.

| Param | Type | Default | Meaning |
|---|---|---|---|
| `dtype` | int | `0` | 0 HP filter, 1 boosted HP, 2 polynomial fit, 3 cubic spline, 4 one-sided HP (Kalman), 9 none |
| `lbda` | int | `0` | HP lambda (0 = automatic) |
| `ret` | bool | `false` | `false`: cyclic component, `true`: trend component |

Answer: number array, same length as the input.

Note: the live schema numbers the Detrend types 2 = polynomial, 3 = spline, while CycleScanner's
`dType` lists 2 = spline, 3 = polynomial. If you depend on 2 or 3, check both results.

---

## 8. SavGol

**POST** `/api/DSP/SavGol`

Savitzky-Golay smoothing.

| Param | Type | Default | Meaning |
|---|---|---|---|
| `sidelength` | int | `16` | Half window; window = 2 × sidelength + 1 |
| `degree` | int | `4` | Polynomial degree |

Answer: smoothed number array, same length as the input.

---

## 9. SincSmoother

**POST** `/api/DSP/SincSmoother`

Smoothing with a modified sinc kernel (Schmid & Diebold); less ringing than Savitzky-Golay.

| Param | Type | Default | Meaning |
|---|---|---|---|
| `m` | int | `16` | Kernel half-width |
| `degree` | int | `4` | Degree |
| `isMS1` | bool | `false` | `true`: MS1 variant, `false`: MS |

Answer: smoothed number array.

---

## 10. RSDtest

**POST** `/api/DSP/RSDtest`

Running slope difference t-test: finds the bars where the trend turned (change in trend, CIT).

**Use it when** you want dated trend turns in the past, e.g. to check how well cycle tops and lows
lined up with real turns.

| Param | Type | Default | Meaning |
|---|---|---|---|
| `window` | int | `50` | Bars before and after each test point |
| `conf` | int | `80` | Confidence level in %: 68, 80, 90 or 95 |
| `denoise` | bool | `true` | Denoise before testing |
| `border` | bool | `false` | Include the series borders |
| `ftrend` | bool | `false` | Return the final trend slope |
| `ret_type` | int | `3` | 1 full, 2 CIT indices only, 3 CIT indices + result series |

Answer (`RSD_Series`): `cit_index` (bar indices of trend turns), `cit_confidence_level`,
`final_trend_slope`, `final_trend_cit_distance`, `values` (per bar: `x`, `y`, `y2`, `ts`, `cit`,
`cit_type`, `confidence_level`), `yin`, `yout`, `ts`.

---

## 11. CyclePowerScanner

**POST** `/api/DSP/CyclePowerScanner`

A second spectrum implementation. Same idea as CycleScanner, fewer options (no detrending choice,
no PRO steps). Answers with a JSON string in the shape of `CycleScannerResults`; parse it.

Params: `minCycleLength` (default 5), `maxCycleLength` (400), `bartelsLimit` (49), `amplitudeMulti`,
`cycleResolution`, `savgolSmoothing`, `savgolSmoothingSpectrum`, `sortByStrength`,
`includeSpectrum`, `humanReadableText`.

Prefer CycleScanner unless you are comparing the two.

---

## 12. KDE clustering

Groups numbers that lie close together, typically cycle lengths from several scans (e.g. 42, 43, 41
and 84, 85 form two clusters). No `datasetid`.

**Use it when** you scanned several series or windows and want the cycle lengths that recur.

- **POST** `/api/DSP/kde` with body `{ "values": [42.0, 43.5, 84.0, 41.0, 85.5], "gridSize": null, "minDensityDrop": null }`
  → `number[][]`, one array per cluster.
- **GET** `/api/DSP/kde-auto?values=42,43,84,41,85` → `number[][]`, automatic settings, range 30–400.
- **GET** `/api/DSP/kde-auto-summary?values=42,43,84,41,85` → per cluster
  `{ median, count, center, values, distinctValues }`.

---

## 13. Datasets

Your own stored series, for use with `?datasetid=`.

| Method | Path | Does |
|---|---|---|
| GET | `/api/datasets` | List your datasets and your quota |
| GET | `/api/datasets/{name}` | One dataset with its bars; optional `from`, `to` (`yyyy-MM-dd HH:mm:ssZ`), `maxbars` |
| PUT | `/api/datasets/{name}` | Create or replace. Body: `[{ "dateUnix": 1758672000, "close": 101.2 }, ...]` |
| POST | `/api/datasets/{name}/bars` | Append bars (same body). Bars before the first or after the last are added; the last bar may be corrected |
| PATCH | `/api/datasets/{name}` | Body `{ "isPrivate": true }` or `false`. Private: only you read it |
| DELETE | `/api/datasets/{name}` | Delete; frees the place when it was stored with a key |

- `dateUnix` in seconds, bars oldest first.
- Names: letters, digits, `. - _ = ^`; no colon. A name you already used is replaced by `PUT`.
- `GET /api/datasets` answer (`DatasetList`): `tier`, `storedWithKey`, `maxDatasets`,
  `maxBarsPerDataset`, `datasets`. Each dataset (`DatasetInfo`): `name`, `type`, `storedWithKey`,
  `bars`, `firstBar`, `lastBar`, `lastUpdate`, `isPrivate`.
- The quota (`maxDatasets`) counts datasets stored with a key: Free and FSC member 3, the trial and Pay as you go 50, Scale 500.

---

## 14. Limits and usage

- **GET** `/api/me/limits` → `tier`, `plan` (7-day trial, Free, FSC member, Pay as you go, Scale),
  `limits` (`perMinute`, `perDay`, `usedToday`, `allowance`, `allowanceCalls`, `allowanceUsed`,
  `allowanceResetsAt`, `pro`, `datasets`, `barsPerDataset`, `streams`, `uploads`, `counting`), and per
  endpoint group `included` (false = not in your plan: streams, uploads), `reason`, and your calls
  `today` and this `month`. Totals: `totalToday`, `totalMonth`. `trial` says when a trial ends.
  Every call counts once, whatever the route; the numbers lag live counters by about a minute.
- **GET** `/api/me/usage?days=31` (max 92) → `usedThisMonth`, `quotaMonthly`, `byGroup`,
  `byChannel`, and `days` (`day`, `calls`, `byGroup`).

Call `/api/me/limits` once at the start of a session to see what the key can do.

---

## 15. SubmitStreamData *(plans with streams)*

**POST** `/api/Stream/SubmitStreamData`

Pushes live bars into a stream dataset. Only for plans with streams (the trial 3, Pay as you go 50,
Scale 100; FSC members by their membership); others get `403`. A stream is a dataset you keep appending to; analyse it with `?datasetid=<streamid>`.

```json
{
  "streamid": "NT-BTCUSD-1M",
  "symbolname": "BTCUSD",
  "messagetype": "upsert",
  "dates": ["2020-10-08T14:05:00", "2020-10-08T14:10:00"],
  "values": [1189.55, 1180.10]
}
```

- `messagetype`: `history` (create and seed with past bars, once), `create` (create empty, once),
  `upsert` (add bars; always send the current and the previous bar).
- Dates `yyyy-MM-ddTHH:mm:ss`; `dates` and `values` of equal length.
- Each allowed stream has 300 updates a day (one every five minutes). Over budget: `429` with the
  time to wait. Do not stream intra-bar ticks.
- The number of streams depends on the plan or membership; a stream beyond it is refused.
- Stream submissions never count toward the plan's calls.

For a one-off upload without live updates use `PUT /api/datasets/{name}` instead.

---

## 16. Status codes

| Status | Meaning |
|---|---|
| `200` / `201` | OK |
| `202` | Documented for CycleScanner, CyclePowerScanner and PeakFinder; the body is a text message, read it |
| `400` | Bad input; body is `ProblemDetails` (`title`, `status`, `detail`) |
| `401` | No valid key or sign-in |
| `403` + message | Route or feature not in your plan; retrying does not help |
| `404` | Dataset not found or empty window |
| `429` + `Retry-After` | A limit of the plan: the speed, the day, or the allowance of the month or the trial. JSON body with `message`, `tier`, `plan`, `limit`, `upgradeUrl`; `Retry-After` = seconds until the counter allows calls again |

---

## 17. SearchSymbols and market ids

**GET** `/api/data/SearchSymbols?search=<ticker or name>&limit=20`

Finds markets by ticker or name (stocks, ETFs, crypto, forex, economic series; index series are not
offered). Each hit: `symbol`, `symbolId` (the market's id, ending in `:HID`), `shortName`, `exchange`,
`currency`, `type`. Pass the `symbolId` as `?datasetid=` to a cycle analysis (CycleScanner,
CycleExplorer, CyclePowerScanner, CRSI, the consensus calculation): the API brings the market up to
date, analyses its last 1,250 days unless `maxbars` or `from` say otherwise, and names the window in the
answer's `analysedWindow` (`datasetId`, `first`, `last`, `bars`, `symbol`, `name`). The price fields are
left out of the answer; the raw bars are the feature `MarketDataAccess` (granted on request), as are
market ids on the routes whose answer is the series itself (Detrend, SavGol, SincSmoother, RSDtest). Not
available through the MCP server.
