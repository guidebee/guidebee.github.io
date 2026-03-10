---
layout: single
title: "Integrated Weekly Plan - All Enhancements Consolidated"
permalink: /docs/16-weekly-plan-integrated/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Integrated Weekly Plan - All Enhancements Consolidated

**Version:** 3.0 - Triple-AI Enhanced (Grok + DeepSeek + Qwen)
**Date:** December 2, 2025
**Total Effort:** 462-840 hours over 9 months (11-21 hrs/week)
**Success Probability:** 78-80%

---

## Quick Reference: Effort by Phase

| Phase | Duration | Original | +Enhancements | **Total** | Key Additions |
|-------|----------|----------|---------------|-----------|---------------|
| **Phase 1** | 2 weeks | 20-40 hrs | +18-24 hrs | **38-64 hrs** | Kill switch, tax, nonce, wallet design |
| **Phase 2** | 3 weeks | 30-60 hrs | +4 hrs | **34-64 hrs** | Circuit breakers (already included) |
| **Phase 3** | 5-6 weeks | 50-120 hrs | +16-22 hrs | **66-142 hrs** | jito-go, paper trading, replay++ |
| **Phase 3.5** | 1 week | 0 hrs | +10-15 hrs | **10-15 hrs** | ⭐ **NEW: Market viability checkpoint** |
| **Phase 4** | 8 weeks | 80-160 hrs | +42-67 hrs | **122-227 hrs** | Dynamic tipping, network monitor, regime monitor |
| **Phase 5** | 16 weeks | 160-320 hrs | +8-10 hrs | **168-330 hrs** | Liquidity monitor, wallet expansion |
| **TOTAL** | 35 weeks | 340-680 hrs | +98-142 hrs | **438-822 hrs** | ~9 months |

---

## Phase 1: Foundation (Dec 9-22, 2025)

**Duration:** 2 weeks | **Effort:** 38-64 hours (was 20-40) | **Goal:** Infrastructure + safety mechanisms ready

### Week 1 (Dec 9-15): Repository & Infrastructure

**Time Budget:** 14-26 hours (was 10-20)

**Original Tasks:**
- [ ] **Repository Structure** (2 hours)
  - Set up monorepo (Rust workspace + pnpm for TypeScript)
  - Initialize Cargo workspace, pnpm workspace
  - Set up .gitignore, .env.example

- [ ] **Docker Infrastructure** (4 hours)
  - Docker Compose: Redis, PostgreSQL, NATS, Prometheus, Grafana
  - Initialize database schema (trades, opportunities, strategy_health)
  - Test all services

- [ ] **Token & Pool Configuration** (2 hours)
  - Define LST mints (jitoSOL, mSOL, bSOL)
  - Configure priority pairs
  - Document pool addresses per DEX

- [ ] **Development Environment** (2 hours)
  - Rust/TypeScript/Node.js setup
  - IDE configuration
  - Install dependencies

**NEW: DeepSeek Additions** (2-4 hours)
- [ ] **Alternative Niche Research** (2-4 hours) ⭐ **CRITICAL**
  - Research Meteora DLMM pools (30 min)
  - Research Pump.fun new launches (30 min)
  - Research cross-DEX spreads (30 min)
  - Research stablecoin triangular arb (30 min)
  - Document findings in `docs/alternative-niches.md`
  - **Why:** Have pivot strategy ready if LST arb dies
  - **GitHub Issue:** #23

**Deliverable:** ✅ Clean dev environment + alternative strategies documented

---

### Week 2 (Dec 16-22): Core Framework + Critical Safety

**Time Budget:** 24-38 hours (was 10-20) ⚠️ **HEAVY WEEK**

**Original Tasks:**
- [ ] **Generic Market Events (Rust)** (6 hours)
  - Define `MarketEvent` enum
  - NATS subject routing
  - Event serialization/deserialization
  - Basic event publisher

- [ ] **Strategy Framework Traits (Rust)** (6 hours)
  - Define `TradingStrategy` trait
  - Define `TradeOpportunity` struct
  - Define `StrategyConfig` struct
  - Strategy health monitoring types

- [ ] **Configuration System** (4 hours)
  - TOML configuration parser
  - Strategy configuration schema
  - Environment variable handling

**NEW: DeepSeek Additions** (10 hours) ⭐ **CRITICAL SAFETY**
- [ ] **Kill Switch Infrastructure** (6-8 hours) ⭐ **HIGHEST PRIORITY**
  - Create `KillSwitch` struct with atomic flags
  - Manual kill switch (API endpoint)
  - Slot drift detection (>300 slots → halt)
  - Stalled system detection (no trades 6 hrs → halt)
  - Circuit breaker integration
  - Network partition detection (multi-RPC comparison)
  - HTTP endpoints: `/api/killswitch/enable` and `/disable`
  - **Why:** Prevents catastrophic losses from network issues
  - **GitHub Issue:** #21

