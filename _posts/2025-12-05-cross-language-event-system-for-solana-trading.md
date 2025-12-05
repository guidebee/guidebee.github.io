---
layout: single
title: "Building a Cross-Language Event System for Solana Trading: Rust, Go, and TypeScript"
date: 2025-12-05
permalink: /posts/2025/12/cross-language-event-system-for-solana-trading/
categories:
  - blog
tags:
  - solana
  - rust
  - golang
  - typescript
  - microservices
  - nats
  - event-driven
  - trading
excerpt: "Implementing a unified event system across Rust, Go, and TypeScript for the Solana trading system. Includes market data events, system health monitoring, kill switches, and arbitrage opportunities - all communicating via NATS JetStream."
---

## TL;DR

Today I built a comprehensive cross-language event system for the Solana trading system that enables Rust, Go, and TypeScript services to communicate seamlessly via NATS. The system includes market data events, system lifecycle management, health monitoring, kill switches, and arbitrage opportunity detection - all with priority-based routing and type-safe serialization.

## What Got Built Today

### 1. Market Events Package (Rust, Go, TypeScript)

Created matching event definitions across all three languages:

**Event Categories:**
- **Market Data Events**: Price updates, liquidity changes, large trades, spreads, volume spikes
- **System Lifecycle Events**: System start/shutdown, kill switch activation
- **System Health Events**: Slot drift detection, system stall monitoring, RPC/WebSocket health
- **Arbitrage Events**: Price discrepancies, multi-hop routes, triangular arbitrage

**Key Achievement:** All three implementations serialize to identical JSON, enabling seamless cross-language communication.

### 2. Event Preparer Service (TypeScript)

Built a TypeScript service that initializes and manages the NATS JetStream infrastructure:

**Features:**
- Priority-based stream configuration (Critical, High, Normal, Low)
- Automatic stream and consumer creation
- High-level publisher/subscriber interfaces
- Health check endpoints
- Docker containerization with multi-stage builds

**Why Priority-Based Streams?**

Instead of organizing streams by event type, I used priority levels:

| Priority | Retention | Storage | Use Case |
|----------|-----------|---------|----------|
| Critical | 24 hours | File | Kill switches, system failures |
| High | 5 minutes | Memory | Arbitrage opportunities, large trades |
| Normal | 1 hour | Memory | Price updates, liquidity changes |
| Low | 5 minutes | Memory | Slot updates, routine monitoring |

This approach allows different processing guarantees and latency optimization based on event importance rather than event type.

## Architecture Highlights

### NATS Subject Hierarchy

```
market.*
├── price.{symbol}                   # Price updates
├── liquidity.{poolId}               # Pool liquidity
├── trade.large.{dex}                # Large trades
├── spread.{buyDex}.{sellDex}        # Price spreads
├── volume.spike.{symbol}            # Volume spikes
├── pool.{dex}.{poolId}              # Pool state
└── slot.{slot}                      # Slot updates

system.*
├── lifecycle.*
│   ├── start.{environment}          # System initialization
│   └── shutdown.{reason}            # Graceful/emergency shutdown
├── killswitch.{trigger}             # Emergency stop
└── health.*
    ├── slot_drift.{severity}        # Lagging behind chain
    ├── stalled.{component}          # Component failures
    └── connection.{type}.{status}   # RPC/WebSocket status

arbitrage.*
├── opportunity.{buyDex}.{sellDex}   # Simple arbitrage
├── route.{hops}_hops                # Multi-hop paths
└── triangular.{id}                  # Cyclic arbitrage
```

### Cross-Language Compatibility Example

Here's how the same event works across all three languages:

**Rust:**
```rust
let event = MarketEvent::PriceUpdate(PriceUpdateEvent {
    token: "So11111111111111111111111111111111111111112".to_string(),
    symbol: "SOL".to_string(),
    price_usd: 150.50,
    price_sol: 1.0,
    source: "pyth".to_string(),
    timestamp: get_timestamp(),
    slot: 12345678,
});
```

**Go:**
```go
event := events.NewPriceUpdateEvent(
    "So11111111111111111111111111111111111111112",
    "SOL",
    150.50,
    1.0,
    "pyth",
    time.Now().UnixMilli(),
    12345678,
)
```

