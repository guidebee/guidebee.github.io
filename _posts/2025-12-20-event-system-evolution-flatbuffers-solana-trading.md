---
layout: single
title: "Event System Evolution: FlatBuffers Migration and Event-Type Architecture for High-Frequency Solana Trading"
date: 2025-12-20
permalink: /posts/2025/12/event-system-evolution-flatbuffers-solana-trading/
categories:
  - blog
tags:
  - solana
  - flatbuffers
  - nats
  - event-driven
  - high-frequency-trading
  - performance
  - microservices
  - zero-copy
excerpt: "How we achieved 35% latency reduction and 87% CPU savings by migrating from JSON to FlatBuffers and evolving from priority-based to event-type NATS streams in our Solana HFT trading system."
---

## TL;DR

Our Solana HFT trading system underwent a major event architecture evolution, migrating from JSON-based priority streams to FlatBuffers-based event-type streams. The results: **35% latency reduction** (147ms → 95ms), **87% CPU reduction** for serialization, **44% smaller messages**, and a more intuitive architecture. This post details the journey, architectural decisions, and comprehensive event type design.

## The Problem: JSON Isn't Fast Enough

When building a high-frequency trading system targeting sub-200ms execution times, every millisecond matters. Our initial JSON-based event system had several bottlenecks:

### Performance Issues
- **JSON Serialization**: 12-15ms average, unpredictable spikes to 30ms
- **Memory Allocations**: JSON parsing creates temporary objects, triggering GC
- **Priority-Based Routing**: Events of different types mixed in priority streams
- **Schema Drift**: No compile-time validation across TypeScript, Go, and Rust

### The Numbers That Mattered
```
Scanner → Planner: 95ms (JSON) → 16ms (FlatBuffers)  [6x faster!]
Full Pipeline:    147ms (JSON) → 95ms (FlatBuffers)  [35% faster!]
CPU Usage:        100% (JSON) → 13% (FlatBuffers)    [87% reduction!]
Message Size:     450 bytes → 250 bytes              [44% smaller!]
```

We needed **zero-copy deserialization**, **type safety**, and **predictable latency**.

## The Solution: FlatBuffers + Event-Type Streams

### Part 1: FlatBuffers Migration

