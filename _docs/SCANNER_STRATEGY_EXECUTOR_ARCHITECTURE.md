---
layout: single
title: "Scanner → Strategy → Executor Architecture"
permalink: /docs/scanner-strategy-executor-architecture/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Scanner → Strategy → Executor Architecture

## Overview

This document describes the complete data flow from opportunity detection (Scanner) through planning (Strategy) to execution (Executor) in the Solana HFT trading system.

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SOLANA BLOCKCHAIN                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   Raydium    │  │    Orca      │  │   Meteora    │  │   PumpSwap   │   │
│  │   AMM/CLMM   │  │  Whirlpools  │  │    DLMM      │  │     AMM      │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
          │                    │                  │                │
          └────────────────────┼──────────────────┼────────────────┘
                               │                  │
                               ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          GO QUOTE SERVICE                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Pool Discovery │ Multi-Protocol Router │ Quote Cache (30s refresh) │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    │ gRPC Quote Stream                      │
│                                    ▼                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
          ┌──────────────────────────┼──────────────────────────┐
          │                          │                          │
          ▼                          ▼                          ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ SCANNER SERVICE │     │STRATEGY SERVICE │     │EXECUTOR SERVICE │
│   (TypeScript)  │     │   (TypeScript)  │     │   (TypeScript)  │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│ • gRPC Consumer │     │ • NATS Consumer │     │ • NATS Consumer │
│ • Arb Detection │────▶│ • Deduplication │────▶│ • Wallet Mgmt   │
│ • Price Analysis│     │ • Risk Scoring  │     │ • Tx Building   │
│ • Event Publish │     │ • Plan Building │     │ • Multi-Executor│
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        │ TwoHopArbitrageEvent  │ ExecutionPlanEvent   │
        │ (FlatBuffers)         │ (FlatBuffers)        │
        ▼                       ▼                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NATS JETSTREAM                                      │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │ OPPORTUNITIES  │  │    PLANNED     │  │    RESULTS     │                │
│  │    Stream      │  │    Stream      │  │    Stream      │                │
│  │ opportunity.*  │  │ execution.plan │  │ execution.result│               │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
└─────────────────────────────────────────────────────────────────────────────┘
                                                        │
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      TRANSACTION SUBMISSION                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  Jito Bundle │  │  RPC Direct  │  │   Solayer    │  │  TPU Client  │   │
│  │  (MEV Prot)  │  │  (Fallback)  │  │  (Alt Route) │  │ (Low Latency)│   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Service Responsibilities

### 1. Scanner Service ✅ (Complete)

**Location**: `ts/apps/scanner-service/`

**Responsibilities**:
- Subscribe to Go quote service via gRPC streaming
- Compare quotes from multiple DEXs/pools
- Detect two-hop arbitrage opportunities (A→B→A)
- Calculate estimated profit and price impact
- Publish `TwoHopArbitrageEvent` to NATS

**Input**: Real-time quotes from Go quote service
**Output**: `opportunity.arb.{tokenPair}` events to NATS

**Key Files**:
- `src/index.ts` - Main entry, gRPC connection
- `src/handlers/oracle-arbitrage-handler.ts` - Arbitrage detection logic
- `src/flatbuffers-helper.ts` - Event serialization

### 2. Strategy Service ⚠️ (60% Complete)

**Location**: `ts/apps/strategy-service/`

**Responsibilities**:
- Consume opportunities from NATS (worker pool)
- Deduplicate opportunities (LRU + Redis)
- Validate opportunities (age, profit, confidence)
- Fetch swap instructions from Jupiter API
- Merge two hops into single transaction
- Simulate transaction on-chain
- Calculate risk score
- Publish `ExecutionPlanEvent` to NATS

**Input**: `opportunity.arb.>` from NATS
**Output**: `execution.plan.{opportunityId}` to NATS

**What's Missing**:
- [ ] Jupiter swap instructions fetching
- [ ] Route merging logic
- [ ] Transaction simulation
- [ ] Execution plan publishing
- [ ] Template caching

### 3. Executor Service ⚠️ (20% Complete)

**Location**: `ts/apps/executor-service/`

**Responsibilities**:
- Consume execution plans from NATS
- Select appropriate execution method
- Build and sign transactions
- Submit via Jito/RPC/Solayer/TPU
- Monitor confirmation status
- Analyze actual profit
- Publish `ExecutionResultEvent` to NATS

**Input**: `execution.plan.>` from NATS
**Output**: `execution.result.{opportunityId}` to NATS

**What's Missing**:
- [ ] Transaction building with @solana/kit
- [ ] Jito bundle submission
- [ ] RPC fallback with retry
- [ ] Solayer integration
- [ ] Confirmation polling
- [ ] Profit analysis

## Event Schemas (FlatBuffers)

### TwoHopArbitrageEvent