**TypeScript:**
```typescript
const event: PriceUpdateEvent = {
  type: 'PriceUpdate',
  token: 'So11111111111111111111111111111111111111112',
  symbol: 'SOL',
  priceUsd: 150.50,
  priceSol: 1.0,
  source: 'pyth',
  timestamp: Date.now(),
  slot: 12345678,
};
```

All three serialize to the same JSON:
```json
{
  "type": "PriceUpdate",
  "token": "So11111111111111111111111111111111111111112",
  "symbol": "SOL",
  "priceUsd": 150.50,
  "priceSol": 1.0,
  "source": "pyth",
  "timestamp": 1733400000,
  "slot": 12345678
}
```

## Kill Switch Foundation (Issue #21)

The event system provides the foundation for implementing the kill switch described in the project roadmap:

**Kill Switch Triggers:**
- Manual (operator-initiated)
- SlotDrift (system lagging >100 slots behind chain)
- SystemStalled (no updates for >30 seconds)
- ConnectionFailure (all RPC connections down)
- ExcessiveLoss (loss threshold exceeded)
- RpcFailure (critical RPC errors)
- InvalidState (inconsistent system state)

**Example Implementation:**
```rust
// Monitor slot drift
if current_slot - expected_slot > 100 {
    let kill_switch = MarketEvent::KillSwitch(KillSwitchEvent {
        trigger: KillSwitchTrigger::SlotDrift,
        severity: Severity::Critical,
        message: format!("Drift: {} slots", drift),
        timestamp: get_timestamp(),
        metadata: Some(r#"{"drift_slots": 100}"#.to_string()),
    });
    publisher.publish(kill_switch).await?;
}

// All services subscribe to kill switch
subscriber.subscribe("system.killswitch.*").await?;
while let Some(MarketEvent::KillSwitch(e)) = subscriber.next().await {
    eprintln!("KILL SWITCH: {:?}", e.trigger);
    halt_all_trading().await?;
}
```

## Go Integration Highlight

Special attention was paid to integrating with the existing Go quote-service. The Rust `SwapHop` struct matches the Go `SwapInfo` struct exactly:

**Go (quote-service):**
```go
type SwapInfo struct {
    AMMKey     string `json:"ammKey"`
    Label      string `json:"label"`
    InputMint  string `json:"inputMint"`
    OutputMint string `json:"outputMint"`
    InAmount   string `json:"inAmount"`
    OutAmount  string `json:"outAmount"`
    FeeAmount  string `json:"feeAmount"`
    FeeMint    string `json:"feeMint"`
}
```

**Rust (market-events):**
```rust
#[derive(Serialize, Deserialize)]
pub struct SwapHop {
    #[serde(rename = "ammKey")]
    pub amm_key: String,
    pub label: String,
    #[serde(rename = "inputMint")]
    pub input_mint: String,
    #[serde(rename = "outputMint")]
    pub output_mint: String,
    #[serde(rename = "inAmount")]
    pub in_amount: String,
    #[serde(rename = "outAmount")]
    pub out_amount: String,
    #[serde(rename = "feeAmount")]
    pub fee_amount: String,
    #[serde(rename = "feeMint")]
    pub fee_mint: String,
}
```

This enables the Rust scanner to directly consume routes from the Go quote-service and emit them as events without any transformation.

## Docker Deployment

The event-preparer service is fully containerized with a multi-stage Dockerfile:

```dockerfile
# Stage 1: Build dependencies
FROM node:20-alpine AS builder
WORKDIR /app
# Install pnpm and build all packages

# Stage 2: Production runtime
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/ts/node_modules ./node_modules
COPY --from=builder /app/ts/packages ./packages
COPY --from=builder /app/ts/apps/event-preparer ./apps/event-preparer
CMD ["node", "apps/event-preparer/dist/index.js"]
```

Integrated into the main docker-compose stack:

```yaml
event-preparer:
  build:
    context: ../..
    dockerfile: ts/apps/event-preparer/Dockerfile
  environment:
    - NATS_SERVERS=nats://nats:4222
    - NODE_ENV=production
  depends_on:
    - nats
```

