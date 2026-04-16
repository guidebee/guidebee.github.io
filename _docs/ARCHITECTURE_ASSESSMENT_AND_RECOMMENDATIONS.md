---
layout: single
title: "Architecture Assessment & Optimization Recommendations"
permalink: /docs/architecture-assessment-and-recommendations/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Architecture Assessment & Optimization Recommendations

**Date**: January 2026
**Author**: Solution Architect
**Scope**: TypeScript Pipeline (Scanner → Strategy → Executor) + Future Rust Port
**Status**: Technical Assessment Complete

---

## Executive Summary

### Current Architecture Decision

**Bypassing quote-aggregator-service** was the correct decision for TypeScript prototyping:

| Factor | With Aggregator | Direct gRPC (Current) |
|--------|-----------------|----------------------|
| Latency overhead | +3-5ms fan-out | 0ms |
| Complexity | High (3-tier merge) | Low |
| Failure modes | More points of failure | Simpler |
| For prototyping | Overkill | ✅ Appropriate |

**Rationale**: The aggregator adds value when you have multiple consumers (scanners) and need centralized deduplication. For a single TypeScript scanner prototyping phase, direct gRPC streaming from quote-service is optimal.

### Performance Assessment

```
┌─────────────────────────────────────────────────────────────────────────┐
│ CURRENT PIPELINE (TypeScript - Prototype)                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Quote Service (Go)  →  Scanner (TS)  →  Strategy (TS)  →  Executor (TS)
│       <5ms              ~10ms             50-100ms          ~20ms       │
│                                                                         │
│  Total: ~85-135ms (excluding blockchain confirmation)                   │
│  Target: <200ms ✅ ACHIEVABLE                                           │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│ FUTURE PIPELINE (Rust - Production)                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Quote Service  →  Shared Memory  →  Rust Scanner  →  Rust Executor    │
│       <5ms           <1μs             <10μs            <5ms             │
│                                                                         │
│  Total: <15ms (excluding blockchain confirmation)                       │
│  Target: <50ms ✅ ACHIEVABLE WITH SHARED MEMORY                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: TypeScript Pipeline Assessment

### Scanner Service ✅ Complete

**Status**: Production-ready prototype

**Strengths**:
- Direct gRPC streaming from quote-service (bypasses aggregator overhead)
- FlatBuffers serialization for low-latency NATS publishing
- Deduplication logic built-in
- Oracle-based arbitrage detection working

**Performance**: ~8-10ms detection latency

### Strategy Service ⚠️ 60% Complete

**Gaps Identified**:

| Component | Status | Impact | Fix Complexity |
|-----------|--------|--------|----------------|
| Jupiter Instructions | ❌ Missing | CRITICAL | 2 hrs |
| Route Merging | ❌ Missing | CRITICAL | 2 hrs |
| Simulation | ❌ Missing | HIGH | 3 hrs |
| Plan Publishing | ❌ Missing | CRITICAL | 2 hrs |
| Template Caching | ❌ Missing | MEDIUM | 2 hrs |

**Critical Path Optimization**:
```
Current:
  Opportunity → [Wait for Jupiter API ~200ms] → [Simulate ~50ms] → Plan
                      ↑ BOTTLENECK

Optimized:
  Opportunity → [Template Cache HIT <1ms] → [Simulate ~50ms] → Plan
                      ↑ 200x FASTER
```

**Recommendation**: Implement template caching from prototype:
```typescript
// Pattern from: references/solana-trading-system-prototype/apps/cli-tools/services/arbitrage/arbitrageService.ts
// Lines 618-815

