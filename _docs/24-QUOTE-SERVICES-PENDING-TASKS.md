---
layout: single
title: "Pending Tasks - Quote Services Microservices Architecture"
permalink: /docs/24-quote-services-pending-tasks/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Pending Tasks - Quote Services Microservices Architecture

**Document Version:** 3.1 ⭐ **REORDERED BASED ON LOGICAL DEPENDENCIES**
**Last Updated:** December 31, 2025
**Status:** Active Development - Dependency-Corrected Implementation Plan
**Architecture Doc:** `30-QUOTE-SERVICE-ARCHITECTURE.md` v3.1
**Review Docs:**
- `30.1-QUOTE-SERVICE-ARCHITECTURE-REVIEW.md` (Initial architectural review)
- `30.2-SHARED-MEMORY-HYBRID-CHANGE-DETECTION.md` (Hybrid change detection)
- `30.3-REFRESH-RATE-ANALYSIS.md` (Gemini refresh rate critique response)
- `30.4-CHATGPT-REVIEW-RESPONSE.md` (ChatGPT HFT architect review)

**Key Change in v3.1**: ✅ **Task ordering corrected based on logical dependencies**
- Torn read prevention now comes AFTER shared memory implementation (was before)
- Quick wins (1s refresh, confidence algorithm) moved to Phase 0
- Rust scanner integration separated into new Phase 4
- Explicit aggregator timeouts moved to Phase 3 (where aggregator is built)

---

## 🎯 CRITICAL UPDATES FROM REVIEWS

### **ChatGPT Review Score**: 9.3/10 ⭐⭐⭐⭐⭐
**Verdict**: *"This is no longer a 'crypto bot architecture' — this is an exchange-style quoting engine"*

### **Key Changes Incorporated**:

1. ✅ **Torn Read Prevention** (Critical Correctness Issue)
   - Implement double-read verification in shared memory reader
   - **Priority**: P0 - Must implement before production

2. ✅ **Confidence Score Algorithm** (HFT Requirement)
   - Define deterministic confidence calculation (5 factors)
   - **Priority**: P0 - Required for scanner decision-making

3. ✅ **Refresh Rate Optimization** (Gemini Critique Response)
   - Phase 1: AMM 10s → 1s (10× faster, $0 cost)
   - Phase 2: CLMM 30s → 5s (event-driven, $100/mo)
   - **Priority**: P1 - Phase 1 immediate, Phase 2 after validation

4. ✅ **Non-Blocking Aggregator** (Explicit Timeout Policy)
   - Local timeout: 10ms (fast fail)
   - External timeout: 100ms (opportunistic)
   - **Priority**: P0 - Prevents tail latency amplification

5. ✅ **Split External Cache** (Route vs Price)
   - Route topology: 30s TTL (static)
   - Price data: 2s TTL (dynamic, configurable for LST = 10s)
   - **Priority**: P2 - Nice-to-have optimization

---

## 📊 Migration Status Summary

**Overall Completion**: **0% Complete** - Design complete, review-enhanced, ready for implementation ✅

**Current State**:
- ✅ Quote Service (monolithic) - 95% complete, production-ready
- ✅ Pool Discovery Service - 100% complete
- ✅ Rust RPC Proxy - 100% complete
- ❌ 3-microservice architecture - Not yet implemented
- ❌ Review enhancements - Not yet implemented

**Target State**:
- 🎯 Local Quote Service - With 1s AMM refresh, parallel paired quotes
- 🎯 External Quote Service - With split cache, parallel paired quotes
- 🎯 Quote Aggregator Service - With confidence scoring, dual shared memory, explicit timeouts
- 🎯 Shared Memory IPC - With torn read prevention, hybrid change detection

**Expected Timeline**: **5 weeks** (72-95 hours for solo developer including review enhancements)

---

## 🏗️ Enhanced Architecture Overview

### **Microservices Design (Review-Enhanced)**

```
┌─────────────────────────────────────────────────────────────────┐
│                RUST PRODUCTION SCANNERS ⭐ NEW                   │
│  • Shared memory readers (dual: internal + external)            │
│  • Hybrid change detection (<1μs reads, 200× faster)            │
│  • Torn read prevention (double-read verification)              │
│  • Confidence-based arbitrage detection                         │
└────────────────────┬────────────────────────────────────────────┘
                     ↓ Shared Memory IPC (<1μs latency)
          ┌──────────────────────────────┐
          │ Quote Aggregator Service      │  ◄── Client-facing API
          │ (Port 50051, gRPC)            │      ⭐ ENHANCED
          │ • Confidence scoring (5 factors)                        │
          │ • Dual shared memory writer                            │
          │ • Explicit timeouts (local 10ms, external 100ms)       │
          │ • Non-blocking parallel fan-out                        │
          └──────────┬──────────┬─────────┘
                     │          │
          ┌──────────┘          └──────────┐
          ▼                                 ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│ Local Quote Service      │    │ External Quote Service   │
│ (Port 50052, gRPC)       │    │ (Port 50053, gRPC)       │
│ ⭐ ENHANCED              │    │ ⭐ ENHANCED              │
│                          │    │                          │
│ • 1s AMM refresh (Phase 1)│   │ • Split cache (route/price)│
│ • 5s CLMM (Phase 2)      │    │ • Parallel paired quotes │
│ • Parallel paired quotes  │    │ • Rate limiting (1 RPS)  │
│ • Dual cache (pool+quote)│    │ • Circuit breakers       │
│ • <5ms latency           │    │ • 10s refresh (LST mode) │
└──────────────────────────┘    └──────────────────────────┘
```

---

## 📋 REORDERED MIGRATION TASKS (Based on Logical Dependencies)

### **PHASE 0: Quick Wins (In Current Monolith)** (Week 0.5, 5-6 hours) ⭐ NEW ORDER

**Goal**: Implement standalone improvements in current quote-service before microservices split

**Why First**: These tasks don't depend on microservices architecture and provide immediate value

---