```flatbuffers
table TwoHopArbitrageEvent {
  opportunity_id: string;
  trace_id: string;
  source: string;
  timestamp: int64;

  input_mint: string;
  output_mint: string;
  input_amount: int64;
  expected_output: int64;
  estimated_profit_bps: int32;

  hop1: SwapHop;
  hop2: SwapHop;

  confidence: float;
}

table SwapHop {
  input_mint: string;
  output_mint: string;
  input_amount: int64;
  expected_output: int64;
  dex: string;
  pool_address: string;
  price_impact_bps: int32;
  pool_liquidity: int64;
}
```

### ExecutionPlanEvent

```flatbuffers
table ExecutionPlanEvent {
  opportunity_id: string;
  trace_id: string;
  source: string;
  timestamp: int64;

  input_mint: string;
  output_mint: string;
  input_amount: int64;
  expected_output: int64;
  estimated_profit_bps: int32;

  hops: [SwapHop];

  serialized_instructions: [ubyte];
  address_lookup_tables: [string];
  compute_unit_limit: int32;
  compute_unit_price: int32;

  risk_score: float;
  confidence: float;
  max_slippage_bps: int32;

  expires_at: int64;
}
```

### ExecutionResultEvent

```flatbuffers
table ExecutionResultEvent {
  opportunity_id: string;
  trace_id: string;
  timestamp: int64;

  status: ExecutionStatus;
  signature: string;

  expected_profit_usd: float;
  actual_profit_usd: float;

  latency_ms: int32;
  executor: string;

  error: string;
}

enum ExecutionStatus : byte {
  SUCCESS = 0,
  FAILED = 1,
  TIMEOUT = 2,
  REJECTED = 3,
}
```

## Data Flow Example

### Complete Arbitrage Trade

```
1. SCANNER detects opportunity:
   ┌─────────────────────────────────────────────┐
   │ Quote: SOL→USDC (Raydium) = 150.05         │
   │ Quote: USDC→SOL (Orca)    = 0.00668 (≈149.7)│
   │ Profit: 0.35 SOL (0.23%)                   │
   │ Confidence: 0.85                           │
   └─────────────────────────────────────────────┘
   Publishes → opportunity.arb.SOL-USDC

2. STRATEGY processes opportunity:
   ┌─────────────────────────────────────────────┐
   │ ✓ Dedup check: New opportunity             │
   │ ✓ Validation: Profit 23bps > 50bps min     │
   │ ✓ Age: 200ms < 5000ms max                  │
   │ → Fetch Jupiter instructions (parallel)    │
   │ → Merge routes into single tx              │
   │ → Simulate: Success, 450k CU used          │
   │ → Risk score: 0.78 → EXECUTE               │
   └─────────────────────────────────────────────┘
   Publishes → execution.plan.{id}

3. EXECUTOR executes plan:
   ┌─────────────────────────────────────────────┐
   │ → Select wallet (1.5 SOL balance)          │
   │ → Build transaction (add tip, CU budget)   │
   │ → Sign transaction                         │
   │ → Submit via Jito bundle                   │
   │ → Monitor confirmation (2.3s)              │
   │ → Analyze profit: 0.32 SOL actual          │
   └─────────────────────────────────────────────┘
   Publishes → execution.result.{id}

4. MONITORING:
   ┌─────────────────────────────────────────────┐
   │ Grafana Dashboard:                         │
   │ • Opportunity rate: 15/min                 │
   │ • Plan rate: 8/min                         │
   │ • Execution rate: 6/min                    │
   │ • Success rate: 75%                        │
   │ • Avg profit: 0.28 SOL                     │
   │ • Avg latency: 2.1s                        │
   └─────────────────────────────────────────────┘
```

## Execution Methods

### Method Comparison

| Method | Latency | Cost | MEV Protection | Landing Rate | Use Case |
|--------|---------|------|----------------|--------------|----------|
| **Jito Bundle** | 2-3s | Tip 5k-5M lamports | ✅ Yes | 95%+ | Default for HFT |
| **RPC Direct** | 1-2s | Network only | ❌ No | 70-85% | Fallback |
| **Solayer** | 1-2s | Network only | ✅ Routing | 80-90% | Alternative |
| **TPU Client** | <500ms | Network only | ❌ No | 60-80% | Speed-critical |

### Executor Selection Logic

```typescript
function selectExecutor(opportunity: TradeOpportunity): Executor {
  // 1. High-profit trades → Jito (MEV protection)
  if (opportunity.expectedProfitUsd >= 0.10) {
    return jitoExecutor;
  }

  // 2. Time-critical → TPU (fastest)
  if (opportunity.urgency === 'critical') {
    return tpuExecutor;
  }

  // 3. Fallback chain
  for (const executor of [jitoExecutor, solayerExecutor, rpcExecutor]) {
    if (executor.isHealthy()) return executor;
  }

  throw new Error('No healthy executor');
}
```