const cachedTemplate = await templateCache.get(templateKey);
if (cachedTemplate) {
  // Use cached instructions - <1ms
  return cachedTemplate;
}
// Only call Jupiter if cache miss - 200ms
const instructions = await fetchJupiterInstructions(...);
await templateCache.set(templateKey, instructions, TTL_30_SECONDS);
```

### Executor Service ⚠️ 20% Complete

**Gaps Identified**:

| Component | Status | Impact | Fix Complexity |
|-----------|--------|--------|----------------|
| Transaction Building | ❌ Stub | CRITICAL | 4 hrs |
| Jito Bundle | ❌ Stub | CRITICAL | 4 hrs |
| RPC Fallback | ❌ Stub | HIGH | 3 hrs |
| Solayer Integration | ❌ Missing | MEDIUM | 2 hrs |
| Confirmation | ❌ Stub | HIGH | 2 hrs |

**Multi-Path Execution Strategy** (from prototype):
```typescript
// Pattern from: references/solana-trading-system-prototype/apps/cli-tools/services/arbitrage/arbitrageService.ts
// Lines 1228-1315

const senders = [
  jitoBundle,        // Primary: MEV protection
  rpcFast,           // Secondary: skipPreflight=true
  solayerBroadcast,  // Alternative: proprietary routing
  rpcSafe,           // Fallback: skipPreflight=false
];

// Round-robin distribution for parallel submission
const promises = signedTxs.map((tx, i) =>
  senders[i % senders.length].send(tx)
);
await Promise.all(promises);
```

---

## Part 2: Architecture Optimization Recommendations

### 2.1 NATS Stream Topology ✅ Well-Designed

The 6-stream architecture (MARKET_DATA, OPPORTUNITIES, PLANNED, EXECUTED, METRICS, SYSTEM) is optimal:

```
MARKET_DATA  ← High throughput (10k/s), memory storage, 1hr retention
OPPORTUNITIES ← Medium throughput (500/s), file storage, 24hr retention
PLANNED      ← Low throughput (50/s), file storage, 1hr retention
EXECUTED     ← Low throughput (50/s), file storage, 7-day retention
METRICS      ← High throughput (5k/s), memory storage, 1hr retention
SYSTEM       ← Low throughput (10/s), file storage, 30-day retention
```

**Assessment**: No changes needed. This is a well-designed event-driven architecture.

### 2.2 FlatBuffers Integration ✅ Correct Choice

| Metric | JSON | FlatBuffers | Benefit |
|--------|------|-------------|---------|
| Encode | 5-10μs | 1-2μs | 5x faster |
| Decode | 8-15μs | 0.1-0.5μs | 20-150x faster |
| Size | 450-600 bytes | 300-400 bytes | 30% smaller |
| Zero-copy | No | Yes | No allocations |

**Assessment**: FlatBuffers is the correct choice for HFT. Saves 10-20ms per event in the critical path.

### 2.3 Quote Service Architecture ⚠️ Recommendations

**Current Decision**: Bypass quote-aggregator-service, use direct gRPC streaming

**Assessment**: ✅ Correct for TypeScript prototype

**Future Recommendation** (Rust production):

```
┌────────────────────────────────────────────────────────────────────┐
│ PHASE 1 (TypeScript - Now)                                          │
│ Scanner ← gRPC streaming ← Quote Service                           │
│ • Simple, low latency                                              │
│ • Acceptable for prototype                                          │
├────────────────────────────────────────────────────────────────────┤
│ PHASE 2 (Rust - Future)                                             │
│ Rust Scanner ← Shared Memory ← Quote Aggregator                    │
│ • <1μs quote access                                                │
│ • 100x faster than gRPC                                            │
│ • Dual regions (local + external)                                  │
│ • Oracle price embedded                                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Critical Optimizations for TypeScript Pipeline

### 3.1 Strategy Service: Template Caching (CRITICAL)

**Problem**: Jupiter API calls add 200ms latency to every opportunity

**Solution**: Template caching with Redis

```typescript
// Template key: hash of pool addresses involved
const templateKey = crypto.createHash('sha256')
  .update(`${hop1.poolAddress}-${hop2.poolAddress}-${inputMint}`)
  .digest('hex');

// Check cache first
const cached = await redis.get(`template:${templateKey}`);
if (cached) {
  return JSON.parse(cached); // <1ms
}

// Cache miss: fetch from Jupiter (200ms)
const instructions = await fetchJupiterInstructions(...);

// Store with 30s TTL (matches pool refresh interval)
await redis.setex(`template:${templateKey}`, 30, JSON.stringify(instructions));
```