#### Task 0.1: 1-Second AMM Refresh (Phase 1) ❌
**Priority**: **P1 - QUICK WIN**
**Estimated Effort**: 1 hour
**Status**: Not started
**Review Source**: Gemini critique, Doc 30.3 Phase 1
**Design Doc**: `30.3-REFRESH-RATE-ANALYSIS.md` (lines 300-360)
**Dependencies**: NONE - Simple config change

**What to Implement**:
- ❌ Change AMM refresh interval: 10s → 1s in current monolith
- ❌ Update quote service config
- ❌ Monitor Redis load (expected: +10 reads/s, negligible)
- ❌ Test with production pairs (24 hours)
- ❌ Measure opportunity capture rate improvement

**Configuration Change**:
```go
// go/internal/quote-service/service/quote_service.go or main.go

// OLD
ammRefreshInterval := 10 * time.Second

// NEW
ammRefreshInterval := 1 * time.Second  // ✅ 10× faster
```

**Expected Impact**:
- Opportunity capture: 90% → 98% (+8%)
- Latency: No change (still <5ms)
- Cost: $0 (uses existing Redis updates)

**Acceptance Criteria**:
- ✅ AMM pools refresh every 1s in current monolith
- ✅ Redis load increase <5%
- ✅ Quote freshness improved 10×
- ✅ No performance degradation

**Files to Modify**:
- `go/cmd/quote-service/main.go` (or wherever refresh interval is configured)

---

#### Task 0.2: Confidence Score Algorithm (Standalone Library) ❌
**Priority**: **P0 - CRITICAL HFT REQUIREMENT**
**Estimated Effort**: 4 hours
**Status**: Not started
**Review Source**: ChatGPT critique #3 (Critical)
**Design Doc**: `30.4-CHATGPT-REVIEW-RESPONSE.md` (lines 253-380)
**Dependencies**: NONE - Standalone algorithm library

**What to Implement**:
- ❌ Create standalone `pkg/confidence/calculator.go` package
- ❌ Implement 5-factor weighted algorithm:
  - Pool state age: 30% weight
  - Route hop count: 20% weight
  - Oracle deviation: 30% weight
  - Provider reliability: 10% weight
  - Slippage vs depth: 10% weight
- ❌ Add comprehensive unit tests
- ❌ Document algorithm in code comments

**5-Factor Confidence Algorithm**:
```go
func CalculateConfidence(quote *Quote, oracle *OraclePrice) float64 {
    // 1. Pool State Age (30%)
    ageSeconds := time.Since(quote.PoolLastUpdate).Seconds()
    poolAgeFactor := math.Max(0, 1.0 - ageSeconds/60.0)  // 60s = 0%

    // 2. Route Hop Count (20%)
    hopPenalty := float64(quote.RouteHops - 1) * 0.2
    routeFactor := math.Max(0, 1.0 - hopPenalty)

    // 3. Oracle Deviation (30%)
    quotePrice := float64(quote.OutputAmount) / float64(quote.InputAmount)
    deviation := math.Abs(quotePrice - oracle.PriceUSD) / oracle.PriceUSD
    oracleFactor := math.Max(0, 1.0 - deviation*10)  // 10% dev = 0%

    // 4. Provider Reliability (10%)
    providerFactor := GetProviderUptime(quote.Provider)  // 0.0-1.0

    // 5. Slippage vs Depth (10%)
    expectedSlippage := EstimateSlippage(quote.InputAmount, quote.Pool.Depth)
    actualSlippage := quote.PriceImpactBps / 10000.0
    slippageFactor := math.Min(1.0, expectedSlippage / math.Max(actualSlippage, 0.0001))

    // Weighted sum
    confidence := poolAgeFactor*0.30 + routeFactor*0.20 + oracleFactor*0.30 +
                  providerFactor*0.10 + slippageFactor*0.10
    return confidence
}
```

**Decision Thresholds** (for Rust scanners):
```rust
match confidence {
    0.9..=1.0  => Strategy::Execute,     // High confidence
    0.7..=0.9  => Strategy::Verify,      // Medium (re-check)
    0.5..=0.7  => Strategy::Cautious,    // Low (reduce size)
    _          => Strategy::Skip,        // Very low (ignore)
}
```

**Acceptance Criteria**:
- ✅ Confidence algorithm deterministic (same inputs = same output)
- ✅ All 5 factors contribute to final score
- ✅ Score always in range [0.0, 1.0]
- ✅ Confidence factors exposed for debugging
- ✅ Can be used standalone (no service dependencies)

**Files to Create**:
- `go/pkg/confidence/calculator.go` (NEW - standalone package)
- `go/pkg/confidence/calculator_test.go` (NEW)
- `go/pkg/confidence/types.go` (NEW - data structures)

---

### **PHASE 1: Local Quote Service** (Week 1, 15-20 hours)

**Goal**: Standalone local quote service with background pool refresh + parallel paired quotes

#### Task 1.1: Proto Definitions for Local Quote Service ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 2 hours
**Status**: Not started

**What to Create**:
- ❌ Create `proto/local_quote.proto`
- ❌ Define `LocalQuoteService` with batch streaming support:
  - `StreamBatchQuotes(BatchQuoteRequest) → stream LocalQuote` ⭐ NEW
  - `GetQuote(LocalQuoteRequest) → LocalQuote`
  - `GetPoolState(PoolStateRequest) → PoolState`
  - `Health(HealthRequest) → HealthResponse`
- ❌ Generate Go code: `go/proto/local_quote/`

**Enhanced Proto Definition** (Batch Streaming):
```protobuf
service LocalQuoteService {
  // ⭐ NEW: Batch streaming (one request, all pairs at startup)
  rpc StreamBatchQuotes(BatchQuoteRequest) returns (stream LocalQuote);

  // Legacy single-pair API
  rpc GetQuote(LocalQuoteRequest) returns (LocalQuote);
  rpc GetPoolState(PoolStateRequest) returns (PoolState);
  rpc Health(HealthRequest) returns (HealthResponse);
}

message BatchQuoteRequest {
  repeated TokenPair pairs = 1;       // All interested pairs
  repeated uint64 amounts = 2;        // All amount levels
  uint32 refresh_interval_ms = 3;     // Update frequency (default: 1000ms)
}

message TokenPair {
  string input_mint = 1;
  string output_mint = 2;
}

message LocalQuote {
  string input_mint = 1;
  string output_mint = 2;
  uint64 input_amount = 3;
  uint64 output_amount = 4;
  double price_impact = 5;
  string pool_id = 6;
  string protocol = 7;
  int64 pool_state_age_ms = 9;
  int64 quote_cache_age_ms = 10;
  bool is_stale = 11;
  double oracle_price = 13;
  double deviation_percent = 14;
  uint64 version = 15;                // ⭐ NEW: For staleness detection
}
```