[FlatBuffers](https://google.github.io/flatbuffers/) provides zero-copy deserialization - you can read serialized data directly without parsing. Perfect for HFT.

#### Key Benefits

1. **Zero-Copy Access**: Read fields directly from buffer
2. **Type Safety**: Schema-driven code generation for TypeScript, Go, Rust
3. **Compact Format**: Smaller messages = faster network transfer
4. **Backward Compatibility**: Schema evolution without breaking changes

#### Schema-Driven Architecture

All event types defined in `.fbs` schema files:

```fbs
// opportunities.fbs
namespace HFT.Events;

table TwoHopArbitrageEvent {
  type: string;                  // Set by application to "TwoHopArbitrage"
  trace_id: string;
  timestamp: int64;
  slot: uint64;

  opportunity_id: string;
  token_start: string;           // Base58 Solana address
  token_intermediate: string;

  path: [SwapHop];               // [A→B, B→A]
  amount_in: string;
  expected_amount_out: string;
  estimated_profit_usd: double;
  estimated_profit_bps: int32;
  net_profit_usd: double;

  confidence: double;            // 0.0 - 1.0
  expires_at: int64;
}
```

One schema → Code generation for all languages → Guaranteed compatibility.

### Part 2: Event-Type Stream Architecture

We evolved from **priority-based streams** to **event-type streams** for better logical separation.

#### Old Architecture (Priority-Based)
```
EVENTS_CRITICAL → System lifecycle, kill switches
EVENTS_HIGH     → Arbitrage opportunities + large trades
EVENTS_NORMAL   → Price updates + liquidity updates
EVENTS_LOW      → Slot updates + routine monitoring
```

**Problem**: Mixed concerns. Arbitrage opportunities and price updates don't belong in the same stream just because they're both "high priority."

#### New Architecture (Event-Type Based)
```
MARKET_DATA    → Price, liquidity, pools, slots (10k events/sec)
OPPORTUNITIES  → Arbitrage opportunities, swap routes (500/sec)
EXECUTION      → Validated execution plans (50/sec)
EXECUTED       → Execution results, audit trail (50/sec)
METRICS        → Performance metrics (1-5k/sec)
SYSTEM         → Lifecycle, health, kill switches (1-10/sec)
```

**Benefit**: Each stream has a clear purpose. Consumers subscribe only to relevant events.

## Event System Design: Complete Taxonomy

### 1. MARKET_DATA Stream

Real-time market data from Quote Service and chain monitoring.

#### PriceUpdateEvent
```fbs
table PriceUpdateEvent {
  type: string;               // Set by application to "PriceUpdate"
  trace_id: string;
  token: string;              // SOL, USDC, jitoSOL (Base58)
  symbol: string;
  price_usd: double;
  price_sol: double;
  source: string;             // "raydium_amm", "orca", "pyth"
  timestamp: int64;
  slot: uint64;
}
```
**Use Case**: Real-time price tracking across DEXes and oracles (Pyth, Raydium, Orca).

**Subject**: `market.price.{symbol}` (e.g., `market.price.SOL`)

#### LiquidityUpdateEvent
```fbs
table LiquidityUpdateEvent {
  type: string;               // Set by application to "LiquidityUpdate"
  trace_id: string;
  pool_id: string;            // AMM pool address
  dex: string;
  token_a: string;
  token_b: string;
  reserve_a: string;          // u64 as string
  reserve_b: string;
  liquidity_usd: double;
  timestamp: int64;
  slot: uint64;
}
```
**Use Case**: Detect liquidity changes that affect price impact and slippage.

**Subject**: `market.liquidity.{poolId}`

#### SwapRouteEvent
```fbs
table SwapRouteEvent {
  type: string;               // Set by application to "SwapRoute"
  trace_id: string;
  timestamp: int64;
  slot: uint64;
  route_id: string;
  token_in: string;
  token_out: string;
  amount_in: string;
  expected_amount_out: string;
  hops: [SwapHop];            // Multi-hop route
  total_fee_amount: string;
  price_impact_pct: string;
  estimated_profit_bps: int32 = -1; // vs direct swap (-1 = not set)
}
```
**Use Case**: Quote Service publishes optimal swap routes every 30 seconds. Scanners detect arbitrage opportunities.

**Subject**: `market.swap_route.{tokenIn}.{tokenOut}`

**Frequency**: ~10,000 events/sec during peak

**Retention**: 1 hour (memory storage for speed)

### 2. OPPORTUNITIES Stream

Detected arbitrage opportunities from scanners.

#### TwoHopArbitrageEvent
```fbs
table TwoHopArbitrageEvent {
  type: string;               // Set by application to "TwoHopArbitrage"
  trace_id: string;
  timestamp: int64;
  slot: uint64;

  // Opportunity identification
  opportunity_id: string;

  // Tokens (A → B → A)
  token_start: string;        // e.g., SOL
  token_intermediate: string; // e.g., jitoSOL

  // Execution path
  path: [SwapHop];            // [SOL→jitoSOL, jitoSOL→SOL]

  // Profit analysis
  amount_in: string;
  expected_amount_out: string;
  estimated_profit_usd: double;
  estimated_profit_bps: int32;   // 50 bps = 0.5%
  net_profit_usd: double;         // After fees

  // Risk metrics
  total_fees: string;
  price_impact_bps: int32;
  required_capital_usd: double;   // For flash loan sizing
  confidence: double;             // Scanner confidence 0.0-1.0

  // Liquidity context
  hop1_liquidity_usd: double;
  hop2_liquidity_usd: double;

  // Timing
  expires_at: int64;
  detected_at_slot: uint64;

  // Scanner metadata
  scanner_name: string;
  scanner_metadata: string;       // JSON for extensibility
}
```

**Use Case**: LST arbitrage (SOL ↔ jitoSOL), stablecoin arbitrage (USDC ↔ USDT).

**Subject**: `opportunity.arbitrage.two_hop.{token1}.{token2}`

**Example**:
```typescript
{
  opportunity_id: "550e8400-e29b-41d4-a716-446655440000",
  token_start: "So11111111111111111111111111111111111111112", // SOL
  token_intermediate: "J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn", // jitoSOL
  path: [
    { label: "Raydium", in_amount: "1000000000", out_amount: "1005000000" },
    { label: "Orca", in_amount: "1005000000", out_amount: "1008000000" }
  ],
  estimated_profit_bps: 80,      // 0.8% profit
  net_profit_usd: 1.20,
  confidence: 0.92
}
```

#### TriangularArbitrageEvent
```fbs
table TriangularArbitrageEvent {
  type: string;               // Set by application to "TriangularArbitrage"
  trace_id: string;
  timestamp: int64;
  slot: uint64;
  opportunity_id: string;

  token_a: string;            // SOL
  token_b: string;            // USDC
  token_c: string;            // USDT

  path: [SwapHop];            // [A→B, B→C, C→A]
  estimated_profit_usd: double;
  estimated_profit_bps: int32;
  confidence: double;
  expires_at: int64;
}
```

**Use Case**: SOL → USDC → USDT → SOL triangular arbitrage.

**Subject**: `opportunity.arbitrage.triangular.{id}`

**Frequency**: ~500 events/sec (only profitable opportunities)

**Retention**: 24 hours (file storage for analysis)

### 3. EXECUTION Stream

Validated execution plans from planner services.

#### ExecutionPlanEvent
```fbs
table ExecutionPlanEvent {
  type: string;               // Set by application to "ExecutionPlan"
  trace_id: string;
  timestamp: int64;
  slot: uint64;

  // Reference to opportunity
  opportunity_id: string;
  source_event: string;       // "TwoHopArbitrage" | "TriangularArbitrage"

  // Execution strategy
  execution_method: ExecutionMethod;  // JitoBundle | TpuDirect | RpcFallback
  path: [SwapHop];

  // Validation results (optional, null if not yet simulated)
  simulation_result: SimulationResult;

  // Execution parameters
  jito_tip: string;           // Jito bundle tip (empty if not using Jito)
  compute_budget: uint64;
  priority_fee: string;

  // Risk assessment
  confidence: double;         // Updated after simulation
  risk_score: double;         // 0.0-1.0 (lower = safer)

  // Strategy context
  strategy_name: string;
  strategy_metadata: string;

  // Timing
  valid_until: int64;
}
```

**Use Case**: Planner validates opportunity via RPC simulation, plans execution method (Jito bundle vs TPU direct).

**Subject**: `execution.plan.{opportunityId}`

#### ExecutionRejectedEvent
```fbs
table ExecutionRejectedEvent {
  type: string;               // Set by application to "ExecutionRejected"
  trace_id: string;
  timestamp: int64;
  opportunity_id: string;
  reason: string;             // "insufficient_profit", "simulation_failed", "expired"
  details: string;            // Additional details (JSON string)
}
```

**Use Case**: Track rejected opportunities for strategy optimization.

**Subject**: `execution.rejected.{opportunityId}`

**Frequency**: ~50 events/sec

**Retention**: 1 hour (ephemeral)

### 4. EXECUTED Stream

Execution results from executor services.

#### ExecutionResultEvent
```fbs
table ExecutionResultEvent {
  type: string;               // Set by application to "ExecutionResult"
  trace_id: string;
  timestamp: int64;
  opportunity_id: string;

  signature: string;          // On-chain transaction signature
  actual_profit: double;
  actual_profit_usd: double;
  gas_used: uint64;
  block_time: int64;

  execution_method: ExecutionMethod;
  status: ExecutionStatus;    // Success | PartialFill | Failed
}
```

**Use Case**: Audit trail, profitability analysis, strategy backtesting.

**Subject**: `execution.result.{opportunityId}`

#### ExecutionFailedEvent
```fbs
table ExecutionFailedEvent {
  type: string;               // Set by application to "ExecutionFailed"
  trace_id: string;
  timestamp: int64;
  opportunity_id: string;
  error: string;              // Error message
  error_code: string;         // "SLIPPAGE_EXCEEDED", "INSUFFICIENT_FUNDS"
  execution_method: ExecutionMethod;
}
```

**Frequency**: ~50 events/sec

**Retention**: 7 days (audit trail)

### 5. SYSTEM Stream

System lifecycle and health monitoring.

#### SystemStartEvent
```fbs
table SystemStartEvent {
  type: string;               // Set by application to "SystemStart"
  trace_id: string;
  timestamp: int64;
  version: string;
  environment: string;        // "production" | "staging" | "development"
  components: [string];       // ["scanner", "planner", "executor"]
  config_hash: string;        // Hash of configuration for audit trail
}
```

**Subject**: `system.lifecycle.start.{environment}`

#### KillSwitchEvent
```fbs
table KillSwitchEvent {
  type: string;               // Set by application to "KillSwitch"
  trace_id: string;
  timestamp: int64;
  trigger: KillSwitchTrigger; // Manual | SlotDrift | SystemStalled | ExcessiveLoss
  severity: Severity;         // Critical
  message: string;
  metadata: string;           // JSON metadata for debugging (empty if not set)
}
```

**Use Case**: Emergency stop for system safety. All services must subscribe and halt immediately.

**Subject**: `system.killswitch.{trigger}`

**Target Latency**: <100ms shutdown

#### ConnectionHealthEvent
```fbs
table ConnectionHealthEvent {
  type: string;               // Set by application to "ConnectionHealth"
  trace_id: string;
  timestamp: int64;
  connection_type: string;    // "rpc" | "websocket" | "nats"
  endpoint: string;
  status: ConnectionStatus;   // Connected | Degraded | Reconnecting | Disconnected
  latency_ms: int64 = -1;     // -1 if not set
  error_count: int32;
}
```

**Subject**: `system.health.connection.{type}.{status}`

**Frequency**: 1-10 events/sec

**Retention**: 30 days (file storage)

### 6. METRICS Stream

Performance metrics for observability.

#### LatencyMetricEvent
```fbs
table LatencyMetricEvent {
  type: string;               // Set by application to "LatencyMetric"
  trace_id: string;
  timestamp: int64;
  component: string;          // "scanner", "planner", "executor"
  operation: string;          // "detect_arbitrage", "simulate", "submit"
  latency_us: uint64;         // Latency in microseconds
  p50_us: uint64 = 0;         // 50th percentile (0 if not set)
  p95_us: uint64 = 0;         // 95th percentile
  p99_us: uint64 = 0;         // 99th percentile
}
```

#### ThroughputMetricEvent
```fbs
table ThroughputMetricEvent {
  type: string;               // Set by application to "ThroughputMetric"
  trace_id: string;
  timestamp: int64;
  component: string;
  metric_type: string;        // "events_per_sec", "opportunities_per_sec"
  value: double;
  window_sec: uint32;         // Measurement window in seconds
}
```

#### PnLMetricEvent
```fbs
table PnLMetricEvent {
  type: string;               // Set by application to "PnLMetric"
  trace_id: string;
  timestamp: int64;
  strategy: string;           // "two_hop_arbitrage", "triangular_arbitrage"
  token_pair: string;         // "SOL-USDC" or empty for aggregate
  realized_pnl_usd: double;
  unrealized_pnl_usd: double;
  total_trades: uint64;
  winning_trades: uint64;
  losing_trades: uint64;
  win_rate: double;           // 0.0 - 1.0
  average_profit_usd: double;
  max_profit_usd: double;
  max_loss_usd: double;
}
```

**Subject**: `metrics.latency.{component}`, `metrics.throughput.{component}`, `metrics.pnl.{strategy}`

**Frequency**: 1-5k events/sec

**Retention**: 1 hour (memory)

## Performance Results

### Latency Breakdown (Before → After)

| Stage | JSON | FlatBuffers | Improvement |
|-------|------|-------------|-------------|
| Scanner Detection | 8ms | 8ms | - |
| Event Serialization | 12ms | 1ms | **92% faster** |
| NATS Publish | 3ms | 1ms | **67% faster** |
| NATS Subscribe | 2ms | 1ms | **50% faster** |
| Event Deserialization | 15ms | <1ms | **94% faster** |
| Planner Processing | 55ms | 8ms | **85% faster** |
| **Scanner → Planner** | **95ms** | **16ms** | **6x faster** |
| **Full Pipeline** | **147ms** | **95ms** | **35% faster** |

### Resource Usage

```
CPU (Event Processing):
- JSON:        100% baseline
- FlatBuffers: 13%           [87% reduction!]

Memory Allocations:
- JSON:        ~50 allocations/event
- FlatBuffers: 1 allocation/event   [98% reduction!]

Message Size (TwoHopArbitrageEvent):
- JSON:        450 bytes
- FlatBuffers: 250 bytes     [44% smaller!]
```

## Implementation Highlights

### TypeScript: Zero-Copy Consumption

```typescript
import { Consumer } from '@repo/flatbuf-events';
import { TwoHopArbitrageEvent } from './generated/opportunities';

const consumer = new Consumer(natsConnection);

// Subscribe to opportunities
await consumer.subscribe<TwoHopArbitrageEvent>(
  'OPPORTUNITIES',
  'opportunity.arbitrage.>',
  'arbitrage-scanner',
  async (event, ack, nak) => {
    // Zero-copy access - no JSON.parse()!
    const profitBps = event.estimatedProfitBps();
    const tokenStart = event.tokenStart();

    if (profitBps > 50) {  // 0.5% minimum
      console.log(`Opportunity: ${tokenStart} → ${profitBps}bps`);
      await ack();
    }
  }
);
```

### Go: High-Performance Publishing

```go
import (
    "github.com/google/flatbuffers/go"
    "github.com/nats-io/nats.go"
    "solana-trading/flatbuf/opportunities"
)

// Build event
builder := flatbuffers.NewBuilder(512)
event := opportunities.CreateTwoHopArbitrageEvent(builder, /* ... */)
builder.Finish(event)

// Publish to NATS
subject := fmt.Sprintf("opportunity.arbitrage.two_hop.%s.%s", token1, token2)
nc.Publish(subject, builder.FinishedBytes())  // Zero-copy!
```

### Rust: Zero-Allocation Parsing

```rust
use flatbuffers::root;
use crate::flatbuf::opportunities::TwoHopArbitrageEvent;

// Receive from NATS
let msg = subscription.next().await?;

// Zero-allocation parse
let event = root::<TwoHopArbitrageEvent>(&msg.data)?;

// Direct field access
let profit_bps = event.estimated_profit_bps();
let confidence = event.confidence();

if profit_bps > 50 && confidence > 0.8 {
    // Execute trade
}
```

## Helper Methods: Simplified Consumption

Instead of manually routing events by type, we provide helper methods:

```typescript
// Subscribe to market data (MARKET_DATA stream)
await subscriber.subscribeMarketData('price-monitor', async (event, ack, nak) => {
  // Handles: PriceUpdate, LiquidityUpdate, SwapRoute, etc.
});

// Subscribe to arbitrage opportunities (OPPORTUNITIES stream)
await subscriber.subscribeArbitrage('arb-scanner', async (event, ack, nak) => {
  // Handles: TwoHopArbitrage, TriangularArbitrage
});

// Subscribe to system events (SYSTEM stream)
await subscriber.subscribeSystemLifecycle('monitor', async (event, ack, nak) => {
  // Handles: SystemStart, SystemShutdown, KillSwitch
});
```

## Migration Path

### Phase 1-3: Infrastructure ✅ Complete
- FlatBuffer schemas defined
- Code generation for TypeScript, Go, Rust
- `@repo/flatbuf-events` package with Consumer/Publisher

### Phase 4-5: Scanner & Planner ✅ Complete
- Scanner publishes `TwoHopArbitrageEvent` via FlatBuffers
- Planner subscribes and validates
- 6x faster Scanner→Planner

### Phase 6: Executor ⚠️ In Progress
- Executor consumes `ExecutionPlanEvent`
- Publishes `ExecutionResultEvent`
- Target: <20ms execution latency

### Phase 7: End-to-End Testing ⏳ Next
- Full pipeline testing
- Load testing (1000+ opportunities/sec)
- Failover testing

## Key Takeaways

### 1. Zero-Copy Is Worth It
FlatBuffers' zero-copy deserialization eliminated 15ms+ of JSON parsing latency. In HFT, this matters.

### 2. Event-Type Streams > Priority Streams
Organizing streams by **what** events represent (market data vs opportunities) is more intuitive than **how urgent** they are.

### 3. Schema-Driven Development Rocks
One `.fbs` file → Type-safe code for TypeScript, Go, Rust. Schema evolution without breaking changes.

### 4. Measure Everything
Without profiling, we wouldn't have known JSON serialization was 12-15ms. Measure first, optimize second.

### 5. FlatBuffers + NATS = Fast Microservices
NATS provides low-latency messaging (1-2ms), FlatBuffers provides zero-copy serialization. Together, they enable sub-100ms pipelines.

## What's Next

1. **Complete Executor Implementation**: Finish Phase 6 with `ExecutionResultEvent` publishing
2. **Load Testing**: Stress test with 1000+ opportunities/sec
3. **Grafana Dashboards**: Visualize event flows and latencies
4. **Strategy Optimization**: Use audit trail (EXECUTED stream) for backtesting
5. **Shredstream Integration**: Sub-1ms slot updates for even faster execution

## Resources

- **Code**: [solana-trading-system on GitHub](https://github.com/guidebee/solana-trading-system)
- **FlatBuffers**: [https://google.github.io/flatbuffers/](https://google.github.io/flatbuffers/)
- **NATS JetStream**: [https://docs.nats.io/nats-concepts/jetstream](https://docs.nats.io/nats-concepts/jetstream)
- **Related Post**: [Building a Cross-Language Event System](/posts/2025/12/cross-language-event-system-for-solana-trading/)

---

*Building a Solana HFT trading system requires obsessive attention to latency. This FlatBuffers migration cut 52ms from our pipeline - that's 52ms more time to execute trades before opportunities disappear. Every millisecond counts.*

**Questions? Thoughts?** Find me on [Twitter/X](https://twitter.com/guidebee) or [GitHub](https://github.com/guidebee).
