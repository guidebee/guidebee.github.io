---
layout: single
title: "Quote Service Rewrite: Clean Architecture for Long-Term Maintainability"
date: 2025-12-25
permalink: /posts/2025/12/quote-service-rewrite-clean-architecture-for-maintainability/
categories:
  - blog
tags:
  - solana
  - go
  - architecture
  - clean-architecture
  - refactoring
  - design-patterns
  - microservices
  - hft
  - trading
  - christmas
excerpt: "🎄 Merry Christmas! Planning a comprehensive rewrite of quote-service with clean architecture principles: 85% code reduction, 20x faster quotes, 4x better test coverage, and dramatically improved maintainability. Why we're rebuilding the foundation for long-term success."
---

## 🎄 Merry Christmas and Happy New Year! 🎄

On this Christmas Day 2025, I'm taking a moment to reflect on the journey of building this Solana HFT trading system. As we celebrate with family and friends, I'm also planning the next major evolution of our architecture.

**Wishing everyone a Merry Christmas and a prosperous Happy New Year!** May 2026 bring successful trades, robust systems, and minimal bugs! 🎉

Today's post is a bit different—instead of implementation details, I'm sharing the **architectural rewrite plan** for our quote-service. It's a story of **technical debt, lessons learned, and the path to sustainable architecture**.

---

## TL;DR

Planning a comprehensive rewrite of quote-service with clean architecture principles **AND HFT integration**:

1. **85% Code Reduction**: 50K lines → 15K lines through proper separation of concerns
2. **Sub-10ms Cached Quotes**: < 10ms HFT-critical latency (vs current 200ms uncached)
3. **4x Better Test Coverage**: 20% → 80%+ with dependency injection and interfaces
4. **Dramatically Better Maintainability**: Internal packages, clean architecture, single responsibility
5. **Service Separation**: 3 services (quote, pool discovery, RPC proxy) vs 1 monolith
6. **Technology Decision**: Go for speed (2-3 weeks), Rust RPC proxy for shared infrastructure
7. **HFT Pipeline Integration**: Shredstream cache (300-800ms head start), FlatBuffers events (20-150x faster), NATS MARKET_DATA stream

**The Core Insight**: The current quote-service **works**, but it's **unmaintainable** and **not HFT-ready**. We need to rebuild the foundation now before technical debt makes future changes impossible, AND we need to integrate with the HFT pipeline for sub-200ms end-to-end execution.

---

## Table of Contents