**Acceptance Criteria**:
- ✅ Proto compiles without errors
- ✅ Go code generated in `go/proto/local_quote/`
- ✅ Batch streaming API supports 45 pairs × 40 amounts = 1800 quotes
- ✅ Version field added for staleness tracking

**Files to Create**:
- `proto/local_quote.proto` (NEW)
- `go/proto/local_quote/local_quote.pb.go` (GENERATED)
- `go/proto/local_quote/local_quote_grpc.pb.go` (GENERATED)

---

#### Task 1.2: Parallel Paired Quote Calculation ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 4 hours
**Status**: Not started
**Review Source**: ChatGPT praise #1 (Exceptional)
**Design Doc**: `30-QUOTE-SERVICE-ARCHITECTURE.md` Section 5.3

**What to Implement**:
- ❌ Implement `CalculatePairedQuotes()` in Local Quote Service
- ❌ Parallel goroutines for forward + reverse
- ❌ Shared pool snapshot (same logical time)
- ❌ Timeout: 100ms (fallback to single quote)
- ❌ Consistent slot calculation

**Parallel Paired Quote Pattern**:
```go
func (s *LocalQuoteService) CalculatePairedQuotes(
    inputMint, outputMint string, amount uint64,
) (*PairedQuotes, error) {
    // ✅ Take snapshot ONCE (same pool state for both)
    poolSnapshot := s.poolCache.GetSnapshot(inputMint, outputMint)

    forwardChan := make(chan *Quote, 1)
    reverseChan := make(chan *Quote, 1)
    errChan := make(chan error, 2)

    // ⭐ PARALLEL calculation with shared snapshot
    go func() {
        quote, err := s.calculator.Calculate(poolSnapshot, amount, FORWARD)
        if err != nil { errChan <- err; return }
        forwardChan <- quote
    }()

    go func() {
        quote, err := s.calculator.Calculate(poolSnapshot, amount, REVERSE)
        if err != nil { errChan <- err; return }
        reverseChan <- quote
    }()

    // Wait for both (with timeout)
    timeout := time.After(100 * time.Millisecond)
    var forward, reverse *Quote

    for i := 0; i < 2; i++ {
        select {
        case forward = <-forwardChan:
        case reverse = <-reverseChan:
        case err := <-errChan:
            log.Warn("Paired quote failed", "error", err)
        case <-timeout:
            return nil, errors.New("paired quote timeout")
        }
    }

    return &PairedQuotes{Forward: forward, Reverse: reverse}, nil
}
```

**Why This Matters**:
- Sequential: 50ms + 50ms = 100ms (slot drift risk)
- Parallel: max(50ms, 50ms) = 50-60ms (same slot)
- Eliminates fake arbitrage from slot drift

**Acceptance Criteria**:
- ✅ Forward + reverse use same pool snapshot
- ✅ Parallel execution (2× faster)
- ✅ Timeout enforced (100ms)
- ✅ No fake arbitrage from slot drift

**Files to Create**:
- `go/internal/local-quote-service/calculator/paired_calculator.go` (NEW)
- `go/internal/local-quote-service/calculator/paired_calculator_test.go` (NEW)

---

#### Task 1.3: Background Pool Refresh Manager (Enhanced) ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 4 hours
**Status**: Not started
**Enhancement**: 1s AMM refresh (was 10s)

**What to Create**:
- ❌ Implement `internal/local-quote-service/refresh/manager.go`
- ❌ **Dual cache architecture** (pool + quote):
  - Layer 1: Pool State Cache (AMM: **1s** ⭐ CHANGED, CLMM: 30s)
  - Layer 2: Quote Response Cache (2s TTL)
- ❌ **Pool-aware cache invalidation**
- ❌ **Background schedulers**:
  - AMM pools: **1s** interval ⭐ CHANGED (was 10s)
  - CLMM pools: 30s interval (Phase 2: event-driven)
  - Staleness monitor: 5s interval
- ❌ **Priority queue**: On-demand refresh
- ❌ Prometheus metrics

**Configuration**:
```bash
# Environment variables
AMM_REFRESH_INTERVAL=1s      # ⭐ CHANGED from 10s
CLMM_REFRESH_INTERVAL=30s    # Phase 2: event-driven
POOL_CACHE_STALENESS_THRESHOLD=60s
QUOTE_CACHE_TTL=2s
```

**Acceptance Criteria**:
- ✅ AMM pools refresh every **1s** (10× faster)
- ✅ CLMM pools refresh every 30s
- ✅ Pool refresh triggers quote cache invalidation
- ✅ Staleness detection working
- ✅ Metrics show cache hit rates >90%

**Files to Create**:
- `go/internal/local-quote-service/refresh/manager.go` (NEW)
- `go/internal/local-quote-service/cache/pool_state_cache.go` (NEW)
- `go/internal/local-quote-service/cache/quote_response_cache.go` (NEW)

---

#### Task 1.4: Local Quote Service Tests (Enhanced) ❌
**Priority**: **P1 - CRITICAL**
**Estimated Effort**: 4 hours (was 3h, +1h for new tests)
**Status**: Not started

**Additional Test Coverage** (from reviews):
- ✅ Parallel paired quotes (forward + reverse)
- ✅ 1s AMM refresh rate
- ✅ Batch streaming API
- ✅ Quote versioning
- ✅ Pool-aware cache invalidation

**Test Cases**:
1. **Parallel Paired Quotes**:
   - Input: SOL/USDC pair, 1 SOL
   - Expected: Both quotes use same pool snapshot
   - Assertions: Forward + reverse calculated in <60ms

