---
layout: single
title: "Pool Discovery Service: Triangular Arbitrage Support and Production Insights"
date: 2025-12-29
permalink: /posts/2025/12/pool-discovery-triangular-arbitrage-support-production-insights/
categories:
  - blog
tags:
  - solana
  - trading
  - pool-discovery
  - arbitrage
  - triangular-arbitrage
  - architecture
  - observability
  - grafana
excerpt: "Pool Discovery Service reaches production milestone with triangular arbitrage support (45 token pairs), completing the Quote Service Rewrite architecture. Observability reveals surprising market insights: SOL/USDT has 278B liquidity (vs SOL/USDC 226B), and forward/reverse pool discovery yields asymmetric results."
---

## TL;DR

Today marks the completion of pool-discovery-service with triangular arbitrage support, a critical component of the Quote Service Rewrite:

1. **Triangular Arbitrage Support**: 45 token pairs (14 LST/SOL + 14 LST/USDC + 14 LST/USDT + SOL/stablecoins + USDC/USDT)
2. **Quote Service Rewrite Progress**: Completed solana-rpc-proxy and pool-discovery-service (2 of 3 components)
3. **10K+ Pool Discovery Capability**: Can technically support 10,000+ pairs (Priority 3: Protocol Expansion), but **Full Pool Indexer NOT Recommended for LST HFT**
4. **Performance Improvements**: WebSocket-first architecture (99% RPC reduction), 8-hour crash recovery (5s vs 5-10min)
5. **Production Dashboard Insights**: Grafana metrics reveal surprising market structure—SOL/USDT pools have MORE liquidity than SOL/USDC
6. **Asymmetric Pool Discovery**: Forward direction discovers more pools than reverse direction
7. **Not All LSTs Have Direct Pools**: Some LST tokens lack LST/SOL pairs, requiring multi-hop routing

The observability-driven approach proved critical: Grafana dashboards revealed architectural flaws and market insights that informed design decisions for the upcoming quote-service rewrite.

---

## Table of Contents