1. [The Problem: Why Rewrite a Working System?](#the-problem-why-rewrite-a-working-system)
2. [Current Architecture: Design Flaws](#current-architecture-design-flaws)
3. [New Architecture: Clean Separation](#new-architecture-clean-separation)
4. [Go vs Rust Decision](#go-vs-rust-decision)
5. [HTTP + gRPC: Combined vs Split](#http--grpc-combined-vs-split)
6. [HFT Integration Requirements](#hft-integration-requirements) ← **NEW**
7. [Clean Architecture Benefits](#clean-architecture-benefits)
8. [Technology Stack Decisions](#technology-stack-decisions)
9. [Expected Improvements](#expected-improvements)
10. [Conclusion: Building for the Future](#conclusion-building-for-the-future)

---

## The Problem: Why Rewrite a Working System?

### It Works, But...

The current quote-service is **feature-complete and functional**:
- ✅ Serves quotes via HTTP and gRPC
- ✅ Supports 6 DEX protocols (Raydium, Meteora, Orca, Pump.fun)
- ✅ Real-time WebSocket updates
- ✅ 99.99% availability with RPC pool
- ✅ Redis crash recovery
- ✅ Full observability (Grafana LGTM stack: Loki, Grafana, Tempo, Mimir)

**So why rewrite?**

Because **"works" is not enough for long-term success**. The system has critical architectural flaws that make it:
1. **Difficult to maintain** - 96KB `cache.go` file with 50+ methods
2. **Hard to test** - Tightly coupled components, 20% test coverage
3. **Slow to extend** - Adding features requires touching multiple files
4. **Risky to deploy** - No confidence in changes due to poor testing
5. **Impossible to reason about** - Mixed concerns everywhere

### The Technical Debt Reality

```
Current Codebase Health:
├── Lines of Code: 50,000+ (monolithic)
├── Test Coverage: ~20% (hard to test)
├── Files in cmd/: 20+ files (violates Go standards)
├── Largest File: 96KB cache.go (unmaintainable)
└── Architectural Pattern: Big Ball of Mud ❌
```

**This is a ticking time bomb.** Every feature we add makes it worse. Every bug fix becomes harder. Eventually, we'll reach a point where the system is **too complex to understand** and **too risky to change**.

**The time to fix this is NOW, while we still can.**

---

## Current Architecture: Design Flaws

### Flaw #1: Monolithic `cache.go` (96KB, 50+ methods)

**The Problem:**

```go
// cache.go mixes EVERYTHING in one file:
type QuoteCache struct {
    router            *pkg.SimpleRouter      // Pool routing
    solClient         *sol.Client            // RPC client ❌
    wsPool            *subscription.WSPool   // WebSocket ❌
    oraclePriceFetcher *oracle.PriceFetcher  // Oracle
    cache             map[string]*CachedQuote // Actual cache
    poolLiquidity     map[string]float64     // Pool state ❌
    // ... 20 more fields
}

// 50+ methods that do everything:
func (c *QuoteCache) UpdateQuote()          // Quote refresh
func (c *QuoteCache) DiscoverPools()        // Pool discovery ❌
func (c *QuoteCache) ManageRPCPool()        // RPC management ❌
func (c *QuoteCache) HandleWebSocket()      // WebSocket ❌
// ... 46 more methods
```

**Why This Is Bad:**
- **Violates Single Responsibility Principle** - Does 5 different things
- **Impossible to test in isolation** - Too many dependencies
- **Cannot reason about code** - 96KB file is too large to hold in your head
- **Changes have unpredictable side effects** - Everything is interconnected

**What Should Happen:**
- `QuoteCache` should ONLY cache quotes (1 responsibility)
- Pool discovery → Separate service
- RPC management → Rust RPC Proxy
- WebSocket → Pool discovery service

### Flaw #2: RPC Logic Embedded in Service

**The Problem:**

```
pkg/sol/rpc_pool.go (1200+ lines)
├── RPC pool management
├── Health monitoring
├── Rate limiting
├── Failover logic
└── Cannot be reused by other services ❌
```

**Why This Is Bad:**
- **Code duplication** - Scanner needs RPC pool, must copy-paste
- **Inconsistent behavior** - Each service implements RPC differently
- **Wasted effort** - Solving the same problem multiple times
- **Bugs multiply** - Fix a bug in quote-service, scanner still broken

**What Should Happen:**
- Centralized Rust RPC Proxy (see [docs/25-RUST-RPC-PROXY-DESIGN.md](https://github.com/guidebee/solana-trading-system))
- Used by ALL services (quote, scanner, executor)
- Single source of truth for RPC management

### Flaw #3: Pool Discovery During Quote Serving

**The Problem:**

```
Every 30 seconds:
1. UpdateQuote() triggered
2. For each pair:
   ├─ QueryAllPools() ← Makes RPC calls! ❌
   ├─ Fetch pool state from blockchain (200ms)
   ├─ Calculate quote
   └─ Cache result

PROBLEM: Discovery blocks quote serving!
```

**Why This Is Bad:**
- **Slow** - Discovery takes 200ms, blocks quote serving
- **Unreliable** - RPC failures cause quote serving to fail
- **Wasteful** - Discovering same pools every 30s
- **Tight coupling** - Quote logic mixed with discovery logic

**What Should Happen:**
- Separate pool-discovery-service (runs every 5 minutes)
- Writes discovered pools to Redis
- Quote-service just reads from Redis (0.5ms)
- **No blocking, no coupling**

### Flaw #4: No Internal Packages

**The Problem:**

```
Current (WRONG):
go/cmd/quote-service/
├── main.go
├── cache.go
├── grpc_server.go
├── handler_*.go (10 files)
└── ... all logic in cmd/ ❌

Problems:
- Violates Go project layout standards
- Cannot import logic in other services
- Difficult to test (no interfaces)
- Everything is tightly coupled
```

**What Should Happen:**

```
Correct Structure:
go/
├── cmd/quote-service/
│   └── main.go (ONLY DI wiring, 100 lines)
│
└── internal/quote-service/
    ├── domain/       # Interfaces + models
    ├── repository/   # Data access (Redis, cache)
    ├── calculator/   # Quote calculation
    ├── service/      # Business logic
    └── api/          # HTTP + gRPC handlers
```

**Benefits:**
- ✅ Clean separation of concerns
- ✅ Easy to test (inject mocks via interfaces)
- ✅ Each package has ONE responsibility
- ✅ Follows Go best practices

### Flaw #5: Hard to Test

**Current Test Coverage: 20%** ❌

**Why So Low?**

```go
// Current code (impossible to test):
func (c *QuoteCache) UpdateQuote() {
    // Hard-coded RPC client ❌
    pools := c.solClient.QueryAllPools(...)

    // Hard-coded WebSocket ❌
    c.wsPool.Subscribe(...)

    // No interfaces, cannot inject mocks ❌
}

// To test this, you need:
- Real RPC endpoint (flaky, slow)
- Real WebSocket connection (flaky, slow)
- Real Redis (integration test, not unit test)
- Full infrastructure (NATS, Prometheus, etc.)

Result: Nobody writes tests, coverage stays at 20%
```

**What Should Happen:**

```go
// New code (easy to test):
type QuoteService struct {
    poolRepo      domain.PoolReader      // Interface! ✅
    calculator    domain.PriceCalculator // Interface! ✅
    cacheManager  domain.CacheManager    // Interface! ✅
}

// To test this:
func TestQuoteService(t *testing.T) {
    // Inject mocks! No real infrastructure needed!
    mockPoolRepo := &MockPoolReader{}
    mockCalculator := &MockPriceCalculator{}
    mockCache := &MockCacheManager{}

    service := NewQuoteService(mockPoolRepo, mockCalculator, mockCache)

    // Test business logic in isolation ✅
    quote, err := service.GetQuote(ctx, "SOL", "USDC", 1000000000)
    assert.NoError(t, err)
    assert.Equal(t, expectedOutput, quote.OutputAmount)
}

Result: 80%+ test coverage, fast unit tests ✅
```

---

## New Architecture: Clean Separation

### High-Level Architecture

**Before (Monolithic):**

```
┌───────────────────────────────────────────────────┐
│          Quote Service (Single Monolith)          │
│                                                   │
│  • Quote caching     (Good ✅)                    │
│  • Pool discovery    (Blocks serving ❌)          │
│  • RPC management    (Should be shared ❌)        │
│  • WebSocket updates (Blocks serving ❌)          │
│  • HTTP API          (Good ✅)                    │
│  • gRPC streaming    (Good ✅)                    │
│                                                   │
│  PROBLEMS:                                        │
│  - 50K lines, unmaintainable                      │
│  - Discovery blocks quote serving                 │
│  - RPC logic cannot be reused                     │
│  - Hard to test (20% coverage)                    │
└───────────────────────────────────────────────────┘
```

**After (Clean Separation + HFT Integration):**

```
┌─────────────────────────────────────────────────────┐
│    Shredstream Scanner (Rust - 300-800ms Advance)   │
│  • QUIC protocol for unconfirmed slot data          │
│  • Publishes: pool.state.updated.* (NATS)           │
│  • Provides 300-800ms head start over RPC           │
└────────────────────┬────────────────────────────────┘
                     ↓ NATS pool.state.updated.*
┌─────────────────────────────────────────────────────┐
│      Pool Discovery Service (NEW - Independent)     │
│  • Discovers pools every 5 minutes                  │
│  • Writes to Redis (pool metadata)                  │
│  • Solscan enrichment (TVL, 24h volume)             │
│  • Pool quality filtering (liquidity, status)       │
│  • 8K lines, single responsibility ✅               │
└────────────────────┬────────────────────────────────┘
                     ↓ Redis (pool metadata)
┌─────────────────────────────────────────────────────┐
│   Quote Service (REWRITTEN - Clean + HFT Ready)     │
│                                                     │
│  INPUTS:                                            │
│  • Redis pool metadata (5-10ms)                     │
│  • NATS pool.state.updated.* (Shredstream cache)    │
│                                                     │
│  CORE:                                              │
│  • Hybrid cache: Shredstream (5ms) → In-memory      │
│  • Slot-based consistency (only update if newer)    │
│  • Thread-safe pool cache (sync.RWMutex)            │
│  • 15K lines, clean architecture ✅                 │
│  • 80%+ test coverage ✅                            │
│                                                     │
│  OUTPUTS:                                           │
│  • HTTP API :8080 (< 10ms quotes)                   │
│  • gRPC streaming :50051                            │
│  • NATS market.swap_route.* (FlatBuffers events)    │
│                                                     │
│  Internal Structure:                                │
│  ├── domain/      (interfaces, models)              │
│  ├── repository/  (Redis, cache, oracle)            │
│  ├── cache/       (Shredstream pool cache) ← NEW    │
│  ├── calculator/  (pool math, routing)              │
│  ├── service/     (business logic)                  │
│  ├── events/      (FlatBuffers publisher) ← NEW     │
│  ├── nats/        (NATS subscriber) ← NEW           │
│  └── api/         (HTTP + gRPC)                     │
└────────────────────┬────────────────────────────────┘
                     ↓ NATS MARKET_DATA stream
┌─────────────────────────────────────────────────────┐
│      Scanner Service (Stage 1: Opportunity Det.)    │
│  • Subscribes: market.swap_route.*                  │
│  • Detects arbitrage opportunities                  │
│  • Publishes: opportunity.* (< 50ms)                │
└─────────────────────────────────────────────────────┘
                     ↓ HTTP (RPC calls)
┌─────────────────────────────────────────────────────┐
│         Rust RPC Proxy (Shared Infrastructure)      │
│  • Centralized RPC management                       │
│  • Used by ALL services (quote, scanner, executor)  │
│  • Rate limiting, health monitoring                 │
│  • Connection pooling, circuit breaker              │
└─────────────────────────────────────────────────────┘
```

**HFT Pipeline Flow (Stage 0 → Stage 1):**

```
Stage 0: Quote Service (< 10ms per quote)
    ↓ publishes: market.swap_route.* (FlatBuffers, <1ms)
Stage 1: Scanner (< 50ms detection)
    ↓ publishes: opportunity.*
Stage 2: Planner (< 50ms planning)
    ↓ publishes: execution.planned
Stage 3: Executor (< 90ms execution)
    ↓ publishes: execution.completed

TOTAL: < 200ms end-to-end (vs current 1.7s = 8.5x faster)
```

### Key Improvements

| Aspect | Before (Monolithic) | After (Clean) | Benefit |
|--------|---------------------|---------------|---------|
| **Quote Latency** | ~200ms (discovery included) | < 10ms (Redis lookup) | **20x faster** |
| **Code Size** | 50K lines | 15K lines (quote) + 8K (discovery) | **85% reduction** |
| **Test Coverage** | 20% | > 80% target | **4x better** |
| **Maintainability** | Poor (monolithic) | Excellent (clean architecture) | **High** |
| **RPC Reusability** | No (embedded) | Yes (shared proxy) | **High** |
| **Deployment Risk** | High (single service) | Low (independent services) | **Lower** |

---

## Go vs Rust Decision

### Performance Analysis: Is Rust Worth It?

**Go (Optimized):**
```
Redis pool lookup:      0.5ms
Pool math calculation:  0.2ms
Price calculation:      0.1ms
Response serialization: 0.1ms
─────────────────────────────
TOTAL:                  0.9ms ✅ Excellent
```

**Rust (Theoretical):**
```
Redis pool lookup:      0.3ms  (faster client)
Pool math calculation:  0.1ms  (zero-cost abstractions)
Price calculation:      0.05ms (SIMD)
Response serialization: 0.05ms (serde zero-copy)
─────────────────────────────
TOTAL:                  0.5ms ✅ Better, but marginal
```

**Verdict: 0.4ms improvement (44% faster) is NOT worth 5 extra weeks**

### Decision Matrix

| Factor | Go | Rust | Winner |
|--------|-----|------|--------|
| **Development Speed** | 2-3 weeks ✅ | 6-8 weeks ⚠️ | **Go** |
| **Team Knowledge** | Proven ✅ | Learning curve ⚠️ | **Go** |
| **Performance** | <10ms ✅ | <5ms ✅ | Tie (both good enough) |
| **Code Reuse** | Can reuse router/pool ✅ | Rewrite everything ❌ | **Go** |
| **Risk** | Low ✅ | High ⚠️ | **Go** |

**Decision: Go for Quote Service ✅**

**Rationale:**
1. Solo developer - stick to known language
2. Time to market - 2-3 weeks vs 6-8 weeks
3. Performance - <10ms target easily met with Go
4. Code reuse - can reuse existing `pkg/router`, `pkg/pool`
5. Risk mitigation - proven technology, easy rollback

### Hybrid Approach (Best of Both Worlds)

```
Use Go for:
✅ Quote Service (fast delivery, good enough performance)
✅ Pool Discovery (I/O bound, Go is perfect)

Use Rust for:
✅ RPC Proxy (shared infrastructure, worth investment)
✅ Transaction Builder (memory-critical, zero-copy)
✅ Shredstream Parser (ultra-low latency)
```

**Result: Fast delivery where it matters, peak performance where it counts**

---

## HTTP + gRPC: Combined vs Split

### The Question

Should HTTP and gRPC be in one service or split into two separate services?

### Option 1: Combined (RECOMMENDED ✅)

```
┌─────────────────────────────────────────┐
│    Quote Service (Single Process)       │
│                                         │
│  ┌─────────────┐   ┌────────────────┐  │
│  │ HTTP :8080  │   │ gRPC :50051    │  │
│  └──────┬──────┘   └────────┬───────┘  │
│         │                    │          │
│         └────────┬───────────┘          │
│                  ▼                      │
│    ┌──────────────────────────┐        │
│    │  In-Memory Cache         │        │
│    │  (SHARED! ✅)            │        │
│    │  0.3ms access            │        │
│    └──────────────────────────┘        │
└─────────────────────────────────────────┘
```

**Performance:**
- HTTP cached quote: **0.3ms** ✅
- gRPC stream update: **0.15ms** ✅
- Throughput: **10,000 req/s** ✅

### Option 2: Split (NOT RECOMMENDED ⚠️)

```
┌──────────────────────────┐
│  HTTP Service :8080      │
│  Uses Redis cache        │
└───────┬──────────────────┘
        ▼
   Redis (1ms overhead)
        ▲
┌───────┴──────────────────┐
│  gRPC Service :50051     │
│  Uses Redis cache        │
└──────────────────────────┘
```

**Performance:**
- HTTP cached quote: **1.2ms** (4x slower ❌)
- gRPC stream update: **1.05ms** (7x slower ❌)
- Throughput: **1,000 req/s** (10x less ❌)

### Performance Comparison

| Scenario | Combined | Split (Redis) | Difference |
|----------|----------|---------------|------------|
| **Cached Quote (HTTP)** | 0.3ms ✅ | 1.2ms ⚠️ | **4x slower** |
| **gRPC Stream Update** | 0.15ms ✅ | 1.05ms ⚠️ | **7x slower** |
| **Throughput** | 10K req/s ✅ | 1K req/s ⚠️ | **10x less** |
| **Memory** | 300MB ✅ | 600MB ⚠️ | **2x more** |
| **Services to Deploy** | 1 ✅ | 2 ⚠️ | **2x ops** |

### Decision: COMBINED ✅

**Why Combined Wins:**

1. **Performance** - 4-7x faster (CRITICAL for HFT)
   - In-memory cache: 0.3ms
   - Redis cache: 1.2ms
   - **Redis overhead kills performance**

2. **Throughput** - 10x higher capacity
   - Combined: 10K req/s
   - Split: 1K req/s (Redis bottleneck)

3. **Simplicity** - Solo developer
   - 1 service vs 2 services
   - 1 deployment vs 2 deployments

4. **Memory Efficiency** - 50% less RAM
   - Combined: 300MB (single in-memory cache)
   - Split: 600MB (2x Redis storage)

**The Insight:** For HFT systems targeting sub-10ms latency, **in-memory cache sharing** between HTTP and gRPC is non-negotiable. The 1ms Redis overhead destroys performance gains from service separation.

---

## HFT Integration Requirements

Quote-service is **Stage 0** of the HFT pipeline. These requirements are **NON-NEGOTIABLE** for sub-200ms end-to-end execution.

### Performance Targets ⚡

**CRITICAL:** Quote-service must meet these latency targets to enable the full HFT pipeline.

| Metric | Target | HFT Requirement |
|--------|--------|-----------------|
| **Cached Quote (Cache Hit)** | **< 10ms** | **MANDATORY** |
| **Cached Quote (Shredstream)** | **< 5ms** | **OPTIMAL** |
| **NATS Event Publishing** | **< 1ms** | **10,000 events/sec** |
| **Pool State Update** | **Slot-based** | **Only if newer slot** |
| **Cache Hit Rate** | **> 95%** | **Minimize RPC calls** |

### 1. Shredstream Pool State Cache (300-800ms Advance)

Shredstream provides **unconfirmed slot data** via QUIC protocol, giving us a **300-800ms head start** over RPC.

**Implementation:**

```go
// internal/quote-service/cache/shredstream_cache.go

type PoolStateCache struct {
    mu     sync.RWMutex
    pools  map[string]*PoolState // key: pool address
    config CacheConfig
}

type PoolState struct {
    Address      string
    BaseMint     string
    QuoteMint    string
    BaseReserve  uint64
    QuoteReserve uint64
    Liquidity    float64
    Price        float64
    Slot         uint64       // CRITICAL: For consistency
    LastUpdated  time.Time
}

// Slot-based consistency: ONLY update if newer slot
func (c *PoolStateCache) Update(state *PoolState) {
    c.mu.Lock()
    defer c.mu.Unlock()

    existing, exists := c.pools[state.Address]
    if exists && existing.Slot >= state.Slot {
        return // Ignore stale update
    }

    state.LastUpdated = time.Now()
    c.pools[state.Address] = state
}

// Thread-safe read
func (c *PoolStateCache) Get(address string) (*PoolState, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    state, exists := c.pools[address]
    if !exists {
        return nil, false
    }

    // Check staleness (30s threshold)
    if time.Since(state.LastUpdated) > 30*time.Second {
        return nil, false
    }

    return state, true
}
```

### 2. NATS Subscriber for Shredstream Events

Subscribe to `pool.state.updated.*` events from Shredstream Scanner.

**Implementation:**

```go
// internal/quote-service/nats/subscriber.go

type ShredstreamSubscriber struct {
    nc    *nats.Conn
    js    nats.JetStreamContext
    cache *cache.PoolStateCache
}

func (s *ShredstreamSubscriber) Start(ctx context.Context) error {
    // Subscribe to pool state updates
    sub, err := s.js.Subscribe(
        "pool.state.updated.*",
        func(msg *nats.Msg) {
            s.handlePoolUpdate(msg)
            msg.Ack()
        },
        nats.Durable("quote-service-pool-updates"),
        nats.DeliverAll(),
    )
    if err != nil {
        return fmt.Errorf("subscribe failed: %w", err)
    }

    // Background eviction loop
    go s.evictionLoop(ctx)

    return nil
}

func (s *ShredstreamSubscriber) handlePoolUpdate(msg *nats.Msg) {
    var state cache.PoolState
    if err := json.Unmarshal(msg.Data, &state); err != nil {
        log.Warn("Failed to unmarshal pool state", "error", err)
        return
    }

    // Update cache with slot-based consistency
    s.cache.Update(&state)
}

// Evict stale entries every 60s
func (s *ShredstreamSubscriber) evictionLoop(ctx context.Context) {
    ticker := time.NewTicker(60 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            s.cache.Evict(30 * time.Second)
        case <-ctx.Done():
            return
        }
    }
}
```

### 3. FlatBuffers Event Publishing (20-150x Faster)

Publish swap route events to NATS MARKET_DATA stream using FlatBuffers for **zero-copy serialization**.

**FlatBuffers Schema:**

```flatbuffers
// internal/quote-service/events/schemas.fbs

namespace events;

table SwapRouteEvent {
  token_in: string;
  token_out: string;
  amount_in: uint64;
  amount_out: uint64;
  price: double;
  price_impact_bps: uint32;
  route: [RouteHop];
  protocol: string;
  pool_address: string;
  slot: uint64;
  timestamp: uint64;
  trace_id: string;
}

table RouteHop {
  protocol: string;
  pool_address: string;
  input_mint: string;
  output_mint: string;
  amount_in: uint64;
  amount_out: uint64;
  fee_bps: uint32;
}
```

**Publisher Implementation:**

```go
// internal/quote-service/events/publisher.go

type FlatBuffersPublisher struct {
    js      nats.JetStreamContext
    builder *flatbuffers.Builder
}

func (p *FlatBuffersPublisher) PublishSwapRoute(
    ctx context.Context,
    quote *domain.Quote,
) error {
    // Reset builder for reuse
    p.builder.Reset()

    // Build FlatBuffers message
    tokenIn := p.builder.CreateString(quote.InputMint)
    tokenOut := p.builder.CreateString(quote.OutputMint)
    protocol := p.builder.CreateString(quote.Protocol)
    poolAddr := p.builder.CreateString(quote.PoolAddress)
    traceID := p.builder.CreateString(observability.TraceID(ctx))

    SwapRouteEventStart(p.builder)
    SwapRouteEventAddTokenIn(p.builder, tokenIn)
    SwapRouteEventAddTokenOut(p.builder, tokenOut)
    SwapRouteEventAddAmountIn(p.builder, quote.AmountIn)
    SwapRouteEventAddAmountOut(p.builder, quote.AmountOut)
    SwapRouteEventAddPrice(p.builder, quote.Price)
    SwapRouteEventAddPriceImpactBps(p.builder, quote.PriceImpactBps)
    SwapRouteEventAddProtocol(p.builder, protocol)
    SwapRouteEventAddPoolAddress(p.builder, poolAddr)
    SwapRouteEventAddSlot(p.builder, quote.Slot)
    SwapRouteEventAddTimestamp(p.builder, uint64(time.Now().Unix()))
    SwapRouteEventAddTraceId(p.builder, traceID)
    event := SwapRouteEventEnd(p.builder)

    p.builder.Finish(event)

    // Publish to NATS (< 1ms)
    subject := fmt.Sprintf("market.swap_route.%s.%s",
        quote.InputMint[:8], quote.OutputMint[:8])

    _, err := p.js.Publish(subject, p.builder.FinishedBytes(),
        nats.MsgId(traceID))

    return err
}
```

**Performance Comparison:**

| Format | Encode | Decode | Size | Performance |
|--------|--------|--------|------|-------------|
| **FlatBuffers** | **100ns** | **50ns** | **400 bytes** | **20-150x faster** ✅ |
| JSON | 500ns | 2000ns | 1200 bytes | Baseline |
| Protobuf | 200ns | 800ns | 600 bytes | 2-10x faster |

### 4. Hybrid Cache Strategy

Three-tier cache strategy for optimal latency:

```go
// internal/quote-service/service/quote_service.go

func (s *QuoteService) GetQuote(
    ctx context.Context,
    inputMint, outputMint string,
    amount uint64,
) (*domain.Quote, error) {

    // Strategy 1: Try Shredstream pool cache (5-10ms)
    if s.config.Shredstream.Enabled {
        quote, err := s.getQuoteFromShredstream(inputMint, outputMint, amount)
        if err == nil {
            s.metrics.CacheHits.Inc()
            return quote, nil
        }
    }

    // Strategy 2: Try in-memory quote cache (< 5ms)
    if cached, ok := s.cache.Get(inputMint, outputMint, amount); ok {
        if time.Since(cached.Timestamp) < s.config.Cache.TTL {
            s.metrics.CacheHits.Inc()
            return cached, nil
        }
    }

    // Strategy 3: Calculate fresh quote (100-200ms fallback)
    s.metrics.CacheMisses.Inc()
    quote, err := s.calculateQuote(ctx, inputMint, outputMint, amount)
    if err != nil {
        return nil, err
    }

    // Cache for future requests
    s.cache.Set(inputMint, outputMint, amount, quote)

    return quote, nil
}
```

### 5. Configuration

Environment variables for HFT integration:

```bash
# Shredstream Integration
SHREDSTREAM_ENABLED=true
SHREDSTREAM_CACHE_MAX_STALENESS=30s
SHREDSTREAM_EVICTION_INTERVAL=60s

# NATS Configuration
NATS_URL=nats://localhost:4222
NATS_SUBJECT_POOL_UPDATES="pool.state.updated.*"
NATS_SUBJECT_SWAP_ROUTE="market.swap_route"
NATS_DURABLE_NAME="quote-service-pool-updates"

# HFT Performance Targets
HFT_QUOTE_LATENCY_TARGET_MS=10
HFT_EVENT_PUBLISH_RATE_TARGET=10000
HFT_CACHE_HIT_RATE_TARGET=0.95

# FlatBuffers
FLATBUFFERS_ENABLED=true
FLATBUFFERS_BUILDER_INITIAL_SIZE=1024
```

### 6. Updated Package Structure

```
internal/quote-service/
├── cache/              # NEW: Shredstream pool cache
│   ├── shredstream_cache.go
│   └── eviction.go
├── events/             # NEW: FlatBuffers event publishing
│   ├── publisher.go
│   └── schemas.fbs     # FlatBuffers schema
└── nats/               # NEW: NATS integration
    ├── subscriber.go   # Pool state updates
    └── kill_switch.go  # Emergency stop
```

### 7. Why FlatBuffers Over JSON/Protobuf?

**FlatBuffers Advantages:**

1. **Zero-copy deserialization** - Access data without parsing
2. **20-150x faster** than JSON encoding/decoding
3. **Smaller message size** - 400 bytes vs 1200 bytes (JSON)
4. **Backward/forward compatible** - Schema evolution
5. **No runtime serialization** - Data stored in-memory ready to send

**When to Use FlatBuffers:**

- ✅ High-frequency events (10,000/sec)
- ✅ Latency-critical paths (< 1ms publish)
- ✅ Large message volumes
- ❌ Human-readable debugging (use JSON for admin APIs)

### 8. HFT Pipeline Integration

Quote-service is **Stage 0** of the 4-stage HFT pipeline:

```
┌─────────────────────────────────────────────┐
│ Stage 0: Quote Service (< 10ms)             │
│ ─────────────────────────────────────────── │
│ INPUT:  HTTP/gRPC request                   │
│ PROCESS: Hybrid cache (Shredstream → Mem)  │
│ OUTPUT: FlatBuffers event → MARKET_DATA     │
└────────────────┬────────────────────────────┘
                 ↓ NATS: market.swap_route.*
┌─────────────────────────────────────────────┐
│ Stage 1: Scanner (< 50ms)                   │
│ Detects arbitrage opportunities             │
└────────────────┬────────────────────────────┘
                 ↓ NATS: opportunity.*
┌─────────────────────────────────────────────┐
│ Stage 2: Planner (< 50ms)                   │
│ Plans execution strategy                    │
└────────────────┬────────────────────────────┘
                 ↓ NATS: execution.planned
┌─────────────────────────────────────────────┐
│ Stage 3: Executor (< 90ms)                  │
│ Submits Jito bundle                         │
└─────────────────────────────────────────────┘

TOTAL: < 200ms end-to-end (vs current 1.7s)
```

**Quote Service Responsibilities:**

- ✅ Serve quotes in < 10ms (Stage 0 target)
- ✅ Publish FlatBuffers events to MARKET_DATA stream
- ✅ Subscribe to Shredstream pool state updates
- ✅ Maintain > 95% cache hit rate
- ✅ Handle 10,000 events/sec throughput

---

## Clean Architecture Benefits

### Internal Package Structure

**New Directory Layout:**

```
go/
├── cmd/
│   ├── quote-service/
│   │   └── main.go                    # 100 lines (ONLY DI wiring)
│   └── pool-discovery-service/
│       └── main.go
│
└── internal/
    ├── quote-service/
    │   ├── domain/                    # Core business logic
    │   │   ├── interfaces.go          # PoolReader, PriceCalculator
    │   │   ├── quote.go               # Quote, Pool models
    │   │   └── errors.go              # Business errors
    │   │
    │   ├── repository/                # Data access
    │   │   ├── pool_repository.go     # Redis pool reader
    │   │   ├── cache_repository.go    # In-memory cache
    │   │   └── oracle_repository.go   # Pyth/Jupiter
    │   │
    │   ├── calculator/                # Business logic
    │   │   ├── pool_calculator.go     # AMM math
    │   │   ├── slippage_calculator.go # Price impact
    │   │   └── route_optimizer.go     # Best route
    │   │
    │   ├── service/                   # Orchestration
    │   │   ├── quote_service.go       # Quote orchestration
    │   │   ├── price_service.go       # Price calculation
    │   │   └── cache_service.go       # Cache management
    │   │
    │   └── api/                       # HTTP + gRPC
    │       ├── http/handler.go        # Gin handlers
    │       └── grpc/server.go         # gRPC streaming
    │
    └── pool-discovery/
        ├── scanner/                   # DEX scanners
        ├── storage/                   # Redis writer
        └── scheduler/                 # Periodic job
```

### Code Size Reduction

**Before (Monolithic):**
```
cmd/quote-service/
├── main.go           52,844 bytes ❌
├── cache.go          96,419 bytes ❌
├── grpc_server.go    40,734 bytes ❌
└── ... 17 more files

TOTAL: 317KB (50K+ lines) ❌
```

**After (Clean Architecture):**
```
internal/quote-service/
├── domain/           4,500 bytes ✅
├── repository/       10,000 bytes ✅
├── calculator/       10,000 bytes ✅
├── service/          9,000 bytes ✅
└── api/              10,000 bytes ✅

cmd/quote-service/
└── main.go           3,000 bytes ✅

TOTAL: 46.5KB (15K lines) ✅

REDUCTION: 85% less code! ✅
```

### Testability Example

**Before (Impossible to Test):**

```go
// All dependencies hard-coded
func (c *QuoteCache) UpdateQuote() {
    pools := c.solClient.QueryAllPools(...) // Hard-coded RPC ❌
    c.wsPool.Subscribe(...)                  // Hard-coded WS ❌
    // Cannot inject mocks, must use real infrastructure
}

// Test coverage: 20% (too hard to test)
```

**After (Easy to Test):**

```go
// All dependencies are interfaces
type QuoteService struct {
    poolRepo     domain.PoolReader      // Interface ✅
    calculator   domain.PriceCalculator // Interface ✅
    cacheManager domain.CacheManager    // Interface ✅
}

// Test with mocks
func TestGetQuote(t *testing.T) {
    mockPoolRepo := &MockPoolReader{
        pools: testPools, // Inject test data
    }
    mockCalculator := &MockPriceCalculator{
        output: expectedOutput,
    }
    mockCache := &MockCacheManager{}

    service := NewQuoteService(mockPoolRepo, mockCalculator, mockCache)

    quote, err := service.GetQuote(ctx, "SOL", "USDC", 1000000000)

    assert.NoError(t, err)
    assert.Equal(t, expectedOutput, quote.OutputAmount)
}

// Test coverage: 80%+ (easy to test with mocks) ✅
```

### Single Responsibility Principle

**Each package has ONE job:**

| Package | Responsibility | Example |
|---------|----------------|---------|
| `domain/` | Define interfaces and models | `type PoolReader interface { ... }` |
| `repository/` | Data access (Redis, cache) | `GetPoolsByPair(...)` |
| `calculator/` | Business logic (pool math) | `CalculateQuote(pool, amount)` |
| `service/` | Orchestration | `GetQuote()` - coordinates repositories + calculators |
| `api/` | HTTP + gRPC handlers | Parse request, call service, return response |

**Benefits:**
- ✅ Easy to understand (each package is small and focused)
- ✅ Easy to test (inject dependencies via interfaces)
- ✅ Easy to change (modify one package without affecting others)
- ✅ Easy to extend (add new calculators, repositories, etc.)

---

## Technology Stack Decisions

### Final Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Quote Service** | **Go** | Fast delivery (2-3 weeks), proven, <10ms easily met, can reuse code |
| **Pool Discovery** | **Go** | I/O bound (RPC calls), Go perfect for concurrency |
| **RPC Proxy** | **Rust** | Shared by ALL services, worth investment, ideal for connection pooling |
| **HTTP + gRPC** | **Combined in ONE service** | Shared cache critical (4-7x faster), simpler deployment |

### Architecture Principles

1. **Clean Architecture** ✅
   - Domain layer (interfaces + models)
   - Service layer (business logic)
   - Repository layer (data access)
   - API layer (HTTP + gRPC handlers)

2. **Service Separation** ✅
   - Pool Discovery: Independent background job
   - Quote Service: Pure calculation + serving
   - RPC Proxy: Centralized RPC management

3. **Cache Strategy** ✅
   - Pool metadata: Redis (slow-changing, shared)
   - Quote cache: In-memory (fast, instance-local)
   - NO shared quote cache via Redis (defeats performance)

4. **Testing Strategy** ✅
   - Unit tests: >80% coverage (table-driven, mocks)
   - Integration tests: Real Redis, synthetic data
   - Load tests: 1000 req/s sustained

---

## Expected Improvements

### Performance Metrics

| Metric | Before | After (Clean) | After (HFT) | Improvement |
|--------|--------|---------------|-------------|-------------|
| **Quote Latency (cached)** | ~5ms | < 5ms | **< 5ms** ✅ | Same (already fast) |
| **Quote Latency (Shredstream)** | N/A | N/A | **< 5ms** ✅ | **NEW: 300-800ms advance** |
| **Quote Latency (uncached)** | ~200ms | < 50ms | < 50ms | **4x faster** |
| **NATS Event Publishing** | N/A | N/A | **< 1ms** ✅ | **NEW: 10K events/sec** |
| **Throughput** | 500 req/s | 10K req/s | **10K req/s** ✅ | **20x higher** |
| **Memory Usage** | 800MB | 300MB | 350MB | **56% reduction** |
| **Cache Hit Rate** | ~80% | ~90% | **> 95%** ✅ | **HFT: Critical** |

### HFT Pipeline Metrics (NEW)

| Stage | Service | Latency Target | Current | Status |
|-------|---------|----------------|---------|--------|
| **Stage 0** | Quote Service | **< 10ms** | 5-10ms | ✅ **HFT Ready** |
| Stage 1 | Scanner | < 50ms | TBD | 🚧 In Progress |
| Stage 2 | Planner | < 50ms | TBD | 🚧 In Progress |
| Stage 3 | Executor | < 90ms | TBD | 🚧 In Progress |
| **TOTAL** | **End-to-End** | **< 200ms** | **1.7s** | **8.5x improvement planned** |

### Code Quality Metrics

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Lines of Code** | 50K+ | 15K | **70% reduction** |
| **Test Coverage** | ~20% | > 80% | **4x better** |
| **Largest File** | 96KB | < 10KB | **90% reduction** |
| **Package Structure** | Monolithic | Clean architecture | **Excellent** |

### Maintainability Improvements

**Before:**
- ❌ Adding a new DEX protocol: Touch 5+ files, 200+ lines
- ❌ Fixing a bug: Search through 50K lines, unpredictable side effects
- ❌ Writing tests: Requires full infrastructure (Redis, NATS, RPC)
- ❌ Understanding code: Must read entire 96KB `cache.go`

**After:**
- ✅ Adding a new DEX protocol: Implement `Protocol` interface, register in DI (50 lines)
- ✅ Fixing a bug: Isolated in one package (100-200 lines to search)
- ✅ Writing tests: Unit tests with mocks (no infrastructure)
- ✅ Understanding code: Read one package at a time (500-1000 lines max)

---

## Conclusion: Building for the Future

### Why This Matters

Building trading systems is not just about **making it work today**—it's about **building for tomorrow**. The difference between a successful system and a failed one often comes down to **maintainability**.

**Bad architecture compounds:**
- Year 1: "It's a bit messy, but it works"
- Year 2: "Adding features is getting harder"
- Year 3: "We can't change anything without breaking something"
- Year 4: "We need to rewrite everything" ← Too late

**Good architecture scales:**
- Year 1: "Clean architecture takes more time upfront"
- Year 2: "Adding features is still easy"
- Year 3: "We can refactor safely with 80% test coverage"
- Year 4: "The system is maintainable and growing" ← Success

### The Investment

**Time Required:** 6 weeks
- Week 1-3: Parallel development (no disruption)
- Week 4: Canary testing (10% traffic)
- Week 5: Gradual rollout (10% → 100%)
- Week 6: Production hardening

**Risk:** Low (incremental, rollback-friendly)

**Outcome:** Production-ready, maintainable, performant quote service for the next 5+ years

### The Alternative

**If we don't rewrite:**
- Technical debt grows exponentially
- Adding features becomes impossible
- Bug fixes become dangerous
- Team velocity grinds to zero
- Eventually forced to rewrite under pressure (high risk)

**The choice is clear: Invest 6 weeks now, or pay 10x more later.**

### Merry Christmas! 🎄

As we close out 2025 and look toward 2026, I'm excited about this architectural evolution. Building robust, maintainable systems is what separates **hobby projects** from **production systems**.

**Here's to clean architecture, sustainable codebases, and successful trading in 2026!** 🎉

**Wishing everyone a Merry Christmas and a Happy New Year!** May your trades be profitable and your bugs be few! 🚀

---

## References

- [Quote Service Rewrite Plan (docs/26-QUOTE-SERVICE-REWRITE-PLAN.md)](https://github.com/guidebee/solana-trading-system)
- [Rust RPC Proxy Design (docs/25-RUST-RPC-PROXY-DESIGN.md)](https://github.com/guidebee/solana-trading-system)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Go Project Layout Standards](https://github.com/golang-standards/project-layout)

---

**Next Post:** Quote Service Rewrite - Phase 1 Implementation (Foundation Skeleton)

**Stay tuned for the journey from architectural debt to clean, maintainable code!** 🎄