2. **1s AMM Refresh**:
   - Input: AMM pool, monitor for 5 seconds
   - Expected: 5 refresh cycles
   - Assertions: Refresh every 1s ± 100ms

3. **Batch Streaming**:
   - Input: 45 pairs × 40 amounts
   - Expected: Stream emits 1800 quotes
   - Assertions: All quotes emitted within 5s

**Files to Create**:
- `go/internal/local-quote-service/calculator/paired_calculator_test.go` (NEW)
- `go/internal/local-quote-service/refresh/manager_test.go` (ENHANCED)
- `go/internal/local-quote-service/server/batch_streaming_test.go` (NEW)

---

### **PHASE 2: External Quote Service** (Week 2, 14-17 hours)

**Goal**: Standalone external quote service with split cache + parallel paired quotes

#### Task 2.1: Split Cache Strategy (Route vs Price) ❌
**Priority**: **P2 - OPTIMIZATION**
**Estimated Effort**: 3 hours
**Status**: Not started
**Review Source**: ChatGPT critique #5 (Optional)
**Design Doc**: `30.4-CHATGPT-REVIEW-RESPONSE.md` (lines 382-476)

**What to Implement**:
- ❌ Implement dual cache in External Quote Service:
  - `routeCache`: 30s TTL (route topology, DEX hops)
  - `priceCache`: 2s TTL for arb, **10s for LST** (configurable)
- ❌ Partial refresh: Fetch price only if route cached
- ❌ Configuration: `EXTERNAL_PRICE_CACHE_TTL` (2s or 10s)

**Split Cache Architecture**:
```go
type ExternalQuoteCache struct {
    // Cache 1: Route topology (30s TTL)
    routeCache map[string]*RouteTopology
    routeTTL   time.Duration  // 30s

    // Cache 2: Price data (configurable TTL)
    priceCache map[string]*PriceData
    priceTTL   time.Duration  // 2s (arb) or 10s (LST)
}

type RouteTopology struct {
    RouteSteps    []RouteStep  // DEX hops (rarely changes)
    PoolAddresses []string
    LastUpdate    time.Time
}

type PriceData struct {
    OutputAmount     uint64    // Changes frequently
    PriceImpactBps   uint32
    OraclePriceUSD   float64
    LastUpdate       time.Time
}
```

**Configuration**:
```bash
# For arbitrage (major pairs)
EXTERNAL_PRICE_CACHE_TTL=2s

# For LST arbitrage (our use case)
EXTERNAL_PRICE_CACHE_TTL=10s  # Default
```

**Benefits**:
- Route topology cached 30s (saves bandwidth)
- Price-only refresh when route cached
- Configurable freshness for different strategies

**Acceptance Criteria**:
- ✅ Route cache works (30s TTL)
- ✅ Price cache works (2s or 10s configurable)
- ✅ Partial refresh fetches price only
- ✅ Bandwidth savings measurable

**Files to Create**:
- `go/internal/external-quote-service/cache/split_cache.go` (NEW)
- `go/internal/external-quote-service/cache/split_cache_test.go` (NEW)

---

#### Task 2.2: Parallel Paired Quotes (External) ❌
**Priority**: **P1 - CRITICAL**
**Estimated Effort**: 3 hours
**Status**: Not started
**Review Source**: Architectural review #2 (Critical enhancement)
**Design Doc**: `30.1-QUOTE-SERVICE-ARCHITECTURE-REVIEW.md`

**What to Implement**:
- ❌ Extend parallel paired quotes to External Quote Service
- ❌ Pre-check rate limit tokens before launching goroutines
- ❌ Shared API response (same external quote for forward + reverse)
- ❌ Timeout: 500ms (external API latency)

**Parallel External Quotes with Rate Limit**:
```go
func (s *ExternalQuoteService) CalculatePairedQuotes(
    inputMint, outputMint string, amount uint64,
) (*PairedQuotes, error) {
    // ✅ Pre-check: Do we have 2 rate limit tokens?
    if !s.rateLimiter.Reserve(2) {
        return nil, errors.New("rate limit exceeded")
    }

    // ⭐ PARALLEL calculation (both use same API response)
    forwardChan := make(chan *Quote, 1)
    reverseChan := make(chan *Quote, 1)

    go func() {
        quote, err := s.fetchExternalQuote(inputMint, outputMint, amount, FORWARD)
        if err == nil { forwardChan <- quote }
    }()

    go func() {
        quote, err := s.fetchExternalQuote(inputMint, outputMint, amount, REVERSE)
        if err == nil { reverseChan <- quote }
    }()

    // Wait with timeout
    timeout := time.After(500 * time.Millisecond)
    // ... (similar to local paired quotes)
}
```

**Acceptance Criteria**:
- ✅ Forward + reverse calculated in parallel
- ✅ Rate limit tokens checked before launch
- ✅ Timeout enforced (500ms)
- ✅ Both quotes use same API response

**Files to Create**:
- `go/internal/external-quote-service/quoters/paired_quoter.go` (NEW)
- `go/internal/external-quote-service/quoters/paired_quoter_test.go` (NEW)

---

### **PHASE 3: Quote Aggregator Service** (Week 3, 20-25 hours)

**Goal**: Client-facing aggregator with confidence scoring + dual shared memory

#### Task 3.1: Proto Definitions for Quote Aggregator ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 2 hours
**Status**: Not started
**Dependencies**: Phase 1 & 2 proto definitions

**What to Create**:
- ❌ Create `proto/quote_aggregator.proto`
- ❌ Define `AggregatorService` with streaming support
- ❌ Add confidence score fields to `AggregatedQuote`
- ❌ Generate Go code: `go/proto/quote_aggregator/`

**Files to Create**:
- `proto/quote_aggregator.proto` (NEW)
- `go/proto/quote_aggregator/quote_aggregator.pb.go` (GENERATED)

---

#### Task 3.2: Dual Shared Memory Writer ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 5 hours
**Status**: Not started
**Review Source**: Architectural review #4 (Critical)
**Design Doc**: `30-QUOTE-SERVICE-ARCHITECTURE.md` Section 4.2

**What to Implement**:
- ❌ Write to TWO shared memory files:
  - `quotes-internal.mmap` (local quotes)
  - `quotes-external.mmap` (external quotes)
