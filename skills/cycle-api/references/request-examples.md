# Request Examples

Base URL `https://api.cyclesiq.com`. Every example sends the key in the `X-API-Key` header.
Values are closes, oldest first, at least 100.

Contents: [curl](#curl) · [Python](#python) · [JavaScript](#javascript) · [C#](#c)

---

## curl

```bash
KEY=YOUR_KEY
API=https://api.cyclesiq.com

# What can this key do?
curl -H "X-API-Key: $KEY" $API/api/me/limits

# 1. Scan a series sent in the body
curl -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
     -X POST "$API/api/cycles/CycleScanner?minCycleLength=10&maxCycleLength=200" \
     -d "[101.2, 101.9, 102.4, 101.7, ...]"

# 2. Store it once ...
curl -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
     -X PUT $API/api/datasets/MYSERIES \
     -d '[{"dateUnix": 1758672000, "close": 101.2}, {"dateUnix": 1758758400, "close": 101.9}]'

# ... then analyse it by name (no body); -i shows the X-Dataset-* headers
curl -i -H "X-API-Key: $KEY" -X POST "$API/api/cycles/CycleScanner?datasetid=MYSERIES&maxbars=500"
curl -H "X-API-Key: $KEY" -X POST \
     "$API/api/cycles/CycleExplorer?datasetid=MYSERIES&minCycleLength=30&maxCycleLength=200&plotForward=50"

# Append new bars later
curl -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
     -X POST $API/api/datasets/MYSERIES/bars -d '[{"dateUnix": 1758844800, "close": 102.4}]'

# 3. Consensus: the body is an object (or use ?datasetid= and send {})
curl -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
     -X POST "$API/api/CycleConsensus/calculate?datasetid=MYSERIES" -d '{"includeCrsi": true}'

# 4. CRSI tuned to a 60-bar cycle (length = half the cycle)
curl -H "X-API-Key: $KEY" -X POST "$API/api/DSP/CRSI?datasetid=MYSERIES&length=30"
```

---

## Python

```python
import time
import requests

API = "https://api.cyclesiq.com"
S = requests.Session()
S.headers["X-API-Key"] = "YOUR_KEY"


def call(method, path, **kw):
    """One request; waits and retries on a rate limit, stops on the monthly cap.

    Both come back as 429 with Retry-After. The monthly cap waits until the 1st of next month
    (a JSON body with quotaMonthly); a rate limit waits seconds.
    """
    for attempt in range(5):
        r = S.request(method, API + path, timeout=60, **kw)
        if r.status_code != 429:
            r.raise_for_status()
            return r
        wait = float(r.headers.get("Retry-After", "1"))
        if wait > 60:
            raise RuntimeError(f"Monthly cap or long block, retry after {wait:.0f}s: {r.text}")
        time.sleep(wait * (1 + attempt))
    raise RuntimeError("Still rate limited after 5 attempts")


closes = [...]      # your values, oldest first
dates = [...]       # Unix seconds, same length

# What can this key do?
limits = call("GET", "/api/me/limits").json()
print(limits["tier"], [g["group"] for g in limits["groups"] if g["included"]])

# 1. Scan with the values in the body
scan = call("POST", "/api/cycles/CycleScanner",
            params={"minCycleLength": 10, "maxCycleLength": 200}, json=closes).json()
peaks = [p for p in scan["peaks"] if p["cycleLength"] >= 30]
peaks.sort(key=lambda p: p["strength"], reverse=True)
# Without a PRO-level key stabilityScore and dominantRank are 0: rank by strength only.
dominant = peaks[0]
print(dominant["cycleLength"], dominant["avgPhaseStatus"], dominant["avgPhaseScore"])

# 2. Store once, analyse several times by name
bars = [{"dateUnix": d, "close": c} for d, c in zip(dates, closes)]
call("PUT", "/api/datasets/MYSERIES", json=bars)

r = call("POST", "/api/cycles/CycleExplorer",
         params={"datasetid": "MYSERIES", "minCycleLength": 30,
                 "maxCycleLength": 200, "plotForward": 50})
window = {h: r.headers.get(h) for h in ("X-Dataset-Bars", "X-Dataset-First", "X-Dataset-Last")}
explorer = r.json()
print(explorer["length"], explorer["phase_status"], explorer["nexttop"], explorer["nextlow"], window)

# 3. Consensus (object body; with datasetid the datapoints come from the dataset)
consensus = call("POST", "/api/CycleConsensus/calculate",
                 params={"datasetid": "MYSERIES"}, json={"includeCrsi": True}).json()
print(consensus["combinedScore"], consensus["crsiSignal"])
print(consensus["combinedScoreReasoning"])

# 4. CRSI tuned to the dominant cycle, capped at data length / 3
cycle = min(dominant["cycleLength"], len(closes) / 3)
crsi = call("POST", "/api/DSP/CRSI",
            params={"datasetid": "MYSERIES", "length": max(5, round(cycle / 2))}).json()
last = crsi["crsi"][-1]
state = "overbought" if last > crsi["ub"][-1] else "oversold" if last < crsi["lb"][-1] else "inside bands"
print(round(last, 1), state)
```

---

## JavaScript

```javascript
const API = "https://api.cyclesiq.com";
const KEY = "YOUR_KEY";

async function call(method, path, { params, body } = {}) {
  const url = new URL(API + path);
  for (const [k, v] of Object.entries(params ?? {})) url.searchParams.set(k, v);
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(url, {
      method,
      headers: { "X-API-Key": KEY, "Content-Type": "application/json" },
      body: body === undefined ? undefined : JSON.stringify(body),
    });
    if (res.status !== 429) {
      if (!res.ok) throw new Error(`${res.status}: ${await res.text()}`);
      return res;
    }
    // Rate limit and monthly cap are both 429 + Retry-After; the cap waits until next month.
    const wait = Number(res.headers.get("Retry-After") ?? 1);
    if (wait > 60) throw new Error(`Monthly cap or long block (${wait}s): ${await res.text()}`);
    await new Promise(r => setTimeout(r, wait * 1000 * (1 + attempt)));
  }
  throw new Error("Still rate limited after 5 attempts");
}

const closes = [/* oldest first, at least 100 */];
const dates = [/* Unix seconds */];

// 1. Scan with the values in the body
const scan = await (await call("POST", "/api/cycles/CycleScanner", {
  params: { minCycleLength: 10, maxCycleLength: 200 }, body: closes,
})).json();
const peaks = scan.peaks.filter(p => p.cycleLength >= 30).sort((a, b) => b.strength - a.strength);

// 2. Store once, analyse by name
await call("PUT", "/api/datasets/MYSERIES", {
  body: dates.map((d, i) => ({ dateUnix: d, close: closes[i] })),
});
const res = await call("POST", "/api/cycles/CycleExplorer", {
  params: { datasetid: "MYSERIES", minCycleLength: 30, maxCycleLength: 200, plotForward: 50 },
});
console.log(res.headers.get("X-Dataset-Last"), await res.json());

// 3. Consensus
const consensus = await (await call("POST", "/api/CycleConsensus/calculate", {
  params: { datasetid: "MYSERIES" }, body: { includeCrsi: true },
})).json();
console.log(consensus.combinedScore, consensus.crsiSignal);
```

Browsers: the API sends `Access-Control-Allow-Origin: *`, so `fetch` works from a web page. Never
ship your key in public front-end code; call the API from your server instead.

---

## C#

```csharp
using System.Net;
using System.Net.Http.Json;
using System.Text.Json;

var http = new HttpClient { BaseAddress = new Uri("https://api.cyclesiq.com") };
http.DefaultRequestHeaders.Add("X-API-Key", "YOUR_KEY");

async Task<HttpResponseMessage> Send(Func<HttpRequestMessage> make)
{
    for (var attempt = 0; attempt < 5; attempt++)
    {
        var res = await http.SendAsync(make());
        if (res.StatusCode != HttpStatusCode.TooManyRequests)
        {
            res.EnsureSuccessStatusCode();
            return res;
        }
        // Rate limit and monthly cap are both 429 + Retry-After; the cap waits until next month.
        var wait = res.Headers.RetryAfter?.Delta ?? TimeSpan.FromSeconds(1);
        if (wait > TimeSpan.FromSeconds(60))
            throw new InvalidOperationException($"Monthly cap or long block ({wait}): " + await res.Content.ReadAsStringAsync());
        await Task.Delay(wait * (1 + attempt));
    }
    throw new InvalidOperationException("Still rate limited after 5 attempts");
}

double[] closes = /* oldest first, at least 100 */ [];
long[] dates = /* Unix seconds */ [];

// 1. Scan with the values in the body
var scanRes = await Send(() => new HttpRequestMessage(HttpMethod.Post,
    "/api/cycles/CycleScanner?minCycleLength=10&maxCycleLength=200") { Content = JsonContent.Create(closes) });
using var scan = JsonDocument.Parse(await scanRes.Content.ReadAsStringAsync());
var top = scan.RootElement.GetProperty("peaks").EnumerateArray()
    .Where(p => p.GetProperty("cycleLength").GetDouble() >= 30)
    .OrderByDescending(p => p.GetProperty("strength").GetDouble())
    .First();

// 2. Store once, analyse by name
var bars = dates.Zip(closes, (d, c) => new { dateUnix = d, close = c });
await Send(() => new HttpRequestMessage(HttpMethod.Put, "/api/datasets/MYSERIES") { Content = JsonContent.Create(bars) });

var exRes = await Send(() => new HttpRequestMessage(HttpMethod.Post,
    "/api/cycles/CycleExplorer?datasetid=MYSERIES&minCycleLength=30&maxCycleLength=200&plotForward=50"));
Console.WriteLine(exRes.Headers.GetValues("X-Dataset-Last").First());

// 3. Consensus: object body
var cRes = await Send(() => new HttpRequestMessage(HttpMethod.Post,
    "/api/CycleConsensus/calculate?datasetid=MYSERIES") { Content = JsonContent.Create(new { includeCrsi = true }) });
using var consensus = JsonDocument.Parse(await cRes.Content.ReadAsStringAsync());
Console.WriteLine(consensus.RootElement.GetProperty("combinedScore").GetDouble());
```
