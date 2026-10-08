---
name: cycle-api
description: >
  How to use the Cycles IQ API (api.cyclesiq.com, cycle analysis) from an AI agent or code, and
  how to read every result. The API analyses the caller's own time series: send the values
  with each call or store them once as a dataset and name it with ?datasetid=; markets found
  with the symbol search can be analysed by id (the answer carries the cycles, not the prices). Covers cycle
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
  "store my series", or any request involving the Cycles IQ API.
---

# Cycles IQ API (cycle analysis)

Base URL: `https://api.cyclesiq.com` (the older `api.marketzeitgeist.com` still answers)
Auth: your key in the `X-API-Key` header on every request (`Authorization: Bearer <key>` works too;
`?api_key=` in the query also works but ends up in logs). Every call needs an account: there is no
access without one.
Key: sign up free at app.cyclesiq.com (a new account starts with a 30-day trial), confirm your e-mail
address and create the key on its API page (FSC members: app.cycles.org). The full key is shown once.
Live schema (authoritative for types and fields):
https://api.cyclesiq.com/specs/index.html?url=/apidocs/v1/swagger.json
Human documentation: https://marketzeitgeist.com/docs. Skill checked against the live API on 2026-09-30.

The API analyses **your own data**: you bring the values, with every call or as a stored dataset.
It can also analyse a market you find with the symbol search, by its id (see **Market data by id**);
such an answer carries the analysis and the analysed window, never the prices.

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
numbers, oldest first: at least 101 values for the cycle analyses, 11 for the DSP filters and the CRSI
(see Constraints). (Exceptions: `CycleConsensus/calculate` and `DSP/kde` take an object, see below.)

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

## Plans and limits

Every call with a key or a sign-in counts once on the account, whatever the route (REST, MCP, code);
the keys and connectors of one account share the counter. Stream updates, the apps' own calls and
`GET /api/me/*` are outside. Check `GET /api/me/limits` first: `plan`, `limits` (calls a minute, a day,
the allowance of the month or the trial and what is used, when it resets, datasets, streams, PRO),
and per endpoint group your calls today and this month.