1. [Quote Service Rewrite: Completion Status](#quote-service-rewrite-completion-status)
2. [Triangular Arbitrage Support](#triangular-arbitrage-support)
3. [10K+ Pool Discovery: Why We Won't Use It](#10k-pool-discovery-why-we-wont-use-it)
4. [Performance Improvements](#performance-improvements)
5. [Production Dashboard Insights](#production-dashboard-insights)
6. [Surprising Market Structure Discoveries](#surprising-market-structure-discoveries)
7. [Observability: The Key to Design Excellence](#observability-the-key-to-design-excellence)
8. [Next Steps: Quote Service Rewrite](#next-steps-quote-service-rewrite)
9. [Conclusion](#conclusion)

---

## Quote Service Rewrite: Completion Status

As outlined in our [Quote Service Rewrite plan](/posts/2025/12/quote-service-rewrite-clean-architecture-for-maintainability/), we identified three critical services needed for the clean architecture:

**Service Separation Progress:**

| Service | Status | Purpose | Completion |
|---------|--------|---------|------------|
| **1. Solana RPC Proxy** | ✅ Complete | Centralized RPC management with failover | [Dec 28](/posts/2025/12/pool-discovery-service-rpc-proxy-architecture/) |
| **2. Pool Discovery Service** | ✅ **Complete** | Independent pool scanning and caching | **Today** |
| **3. Quote Service** | 🚧 In Progress | Clean architecture, HFT-ready quotes | **Next** |

**Today's Milestone:** Pool-discovery-service is production-ready with full triangular arbitrage support. This completes the infrastructure layer, allowing us to focus on the quote-service rewrite itself—the engine core of our [HFT trading system](/posts/2025/12/quote-service-architecture-hft-engine-core/).

---

## Triangular Arbitrage Support

### The Problem: Incomplete Arbitrage Path Coverage

Our previous pool discovery only covered **LST/SOL pairs** (14 pairs), which limited arbitrage strategies to simple two-hop trades:

```
USDC → SOL → JitoSOL → SOL → USDC (inefficient)
```

**What was missing:**
- LST/USDC direct pairs (skip SOL intermediate)
- LST/USDT pairs (alternative stablecoin routes)
- SOL/USDC and SOL/USDT pairs (triangular arbitrage anchors)
- USDC/USDT pair (cross-stablecoin arbitrage)

### The Solution: 45-Pair Triangular Coverage

The service now supports `TRIANGULAR` mode, discovering all pairs needed for complete arbitrage path finding:

**Token Pair Breakdown:**

| Pair Type | Count | Example | Purpose |
|-----------|-------|---------|---------|
| **LST/SOL** | 14 | JitoSOL/SOL | Core LST liquidity |
| **LST/USDC** | 14 | JitoSOL/USDC | Direct LST-to-stablecoin |
| **LST/USDT** | 14 | mSOL/USDT | Alternative stablecoin routes |
| **SOL/USDC** | 1 | SOL/USDC | Triangular anchor |
| **SOL/USDT** | 1 | SOL/USDT | Triangular anchor |
| **USDC/USDT** | 1 | USDC/USDT | Cross-stablecoin arbitrage |
| **TOTAL** | **45** | | **Complete coverage** |

### Example Triangular Paths

**Path 1: USDC → SOL → LST → USDC** (Three-hop)
```
1. USDC → SOL (Pool: SOL/USDC, Raydium AMM)
2. SOL → JitoSOL (Pool: JitoSOL/SOL, Raydium CLMM)
3. JitoSOL → USDC (Pool: JitoSOL/USDC, Orca Whirlpool)

Result: Start USDC, end USDC (profit if spread exists)
```

**Path 2: USDT → LST → USDC → USDT** (Cross-stablecoin via LST)
```
1. USDT → mSOL (Pool: mSOL/USDT, Raydium CPMM)
2. mSOL → USDC (Pool: mSOL/USDC, Meteora DLMM)
3. USDC → USDT (Pool: USDC/USDT, Raydium AMM)

Result: Exploit USDC/USDT spread through LST intermediary
```

**Path 3: Direct Stablecoin Arbitrage**
```
1. USDC → SOL (Pool: SOL/USDC, Raydium AMM)
2. SOL → USDT (Pool: SOL/USDT, Orca Whirlpool)

Result: Simple two-hop stablecoin arbitrage
```

### Configuration

**Command Line:**
```bash
pool-discovery-service -pairs TRIANGULAR
```

**Docker Compose:**
```yaml
pool-discovery-service:
  command: >
    /app/pool-discovery-service
    -pairs TRIANGULAR
    -discovery-interval 300
    -pool-ttl 28800
```

**Performance:**
- Discovery time: ~45-60 seconds (540 RPC queries)
- WebSocket updates: <1 second (real-time)
- Expected pools: ~270 (45 pairs × ~6 pools/pair)

---

## 10K+ Pool Discovery: Why We Won't Use It

### Technical Capability vs. Practical Strategy

The pool-discovery-service **can technically support 10,000+ token pairs** through protocol expansion:

**Scalability Tiers:**

| Priority | Scope | Pairs | Use Case | Status |
|----------|-------|-------|----------|--------|
| **Priority 1: LST HFT** | 14 LST/SOL | 14 | LST arbitrage only | ✅ Production |
| **Priority 2: Triangular** | Full triangular | 45 | Complete arbitrage | ✅ **Today** |
| **Priority 3: Protocol Expansion** | Additional DEXes | 100-500 | Broader coverage | ⚠️ Possible |
| **Priority 4: Full Indexer** | ALL tokens | 10,000+ | Universal indexing | ❌ **NOT RECOMMENDED** |

### Why Full Pool Indexer Is NOT Recommended

**Problem 1: Performance Degradation**
```
45 pairs × 6 DEXes × 2 directions = 540 RPC queries (~50s)
10,000 pairs × 6 DEXes × 2 directions = 120,000 RPC queries (~2-3 hours!)
```

**Problem 2: HFT Focus Dilution**
- LST tokens have predictable liquidity (high TVL, stable)
- Long-tail tokens are illiquid, high-slippage, risky
- HFT strategies require **quality over quantity**

**Problem 3: RPC Load**
- WebSocket subscriptions: 45 pairs = ~270 subscriptions (manageable)
- WebSocket subscriptions: 10,000 pairs = ~60,000 subscriptions (RPC overwhelmed)

**Problem 4: Redis Memory**
- Current: 45 pairs × 300 bytes = ~81 KB
- Full indexer: 10,000 pairs × 300 bytes = ~18 MB (100x increase)

**Our Decision: Focus on LST HFT**

We're sticking with **45-pair triangular mode** because:
- ✅ Covers all arbitrage paths for LST tokens
- ✅ Maintains <1s WebSocket update latency
- ✅ Keeps RPC load manageable
- ✅ Focuses on high-liquidity, low-slippage opportunities
- ✅ Aligns with HFT strategy (quality > quantity)

**Quote from design doc:**
> "Full Pool Indexer NOT Recommended for LST HFT"

---

## Performance Improvements

### WebSocket-First Architecture (99% RPC Reduction)

**Before (RPC Polling):**
```
45 pairs × 6 DEXes × 2 directions × 120 polls/hour = 64,800 RPC calls/hour
```

**After (WebSocket-First):**
```
Initial: 540 RPC queries (one-time startup)
WebSocket: Real-time updates (<100ms latency)
RPC Backup: 540 queries every 5 minutes (only if WebSocket down)
Total: ~540-1,080 RPC calls/hour (99% reduction!)
```

### 8-Hour Crash Recovery (5s vs 5-10min)

**Before (10-minute TTL):**
```
Service crashes → Redis cache expires in 10 min → Full re-scan (5-10 min)
Downtime: 15-20 minutes
```

**After (8-hour TTL):**
```
Service crashes → Redis cache valid for 8 hours → Instant restore from Redis
Downtime: 5 seconds
```

**Benefits:**
- ✅ 120-180x faster recovery (5s vs 15-20min)
- ✅ Service operational immediately
- ✅ No re-scanning on restart (unless cache stale)

### Bidirectional Discovery (2x Pool Coverage)

**Problem:** Original implementation only queried one direction:
```
FetchPoolsByPair(SOL, USDC)
→ Finds: BaseMint=SOL, QuoteMint=USDC
→ Misses: BaseMint=USDC, QuoteMint=SOL (50% of pools!)
```

**Solution:** Query both directions:
```
FetchPoolsByPair(SOL, USDC) // Forward
FetchPoolsByPair(USDC, SOL) // Reverse
→ Deduplicate by pool ID
→ Result: 2x pool discovery
```

---

## Production Dashboard Insights

The [pool-discovery-lst-pairs.json](https://github.com/guidebee/solana-trading-system/blob/master/deployment/monitoring/grafana/provisioning/dashboards/pool-discovery-lst-pairs.json) dashboard tracks critical metrics for pool discovery:

![Pool Discovery LST Pairs Dashboard](../images/pool-discovery-lst-pairs.png)

### Key Metrics Tracked

**1. Total LST Pairs**
- Current: 45 pairs (triangular mode)
- Query: `count(count by (base_mint, quote_mint) (pool_discovery_pools_by_dex))`
- **Why useful:** Confirms token pair configuration is correct

**2. Total Pools Discovered**
- Current: ~270 pools (45 pairs × ~6 pools/pair or 870 pools multiple pools for same pair)
- Query: `pool_discovery_pools_total`
- **Why useful:** Validates bidirectional discovery is working

**3. Pools by DEX Protocol**
- Raydium AMM: ~50 pools
- Raydium CLMM: ~60 pools
- Meteora DLMM: ~40 pools
- Orca Whirlpool: ~50 pools
- **Why useful:** Identifies protocol coverage gaps

**4. Pool Discovery Duration (p50, p95, p99)**
- p50: ~30 seconds
- p95: ~50 seconds
- p99: ~60 seconds
- **Why useful:** Detects RPC performance degradation

**5. WebSocket Update Rate**
- Real-time: <100ms latency
- Update frequency: ~10-50 updates/min
- **Why useful:** Ensures WebSocket-first architecture is working

**6. Forward vs. Reverse Pool Counts**
- Forward direction: 150-180 pools
- Reverse direction: 90-120 pools
- **Why useful:** Reveals asymmetric pool distribution (see next section)

### Design Insights from Dashboard

**Insight 1: Observability Reveals Architectural Flaws**

Before implementing the dashboard, we had **no visibility** into:
- How many pools were discovered per direction (forward vs reverse)
- Which DEX protocols had coverage gaps
- Whether WebSocket subscriptions were actually working
- RPC performance bottlenecks

**After implementing the dashboard:**
- Discovered bidirectional discovery was missing (50% pool loss)
- Identified Pump.fun pools were being skipped (parser bug)
- Found WebSocket reconnection was failing silently

**Lesson:** "You can't improve what you can't measure"

**Insight 2: Metrics Guide Quote Service Design**

The dashboard metrics directly inform the upcoming quote-service rewrite:

| Metric | Design Decision |
|--------|-----------------|
| **Pool count per pair** | Cache sizing (270 pools = ~81 KB Redis) |
| **Discovery duration** | Quote cache TTL (30s vs 5min trade-off) |
| **WebSocket update rate** | Cache invalidation strategy (event-driven vs polling) |
| **Forward/reverse asymmetry** | Router needs to check BOTH directions |

**Lesson:** Good observability enables data-driven architecture

---

## Surprising Market Structure Discoveries

### Discovery 1: SOL/USDT > SOL/USDC Liquidity

**The Assumption:**
> "SOL/USDC is the most liquid pool on Solana"

**The Reality:**
```
SOL/USDT Pool:  278B liquidity
SOL/USDC Pool:  226B liquidity

SOL/USDT is 23% MORE liquid than SOL/USDC! 🤯
```

**Why This Matters:**
- Quote-service should prioritize SOL/USDT for routing
- Scanner should monitor SOL/USDT more frequently
- Arbitrage strategies should include USDT routes

**Why This Happened:**
- USDT is more popular in Asian markets (Binance preference)
- SOL/USDT pools have higher trading volume
- Some whales prefer USDT over USDC

**Lesson:** Assumptions must be validated with data

### Discovery 2: Not All LSTs Have Direct SOL Pools

**The Assumption:**
> "Every LST token has an LST/SOL pool"

**The Reality:**

Some LST tokens lack direct LST/SOL pairs:
- **bSOL** (BlazeStake): No direct bSOL/SOL pool found
- **dSOL** (Drift): Limited liquidity, only on Meteora DLMM
- **laineSOL** (Laine): Only found on Orca Whirlpool

**Why This Matters:**
- Quote-service must handle "no pool found" gracefully
- Scanner must support multi-hop routing (LST → USDC → SOL)
- Not all LSTs are equally liquid

**Lesson:** LST market is fragmented, not uniform

### Discovery 3: Forward vs. Reverse Pool Asymmetry

**The Assumption:**
> "Forward and reverse queries should find the same pools"

**The Reality:**

```
Forward direction (SOL → USDC):  180 pools
Reverse direction (USDC → SOL):  120 pools

Forward finds 50% MORE pools than reverse! 🤔
```

**Why This Happened:**

DEX protocols store pools with **canonical token ordering**:
- Raydium AMM: BaseMint < QuoteMint (lexicographic order)
- Meteora DLMM: TokenX < TokenY (address comparison)
- Orca Whirlpool: TokenA < TokenB (program convention)

**Example:**
```
Pool 1: BaseMint=SOL, QuoteMint=USDC (canonical)
Pool 2: BaseMint=USDC, QuoteMint=SOL (non-canonical, rare)

Forward query (SOL → USDC): Finds Pool 1 ✅
Reverse query (USDC → SOL): Finds Pool 2 ✅
Bidirectional query: Finds BOTH ✅
```

**Why This Matters:**
- Unidirectional queries miss 30-50% of pools
- Quote-service router must try BOTH directions
- Pool discovery must deduplicate by pool ID

**Lesson:** DEX protocol conventions create asymmetric liquidity

---

## Observability: The Key to Design Excellence

### Observability Takes More Time, But It's Worth It

**Time Investment:**
- Writing code: 40% of time
- Writing observability (Prometheus metrics, Grafana dashboards, Loki logging): 40% of time
- Writing tests: 20% of time

**Why Observability Is Worth It:**

**1. Finds Design Flaws Early**

Without observability:
```
"Pool discovery seems slow, not sure why"
→ No visibility into RPC latency, DEX protocol timing, or WebSocket health
→ Can't identify bottlenecks
→ Guessing at optimizations
```

With observability:
```
Dashboard shows: Raydium CLMM takes 15s (3x slower than other DEXes)
→ Profile Raydium CLMM parser
→ Find unnecessary JSON parsing in hot path
→ Optimize: 15s → 5s (3x speedup)
```

**2. Validates Architectural Decisions**

Without observability:
```
"WebSocket-first architecture should reduce RPC load"
→ No metrics to confirm
→ Implement and hope
```

With observability:
```
Metric: rpc_backup_triggered_total = 0 (WebSocket healthy)
Metric: pool_update_source_total{source="websocket"} = 99%
→ Confirms 99% RPC reduction
→ Architecture validated ✅
```

**3. Enables Data-Driven Design**

The upcoming quote-service rewrite benefits from observability insights:

| Design Decision | Data Source |
|----------------|-------------|
| Cache TTL (30s) | Discovery duration (p95: 50s) |
| Pool count (270) | Total pools discovered metric |
| Redis memory (81 KB) | Pool count × avg size |
| Bidirectional routing | Forward/reverse asymmetry |
| SOL/USDT prioritization | Liquidity comparison (278B vs 226B) |

**Lesson:** Observability transforms guesswork into engineering

### Observability Is Challenging, But Rewarding

**Challenges:**
- Learning Prometheus PromQL query language
- Designing useful Grafana dashboards (signal vs noise)
- Structured logging without log spam
- Distributed tracing overhead

**Rewards:**
- High visibility into service behavior
- Rapid debugging (trace ID → logs → metrics → root cause)
- Confidence in production deployments
- Data-driven optimization

**Quote:**
> "Observability is like having X-ray vision for your system. It takes effort to build, but once you have it, you'll never want to go back."

---

## Next Steps: Quote Service Rewrite

With pool-discovery-service complete, we can now focus on the **quote-service rewrite**—the engine core of our HFT trading system.

### Quote Service Architecture Goals

As outlined in our [Quote Service Rewrite plan](/posts/2025/12/quote-service-rewrite-clean-architecture-for-maintainability/):

**1. Clean Architecture (85% Code Reduction)**
```
Current: 50K lines (monolithic)
Target: 15K lines (clean architecture)
```

**2. Sub-10ms Cached Quotes**
```
Current: ~5ms cached, ~200ms uncached
Target: <10ms cached, <50ms uncached (via pool-discovery cache)
```

**3. 4x Better Test Coverage**
```
Current: 20% coverage (hard to test)
Target: 80%+ coverage (dependency injection)
```

**4. Service Separation**
```
Current: 1 monolith (quote + discovery + RPC management)
Target: 3 services (quote, pool-discovery ✅, RPC proxy ✅)
```

### Integration with Pool Discovery Service

Quote-service will consume pool discovery data:

**Data Flow:**
```
pool-discovery-service
    ↓ Redis (pool metadata)
quote-service
    ↓ Calculate quotes using cached pools
    ↓ Publish NATS events (FlatBuffers)
scanner-service
```

**Benefits:**
- ✅ No RPC calls in quote calculation (instant <10ms)
- ✅ Real-time pool updates (WebSocket-driven cache invalidation)
- ✅ Clean separation (quote-service doesn't manage pools)
- ✅ Bidirectional routing (thanks to pool-discovery asymmetry fix)

### Technology Stack

**Decided: Go for Quote Service** (from rewrite plan)
- Fast delivery (2-3 weeks vs 6-8 weeks in Rust)
- Proven technology (existing codebase to refactor)
- Performance target easily met (<10ms with Go)

**Decided: Combined HTTP + gRPC** (from rewrite plan)
- Shared in-memory cache (4-7x faster than Redis)
- Simpler deployment (1 service vs 2)
- HFT-critical latency (<10ms requires shared cache)

---

## Conclusion

Today marks a significant milestone: **pool-discovery-service is production-ready with full triangular arbitrage support**.

**What We Built:**
- ✅ 45-pair triangular arbitrage coverage
- ✅ WebSocket-first architecture (99% RPC reduction)
- ✅ 8-hour crash recovery (5s vs 5-10min)
- ✅ Bidirectional discovery (2x pool coverage)
- ✅ Comprehensive Grafana dashboard

**What We Learned:**
- 🎯 SOL/USDT has MORE liquidity than SOL/USDC (278B vs 226B)
- 🎯 Forward/reverse pool discovery is asymmetric (50% more pools in forward direction)
- 🎯 Not all LSTs have direct SOL pairs (requires multi-hop routing)
- 🎯 Observability is critical for finding design flaws
- 🎯 Data-driven design beats assumptions

**What's Next:**
- 🚀 Quote-service rewrite (clean architecture, <10ms quotes)
- 🚀 HFT pipeline integration (Stage 0: quote-service)
- 🚀 Scanner service (Stage 1: arbitrage detection)

**The Bottom Line:** Observability takes more time, but it's the difference between guessing and engineering. The insights from production dashboards directly informed our architecture decisions and revealed surprising market structure that will shape our HFT strategies.

Building robust, observable infrastructure is challenging but rewarding. With pool-discovery-service complete, we now have the foundation to rebuild quote-service the right way—with clean architecture, high test coverage, and sub-10ms latency.

---

## Impact

**Architectural Achievement:**
- ✅ Triangular arbitrage support (45 token pairs)
- ✅ WebSocket-first architecture (99% RPC reduction)
- ✅ 8-hour crash recovery (120-180x faster)
- ✅ Bidirectional discovery (2x pool coverage)
- ✅ Production-ready observability (Grafana + Prometheus + Loki)

**Business Insight:**
- 🎯 SOL/USDT more liquid than SOL/USDC (23% higher TVL)
- 🎯 Forward/reverse asymmetry (50% more pools in canonical direction)
- 🎯 LST market fragmentation (not all tokens have direct pools)

**Technical Foundation:**
- 🏗️ Quote Service Rewrite: 2 of 3 services complete
- 🏗️ Clean architecture foundation ready
- 🏗️ HFT pipeline infrastructure in place

---

## Related Posts

- [Pool Discovery Service: Real-Time Liquidity Tracking and Intelligent RPC Proxy](/posts/2025/12/pool-discovery-service-rpc-proxy-architecture/) - Pool discovery architecture (Dec 28)
- [Quote Service Rewrite: Clean Architecture for Maintainability](/posts/2025/12/quote-service-rewrite-clean-architecture-for-maintainability/) - Rewrite rationale (Dec 25)
- [Quote Service Architecture: The HFT Engine Core](/posts/2025/12/quote-service-architecture-hft-engine-core/) - Current architecture (Dec 22)

---

## Technical Documentation

- [Pool Discovery Design (docs/25-POOL-DISCOVERY-DESIGN.md)](https://github.com/guidebee/solana-trading-system) - Complete design doc
- [Quote Service Rewrite Plan (docs/26-QUOTE-SERVICE-REWRITE-PLAN.md)](https://github.com/guidebee/solana-trading-system) - Rewrite roadmap
- [Grafana Dashboard: pool-discovery-lst-pairs.json](https://github.com/guidebee/solana-trading-system/blob/master/deployment/monitoring/grafana/provisioning/dashboards/pool-discovery-lst-pairs.json) - Production metrics

---

## Connect

- GitHub: [@guidebee](https://github.com/guidebee)
- LinkedIn: [James Shen](https://www.linkedin.com/in/guidebee/)

---

*This is post #20 in the Solana Trading System development series. Pool-discovery-service is production-ready with triangular arbitrage support, completing the infrastructure layer for the Quote Service Rewrite. Observability-driven development revealed critical market insights that will shape our HFT strategies.*