## Event Priority System

Extended the priority system from 3 to 4 levels:

| Priority | Value | Use Cases |
|----------|-------|-----------|
| Critical | 4 | Kill switches, system lifecycle, immediate action required |
| High | 3 | Arbitrage opportunities, severe health issues, time-sensitive |
| Normal | 2 | Price updates, liquidity changes, regular market data |
| Low | 1 | Slot updates, healthy connections, background monitoring |

Priority determines both NATS stream routing and processing order, allowing critical events to bypass normal queues.

## Package Structure

```
solana-trading-system/
├── rust/
│   └── market-events/              # Rust event definitions
│       ├── src/
│       │   ├── events.rs           # All event types
│       │   ├── builder.rs          # Helper constructors
│       │   ├── publisher.rs        # NATS publishing
│       │   └── subscriber.rs       # NATS subscribing
│       ├── examples/
│       │   └── system_events.rs    # Usage examples
│       └── EVENT_SYSTEM.md         # Documentation
├── go/
│   └── pkg/events/                 # Go event definitions
│       ├── events.go               # Event types
│       ├── events_test.go          # Tests
│       └── README.md               # Go-specific docs
└── ts/
    ├── packages/
    │   └── market-events/          # TypeScript event types
    │       ├── src/
    │       │   ├── events.ts       # Type definitions
    │       │   └── index.ts        # Exports
    │       └── README.md           # TS-specific docs
    └── apps/
        └── event-preparer/         # NATS infrastructure service
            ├── src/
            │   ├── service.ts      # Main service
            │   ├── nats-manager.ts # JetStream setup
            │   ├── publisher.ts    # Publishing API
            │   └── subscriber.ts   # Subscribing API
            ├── examples/           # Usage examples
            ├── Dockerfile          # Container build
            └── README.md           # Service documentation
```

## Integration Points

The event system now provides the foundation for:

1. **Scanner Service** (Rust): Emit market data events, detect opportunities
2. **Executor Service** (TypeScript/Rust): Subscribe to arbitrage events, execute trades
3. **Quote Service** (Go): Emit swap routes, consume price updates
4. **Monitor Service**: Subscribe to all health events, trigger kill switches
5. **Kill Switch Coordinator**: Central safety system subscribing to all critical events

## What's Next

Now that the event infrastructure is in place, the next steps are:

1. **Week 1 (Current)**: Complete environment setup and monitoring stack
2. **Week 2**: Implement kill switch infrastructure using these events
3. **Week 3-4**: Build scanner service to emit market data events
4. **Week 5-7**: Build executor service to consume arbitrage events
5. **Week 8+**: Production LST arbitrage with full event-driven coordination

## Technical Wins

**Type Safety:** All events are strongly typed in their respective languages, catching errors at compile time.

**Zero-Copy Serialization:** Events serialize directly to JSON without intermediate transformations.

**Backward Compatibility:** New event types can be added without breaking existing subscribers.

**Scalability:** Priority-based streams allow independent scaling of critical vs normal event processing.

**Observability:** NATS JetStream provides built-in monitoring of message rates, consumer lag, and stream health.

## Resources

- [Project Repository](https://github.com/guidebee/solana-trading-system) (if public)
- [NATS JetStream Docs](https://docs.nats.io/nats-concepts/jetstream)
- [Previous Post: Getting Started](/posts/2025/12/getting-started-building-solana-trading-system/)

## Conclusion

Building a cross-language event system is critical for polyglot microservices. By ensuring type-safe, compatible event definitions across Rust, Go, and TypeScript, the trading system can now coordinate across services efficiently while maintaining the performance benefits of each language:

- **Rust**: Zero-copy market data parsing with Shredstream
- **Go**: Fast local pool math with SolRoute SDK (2-10ms)
- **TypeScript**: Flexible business logic for strategies

The event-preparer service provides a solid foundation for the rest of the system, enabling priority-based routing, guaranteed delivery, and centralized monitoring. Most importantly, the kill switch foundation is now in place to prevent catastrophic losses during development and production.

Next up: implementing the actual kill switch logic and integrating it with all services!
