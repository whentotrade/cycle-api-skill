# Cycle Tools API — Request Examples

Base URL: `https://api.marketzeitgeist.com`
API Key query param: `?api_key=<key>`

---

## .NET Core / C# (HttpClient)

### SearchSymbols — find a ticker ID
```csharp
var apiKey = Environment.GetEnvironmentVariable("CYCLE_TOOLS_API_KEY")
             ?? throw new InvalidOperationException("CYCLE_TOOLS_API_KEY not set");

using var client = new HttpClient();

var url = $"https://api.marketzeitgeist.com/api/data/SearchSymbols?api_key={apiKey}&search=Apple&limit=5";
var result = await client.GetStringAsync(url);
// Returns JSON array: [{ "tickerid": "AAPL.US-D-1:FSC1", "name": "Apple Inc.", ... }]
```

### UpdateDataset — two-step orchestration (mirrors MCP server logic)
```csharp
// Step 1: EnsureCompleteDataset
var unixNow = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
var ensureUrl = $"https://api.marketzeitgeist.com/api/data/EnsureCompleteDataset" +
                $"?api_key={apiKey}&tickerId={tickerId}&unixFrom=0&unixTo={unixNow}&lastclose=true";

var request = new HttpRequestMessage(HttpMethod.Get, ensureUrl);
request.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
var response = await client.SendAsync(request);
response.EnsureSuccessStatusCode();
var json = await response.Content.ReadAsStringAsync();

using var doc = JsonDocument.Parse(json);
var root = doc.RootElement;
var isComplete = root.TryGetProperty("isComplete", out var ic) && ic.GetBoolean();
var trackingId = root.TryGetProperty("trackingId", out var tp) ? tp.GetString() : null;

if (!isComplete && !string.IsNullOrEmpty(trackingId))
{
    // Step 2: WaitUntilUpdateCompleted
    var waitUrl = $"https://api.marketzeitgeist.com/api/data/WaitUntilUpdateCompleted" +
                  $"?api_key={apiKey}&requestId={trackingId}&timeoutSeconds=30";

    var waitRequest = new HttpRequestMessage(HttpMethod.Get, waitUrl);
    waitRequest.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    var waitResponse = await client.SendAsync(waitRequest);
    waitResponse.EnsureSuccessStatusCode();
    var waitJson = await waitResponse.Content.ReadAsStringAsync();
    // { "status": true/false, "duration": 1234 }
}
// Dataset is now current — proceed to GetDatasetSeries
```

### GetDatasetSeries — load OHLCV price data
```csharp
var url = $"https://api.marketzeitgeist.com/api/data/GetDatasetSeries" +
          $"?api_key={apiKey}&tickerid=AAPL.US-D-1:FSC1&maxbars=500";

var json = await client.GetStringAsync(url);
var bars = JsonSerializer.Deserialize<OhlcvBar[]>(json);
var closes = bars.Select(b => b.Close).ToArray(); // feed into analysis endpoints
```

