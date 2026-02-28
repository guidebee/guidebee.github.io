---
layout: single
title: "Initial HFT Architecture: From Working Prototype to Sub-500ms Execution"
permalink: /docs/07-initial-hft-architecture/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Initial HFT Architecture: From Working Prototype to Sub-500ms Execution

## Overview

This document outlines the **initial architecture design** for building a production-grade HFT (High-Frequency Trading) system on Solana. The goal is to evolve from a working arbitrage prototype (~1.7s execution) to a **sub-500ms sub-second HFT system** with 95%+ Jito bundle landing rate.

**Starting Point:** Working arbitrage bot with Jupiter API integration (~1.7-2s execution)
**Target State:** Sub-500ms execution with local pool math, Shredstream integration, and optimized Jito submission
**Timeline:** 4 weeks for core optimizations, 12 weeks for full production system

---

## Critical HFT Principles

### 1. Latency is Everything
**Target:** Complete execution path < 500ms (ideally < 200ms)

```
Market Event → Opportunity Detection → Transaction Submission
    |              |                        |
  <50ms          <100ms                  <100ms
```

**Latency Budget Breakdown:**
- Market event detection: 50ms (Shredstream)
- Quote calculation: 5-10ms (local pool math)
- Opportunity validation: 10ms (profit calculation)
- Transaction building: 20ms (instruction assembly)
- Jito bundle submission: 50ms (network + processing)
- Confirmation: 400ms-2s (Solana block time)

### 2. Capital Efficiency is Paramount
- Flash loans for zero-capital arbitrage (Kamino 0.05% fee)
- Dynamic position sizing based on liquidity
- Multi-wallet parallelization (5-10 concurrent trades)
- Immediate profit realization (no inventory holding)

### 3. Reliability > Features
- 99.99% uptime requirement
- Zero data loss (all opportunities logged)
- Graceful degradation (fallback to slower paths)
- Circuit breakers (stop on consecutive failures)

---

## Current Bottlenecks Analysis

Based on the working prototype, here are the key bottlenecks to address:

### 1. Jupiter Quote API Latency (100-300ms per call)
**Problem:** Every quote requires HTTP request to Jupiter API
**Impact:** 150ms average per quote

**Current (SLOW):**
```typescript
// 150ms per call
const quote = await fetch('https://quote-api.jup.ag/v6/quote', {
  params: { inputMint, outputMint, amount }
});
```

**Solution:** Local pool math in Go (2-10ms)
```go
// 5ms per quote
quote := quoteEngine.GetQuote(inputMint, outputMint, amount)
```

**Expected Improvement:** 100-150ms → 5ms (20-30x faster)

### 2. Sequential Quote Generation
**Problem:** Quotes fetched one at a time
**Impact:** 300ms total for round-trip arbitrage

**Current (SLOW):**
```typescript
const quote1 = await getQuote("SOL", "USDC", amount); // 150ms
const quote2 = await getQuote("USDC", "SOL", quote1.output); // 150ms
// Total: 300ms
```

**Solution:** Concurrent quote generation
```typescript
const [quote1, quote2] = await Promise.all([
  getQuote("SOL", "USDC", amount),
  getQuote("USDC", "SOL", estimatedOutput)
]);
// Total: 150ms
```

**Expected Improvement:** 300ms → 150ms (2x faster)

### 3. RPC Call Overhead
**Problem:** Multiple sequential RPC calls for pool data
**Impact:** 200ms per pool

**Solution:** Batch RPC + cache
```typescript
// Batch fetch 10 pools in 50ms
const poolDatas = await connection.getMultipleAccountsInfo(pools);
const quotes = poolDatas.map(data => calculateQuote(data, amount));
```

**Expected Improvement:** 200ms/pool → 6ms/pool (33x faster)

### 4. Blockhash Fetching Overhead
**Problem:** Fetching recent blockhash every transaction
**Impact:** 50ms per transaction

**Solution:** Cache blockhash (valid for ~30 seconds)
```typescript
class BlockhashCache {
  private cachedBlockhash: string | null = null;
  private lastFetch: number = 0;

  async get(connection: Connection): Promise<string> {
    const now = Date.now();
    if (!this.cachedBlockhash || now - this.lastFetch > 20_000) {
      const { blockhash } = await connection.getLatestBlockhash();
      this.cachedBlockhash = blockhash;
      this.lastFetch = now;
    }
    return this.cachedBlockhash;
  }
}
```