- [ ] **Tax & Legal Research** (4 hours) ⭐ **COMPLIANCE**
  - Consult crypto tax professional ($200-500 fee)
  - Set up transaction tracking system (CoinTracker/Koinly)
  - Research LLC formation (optional, document findings)
  - Document legal considerations (Jito ToS, front-running rules)
  - **Why:** Avoid tax penalties that can exceed profits
  - **GitHub Issue:** #22

**NEW: Grok Additions** (4-6 hours) ⭐ **PERFORMANCE**
- [ ] **Signer Nonce Accounts Setup** (4-6 hours) ⭐ **HIGH ROI**
  - Create on-chain nonce accounts for each wallet
  - Implement nonce-based transaction builder
  - Test nonce advancement and reuse protection
  - **Why:** Eliminates 8-15% bundle failures from blockhash expiration
  - **GitHub Issue:** #15

**NEW: Qwen Additions** (4 hours) **FUTURE-PROOFING**
- [ ] **Wallet Rotation Design** (4 hours)
  - Design `WalletPool` abstraction
  - Implement master seed derivation (BIP32-style)
  - Support N wallets (use N=1 initially)
  - Round-robin rotation strategy
  - Per-wallet usage statistics
  - Document expansion plan for Phase 5
  - **Why:** Prevents spam filtering, easy to expand later
  - **GitHub Issue:** #31

**Deliverable:** ✅ Core framework + kill switch + nonce accounts + legal compliance ✅

**⚠️ Note:** This is the heaviest week (24-38 hours). If needed, defer wallet rotation design to Week 3 to balance workload.

---

## Phase 2: Market Data & Quotes (Jan 6-25, 2026)

**Duration:** 3 weeks | **Effort:** 34-64 hours (was 30-60) | **Goal:** Market data streaming + quote service operational

### Week 3 (Jan 6-12): Shredstream Enhancement

**Time Budget:** 12-24 hours (unchanged)

**Tasks:**
- [ ] **Enhance Shredstream Client** (8 hours)
  - Add LST pool address monitoring
  - Implement generic event emission
  - Connect to NATS for publishing
  - Filter by priority pools (top 20 LST pools)

- [ ] **Pool State Cache (Rust)** (4 hours)
  - In-memory pool state storage
  - Lock-free reads for concurrent access
  - Update from Shredstream events

- [ ] **Heartbeat Sync for State Validation** (4 hours) ⭐ **CRITICAL** (Already included from v2.1)
  - Implement 60-second RPC verification
  - Compare local cache vs on-chain state
  - Auto-flush and resync on mismatch
  - Log drift events
  - **Why:** UDP packets WILL drop, prevents state corruption
  - **GitHub Issue:** Part of #3

- [ ] **Testing** (4 hours)
  - Verify events published to NATS
  - Verify pool state updates
  - Test state drift recovery
  - Load test with historical data

**Deliverable:** ✅ Market data streaming <100ms latency + auto state recovery

---

### Week 4-5 (Jan 13-25): Quote Service Integration

**Time Budget:** 22-40 hours (unchanged, circuit breaker already included from v2.2)

**Tasks:**
- [ ] **Enhance Go Quote Service** (12 hours)
  - Add LST pairs (jitoSOL, mSOL, bSOL)
  - Implement `/lst/opportunities` endpoint
  - Add round-trip profit calculation
  - Enable dark pool simulation (optional)

- [ ] **Rust HTTP Client for Quote Service** (6 hours)
  - HTTP client with connection pooling (reqwest + Keep-Alive)
  - Persistent connections (save 5-15ms per quote)
  - Quote caching (10-second TTL)
  - Error handling and retries
  - Timeout set to 50ms (fail-fast)

- [ ] **Quote Service Circuit Breaker** (2 hours) ⭐ **CRITICAL** (Already included from v2.2)
  - Detect 3 consecutive timeouts (>50ms)
  - Fallback to Jupiter API when Go service down
  - Auto-recovery: retry every 30 seconds
  - Health check endpoint (cache freshness <30s)
  - **Why:** Prevents hot path blocking on single point of failure
  - **GitHub Issue:** Part of #4

- [ ] **Quote Service Deployment** (2 hours)
  - Deploy Go service locally
  - Configure refresh intervals (20 seconds)
  - Set up health checks