## Performance Targets

| Metric | Target | Current | Gap |
|--------|--------|---------|-----|
| End-to-end latency | < 500ms | ~1.7s | -70% |
| Scanner → Strategy | < 50ms | ~100ms | -50% |
| Strategy planning | < 100ms | ~200ms | -50% |
| Executor submission | < 100ms | N/A | Implement |
| Confirmation | < 3s | N/A | Implement |
| Jito landing rate | 95%+ | N/A | Implement |

## Implementation Priority

### Week 1: Core Flow (Days 1-3)

**Strategy Service**:
1. Jupiter swap instructions fetching
2. Route merging (two hops → one tx)
3. Transaction simulation
4. Execution plan publishing
5. Risk scoring

**Executor Service**:
1. @solana/kit integration
2. Transaction building
3. Wallet management
4. Jito bundle submission

### Week 1: Fallbacks (Days 4-5)

**Executor Service**:
1. RPC direct submission
2. Retry logic with backoff
3. Confirmation polling
4. Profit analysis

### Week 2: Optimization

1. Template caching (Strategy)
2. Solayer integration (Executor)
3. Multi-wallet rotation
4. Dynamic tip calculation
5. End-to-end testing

## Configuration

### NATS Streams

```bash
# Create streams
nats stream add OPPORTUNITIES --subjects "opportunity.>" --storage file --replicas 1
nats stream add PLANNED --subjects "execution.plan.>" --storage file --replicas 1
nats stream add RESULTS --subjects "execution.result.>" --storage file --replicas 1
nats stream add SYSTEM --subjects "system.>" --storage file --replicas 1
```

### Environment Variables

```bash
# Shared
NATS_SERVERS=nats://localhost:4222
REDIS_HOST=localhost
RPC_ENDPOINTS=https://api.mainnet-beta.solana.com

# Scanner Service
SCANNER_GRPC_ENDPOINT=http://localhost:8080
SCANNER_PAIRS=SOL/USDC,SOL/USDT

# Strategy Service
STRATEGY_WORKER_COUNT=10
STRATEGY_MIN_PROFIT_BPS=50
STRATEGY_SIMULATION_TIMEOUT_MS=2000

# Executor Service
EXECUTOR_JITO_ENABLED=true
EXECUTOR_JITO_ENDPOINT=https://mainnet.block-engine.jito.wtf
EXECUTOR_MAX_CONCURRENT=5
```

## Monitoring

### Grafana Dashboards

1. **Scanner Dashboard**
   - Quotes/second by DEX
   - Opportunities detected/minute
   - Quote latency histogram

2. **Strategy Dashboard**
   - Plans created/minute
   - Dedup hit rate
   - Risk score distribution
   - Simulation success rate

3. **Executor Dashboard**
   - Executions/minute
   - Success rate by executor
   - Profit distribution
   - Confirmation latency

### Prometheus Metrics

```yaml
# Scanner
scanner_quotes_received_total{dex="raydium"}
scanner_opportunities_detected_total{pair="SOL-USDC"}

# Strategy
strategy_plans_published_total
strategy_dedup_hits_total
strategy_simulations_total{result="success|failed"}
strategy_risk_score_histogram

# Executor
executor_executions_total{executor="jito|rpc|solayer",status="success|failed"}
executor_latency_histogram{executor="jito"}
executor_profit_usd_histogram
```

## Error Handling

### Scanner → Strategy

```typescript
// Strategy worker error handling
try {
  await handler.handleOpportunity(event);
  msg.ack();
} catch (error) {
  if (isRetryable(error)) {
    msg.nak(1000); // Retry after 1s
  } else {
    msg.term(); // Don't retry
    logger.error({ error }, 'Non-retryable error');
  }
}
```

### Strategy → Executor

```typescript
// Executor error handling
try {
  const result = await executor.execute(plan);
  if (result.status === 'success') {
    publishResult(result);
  } else {
    // Try fallback executor
    const fallbackResult = await fallbackExecutor.execute(plan);
    publishResult(fallbackResult);
  }
} catch (error) {
  publishResult({
    status: 'failed',
    error: error.message,
  });
}
```

## Testing Strategy

### Unit Tests
- Route merging logic
- Risk score calculation
- Transaction building
- Tip calculation

### Integration Tests
- Scanner → Strategy flow
- Strategy → Executor flow
- Multi-executor fallback

### End-to-End Tests
- Full opportunity lifecycle
- Error recovery scenarios
- Performance under load

## Next Steps

1. **Immediate**: Complete Strategy service plan publishing
2. **Short-term**: Implement Executor Jito integration
3. **Medium-term**: Add RPC fallback and confirmation
4. **Ongoing**: Performance optimization and monitoring