**Expected Improvement:** 50ms → 1ms (50x faster)

### 5. Confirmation Polling Inefficiency
**Problem:** Polling every 1-2 seconds
**Impact:** 1-2s added latency

**Solution:** WebSocket subscription + exponential backoff polling
```typescript
const confirmed = await Promise.race([
  this.subscribeToSignature(sig), // WebSocket
  this.pollWithBackoff(sig, [100, 200, 300, 500]) // Exponential
]);
```

**Expected Improvement:** 1000-2000ms → 400ms (2.5-5x faster)

---

## HFT System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   MARKET DATA LAYER (HOT PATH)                   │
│  Shredstream → Slot Notifications → Account Changes → Filtering  │
│                      < 50ms latency                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                 QUOTE ENGINE (CRITICAL PATH)                     │
│  Local Pool Math (Go/Rust) → Concurrent Calculations → Ranking  │
│                      < 10ms latency                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              STRATEGY ENGINE (DECISION LAYER)                    │
│  Arbitrage | Market Making | Liquidations | Statistical Arb     │
│                      < 20ms latency                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              PLANNED LAYER (SUBMISSION)                        │
│  Flash Loan Wrapping → Jito Bundle → Direct TPU (backup)        │
│                      < 100ms submission                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                 CONFIRMATION & SETTLEMENT                        │
│  Status Polling → Log Settlement → Update Positions             │
│                      400ms-2s                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Deep Dive

### 1. Market Data Layer (Shredstream Integration)

**Purpose:** Get market updates ~400ms BEFORE they hit RPC nodes

#### Shredstream Architecture
```typescript
interface ShredstreamConfig {
  endpoint: "wss://shredstream.jito.wtf";
  subscriptions: [
    "raydium_amm",
    "raydium_clmm",
    "meteora_dlmm",
    "orca_whirlpool",
    "jupiter_swap"
  ];
  filterByAccounts: [
    "58oQChx4yWmvKdwLLZzBi4ChoCc2fqCUWBkwMihLYQo2", // SOL/USDC Raydium
    "HJPjoWUrhoZzkNfRpHuieeFk9WcZWjwy6PBjZ81ngndJ", // SOL/USDC Orca
    // ... top 100 pools
  ];
}

class ShredstreamProcessor {
  async processSlotUpdate(slot: number, accounts: AccountUpdate[]) {
    const timestamp = performance.now();

    for (const account of accounts) {
      this.eventBus.publish(`account.${account.pubkey}`, {
        slot,
        account: account.data,
        receivedAt: timestamp
      });
    }

    this.metrics.recordLatency("shredstream_to_process",
      performance.now() - timestamp);
  }
}
```

**Optimization Techniques:**
- **Client-Side Filtering:** Filter by pool addresses in WebSocket subscription (reduces bandwidth 99%)
- **Binary Parsing:** Decode Borsh directly in Rust/Go (avoid JSON overhead)
- **Lock-Free Queues:** Use ring buffers for zero-allocation event handling
- **Hot Pool Set:** Keep top 100 pools in memory (L1 cache locality)

#### In-Memory Pool State (Rust)
```rust
#[repr(C, align(64))]  // Cache line aligned
struct PoolState {
    pool_id: Pubkey,
    base_reserve: u64,
    quote_reserve: u64,
    base_decimals: u8,
    quote_decimals: u8,
    fee_rate: u16,
    last_update_slot: u64,
    last_update_time: u64,
}

impl PoolState {
    fn update_reserves(&mut self, base: u64, quote: u64, slot: u64) {
        self.base_reserve = base;
        self.quote_reserve = quote;
        self.last_update_slot = slot;
        self.last_update_time = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_micros() as u64;
    }
}
```

---

### 2. Quote Engine (Local Pool Math)

**Purpose:** Calculate quotes in < 10ms without external API calls