**Deliverable:** ✅ Quote service 2-10ms with automatic failover

---

## Phase 3: Arbitrage Strategy & Execution (Feb 21 - Mar 31, 2026)

**Duration:** 5-6 weeks | **Effort:** 66-142 hours (was 50-120) | **Goal:** First profitable trade + validation

### Week 6-7 (Feb 21 - Mar 9): Arbitrage Detection

**Time Budget:** 20-40 hours (unchanged)

**Tasks:**
- [ ] **Arbitrage Strategy Implementation (Rust)** (16 hours)
  - Implement `TradingStrategy` trait for arbitrage
  - LST two-hop detection (SOL → LST → SOL)
  - LST three-hop detection (SOL → LST1 → LST2 → SOL)
  - USDC two-hop detection (conservative, low priority)
  - Profit calculation with fees
  - Priority ranking

- [ ] **Strategy Manager (Rust)** (8 hours)
  - Strategy registration system
  - Event routing to strategies
  - Opportunity aggregation
  - Strategy health monitoring

- [ ] **NATS Integration** (4 hours)
  - Subscribe to market events
  - Publish trade opportunities
  - Consumer management

**Deliverable:** ✅ Arbitrage opportunities detected and published

---

### Week 8-9 (Mar 10-23): Rust Executor

**Time Budget:** 24-44 hours (was 20-40)

**Original Tasks:**
- [ ] **Transaction Builder (Rust)** (12 hours)
  - Two-hop arbitrage TX builder
  - Three-hop arbitrage TX builder
  - Compute budget optimization
  - Jito tip instruction

- [ ] **Smart Trade Validator (Rust)** (6 hours) ⭐ (Already includes per-pool circuit breaker from v2.1)
  - Fast sanity checks (~5ms total):
    * Cached balance verification
    * Pool existence check
    * Profit threshold validation (Go quote)
    * Hardcoded compute units by hop count
  - NO expensive RPC simulation (saves 150ms)
  - **Per-Pool Circuit Breaker:**
    * Track Jito drop rate per pool
    * 3 strikes in 5 min = blacklist for 5 min
    * Force RPC resync before re-enabling
  - Global circuit breaker (3 consecutive failures)
  - **GitHub Issue:** Part of #6

**NEW: Grok Additions** (4 hours) ⭐ **HUGE TIME SAVER**
- [ ] **Switch to helix-labs/jito-go SDK** (4 hours) ⭐ **REPLACES CUSTOM CODE**
  - Install: `go get github.com/helius-labs/jito-go`
  - Initialize Jito client with tip rotation
  - Implement `SubmitBundle()` with dynamic tips
  - Add `WaitForBundleConfirmation()` with retry
  - Test on devnet
  - Integrate with Rust executor (HTTP bridge or rewrite in Go)
  - **Why:** Saves 40-60 hours, 75-85% success rate out-of-box
  - **GitHub Issue:** #16

**Original Task (now simplified):**
- [ ] **Jito Executor** (2 hours, was 6 hours - simplified by jito-go)
  - Integration with jito-go service
  - Confirmation monitoring
  - Success/failure metrics

**Deliverable:** ✅ Executor can build, validate (fast), and submit via jito-go

---

### Week 10-11 (Mar 24-31): Integration, Testing & First Trade

**Time Budget:** 27-38 hours (was 15-25)

**Original Tasks:**
- [ ] **End-to-End Integration** (8 hours)
  - Connect all components
  - Test with devnet first
  - Dry run on mainnet (no execution)
  - Validate strategy logic

- [ ] **Risk Management** (4 hours)
  - Per-pool circuit breakers (already built in Week 8-9)
  - Global circuit breakers
  - Position limits (max 10 SOL/trade)
  - Balance monitoring with alerts

**NEW: DeepSeek Additions** (12-14 hours) ⭐ **RISK MANAGEMENT**
- [ ] **Replay Testing Harness** (5 hours) ⭐ (Already included from v2.1)
  - Record 1 hour of Shredstream events (JSON Lines)
  - Build test harness (replay at 10x speed)
  - Verify bot WOULD have triggered at actual prices
  - Backtest without real money/RPC costs
  - **Why:** Your ONLY safe validation without real money
  - **GitHub Issue:** Part of #7

- [ ] **Dark Launch / Paper Trading** (8 hours) ⭐ **CRITICAL**
  - Build dual-mode executor (Paper/Real/Parallel)
  - Paper trading simulator (log what WOULD happen)
  - Run 1-week paper trading validation
  - **Success criteria (ALL must pass):**
    * Paper trading success rate >40%
    * Avg profit >$0.30 per trade
    * >50 opportunities in 1 week
    * Zero critical errors
    * Quote age <200ms for 95% of opps
  - **Why:** Validate before risking money
  - **GitHub Issue:** #24