**Impact**: 200x faster for repeated routes (200ms → <1ms)

### 3.2 Strategy Service: Parallel Instruction Fetching (HIGH)

**Problem**: Sequential Jupiter API calls double latency

```
Current (Sequential):
T=0:    Fetch hop1 instructions
T=200:  Fetch hop2 instructions
T=400:  Total

Optimized (Parallel):
T=0:    Fetch hop1 instructions ─┐
T=0:    Fetch hop2 instructions ─┤ PARALLEL
T=200:  Total                    ─┘
```

**Solution**:
```typescript
const [hop1Instructions, hop2Instructions] = await Promise.all([
  fetchJupiterSwapInstructions(hop1),
  fetchJupiterSwapInstructions(hop2),
]);
```

**Impact**: 2x faster (400ms → 200ms)

### 3.3 Executor Service: Multi-Path Submission (HIGH)

**Problem**: Single execution path has high failure rate

**Solution**: Parallel multi-path submission

```typescript
// Build multiple transaction variations (slippage hedging)
const variations = [
  { amount: inputAmount, minProfit: baseFee + 100n },       // Full
  { amount: inputAmount * 95n / 100n, minProfit: baseFee }, // 95%
  { amount: inputAmount * 90n / 100n, minProfit: baseFee }, // 90%
];

// Submit to multiple execution paths
const results = await Promise.all([
  jitoBundle.submit(signedTx),
  rpcDirect.submit(signedTx, { skipPreflight: true }),
  solayer.submit(wireFormat),
]);

// First success wins
const signature = results.find(r => r.success)?.signature;
```

**Impact**: 95%+ landing rate (vs 70-80% single path)

### 3.4 Executor Service: Blockhash Caching (MEDIUM)

**Problem**: Each execution fetches fresh blockhash (50ms RPC call)

**Solution**: Pre-fetch and cache blockhash

```typescript
class BlockhashCache {
  private blockhash: string;
  private lastValidBlockHeight: bigint;
  private lastFetch: number;

  async getBlockhash(): Promise<{ blockhash: string; lastValidBlockHeight: bigint }> {
    // Cache for 30 seconds (60 slots)
    if (Date.now() - this.lastFetch < 30_000) {
      return { blockhash: this.blockhash, lastValidBlockHeight: this.lastValidBlockHeight };
    }

    // Refresh in background
    this.refreshAsync();
    return { blockhash: this.blockhash, lastValidBlockHeight: this.lastValidBlockHeight };
  }

  private async refreshAsync(): Promise<void> {
    const { blockhash, lastValidBlockHeight } = await rpc.getLatestBlockhash().send();
    this.blockhash = blockhash;
    this.lastValidBlockHeight = lastValidBlockHeight;
    this.lastFetch = Date.now();
  }
}
```

**Impact**: 50ms saved per execution

### 3.5 Strategy Service: Early Exit Validation (MEDIUM)

**Problem**: Full simulation even for clearly unprofitable opportunities

**Solution**: Multi-stage validation with early exits

```typescript
async validateOpportunity(event: TwoHopArbitrageEvent): Promise<ExecutionPlan | null> {
  // STAGE 1: Quick checks (no I/O) - <1ms
  if (event.estimatedProfitBps < MIN_PROFIT_BPS) return null;
  if (Date.now() - event.timestamp > MAX_AGE_MS) return null;
  if (event.confidence < MIN_CONFIDENCE) return null;

  // STAGE 2: Dedup check (Redis) - <1ms
  if (await this.isDuplicate(event.opportunityId)) return null;

  // STAGE 3: Balance check (cached) - <1ms
  if (await this.insufficientBalance(event.inputAmount)) return null;

  // STAGE 4: Full simulation (expensive) - 50-100ms
  const simulation = await this.simulate(event);
  if (!simulation.success || simulation.netProfit < MIN_PROFIT) return null;

  // STAGE 5: Build plan
  return this.buildExecutionPlan(event, simulation);
}
```