#### Go Implementation (Concurrent Quote Engine)
```go
package quote

import (
    "cosmossdk.io/math"
    "sync"
)

type QuoteEngine struct {
    pools     map[string]*PoolState
    poolsLock sync.RWMutex
}

func (qe *QuoteEngine) GetQuote(
    inputMint string,
    outputMint string,
    amountIn math.Int,
) (*Quote, error) {
    start := time.Now()
    defer func() {
        metrics.RecordLatency("quote_calculation", time.Since(start))
    }()

    // Find all pools for this pair
    pools := qe.findPoolsForPair(inputMint, outputMint)

    // Concurrent quote calculation
    results := make(chan *Quote, len(pools))
    var wg sync.WaitGroup

    for _, pool := range pools {
        wg.Add(1)
        go func(p *PoolState) {
            defer wg.Done()
            quote := p.calculateQuote(amountIn)
            if quote != nil {
                results <- quote
            }
        }(pool)
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    // Select best quote
    var bestQuote *Quote
    for quote := range results {
        if bestQuote == nil || quote.OutputAmount.GT(bestQuote.OutputAmount) {
            bestQuote = quote
        }
    }

    return bestQuote, nil
}

// Constant product formula with fees
func (p *PoolState) calculateQuote(amountIn math.Int) *Quote {
    // Apply fee (e.g., 0.3% = 30 bps)
    amountInWithFee := amountIn.Mul(math.NewInt(10000 - int64(p.FeeRate)))
    amountInWithFee = amountInWithFee.Quo(math.NewInt(10000))

    // Calculate output
    numerator := amountInWithFee.Mul(math.NewInt(int64(p.QuoteReserve)))
    denominator := math.NewInt(int64(p.BaseReserve)).Add(amountInWithFee)
    amountOut := numerator.Quo(denominator)

    return &Quote{
        PoolID:       p.PoolID,
        Protocol:     p.Protocol,
        InputAmount:  amountIn,
        OutputAmount: amountOut,
    }
}
```

#### Rust Implementation (Maximum Performance)
```rust
use rayon::prelude::*;

pub struct QuoteEngine {
    pools: Arc<Vec<PoolState>>,
}

impl QuoteEngine {
    pub fn get_best_quote(
        &self,
        input_mint: &Pubkey,
        output_mint: &Pubkey,
        amount_in: u64,
    ) -> Option<Quote> {
        let start = Instant::now();

        // Parallel quote calculation using rayon
        let best_quote = self.pools
            .par_iter()
            .filter(|p| p.matches_pair(input_mint, output_mint))
            .filter_map(|pool| pool.calculate_quote(amount_in))
            .max_by_key(|q| q.output_amount);

        metrics::record_latency("quote_calculation", start.elapsed());
        best_quote
    }
}
```

**Performance Optimizations:**
1. **Memory-Mapped Pools:** Keep hot pools in L1/L2 cache
2. **SIMD Instructions:** Use AVX2/AVX-512 for parallel math
3. **Lock-Free Reads:** Read pool state without locking
4. **Concurrent Calculation:** Quote all pools in parallel
5. **Precomputed Constants:** Cache fee multipliers

---

### 3. Strategy Engine

#### 3.1 Cross-DEX Arbitrage (Primary Strategy)
```typescript
class CrossDexArbitrageStrategy {
  async detectOpportunity(): Promise<ArbitrageOpportunity | null> {
    // Get quotes from multiple DEXes concurrently
    const quotes = await Promise.all([
      this.quoteEngine.getQuote("SOL", "USDC", this.testAmount, { dex: "raydium" }),
      this.quoteEngine.getQuote("SOL", "USDC", this.testAmount, { dex: "orca" }),
      this.quoteEngine.getQuote("SOL", "USDC", this.testAmount, { dex: "meteora" }),
    ]);

    const bestBuy = quotes.reduce((best, q) =>
      q.outputAmount > best.outputAmount ? q : best
    );

    // Get reverse quotes
    const reverseQuotes = await Promise.all([
      this.quoteEngine.getQuote("USDC", "SOL", bestBuy.outputAmount, { dex: "raydium" }),
      this.quoteEngine.getQuote("USDC", "SOL", bestBuy.outputAmount, { dex: "orca" }),
      this.quoteEngine.getQuote("USDC", "SOL", bestBuy.outputAmount, { dex: "meteora" }),
    ]);

    const bestSell = reverseQuotes.reduce((best, q) =>
      q.outputAmount > best.outputAmount ? q : best
    );

    // Calculate profit
    const profit = bestSell.outputAmount - this.testAmount;
    const profitPercent = (profit / this.testAmount) * 100;

    if (profitPercent > 0.05) { // 5 basis points minimum
      return {
        type: "cross_dex",
        buyLeg: bestBuy,
        sellLeg: bestSell,
        expectedProfit: profit - this.calculateFees(bestBuy, bestSell),
        profitPercent,
      };
    }

    return null;
  }
}
```