- [ ] **Enhanced Replay Testing** (4-6 hours)
  - Competitor simulation (1-2 bots, random latency)
  - Network latency simulation (0ms, 100ms, 200ms, 500ms)
  - Failure injection (5%, 15%, 30% failure rates)
  - **Why:** Test adversarial scenarios
  - **GitHub Issue:** #25

**Original Task:**
- [ ] **First Live Trade** (4 hours)
  - ONLY after paper trading validation passes!
  - Start with 0.1 SOL trades
  - Monitor closely
  - Record results
  - Compare vs replay predictions

**Milestone:** 🎉 **FIRST PROFITABLE TRADE EXECUTED!**

**Expected Results:**
- 10-30 opportunities/day
- 50-60% success rate
- 5-15 profitable trades/day
- $5-20 profit/day

---

## **NEW Phase 3.5:** Market Viability Checkpoint (Mid-April 2026)

**Duration:** 1 week analysis | **Effort:** 10-15 hours | **Goal:** Go/No-Go decision

**When:** 2 weeks after first profitable trade (mid-April 2026)

### Week 12 (Apr 14-20): Market Assessment & Pivot Decision

**Time Budget:** 10-15 hours

**DeepSeek Critical Addition** ⭐ **MOST IMPORTANT CHECKPOINT**
- [ ] **Opportunity Trend Analysis** (4 hours)
  - Track daily opportunity count over 2 weeks
  - Calculate average opps/day
  - Identify declining trends
  - **Red flag:** >30% drop in 2 weeks
  - **Baseline:** 100-200 opps/day expected

- [ ] **Competitor Detection** (3 hours)
  - Monitor TX signatures on target pools
  - Identify recurring bot addresses
  - Measure: How often losing to same address?
  - **Red flag:** >50% of opps taken by 1-2 bots

- [ ] **Profit Margin Erosion** (2 hours)
  - Track profit per trade over time
  - Calculate Week 1 avg vs Week 4 avg
  - **Red flag:** >40% margin compression
  - **Expected:** $0.30-1.00 per trade

- [ ] **Pool Liquidity Changes** (2 hours)
  - Monitor daily volume on LST pools
  - Track pool depth
  - **Red flag:** Pool volume declining >30% month-over-month

- [ ] **Pivot Decision** (if needed) (4 hours)
  - Review alternative niches (from Week 1)
  - Select best alternative (Meteora, Pump.fun, etc.)
  - Plan 1-2 week implementation sprint
  - Execute pivot immediately if 3+ critical metrics

### Pivot Decision Matrix

| Metric | Healthy ✅ | Warning ⚠️ | Critical 🚨 (Pivot) |
|--------|-----------|-----------|---------------------|
| Daily opportunities | >100 | 50-100 | <50 |
| Profit per trade | >$0.50 | $0.20-0.50 | <$0.20 |
| Competitor count | 0-1 | 2-3 | 4+ |
| Monthly profit | >$3k | $1.5k-3k | <$1.5k |
| Pool volume trend | Growing/Stable | Flat | Declining >20% |

**PIVOT RULE:** If 3+ metrics in "Critical" → Execute pivot within 1 week

**Outcomes:**
- **Healthy (70% probability):** Continue to Phase 4
- **Marginal (20% probability):** Optimize + re-assess in 2 weeks
- **Dead (10% probability):** Pivot to alternative niche NOW

**Deliverable:** ✅ Go/No-Go decision with data backing

**GitHub Issue:** #26

---

## Phase 4: Optimization & Reliability (Apr 21 - May 26, 2026)

**Duration:** 8 weeks (5 weeks after Phase 3.5) | **Effort:** 122-227 hours (was 80-160) | **Goal:** 70% success rate, reliable 24/7 operation

### Week 13-14 (Apr 21 - May 4): Performance Optimization

**Time Budget:** 38-64 hours (was 24-44) ⚠️ **HEAVY 2 WEEKS**

**Original Tasks:**
- [ ] **Latency Profiling & Monitoring** (4 hours) ⭐ (Already included from v2.2)
  - Instrument hot path with checkpoints
  - Log component latencies to Prometheus
  - Track p95 and max latencies
  - Alert if exceeding budget (quote >10ms, validation >5ms)
  - **Why:** Data-driven optimization
  - **GitHub Issue:** Part of #8