Every successful answer outside `/api/me/*` names the call's value in tokens in the header
`X-CyclesIQ-Tokens`: the rating of its route plus one token for every 1,000 data points it works on or
part of them; ten stream updates are one token. `GET /api/me/limits` adds them up (`tokensToday`, `tokensMonth`,
`tokensMonthByChannel`). The plan limits calls; the tokens are the value of the usage: Pay as you go is
billed by the tokens of calls by API key and MCP, stream updates included (never the app's own calls), at
1.75 EUR per 1,000 tokens plus VAT where it applies. A Pay as you go account has a monthly spending limit it
sets itself on the API page of the app (50 EUR a month unless changed): at 100 % every call by key or MCP,
stream updates included, is refused with `429` and "Your spending limit for this month is reached. Raise it
on the API page, or wait until the 1st." (`Retry-After` = seconds to the 1st; a refused call costs nothing);
from 80 % every answer carries `X-CyclesIQ-Spending: 82% of 50 EUR`. `GET /api/me/limits` and `my_limits`
show it under `spending` (`limitEur`, `limitTokens`, `usedTokens`, `usedEur`, `percent`, `notice`,
`reached`, `resetsAt`); on every other plan `spending` is null.

| Plan | Who | Per minute | Per day | Allowance | PRO features | Stored datasets | Live streams |
|---|---|---|---|---|---|---|---|
| 30-day trial | every new account, 30 days from sign-up or until its 5,000 calls are used, whichever comes first | 300 | – | 5,000 calls in the trial | yes | 50 | 3 |
| Free | after the trial | 20 | 200 | 1,000 a month | no | 3 | none |
| FSC member | FSC members, from the FSC page | 60 | 500 | 2,000 a month | no | 3 | by membership |
| Pay as you go | a Cycles IQ account; booked on the API page of the app once self-service opens | 300 | 20,000 (safety cap) | none; billed per token, capped by your own spending limit (default 50 EUR a month) | yes | 50 | 50 |
| Scale | by agreement | 1,500 | 100,000 (safety cap) | none | yes | 500 | 100 |

PRO features: `dominantPeakFinder`, `CycleSpectrumPeakFinder` (`useStability` is open to every plan since 8 October
2026). The raw bars of market
data are part of no plan (the feature `MarketDataAccess`, granted on request). Read the current numbers
from `GET /api/me/limits` rather than hard-coding them.

**Answers you will see**

| Status | Meaning | What to do |
|---|---|---|
| `400` | Bad input (too few values, body and `datasetid` together, bad name). Body is a `ProblemDetails` object | Fix the request |
| `401` | No valid key or sign-in | Check the header; sign up if you have no account |
| `403` with a message | The route or feature is not part of your plan (raw bars, another user's dataset, streams) | Retrying does not help; the message says what to do |
| `404` | `datasetid` names no dataset of yours, or the window is empty | Check `GET /api/datasets` |
| `429` with `Retry-After` | A limit: the speed, the day, or the allowance of the month or the trial; the body names the plan, the limit and `upgradeUrl` | Wait the given seconds; a long `Retry-After` means the allowance, not a burst |

---

## Market data by id (REST only)

`GET /api/data/SearchSymbols?search=apple` finds markets (stocks, ETFs, crypto, forex, economic
series; no index series). Each hit's `symbolId` (ending in `:HID`) is the id: pass it as
`?datasetid=<symbolId>` to a cycle analysis (CycleScanner, CycleExplorer, CyclePowerScanner, CRSI, the
consensus calculation). The API analyses the market's last 1,250 days unless `maxbars` or `from` say
otherwise, brings the data up to date first, and names what it analysed in `analysedWindow` (`first`,
`last`, `bars`, `symbol`, `name`). The answer carries the cycles; the price fields are left out (the
feature `MarketDataAccess` opens them). Smoothing, detrending and the RSD test take a market id only
with that feature. The MCP server has no market data at all (below).

---

## Using it through MCP

The same API is an MCP server: `https://api.cyclesiq.com/mcp` (Streamable HTTP). Its 15 tools are the
analyses of **your own data** (scanner, explorer, consensus, CRSI, DSP filters, RSD test, KDE, DTW
clustering of your datasets, store/list/get your datasets, `my_limits`, `help`); a tool call is judged
and counted like a direct call. The server provides no market data and has no symbol search: take the
values from the user, a file or another connector and pass them as `datapoints`, or store them once
with `store_dataset` and name them.

- **Every call needs an account.** Clients with OAuth (Claude.ai, Claude Desktop, ChatGPT, Claude Code,
  Cursor) are asked to sign in with a Cycles IQ account; signing up is free and starts the 30-day trial.
- **With a key** instead of the sign-in:
  `claude mcp add --transport http cycles-iq https://api.cyclesiq.com/mcp --header "X-API-Key: <key>"`.
- The `help` and `my_limits` tools say what the current credential allows.

**MCP or a script?** Through MCP every value and every answer passes through the conversation: right for a
look at one series. For many series, long histories or repeated runs, a script that calls the REST routes
costs far less, because the numbers never enter the conversation. Where you can run code, offer the user
this route; the choice is theirs. The user creates a key on the app's API page and keeps it in an
environment variable (for example `CYCLESIQ_API_KEY`) or a local file the script reads and version
control ignores; the script sends it as `X-API-Key`. Never ask for the key in the chat: conversations
are stored. Same account, same limits: every call counts once, by either route.

The rest of this skill applies unchanged: same parameters, same answers, same pitfalls.

---

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/cycles/CycleScanner` | Full cycle spectrum: every significant cycle with length, strength, phase; `includeConsensus=true` adds the consensus of exactly these cycles |
| POST | `/api/cycles/CycleDetails` | One cycle on a series: its profitability, stability and highlighter for the phase you give |
| POST | `/api/cycles/CycleExplorer` | The dominant cycle in a length window, projected forward |
| POST | `/api/cycles/CycleSpectrumPeakFinder` | Rank the peaks of a spectrum you already have *(PRO)* |
| POST | `/api/CycleConsensus/calculate` | Cycle Consensus score (−100…+100) with per-cycle contributions and CRSI |
| POST | `/api/DSP/CRSI` | Cyclic Smoothed RSI with dynamic upper/lower bands |
| POST | `/api/DSP/CycleSwing` | Cycle swing (CSI): the acceleration of the dominant cycle, one value per bar |
| POST | `/api/DSP/TDSequential` | TD Sequential buy and sell setups per bar, counted to 9 or 13 |
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
| GET | `/api/data/SearchSymbols` | Find a market; its `symbolId` is the id for `?datasetid=` |
| GET | `/api/me/limits` · `/api/me/usage` | Your plan, limits and usage |
| POST | `/api/Stream/SubmitStreamData` | Live data (plans with streams) |

All analysis routes except `CycleSpectrumPeakFinder` and the three `kde` routes accept `datasetid`.
Details, parameters and when to use each: `references/endpoints.md`.

---

## Typical workflows

**What cycles are in this series?**
`CycleScanner` → sort peaks by `strength` → drop `cycleLength < 30` → look for a clear strength gap
between the leading peaks and the rest. Add `useStability=true` (every plan) and, in the trial, Pay as you go or Scale,
`dominantPeakFinder=true`; prefer peaks with `stabilityScore >= 0.5` and, where ranked, `dominantRank > 0`. With
`includeConsensus=true` the same answer carries the consensus of exactly these cycles (`consensus`, the answer of
`CycleConsensus/calculate`), so one call gives the cycles and the verdict.

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
- `stabilityScore` (0–1): **only computed with `useStability=true`** (every plan since 8 October 2026); otherwise 0.
  `dominantRank` (1 = most dominant, 0 = unranked): **only with `dominantPeakFinder=true` and a plan with the PRO
  features (the trial, Pay as you go, Scale)**; otherwise 0 and `license` says the step was skipped. Never filter on a
  field that was not computed, or every cycle disappears.
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

`combinedScore` (−100…+100, positive = bullish), `bullishConsensus` / `bearishConsensus` (the weighted
vote of each side: raw sums, not percentages), `bullishCycleCount` / `bearishCycleCount`,
`breadthFactor` (0–1, information only), `crsiScore` (−3…+3), `crsiSignal`, `crsiLength`,
`crsiSourceCycleLength`, `hasBullishDivergence` / `hasBearishDivergence`, `combinedScoreReasoning`
(step-by-step text, every number in the sign of the score), and four arrays of per-cycle contributions:
`toppingCycles`, `bottomingCycles`, `risingCycles`, `fallingCycles`. The arrays follow each cycle's
**average** phase; every entry reports both phases (`avgPhaseScore` / `avgPhaseStatus` and
`currentPhaseScore` / `currentPhaseStatus`), and the current one can already be one array further.

Score composition: cycles contribute up to ±80, CRSI up to ±20 with the opposite sign of `crsiScore`
(`−(crsiScore / 3) × 20`: overbought states lower the score, oversold states raise it).

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

- At least **101 values** for the cycle analyses (CycleScanner, CycleExplorer, CyclePowerScanner,
  CycleComposite, CycleConsensus/calculate): 100 values or fewer are refused. At least **11** for the DSP
  filters (Detrend, SavGol, SincSmoother) and the CRSI. RSDtest: more than twice its `window` (101 with
  the default window of 50).
- Cycle length windows: CycleScanner 5–400 (default 5–400); CycleExplorer 20–400 (default 30–290);
  consensus 15–400 by default.
- Bartels limit 0–99: default 49 for CycleScanner and CycleExplorer, 10 for consensus. Lower
  includes weaker cycles.
- CRSI needs about three full repetitions of the cycle it is tuned to: keep the cycle length at
  most **data length / 3**, otherwise `ub` and `lb` come back as NaN.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Filtering on `stabilityScore` without `useStability`, or on `dominantRank` without `dominantPeakFinder` and the PRO features, discards every cycle | The field is 0 when its option did not run. Rank by `strength` and `bartelsValue` instead |
| Treating `strength` as a percentage | It is a raw spectral weight; compare peaks relative to each other |
| Mixing the average and current phase fields | Use `avgPhase*` + `minBarNum` together, or `phase*` + `minBarNumCurrent` together |
| Reading direction from the phase score sign | Read the phase string (`Uptrend_Starting` = −95) |
| CRSI bands all NaN | Cycle length above data length / 3; pick a shorter cycle |
| Results differ from the Cycle Scanner app | The app detrends with HP filter (`dType=0`) and uses its own band and window; send the same settings. `dType=9` (no detrending) gives very different strengths and ranks. `dType=4` (one-sided HP) avoids end-of-series bias when the latest bars matter |
| Sending a bare array to `CycleConsensus/calculate` | It takes an object: `{"datapoints": [...], ...}` |
| Body and `?datasetid=` in the same call | Refused with 400; send one or the other |
| Retrying a `403` | It will not pass; the route or feature is outside your plan |
| Bursts of calls get `429` | Honour `Retry-After`; Free allows 20 calls a minute, the trial and Pay as you go 300 |
| `429` with "Your spending limit for this month is reached" | Pay as you go only: the account's own monthly limit is used up. Raise it on the API page of the app, or wait until the 1st (`Retry-After`); `my_limits` shows `spending` |
| A stored series gives a different result than last week | It grew. Pin the window with `from`/`to` or read `X-Dataset-First`/`X-Dataset-Last` |

---

## Building cycle applications

This skill covers the API. For charts, dashboards and complete cycle applications built on it
(composite cycle reconstruction, charting integration, phase visualization, continuous 0–100
scoring methods), see the companion **cycle-tools** skill set.