#### 3.2 Triangular Arbitrage
```typescript
class TriangularArbitrageStrategy {
  async detectOpportunity(): Promise<ArbitrageOpportunity | null> {
    // Three-hop arbitrage: SOL → USDC → TOKEN → SOL
    const [quote1, quote2, quote3] = await Promise.all([
      this.quoteEngine.getQuote("SOL", "USDC", this.testAmount),
      this.quoteEngine.getQuote("USDC", "TOKEN", estimatedOutput1),
      this.quoteEngine.getQuote("TOKEN", "SOL", estimatedOutput2),
    ]);

    const profit = quote3.outputAmount - this.testAmount;
    const profitPercent = (profit / this.testAmount) * 100;

    // Account for fees
    const flashLoanFee = this.testAmount * 0.0005; // Kamino 0.05%
    const jitoTip = 5000 + Math.random() * 5000;
    const gasFee = 10000;
    const netProfit = profit - flashLoanFee - jitoTip - gasFee;

    if (profitPercent > 0.1) {
      return {
        type: "triangular",
        legs: [quote1, quote2, quote3],
        expectedProfit: netProfit,
        profitPercent,
      };
    }

    return null;
  }
}
```

---

### 4. Execution Layer (Flash Loans + Jito)

#### Flash Loan Transaction Structure
```typescript
async function buildFlashLoanArbitrageTx(
  opportunity: ArbitrageOpportunity,
  wallet: Keypair,
): Promise<VersionedTransaction> {
  const instructions: TransactionInstruction[] = [];

  // 1. Compute budget
  instructions.push(
    ComputeBudgetProgram.setComputeUnitLimit({ units: 600_000 }),
    ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 2_000_000 })
  );

  // 2. Flash loan borrow
  const { flashBorrowIx, borrowIndex } = await buildKaminoFlashBorrowIx({
    owner: wallet.publicKey,
    reserve: USDC_RESERVE,
    amount: opportunity.requiredAmount,
  });
  instructions.push(flashBorrowIx);

  // 3. Arbitrage swaps
  for (const leg of opportunity.legs) {
    const swapIx = await buildJupiterSwapIx({
      wallet: wallet.publicKey,
      inputMint: leg.inputMint,
      outputMint: leg.outputMint,
      amount: leg.inputAmount,
      slippageBps: 50,
    });
    instructions.push(...swapIx);
  }

  // 4. Flash loan repay
  const flashRepayIx = await buildKaminoFlashRepayIx({
    owner: wallet.publicKey,
    reserve: USDC_RESERVE,
    amount: opportunity.requiredAmount,
    borrowIndex,
  });
  instructions.push(flashRepayIx);

  // 5. Build versioned transaction with ALT
  const message = TransactionMessage.compile({
    payerKey: wallet.publicKey,
    instructions,
    recentBlockhash: await this.blockhashCache.get(),
    addressLookupTableAccounts: [
      await fetchAddressLookupTable(DEX_ALT_1),
      await fetchAddressLookupTable(DEX_ALT_2),
    ],
  });

  const transaction = new VersionedTransaction(message);
  transaction.sign([wallet]);

  return transaction;
}
```

#### Jito Bundle Submission
```typescript
class JitoBundleExecutor {
  async submitBundle(
    transactions: VersionedTransaction[],
    options: { tip?: number; urgency: "low" | "medium" | "high" }
  ): Promise<BundleResult> {
    const tip = options.tip || this.calculateOptimalTip(options.urgency);
    const tipAccount = await this.jitoClient.getRandomTipAccount();

    const tipTx = new Transaction().add(
      SystemProgram.transfer({
        fromPubkey: this.wallet.publicKey,
        toPubkey: tipAccount,
        lamports: tip,
      })
    );

    const bundleId = await this.jitoClient.sendBundle({
      transactions: [tipTx, ...transactions],
      skipPreflightValidation: false,
    });

    return await this.monitorBundleConfirmation(bundleId, 30_000);
  }

  calculateOptimalTip(urgency: string): number {
    const base = 10_000;
    const multipliers = { low: 1, medium: 3, high: 10 };
    const randomness = Math.random() * base * 0.5;
    return base * multipliers[urgency] + randomness;
  }
}
```

