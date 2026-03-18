---
layout: single
title: "Pool Enricher Deep Dive: Decoding Solscan's Binary API and the Preflight Pattern"
date: 2026-03-18
permalink: /posts/2026/03/pool-enricher-solscan-api-decoding-puppeteer-preflight/
categories:
  - blog
tags:
  - solana
  - trading
  - infrastructure
  - typescript
  - go
  - architecture
  - pool-discovery
  - market-data
excerpt: "How we fixed silent enrichment failures in the pool-discovery service: decoding Solscan's XOR-encoded binary API responses, implementing a preflight URL pattern for cross-origin session sharing, and replacing a fallback enricher with an alternating design that keeps both data sources continuously exercised."
---

## TL;DR

- **Solscan API is XOR-encoded**: responses are a 4-byte header + body XOR'd with a per-response key derived from the first byte — not plain JSON
- **Preflight URL pattern**: Solscan's API subdomain requires session cookies set by the main domain; `page.evaluate(fetch(...))` shares the same cookie jar after a preflight navigation
- **Three Puppeteer GET-path bugs fixed**: `waitForChallengeResolution` not called, `navResponse.text()` returning stale content, and missing `waitForNetworkIdle` after page transition
- **AlternatingEnricher replaces FallbackEnricher**: round-robin across Birdeye and Solscan at pool granularity — both paths continuously exercised, not just on failure

---

## Background: Silent Enrichment Failures

Pool enrichment is the step where raw pool addresses get annotated with live market data — TVL, 24h volume, reserve amounts. Without it, the pool-discovery service has no basis for ranking pools or identifying which ones are active.

Before the fixes described in this post, both enrichment sources were failing silently:

```
"Birdeye enricher failed" (every pool, every run)
"No valid reserve data from Solscan, setting liquidity to 0"  (TVL=$354k pool)
```

The second log line is the more alarming one: a pool with $354k in TVL being classified as having zero liquidity. Something was clearly wrong with the API integration itself, not just intermittent network errors.

---

## Architecture Overview