- [ ] **Latency Optimization** (10 hours)
  - Profile with flamegraphs
  - Optimize hot paths
  - Cache blockhash (20s refresh, saves 50ms)
  - WebSocket confirmations (2x faster)
  - Batch RPC calls (10x faster)

- [ ] **Quote Caching** (6 hours)
  - Redis integration (sub-ms access)
  - Route template caching
  - Opportunity deduplication
  - Stale quote detection (>200ms)

- [ ] **Parallel Processing** (4 hours)
  - Concurrent opportunity evaluation
  - Parallel quote fetching
  - Multi-RPC load balancing

**NEW: Grok Additions** (6-8 hours) ⭐ **BIGGEST SUCCESS LEVER**
- [ ] **Dynamic Tip Bidding** (6-8 hours) ⭐ **CRITICAL**
  - Implement `get_recent_performance_samples()` Jito RPC
  - Calculate 95th percentile tip from recent bundles
  - Bid 5-10% above p95 (configurable)
  - Implement `calculate_max_profitable_tip()` (cap at 30% profit)
  - Absolute floor (0.0001 SOL) and ceiling (0.01 SOL)
  - Optional: Predict next leader schedule
  - Optional: Adjust tips per validator
  - Monitor tip efficiency in Grafana
  - **Why:** 10-15% success rate improvement during congestion
  - **GitHub Issue:** #17

**NEW: Qwen Additions** (10-12 hours)
- [ ] **Network Congestion Monitoring** (4-6 hours)
  - Subscribe to block updates (WebSocket)
  - Track block time (ms between blocks)
  - Calculate 10-min avg block time
  - Classify: Healthy/Degraded/Congested/Critical
  - Track TX failure rate (last 100 attempts)
  - Monitor median Jito tip
  - Adaptive recommendations:
    * Healthy: Trade normally
    * Degraded: 0.7x size, 1.2x tip
    * Congested: 0.3x size, 1.5x tip
    * Critical: Pause trading
  - **Why:** Prevents losses during Solana network chaos
  - **GitHub Issue:** #30

- [ ] **Dry Run Replay Before Deploy** (4-6 hours)
  - Build replay validation pipeline
  - Compare last 24h metrics (baseline vs test)
  - Block deployment if >10% regression
  - GitHub Actions integration (pre-deploy check)
  - **Why:** Prevent regressions in production
  - **GitHub Issue:** #33

**Target:** 750ms → 300ms execution (data-driven via profiling)

**Key Optimizations:**
- Blockhash caching: 50ms saved
- HTTP Keep-Alive: 5-15ms saved per quote
- WebSocket vs polling: 500ms → 200ms confirmation
- Parallel quotes: 2x faster

**Deliverable:** ✅ Sub-300ms execution + dynamic tipping + network awareness

---

### Week 15-16 (May 5-18): Multi-Pair Support

**Time Budget:** 20-40 hours (unchanged)

**Tasks:**
- [ ] **Add mSOL and bSOL Pairs** (8 hours)
  - Configure in Go quote service
  - Update Rust arbitrage detector
  - Test separately per pair

- [ ] **Pair-Specific Configuration** (4 hours)
  - Different profit thresholds per pair
  - Different simulation requirements
  - Pair rotation (prevent over-trading)

- [ ] **LST Triangular Logic** (8 hours)
  - Implement LST pair scoring
  - Configure jitoSOL ↔ mSOL (highest score)
  - Add mSOL ↔ bSOL route

**Expected Results:**
- 100-200 opportunities/day (3 LST pairs)
- 60-65% success rate
- 60-130 profitable trades/day
- $30-80 profit/day ($900-2,400/month)

**Deliverable:** ✅ Multiple LST pairs operational

---

### Week 17-20 (May 19 - June 8): Reliability & Monitoring

**Time Budget:** 62-107 hours (was 44-84) ⚠️ **HEAVY 4 WEEKS**

**Original Tasks:**
- [ ] **Error Handling** (12 hours)
  - Comprehensive error types
  - Retry logic for RPC failures
  - Transaction timeout handling
  - Dead letter queue for failed trades

- [ ] **Analytics Service (TypeScript)** (16 hours)
  - PnL calculator
  - Trade analysis by strategy
  - Success rate tracking
  - Performance metrics

- [ ] **Monitoring Dashboards** (12 hours)
  - Grafana dashboard setup
  - Prometheus metrics integration
  - Key metrics: opportunities, success rate, PnL
  - Alerts: circuit breaker, low balance, high error rate