---

### 5. Risk Management & Circuit Breakers

```typescript
class RiskManager {
  private consecutiveFailures = 0;
  private dailyLoss = 0;
  private positionLimits = new Map<string, number>();

  async validateTrade(opportunity: TradeOpportunity): Promise<boolean> {
    // 1. Circuit breaker - stop after 5 consecutive failures
    if (this.consecutiveFailures >= 5) {
      logger.error("Circuit breaker triggered");
      await this.notifyAdmin("Circuit breaker triggered");
      return false;
    }

    // 2. Daily loss limit - stop after losing 0.5 SOL
    if (this.dailyLoss > 0.5 * LAMPORTS_PER_SOL) {
      logger.error("Daily loss limit exceeded");
      return false;
    }

    // 3. Position size limit
    if (opportunity.amount > this.getPositionLimit(opportunity.tokenPair)) {
      logger.warn("Position size exceeds limit");
      return false;
    }

    // 4. Minimum profit threshold
    const minProfit = this.calculateMinProfitThreshold();
    if (opportunity.expectedProfit < minProfit) {
      return false;
    }

    // 5. Price impact check
    if (opportunity.priceImpact > 0.05) {
      logger.warn("Price impact too high");
      return false;
    }

    return true;
  }

  onTradeSuccess(profit: number) {
    this.consecutiveFailures = 0;
    this.dailyProfit += profit;
  }

  onTradeFailure(loss: number) {
    this.consecutiveFailures++;
    this.dailyLoss += loss;
  }
}
```

---

## 4-Week Optimization Roadmap

### Week 1: Local Quote Engine (1.7s → 800ms)

**Goal:** Replace Jupiter API with local pool math

**Tasks:**
1. ✅ Deploy Go quote service with 30s cache
2. ✅ Implement concurrent quote calculation
3. ✅ Cache blockhash (50ms saved)
4. ✅ Parallel quote fetching (2x faster)

**Quick Wins (Start Here):**

**1. Cache Blockhash** (30 minutes, 50ms saved)
```typescript
class BlockhashCache {
  private cachedBlockhash: string | null = null;
  private lastFetch: number = 0;

  async get(connection: Connection): Promise<string> {
    const now = Date.now();
    if (!this.cachedBlockhash || now - this.lastFetch > 20_000) {
      const { blockhash } = await connection.getLatestBlockhash();
      this.cachedBlockhash = blockhash;
      this.lastFetch = now;
    }
    return this.cachedBlockhash;
  }
}
```

**2. Parallel Quote Fetching** (1 hour, 2x faster)
```typescript
// Before: Sequential
const quote1 = await getQuote(inputA, outputB, amount);
const quote2 = await getQuote(outputB, inputA, quote1.output);

// After: Parallel
const [quote1, quote2] = await Promise.all([
  getQuote(inputA, outputB, amount),
  getQuote(outputB, inputA, estimatedOutput)
]);
```

**3. Batch RPC Calls** (2 hours, 10x faster)
```typescript
// Before: Sequential
for (const poolId of poolIds) {
  const pool = await connection.getAccountInfo(poolId);
  pools.push(pool);
}

// After: Batch
const pools = await connection.getMultipleAccountsInfo(poolIds);
```

**Expected Impact:** 1.7s → 800ms

---

### Week 2: Shredstream Integration (800ms → 500ms)

**Goal:** Get market updates 400ms before other bots

**Implementation:**
```typescript
class ShredstreamClient {
  private ws: WebSocket;

  async connect() {
    this.ws = new WebSocket("wss://shredstream.jito.wtf");

    this.ws.send(JSON.stringify({
      method: "subscribe",
      params: {
        accounts: [
          "58oQChx4yWmvKdwLLZzBi4ChoCc2fqCUWBkwMihLYQo2", // SOL/USDC Raydium
          // Add top 50 pools
        ]
      }
    }));
  }

  onAccountUpdate(pubkey: string, handler: (data: AccountUpdate) => void) {
    this.handlers.set(pubkey, handler);
  }
}
```

**Expected Impact:** 400ms early alpha + faster opportunity detection

---

### Week 3: Flash Loan Optimization (500ms → 300ms)

**Goal:** Zero capital arbitrage with optimized Kamino integration

**Key Points:**
- Kamino charges 0.05% fee
- Must repay in same transaction
- High compute budget required (600k-800k units)