### Full pipeline: Search → Update → Load → Analyze
```csharp
// 1. Find ticker
var searchUrl = $"https://api.marketzeitgeist.com/api/data/SearchSymbols?api_key={apiKey}&search=AAPL";
var symbols = JsonSerializer.Deserialize<SymbolResult[]>(await client.GetStringAsync(searchUrl));
var tickerId = symbols[0].SymbolId; // field is 'symbolId', not 'tickerid' // e.g. "AAPL.US-D-1:FSC1"

// 2. Ensure data is current (two-step: EnsureCompleteDataset → WaitUntilUpdateCompleted)
var unixNow = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
var ensureUrl = $"https://api.marketzeitgeist.com/api/data/EnsureCompleteDataset?api_key={apiKey}&tickerId={tickerId}&unixFrom=0&unixTo={unixNow}&lastclose=true";
var ensureReq = new HttpRequestMessage(HttpMethod.Get, ensureUrl);
ensureReq.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
var ensureResp = await client.SendAsync(ensureReq);
var ensureJson = await ensureResp.Content.ReadAsStringAsync();
using var ensureDoc = JsonDocument.Parse(ensureJson);
var isComplete = ensureDoc.RootElement.TryGetProperty("isComplete", out var ic) && ic.GetBoolean();
var trackingId = ensureDoc.RootElement.TryGetProperty("trackingId", out var tp) ? tp.GetString() : null;
if (!isComplete && !string.IsNullOrEmpty(trackingId))
{
    var waitUrl = $"https://api.marketzeitgeist.com/api/data/WaitUntilUpdateCompleted?api_key={apiKey}&requestId={trackingId}&timeoutSeconds=30";
    var waitReq = new HttpRequestMessage(HttpMethod.Get, waitUrl);
    waitReq.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    await client.SendAsync(waitReq);
}

// 3. Load price series
var seriesUrl = $"https://api.marketzeitgeist.com/api/data/GetDatasetSeries?api_key={apiKey}&tickerid={tickerId}&maxbars=500";
var bars = JsonSerializer.Deserialize<OhlcvBar[]>(await client.GetStringAsync(seriesUrl));
var closes = bars.Select(b => b.Close).ToArray();

// 4. Run cycle analysis
var analysisUrl = $"https://api.marketzeitgeist.com/api/cycles/CycleExplorer?api_key={apiKey}";
var body = new StringContent(JsonSerializer.Serialize(closes), Encoding.UTF8, "application/json");
var cycleResult = await client.PostAsync(analysisUrl, body);
var cycle = JsonSerializer.Deserialize<DominantCycleAnalysisSet>(
    await cycleResult.Content.ReadAsStringAsync());
```

### POST endpoint (e.g. CycleExplorer with raw data)
```csharp
var url = $"https://api.marketzeitgeist.com/api/cycles/CycleExplorer" +
          $"?api_key={apiKey}&minCycleLength=20&maxCycleLength=200";

var json = JsonSerializer.Serialize(closes);
var content = new StringContent(json, Encoding.UTF8, "application/json");
var response = await client.PostAsync(url, content);
var cycle = JsonSerializer.Deserialize<DominantCycleAnalysisSet>(
    await response.Content.ReadAsStringAsync());
```

### CRSI example
```csharp
var url = $"https://api.marketzeitgeist.com/api/DSP/CRSI?api_key={apiKey}&length=30";
var content = new StringContent(JsonSerializer.Serialize(closes), Encoding.UTF8, "application/json");
var response = await client.PostAsync(url, content);
// Response: { "crsi": [...], "ub": [...], "lb": [...] }
```

### Response models
```csharp
public record DominantCycleAnalysisSet(
    string Symbol, double Length, double Amplitude,
    double Phase, string Phase_status, int Phase_score,
    double Nexttop, double Nextlow, double Lasttop, double Lastlow,
    int PhasingScore, double CycleProfitability, int Barsused, string StatusCode
);

public record OhlcvBar(
    string Date, double Open, double High, double Low, double Close, double Volume
);

public record SymbolResult(
    string TickerId, string Name, string Exchange, string Currency, string MarketType
);
```

### Error handling
```csharp
if (!response.IsSuccessStatusCode)
{
    var error = await response.Content.ReadFromJsonAsync<ProblemDetails>();
    // error.Detail contains the explanation
}
```

---

## JavaScript / TypeScript (fetch)

### SearchSymbols
```javascript
const API_KEY = process.env.CYCLE_TOOLS_API_KEY;

const res = await fetch(
  `https://api.marketzeitgeist.com/api/data/SearchSymbols?api_key=${API_KEY}&search=Apple&limit=5`
);
const symbols = await res.json();
const tickerId = symbols[0].tickerid; // e.g. "AAPL.US-D-1:FSC1"
```

### UpdateDataset (two-step orchestration)
```javascript
const JSON_HEADERS = { "Accept": "application/json", "Content-Type": "application/json" };