- ❌ Implement atomic versioning (odd = writing, even = readable)
- ❌ Ring buffer change notification (512 slots)
- ❌ Hybrid change detection

**Dual Shared Memory Writer**:
```go
type SharedMemoryWriter struct {
    internalFile *os.File
    externalFile *os.File

    internalQuotes []QuoteMetadata  // 2000 entries
    externalQuotes []QuoteMetadata  // 2000 entries

    changeNotification *ChangeNotification
    changedPairs       []ChangedPairNotification  // 512 slots
}

func (w *SharedMemoryWriter) WriteQuote(
    pairIndex uint32,
    localQuote *LocalQuote,
    externalQuote *ExternalQuote,
) {
    // Write to internal memory
    if localQuote != nil {
        w.writeInternal(pairIndex, localQuote)
        w.notifyChange(pairIndex, localQuote.Version)
    }

    // Write to external memory
    if externalQuote != nil {
        w.writeExternal(pairIndex, externalQuote)
        w.notifyChange(pairIndex, externalQuote.Version)
    }
}

func (w *SharedMemoryWriter) writeInternal(idx uint32, quote *LocalQuote) {
    quotePtr := &w.internalQuotes[idx]

    // Step 1: Mark as writing (odd version)
    version := quotePtr.Version.Add(1)

    // Step 2: Write struct
    *quotePtr = convertToMetadata(quote)

    // Step 3: Commit (even version)
    quotePtr.Version.Add(1)
}
```

**Memory Layout**:
```
/var/quote-service/quotes-internal.mmap:
├─ Change Notification Header (64 bytes)
├─ Ring Buffer (32,768 bytes)
└─ Quote Metadata (256,000 bytes)
Total: 282 KB

/var/quote-service/quotes-external.mmap:
├─ Change Notification Header (64 bytes)
├─ Ring Buffer (32,768 bytes)
└─ Quote Metadata (256,000 bytes)
Total: 282 KB

Grand Total: 564 KB (fits in L2 cache on modern CPUs)
```

**Acceptance Criteria**:
- ✅ Two shared memory files created
- ✅ Atomic versioning works (odd/even)
- ✅ Ring buffer notifications work
- ✅ Rust scanner can read both files

**Files to Create**:
- `go/internal/quote-aggregator-service/shared_memory/writer.go` (NEW)
- `go/internal/quote-aggregator-service/shared_memory/writer_test.go` (NEW)

---

#### Task 3.3: Explicit Aggregator Timeouts ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 2 hours
**Status**: Not started
**Review Source**: ChatGPT critique #2 (Partially valid)
**Design Doc**: `30.4-CHATGPT-REVIEW-RESPONSE.md` (lines 127-234)
**Dependencies**: Phase 1 & 2 services must be implemented

**What to Implement**:
- ❌ Add explicit timeout constants
- ❌ Local quote timeout: 10ms (fast fail)
- ❌ External quote timeout: 100ms (opportunistic)
- ❌ Emit local-only result immediately
- ❌ Update with external later (if available)
- ❌ Add timeout metrics

**Non-Blocking Aggregator Pattern**:
```go
const (
    LocalQuoteTimeout    = 10 * time.Millisecond   // Fast fail
    ExternalQuoteTimeout = 100 * time.Millisecond  // Opportunistic
)

func (s *AggregatorService) StreamQuotes(req, stream) error {
    localChan := make(chan *LocalQuote, 1)
    externalChan := make(chan *ExternalQuote, 1)

    // Launch with EXPLICIT timeouts
    go func() {
        ctx, cancel := context.WithTimeout(ctx, LocalQuoteTimeout)
        defer cancel()
        if quote, err := s.localClient.GetQuote(ctx, req); err == nil {
            localChan <- quote
        }
    }()

    // ⭐ EMIT LOCAL-ONLY IMMEDIATELY
    firstEmit := false
    for {
        select {
        case local := <-localChan:
            if !firstEmit {
                stream.Send(AggregatedQuote{
                    BestLocal:  local,
                    BestSource: LOCAL,
                })
                firstEmit = true
            }
        case external := <-externalChan:
            stream.Send(AggregatedQuote{
                BestLocal:    bestLocal,
                BestExternal: external,
                BestSource:   selectBest(bestLocal, external),
            })
        }
    }
}
```

**Acceptance Criteria**:
- ✅ Local timeout enforced (10ms)
- ✅ External timeout enforced (100ms)
- ✅ First emit uses local-only (<10ms)
- ✅ External never blocks local path
- ✅ Metrics track timeout occurrences

**Files to Modify**:
- `go/internal/quote-aggregator-service/aggregator/merger.go` (MODIFY)
- `go/internal/quote-aggregator-service/server/grpc_server.go` (MODIFY)

---

#### Task 3.4: Confidence Score Integration ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 3 hours
**Status**: Not started
**Dependencies**: Task 0.2 (Confidence Score Algorithm)

**What to Integrate**:
- ❌ Import `pkg/confidence` package in aggregator
- ❌ Call `ConfidenceCalculator` in aggregator merge logic
- ❌ Add confidence score to `AggregatedQuote` response
- ❌ Add confidence factors for debugging
- ❌ Add Prometheus metrics for confidence distribution

**Integration in Aggregator**:
```go
func (a *QuoteAggregator) mergeQuotes(
    local *LocalQuote,
    external *ExternalQuote,
) *AggregatedQuote {
    // Calculate confidence for both quotes
    localConfidence := 0.0
    externalConfidence := 0.0

    if local != nil {
        localConfidence = a.confidenceCalc.Calculate(local, oracle)
    }
    if external != nil {
        externalConfidence = a.confidenceCalc.Calculate(external, oracle)
    }

    // Select best based on confidence (not just output amount)
    var bestSource QuoteSource
    if localConfidence > externalConfidence {
        bestSource = QuoteSource_LOCAL
    } else {
        bestSource = QuoteSource_EXTERNAL
    }

    return &AggregatedQuote{
        LocalQuote:        local,
        ExternalQuote:     external,
        BestSource:        bestSource,
        LocalConfidence:   localConfidence,
        ExternalConfidence: externalConfidence,
        // ... other fields
    }
}
```