- [ ] **Production Alerting** (4 hours) ⭐ (Already included from v2.2)
  - Slack webhook for critical alerts
  - Email daily summary (PnL, success, errors)
  - Alert thresholds:
    * Error rate >30% last hour
    * No trades in 4 hours
    * Pool blacklist >5 pools/hour
    * Circuit breaker triggered
    * Wallet balance <0.5 SOL
  - **Why:** Part-time dev needs proactive notifications
  - **GitHub Issue:** Part of #10

- [ ] **Production Hardening** (12 hours)
  - Graceful shutdown handling
  - State persistence
  - Log rotation
  - Health check endpoints

**NEW: Grok Additions** (6 hours)
- [ ] **Opportunity Heatmap + Route Blacklisting** (6 hours) ⭐
  - Track profit per specific route
  - Auto-blacklist routes with negative profit (last 20 attempts)
  - Blacklist for 30-60 minutes (configurable)
  - Grafana dashboard for route performance
  - Alert on high blacklist rate (>5 routes/hour)
  - **Why:** Prevents bleeding when faster bot takes over route
  - **GitHub Issue:** #18

**NEW: DeepSeek Additions** (4-6 hours)
- [ ] **Enhanced Bundle Quality Metrics** (4-6 hours)
  - Track: submitted, landed, dropped counts
  - Calculate success rate (landed/submitted)
  - Track average tip paid per bundle
  - Calculate tip-to-profit ratio (should be <0.3)
  - Monitor competitor tips (p50, p95)
  - Competitive position analysis
  - Reputation risk detector (>50% drops/hour, >100 bundles/hour)
  - Grafana dashboard additions
  - **Why:** Optimizes tip spending, prevents Jito reputation damage
  - **GitHub Issue:** #28

**NEW: Qwen Additions** (14-19 hours) ⭐ **OPERATIONAL INTELLIGENCE**
- [ ] **Market Regime Monitor** (6-8 hours) ⭐ **HIGH PRIORITY**
  - Track 1-hour price volatility (std dev %)
  - Monitor 24h pool volume
  - Calculate opportunity frequency (per hour)
  - Classify regime: Favorable/Marginal/Unfavorable
  - Trading decisions:
    * Favorable: 1.0x size
    * Marginal: 0.5x size
    * Unfavorable: Pause trading
  - Alert if unfavorable >4 hours (pivot signal?)
  - **Why:** Only trade when market conditions favorable
  - **GitHub Issue:** #29

- [ ] **Copy Bot Detector** (2-3 hours)
  - Track 30-day average success rate per route (baseline)
  - Compare last 24h vs baseline
  - Detect >30% drop with stable volume/volatility
  - Automatic response:
    * 30-50% drop: Increase profit buffer 1.5x
    * >50% drop: Pause route for 1 hour
  - Alert operator
  - **Why:** Early competition detection
  - **GitHub Issue:** #32

- [ ] **Post-Mortem System Setup** (1 hour)
  - Create post-mortem template
  - Location: `docs/post-mortems/`
  - Document every failure (even small ones)
  - Track time-to-recovery (MTTR)

**Expected Results:**
- System runs 24/7 reliably
- Full visibility into performance
- Quick issue detection via Slack/email
- 70%+ success rate

**Deliverable:** ✅ Production-grade monitoring + market awareness

---

## Phase 5: Multi-Strategy Platform (Jun 9 - Sep 20, 2026)

**Duration:** 14 weeks (was 16) | **Effort:** 168-330 hours (was 160-320) | **Goal:** Complete platform with scaling

**Note:** DCA and Grid Trading are **deprioritized**. They run on "cold path" with no latency requirements. Focus on perfecting LST arbitrage first.

### Week 21-24 (Jun 9 - Jul 6): DCA Strategy

**Time Budget:** 44-84 hours (unchanged, strategy coordination already included from v2.2)

**Tasks:**
- [ ] **DCA Strategy Implementation (TypeScript)** (16 hours)
  - Interval-based buying
  - Price threshold checking
  - RSI-based filtering (optional)
  - Integration with strategy framework

- [ ] **Strategy Coordination Layer** (4 hours) ⭐ (Already included from v2.2)
  - In-memory pool reservation (500ms lock)
  - Priority queue: Arbitrage > Grid > DCA
  - Prevent duplicate bundle submissions
  - Coordination via shared state
  - **Why:** Avoids internal strategy competition
  - **GitHub Issue:** Part of #11

- [ ] **DCA Configuration** (4 hours)
  - Configure for SOL (hourly)
  - Configure for jitoSOL (daily)
  - Set price limits
  - Integrate with pool reservation

- [ ] **Testing & Deployment** (8 hours)
  - Paper trading for 1 week
  - Deploy with small amounts
  - Monitor performance
  - Test cross-strategy contention