// Step 1: EnsureCompleteDataset
const unixNow = Math.floor(Date.now() / 1000);
const ensureRes = await fetch(
  `https://api.marketzeitgeist.com/api/data/EnsureCompleteDataset?api_key=${API_KEY}&tickerId=${tickerId}&unixFrom=0&unixTo=${unixNow}&lastclose=true`,
  { headers: JSON_HEADERS }
);
const ensure = await ensureRes.json();
// { isComplete: bool, status: string, trackingId: string|null }

if (!ensure.isComplete && ensure.trackingId) {
  // Step 2: WaitUntilUpdateCompleted
  await fetch(
    `https://api.marketzeitgeist.com/api/data/WaitUntilUpdateCompleted?api_key=${API_KEY}&requestId=${ensure.trackingId}&timeoutSeconds=30`,
    { headers: JSON_HEADERS }
  );
  // { status: true/false, duration: ms }
}
// Dataset is now current
```

### GetDatasetSeries + extract closes
```javascript
const res = await fetch(
  `https://api.marketzeitgeist.com/api/data/GetDatasetSeries?api_key=${API_KEY}&tickerid=${tickerId}&maxbars=500`
);
const bars = await res.json();
const closes = bars.map(b => b.close); // ready for analysis endpoints
```

### Full pipeline in one function
```javascript
async function analyzeSymbol(search) {
  const BASE = "https://api.marketzeitgeist.com";

  // 1. Search
  const symbols = await fetch(`${BASE}/api/data/SearchSymbols?api_key=${API_KEY}&search=${search}`)
    .then(r => r.json());
  const tickerId = symbols[0].tickerid;

  // 2. Update (two-step)
  const unixNow = Math.floor(Date.now() / 1000);
  const ensure = await fetch(`${BASE}/api/data/EnsureCompleteDataset?api_key=${API_KEY}&tickerId=${tickerId}&unixFrom=0&unixTo=${unixNow}&lastclose=true`, { headers: JSON_HEADERS }).then(r => r.json());
  if (!ensure.isComplete && ensure.trackingId) {
    await fetch(`${BASE}/api/data/WaitUntilUpdateCompleted?api_key=${API_KEY}&requestId=${ensure.trackingId}&timeoutSeconds=30`, { headers: JSON_HEADERS });
  }

  // 3. Load closes
  const bars = await fetch(`${BASE}/api/data/GetDatasetSeries?api_key=${API_KEY}&tickerid=${tickerId}&maxbars=500`)
    .then(r => r.json());
  const closes = bars.map(b => b.close);

  // 4. Analyze
  const cycle = await fetch(`${BASE}/api/cycles/CycleExplorer?api_key=${API_KEY}`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(closes)
  }).then(r => r.json());

  return cycle;
}
```

### CycleScanner with spectrum
```javascript
const result = await fetch(
  `https://api.marketzeitgeist.com/api/cycles/CycleScanner?api_key=${API_KEY}&includeSpectrum=true&sortByStrength=true`,
  { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(closes) }
).then(r => r.json());

result.peaks.forEach(p =>
  console.log(`Cycle ${p.cycleLength} bars | Bartels: ${p.bartelsValue} | ${p.phaseStatus}`)
);
```

---

## Python (requests)

```python
import requests
import os

API_KEY = os.environ["CYCLE_TOOLS_API_KEY"]
BASE = "https://api.marketzeitgeist.com"
```

### SearchSymbols
```python
symbols = requests.get(
    f"{BASE}/api/data/SearchSymbols",
    params={"api_key": API_KEY, "search": "Apple", "limit": 5}
).json()
ticker_id = symbols[0]["symbolId"]  # field is 'symbolId', not 'tickerid'  # e.g. "AAPL.US-D-1:FSC1"
```

### UpdateDataset (two-step orchestration)
```python
JSON_HEADERS = {"Accept": "application/json", "Content-Type": "application/json"}

