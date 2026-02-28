---
layout: single
title: "External Quote Arbitrage Investigation Summary"
permalink: /docs/08-external-quote-arbitrage-investigation/
author_profile: true
toc: true
toc_label: "On This Page"
---

# External Quote Arbitrage Investigation Summary

**Date**: 2026-01-24
**Status**: In Progress
**Issue**: External quotes not generating arbitrage opportunities

## Problem Statement

The scanner-service was detecting arbitrage opportunities only from LOCAL quote sources, with zero opportunities from EXTERNAL (Jupiter API) quotes. This was unexpected because the prototype system (using only Jupiter external quotes) detected many arbitrage opportunities.

## Architecture Overview

```
┌─────────────────────┐     ┌─────────────────────┐
│  local-quote-service │     │ external-quote-service│
│   (Go - SolRoute)   │     │   (Go - Jupiter API) │
└─────────┬───────────┘     └─────────┬───────────┘
          │                           │
          │ gRPC Streaming            │ gRPC Streaming
          ▼                           ▼
    ┌─────────────────────────────────────┐
    │     quote-aggregator-service        │
    │  (Combines local + external quotes) │
    └─────────────────┬───────────────────┘
                      │
                      │ gRPC Streaming (AggregatedQuote)
                      ▼
    ┌─────────────────────────────────────┐
    │       ts-scanner-service            │
    │  (Detects arbitrage opportunities)  │
    └─────────────────────────────────────┘
```

## Root Causes Identified

### 1. PairID Mismatch Bug (FIXED)

**Location**: `go/internal/external-quote-service/grpc/server.go:815`

**Problem**: The `pairid.Generate()` function includes the amount in the hash:
```go
func Generate(inputMint, outputMint string, amount uint64) string {
    // Hash includes amount, so different amounts = different pairIDs
}
```

For arbitrage detection, forward and reverse quotes must have matching pairIDs:
- **Forward quote**: SOL → USDC with subscription amount (e.g., 1,000,000,000 lamports)
- **Reverse quote**: USDC → SOL with oracle-estimated amount (different number)

This caused pairID mismatch, preventing forward/reverse quote matching.

**Fix Applied**:
```go
// BEFORE (BUG):
response := s.convertRouteAndPriceToGRPC(cachedRoute, cachedPrice, slippageBps,
    pairid.Generate(outputMint, inputMint, reverseAmount))

// AFTER (FIX):
// Use original subscription amount for pairID to ensure forward/reverse quotes match
response := s.convertRouteAndPriceToGRPC(cachedRoute, cachedPrice, slippageBps,
    pairid.Generate(outputMint, inputMint, amount))
```

### 2. Scanner Only Processing bestQuote (FIXED)

**Location**: `ts/apps/scanner-service/src/scanners/arbitrage-quote-scanner-flatbuf.ts`

**Problem**: The scanner's `process()` method only extracted `bestQuote` from the aggregated response, ignoring the separate `localQuote` and `externalQuote` fields. When local quotes "won" (had better prices), external quotes were never processed for arbitrage detection.

**Fix Applied**: Modified `process()` to handle both quotes independently:
```typescript
// Process LOCAL quote independently (if available)
if (aggregatedQuote.localQuote) {
    const localEvents = await this.processQuoteWithSource(
        aggregatedQuote.localQuote,
        'QUOTE_SOURCE_LOCAL',
        aggregatedQuote.localConfidence
    );
    events.push(...localEvents);
}

// Process EXTERNAL quote independently (if available)
if (aggregatedQuote.externalQuote) {
    const externalEvents = await this.processQuoteWithSource(
        aggregatedQuote.externalQuote,
        'QUOTE_SOURCE_EXTERNAL',
        aggregatedQuote.externalConfidence
    );
    events.push(...externalEvents);
}
```

### 3. Backpressure / Channel Full Issue (FIXED)

**Location**: `go/internal/quote-aggregator-service/server/server.go:88`

**Problem**: Output channel buffer was too small (100), causing quotes to be dropped:
```
⚠️ Output channel full, dropping quote
```

**Fix Applied**: Increased buffer size from 100 to 1000:
```go
// BEFORE:
outputChan := make(chan *aggregatorProto.AggregatedQuote, 100)

// AFTER:
outputChan := make(chan *aggregatorProto.AggregatedQuote, 1000)
```

## Remaining Issues

### 1. External Quotes Not in Aggregated Response

Despite the fixes above, the scanner is still not receiving `externalQuote` in the aggregated response. Debug logging shows:
- `hasLocal=true`
- `hasExternal=false` (unexpected)
- `hasBest=true`