**Expected Results:**
- 24-48 DCA opportunities/day
- Passive LST accumulation
- $20-50 profit/day (DCA + arbitrage)
- Zero bundle conflicts

---

### Week 25-28 (Jul 7 - Aug 3): Grid Trading Strategy

**Time Budget:** 40-80 hours (unchanged)

**Tasks:**
- [ ] **Grid Trading Implementation (TypeScript)** (20 hours)
  - Grid level initialization
  - Price crossing detection
  - Buy/sell order generation
  - Rebalancing logic

- [ ] **Grid Configuration** (4 hours)
  - Configure for JUP token
  - Set price range and grid levels
  - Set amount per level

- [ ] **Testing & Optimization** (8 hours)
  - Backtest on historical data
  - Optimize grid parameters
  - Deploy and monitor

**Expected Results:**
- 20-50 grid opportunities/day
- Profit from range-bound volatility
- $30-80 profit/day (all strategies)

---

### Week 29-32 (Aug 4-31): Long/Short Strategy

**Time Budget:** 40-80 hours (unchanged)

**Tasks:**
- [ ] **Long/Short Implementation (Rust)** (24 hours)
  - Moving average calculations
  - Trend detection (golden cross, death cross)
  - Position management
  - Integration with lending protocols (Kamino/Drift)

- [ ] **Risk Management** (8 hours)
  - Stop-loss logic
  - Take-profit logic
  - Position sizing
  - Leverage limits

- [ ] **Testing** (8 hours)
  - Paper trading for 2 weeks
  - Start with small positions
  - Monitor closely

**Expected Results:**
- 5-15 long/short signals/day
- Higher risk, higher reward
- $50-150 profit/day (all strategies)

---

### Week 33-36 (Sep 1-20): Production Scaling 🎉

**Time Budget:** 48-90 hours (was 40-80)

**Original Tasks:**
- [ ] **Performance Tuning** (16 hours)
  - Profile entire system
  - Optimize critical paths
  - Target: <200ms execution
  - Memory optimization

- [ ] **Flash Loan Integration** (16 hours)
  - Kamino flash loan support
  - Zero-capital arbitrage
  - Larger position sizes (10-100 SOL)

- [ ] **Deployment to Production Server** (8 hours)
  - Bare metal server setup (Hetzner/OVH)
  - Move from Docker Compose to production
  - Set up monitoring and alerts
  - Optimize for latency

**NEW: DeepSeek Additions** (6-8 hours)
- [ ] **Liquidity Monitoring & Auto-Scaling** (6-8 hours)
  - Monitor pool depth for all pools
  - Optimal trade size calculator:
    * Never >2% of pool reserves
    * Never >5% of daily volume
    * Absolute ceiling: 10 SOL
  - Add more LST pairs (stSOL, laineSOL, scnSOL, daoSOL)
  - Flash loan evaluation (if constrained)
  - **Why:** Prevents slippage losses, enables scaling
  - **GitHub Issue:** #27

**NEW: Qwen Additions** (2 hours)
- [ ] **Wallet Rotation Expansion** (2 hours)
  - Expand from 1 wallet → 5 wallets
  - Fund each with 5-10 SOL
  - Configure executor to rotate
  - Monitor per-wallet success rates
  - **Why:** Prevents spam filtering at scale
  - **GitHub Issue:** #31 (continuation)

**Expected Results (Conservative - Single/Multi-Wallet):**
- 200-400 opportunities/day (all strategies)
- 60-70% success rate (realistic with competition)
- 120-280 profitable trades/day
- $150-400 profit/day
- **$4,500-12,000 profit/month (baseline)** ✅

**Expected Results (Optimistic - With Scaling):**
- Requires: Flash loans (10-100 SOL), 5-10 wallets, triangular arb
- 800-1,500 opportunities/day
- 70-75% success rate
- 560-1,125 profitable trades/day
- $500-800 profit/day
- **$15,000-25,000 profit/month** (requires significant additional work)

**Final Milestone:** 🎉 **PRODUCTION SYSTEM COMPLETE - $4,500-12,000/MONTH BASELINE**

---

## Optional: After Profitable (When Ready)

### Upgrade Path (When >$8k/mo Gross for 2 Months)
- [ ] **Paid ShredStream** ($250-500/mo)
  - 99.9% packet delivery
  - Multi-region failover
  - Only when cost-justified
  - **GitHub Issue:** #19