**Expected Impact:** Zero capital required, larger position sizes

---

### Week 4: Jito Bundle Optimization (300ms → 200ms)

**Goal:** 95% bundle landing rate

**Dynamic Tip Strategy:**
```typescript
class JitoTipOptimizer {
  calculateOptimalTip(urgency: "low" | "medium" | "high"): number {
    const bases = { low: 5_000, medium: 20_000, high: 100_000 };
    const baseTip = bases[urgency];

    const avgLandingRate = this.getAvgLandingRate();
    if (avgLandingRate < 0.8) {
      return baseTip * 1.5; // Increase tip
    } else if (avgLandingRate > 0.95) {
      return baseTip * 0.8; // Decrease tip
    }

    return baseTip;
  }
}
```

**Expected Impact:** 90-95% landing rate, optimized tip costs

---

## Performance Targets

### Current State (Prototype)
```
Market Event → Quote (Jupiter API) → Decision → Execution
   500ms          150ms                50ms        1000ms

Total: ~1.7 seconds
```

### After 4 Weeks (Optimized)
```
Market Event → Quote (Local) → Decision → Execution
   50ms           5ms            20ms       100ms

Total: 175ms (10x improvement)
```

### Stretch Goal (Full HFT)
```
Shredstream → Quote (Rust) → Decision → Jito
   20ms          2ms           10ms       50ms

Total: 82ms (20x improvement)
```

---

## Technology Stack

### Core Services

| Component | Technology | Justification |
|-----------|-----------|---------------|
| **Market Data** | Rust | Zero-copy parsing, lock-free queues |
| **Quote Engine** | Go | Concurrent calculations, < 10ms latency |
| **Strategy Engine** | TypeScript | Balance speed and flexibility |
| **Execution** | TypeScript | Rich Solana SDK, Jito integration |
| **Risk Management** | TypeScript | Business logic, easy to modify |

### Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Hot Cache** | Redis (in-memory) | Sub-ms pool state access |
| **Event Bus** | NATS | Low-latency pub/sub |
| **Metrics** | Prometheus | High-cardinality metrics |
| **Tracing** | Jaeger | Latency profiling |
| **Logging** | Loki | Centralized logs |

---

## Deployment Architecture

```
┌──────────────────────────────────────────────────────────┐
│         Bare Metal Server (Colocation near Solana)        │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Market Data │  │ Quote Engine│  │  Strategy   │     │
│  │  (Rust)     │  │   (Go)      │  │  (TS/Rust)  │     │
│  │ CPU: 2 cores│  │ CPU: 4 cores│  │ CPU: 2 cores│     │
│  │ RAM: 2GB    │  │ RAM: 8GB    │  │ RAM: 4GB    │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐                       │
│  │  Executor   │  │     Redis   │                       │
│  │ (TypeScript)│  │  (in-memory)│                       │
│  │ CPU: 2 cores│  │ RAM: 4GB    │                       │
│  │ RAM: 4GB    │  │             │                       │
│  └─────────────┘  └─────────────┘                       │
│                                                           │
│  Total: 12 CPU cores, 24GB RAM                          │
└──────────────────────────────────────────────────────────┘
```

**Hardware Recommendations:**
- **CPU:** AMD Ryzen 9 7950X or Intel i9-13900K
- **RAM:** 32GB DDR5 (ECC preferred)
- **SSD:** 1TB NVMe (Samsung 990 Pro)
- **Network:** 10Gbps, <1ms to Solana RPC
- **Location:** Colocation near Solana validators

**Cost:** ~$300-500/month bare metal rental

---

## Monitoring & Alerting

### Critical Metrics
```typescript
const metrics = {
  // Latency (p50, p95, p99)
  shredstream_latency: new Histogram({
    name: "shredstream_latency_ms",
    buckets: [10, 25, 50, 100, 200, 500],
  }),

  quote_latency: new Histogram({
    name: "quote_latency_ms",
    buckets: [1, 2, 5, 10, 20, 50],
  }),

  execution_latency: new Histogram({
    name: "execution_latency_ms",
    buckets: [50, 100, 200, 500, 1000],
  }),

  // Success metrics
  opportunities_detected: new Counter({
    name: "opportunities_detected_total",
    labelNames: ["strategy"],
  }),

  trades_executed: new Counter({
    name: "trades_executed_total",
    labelNames: ["strategy", "status"],
  }),

  profit_realized: new Gauge({
    name: "profit_realized_lamports",
    labelNames: ["strategy"],
  }),

  // System health
  consecutive_failures: new Gauge({
    name: "consecutive_failures",
  }),
};
```