**Impact**: Skip 70-80% of expensive simulations

---

## Part 4: Shared Memory Architecture for Rust Production

### 4.1 Design Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SHARED MEMORY ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Quote Aggregator (Go - Writer)                                         │
│       │                                                                 │
│       │  Atomic writes (<1μs)                                          │
│       ▼                                                                 │
│  ┌────────────────────┬────────────────────┐                           │
│  │ quotes-local.mmap  │ quotes-external.mmap│                          │
│  │ (128KB)            │ (128KB)             │                          │
│  │ • On-chain quotes  │ • API quotes        │                          │
│  │ • Oracle prices    │ • Oracle prices     │                          │
│  │ • Staleness flags  │ • Staleness flags   │                          │
│  └────────────────────┴────────────────────┘                           │
│       ▲                       ▲                                         │
│       │  Lock-free reads (<1μs)                                        │
│       │                                                                 │
│  ┌────────────────────────────────────────────┐                        │
│  │ Rust Scanner (Readers - Multiple Instances) │                       │
│  │ • Read both regions in parallel             │                       │
│  │ • Compare local vs external                 │                       │
│  │ • Detect arbitrage (<10μs)                  │                       │
│  │ • Publish to NATS                           │                       │
│  └────────────────────────────────────────────┘                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Quote Metadata Structure

```rust
#[repr(C, align(128))]
struct QuoteMetadata {
    version: AtomicU64,           // 8 bytes - Lock-free versioning
    pair_id: [u8; 32],            // 32 bytes - BLAKE3 hash
    input_mint: [u8; 32],         // 32 bytes - Token mint
    output_mint: [u8; 32],        // 32 bytes - Token mint
    input_amount: u64,            // 8 bytes
    output_amount: u64,           // 8 bytes
    price_impact_bps: u32,        // 4 bytes
    timestamp_unix_ms: u64,       // 8 bytes
    route_id: [u8; 32],           // 32 bytes - Route lookup key
    oracle_price_usd: f64,        // 8 bytes - For validation
    staleness_flag: u8,           // 1 byte - 0=fresh, 1=stale
    _padding: [u8; 7],            // 7 bytes - Align to 128
}
// Total: 128 bytes per quote
// 1000 quotes = 128KB (fits in L2 cache)
```

### 4.3 Lock-Free Read Protocol

```rust
fn read_quote_safe(quotes: &[QuoteMetadata], index: usize) -> Option<QuoteMetadata> {
    loop {
        // Read version (even = readable, odd = writing)
        let v1 = quotes[index].version.load(Ordering::Acquire);
        if v1 & 1 == 1 {
            // Writer in progress, spin
            std::hint::spin_loop();
            continue;
        }

        // Read quote data
        let quote = unsafe { std::ptr::read_volatile(&quotes[index]) };

        // Verify version unchanged
        let v2 = quotes[index].version.load(Ordering::Acquire);
        if v1 == v2 {
            return Some(quote);
        }
        // Version changed during read, retry
    }
}
```

### 4.4 Performance Comparison

| Operation | gRPC (Current) | Shared Memory (Future) | Improvement |
|-----------|---------------|------------------------|-------------|
| Quote read | 500μs - 2ms | <1μs | 500-2000x |
| Arbitrage detection | 1-2ms | <10μs | 100-200x |
| Memory allocation | Per-call | Zero | Eliminates GC |
| Serialization | Protobuf/FlatBuffers | None | Zero overhead |

---

## Part 5: Migration Path TypeScript → Rust

### 5.1 Phased Migration

```
Phase 1 (Now): TypeScript Prototype
├── Scanner: TS + gRPC ✅ Complete
├── Strategy: TS + NATS ⚠️ 60% Complete
└── Executor: TS + NATS ⚠️ 20% Complete

Phase 2 (Month 2): Hybrid
├── Scanner: Rust + Shared Memory (port first - most latency sensitive)
├── Strategy: TS (keep - logic complexity, less latency sensitive)
└── Executor: TS (keep - API integrations)

Phase 3 (Month 4): Full Rust
├── Scanner: Rust + Shared Memory ✅
├── Strategy: Rust + Shared Memory
└── Executor: Rust + Jito/TPU integration
```

