---
name: cycle-api
description: >
  How to use the Cycle Analysis API (api.marketzeitgeist.com) from an AI agent or code, and
  how to read every result. The API analyses the caller's own time series: send the values
  with each call or store them once as a dataset and name it with ?datasetid=. Covers cycle
  scanning (CycleScanner), the dominant cycle with forward projection (CycleExplorer), DSP
  filters (detrend, Savitzky-Golay, sinc smoothing, CRSI), trend turning points (RSD t-test),
  cycle-length clustering (KDE), the Cycle Consensus score, stored datasets, and limits/usage.
  Use this skill whenever the user asks to run cycle analysis, scan for cycles, find the
  dominant cycle, detrend or smooth a series, compute CRSI, detect turning points, get a
  consensus score, store a series for analysis, or interpret cycle phase, strength,
  stability, or CRSI signals. Trigger on phrases like "run cycle analysis", "what is the
  dominant cycle", "scan cycles", "detrend this data", "apply CRSI", "detect turning points",
  "smooth this series", "what cycles exist in", "cycle spectrum", "consensus score",
  "bullish or bearish consensus", "cycle consensus", "CRSI signal", "cycle phase",
  "store my series", or any request involving the Cycle Analysis API.
---

# Cycle Analysis API

Base URL: `https://api.marketzeitgeist.com`
Auth: your key in the `X-API-Key` header on every request (`Authorization: Bearer <key>` works too;
`?api_key=` in the query also works but ends up in logs)
Key: create one on the API page of the app (app.marketzeitgeist.com; FSC members: app.cycles.org).
The full key is shown once.
Live schema (authoritative for types and fields):
https://api.marketzeitgeist.com/specs/index.html?url=/apidocs/v1/swagger.json

The API analyses **your own data**. It does not look up market prices for you: you bring the
values, either with every call or as a stored dataset.

| Reference | What's in it |
|---|---|
| `references/endpoints.md` | Every public endpoint: purpose, parameters, body, response, when to use it |
| `references/request-examples.md` | Working code: curl, Python, JavaScript, C# |
| `references/phase-guide.md` | Phase strings, phase scores, average vs current field groups |
| `references/consensus-guide.md` | The Cycle Consensus score: request, fields, formula, how to read it |
| `references/crsi-signals.md` | How the API derives `crsiScore` / `crsiSignal` |

---

## Two ways to supply data

**1. In the body.** Every analysis route takes the values in the request body: a plain JSON array of
numbers, oldest first, at least 100 values. (Exceptions: `CycleConsensus/calculate` and `DSP/kde` take
an object, see below.)

**2. As a stored dataset, named with `?datasetid=`.** Store the series once, then leave the body out
and name it:

```http
PUT  /api/datasets/MYSERIES          body: [{"dateUnix": 1758672000, "close": 101.2}, ...]
POST /api/cycles/CycleScanner?datasetid=MYSERIES&maxbars=500      (no body)
```

- `dateUnix` is Unix seconds, bars oldest first. Names: letters, digits, `. - _ = ^`, no colon.
- `PUT` creates or replaces; `POST /api/datasets/{name}/bars` appends newer (or older) bars and may
  correct the last one; `GET /api/datasets` lists your datasets and your quota;
  `DELETE /api/datasets/{name}` frees the place.
- `maxbars` (last n bars), `from` and `to` (UTC date or date-time) choose the window.
- The answer carries `X-Dataset-Id`, `X-Dataset-Bars`, `X-Dataset-First`, `X-Dataset-Last`. A stored
  series changes as you append, so record these headers when a result must be reproducible.
- `datasetid` also reaches datasets you uploaded in the app and your own streams.
- Body **and** `datasetid` together is refused (400).
- Each dataset has an `isPrivate` flag (see `GET /api/datasets`). Private: only you read it.
  `PATCH /api/datasets/{name}` with `{"isPrivate": true|false}` switches it.

Use the body for one-off analyses of small series. Use a stored dataset when you run several
analyses on the same series (scan, then CRSI, then consensus), or when the series grows over time.

---

## Key levels and limits

Check `GET /api/me/limits` first: it shows your tier, the limits per endpoint group, which groups
your key includes, and your calls today and this month. `GET /api/me/usage?days=31` gives the history.