### Alerting Rules
```yaml
groups:
  - name: hft_alerts
    rules:
      - alert: SystemDown
        expr: up{job="hft"} == 0
        for: 30s

      - alert: HighQuoteLatency
        expr: histogram_quantile(0.95, quote_latency_ms) > 20
        for: 1m

      - alert: CircuitBreakerTriggered
        expr: consecutive_failures >= 5

      - alert: LowSuccessRate
        expr: |
          rate(trades_executed_total{status="success"}[5m]) /
          rate(trades_executed_total[5m]) < 0.8
        for: 5m
```

---

## Advanced Techniques (Future Enhancements)

### 1. Memory-Mapped Pool State (Rust)
```rust
use memmap2::MmapMut;

struct PoolStateStore {
    mmap: MmapMut,
    index: HashMap<Pubkey, usize>,
}

impl PoolStateStore {
    fn get_pool(&self, pubkey: &Pubkey) -> Option<&PoolState> {
        let index = self.index.get(pubkey)?;
        unsafe {
            let ptr = self.mmap.as_ptr() as *const PoolState;
            Some(&*ptr.add(*index))
        }
    }
}
```

### 2. SIMD Quote Calculations
```rust
#[cfg(target_feature = "avx2")]
pub fn calculate_quotes_simd(
    base_reserves: &[u64; 8],
    quote_reserves: &[u64; 8],
    amount_in: u64,
) -> [u64; 8] {
    // Calculate 8 quotes simultaneously using AVX2
    // 8x speedup for batch operations
}
```

### 3. Multi-Wallet Parallel Execution
```typescript
class ParallelExecutor {
  private walletPool: Wallet[] = [];

  async executeParallel(opportunities: Opportunity[]): Promise<void> {
    const executions = opportunities.map(async (opp) => {
      const wallet = await this.acquireWallet();
      try {
        await this.execute(opp, wallet);
      } finally {
        this.releaseWallet(wallet);
      }
    });

    await Promise.all(executions);
  }
}
```

---

## Implementation Timeline

### Phase 1: Hot Path (Weeks 1-4) - **START HERE**
1. ✅ Local quote engine (Go) - Week 1
2. ✅ Shredstream integration - Week 2
3. ✅ Flash loan optimization - Week 3
4. ✅ Jito bundle executor - Week 4

### Phase 2: Advanced Strategies (Weeks 5-8)
1. ✅ Triangular arbitrage (3-leg)
2. ✅ Cross-DEX arbitrage (Raydium, Orca, Meteora)
3. ✅ Statistical arbitrage (mean reversion)
4. ✅ Market making (optional)

### Phase 3: Production Hardening (Weeks 9-12)
1. ✅ Risk management & circuit breakers
2. ✅ Monitoring & alerting
3. ✅ Performance profiling & optimization
4. ✅ Bare metal deployment

**Total Time:** 12 weeks full-time to production HFT system

---

## Measurement & Profiling

### Latency Tracking
```typescript
class LatencyTracker {
  private spans: Map<string, number> = new Map();

  start(name: string) {
    this.spans.set(name, performance.now());
  }

  end(name: string): number {
    const start = this.spans.get(name);
    if (!start) return 0;

    const latency = performance.now() - start;
    metrics.recordLatency(name, latency);
    return latency;
  }

  async measure<T>(name: string, fn: () => Promise<T>): Promise<T> {
    this.start(name);
    try {
      return await fn();
    } finally {
      this.end(name);
    }
  }
}
```

---

## Summary

This initial architecture provides a clear path from a working prototype (~1.7s) to a production HFT system (< 200ms):

**Week 1:** Local quote engine → 1.7s → 800ms
**Week 2:** Shredstream integration → 800ms → 500ms
**Week 3:** Flash loan optimization → 500ms → 300ms
**Week 4:** Jito bundle optimization → 300ms → 200ms

**Result:** 8-10x improvement in 4 weeks, with a foundation for further optimizations.

**Next Step:** Start with "Quick Wins" (blockhash cache, parallel fetching, batch RPC) for immediate 2-3x improvement this week.