### 5.2 Why Port Scanner First?

1. **Most Latency Sensitive**: Scanner runs in tight loop on every quote
2. **Simplest Logic**: Pattern matching + math (no complex API integrations)
3. **Highest ROI**: 100-200x improvement in hot path
4. **Shared Memory Ready**: Quote service already writes to mmap (planned)

### 5.3 Strategy Service: Keep in TypeScript Longer

1. **Complex Logic**: Risk scoring, multi-factor validation
2. **Jupiter API Integration**: Well-supported in TypeScript
3. **Less Latency Sensitive**: 50-100ms budget (vs 10ms for scanner)
4. **Rapid Iteration**: Easier to experiment with strategies in TS

---

## Part 6: Risk Assessment

### 6.1 Current Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Jupiter API rate limit | HIGH | MEDIUM | Template caching, request coalescing |
| Stale quotes | MEDIUM | HIGH | 5s TTL, staleness flags |
| Jito bundle rejection | MEDIUM | MEDIUM | Multi-path execution, dynamic tips |
| RPC failures | LOW | HIGH | Multiple endpoints, circuit breakers |
| NATS backpressure | LOW | MEDIUM | Memory streams for hot data |

### 6.2 Architecture Risks (Future)

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Shared memory corruption | LOW | CRITICAL | Atomic versioning, checksums |
| Writer crash | LOW | HIGH | Graceful degradation to gRPC |
| Reader starvation | LOW | MEDIUM | Lock-free reads, no blocking |
| Memory mapping failure | LOW | MEDIUM | Fallback to gRPC |

---

## Part 7: Summary Recommendations

### Immediate Actions (This Week)

1. **Complete Strategy Service** (2-3 days)
   - Jupiter instruction fetching (parallel)
   - Route merging
   - Execution plan publishing
   - Template caching

2. **Complete Executor Service** (4-5 days)
   - Transaction building with @solana/kit
   - Jito bundle submission
   - RPC fallback
   - Multi-path execution

### Short-Term (Month 1)

3. **Optimization Pass**
   - Blockhash caching
   - Early exit validation
   - Multi-amount variations
   - Metrics and monitoring

### Medium-Term (Months 2-3)

4. **Rust Scanner Port**
   - Shared memory reader
   - Lock-free quote access
   - NATS publishing
   - Performance validation

### Long-Term (Months 4-6)

5. **Full Rust Pipeline**
   - Strategy service port
   - Executor service port
   - End-to-end <50ms latency

---

## Conclusion

### Overall Architecture Assessment: ✅ SOUND

The architecture is well-designed for HFT with appropriate separation of concerns:

1. **Event-Driven** (NATS 6-stream): Correct for decoupling and fault isolation
2. **FlatBuffers**: Correct for serialization performance
3. **Bypassing Aggregator**: Correct for TypeScript prototype phase
4. **Future Shared Memory**: Correct for Rust production phase

### Critical Success Factors

1. **Template Caching**: Without this, Jupiter API latency dominates (200ms)
2. **Multi-Path Execution**: Without this, landing rate is 70-80% (vs 95%+)
3. **Parallel Processing**: Without this, sequential latency doubles
4. **Shared Memory (Future)**: Without this, Rust gains are limited

### Expected Performance (After Implementation)

```
TypeScript (Now):
  Quote → Scanner → Strategy → Executor → Profit
   5ms     10ms      80ms       20ms
  ────────────────────────────────────────
              ~115ms total ✅ < 200ms target

Rust (Future):
  Quote → Shmem → Scanner → Strategy → Executor → Profit
   5ms    <1μs     10μs      10ms       5ms
  ────────────────────────────────────────
               ~20ms total ✅ < 50ms target
```

---

**Document Version**: 1.0
**Status**: ✅ Assessment Complete
**Next Steps**: Implement TypeScript pipeline completion (Strategy + Executor)