| | Guest (free) | Paid tiers |
|---|---|---|
| Analysis with body or `datasetid` | yes | yes |
| Stored datasets | 3 (up to the bar size shown in `GET /api/datasets`) | more, larger |
| Cycle Consensus (`calculate`) | yes, low limits | yes |
| PRO features (`useStability`, `dominantPeakFinder`, `CycleSpectrumPeakFinder`) | no | with a PRO-level tier |
| Live streaming (`SubmitStreamData`) | no | tiers and memberships with streaming |
| Monthly cap | 2,000 key calls (streams never count) | by tier |

Guest rate limits in September 2026 were 1 call per second for the `cycles` group; read the current
numbers from `GET /api/me/limits` rather than hard-coding them.

**Answers you will see**

| Status | Meaning | What to do |
|---|---|---|
| `400` | Bad input (too few values, body and `datasetid` together, bad name). Body is a `ProblemDetails` object | Fix the request |
| `401` | No valid key | Check the header |
| `403` with a message | The route or feature is not part of your tier | Retrying does not help; the message says what to do |
| `404` | `datasetid` names no dataset of yours, or the window is empty | Check `GET /api/datasets` |
| `429` with `Retry-After` | Rate limit | Wait the given seconds, then retry |
| `429` with a quota body (`quotaMonthly`, `usedThisMonth`) | Monthly cap reached; `Retry-After` counts to the 1st of next month | Stop; a long `Retry-After` means cap, not a burst |

---

## Using it through MCP

The same API is an MCP server: `https://api.marketzeitgeist.com/mcp` (Streamable HTTP). Its tools
mirror the public routes (scanner, explorer, DSP, consensus, your datasets, limits), and a tool call
is judged and counted like a direct call.

- **No account needed to start.** Without a credential the tools run on a small free allowance per
  address (own data only, sent with the call; no stored datasets, no consensus). When it is used up,
  clients with OAuth (Claude.ai, Claude Desktop, Cursor, ChatGPT) are asked to sign in with a
  MarketZeitgeist account; `…/mcp?account=1` asks at once.
- **With a key:** send it as a header, e.g.
  `claude mcp add --transport http cycle-tools https://api.marketzeitgeist.com/mcp --header "X-API-Key: <key>"`.
- The `help` and `my_limits` tools say what the current credential allows.

The rest of this skill applies unchanged: same parameters, same answers, same pitfalls.

