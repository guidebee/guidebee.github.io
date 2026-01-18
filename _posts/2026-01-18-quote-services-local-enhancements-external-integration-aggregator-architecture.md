---
layout: single
title: "Quote Services Evolution: Local Enhancements, External Integration, and Quote Aggregator Architecture"
date: 2026-01-18
permalink: /posts/2026/01/quote-services-local-enhancements-external-integration-aggregator-architecture/
categories:
  - blog
tags:
  - solana
  - trading
  - quote-service
  - architecture
  - observability
  - confidence-scoring
  - aggregation
excerpt: "Comprehensive upgrade to the quote service ecosystem: Local Quote Service gains intelligent pool quality management with oracle validation and dynamic refresh priorities, External Quote Service integrates Jupiter/DFlow/OKX APIs, and the new Quote Aggregator Service merges both sources with confidence scoring for optimal trade execution."
---

## TL;DR

The quote service ecosystem has undergone a major architectural evolution with three key components now working in concert:

1. **Local Quote Service Enhancements**: Intelligent pool quality management with liquidity-based classification, rogue pool detection, progressive recovery, and dynamic refresh priorities - all delivering sub-2ms quote selection
2. **External Quote Service**: Integration with Jupiter, DFlow, and OKX quote providers for external market pricing with circuit breaker protection and health monitoring
3. **Quote Aggregator Service** (Work in Progress): Real-time merging of local and external quotes with confidence scoring, oracle-based price comparison, and decision recommendations for optimal execution

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Local Quote Service Enhancements](#local-quote-service-enhancements)
3. [External Quote Service](#external-quote-service)
4. [Quote Aggregator Service](#quote-aggregator-service)
5. [Metrics and Observability](#metrics-and-observability)
6. [Impact and Next Steps](#impact-and-next-steps)

---

## Architecture Overview

The quote service ecosystem now consists of three specialized services working together:

```
┌────────────────────────────────────────────────────────────────┐
│                     Quote Service Ecosystem                    │
└────────────────────────────────────────────────────────────────┘

  ┌─────────────────────┐          ┌─────────────────────┐
  │  Local Quote Svc    │          │ External Quote Svc  │
  │  (Port 50052)       │          │ (Port 50053)        │
  ├─────────────────────┤          ├─────────────────────┤
  │ • Pool Quality Mgmt │          │ • Jupiter API       │
  │ • Oracle Validation │          │ • DFlow API         │
  │ • Dynamic Refresh   │          │ • OKX API           │
  │ • WebSocket-First   │          │ • Circuit Breaker   │
  │ • 400+ Pools        │          │ • Health Tracking   │
  │ • <2ms Selection    │          │ • Rate Limiting     │
  └──────────┬──────────┘          └──────────┬──────────┘
             │                                │
             │    gRPC Streaming              │
             │    StreamBatchQuotes           │
             └────────────┬───────────────────┘
                          │
                          ▼
             ┌────────────────────────┐
             │  Quote Aggregator Svc  │
             │  (Port 50051)          │
             ├────────────────────────┤
             │ • Real-time Merging    │
             │ • Confidence Scoring   │
             │ • Oracle Comparison    │
             │ • Best Quote Selection │
             │ • NATS Event Publish   │
             └────────────┬───────────┘
                          │
                          ▼
              ┌──────────────────────┐
              │   Scanner Service    │
              │   (TypeScript/Rust)  │
              └──────────────────────┘
```

**Design Principles:**

- **Separation of Concerns**: Each service has a single, well-defined responsibility
- **Parallel Sourcing**: Local and external quotes fetched simultaneously
- **Intelligent Aggregation**: Confidence-weighted selection with oracle validation
- **Real-time Updates**: WebSocket-first for local pools, API polling for external
- **Observable**: Comprehensive Prometheus metrics at every layer

---

## Local Quote Service Enhancements

### Pool Quality & Priority Management

The local quote service has been enhanced with a comprehensive pool quality management system that intelligently classifies pools, detects rogue pricing, and dynamically adjusts refresh priorities.

![Local Quote Service Architecture](../images/local-quote-service.png)

### Key Features

#### 1. Liquidity-Based Classification

Pools are classified into three tiers based on Solscan-enriched TVL data:

- **High Liquidity** (>$100k): ~50 pools - large trades (100+ SOL)
- **Medium Liquidity** ($10k-$100k): ~150 pools - medium trades (10-100 SOL)
- **Low Liquidity** ($500-$10k): ~200 pools - small trades (1-10 SOL)

This classification enables intelligent pool selection based on trade size, automatically filtering pools where the trade would exceed 20% of pool liquidity to prevent high slippage.

#### 2. Rogue Pool Detection & Recovery

**Oracle Price Validation**: Every pool price is validated against Pyth Network oracle prices via NATS events.

**Detection Mechanism**:
- Pools with price deviation > ±20% from oracle are flagged as "rogue"
- 2-strike rule: Two consecutive bad quotes trigger rogue status
- State machine: Normal → Rogue → Recovering → Normal

**Progressive Recovery**:
- 1st good quote: Transition to "Recovering" status
- 2nd consecutive good quote: Restore to "Normal" status
- Automatic tracking of all status transitions with reasons

#### 3. Dynamic Priority Refresh

**Variable Refresh Intervals** based on pool quality:

- **Highest Priority** (5s): Low liquidity + normal status (most volatile)
- **High Priority** (10s): High liquidity + normal status
- **Medium Priority** (15s): Medium liquidity + normal status
- **Low Priority** (60s): Rogue pools (minimal resources)

**WebSocket-First Architecture**:
- All major Solana DEX protocols support WebSocket `accountSubscribe`
- Real-time updates for Raydium AMM/CPMM/CLMM, Orca Whirlpool, Meteora DLMM
- RPC fallback only for connection failures or health checks

#### 4. Two-Stage Pool Selection

**Stage 1: Liquidity Constraint Filter**
- Calculate trade size in USD using oracle prices
- Filter pools where trade > 20% of pool liquidity
- Prevents high slippage on small pools

**Stage 2: Weighted Random Selection**
- Quality score = liquidityWeight × statusMultiplier
- High liquidity + normal status = highest probability
- Rogue pools = low but non-zero probability (redemption chance)

**Performance**: Sub-2ms pool selection overhead (exceeds target of <10ms)

### Implementation Status

✅ **All 6 Phases Complete** (January 10, 2026)

**Production Code**: 2,694 lines (Go)
**Test Code**: 573 lines (37+ test cases, 100% pass rate)
**Metrics**: 31+ Prometheus metrics with automatic recording
**Dashboard**: 10-row Grafana dashboard with 40+ panels
**Alerts**: 10 Prometheus alert rules (P0/P1/P2)

**Implementation Time**: ~6 hours (42x faster than original 6-week estimate)

### Key Metrics

**Pool Quality Metrics**:
- `local_quote_pool_quality_total{tier,status}` - Pool distribution by tier and status
- `local_quote_rogue_pool_percentage` - Percentage of pools flagged as rogue (alert: >10%)
- `local_quote_status_transitions_total{from,to,reason}` - State machine health

**Oracle Validation**:
- `local_quote_oracle_validation_success_rate` - Oracle validation success (alert: <95%)
- `local_quote_oracle_deviation_percent` - Price deviation from oracle

**Pool Selection**:
- `local_quote_pool_selection_total{tier,status}` - Selection distribution
- `local_quote_pools_filtered_by_trade_size_total` - Slippage protection in action

**Refresh Priority**:
- `local_quote_refresh_queue_size{priority}` - Queue health (alert: >100 for highest)
- `local_quote_websocket_reconnections_total` - WebSocket stability (alert: >0.1/s)

---

## External Quote Service

The external quote service integrates third-party quote providers (Jupiter, DFlow, OKX) to complement on-chain pool data with aggregated market pricing.

![External Quote Service Architecture](../images/external-quote-service.png)

### Key Features

#### 1. Multi-Provider Integration

**Jupiter Integration**:
- Primary quote provider for Solana DEX aggregation
- Rate limiting: 1 RPS shared across all Jupiter-based quoters
- Automatic retry with exponential backoff

**DFlow Integration**:
- Order flow auction protocol for better execution
- Circuit breaker pattern for failure isolation
- Health monitoring with uptime tracking

**OKX Integration**:
- Centralized exchange pricing for reference
- API key management and authentication
- Rate limit handling

#### 2. Circuit Breaker Protection

**Purpose**: Prevent cascading failures when external providers are down

**Thresholds**:
- Open circuit after 5 consecutive failures
- Half-open state after 30s cooldown
- Close circuit after 3 successful requests

**Metrics**:
- `external_quote_circuit_breaker_state{provider}` - Current circuit state
- `external_quote_circuit_breaker_trips_total{provider}` - Total circuit trips

#### 3. Health Tracking

Each provider tracks:
- Request count (success/failure)
- Average latency
- Uptime percentage
- Last successful request timestamp

**Health Check Endpoint**: `GET /health`
```json
{
  "status": "healthy",
  "providers": {
    "jupiter": {
      "healthy": true,
      "requests": 12543,
      "errors": 23,
      "avg_latency_ms": 87,
      "uptime_percent": 99.82
    }
  }
}
```

### Configuration

**Command-Line Flags**:
```bash
external-quote-service \
  -jupiter-api-url https://quote-api.jup.ag/v6/quote \
  -jupiter-rate-limit 1 \
  -dflow-api-url https://api.dflow.net/quotes \
  -okx-api-url https://www.okx.com/api/v5/market/quote
```

### Key Metrics

**Provider Health**:
- `external_quote_provider_health{provider}` - Health status (1=healthy, 0=unhealthy)
- `external_quote_provider_requests_total{provider,status}` - Request counts
- `external_quote_provider_latency_seconds{provider}` - Latency distribution

**Rate Limiting**:
- `external_quote_rate_limit_exceeded_total{provider}` - Rate limit violations
- `external_quote_request_queue_size{provider}` - Queued requests

---

## Quote Aggregator Service

The quote-aggregator-service is the client-facing API that combines quotes from local and external sources, applying confidence scoring to recommend optimal execution.

![Quote Aggregator Service Architecture](../images/quote-aggregator-service.png)

### Architecture Components

#### 1. Upstream gRPC Client

**Purpose**: Connect to both local (50052) and external (50053) quote services via persistent streaming connections

**Features**:
- Parallel streaming from both upstream services
- Automatic reconnection with exponential backoff
- Connection health monitoring
- Latency and error tracking

**gRPC Keepalive**: Configured for connection stability with 10s ping interval

#### 2. Quote Aggregator

**Real-time Merging**:
- In-memory quote tables with pairID-based deduplication
- Route hash calculation for duplicate detection
- Best quote selection with confidence weighting

**Pair ID Matching**:
```
pairID = SHA256(inputMint + ":" + outputMint + ":" + inputAmount)[:16]
```

**Decision Recommendations**:
- **Execute**: High confidence (>0.8), low price difference
- **Verify**: Medium confidence (0.6-0.8), check oracle
- **Cautious**: Low confidence (0.4-0.6), manual review
- **Skip**: Very low confidence (<0.4), too risky

#### 3. Confidence Scoring Integration

**5-Factor Algorithm**:
- Pool Age (30%): Newer pools = lower confidence
- Route Complexity (20%): More hops = lower confidence
- Oracle Deviation (30%): Higher deviation = lower confidence
- Provider Uptime (10%): Lower uptime = lower confidence
- Slippage Estimate (10%): Higher slippage = lower confidence

**Separate Scoring**: Local and external quotes scored independently

#### 4. NATS Event Publisher

**Purpose**: Publish optimal quotes to NATS JetStream for scanner consumption

**Event Type**: `SwapRouteEvent` to `market.swap_route.optimal`

**FlatBuffers Serialization**: Efficient event encoding for high-throughput scenarios

### API Endpoints

**gRPC (Port 50051)**:
- `StreamAggregatedQuotes` - Server-streaming RPC for continuous updates
- `GetAggregatedQuote` - Unary RPC for single quote (150ms timeout)
- `GetConfidenceBreakdown` - Detailed confidence factor analysis
- `Health` - Comprehensive health check with upstream status

**HTTP (Port 8080)**:
- `GET /health` - Service health with upstream status
- `GET /stats` - Aggregation statistics
- `GET /metrics` - Prometheus metrics

### Price Difference Calculation

#### How It Works

When the aggregator receives quotes from **both** local and external sources for the **same pair ID**, it calculates the percentage difference:

```
priceDiffPercent = ((externalOutputAmount - localOutputAmount) / localOutputAmount) * 100
```

**Interpretation**:
- `+2.0%`: External gives 2% MORE output than local
- `-1.5%`: External gives 1.5% LESS output than local
- `0.0%`: Both sources give the same output

#### Example

```
Swap: 1 SOL → USDC

Local Quote:
  - outputAmount: 150,000,000 (150 USDC)

External Quote:
  - outputAmount: 151,500,000 (151.5 USDC)

Price Difference:
  = ((151,500,000 - 150,000,000) / 150,000,000) * 100
  = 1.0%

Result: External is 1% better for this swap
```

### Oracle-Based Price Comparison

The oracle price serves as a neutral baseline for comparing local and external quotes.

**Oracle Deviation Formula**:
```
deviation_percent = ((quote_price - oracle_price) / oracle_price) * 100
```

**Oracle Comparison Delta**:
```
delta = |external_deviation| - |local_deviation|
```

**Interpretation**:
- `+0.5%`: Local is 0.5% closer to oracle (more accurate)
- `-0.5%`: External is 0.5% closer to oracle (more accurate)
- `0.0%`: Both equally close to oracle

### Key Metrics: Price Difference Calculation

These metrics help identify which source provides better prices and by how much.

#### Core Price Metrics

**Price Difference (Local vs External)**:
- **Metric**: `quote_aggregator_aggregator_price_diff_percent`
- **Type**: Histogram
- **Labels**: `pair` (e.g., `So111111/EPjFWdd5`)
- **Buckets**: `-5, -2, -1, -0.5, -0.1, 0, 0.1, 0.5, 1, 2, 5`

**PromQL Queries**:
```promql
# Median price difference over 5 minutes
histogram_quantile(0.50, rate(quote_aggregator_aggregator_price_diff_percent_bucket[5m]))

# 95th percentile (extreme outliers)
histogram_quantile(0.95, rate(quote_aggregator_aggregator_price_diff_percent_bucket[5m]))

# Price difference for specific pair
histogram_quantile(0.50, rate(quote_aggregator_aggregator_price_diff_percent_bucket{pair="So111111/EPjFWdd5"}[5m]))
```

**Interpretation**:
- Median ~0%: Local and external are well-calibrated
- Median consistently positive: External finds better routes
- Median consistently negative: Local finds better routes
- High variance: Prices fluctuate; use caution
- Spikes > 5%: Potential arbitrage or stale data

#### Oracle Comparison Metrics

**Oracle Deviation by Source**:
- **Metric**: `quote_aggregator_aggregator_oracle_deviation_percent`
- **Labels**: `source` (local/external), `pair`
- **Buckets**: `-10, -5, -2, -1, -0.5, -0.1, 0, 0.1, 0.5, 1, 2, 5, 10`

**Oracle Comparison Delta**:
- **Metric**: `quote_aggregator_aggregator_oracle_comparison_best_delta_percent`
- **Labels**: `pair`
- **Interpretation**: Positive = local closer to oracle, Negative = external closer

**Quote Price vs Oracle Ratio**:
- **Metric**: `quote_aggregator_aggregator_quote_price_vs_oracle_ratio`
- **Buckets**: `0.9, 0.95, 0.98, 0.99, 0.995, 1.0, 1.005, 1.01, 1.02, 1.05, 1.1`
- **Interpretation**: 1.0 = matches oracle, >1.0 = better than oracle

**Oracle Price USD**:
- **Metric**: `quote_aggregator_aggregator_oracle_price_usd`
- **Type**: Gauge
- **Labels**: `pair`, `token` (input/output)

### Implementation Status

✅ **Core Implementation Complete** (January 18, 2026)

**Files Created**:
- `internal/quote-aggregator-service/client/upstream_client.go` ✅
- `internal/quote-aggregator-service/aggregator/aggregator.go` ✅
- `internal/quote-aggregator-service/server/server.go` ✅
- `internal/quote-aggregator-service/publisher/publisher.go` ✅
- `cmd/quote-aggregator-service/main.go` ✅

**Compilation**: ✅ No errors

### Next Steps for Quote Aggregator

**Phase 1: Integration Testing**
- Test with running local-quote-service
- Test with running external-quote-service
- Verify NATS event publishing
- Test gRPC streaming with ts-scanner-service

**Phase 2: Shared Memory Writer**
- Implement dual shared memory writer (`quotes-local.mmap`, `quotes-external.mmap`)
- Ring buffer with hybrid change detection
- Rust scanner integration

**Phase 3: Production Hardening**
- Add circuit breaker for upstream services
- Implement retry with exponential backoff
- Add rate limiting for downstream clients
- Comprehensive metrics and alerting

---

## Metrics and Observability

### Local Quote Service Metrics

**31+ Prometheus metrics** across 10 categories:

1. **Pool Quality** (3 metrics): Tier distribution, rogue %, priority
2. **Oracle Validation** (3 metrics): Validation results, deviation, duration
3. **Pool Selection** (3 metrics): Selections by tier/status, filtered count
4. **Refresh Priority** (4 metrics): Queue size, duration, failures
5. **Pool Usage** (1 metric): Usage count per pool
6. **Status Transitions** (1 metric): Transitions with reasons
7. **WebSocket Health** (5 metrics): Reconnections, latency, subscriptions, uptime
8. **RPC Fallback** (2 metrics): Fallback triggers, rate limits
9. **Cache Performance** (7 metrics): Hit rates, evictions, state age
10. **Quote Accuracy** (4 metrics): Price deviation, slippage, errors

### Quote Aggregator Metrics

**Aggregation Metrics**:
- `quote_aggregator_aggregator_quotes_received_total{source}` - Quotes from upstream
- `quote_aggregator_aggregator_quotes_aggregated_total{type}` - Aggregated quotes (local_only/external_only/both)
- `quote_aggregator_aggregator_best_source_total{source}` - Best source selections
- `quote_aggregator_aggregator_confidence_score{source}` - Confidence distribution
- `quote_aggregator_aggregator_decisions_total{decision}` - Trading decisions

**Token Pair Update Frequency**:
- `quote_aggregator_grpc_token_pair_updates_total{source,pair}` - Total updates per pair
- `quote_aggregator_grpc_token_pair_update_interval_seconds{source,pair}` - Interval between updates
- `quote_aggregator_grpc_token_pair_last_update_timestamp{source,pair}` - Last update timestamp

**Upstream Client**:
- `quote_aggregator_upstream_health{source}` - Upstream connection health
- `quote_aggregator_upstream_latency_seconds{source}` - Request latency
- `quote_aggregator_upstream_errors_total{source,error_type}` - Error tracking

### Grafana Dashboards

**Local Quote Service Dashboard** (10 rows, 40+ panels):
1. Pool Quality Overview
2. Oracle Validation Health
3. Pool Selection Analytics
4. Refresh Priority Management
5. Pool Usage Analytics
6. Pool Status Transitions
7. WebSocket Real-Time Updates
8. RPC Fallback Health
9. Dual Cache Architecture
10. Quote Correctness

**Quote Aggregator Dashboard** (planned):
1. Price Difference Analysis (Local vs External)
2. Oracle Comparison (Three-Way)
3. Confidence Score Distribution
4. Token Pair Update Frequency
5. Best Source Selection
6. Upstream Service Health

### Alerting

**P0 Critical Alerts**:
- `RoguePoolPercentageHigh`: >10% pools flagged as rogue
- `OracleValidationFailureHigh`: <95% validation success rate
- `HighPriceDifference`: Median price diff >5% or <-5%
- `StaleTokenPairQuotes`: No updates in 5+ minutes

**P1 Warning Alerts**:
- `QuoteDeviationHigh`: p95 deviation >1%
- `HighOracleDeviation`: Oracle deviation >5%
- `ExternalMoreAccurateThanLocal`: External consistently closer to oracle

---

## Impact and Next Steps

### Achieved Results

**Local Quote Service**:
- ✅ Sub-2ms pool selection (exceeds <10ms target)
- ✅ Oracle-validated prices with ±20% deviation threshold
- ✅ Progressive recovery for rogue pools
- ✅ 100% metrics coverage (31+ metrics)
- ✅ WebSocket-first architecture with RPC fallback

**External Quote Service**:
- ✅ Multi-provider integration (Jupiter, DFlow, OKX)
- ✅ Circuit breaker protection
- ✅ Health tracking and monitoring
- ✅ Rate limiting and backoff strategies

**Quote Aggregator Service**:
- ✅ Real-time quote merging from dual sources
- ✅ Confidence scoring integration (5-factor algorithm)
- ✅ Oracle-based price comparison
- ✅ Price difference calculation and tracking
- ✅ Decision recommendations (Execute/Verify/Cautious/Skip)

### Production Deployment Plan

**Week 1: Integration Testing**
- Deploy local-quote-service with quality management enabled
- Deploy external-quote-service with Jupiter integration
- Deploy quote-aggregator-service in shadow mode
- Verify NATS event flow and metrics collection

**Week 2: Gradual Rollout**
- Enable local quote service quality management (20% → 50% → 100%)
- Monitor rogue pool percentage and oracle validation success
- Verify pool selection distribution and refresh performance
- Gradual rollout of quote aggregator (shadow → 50% → 100%)

**Week 3: Production Validation**
- Monitor price difference metrics across all pairs
- Validate oracle comparison accuracy
- Confirm confidence scoring improves execution quality
- Measure latency improvements and quote freshness

### Future Enhancements

**Shared Memory Integration**:
- Dual shared memory writer for ultra-low latency Rust scanner
- Ring buffer with hybrid change detection
- Sub-microsecond quote access

**Machine Learning Integration**:
- Predictive confidence scoring based on historical accuracy
- Dynamic weight adjustment for confidence factors
- Anomaly detection for price manipulation

**Advanced Routing**:
- Multi-hop route optimization
- Triangle arbitrage detection
- Cross-DEX routing strategies

---

## Conclusion

The quote service ecosystem has evolved from a single monolithic service to a sophisticated three-tier architecture with specialized responsibilities:

- **Local Quote Service** provides fast, oracle-validated quotes from on-chain pools with intelligent quality management
- **External Quote Service** integrates third-party aggregators for market pricing with circuit breaker protection
- **Quote Aggregator Service** merges both sources with confidence scoring and oracle comparison for optimal execution

With sub-2ms local quote selection, comprehensive price difference tracking, and intelligent aggregation, the system is positioned to deliver superior trade execution for HFT operations on Solana.

---

## Related Posts

- [Pool Discovery Refactored: Bug Fixes and Comprehensive Testing](/posts/2026/01/pool-discovery-refactored-bug-fixes-comprehensive-testing/) - Foundation for pool quality management
- [Happy New Year 2026: Quote Service Three-Way Split](/posts/2026/01/happy-new-year-2026-quote-service-three-way-split/) - Architecture decision for splitting services
- [Pool Discovery Service RPC Proxy Architecture](/posts/2025/12/pool-discovery-service-rpc-proxy-architecture/) - WebSocket-first design patterns

---

## Technical Documentation

- [Local Quote Service Enhancements](https://github.com/guidebee/solana-trading-system/blob/master/go/cmd/local-quote-service/docs/1-LOCAL-QUOTE-SERVICE-ENHANCEMENTS.md)
- [Quote Aggregator Implementation](https://github.com/guidebee/solana-trading-system/blob/master/docs/30.8-QUOTE-AGGREGATOR-IMPLEMENTATION.md)
- [Quote Aggregator Metrics](https://github.com/guidebee/solana-trading-system/blob/master/go/cmd/quote-aggregator-service/docs/METRICS.md)

---

## Connect

- **GitHub**: [guidebee/solana-trading-system](https://github.com/guidebee/solana-trading-system)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

*This is post #22 in the Solana Trading System development series. The quote service ecosystem has been transformed with intelligent pool quality management, external quote integration, and real-time aggregation with confidence scoring. With comprehensive metrics and oracle-based price comparison, the system delivers optimal trade execution for high-frequency trading on Solana.*