### Productization (When >$4k/mo Net for 2 Months)
- [ ] **"Solana Long-Tail Arb Kit"**
  - Package system (minus keys)
  - Documentation + video tutorials
  - Pricing: $1,500-2,500 one-time OR $200-400/mo SaaS
  - Target: 10-20 customers in year 1
  - Revenue: +$20-60k/year additional
  - **GitHub Issue:** #20

---

## Summary: Total Effort by Week

| Week | Phase | Time Budget | Key Tasks | Issues |
|------|-------|-------------|-----------|--------|
| 1 | 1 | 14-26 hrs | Infrastructure + Alternative niches | #1, #23 |
| 2 | 1 | 24-38 hrs ⚠️ | Framework + Kill switch + Tax + Nonce + Wallet | #2, #21, #22, #15, #31 |
| 3 | 2 | 12-24 hrs | Shredstream + Heartbeat sync | #3 |
| 4-5 | 2 | 22-40 hrs | Quote service + Circuit breaker | #4 |
| 6-7 | 3 | 20-40 hrs | Arbitrage detection | #5 |
| 8-9 | 3 | 24-44 hrs | Executor + jito-go SDK | #6, #16 |
| 10-11 | 3 | 27-38 hrs | Integration + Paper trading + First trade 🎉 | #7, #24, #25 |
| 12 | **3.5** | 10-15 hrs | **Market viability checkpoint** ⭐ | #26 |
| 13-14 | 4 | 38-64 hrs ⚠️ | Optimization + Dynamic tip + Network monitor | #8, #17, #30, #33 |
| 15-16 | 4 | 20-40 hrs | Multi-pair support | #9 |
| 17-20 | 4 | 62-107 hrs ⚠️ | Reliability + Regime monitor + Copy detect | #10, #18, #28, #29, #32 |
| 21-24 | 5 | 44-84 hrs | DCA strategy | #11 |
| 25-28 | 5 | 40-80 hrs | Grid trading | #12 |
| 29-32 | 5 | 40-80 hrs | Long/short | #13 |
| 33-36 | 5 | 48-90 hrs | Scaling + Liquidity + Wallets 🎉 | #14, #27, #31 |

**Total:** 462-840 hours over 35 weeks (~9 months)
**Average:** 13-24 hours/week

**Heavy weeks** (>30 hours):
- Week 2: 24-38 hrs (safety infrastructure)
- Week 13-14: 38-64 hrs (optimization sprint)
- Week 17-20: 62-107 hrs (monitoring sprint)

**Light weeks** (<15 hours):
- Week 1: 14-26 hrs
- Week 12: 10-15 hrs (analysis, not coding)

---

## Key Success Metrics (Celebrate These!)

### Phase 1 (Dec 2025)
- ✅ Infrastructure running
- ✅ Kill switch tested
- ✅ Alternative niches documented

### Phase 2 (Jan 2026)
- ✅ Market data streaming <100ms
- ✅ Quote service 2-10ms
- ✅ Circuit breakers functional

### Phase 3 (Mar 2026)
- 🎉 **First profitable trade!** ($5-20/day)
- ✅ 50-60% success rate
- ✅ Paper trading validation passed

### Phase 3.5 (Apr 2026)
- 🎯 **Market viability checkpoint**
- ✅ Continue or pivot decision

### Phase 4 (May 2026)
- 🎉 **$900-2,400/month profit**
- ✅ 60-65% success rate
- ✅ 24/7 reliable operation

### Phase 5 (Sep 2026)
- 🎉 **$4,500-12,000/month baseline**
- ✅ Multi-strategy platform
- ✅ Production system complete

---

## Final Notes

**This integrated plan includes:**
- ✅ All original weekly tasks
- ✅ All Grok performance enhancements (6 additions)
- ✅ All DeepSeek safety mechanisms (8 additions)
- ✅ All Qwen operational intelligence (5 additions)
- ✅ NEW Phase 3.5 market viability checkpoint

**Success Probability:** 78-80%

**Use this document as your single source of truth for week-by-week implementation.**

**Related Documents:**
- `docs/00-MASTER-SUMMARY.md` - Executive overview
- `docs/12-CONSOLIDATED-PRODUCTION-PLAN.md` - Original plan (still valid for reference)
- `docs/13-GROK-PRODUCTION-ENHANCEMENTS.md` - Performance details
- `docs/14-DEEPSEEK-RISK-MANAGEMENT.md` - Safety details
- `docs/15-QWEN-OPERATIONAL-RESILIENCE.md` - Operations details

**GitHub Issues:** All 33 issues tracked at https://github.com/guidebee/solana-trading-system/issues

---

**Ready to start? Begin with Week 1 (Dec 9-15)!** 🚀

**See you at the first trade celebration! 🎉**