# Step 1: EnsureCompleteDataset
import time
unix_now = int(time.time())
ensure = requests.get(
    f"{BASE}/api/data/EnsureCompleteDataset",
    params={"api_key": API_KEY, "tickerId": ticker_id, "unixFrom": 0, "unixTo": unix_now, "lastclose": "true"},
    headers=JSON_HEADERS
).json()
# { "isComplete": bool, "status": str, "trackingId": str|None }

if not ensure.get("isComplete") and ensure.get("trackingId"):
    # Step 2: WaitUntilUpdateCompleted
    requests.get(
        f"{BASE}/api/data/WaitUntilUpdateCompleted",
        params={"api_key": API_KEY, "requestId": ensure["trackingId"], "timeoutSeconds": 30},
        headers=JSON_HEADERS
    )
    # { "status": True/False, "duration": ms }
# Dataset is now current
```

### GetDatasetSeries + extract closes
```python
bars = requests.get(
    f"{BASE}/api/data/GetDatasetSeries",
    params={"api_key": API_KEY, "tickerid": ticker_id, "maxbars": 500}
).json()
closes = [b["close"] for b in bars]
```

### Full pipeline
```python
def analyze_symbol(search_term):
    # 1. Search
    symbols = requests.get(f"{BASE}/api/data/SearchSymbols",
        params={"api_key": API_KEY, "search": search_term}).json()
    ticker_id = symbols[0]["symbolId"]  # field is 'symbolId', not 'tickerid'

    # 2. Update (two-step)
    unix_now = int(time.time())
    ensure = requests.get(f"{BASE}/api/data/EnsureCompleteDataset",
        params={"api_key": API_KEY, "tickerId": ticker_id, "unixFrom": 0, "unixTo": unix_now, "lastclose": "true"},
        headers=JSON_HEADERS).json()
    if not ensure.get("isComplete") and ensure.get("trackingId"):
        requests.get(f"{BASE}/api/data/WaitUntilUpdateCompleted",
            params={"api_key": API_KEY, "requestId": ensure["trackingId"], "timeoutSeconds": 30},
            headers=JSON_HEADERS)

    # 3. Load closes
    bars = requests.get(f"{BASE}/api/data/GetDatasetSeries",
        params={"api_key": API_KEY, "tickerid": ticker_id, "maxbars": 500}).json()
    closes = [b["close"] for b in bars]

    # 4. Analyze
    return requests.post(
        f"{BASE}/api/cycles/CycleExplorer",
        params={"api_key": API_KEY},
        json=closes
    ).json()
```

### Detrend then scan
```python
detrended = requests.post(
    f"{BASE}/api/DSP/Detrend",
    params={"api_key": API_KEY, "dtype": 0},
    json=closes
).json()

cycles = requests.post(
    f"{BASE}/api/cycles/CycleScanner",
    params={"api_key": API_KEY, "humanReadableText": True},
    json=detrended
).json()
```

---

## Common Patterns

### Recommended data pipeline
```
SearchSymbols → UpdateDataset → GetDatasetSeries → [analysis endpoint]
```
Always run `SearchSymbols` first if you don't have a confirmed `tickerid`.  
Always run `UpdateDataset` before `GetDatasetSeries` when current/live data is needed.

### Chaining DSP + Analysis
Many workflows benefit from pre-processing before analysis:
1. **GetDatasetSeries** → load price data, extract `.close` array
2. **Detrend** → remove price drift, isolate cyclical component
3. **SavGol or SincSmoother** → reduce noise
4. **CycleScanner** → identify all significant cycles in the spectrum
5. **CycleExplorer** → get dominant cycle with phase and timing
6. **CRSI** → momentum oscillator tuned to that cycle length