**Acceptance Criteria**:
- ✅ Confidence calculated for both quotes
- ✅ Best quote selected by confidence (not just amount)
- ✅ Confidence exposed in gRPC response
- ✅ Prometheus metrics track confidence distribution

**Files to Modify**:
- `go/internal/quote-aggregator-service/aggregator/merger.go` (MODIFY)
- `proto/quote_aggregator.proto` (MODIFY - add confidence fields)

---

### **PHASE 4: Rust Scanner Integration** (Week 4, 12-15 hours) ⭐ NEW PHASE

**Goal**: Rust production scanners with shared memory IPC

**Why This Phase**: Shared memory must exist (Task 3.2) before Rust scanners can read from it

---

#### Task 4.1: Rust Shared Memory Reader (Basic) ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 4 hours
**Status**: Not started
**Dependencies**: Task 3.2 (Dual Shared Memory Writer must exist)

**What to Implement**:
- ❌ Create `rust/scanner/src/shared_memory/reader.rs`
- ❌ Memory-map both shared memory files (internal + external)
- ❌ Basic quote reading (without torn read prevention initially)
- ❌ Parse `QuoteMetadata` structs
- ❌ Full scan API for testing

**Basic Reader (Without Torn Read Prevention Yet)**:
```rust
pub struct SharedMemoryReader {
    internal_mmap: Mmap,
    external_mmap: Mmap,
    quotes_internal: &'static [QuoteMetadata],
    quotes_external: &'static [QuoteMetadata],
}

impl SharedMemoryReader {
    pub fn new() -> Result<Self> {
        // Memory-map files
        let internal_file = File::open("/var/quote-service/quotes-internal.mmap")?;
        let external_file = File::open("/var/quote-service/quotes-external.mmap")?;

        // ... mapping logic
    }

    // ⚠️ BASIC read (torn read possible - will fix in Task 4.2)
    pub fn read_quote(&self, pair_index: usize) -> Option<QuoteMetadata> {
        let quote = &self.quotes_internal[pair_index];
        // Just copy the struct (not safe yet)
        Some(*quote)
    }
}
```

**Acceptance Criteria**:
- ✅ Can memory-map both shared memory files
- ✅ Can read quote structs from memory
- ✅ Full scan works (even if not safe yet)
- ✅ Integration tests with Go writer

**Files to Create**:
- `rust/scanner/src/shared_memory/reader.rs` (NEW)
- `rust/scanner/src/shared_memory/mod.rs` (NEW)
- `rust/scanner/src/shared_memory/reader_test.rs` (NEW)

---

#### Task 4.2: Torn Read Prevention in Shared Memory ❌
**Priority**: **P0 - CRITICAL CORRECTNESS**
**Estimated Effort**: 3 hours
**Status**: Not started
**Review Source**: ChatGPT critique #1 (Critical)
**Design Doc**: `30.2-SHARED-MEMORY-HYBRID-CHANGE-DETECTION.md` (lines 406-461)
**Dependencies**: Task 4.1 (Basic shared memory reader must exist)

**What to Implement**:
- ❌ Add `read_quote_safe()` function to Rust shared memory reader
- ❌ Implement double-read verification protocol:
  1. Read version v1 (before struct)
  2. Skip if v1 is odd (write in progress)
  3. Copy entire struct
  4. Read version v2 (after struct)
  5. Accept only if v1 == v2 (no concurrent write)
- ❌ Update all read operations to use safe reads
- ❌ Add unit tests for torn read scenarios

**Implementation**:
```rust
/// ❗ CRITICAL: Safe quote read with torn read prevention
fn read_quote_safe(&self, quote: &QuoteMetadata) -> Option<QuoteMetadata> {
    for _ in 0..10 {  // Max 10 retries
        let v1 = quote.version.load(Ordering::Acquire);
        if v1 % 2 != 0 { continue; }  // Skip odd (writing)

        let quote_copy = *quote;  // Copy entire struct

        let v2 = quote.version.load(Ordering::Acquire);
        if v1 == v2 { return Some(quote_copy); }  // ✅ Valid
    }
    None  // Failed after retries
}
```

**Acceptance Criteria**:
- ✅ Double-read verification implemented
- ✅ No torn reads under 1000 writes/sec load
- ✅ Performance: <100ns typical, <500ns under contention
- ✅ Unit tests pass with concurrent writers

**Files to Modify**:
- `rust/scanner/src/shared_memory/reader.rs` (MODIFY - replace basic read with safe read)

---

#### Task 4.3: Hybrid Change Detection ❌
**Priority**: **P1 - PERFORMANCE**
**Estimated Effort**: 3 hours
**Status**: Not started
**Dependencies**: Task 4.2 (Torn read prevention must be implemented)

**What to Implement**:
- ❌ Ring buffer change notification reader
- ❌ Hybrid scan strategy (ring buffer → full scan fallback)
- ❌ Change notification tracking
- ❌ Performance benchmarks

**Files to Create**:
- `rust/scanner/src/shared_memory/change_detection.rs` (NEW)

---

#### Task 4.4: Rust Scanner Tests ❌
**Priority**: **P1 - CRITICAL**
**Estimated Effort**: 2 hours
**Status**: Not started

**Test Coverage**:
- ✅ Torn read scenarios (concurrent Go writer + Rust reader)
- ✅ Hybrid change detection performance
- ✅ Ring buffer wraparound
- ✅ Memory-mapped file edge cases

**Files to Create**:
- `rust/scanner/src/shared_memory/integration_test.rs` (NEW)

---

### **PHASE 5: Integration & Validation** (Week 5, 15-20 hours) ⭐ RENAMED

**Goal**: Production-ready deployment with all enhancements validated

#### Task 5.1: Confidence Score Validation Tests ❌
**Priority**: **P0 - CRITICAL**
**Estimated Effort**: 3 hours
**Status**: Not started
**Dependencies**: Task 3.4 (Confidence Score Integration)

**What to Test**:
- ❌ All 5 factors contribute to score
- ❌ Score always in [0.0, 1.0]
- ❌ Deterministic (same inputs = same output)
- ❌ Scanner decision thresholds work

