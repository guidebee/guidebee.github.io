---
layout: single
title: "Shredstream Integration Architecture"
permalink: /docs/17-shredstream-architecture-design/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Shredstream Integration Architecture

**Document Version:** 1.0
**Date:** 2025-12-17
**Author:** Solution Architecture
**Status:** RECOMMENDED DESIGN

---

## Executive Summary

This document defines the production architecture for integrating Jito Shredstream into the Solana HFT trading system. After evaluating three architectural patterns, we recommend a **hybrid microservice approach** that separates concerns while minimizing latency.

**Key Decision:** Shredstream Scanner as a separate TypeScript service publishing to NATS, with Go Quote Service subscribing for pool state updates.

**Expected Impact:**
- 300-800ms head start on pool price changes
- 1.7s → 800ms execution time (Week 1)
- Foundation for <200ms target (Week 4)

---

## Table of Contents

1. [Architectural Options Analysis](#1-architectural-options-analysis)
2. [Recommended Architecture](#2-recommended-architecture)
3. [Component Specifications](#3-component-specifications)
4. [Data Flow & Integration](#4-data-flow--integration)
5. [Implementation Roadmap](#5-implementation-roadmap)
6. [Performance Considerations](#6-performance-considerations)
7. [Deployment Architecture](#7-deployment-architecture)
8. [Monitoring & Observability](#8-monitoring--observability)
9. [Risk Analysis & Mitigation](#9-risk-analysis--mitigation)
10. [Migration from Existing Code](#10-migration-from-existing-code)
11. [Success Criteria](#11-success-criteria)
12. [Troubleshooting Guide](#12-troubleshooting-guide)

---

## 1. Architectural Options Analysis

### Option A: Separate Shredstream Service (Microservice)

```
┌─────────────────────────────────────┐
│   Shredstream Scanner Service       │
│   (TypeScript/Node.js)              │
│                                     │
│   - gRPC to Jito Shredstream       │
│   - Entry parsing & filtering       │
│   - Pool state extraction           │
└──────────────┬──────────────────────┘
               │ NATS Events
               │ (pool.state.updated)
               ▼
┌─────────────────────────────────────┐
│   Quote Service (Go)                │
│                                     │
│   - Subscribes to NATS events       │
│   - Updates in-memory pool cache    │
│   - Computes quotes                 │
│   - HTTP API                        │
└─────────────────────────────────────┘
```

**Pros:**
- ✅ **Separation of concerns** - Scanner does one thing well
- ✅ **Language optimization** - TypeScript for I/O, Go for compute
- ✅ **Independent scaling** - Scale scanner separately
- ✅ **Failure isolation** - Scanner crash doesn't kill quotes
- ✅ **Follows Scanner→Planner→Executor pattern** - Architecture consistency
- ✅ **Easy testing** - Mock NATS events for testing
- ✅ **Future flexibility** - Can replace scanner with Rust later

**Cons:**
- ⚠️ **Network hop** - NATS adds 1-5ms latency (acceptable for 400ms alpha)
- ⚠️ **Operational complexity** - Two services to deploy/monitor
- ⚠️ **Serialization overhead** - Event encoding/decoding

**Latency Analysis:**
```
Shredstream → Scanner → NATS → Quote Service → Pool Cache Update
   50ms        20ms      2ms       5ms            10ms
                        ↑ Added latency
Total: ~87ms (still 5-10x faster than WebSocket RPC)
```

---

### Option B: Integrated into Quote Service (Monolithic)

```
┌─────────────────────────────────────────────┐
│   Quote Service (Go)                        │
│                                             │
│   ┌─────────────────────────────────────┐  │
│   │ Shredstream Client (gRPC)           │  │
│   │  - Connect to Jito Shredstream      │  │
│   │  - Entry parsing (Go implementation)│  │
│   └────────────┬────────────────────────┘  │
│                │ Direct memory access       │
│                ▼                            │
│   ┌─────────────────────────────────────┐  │
│   │ Pool State Cache (in-memory)        │  │
│   │  - Lock-free updates                │  │
│   │  - Concurrent reads                 │  │
│   └────────────┬────────────────────────┘  │
│                │                            │
│                ▼                            │
│   ┌─────────────────────────────────────┐  │
│   │ Quote Engine                        │  │
│   │  - Best pool selection              │  │
│   │  - Price calculations               │  │
│   └─────────────────────────────────────┘  │
│                                             │
│   HTTP API (Port 8080)                     │
└─────────────────────────────────────────────┘
```

**Pros:**
- ✅ **Zero network latency** - Direct memory access
- ✅ **Atomic updates** - Pool state and quotes stay in sync
- ✅ **Simpler deployment** - Single binary
- ✅ **Lower operational overhead** - One service to monitor
- ✅ **Easier debugging** - Everything in one codebase

**Cons:**
- ❌ **Go gRPC streaming complexity** - Go's gRPC client is verbose
- ❌ **Entry parsing in Go** - Need to rewrite TypeScript parser
- ❌ **Tight coupling** - Scanner logic mixed with quote logic
- ❌ **Single point of failure** - One crash kills both functions
- ❌ **Harder to iterate** - Go compilation slower than TS for experiments
- ❌ **Scaling challenges** - Can't scale scanner independently

**Development Effort:**
- Rewrite entry parser in Go: ~3-4 days
- Integrate gRPC streaming: ~2 days
- Testing & debugging: ~2-3 days
- **Total: 7-9 days** vs 2-3 days for Option A

---

### Option C: Hybrid with Shared Cache (Sidecar Pattern)

```
┌────────────────────────────┐
│  Shredstream Scanner (TS)  │
│  - gRPC to Shredstream     │
│  - Entry parsing           │
└────────┬───────────────────┘
         │ Updates shared Redis
         ▼
┌────────────────────────────┐
│  Redis (Pool State Cache)  │  ← Shared state
└────────┬───────────────────┘
         │ Reads from Redis
         ▼
┌────────────────────────────┐
│  Quote Service (Go)        │
│  - Read pool states        │
│  - Compute quotes          │
└────────────────────────────┘
```

**Pros:**
- ✅ **Language independence** - Scanner and Quote Service decoupled
- ✅ **Shared state** - Both services access same data
- ✅ **Persistence** - Redis provides durability

**Cons:**
- ❌ **Redis network latency** - 0.5-1ms per read/write
- ❌ **Serialization overhead** - Marshal/unmarshal pool state
- ❌ **Redis as bottleneck** - Single point of contention
- ❌ **Stale reads** - Eventually consistent
- ❌ **Higher complexity** - Another infrastructure dependency

**Latency Analysis:**
```
Shredstream → Scanner → Redis Write → Quote Service → Redis Read → Quote
   50ms        20ms       1ms           5ms            1ms         10ms
                          ↑ Added 2ms round-trip

Total: ~87ms (similar to Option A but more complex)
```

---

## 2. Recommended Architecture

**RECOMMENDED: Option A - Separate Shredstream Scanner with NATS Integration**

### Rationale

| Criteria | Weight | Option A | Option B | Option C | Winner |
|----------|--------|----------|----------|----------|--------|
| **Time to market** | HIGH | ✅ 2-3 days | ❌ 7-9 days | ⚠️ 4-5 days | **A** |
| **Latency** | HIGH | 87ms | 85ms ✅ | 88ms | B (marginal) |
| **Operational complexity** | MED | ⚠️ Moderate | ✅ Low | ❌ High | B |
| **Failure isolation** | HIGH | ✅ Good | ❌ Poor | ✅ Good | **A/C** |
| **Architecture consistency** | MED | ✅ Matches pattern | ❌ Breaks pattern | ⚠️ Sidecar | **A** |
| **Future flexibility** | MED | ✅ High | ❌ Low | ✅ High | **A/C** |
| **Development velocity** | HIGH | ✅ Fast (TS) | ❌ Slow (Go) | ⚠️ Medium | **A** |

**Decision:** Option A wins on **time to market**, **architecture consistency**, and **development velocity**.

The 2ms NATS latency penalty is **negligible** compared to the 300-800ms alpha gained from Shredstream. We're optimizing for 1.7s → 800ms, not 100ms → 98ms.

---

## 3. Component Specifications

### 3.1 Shredstream Scanner Service (NEW)

**Technology:** TypeScript/Node.js
**Location:** `ts/apps/shredstream-scanner/`
**Purpose:** Connect to Jito Shredstream, parse entries, extract pool state changes, publish to NATS

#### Core Responsibilities

1. **Shredstream Connection Management**
   - Maintain gRPC streaming connection
   - Handle reconnection with exponential backoff
   - Send heartbeats if required

2. **Entry Processing Pipeline**
   - Receive entry data from Shredstream
   - Parse `Vec<Entry>` binary format
   - Extract transaction signatures
   - Filter for monitored pool accounts

3. **Pool State Extraction**
   - Decode pool account data from transactions
   - Support multiple DEX protocols (Raydium, Meteora, Orca)
   - Extract: reserves, fees, last update slot

4. **Event Publishing**
   - Publish pool state updates to NATS
   - Use NATS JetStream for guaranteed delivery
   - Event schema: `pool.state.updated.{dex}.{pool_address}`

#### Configuration

```typescript
interface ScannerConfig {
  // Shredstream connection
  shredstream: {
    host: string;           // "10.1.1.160"
    port: number;           // 50051
    regions: string[];      // ["ny", "amsterdam"]
    enableHeartbeat: boolean;
    heartbeatIp?: string;
  };

  // Pool monitoring
  pools: {
    monitored: string[];    // Top 20-50 pool addresses
    dexPrograms: {
      raydium_amm: string;
      raydium_clmm: string;
      meteora_dlmm: string;
      orca_whirlpool: string;
    };
  };

  // NATS configuration
  nats: {
    servers: string[];      // ["nats://localhost:4222"]
    streamName: string;     // "POOL_UPDATES"
    maxAge: number;         // 60 seconds (don't persist old updates)
  };

  // Performance tuning
  performance: {
    maxConcurrentProcessing: number;  // 20
    batchSize: number;                // 10 transactions per batch
    queueSizeLimit: number;           // 100
  };

  // Monitoring
  metrics: {
    prometheusPort: number;  // 9091
    logLevel: string;        // "info"
  };
}
```

#### Event Schema

```typescript
interface PoolStateUpdateEvent {
  // Event metadata
  eventId: string;          // UUID
  timestamp: number;        // Unix timestamp (ms)
  source: "shredstream";
  latency: number;          // ms from shredstream to publish

  // Pool identification
  pool: {
    address: string;        // Pool account address
    dex: string;            // "raydium_amm" | "raydium_clmm" | "meteora_dlmm"
    tokenA: string;         // Token A mint
    tokenB: string;         // Token B mint
  };

  // State data
  state: {
    slot: string;           // Slot number where state changed
    reserveA: string;       // Token A reserves (bigint as string)
    reserveB: string;       // Token B reserves (bigint as string)
    feeNumerator?: string;  // Fee numerator (if available)
    feeDenominator?: string; // Fee denominator (if available)
    sqrtPrice?: string;     // CLMM: sqrt price (Q64.64)
    tick?: number;          // CLMM: current tick
    rawData?: string;       // Base64 encoded raw account data (optional)
  };

  // Transaction context
  transaction: {
    signature: string;      // Transaction signature
    confirmed: boolean;     // false = pending, true = executed
  };
}
```

#### API Endpoints

```
GET  /health          - Health check
GET  /metrics         - Prometheus metrics
GET  /stats           - Scanner statistics
POST /config/pools    - Update monitored pools list
GET  /config          - Get current configuration
```

#### Metrics Exposed

```typescript
// Prometheus metrics
const metrics = {
  // Throughput
  entries_received_total: Counter,
  transactions_processed_total: Counter,
  events_published_total: Counter,

  // Latency
  shredstream_latency_ms: Histogram,      // Shredstream → Scanner
  processing_latency_ms: Histogram,       // Scanner → NATS publish
  end_to_end_latency_ms: Histogram,       // Shredstream → NATS

  // Errors
  parsing_errors_total: Counter,
  nats_publish_errors_total: Counter,
  connection_errors_total: Counter,

  // Queue stats
  processing_queue_size: Gauge,
  dropped_transactions_total: Counter,
};
```

---

### 3.2 Quote Service Integration (UPDATED)

**Technology:** Go
**Location:** `go/cmd/quote-service/`
**Purpose:** Subscribe to pool state updates, maintain in-memory cache, serve quote API

#### New Responsibilities

1. **NATS Subscription**
   - Subscribe to `pool.state.updated.*` events
   - Deserialize events from JSON
   - Update in-memory pool cache

2. **Pool State Cache Management**
   - Maintain thread-safe pool state map
   - Track last update slot per pool
   - Evict stale data (>60s old)

3. **Hybrid Quote Strategy**
   - **Primary:** Use cached pool states from Shredstream
   - **Fallback:** Query RPC if cache miss or stale
   - **Validation:** Periodically verify cache accuracy

#### Code Structure

```
go/cmd/quote-service/
├── main.go                    # Entry point
├── config.go                  # Configuration
├── nats_subscriber.go         # NEW: NATS integration
├── pool_cache.go              # NEW: Thread-safe cache
├── quote_handler.go           # HTTP handlers (updated)
└── metrics.go                 # Prometheus metrics

go/pkg/
├── router/                    # Existing routing logic
├── pool/                      # Existing pool implementations
└── nats/                      # NEW: NATS utilities
    ├── subscriber.go
    └── events.go              # Event type definitions
```

#### Updated Configuration

```go
type Config struct {
    // Existing fields
    Port           int
    RefreshSeconds int
    SlippageBps    int
    RateLimitRPS   int
    RPCEndpoints   []string

    // NEW: NATS configuration
    NATS struct {
        Servers      []string // ["nats://localhost:4222"]
        StreamName   string   // "POOL_UPDATES"
        ConsumerName string   // "quote-service"
        MaxAge       int      // 60 seconds
    }

    // NEW: Cache configuration
    Cache struct {
        Enabled         bool   // true = use Shredstream cache
        StalenessThreshold int // 30 seconds
        MaxPoolCount    int    // 1000 pools
        EvictionInterval int   // 60 seconds
    }

    // NEW: Hybrid strategy
    HybridMode struct {
        Enabled           bool    // true = use cache + RPC fallback
        CacheFirstTimeout int     // 10ms - timeout before RPC fallback
        ValidateInterval  int     // 300 seconds - verify cache accuracy
    }
}
```

#### Pool Cache Implementation

```go
// go/cmd/quote-service/pool_cache.go
package main

import (
    "sync"
    "time"
)

type PoolState struct {
    Address        string
    Dex            string
    TokenA         string
    TokenB         string
    ReserveA       *big.Int
    ReserveB       *big.Int
    Slot           uint64
    LastUpdated    time.Time
    FeeNumerator   uint64
    FeeDenominator uint64
}

type PoolCache struct {
    mu     sync.RWMutex
    pools  map[string]*PoolState // key: pool address
    config CacheConfig
}

type CacheConfig struct {
    StalenessThreshold time.Duration
    MaxPoolCount       int
    EvictionInterval   time.Duration
}

func NewPoolCache(config CacheConfig) *PoolCache {
    cache := &PoolCache{
        pools:  make(map[string]*PoolState),
        config: config,
    }

    // Start background eviction
    go cache.evictStaleEntries()

    return cache
}

// Update pool state from Shredstream event
func (c *PoolCache) Update(state *PoolState) {
    c.mu.Lock()
    defer c.mu.Unlock()

    existing, exists := c.pools[state.Address]

    // Only update if newer slot
    if exists && existing.Slot >= state.Slot {
        return
    }

    state.LastUpdated = time.Now()
    c.pools[state.Address] = state
}

// Get pool state (returns nil if not found or stale)
func (c *PoolCache) Get(address string) *PoolState {
    c.mu.RLock()
    defer c.mu.RUnlock()

    state, exists := c.pools[address]
    if !exists {
        return nil
    }

    // Check staleness
    if time.Since(state.LastUpdated) > c.config.StalenessThreshold {
        return nil
    }

    return state
}

// Background eviction of stale entries
func (c *PoolCache) evictStaleEntries() {
    ticker := time.NewTicker(c.config.EvictionInterval)
    defer ticker.Stop()

    for range ticker.C {
        c.mu.Lock()
        now := time.Now()
        for addr, state := range c.pools {
            if now.Sub(state.LastUpdated) > c.config.StalenessThreshold {
                delete(c.pools, addr)
            }
        }
        c.mu.Unlock()
    }
}

// Stats for monitoring
func (c *PoolCache) Stats() map[string]interface{} {
    c.mu.RLock()
    defer c.mu.RUnlock()

    fresh := 0
    stale := 0
    now := time.Now()

    for _, state := range c.pools {
        if now.Sub(state.LastUpdated) <= c.config.StalenessThreshold {
            fresh++
        } else {
            stale++
        }
    }

    return map[string]interface{}{
        "total_pools": len(c.pools),
        "fresh_pools": fresh,
        "stale_pools": stale,
    }
}
```

#### NATS Subscriber Implementation

```go
// go/cmd/quote-service/nats_subscriber.go
package main

import (
    "context"
    "encoding/json"
    "log"
    "time"

    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

type NATSSubscriber struct {
    nc        *nats.Conn
    js        jetstream.JetStream
    cache     *PoolCache
    config    NATSConfig
    metrics   *Metrics
}

type NATSConfig struct {
    Servers      []string
    StreamName   string
    ConsumerName string
}

func NewNATSSubscriber(config NATSConfig, cache *PoolCache, metrics *Metrics) (*NATSSubscriber, error) {
    // Connect to NATS
    nc, err := nats.Connect(strings.Join(config.Servers, ","))
    if err != nil {
        return nil, fmt.Errorf("failed to connect to NATS: %w", err)
    }

    // Get JetStream context
    js, err := jetstream.New(nc)
    if err != nil {
        return nil, fmt.Errorf("failed to create JetStream context: %w", err)
    }

    return &NATSSubscriber{
        nc:      nc,
        js:      js,
        cache:   cache,
        config:  config,
        metrics: metrics,
    }, nil
}

// Start subscribing to pool state updates
func (s *NATSSubscriber) Start(ctx context.Context) error {
    log.Printf("Starting NATS subscriber for stream: %s", s.config.StreamName)

    // Create consumer
    cons, err := s.js.CreateOrUpdateConsumer(ctx, s.config.StreamName, jetstream.ConsumerConfig{
        Name:          s.config.ConsumerName,
        Durable:       s.config.ConsumerName,
        FilterSubject: "pool.state.updated.*",
        AckPolicy:     jetstream.AckExplicitPolicy,
        MaxAckPending: 100,
    })
    if err != nil {
        return fmt.Errorf("failed to create consumer: %w", err)
    }

    // Consume messages
    _, err = cons.Consume(func(msg jetstream.Msg) {
        s.handleMessage(msg)
    })
    if err != nil {
        return fmt.Errorf("failed to start consuming: %w", err)
    }

    log.Printf("NATS subscriber started successfully")
    return nil
}

// Handle incoming pool state update message
func (s *NATSSubscriber) handleMessage(msg jetstream.Msg) {
    startTime := time.Now()

    // Parse event
    var event PoolStateUpdateEvent
    if err := json.Unmarshal(msg.Data(), &event); err != nil {
        log.Printf("Failed to unmarshal event: %v", err)
        s.metrics.NATSParseErrors.Inc()
        msg.Ack()
        return
    }

    // Convert to PoolState
    reserveA := new(big.Int)
    reserveA.SetString(event.State.ReserveA, 10)

    reserveB := new(big.Int)
    reserveB.SetString(event.State.ReserveB, 10)

    slot, _ := strconv.ParseUint(event.State.Slot, 10, 64)

    poolState := &PoolState{
        Address:     event.Pool.Address,
        Dex:         event.Pool.Dex,
        TokenA:      event.Pool.TokenA,
        TokenB:      event.Pool.TokenB,
        ReserveA:    reserveA,
        ReserveB:    reserveB,
        Slot:        slot,
        LastUpdated: time.Now(),
    }

    // Update cache
    s.cache.Update(poolState)

    // Metrics
    s.metrics.PoolUpdatesReceived.Inc()
    s.metrics.NATSProcessingLatency.Observe(time.Since(startTime).Seconds())

    // Ack message
    msg.Ack()
}

// Close subscriber
func (s *NATSSubscriber) Close() {
    if s.nc != nil {
        s.nc.Close()
    }
}
```

#### Hybrid Quote Logic

```go
// go/cmd/quote-service/quote_handler.go (UPDATED)
func (s *Server) getQuote(inputMint, outputMint string, amount uint64) (*Quote, error) {
    startTime := time.Now()

    // Strategy 1: Try cache first (Shredstream data)
    if s.config.Cache.Enabled {
        quote, err := s.getQuoteFromCache(inputMint, outputMint, amount)
        if err == nil {
            s.metrics.QuoteCacheHits.Inc()
            s.metrics.QuoteLatency.WithLabelValues("cache").Observe(time.Since(startTime).Seconds())
            return quote, nil
        }
        s.metrics.QuoteCacheMisses.Inc()
    }

    // Strategy 2: Fallback to RPC
    quote, err := s.getQuoteFromRPC(inputMint, outputMint, amount)
    if err != nil {
        return nil, err
    }

    s.metrics.QuoteLatency.WithLabelValues("rpc").Observe(time.Since(startTime).Seconds())
    return quote, nil
}

// Get quote using cached pool states
func (s *Server) getQuoteFromCache(inputMint, outputMint string, amount uint64) (*Quote, error) {
    // Find pools for this token pair
    pools := s.findPoolsForPair(inputMint, outputMint)
    if len(pools) == 0 {
        return nil, fmt.Errorf("no pools found in cache")
    }

    var bestQuote *Quote
    var bestOutputAmount uint64

    for _, poolAddr := range pools {
        // Get cached pool state
        poolState := s.cache.Get(poolAddr)
        if poolState == nil {
            continue // Skip if not in cache or stale
        }

        // Calculate quote using cached state (no RPC call!)
        outputAmount := calculateSwapOutput(
            amount,
            poolState.ReserveA,
            poolState.ReserveB,
            poolState.FeeNumerator,
            poolState.FeeDenominator,
        )

        if outputAmount > bestOutputAmount {
            bestOutputAmount = outputAmount
            bestQuote = &Quote{
                InputMint:    inputMint,
                OutputMint:   outputMint,
                InputAmount:  amount,
                OutputAmount: outputAmount,
                Pool:         poolAddr,
                Dex:          poolState.Dex,
                PriceImpact:  calculatePriceImpact(amount, poolState.ReserveA),
                Source:       "cache",
            }
        }
    }

    if bestQuote == nil {
        return nil, fmt.Errorf("no valid quotes from cache")
    }

    return bestQuote, nil
}
```

---

## 4. Data Flow & Integration

### 4.1 End-to-End Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         JITO SHREDSTREAM                            │
│                    (Slot notifications + Entry data)                │
└────────────────────────────┬────────────────────────────────────────┘
                             │ gRPC Stream
                             │ 50-200ms latency
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│               SHREDSTREAM SCANNER SERVICE (TypeScript)              │
│                                                                     │
│  1. Receive entry data from Shredstream                            │
│  2. Parse Vec<Entry> binary format                                 │
│  3. Filter for monitored pool accounts (20-50 pools)               │
│  4. Extract signatures for relevant transactions                   │
│  5. Decode pool state from account data:                           │
│     - Raydium AMM: reserves, fees                                  │
│     - Raydium CLMM: sqrt_price, tick, liquidity                    │
│     - Meteora DLMM: bins, active bin                               │
│  6. Construct PoolStateUpdateEvent                                 │
│  7. Publish to NATS JetStream                                      │
│                                                                     │
│  Latency: ~20ms (parsing + publishing)                             │
└────────────────────────────┬───────────────────────────────────────┘
                             │ NATS JetStream
                             │ Subject: pool.state.updated.{dex}.{pool}
                             │ 1-2ms latency
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   QUOTE SERVICE (Go)                                │
│                                                                     │
│  1. Subscribe to pool.state.updated.* events                       │
│  2. Deserialize PoolStateUpdateEvent                               │
│  3. Update in-memory pool cache (thread-safe)                      │
│  4. Serve HTTP quote requests:                                     │
│     a. Check cache for pool states (< 1ms)                         │
│     b. Calculate quote using cached reserves (< 5ms)               │
│     c. Fallback to RPC if cache miss (100-200ms)                   │
│                                                                     │
│  Quote Latency:                                                    │
│    - Cache hit: 5-10ms                                             │
│    - Cache miss: 100-200ms (RPC fallback)                          │
└────────────────────────────┬───────────────────────────────────────┘
                             │ HTTP/JSON
                             │ GET /quote
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   ARBITRAGE BOT (TypeScript)                        │
│                                                                     │
│  1. Receive pool.state.updated event (OPTIONAL - direct sub)       │
│  2. Request quote from Quote Service                               │
│  3. Check arbitrage opportunity                                    │
│  4. Execute trade if profitable                                    │
│                                                                     │
│  Total Latency: ~100-150ms (vs 1.7s before)                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Event Flow Diagram

```
TIME: T+0ms    │ Large swap hits Solana blockchain
               │
TIME: T+50ms   │ Shredstream broadcasts entry data
               │
TIME: T+70ms   │ Scanner parses and publishes NATS event
               │
TIME: T+72ms   │ Quote Service receives event and updates cache
               │
TIME: T+72ms   │ Arbitrage bot can now get FRESH quotes in 5-10ms
               │
TIME: T+500ms  │ WebSocket RPC finally notifies (traditional bots)
               │
               ▼
         YOUR BOT HAS 400ms HEAD START
```

### 4.3 Cache Consistency Strategy

**Challenge:** Pool state can change between Shredstream notification and actual block confirmation.

**Solution:** Track slot numbers and use latest data

```go
// Quote Service cache update logic
func (c *PoolCache) Update(newState *PoolState) {
    c.mu.Lock()
    defer c.mu.Unlock()

    existing, exists := c.pools[newState.Address]

    if !exists {
        // New pool - add to cache
        c.pools[newState.Address] = newState
        return
    }

    // Only update if newer slot
    if newState.Slot > existing.Slot {
        c.pools[newState.Address] = newState
    }

    // Log if we received out-of-order update
    if newState.Slot < existing.Slot {
        log.Printf("Received stale update for pool %s: slot %d < %d",
            newState.Address, newState.Slot, existing.Slot)
    }
}
```

### 4.4 Failure Scenarios & Recovery

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| **Shredstream disconnects** | Scanner stops receiving data | Auto-reconnect with exponential backoff; Alert on >10s disconnect |
| **NATS down** | Events not delivered | JetStream persistence; Events queued for 60s; Fallback to RPC in Quote Service |
| **Scanner crashes** | No new pool updates | Kubernetes auto-restart; Quote Service continues serving cached data; Stale cache timeout triggers RPC fallback |
| **Quote Service crashes** | No quotes available | Stateless service - restart with empty cache; Refills cache within 30s |
| **Network partition** | Scanner isolated from NATS | Circuit breaker; Health check fails; Alert triggers |

---

## 5. Implementation Roadmap

### Phase 1: Core Infrastructure (Days 1-2)

#### Day 1: Shredstream Scanner Setup

- [ ] **1.1 Create Project Structure**
  ```bash
  cd ts/apps
  mkdir shredstream-scanner
  cd shredstream-scanner
  pnpm init
  ```

- [ ] **1.2 Setup TypeScript Configuration**
  ```json
  // tsconfig.json
  {
    "extends": "../../tsconfig.json",
    "compilerOptions": {
      "outDir": "./dist",
      "rootDir": "./src"
    },
    "include": ["src/**/*"]
  }
  ```

- [ ] **1.3 Install Dependencies**
  ```bash
  pnpm add @grpc/grpc-js @grpc/proto-loader
  pnpm add nats
  pnpm add bs58 uuid
  pnpm add winston
  pnpm add prom-client
  pnpm add -D @types/node @types/uuid
  ```

- [ ] **1.4 Copy and Refactor Existing Code**
  ```bash
  # Copy services
  cp ../../references/apps/cli-tools/services/shredstreamService.ts src/
  cp ../../references/apps/cli-tools/services/entryParserService.ts src/

  # DON'T copy shredstreamRpcBridge.ts - we're deleting this pattern
  ```

- [ ] **1.5 Create NATS Publisher**
  - [ ] Create `src/natsPublisher.ts`
  - [ ] Implement `NATSPublisher` class with JetStream support
  - [ ] Add connection management and error handling
  - [ ] Test NATS publishing locally

- [ ] **1.6 Create Configuration Management**
  - [ ] Create `src/config.ts`
  - [ ] Define `ScannerConfig` interface
  - [ ] Load from environment variables
  - [ ] Add validation

- [ ] **1.7 Create Event Schema**
  - [ ] Create `src/types/events.ts`
  - [ ] Define `PoolStateUpdateEvent` interface
  - [ ] Add validation helpers

#### Day 2: Scanner Service Logic

- [ ] **2.1 Implement Main Scanner Logic**
  - [ ] Create `src/scanner.ts`
  - [ ] Implement filtered entry processing
  - [ ] Use `extractSignaturesForAccounts()` with pool whitelist
  - [ ] Add batch processing with `Promise.allSettled()`

- [ ] **2.2 Create Pool Decoders (Start with Raydium AMM)**
  - [ ] Create `src/poolDecoders/raydiumAmm.ts`
  - [ ] Implement `decodeRaydiumAmmPoolState()`
  - [ ] Test with real pool account data
  - [ ] Document account structure offsets

- [ ] **2.3 Add Health Check Endpoint**
  - [ ] Create `src/healthCheck.ts`
  - [ ] Implement Express server on port 8081
  - [ ] Endpoints: `/health`, `/stats`, `/config`
  - [ ] Add connection status checks

- [ ] **2.4 Add Prometheus Metrics**
  - [ ] Create `src/metrics.ts`
  - [ ] Implement metrics from design doc
  - [ ] Expose on port 9091
  - [ ] Test with Prometheus scraping

- [ ] **2.5 Create Main Entry Point**
  - [ ] Create `src/main.ts`
  - [ ] Wire up all components
  - [ ] Add graceful shutdown handling
  - [ ] Add error logging

- [ ] **2.6 Add Configuration Files**
  - [ ] Create `config/monitored-pools.json` with top 20 pools
  - [ ] Create `.env.example`
  - [ ] Document configuration options

- [ ] **2.7 Test Locally**
  ```bash
  # Terminal 1: Start NATS
  docker run -p 4222:4222 nats:latest --jetstream

  # Terminal 2: Run scanner
  pnpm dev

  # Terminal 3: Subscribe to events
  nats sub "pool.state.updated.*"
  ```

- [ ] **2.8 Write Unit Tests**
  - [ ] Test entry parser with sample data
  - [ ] Test pool decoders
  - [ ] Test NATS publishing
  - [ ] Test error handling

**Deliverable:** Standalone scanner service that publishes events
**Time Estimate:** 2 days

---

### Phase 2: Quote Service Integration (Days 3-4)

#### Day 3: Go NATS Integration

- [ ] **3.1 Add NATS Client to Go Project**
  ```bash
  cd go
  go get github.com/nats-io/nats.go
  ```

- [ ] **3.2 Create NATS Package**
  - [ ] Create `go/pkg/nats/subscriber.go`
  - [ ] Create `go/pkg/nats/events.go` (event type definitions)
  - [ ] Implement JetStream consumer
  - [ ] Add connection management

- [ ] **3.3 Implement Pool Cache**
  - [ ] Create `go/cmd/quote-service/pool_cache.go`
  - [ ] Implement thread-safe cache with `sync.RWMutex`
  - [ ] Add `Update()`, `Get()`, `Stats()` methods
  - [ ] Implement background eviction goroutine
  - [ ] Add slot-based consistency checks

- [ ] **3.4 Create NATS Subscriber**
  - [ ] Create `go/cmd/quote-service/nats_subscriber.go`
  - [ ] Implement `NATSSubscriber` struct
  - [ ] Handle incoming events
  - [ ] Update pool cache
  - [ ] Add error handling and metrics

- [ ] **3.5 Update Configuration**
  - [ ] Update `go/cmd/quote-service/config.go`
  - [ ] Add NATS configuration section
  - [ ] Add cache configuration section
  - [ ] Add hybrid mode settings
  - [ ] Update `.env.example`

#### Day 4: Hybrid Quote Logic

- [ ] **4.1 Implement Hybrid Quote Strategy**
  - [ ] Update `go/cmd/quote-service/quote_handler.go`
  - [ ] Implement `getQuoteFromCache()`
  - [ ] Keep existing `getQuoteFromRPC()` as fallback
  - [ ] Add cache hit/miss metrics
  - [ ] Implement timeout for cache queries

- [ ] **4.2 Add Circuit Breaker**
  - [ ] Create `go/cmd/quote-service/circuit_breaker.go`
  - [ ] Implement circuit breaker pattern
  - [ ] Add to cache query logic
  - [ ] Add metrics for circuit breaker state

- [ ] **4.3 Update Health Check**
  - [ ] Add cache statistics to `/health` endpoint
  - [ ] Show NATS connection status
  - [ ] Display cache hit rate
  - [ ] Show fresh/stale pool counts

- [ ] **4.4 Add Cache Metrics**
  - [ ] Update `go/cmd/quote-service/metrics.go`
  - [ ] Add cache hit/miss counters
  - [ ] Add cache size gauge
  - [ ] Add NATS processing latency histogram
  - [ ] Add pool update counter

- [ ] **4.5 Update Main Function**
  - [ ] Update `go/cmd/quote-service/main.go`
  - [ ] Initialize pool cache
  - [ ] Start NATS subscriber
  - [ ] Add graceful shutdown
  - [ ] Add startup validation

- [ ] **4.6 Test Integration**
  ```bash
  # Terminal 1: NATS + Scanner
  docker-compose up -d nats
  cd ts/apps/shredstream-scanner
  pnpm dev

  # Terminal 2: Quote Service
  cd go/cmd/quote-service
  go run .

  # Terminal 3: Test quotes
  curl "http://localhost:8080/quote?input=So11...&output=EPj...&amount=1000000"
  curl "http://localhost:8080/health"
  ```

- [ ] **4.7 Write Integration Tests**
  - [ ] Test end-to-end flow (Scanner → NATS → Quote Service)
  - [ ] Test cache hit scenario
  - [ ] Test cache miss fallback
  - [ ] Test stale cache handling
  - [ ] Test concurrent requests

**Deliverable:** Quote Service serving quotes from cached pool states
**Time Estimate:** 2 days

---

### Phase 3: Pool Decoders (Days 5-6)

#### Day 5: Additional DEX Support

- [ ] **5.1 Raydium CLMM Decoder**
  - [ ] Create `src/poolDecoders/raydiumClmm.ts`
  - [ ] Decode: sqrt_price, tick, liquidity, token vaults
  - [ ] Test with real CLMM pool data
  - [ ] Integrate into scanner

- [ ] **5.2 Orca Whirlpool Decoder**
  - [ ] Create `src/poolDecoders/orcaWhirlpool.ts`
  - [ ] Decode: sqrt_price, tick, liquidity
  - [ ] Test with real Whirlpool data
  - [ ] Integrate into scanner

#### Day 6: Meteora and Optimization

- [ ] **6.1 Meteora DLMM Decoder**
  - [ ] Create `src/poolDecoders/meteoraDlmm.ts`
  - [ ] Decode: active bin, bins, reserves
  - [ ] Test with real DLMM data
  - [ ] Integrate into scanner

- [ ] **6.2 Create Decoder Registry**
  - [ ] Create `src/poolDecoders/registry.ts`
  - [ ] Map program IDs to decoders
  - [ ] Add fallback handling
  - [ ] Add error logging for unknown formats

- [ ] **6.3 Optimize Entry Processing**
  - [ ] Profile entry parsing with Chrome DevTools
  - [ ] Optimize buffer operations (reuse buffers)
  - [ ] Minimize allocations in hot path
  - [ ] Add performance metrics

- [ ] **6.4 Update Monitored Pools List**
  - [ ] Expand from 20 → 50 pools
  - [ ] Include: Raydium (AMM/CLMM), Orca, Meteora
  - [ ] Prioritize by volume
  - [ ] Test with expanded list

**Deliverable:** Scanner extracting pool states for top 4 DEXes
**Time Estimate:** 2 days

---

### Phase 4: Testing & Validation (Day 7)

- [ ] **7.1 Deploy to Staging**
  ```bash
  cd deployment/docker
  docker-compose -f docker-compose.staging.yml up -d
  ```

- [ ] **7.2 Load Testing**
  - [ ] Simulate high entry rate (1000+ entries/sec)
  - [ ] Test quote service with 100+ req/sec
  - [ ] Monitor CPU and memory usage
  - [ ] Check for memory leaks

- [ ] **7.3 Latency Testing**
  - [ ] Measure end-to-end latency (Shredstream → Quote)
  - [ ] Target: <100ms P95
  - [ ] Measure cache hit rate
  - [ ] Target: >80%

- [ ] **7.4 Accuracy Validation**
  - [ ] Compare cache quotes vs RPC quotes
  - [ ] Acceptable delta: <1% due to slot lag
  - [ ] Log discrepancies for analysis
  - [ ] Tune staleness threshold if needed

- [ ] **7.5 Failure Testing**
  - [ ] Test Shredstream disconnect/reconnect
  - [ ] Test NATS outage (JetStream persistence)
  - [ ] Test scanner crash (Kubernetes restart)
  - [ ] Test quote service crash (empty cache recovery)
  - [ ] Test network partition

- [ ] **7.6 Monitoring Setup**
  - [ ] Import Grafana dashboard from design doc
  - [ ] Configure Prometheus alerts
  - [ ] Test alert firing (manually trigger conditions)
  - [ ] Document alert response procedures

- [ ] **7.7 Write Runbooks**
  - [ ] Scanner disconnected: reconnection steps
  - [ ] Low cache hit rate: pool list tuning
  - [ ] High latency: performance investigation
  - [ ] High error rate: debugging steps

**Deliverable:** Production-ready deployment
**Time Estimate:** 1 day

---

### Phase 5: Production Deployment (Day 8)

- [ ] **8.1 Pre-Deployment Checklist**
  - [ ] All tests passing
  - [ ] Monitoring configured
  - [ ] Alerts tested
  - [ ] Runbooks documented
  - [ ] Rollback procedure defined
  - [ ] Team briefed on deployment

- [ ] **8.2 Create Dockerfile for Scanner**
  - [ ] Create `deployment/docker/Dockerfile.shredstream-scanner`
  - [ ] Multi-stage build for optimization
  - [ ] Add health check
  - [ ] Test image locally

- [ ] **8.3 Update Docker Compose**
  - [ ] Add `shredstream-scanner` service to `docker-compose.yml`
  - [ ] Update `quote-service` with NATS config
  - [ ] Update NATS with JetStream config
  - [ ] Test full stack locally

- [ ] **8.4 Update Quote Service Binary**
  ```bash
  cd go/cmd/quote-service
  go build -o ../../../bin/quote-service.exe .
  ```

- [ ] **8.5 Deploy Scanner Service**
  ```bash
  cd deployment/docker
  docker-compose up -d shredstream-scanner
  docker-compose logs -f shredstream-scanner
  ```

- [ ] **8.6 Deploy Updated Quote Service**
  ```bash
  docker-compose up -d quote-service
  docker-compose logs -f quote-service
  ```

- [ ] **8.7 Validate Deployment**
  - [ ] Check scanner health: `curl http://localhost:8081/health`
  - [ ] Check quote service health: `curl http://localhost:8080/health`
  - [ ] Verify NATS events: `nats sub "pool.state.updated.*"`
  - [ ] Test quote requests: `curl "http://localhost:8080/quote?..."`
  - [ ] Check metrics: Grafana dashboard

- [ ] **8.8 Update Arbitrage Bot**
  - [ ] Point bot to updated quote service
  - [ ] Monitor execution latency
  - [ ] Expected: 1.7s → 800ms
  - [ ] Watch for errors or unusual behavior

- [ ] **8.9 Monitor for 24 Hours**
  - [ ] Watch Grafana dashboard continuously
  - [ ] Check cache hit rate (target: >80%)
  - [ ] Check end-to-end latency (target: <100ms P95)
  - [ ] Monitor error rates (target: <0.1%)
  - [ ] Verify no memory leaks
  - [ ] Document any issues

- [ ] **8.10 Measure Impact**
  - [ ] Record execution time improvement
  - [ ] Calculate cache hit rate
  - [ ] Measure quote latency (cache vs RPC)
  - [ ] Document lessons learned

**Deliverable:** Live system with Shredstream integration
**Time Estimate:** 1 day

---

### Phase 6: Optimization (Ongoing)

- [ ] **6.1 Week 2: Expand Pool Coverage**
  - [ ] Increase monitored pools to 50
  - [ ] Add more DEX programs
  - [ ] Monitor scanner performance
  - [ ] Optimize if needed

- [ ] **6.2 Week 3: Advanced Filtering**
  - [ ] Implement transaction intent detection
  - [ ] Track known bot wallets
  - [ ] Add pattern recognition
  - [ ] Publish intent events to NATS

- [ ] **6.3 Week 4: Multi-Region**
  - [ ] Deploy scanner in multiple regions
  - [ ] Aggregate events from all regions
  - [ ] Implement region failover
  - [ ] Monitor cross-region latency

- [ ] **6.4 Performance Tuning**
  - [ ] Profile scanner with Node.js profiler
  - [ ] Optimize hot paths
  - [ ] Reduce memory allocations
  - [ ] Consider Rust rewrite if needed (>100 req/sec)

- [ ] **6.5 Continuous Monitoring**
  - [ ] Weekly performance review
  - [ ] Monthly architecture review
  - [ ] Track ROI vs infrastructure cost
  - [ ] Adjust strategy based on data

**Time Estimate:** 1-2 days per optimization

---

## 6. Performance Considerations

### 6.1 Latency Breakdown

**Target: <100ms end-to-end (Shredstream → Quote Service → Bot)**

| Component | Operation | Target Latency | Optimization |
|-----------|-----------|----------------|--------------|
| **Shredstream** | Entry broadcast | 50-200ms | Use closest region (NY) |
| **Scanner** | Parse entry | 5-10ms | Reuse buffers, minimize allocations |
| **Scanner** | Filter accounts | 1-5ms | Bloom filter for pool addresses |
| **Scanner** | Decode pool state | 5-10ms | Optimize per-DEX decoders |
| **Scanner** | Publish to NATS | 1-2ms | Local NATS deployment |
| **NATS** | Event delivery | 1-2ms | JetStream with memory storage |
| **Quote Service** | Deserialize event | 0.5ms | Use `json.Unmarshal` (fast) |
| **Quote Service** | Update cache | 0.5ms | Lock-free map with RWMutex |
| **Quote Service** | Serve quote | 1-5ms | In-memory calculation |
| **Network** | HTTP round-trip | 1-2ms | Co-located services |
| **TOTAL** | | **70-90ms** | **5-10x faster than RPC** |

### 6.2 Throughput Targets

**Scanner Service:**
- **Entry rate:** 500-1000 entries/sec (Jito typical)
- **Transaction rate:** 5,000-10,000 tx/sec
- **Relevant tx rate:** 50-100 tx/sec (1-2% after filtering)
- **Event publish rate:** 50-100 events/sec
- **CPU utilization:** <30% on 2 cores
- **Memory:** <500MB

**Quote Service:**
- **NATS events:** 50-100 events/sec
- **HTTP requests:** 100-200 req/sec (from arbitrage bot)
- **Cache size:** 1,000 pools × 200 bytes = 200KB
- **CPU utilization:** <20% on 2 cores
- **Memory:** <200MB

### 6.3 Bottleneck Analysis

**Potential Bottlenecks:**

1. **Entry Parsing (Scanner)**
   - **Risk:** Complex binary parsing is CPU-intensive
   - **Mitigation:**
     - Optimize hot paths (use buffers, avoid allocations)
     - Implement parallel processing (worker pool)
     - Profile with Chrome DevTools

2. **NATS Throughput**
   - **Risk:** 100 events/sec × 2KB/event = 200KB/sec (negligible)
   - **Mitigation:**
     - Use JetStream with memory storage (no disk I/O)
     - Co-locate NATS with services (no network latency)

3. **Cache Lock Contention (Quote Service)**
   - **Risk:** High read/write concurrency on pool cache
   - **Mitigation:**
     - Use `sync.RWMutex` (multiple readers)
     - Consider lock-free data structures (future)

4. **Go GC Pauses**
   - **Risk:** Stop-the-world GC can cause latency spikes
   - **Mitigation:**
     - Minimize allocations in hot paths
     - Tune GOGC and GOMEMLIMIT
     - Monitor with `runtime.ReadMemStats`

### 6.4 Scalability Strategy

**Horizontal Scaling:**

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Scanner        │  │  Scanner        │  │  Scanner        │
│  (Region: NY)   │  │  (Region: AMS)  │  │  (Region: Tokyo)│
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                     │
         └──────────────┬─────┴─────────────────────┘
                        │ NATS (all regions publish to same stream)
                        ▼
         ┌──────────────────────────────────────────┐
         │         Quote Service (multiple replicas)│
         │         (Each subscribes to same stream) │
         └──────────────────────────────────────────┘
```

**Benefits:**
- Multiple regions provide redundancy
- Quote Service replicas load balance HTTP requests
- NATS JetStream handles fan-out automatically

---

## 7. Deployment Architecture

### 7.1 Docker Compose Configuration

```yaml
# deployment/docker/docker-compose.yml (UPDATED)

services:
  # NEW: Shredstream Scanner Service
  shredstream-scanner:
    build:
      context: ../../
      dockerfile: deployment/docker/Dockerfile.shredstream-scanner
    container_name: shredstream-scanner
    environment:
      # Shredstream connection
      SHREDSTREAM_HOST: ${SHREDSTREAM_HOST:-10.1.1.160}
      SHREDSTREAM_PORT: ${SHREDSTREAM_PORT:-50051}
      SHREDSTREAM_REGIONS: ${SHREDSTREAM_REGIONS:-ny,amsterdam}
      SHREDSTREAM_HEARTBEAT: ${SHREDSTREAM_HEARTBEAT:-false}

      # NATS connection
      NATS_SERVERS: nats://nats:4222
      NATS_STREAM_NAME: POOL_UPDATES

      # Monitoring
      MONITORED_POOLS: ${MONITORED_POOLS:-}  # Load from file
      LOG_LEVEL: info
      PROMETHEUS_PORT: 9091
    ports:
      - "9091:9091"  # Prometheus metrics
      - "8081:8081"  # Health check API
    depends_on:
      - nats
    restart: unless-stopped
    networks:
      - solana-trading
    volumes:
      - ./config/monitored-pools.json:/app/config/monitored-pools.json:ro

  # UPDATED: Quote Service with NATS integration
  quote-service:
    build:
      context: ../../
      dockerfile: deployment/docker/Dockerfile.quote-service
    container_name: quote-service
    environment:
      # HTTP API
      PORT: 8080

      # RPC endpoints (fallback)
      RPC_ENDPOINTS: ${RPC_ENDPOINTS}

      # NATS configuration (NEW)
      NATS_ENABLED: true
      NATS_SERVERS: nats://nats:4222
      NATS_STREAM_NAME: POOL_UPDATES
      NATS_CONSUMER_NAME: quote-service

      # Cache configuration (NEW)
      CACHE_ENABLED: true
      CACHE_STALENESS_THRESHOLD: 30
      CACHE_MAX_POOL_COUNT: 1000

      # Hybrid mode (NEW)
      HYBRID_MODE: true
      CACHE_FIRST_TIMEOUT: 10

      # Performance tuning
      REFRESH_SECONDS: 30
      SLIPPAGE_BPS: 50
      RATE_LIMIT_RPS: 20
    ports:
      - "8080:8080"  # HTTP API
      - "9090:9090"  # Prometheus metrics
    depends_on:
      - nats
      - shredstream-scanner
    restart: unless-stopped
    networks:
      - solana-trading

  # NATS JetStream (existing, updated config)
  nats:
    image: nats:2.10-alpine
    container_name: nats
    command:
      - "--jetstream"
      - "--store_dir=/data"
      - "--max_mem_store=1GB"
      - "--max_file_store=10GB"
    ports:
      - "4222:4222"  # Client connections
      - "8222:8222"  # HTTP monitoring
      - "6222:6222"  # Cluster routing
    volumes:
      - nats-data:/data
    restart: unless-stopped
    networks:
      - solana-trading

  # ... other services (Redis, Postgres, Grafana, etc.)

networks:
  solana-trading:
    driver: bridge

volumes:
  nats-data:
```

### 7.2 Dockerfile for Shredstream Scanner

```dockerfile
# deployment/docker/Dockerfile.shredstream-scanner
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY ts/package.json ts/pnpm-lock.yaml ts/pnpm-workspace.yaml ./
COPY ts/apps/shredstream-scanner/package.json ./apps/shredstream-scanner/

# Install dependencies
RUN npm install -g pnpm@9.0.0
RUN pnpm install --frozen-lockfile

# Copy source code
COPY ts/apps/shredstream-scanner ./apps/shredstream-scanner
COPY ts/turbo.json ./

# Build
RUN pnpm --filter=shredstream-scanner build

# Production image
FROM node:20-alpine

WORKDIR /app

# Copy built application
COPY --from=builder /app/apps/shredstream-scanner/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Expose ports
EXPOSE 8081 9091

# Health check
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8081/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

# Run
CMD ["node", "dist/main.js"]
```

### 7.3 Kubernetes Deployment (Future)

**When to migrate to K8s:**
- Traffic >1000 req/sec
- Need multi-region deployment
- Require auto-scaling

```yaml
# deployment/k8s/shredstream-scanner-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shredstream-scanner
  namespace: solana-trading
spec:
  replicas: 2  # Multi-region deployment
  selector:
    matchLabels:
      app: shredstream-scanner
  template:
    metadata:
      labels:
        app: shredstream-scanner
    spec:
      containers:
      - name: scanner
        image: solana-trading/shredstream-scanner:latest
        ports:
        - containerPort: 8081
          name: http
        - containerPort: 9091
          name: metrics
        env:
        - name: SHREDSTREAM_HOST
          value: "10.1.1.160"
        - name: NATS_SERVERS
          value: "nats://nats-service:4222"
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8081
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: shredstream-scanner-service
  namespace: solana-trading
spec:
  selector:
    app: shredstream-scanner
  ports:
  - name: http
    port: 8081
    targetPort: 8081
  - name: metrics
    port: 9091
    targetPort: 9091
```

---

## 8. Monitoring & Observability

### 8.1 Grafana Dashboard

**Dashboard: Shredstream Integration Overview**

**Panel 1: End-to-End Latency (Line Chart)**
```promql
# P50, P95, P99 latency from Shredstream to Quote Service cache update
histogram_quantile(0.50,
  rate(shredstream_scanner_end_to_end_latency_ms_bucket[5m])
) by (le)

histogram_quantile(0.95,
  rate(shredstream_scanner_end_to_end_latency_ms_bucket[5m])
) by (le)

histogram_quantile(0.99,
  rate(shredstream_scanner_end_to_end_latency_ms_bucket[5m])
) by (le)
```

**Panel 2: Throughput (Stacked Area Chart)**
```promql
# Scanner: entries/sec, transactions/sec, events published/sec
rate(shredstream_scanner_entries_received_total[5m])
rate(shredstream_scanner_transactions_processed_total[5m])
rate(shredstream_scanner_events_published_total[5m])

# Quote Service: cache updates/sec, quotes served/sec
rate(quote_service_pool_updates_received_total[5m])
rate(quote_service_quotes_total[5m])
```

**Panel 3: Cache Performance (Gauge + Line Chart)**
```promql
# Cache hit rate
rate(quote_service_quote_cache_hits_total[5m]) /
(rate(quote_service_quote_cache_hits_total[5m]) +
 rate(quote_service_quote_cache_misses_total[5m]))

# Cache size
quote_service_pool_cache_total_pools
quote_service_pool_cache_fresh_pools
quote_service_pool_cache_stale_pools
```

**Panel 4: Error Rates (Bar Chart)**
```promql
# Scanner errors
rate(shredstream_scanner_parsing_errors_total[5m])
rate(shredstream_scanner_nats_publish_errors_total[5m])
rate(shredstream_scanner_connection_errors_total[5m])

# Quote Service errors
rate(quote_service_nats_parse_errors_total[5m])
rate(quote_service_rpc_fallback_errors_total[5m])
```

**Panel 5: Connection Health (Status Panel)**
```promql
# Scanner connection status
shredstream_scanner_connected

# NATS connection status
nats_server_is_up

# Quote Service NATS consumer lag
quote_service_nats_consumer_lag_messages
```

### 8.2 Alert Rules

```yaml
# deployment/monitoring/prometheus/alert-rules.yml

groups:
- name: shredstream
  interval: 10s
  rules:

  # Scanner disconnected for >10 seconds
  - alert: ShredstreamScannerDisconnected
    expr: shredstream_scanner_connected == 0
    for: 10s
    labels:
      severity: critical
      component: shredstream-scanner
    annotations:
      summary: "Shredstream scanner disconnected"
      description: "Scanner has been disconnected from Jito Shredstream for >10s"
      action: "Check scanner logs and Shredstream endpoint health"

  # Cache hit rate below 80%
  - alert: LowCacheHitRate
    expr: |
      (
        rate(quote_service_quote_cache_hits_total[5m]) /
        (rate(quote_service_quote_cache_hits_total[5m]) +
         rate(quote_service_quote_cache_misses_total[5m]))
      ) < 0.80
    for: 2m
    labels:
      severity: warning
      component: quote-service
    annotations:
      summary: "Quote Service cache hit rate below 80%"
      description: "Cache hit rate is {{ $value | humanizePercentage }}, indicating stale or missing pool data"
      action: "Check scanner event publishing rate and pool monitoring config"

  # End-to-end latency >100ms (P95)
  - alert: HighShredstreamLatency
    expr: |
      histogram_quantile(0.95,
        rate(shredstream_scanner_end_to_end_latency_ms_bucket[5m])
      ) > 100
    for: 5m
    labels:
      severity: warning
      component: shredstream-scanner
    annotations:
      summary: "High end-to-end latency from Shredstream to cache"
      description: "P95 latency is {{ $value }}ms, exceeding 100ms target"
      action: "Check network latency, NATS performance, and entry parsing time"

  # High error rate in scanner
  - alert: ShredstreamScannerHighErrorRate
    expr: |
      (
        rate(shredstream_scanner_parsing_errors_total[5m]) +
        rate(shredstream_scanner_nats_publish_errors_total[5m])
      ) > 1
    for: 2m
    labels:
      severity: warning
      component: shredstream-scanner
    annotations:
      summary: "High error rate in Shredstream scanner"
      description: "Scanner is experiencing {{ $value }} errors/sec"
      action: "Check scanner logs for parsing errors or NATS connectivity issues"

  # NATS consumer lag
  - alert: NATSConsumerLag
    expr: quote_service_nats_consumer_lag_messages > 100
    for: 1m
    labels:
      severity: warning
      component: quote-service
    annotations:
      summary: "NATS consumer lagging behind"
      description: "Quote Service has {{ $value }} unprocessed messages in NATS"
      action: "Check Quote Service performance and NATS health"
```

### 8.3 Logging Strategy

**Scanner Service:**
```typescript
// Structured logging with Winston
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'shredstream-scanner' },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error'
    }),
    new winston.transports.File({
      filename: 'logs/combined.log'
    }),
  ],
});

// Log examples
logger.info('Entry received', {
  slot: entry.slot,
  transactionCount: signatures.length,
  latency: Date.now() - entryTimestamp,
});

logger.error('Failed to parse entry', {
  slot: entry.slot,
  error: error.message,
  stack: error.stack,
});

logger.warn('High processing queue size', {
  queueSize: processingQueue.size,
  threshold: PROCESSING_QUEUE_SIZE,
});
```

**Quote Service:**
```go
// Structured logging with zap
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
defer logger.Sync()

// Log examples
logger.Info("Pool state updated",
    zap.String("pool", poolState.Address),
    zap.Uint64("slot", poolState.Slot),
    zap.String("dex", poolState.Dex),
    zap.Int64("reserve_a", poolState.ReserveA.Int64()),
    zap.Int64("reserve_b", poolState.ReserveB.Int64()),
)

logger.Error("Failed to deserialize NATS event",
    zap.Error(err),
    zap.String("subject", msg.Subject),
)

logger.Warn("Cache miss for quote",
    zap.String("input_mint", inputMint),
    zap.String("output_mint", outputMint),
    zap.Float64("cache_hit_rate", cacheHitRate),
)
```

### 8.4 Distributed Tracing

**OpenTelemetry Integration:**

```typescript
// Scanner: Trace entry processing
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('shredstream-scanner');

shredstream.onEntry(async (entry) => {
  const span = tracer.startSpan('process_entry', {
    attributes: {
      'entry.slot': entry.slot,
      'entry.size': entry.entries.length,
    },
  });

  try {
    const ctx = trace.setSpan(context.active(), span);

    // Parse entry
    const parseSpan = tracer.startSpan('parse_entry', undefined, ctx);
    const signatures = extractSignatures(entry.entries);
    parseSpan.end();

    // Publish events
    const publishSpan = tracer.startSpan('publish_events', undefined, ctx);
    await publishPoolUpdates(signatures);
    publishSpan.setAttribute('events.published', signatures.length);
    publishSpan.end();

    span.setStatus({ code: SpanStatusCode.OK });
  } catch (error) {
    span.setStatus({
      code: SpanStatusCode.ERROR,
      message: error.message
    });
    span.recordException(error);
  } finally {
    span.end();
  }
});
```

---

## 9. Risk Analysis & Mitigation

### 9.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Shredstream API changes** | HIGH - Scanner breaks | LOW | Version lock; Monitor Jito changelogs; Automated integration tests |
| **NATS performance bottleneck** | MEDIUM - Event lag | LOW | Capacity planning; JetStream with memory storage; Monitor lag metrics |
| **Cache inconsistency** | HIGH - Incorrect quotes | MEDIUM | Slot-based ordering; Periodic RPC validation; Cache staleness timeout |
| **Entry parsing errors** | MEDIUM - Missed updates | MEDIUM | Comprehensive error handling; Fallback to RPC; Alert on high error rate |
| **Network partition** | HIGH - No updates | LOW | Health checks; Auto-reconnect; Fallback to RPC mode |
| **Scanner memory leak** | MEDIUM - Service crash | LOW | Memory profiling; Automated restart; Monitoring alerts |
| **Go GC pauses** | LOW - Latency spikes | LOW | Tune GOGC; Monitor GC metrics; Use pprof |

### 9.2 Operational Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Deployment failure** | HIGH - Service unavailable | MEDIUM | Blue-green deployment; Rollback procedure; Pre-deployment tests |
| **Configuration error** | HIGH - Wrong pools monitored | MEDIUM | Config validation; Dry-run mode; Gradual rollout |
| **Resource exhaustion** | HIGH - Service degradation | LOW | Resource limits; Auto-scaling; Capacity monitoring |
| **Monitoring blind spot** | MEDIUM - Issues undetected | MEDIUM | Comprehensive metrics; Alert testing; Runbook documentation |
| **Skill gap** | LOW - Slow incident response | LOW | Documentation; Runbooks; Team training |

### 9.3 Business Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Shredstream pricing change** | HIGH - Increased costs | LOW | Monitor announcements; Budget planning; ROI analysis |
| **Competitive advantage lost** | MEDIUM - Others adopt Shredstream | HIGH | Continuous optimization; Strategy evolution; Multiple alpha sources |
| **Regulatory concerns** | LOW - MEV scrutiny | LOW | Compliance review; No front-running; Transparent operations |
| **Over-optimization** | MEDIUM - Diminishing returns | MEDIUM | ROI tracking; Focus on high-impact features; Avoid premature optimization |

### 9.4 Mitigation Strategies

**1. Circuit Breaker Pattern**

```go
// Quote Service: Automatic fallback to RPC
type CircuitBreaker struct {
    maxFailures  int
    resetTimeout time.Duration
    state        string // "closed", "open", "half-open"
    failures     int
    lastFailure  time.Time
}

func (s *Server) getQuoteFromCache(inputMint, outputMint string, amount uint64) (*Quote, error) {
    // Check circuit breaker
    if s.circuitBreaker.IsOpen() {
        return nil, fmt.Errorf("circuit breaker open, using RPC fallback")
    }

    // Try cache
    quote, err := s.tryCache(inputMint, outputMint, amount)
    if err != nil {
        s.circuitBreaker.RecordFailure()
        return nil, err
    }

    s.circuitBreaker.RecordSuccess()
    return quote, nil
}
```

**2. Gradual Rollout Strategy**

```
Week 1: Deploy scanner + quote service, but Quote Service still uses RPC primarily
        - Validate cache accuracy
        - Monitor latency
        - Test under load

Week 2: Enable hybrid mode (cache first, RPC fallback)
        - Monitor cache hit rate (target: >80%)
        - Measure latency improvement
        - Track error rates

Week 3: Increase monitored pools (20 → 50)
        - Validate scaling
        - Optimize performance
        - Update arbitrage bot

Week 4: Full production mode
        - Cache-first for all requests
        - Remove RPC as primary source
        - Monitor ROI
```

**3. Automated Validation**

```typescript
// Periodic cache validation job (runs every 5 minutes)
async function validateCacheAccuracy() {
  const pools = getRandomSample(monitoredPools, 10);

  for (const pool of pools) {
    // Get cached state
    const cached = cache.get(pool);

    // Get RPC state
    const rpc = await fetchPoolStateFromRPC(pool);

    // Compare
    const delta = calculateStateDelta(cached, rpc);

    if (delta > ACCEPTABLE_THRESHOLD) {
      logger.error('Cache drift detected', {
        pool,
        cached: cached.reserves,
        rpc: rpc.reserves,
        delta,
      });

      // Alert and fallback to RPC
      alertCacheDrift(pool, delta);
    }
  }
}
```

---

## 10. Migration from Existing Code

### 10.1 Current State Analysis

**Existing Code Location:**
- `references/apps/cli-tools/services/shredstreamService.ts`
- `references/apps/cli-tools/services/entryParserService.ts`
- `references/apps/cli-tools/services/shredstreamRpcBridge.ts` ❌ DELETE
- `references/apps/cli-tools/commands/accountChangeSubscriptionShredstream.ts`

**Issues to Fix:**

1. **❌ RPC Bridge Pattern** (`shredstreamRpcBridge.ts`)
   - **Problem:** Goes back to RPC after receiving Shredstream data
   - **Fix:** Delete this file entirely; parse pool state directly

2. **❌ No Pool Account Filtering** (`accountChangeSubscriptionShredstream.ts:77`)
   - **Problem:** Extracts ALL signatures, then processes them
   - **Fix:** Use `extractSignaturesForAccounts()` with monitored pool list

3. **❌ Serial Transaction Processing** (`accountChangeSubscriptionShredstream.ts:89-116`)
   - **Problem:** Processes each transaction individually in loop
   - **Fix:** Batch processing with `Promise.allSettled()`

4. **❌ Tightly Coupled to Arbitrage Service** (`accountChangeSubscriptionShredstream.ts:174`)
   - **Problem:** Directly calls `followSwapTransaction()`
   - **Fix:** Publish NATS events instead

### 10.2 Migration Steps

**Step 1: Create New Service Structure**

```bash
# Create new scanner app
cd ts/apps
mkdir shredstream-scanner
cd shredstream-scanner

# Initialize package
pnpm init

# Copy and refactor existing code
cp ../../references/apps/cli-tools/services/shredstreamService.ts src/
cp ../../references/apps/cli-tools/services/entryParserService.ts src/
# DON'T copy shredstreamRpcBridge.ts - DELETE IT
```

**Step 2: Add NATS Publisher**

```typescript
// src/natsPublisher.ts (NEW FILE)
import { connect, NatsConnection, JSONCodec, JetStreamClient } from 'nats';

export class NATSPublisher {
  private nc: NatsConnection;
  private js: JetStreamClient;
  private jc = JSONCodec();

  async connect(servers: string[]) {
    this.nc = await connect({ servers });
    this.js = this.nc.jetstream();

    // Ensure stream exists
    await this.js.streams.add({
      name: 'POOL_UPDATES',
      subjects: ['pool.state.updated.*'],
      retention: 'limits',
      max_age: 60_000_000_000, // 60 seconds in nanoseconds
      storage: 'memory',
    });
  }

  async publishPoolUpdate(event: PoolStateUpdateEvent) {
    const subject = `pool.state.updated.${event.pool.dex}.${event.pool.address}`;
    await this.js.publish(subject, this.jc.encode(event));
  }

  async close() {
    await this.nc.close();
  }
}
```

**Step 3: Add Pool State Decoders**

```typescript
// src/poolDecoders/raydiumAmm.ts (NEW FILE)
export function decodeRaydiumAmmPoolState(accountData: Buffer): PoolState {
  // Raydium AMM V4 account structure (simplified)
  // See: https://github.com/raydium-io/raydium-sdk

  const STATUS_OFFSET = 0;
  const RESERVE_A_OFFSET = 64;
  const RESERVE_B_OFFSET = 72;
  const MINT_A_OFFSET = 400;
  const MINT_B_OFFSET = 432;

  // Read status (u64)
  const status = accountData.readBigUInt64LE(STATUS_OFFSET);

  // Read reserves (u64)
  const reserveA = accountData.readBigUInt64LE(RESERVE_A_OFFSET);
  const reserveB = accountData.readBigUInt64LE(RESERVE_B_OFFSET);

  // Read mints (32 bytes each)
  const mintA = accountData.slice(MINT_A_OFFSET, MINT_A_OFFSET + 32);
  const mintB = accountData.slice(MINT_B_OFFSET, MINT_B_OFFSET + 32);

  return {
    reserveA: reserveA.toString(),
    reserveB: reserveB.toString(),
    tokenA: bs58.encode(mintA),
    tokenB: bs58.encode(mintB),
    feeNumerator: '25',       // Raydium default: 0.25%
    feeDenominator: '10000',
  };
}
```

**Step 4: Update Main Scanner Logic**

```typescript
// src/main.ts (REFACTORED)
import { createShredstreamService } from './shredstreamService';
import { extractSignaturesForAccounts } from './entryParserService';
import { NATSPublisher } from './natsPublisher';
import { decodePoolState } from './poolDecoders';

// Load monitored pools from config
const MONITORED_POOLS = loadMonitoredPools('./config/monitored-pools.json');

const main = async () => {
  // Create NATS publisher
  const natsPublisher = new NATSPublisher();
  await natsPublisher.connect(['nats://localhost:4222']);

  // Create Shredstream service
  const shredstream = createShredstreamService({
    host: process.env.SHREDSTREAM_HOST || '10.1.1.160',
    port: parseInt(process.env.SHREDSTREAM_PORT || '50051'),
    regions: (process.env.SHREDSTREAM_REGIONS || 'ny').split(','),
  });

  // Handle entries
  shredstream.onEntry(async (entry) => {
    const receivedAt = Date.now();

    // STEP 1: Filter for monitored pools FIRST
    const relevantSigs = extractSignaturesForAccounts(
      entry.entries,
      Array.from(MONITORED_POOLS.keys())  // Only our pools
    );

    if (relevantSigs.length === 0) {
      return; // 99% of entries filtered here
    }

    // STEP 2: Batch process relevant transactions
    const poolUpdates = await Promise.allSettled(
      relevantSigs.map(sig => extractPoolStateFromTransaction(sig, entry))
    );

    // STEP 3: Publish events for successful updates
    for (const result of poolUpdates) {
      if (result.status === 'fulfilled' && result.value) {
        const event: PoolStateUpdateEvent = {
          eventId: uuidv4(),
          timestamp: Date.now(),
          source: 'shredstream',
          latency: Date.now() - receivedAt,
          pool: result.value.pool,
          state: result.value.state,
          transaction: {
            signature: result.value.signature,
            confirmed: false,
          },
        };

        await natsPublisher.publishPoolUpdate(event);
      }
    }
  });

  await shredstream.subscribe();
  console.log('Shredstream scanner started');
};

main().catch(console.error);
```

**Step 5: Deploy and Test**

```bash
# Build scanner
pnpm build

# Deploy with Docker Compose
cd deployment/docker
docker-compose up -d shredstream-scanner

# Watch logs
docker-compose logs -f shredstream-scanner

# Check metrics
curl http://localhost:9091/metrics

# Verify NATS events
nats sub "pool.state.updated.*"
```

---

## 11. Success Criteria

### Week 1 (Phase 1-5)
- ✅ Scanner deployed and receiving Shredstream data
- ✅ Quote Service serving quotes from cache
- ✅ Cache hit rate >80%
- ✅ End-to-end latency <100ms (P95)
- ✅ Arbitrage bot execution: 1.7s → 800ms (**53% improvement**)

### Week 2 (Phase 6.1)
- ✅ 50 pools monitored
- ✅ Multiple DEX support
- ✅ Arbitrage bot execution: 800ms → 400ms (combined with Go quotes)

### Week 4 (Phase 6.2-6.3)
- ✅ Advanced filtering and intent detection
- ✅ Multi-region deployment
- ✅ Arbitrage bot execution: <200ms (**TARGET MET** ✅)

---

## 12. Troubleshooting Guide

### Scanner Not Receiving Data
```bash
# Check Shredstream connection
docker-compose logs shredstream-scanner | grep -i "connect"

# Check endpoint accessibility
telnet 10.1.1.160 50051

# Verify proto definitions match
cd proto && ./verify-proto.ps1
```

### Low Cache Hit Rate (<80%)
```bash
# Check pool list coverage
curl http://localhost:8081/config

# Check cache staleness threshold
curl http://localhost:8080/health | jq '.cache'

# Verify NATS event delivery
nats stream info POOL_UPDATES
```

### High Latency (>100ms P95)
```bash
# Check NATS latency
nats bench pub.sub --msgs 1000

# Profile scanner
node --inspect src/main.js

# Check Go service GC
curl http://localhost:8080/debug/pprof/heap
```

### Scanner Crashes
```bash
# Check memory usage
docker stats shredstream-scanner

# Check logs for errors
docker-compose logs --tail=100 shredstream-scanner | grep -i "error"

# Restart with debug logging
LOG_LEVEL=debug docker-compose up shredstream-scanner
```

---

## 13. Conclusion

### Key Takeaways

1. **✅ Separate Shredstream Scanner Service** is the recommended architecture
   - Maintains separation of concerns
   - Follows Scanner→Planner→Executor pattern
   - Fast time to market (2-3 days vs 7-9 days for monolith)
   - TypeScript for I/O, Go for compute

2. **✅ NATS as Integration Layer** provides optimal trade-off
   - 1-2ms latency penalty (negligible vs 400ms alpha)
   - Reliable event delivery with JetStream
   - Decouples services for independent scaling
   - Simpler than Redis, faster than HTTP

3. **✅ Hybrid Quote Strategy** maximizes reliability
   - Cache-first for low latency (5-10ms)
   - RPC fallback for reliability
   - Slot-based cache consistency
   - Periodic validation

4. **✅ Focus on High-Impact Optimizations**
   - Pool state updates (300-800ms alpha) ⭐⭐⭐
   - Local quote engine (10ms quotes) ⭐⭐⭐
   - Targeted pool monitoring (99% filter efficiency) ⭐⭐
   - Bot-following signals (10-15% win rate) ⭐

### Expected Outcomes

**Week 1:** 1.7s → 800ms execution (53% faster)
**Week 2:** 800ms → 400ms execution (additional 50% improvement)
**Week 3:** 400ms → 250ms execution (flash loans)
**Week 4:** 250ms → <200ms execution ✅ **TARGET MET**

**Performance Metrics:**
- End-to-end latency: 70-90ms (Shredstream → Quote Service)
- Cache hit rate: >80%
- Throughput: 100+ events/sec, 200+ quotes/sec
- Alpha: 300-800ms head start on market-moving swaps

### Next Steps

1. **✅ Implement Shredstream Scanner** (Days 1-2)
2. **✅ Integrate with Quote Service** (Days 3-4)
3. **✅ Add Pool Decoders** (Days 5-6)
4. **✅ Test and Validate** (Day 7)
5. **✅ Deploy to Production** (Day 8)
6. **✅ Monitor and Optimize** (Ongoing)

---

**Document Status:** READY FOR IMPLEMENTATION (Merged with implementation checklist)
**Review Date:** 2025-12-20
**Next Review:** After Phase 1 deployment (Day 3)

---

**Note:** This document combines the architectural design and detailed implementation checklist. It replaces both:
- `17-SHREDSTREAM-ARCHITECTURE-DESIGN.md` (original architecture)
- `17A-SHREDSTREAM-IMPLEMENTATION-CHECKLIST.md` (detailed checklist - merged into this document)