The enrichment pipeline routes all external data calls through a Puppeteer microservice ([post #26](/posts/2026/03/puppeteer-service-cloudflare-bypass-stealth-browser-microservice/)) that runs a stealth headless browser. Both Birdeye and Solscan require a real browser session to return usable data — plain HTTP clients receive 403s or HTML instead of JSON.

```
pool-discovery-service (Go)
  └── AlternatingEnricher
        ├── BirdeyeEnricher  ──┐
        └── SolscanEnricher  ──┴──► PuppeteerClient (Go)
                                         │
                                         │  POST /fetch  (JSON body)
                                         ▼
                               puppeteer-service (TypeScript)
                                  Stealth browser + residential proxy
                                         │
                               ┌─────────┴──────────┐
                               │ Birdeye GET path    │ Solscan preflight path
                               │ birdeye.so/forge/.. │ 1. GET solscan.io (session init)
                               │                     │ 2. page.evaluate(fetch(apiUrl))
                               │                     │ 3. XOR-decode binary response
                               └─────────┬──────────┘
                                         │  JSON bytes
                                    caller (Go)
```

The `POST /fetch` request body schema:

```typescript
interface FetchRequest {
  url: string;
  method?: "GET" | "POST";
  headers?: Record<string, string>;
  body?: unknown;
  preflight_url?: string; // visit this URL first to establish session cookies
}
```

The `preflight_url` field is new — it enables the two-step fetch pattern required for Solscan.

---

## Puppeteer Service: Three GET-Path Bug Fixes

### Bug 1 — `waitForChallengeResolution` was never called on GET requests

The service has a helper function that polls the page for known JS-challenge indicators (`cf-browser-verification`, `challenge-form`, etc.) and waits until they disappear. This was called in the preflight path but never in the direct GET path.

The result: when a protected API returned an HTTP 200 with a JS challenge body instead of data, the service immediately tried to parse the challenge HTML as JSON and returned an error.

**Before**:
```typescript
const navResponse = await page.goto(request.url, { waitUntil: "networkidle0", timeout: 60_000 });
// Response may still be a challenge page at this point
const bodyText = await navResponse!.text();
```

**After**:
```typescript
const navResponse = await page.goto(request.url, { waitUntil: "networkidle0", timeout: 60_000 });
await waitForChallengeResolution(page);          // ← wait for any JS challenge to resolve
await page.waitForNetworkIdle({ idleTime: 500, timeout: 30_000 }).catch(() => {});
const bodyText = await page.evaluate(() => document.body.innerText); // ← read final page content
```

### Bug 2 — `navResponse.text()` returns stale challenge content

When `page.goto()` returns a JS challenge page, and the challenge resolves by navigating to the real page, the `Response` object captured from `page.goto()` still belongs to the *initial* request. Calling `.text()` on it returns the original challenge HTML, not the real page.

The fix is to always read page content after navigation via `page.evaluate(() => document.body.innerText)` — this reads from the current DOM, whatever the browser currently shows.

### Bug 3 — Missing `waitForNetworkIdle` after challenge resolution

After a JS challenge resolves and the browser navigates to the real page, there may be background XHR requests still in flight. Reading `document.body.innerText` before those complete can produce a partially-loaded page. Adding `waitForNetworkIdle({ idleTime: 500 })` closes this race.

---

## Solscan Integration: The Preflight + Binary Decode Pattern

### The Cross-Origin Session Problem

Solscan's API lives at `api-v2.solscan.io` — a different origin from the UI at `solscan.io`. Navigating directly to the API URL starts a fresh browser session with no existing cookies. The session cookies needed to authenticate the API request are only set when the browser visits `solscan.io` itself.

The solution: use the new `preflight_url` field to visit `solscan.io` first in the same browser page, then fetch the API from inside that page's JavaScript context using `page.evaluate(fetch(...))`. Because both the preflight navigation and the `fetch()` call happen in the same browser page, they share the same cookie jar automatically.

```typescript
// puppeteer-service receives:
{
  "url": "https://api-v2.solscan.io/v2/defi/pool_info?address=...",
  "headers": { "origin": "https://solscan.io", ... },
  "preflight_url": "https://solscan.io"
}

// Service execution:
// 1. page.goto("https://solscan.io")          → establishes session cookies
// 2. waitForChallengeResolution(page)          → wait for page to fully load
// 3. page.evaluate(fetch("https://api-v2.solscan.io/..."))  → API call with shared cookies
// 4. Receive ArrayBuffer (binary response)
// 5. XOR decode → two JSON documents → merge → return clean JSON
```

The Go `PuppeteerClient` encapsulates this:

```go
func (p *PuppeteerClient) FetchSolscanPoolInfo(ctx context.Context, apiURL string) ([]byte, error) {
    fetchReq := puppeteerFetchRequest{
        URL:          apiURL,
        Method:       http.MethodGet,
        Headers:      solscanHeaders,
        PreflightURL: "https://solscan.io",
    }
    // marshals to JSON → POST /fetch → returns clean JSON bytes
}
```

### Solscan's XOR-Encoded Binary Response Format

Solscan API responses are not plain JSON. The wire format:

```
[4 bytes: opaque header] [body XOR'd with a per-response key]
```

Key derivation:

```
key = body[0] ^ 0x7b    // 0x7b is '{' — expected first byte of valid JSON
```

The key is derived from the first body byte and the expected first character of valid JSON. In practice, `0x7b` (`{`) handles object responses and `0x5b` (`[`) handles array responses. The decode logic tries both and keeps whichever produces valid JSON:

```typescript
function decodeSolscanBinary(bytes: number[]): Record<string, unknown> | null {
  const HEADER_SIZE = 4;
  const body = bytes.slice(HEADER_SIZE);

  if (body.length === 0) return null;

  // Try both possible first-byte values
  for (const expectedFirst of [0x7b, 0x5b]) {
    const key = body[0] ^ expectedFirst;
    const decoded = body.map((b) => b ^ key);
    const text = Buffer.from(decoded).toString("utf8");
    try {
      return splitAndMergeJson(text);
    } catch {
      // try next expected first byte
    }
  }
  return null;
}
```

**Important**: the XOR key is per-response, not fixed. Early debugging attempts used a hardcoded key (`0x11`) — this produced garbage JSON for any response where the first byte happened to encode differently. Always derive the key from the response itself.

### Two Concatenated JSON Documents

After XOR decoding, the body contains two back-to-back JSON objects with no separator:

```
{"total_volume_24h":...,"tokens_info":[...],"tvl":354746.12}{"tokens":{"So111...":{...}},"accounts":{"poolAddr":{...}}}
```

`JSON.parse` fails at the boundary. The fix is to split by bracket counting, then merge keys:

```typescript
function findJsonEnd(text: string, start: number): number {
  let depth = 0;
  let inString = false;
  for (let i = start; i < text.length; i++) {
    const ch = text[i];
    if (ch === '"' && text[i - 1] !== "\\") inString = !inString;
    if (inString) continue;
    if (ch === "{" || ch === "[") depth++;
    if (ch === "}" || ch === "]") {
      depth--;
      if (depth === 0) return i;
    }
  }
  return -1;
}

function splitAndMergeJson(text: string): Record<string, unknown> {
  const end1 = findJsonEnd(text, 0);
  const first = JSON.parse(text.substring(0, end1 + 1));
  const remainder = text.substring(end1 + 1).trim();
  if (!remainder) return first;
  const second = JSON.parse(remainder);
  return { ...first, ...second };
}
```

---

## Solscan Go Integration

### Flat Response Struct

The original Go struct had nested `Data` and `Metadata` sub-fields matching an old API response format. After merging the two JSON documents, the result is a flat object:

```go
type SolscanPoolResponse struct {
    TotalVolume24h       FlexibleFloat64    `json:"total_volume_24h"`
    TotalVolumeChange24h FlexibleFloat64    `json:"total_volume_change_24h"`
    TotalTrades24h       int                `json:"total_trades_24h"`
    TotalTradesChange24h FlexibleFloat64    `json:"total_trades_change_24h"`
    TokensInfo           []SolscanTokenInfo `json:"tokens_info"`
    CreateTxHash         string             `json:"create_tx_hash"`
    CreateBlockTime      int64              `json:"create_block_time"`
    TVL                  float64            `json:"tvl"`

    // From second JSON document — may be absent for some pools
    Tokens   map[string]SolscanTokenMeta   `json:"tokens"`
    Accounts map[string]SolscanAccountMeta `json:"accounts"`
}
```

`FlexibleFloat64` is a custom type that handles Solscan's inconsistent numeric encoding — some fields come as JSON numbers, others as quoted strings:

```go
type FlexibleFloat64 float64

func (f *FlexibleFloat64) UnmarshalJSON(data []byte) error {
    // Try number first
    var n float64
    if err := json.Unmarshal(data, &n); err == nil {
        *f = FlexibleFloat64(n)
        return nil
    }
    // Fall back to string
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    n, err := strconv.ParseFloat(s, 64)
    if err != nil {
        return err
    }
    *f = FlexibleFloat64(n)
    return nil
}
```

### TVL Fallback for Missing Token Decimals

Raw reserve amounts are computed from `tokens_info[].amount` multiplied by `10^decimals`, where decimals come from the `Tokens` metadata map in the second JSON document. That map is occasionally absent.

Without decimals, reserve calculation yields 0, which would incorrectly classify a live pool as inactive. The fix: if reserves are 0 but TVL is positive, the pool is demonstrably active.

```go
func (e *SolscanEnricher) EnrichPool(ctx context.Context, pool *commondomain.Pool) (*commondomain.Pool, error) {
    // ... fetch and decode ...

    // Primary path: use computed reserves
    if enrichedPool.BaseReserve > 0 && enrichedPool.QuoteReserve > 0 {
        enrichedPool.Status = commondomain.PoolStatusActive
        enrichedPool.LastUpdated = time.Now().Unix()
        return enrichedPool, nil
    }

    // TVL fallback: token decimals absent but pool is demonstrably live
    if poolMetrics.TVL > 0 {
        observability.LogInfo("Reserve decimals absent but TVL present — marking pool active", ...)
        enrichedPool.Status = commondomain.PoolStatusActive
        enrichedPool.LastUpdated = time.Now().Unix()
        return enrichedPool, nil
    }

    // Genuinely empty response
    enrichedPool.Status = commondomain.PoolStatusInactive
    return enrichedPool, nil
}
```

---

## AlternatingEnricher: Continuous Validation of Both Paths

### The Problem with Pure Fallback

A fallback design (Birdeye first, Solscan on failure) means Solscan is only exercised when Birdeye is down. Silent regressions in the Solscan integration — encoding format changes, API contract changes, new response structures — would go undetected until Birdeye became unavailable, at which point Solscan would also be broken.

### AlternatingEnricher Design

`AlternatingEnricher` round-robins each `EnrichPool` call between the two enrichers using a lock-free atomic counter. If the primary for a given call fails, the other is tried automatically.

```go
type AlternatingEnricher struct {
    enrichers [2]PoolEnricher
    counter   atomic.Uint64
}

func (e *AlternatingEnricher) EnrichPool(ctx context.Context, pool *commondomain.Pool) (*commondomain.Pool, error) {
    n := e.counter.Add(1)
    primary  := e.enrichers[n%2]
    fallback := e.enrichers[1-n%2]

    enriched, err := primary.EnrichPool(ctx, pool)
    if err == nil {
        return enriched, nil
    }

    observability.LogWarn("Enricher failed — trying alternate", ...)
    metrics.RecordEnrichmentError("enricher_alternated")
    return fallback.EnrichPool(ctx, pool)
}
```

`n%2 == 0` → Birdeye is primary for this call
`n%2 == 1` → Solscan is primary for this call

Alternation happens at pool granularity, not batch granularity — `EnrichPools` delegates to `EnrichPool` per pool, so a batch of 10 pools alternates enricher on each.

### Wiring

```go
poolEnricher := enricher.NewAlternatingEnricher(
    enricher.NewBirdeyeEnricher(5),  // 5 RPS limit
    enricher.NewSolscanEnricher(1),  // 1 RPS limit (slower path)
)
```

Both enrichers are continuously exercised in production. A regression in either will surface as errors in the metrics (`enricher_alternated` counter) rather than silently degrading to zero data.

---

## Data Flow: Before and After

```
BEFORE (Birdeye only, GET path bugs, no Solscan):
──────────────────────────────────────────────────
PuppeteerClient.FetchPoolInfo(apiURL)
   ↓
POST /fetch → puppeteer-service
   ↓
page.goto(url)          ← missing waitForChallengeResolution
navResponse.text()      ← stale content if JS challenge fired
   ↓
JSON.parse fails → "Birdeye enricher failed"


AFTER (Alternating, GET bugs fixed, Solscan XOR decode):
──────────────────────────────────────────────────────────

Birdeye path:                          Solscan path:
──────────────────                     ─────────────────────────────────
PuppeteerClient.FetchPoolInfo()        PuppeteerClient.FetchSolscanPoolInfo()
   ↓                                      ↓
POST /fetch {url}                      POST /fetch {url, preflight_url}
   ↓                                      ↓
page.goto(birdeye_url)                 page.goto("https://solscan.io")   ← preflight
waitForChallengeResolution()           waitForChallengeResolution()
waitForNetworkIdle()                   page.evaluate(fetch(api_url))      ← shared session
page.evaluate(body.innerText)          ArrayBuffer → XOR decode
   ↓                                   splitAndMergeJson()
JSON                                      ↓
                                       flat JSON

             AlternatingEnricher
             ← pools 1, 3, 5 ... → BirdeyeEnricher
             ← pools 2, 4, 6 ... → SolscanEnricher
             (either falls back to the other on error)
```

---

## Key Lessons

### `navResponse.text()` is always the initial response

After a page navigates away (due to a JS redirect), `Response.text()` still returns the body of the original navigation. Reading page content after a redirect must go through `page.evaluate(() => document.body.innerText)`, not through the captured `Response` object.

### `preflight_url` for cross-origin session sharing

APIs on a subdomain that rely on session state established by the main domain need a two-step fetch. The preflight navigation visits the main domain (in the same browser page, same cookie jar), then `page.evaluate(fetch(...))` makes the API call with those cookies already present. A new `page.goto()` to the API URL would start a fresh context with no cookies.

### Per-response XOR key derivation

Never hardcode an XOR key derived from a sample response. The key is `firstBodyByte ^ expectedFirstJsonByte` — it varies per response. The expected first byte is `{` (0x7b) for objects or `[` (0x5b) for arrays; try both and keep whichever decodes to valid JSON.

### Two concatenated JSON objects need bracket counting

`JSON.parse` rejects two adjacent JSON objects. `findJsonEnd` walks the string counting `{}`/`[]` depth and returns the index of the closing bracket for the first document. The remainder is the second document.

### TVL as pool-active signal

Raw reserve amounts require token decimal metadata to be computed correctly. When that metadata is absent (which happens for some pools), fall back to TVL > 0 as the activity signal rather than marking the pool inactive. A pool with six figures in TVL is not inactive.

---

## Impact

| Metric | Before | After |
|--------|--------|-------|
| Birdeye success rate | ~0% (JS challenge not resolved) | ~89% (normal rate under load) |
| Solscan success rate | ~0% (XOR not decoded, no session) | functional |
| $354k TVL pool status | inactive (zero reserves) | active (TVL fallback) |
| Enricher coverage | Birdeye only | Both continuously exercised |

---

## Conclusion

Three bugs in the Puppeteer GET path, a missing XOR decoder, a cross-origin session problem, and a struct that no longer matched the API — any one of these was enough to silently zero out pool enrichment data. Together they produced weeks of enrichment failures that went unnoticed because the fallback behaviour was to set TVL to 0 and continue.

The fixes are individually straightforward. The harder part was understanding why `navResponse.text()` returns the wrong content, why Solscan's API needs a preflight to a different origin, and why the XOR key varies per response. Hopefully these notes save someone else the same investigation.

The AlternatingEnricher ensures both data paths are exercised continuously so neither can regress silently again.

---

## Related Posts

- [Puppeteer Service: A Stealth Browser Microservice for Bot-Protected APIs](/posts/2026/03/puppeteer-service-cloudflare-bypass-stealth-browser-microservice/) — the original puppeteer-service post (#26)
- [Token Configuration Overhaul: Pruning 9 Dead LSTs and Adding Extra Token Pairs](/posts/2026/03/token-enabled-flag-lst-pruning-extra-pairs/) — the token config work this enrichment serves
- [Scanner Service Production Validation: 9.4M Quotes](/posts/2026/03/scanner-service-production-validation-9m-quotes-arb-signals/) — the data quality baseline

---

## Technical Documentation

- [Pool Enricher Bug Fixes — Design Doc](https://github.com/guidebee/solana-trading-system/blob/master/docs/33-POOL-ENRICHER-CLOUDFLARE-BYPASS-FIXES.md)
- [Pool Discovery Service](https://github.com/guidebee/solana-trading-system/blob/master/go/internal/pool-discovery/)
- [Puppeteer Service](https://github.com/guidebee/solana-trading-system/blob/master/ts/apps/puppeteer-service/)

---

## Connect

- **GitHub**: [guidebee/solana-trading-system](https://github.com/guidebee/solana-trading-system)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

*This is post #31 in the Solana Trading System development series. Three Puppeteer GET-path bugs, a per-response XOR decoder for Solscan's binary API format, a preflight URL pattern for cross-origin session sharing, and an AlternatingEnricher that keeps both data sources continuously validated — fixing weeks of silent pool enrichment failures.*