**Test Scenarios**:
1. **High Confidence Quote**:
   - Input: Fresh pool (3s), 1-hop, 0.2% oracle deviation
   - Expected: Confidence >0.9
   - Assertions: poolAgeFactor >0.9, oracleFactor >0.9

2. **Low Confidence Quote**:
   - Input: Stale pool (45s), 3-hop, 8% oracle deviation
   - Expected: Confidence <0.5
   - Assertions: poolAgeFactor <0.3, oracleFactor <0.2

**Files to Create**:
- `go/pkg/confidence/calculator_integration_test.go` (NEW)

---

#### Task 5.2: 1s Refresh Rate Validation ❌
**Priority**: **P1 - QUICK WIN VALIDATION**
**Estimated Effort**: 2 hours
**Status**: Not started
**Dependencies**: Task 0.1 (1s AMM Refresh implemented)

**What to Test**:
- ❌ AMM pools refresh every 1s in microservices
- ❌ Opportunity capture rate improvement
- ❌ Redis load increase acceptable
- ❌ No performance degradation

**Test Scenarios**:
1. **Refresh Frequency**:
   - Input: Monitor AMM pool for 10 seconds
   - Expected: 10 refresh cycles
   - Assertions: Refresh every 1s ± 100ms

2. **Opportunity Capture**:
   - Input: Simulate price change every 5s
   - Expected: Detection within 1s (vs 10s before)
   - Assertions: 98% capture rate (vs 90% with 10s)

**Files to Create**:
- `tests/integration/refresh_rate_validation_test.go` (NEW)

---

#### Task 5.3: End-to-End Integration Tests ❌
**Priority**: **P1 - PRODUCTION READINESS**
**Estimated Effort**: 4 hours
**Status**: Not started

**What to Test**:
- ❌ Full quote flow: Aggregator → Local + External
- ❌ Shared memory: Go writer → Rust reader
- ❌ Parallel paired quotes (forward + reverse)
- ❌ Confidence scoring in aggregated quotes
- ❌ Timeout handling (local 10ms, external 100ms)

**Files to Create**:
- `tests/integration/e2e_quote_flow_test.go` (NEW)

---

#### Task 5.4: Load Testing (Enhanced) ❌
**Priority**: **P1 - PRODUCTION READINESS**
**Estimated Effort**: 4 hours (was 3h, +1h for new scenarios)
**Status**: Not started

**Additional Load Test Scenarios**:
- ✅ Shared memory read throughput (10,000 reads/sec)
- ✅ Parallel paired quotes under load
- ✅ Confidence calculation overhead (<1ms)
- ✅ Dual shared memory write throughput

**New Load Test**: Shared Memory Reads
```rust
// 10,000 reads/sec sustained for 5 minutes
for _ in 0..10_000 {
    let quotes = reader.read_changed_quotes();
    // Process quotes...
}

// Expected:
// - p50 latency: <500ns
// - p99 latency: <5μs
// - 0% errors
// - No memory leaks
```

**Acceptance Criteria**:
- ✅ Shared memory: 10,000 reads/sec, p99 <5μs
- ✅ Aggregator: 1000 req/s with confidence scoring
- ✅ Parallel paired quotes: 2× speedup vs sequential

**Files to Create**:
- `tests/load/shared_memory_load_test.rs` (NEW)
- `tests/load/k6_quote_services_enhanced.js` (MODIFY)

---

#### Task 5.5: Observability Dashboard (Enhanced) ❌
**Priority**: **P1 - PRODUCTION READINESS**
**Estimated Effort**: 4 hours (was 3h, +1h for new panels)
**Status**: Not started

**Additional Dashboard Panels**:
- ✅ Confidence score distribution (histogram)
- ✅ Torn read retry rate
- ✅ Refresh rate intervals (1s AMM, 30s CLMM)
- ✅ Shared memory write rate
- ✅ Ring buffer utilization

**New Panels**:
1. **Confidence Scoring**:
   - Confidence score distribution (0.0-1.0)
   - Per-factor contribution
   - Scanner decision distribution (execute/verify/skip)

2. **Shared Memory Performance**:
   - Read latency (p50/p95/p99)
   - Write latency
   - Ring buffer utilization
   - Torn read retries

3. **Refresh Rates**:
   - AMM refresh interval (target: 1s)
   - CLMM refresh interval (target: 30s)
   - Refresh queue depth

**Files to Modify**:
- `deployment/monitoring/grafana/dashboards/quote-services.json` (ENHANCE)

---

## 🎯 REORDERED Implementation Priority

### **Week 0.5: Quick Wins (In Current Monolith)** (5-6 hours) ⭐ REORDERED
**Goal**: Standalone improvements before microservices split

**Tasks**:
1. 1s AMM refresh (1h) ✅ QUICK WIN - Simple config change
2. Confidence score algorithm (4h) ❗ CRITICAL - Standalone library

**Deliverable**: Immediate performance gains + confidence algorithm ready for integration

**Why First**: No dependencies, can be done in current monolith, provides immediate value

---

### **Week 1: Local Quote Service** (15-20 hours)
**Goal**: Standalone local quote service with 1s refresh + parallel paired quotes

**Tasks**:
1. Proto definitions with batch streaming (2h)
2. Parallel paired quote calculation (4h) ⭐ CRITICAL
3. Background refresh manager (with 1s AMM from Week 0.5) (4h)
4. Tests (4h)
5. Docker (2h)

**Deliverable**: Local quote service on port 50052 with 1s refresh

---

### **Week 2: External Quote Service** (14-17 hours)
**Goal**: Standalone external quote service with split cache + parallel quotes

**Tasks**:
1. Proto definitions (2h)
2. Split cache (route/price) (3h) ⭐ ENHANCEMENT
3. Parallel paired quotes (3h) ⭐ CRITICAL
4. Provider health tracking (2h)
5. Tests (3h)
6. Docker (2h)

**Deliverable**: External quote service on port 50053 with split cache

---

### **Week 3: Quote Aggregator Service** (20-25 hours) ⭐ EXPANDED
**Goal**: Client-facing aggregator with confidence + dual shared memory + explicit timeouts