The aggregator logs show it IS receiving both local and external quotes (evidenced by "Price diff" logs), but the `externalQuote` field is not being populated in the gRPC response sent to the scanner.

**Possible causes**:
1. Proto serialization issue
2. Field mapping mismatch between Go and TypeScript
3. gRPC proto-loader configuration

### 2. Docker Build Not Picking Up Changes

Debug code added to source files is not appearing in the built Docker container. This is a build pipeline issue that needs investigation.

**Symptoms**:
- Source files have debug code
- Local `dist/index.mjs` has debug code
- Container `dist/index.mjs` does NOT have debug code
- `--no-cache` flag doesn't resolve the issue

## Files Modified

| File | Change |
|------|--------|
| `go/internal/external-quote-service/grpc/server.go` | Fixed pairID for cached reverse quotes (line 815) |
| `go/internal/quote-aggregator-service/server/server.go` | Increased output channel buffer (line 88) |
| `ts/apps/scanner-service/src/scanners/arbitrage-quote-scanner-flatbuf.ts` | Process both local and external quotes independently |
| `ts/apps/scanner-service/src/clients/grpc-aggregator-client.ts` | Added debug logging (temporary) |
| `deployment/monitoring/grafana/provisioning/dashboards/ts-scanner-service-dashboard.json` | Added Quote Source Analysis panels |
| `deployment/monitoring/grafana/provisioning/dashboards/ts-strategy-service-dashboard.json` | Added Quote Source Analysis panels |

## Grafana Dashboard Panels Added

Both scanner-service and strategy-service dashboards now have "Quote Source Analysis" rows with:

1. **Local Quote Rate** - Rate of local source opportunities
2. **External Quote Rate** - Rate of external source opportunities
3. **Local Quote Ratio** - Percentage of opportunities from local source
4. **Quote Source Distribution** - Pie chart of opportunity sources
5. **Opportunities by Quote Source** - Detailed source breakdown
6. **Opportunities Rate by Source** - Time series of opportunities per source
7. **Quotes by Pair and Source** - Table view of quotes

## Next Steps

### Immediate (to resolve external quote detection)

1. **Investigate gRPC Response Population**
   - Add logging in `buildAggregatedQuote()` to verify both quotes are being set
   - Check if proto field numbers match between Go and TypeScript
   - Verify proto-loader options (`keepCase: false` converts `external_quote` → `externalQuote`)

2. **Fix Docker Build Pipeline**
   - Investigate why source changes aren't being picked up
   - Check for hidden caching (buildx cache, layer cache)
   - Consider explicit cache invalidation

3. **Add Metrics for Quote Sources**
   - Track how often external quotes are available in aggregated response
   - Track external quote latency and timeout rates

### Medium Term

1. **Prototype Comparison Testing**
   - Run prototype alongside new system with same token pairs
   - Compare opportunity detection rates
   - Validate external quote quality

2. **Performance Optimization**
   - Profile quote matching performance
   - Consider separate caches per quote source
   - Implement quote deduplication

## Key Learnings

1. **PairID Design**: When using hash-based IDs that include variable amounts, ensure both legs of a trade use consistent amounts for ID generation.

2. **Aggregation vs Independent Processing**: When combining multiple data sources, consider whether downstream consumers need access to individual sources or just the aggregated result.

3. **Backpressure Handling**: Size buffers appropriately for expected throughput. Non-blocking sends with drops are better than blocking, but drops should be monitored.

4. **Debug Logging in Production**: Consider adding configurable debug logging levels for troubleshooting without code changes.

## Reference: Quote Matching Logic

The quote-stream handler matches quotes by:
```typescript
const cacheKey = `${quote.pairId}:${quote.inputMint}:${quote.provider}`;
```

For arbitrage detection:
- Forward: SOL → USDC, pairId=X, provider=Y
- Reverse: USDC → SOL, pairId=X, provider=Y (same pairId, same provider)

Both quotes must have:
- Same `pairId` (derived from sorted mints + subscription amount)
- Same `provider` (ensures quotes from same source are matched)
- Opposite direction (forward.outputMint === reverse.inputMint)

## Reference: AggregatedQuote Proto Structure

```protobuf
message AggregatedQuote {
  Quote local_quote = 1;      // Best local (on-chain) quote
  Quote external_quote = 2;   // Best external (API) quote
  Quote best_quote = 3;       // Best quote overall
  QuoteSource best_source = 4;
  double local_confidence = 5;
  double external_confidence = 6;
  // ... additional fields
}
```