---

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/cycles/CycleScanner` | Full cycle spectrum: every significant cycle with length, strength, phase |
| POST | `/api/cycles/CycleExplorer` | The dominant cycle in a length window, projected forward |
| POST | `/api/cycles/CycleSpectrumPeakFinder` | Rank the peaks of a spectrum you already have *(PRO)* |
| POST | `/api/CycleConsensus/calculate` | Cycle Consensus score (−100…+100) with per-cycle contributions and CRSI |
| POST | `/api/DSP/CRSI` | Cyclic Smoothed RSI with dynamic upper/lower bands |
| POST | `/api/DSP/Detrend` | Remove the trend (HP, boosted HP, polynomial, spline, one-sided HP) |
| POST | `/api/DSP/SavGol` | Savitzky-Golay smoothing |
| POST | `/api/DSP/SincSmoother` | Modified sinc smoothing (MS / MS1) |
| POST | `/api/DSP/RSDtest` | Trend turning points (running slope difference t-test) |
| POST | `/api/DSP/CyclePowerScanner` | Cycle power spectrum, second implementation |
| POST | `/api/DSP/kde` | Cluster numbers (e.g. cycle lengths) by kernel density, custom parameters |
| GET | `/api/DSP/kde-auto` | Same, automatic parameters, range 30–400 |
| GET | `/api/DSP/kde-auto-summary` | Same, with a summary per cluster |
| GET · PUT · PATCH · DELETE | `/api/datasets`, `/api/datasets/{name}` | Your stored datasets |
| POST | `/api/datasets/{name}/bars` | Append bars |
| GET | `/api/me/limits` · `/api/me/usage` | Your limits and usage |
| POST | `/api/Stream/SubmitStreamData` | Live data (streaming tiers only) |

All analysis routes except `CycleSpectrumPeakFinder` and the three `kde` routes accept `datasetid`.
Details, parameters and when to use each: `references/endpoints.md`.

---

## Typical workflows

**What cycles are in this series?**
`CycleScanner` → sort peaks by `strength` → drop `cycleLength < 30` → look for a clear strength gap
between the leading peaks and the rest. With a PRO-level key add `useStability=true&dominantPeakFinder=true`
and prefer peaks with `dominantRank > 0` and `stabilityScore >= 0.5`.

**Where is the dominant cycle and when does it turn?**
`CycleExplorer` with a length window (`minCycleLength`, `maxCycleLength`) and `plotForward` → read
`length`, `phase_status`, `nexttop`, `nextlow`. Add `includeTimeseries=true` for bar-by-bar values.

**Is a turn building across many cycles?**
`CycleConsensus/calculate` → `combinedScore`, `bullishConsensus`, `bearishConsensus`, the four phase
arrays and `crsiSignal`. See `references/consensus-guide.md`.

**Is the momentum stretched within the cycle?**
Take the dominant cycle length L from the scan, then `DSP/CRSI?length=<L/2>` (keep L at most
data length / 3, see pitfalls). Compare the last `crsi` value with `ub` and `lb`.

**Did the trend already turn?**
`DSP/RSDtest` → `cit_index` lists the bars where the trend turned, at the chosen confidence (`conf`).

Store the series once (`PUT /api/datasets/NAME`) when you run more than one of these on it.

---

## Response essentials

### `CycleScannerResults` — CycleScanner

```json
{
  "datapoints": 500, "bartelsLimit": 49,
  "peaks": [
    { "cycleLength": 42.0, "amplitude": 18.3, "strength": 2.8, "bartelsValue": 72.4,
      "dominantRank": 1, "stabilityScore": 0.68,
      "avgPhaseStatus": "Uptrend_Neutral", "avgPhaseScore": 30, "minBarNum": 215,
      "phaseStatus": "Uptrend_Neutral", "phaseScore": 40, "minBarNumCurrent": 473 }
  ],
  "spectrum": [],
  "statusCode": "OK",
  "license": "…"
}
```

- `cycleLength`: period in bars. `amplitude`: size of the wave in price units.
- `strength`: a raw number (e.g. 2.8), **not a percentage**. Compare peaks with each other; gaps
  between groups matter more than absolute values.
- `bartelsValue`: statistical significance (higher is more significant); `bartelsLimit` filters it.
- `stabilityScore` (0–1) and `dominantRank` (1 = most dominant, 0 = unranked): **only computed with a
  PRO-level key and `useStability` / `dominantPeakFinder` set**. Otherwise both are 0 and `license`
  says the step was skipped. Never filter on them in that case, or every cycle disappears.
- Two phase groups, never mixed: average (`avgPhaseStatus`, `avgPhaseScore`, `minBarNum`) for scoring
  and regime; current (`phaseStatus`, `phaseScore`, `minBarNumCurrent`) for timing and projection.
  See **Phase strings and scores** below.
- `spectrum`: only with `includeSpectrum=true`.
- `humanReadableText=true` returns plain text instead of JSON.

Sine wave of a peak (trough at `minBarNum`, bar indices counted from the first value sent):

```
value(bar) = amplitude * sin(2π * (bar - minBarNum) / cycleLength - π/2)
```

### `DominantCycleAnalysisSet` — CycleExplorer

Key fields: `length` (bars), `amplitude`, `phase`, `phase_status` / `phase_score` (simple phase:
`Bottom`, `Rising`, `Top`, `Falling`), `nexttop` / `nextlow` (bars from the last value to the next
projected top/low), `lasttop` / `lastlow` (bars back), `phasingScore`, `cycleProfitability`,
`barsused`, `statusCode`, `license`, and with `includeTimeseries=true` a `timeSeries` array of
`{ price, smoothedPrice, date, dateUnix, dominantCycle, cycleHighlighter }`.

### `ConsensusResponse` — CycleConsensus/calculate

`combinedScore` (−100…+100), `bullishConsensus` / `bearishConsensus` (0–100), `bullishCycleCount` /
`bearishCycleCount`, `breadthFactor` (0–1, penalises consensus carried by few cycles), `crsiScore`
(−3…+3), `crsiSignal`, `crsiLength`, `crsiSourceCycleLength`, `hasBullishDivergence` /
`hasBearishDivergence`, `combinedScoreReasoning` (step-by-step text), and four arrays of per-cycle
contributions: `toppingCycles`, `bottomingCycles`, `risingCycles`, `fallingCycles`.

Score composition: cycles contribute up to ±80, CRSI up to ±20 (`(crsiScore / 3) × 20`).

### `Crsi_indicator` — CRSI

```json
{ "crsi": [45.2, 48.1], "ub": [70.0, 70.1], "lb": [30.0, 29.9] }
```

Above `ub`: overbought; below `lb`: oversold. For the discrete signal the consensus derives from
this, see `references/crsi-signals.md`.

---

## Phase strings and scores

The phase score does not sweep continuously from −100 to +100. It follows **two arcs** with sign flips:

- **Trough:** −95 (`BOTTOM_Arrival`) → −100 (`BOTTOM_Departure`) → −95 (`Uptrend_Starting`), then a jump to +30 on entering `Uptrend_Neutral`
- **Peak:** +95 (`TOP_Arrival`) → +100 (`TOP_Departure`) → +95 (`Downtrend_Starting`), then a jump to −30 on entering `Downtrend_Neutral`

| Cycle position | Phase string | Score | Type |
|---|---|---|---|
| Late downtrend | `Downtrend_ApproachingBottom` | −80 | fixed |
| At trough | `BOTTOM_Arrival` | −95 | fixed |
| Leaving trough | `BOTTOM_Departure` | −100 | fixed |
| Early uptrend | `Uptrend_Starting` | −95 | fixed |
| Mid uptrend | `Uptrend_Neutral` | 30 to 60 | range |
| Late uptrend | `Uptrend_ApproachingTop` | 80 | fixed |
| At peak | `TOP_Arrival` | 95 | fixed |
| Leaving peak | `TOP_Departure` | 100 | fixed |
| Early downtrend | `Downtrend_Starting` | 95 | fixed |
| Mid downtrend | `Downtrend_Neutral` | −30 to −60 | range |

**Read the phase string for direction.** `Uptrend_Starting` scores −95 and `Downtrend_Starting`
scores +95, so the sign alone misleads. Use the average pair or the current pair, never one field
from each. Full detail: `references/phase-guide.md`.

---

## Constraints

- At least **100 values** for every analysis route.
- Cycle length windows: CycleScanner 5–400 (default 5–400); CycleExplorer 20–400 (default 30–290);
  consensus 15–400 by default.
- Bartels limit 0–99: default 49 for CycleScanner and CycleExplorer, 10 for consensus. Lower
  includes weaker cycles.
- CRSI needs about three full repetitions of the cycle it is tuned to: keep the cycle length at
  most **data length / 3**, otherwise `ub` and `lb` come back as NaN.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Filtering on `stabilityScore` or `dominantRank` without a PRO-level key discards every cycle | Both are 0 without PRO. Rank by `strength` and `bartelsValue` instead |
| Treating `strength` as a percentage | It is a raw spectral weight; compare peaks relative to each other |
| Mixing the average and current phase fields | Use `avgPhase*` + `minBarNum` together, or `phase*` + `minBarNumCurrent` together |
| Reading direction from the phase score sign | Read the phase string (`Uptrend_Starting` = −95) |
| CRSI bands all NaN | Cycle length above data length / 3; pick a shorter cycle |
| Results differ from the Cycle Scanner app | The app detrends with HP filter (`dType=0`) and uses its own band and window; send the same settings. `dType=9` (no detrending) gives very different strengths and ranks. `dType=4` (one-sided HP) avoids end-of-series bias when the latest bars matter |
| Sending a bare array to `CycleConsensus/calculate` | It takes an object: `{"datapoints": [...], ...}` |
| Body and `?datasetid=` in the same call | Refused with 400; send one or the other |
| Retrying a `403` | It will not pass; the route or feature is outside your tier |
| Bursts of calls get `429` | Honour `Retry-After`; wait between calls (Guest: about 1 per second on `cycles`) |
| A stored series gives a different result than last week | It grew. Pin the window with `from`/`to` or read `X-Dataset-First`/`X-Dataset-Last` |

---

## Building cycle applications

This skill covers the API. For charts, dashboards and complete cycle applications built on it
(composite cycle reconstruction, charting integration, phase visualization, continuous 0–100
scoring methods), see the companion **cycle-tools** skill set.