**Tasks**:
1. Proto definitions (2h)
2. Dual shared memory writer (5h) ⭐ CRITICAL - Foundation for Rust scanners
3. Explicit aggregator timeouts (2h) ❗ CRITICAL - Moved from Phase 0
4. Confidence score integration (3h) ⭐ CRITICAL - Uses Week 0.5 library
5. Quote merging logic (3h)
6. HTTP API (3h)
7. Tests (4h)
8. Docker (2h)

**Deliverable**: Quote aggregator on port 50051 with shared memory + confidence scoring

---

### **Week 4: Rust Scanner Integration** (12-15 hours) ⭐ NEW PHASE
**Goal**: Rust production scanners with shared memory IPC

**Tasks**:
1. Rust shared memory reader (basic) (4h) - Depends on Week 3 Task 2
2. Torn read prevention (3h) ❗ CRITICAL - Depends on Task 1
3. Hybrid change detection (3h) - Performance optimization
4. Rust scanner tests (2h)

**Deliverable**: Production Rust scanners reading from shared memory with torn read prevention

**Why After Week 3**: Shared memory must exist before Rust can read from it

---

### **Week 5: Integration & Validation** (15-20 hours) ⭐ RENAMED
**Goal**: Production-ready with all enhancements validated

**Tasks**:
1. Confidence score validation (3h)
2. 1s refresh validation (2h)
3. End-to-end integration tests (4h)
4. Load testing (enhanced) (4h)
5. Observability dashboard (enhanced) (4h)

**Deliverable**: Production-ready 3-microservice architecture with Rust scanners

---

## 📊 REORDERED Progress Summary

### Completion Status
- **Phase 0: Quick Wins (In Current Monolith)**: 0% ❌ ⭐ REORDERED
- **Phase 1: Local Quote Service**: 0% ❌
- **Phase 2: External Quote Service**: 0% ❌
- **Phase 3: Quote Aggregator Service**: 0% ❌
- **Phase 4: Rust Scanner Integration**: 0% ❌ ⭐ NEW PHASE
- **Phase 5: Integration & Validation**: 0% ❌ ⭐ RENAMED

### Total Remaining Effort (REORDERED)
- **Week 0.5**: 5-6 hours (Quick wins in current monolith) ⭐ REDUCED (was 8-12h)
- **Week 1**: 15-20 hours (Local Quote Service)
- **Week 2**: 14-17 hours (External Quote Service)
- **Week 3**: 20-25 hours (Quote Aggregator Service + Shared Memory) ⭐ EXPANDED
- **Week 4**: 12-15 hours (Rust Scanner Integration) ⭐ NEW PHASE
- **Week 5**: 15-20 hours (Integration & Validation) ⭐ REDUCED

**Total**: **81-103 hours** (5.5 weeks at part-time, 2.5-3 weeks at full-time)

**Key Changes from v2.0**:
- ✅ **Logical dependency order**: Shared memory → Torn read prevention (was reversed)
- ✅ **Separated Rust scanner work**: New Phase 4 (was mixed into Phase 4 testing)
- ✅ **Quick wins first**: Standalone improvements before microservices (was Phase 0)
- ✅ **Clearer dependencies**: Each task lists what it depends on

---

## 🏆 Expected Benefits (Enhanced with Reviews)

### **Correctness** ⭐ NEW
- ✅ **Torn read prevention**: No data corruption under high load
- ✅ **Confidence scoring**: Deterministic arbitrage decisions
- ✅ **Explicit timeouts**: Predictable latency bounds

### **Performance**
- ✅ **1s AMM refresh**: 10× faster opportunity capture (90% → 98%)
- ✅ **Parallel paired quotes**: 2× faster quote calculation
- ✅ **Hybrid change detection**: 200× faster no-change case

### **Reliability**
- ✅ **Failure isolation**: External API failures don't affect local
- ✅ **Circuit breakers**: Per-service resilience
- ✅ **Non-blocking aggregator**: External never blocks local

### **HFT Suitability**
- ✅ **Sub-microsecond reads**: Shared memory with torn read prevention
- ✅ **Confidence-based execution**: No blind arbitrage execution
- ✅ **Exchange-grade architecture**: "This is no longer a crypto bot" (ChatGPT)

---

## 🔗 Related Documents

**Primary References**:
- ⭐ **Architecture**: `30-QUOTE-SERVICE-ARCHITECTURE.md` v3.1 - Source of Truth
- ⭐ **Shared Memory**: `30.2-SHARED-MEMORY-HYBRID-CHANGE-DETECTION.md` - Hybrid change detection
- ⭐ **Test Plan**: `26-QUOTE-SERVICE-TEST-PLAN.md` - Comprehensive testing (updated)

**Review Documents** ⭐ NEW:
- `30.1-QUOTE-SERVICE-ARCHITECTURE-REVIEW.md` - Initial architectural review
- `30.3-REFRESH-RATE-ANALYSIS.md` - Gemini critique response (1s refresh feasibility)
- `30.4-CHATGPT-REVIEW-RESPONSE.md` - ChatGPT HFT architect review (9.3/10)

**Supporting Docs**:
- `07-INITIAL-HFT-ARCHITECTURE.md` - Overall HFT system
- `proto/README.md` - Proto file generation

---

**Last Updated**: 2025-12-31
**Document Version**: 3.1 ⭐ **REORDERED BASED ON LOGICAL DEPENDENCIES**
**Status**: Active Development - Dependency-Corrected Plan ✅
**Next Action**: Implement Phase 0 (Quick Wins: 1s AMM Refresh + Confidence Algorithm) ❗

**Critical Fix in v3.1**: Task dependencies now respect logical order:
1. ✅ Shared memory writer (Phase 3) → Rust reader (Phase 4) → Torn read prevention (Phase 4)
2. ✅ Quick wins first (Phase 0) → Use in microservices (Phases 1-3)
3. ✅ Confidence algorithm (Phase 0) → Integration in aggregator (Phase 3)
4. ✅ Aggregator service (Phase 3) → Explicit timeouts (Phase 3, not Phase 0)
