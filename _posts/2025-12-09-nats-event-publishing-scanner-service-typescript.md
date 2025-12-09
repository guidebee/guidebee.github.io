---
layout: single
title: "Day's Work: NATS Event Publishing and TypeScript Scanner Service Foundation"
date: 2025-12-09
permalink: /posts/2025/12/nats-event-publishing-scanner-service-typescript/
categories:
  - blog
tags:
  - solana
  - trading
  - nats
  - typescript
  - market-data
  - observability
  - scanner
excerpt: "Published market events to NATS JetStream in the quote-service with comprehensive Grafana monitoring, and created a TypeScript scanner-service stub as the foundation for real-time trading opportunity detection. The TypeScript version will be implemented first, with a high-performance Rust version planned for later."
---

## TL;DR

Today's development session focused on two major milestones:

1. **NATS Event Publishing**: Successfully integrated NATS JetStream event publishing into the quote-service with real-time market data distribution
2. **Scanner Service Foundation**: Created TypeScript scanner-service stub at `ts/apps/scanner-service` to scan for trading opportunities

The scanner-service will initially be implemented in TypeScript for rapid development and testing, with a high-performance Rust version planned for production deployment.

## 1. NATS Event Publishing in Quote-Service

### Overview

The quote-service now publishes real-time market events to NATS JetStream, enabling event-driven architecture for trading strategies, monitoring systems, and analytics services.

### Quote-Service Summary

The **SolRoute Quote Service** is a high-performance HTTP service that provides instant DEX quotes by periodically caching the best routes for token pair trades. Instead of querying DEX pools in real-time (which takes 15-30 seconds), the service maintains up-to-date cached quotes and returns them in milliseconds.

**Key Features:**
- **Instant Responses**: Returns cached quotes in milliseconds instead of 15-30 seconds
- **Multi-Protocol Support**: Queries Raydium (AMM, CLMM, CPMM), PumpSwap, Meteora DLMM, and Whirlpool
- **Real-Time Updates**: WebSocket subscriptions for live pool state changes
- **External Quote APIs**: Integration with Jupiter, DFlow, OKX, BLXR, and QuickNode
- **Oracle Integration**: Pyth Network integration for SOL/USD and LST token pricing
- **Dynamic Pair Management**: REST API to add/remove token pairs on-the-fly
- **LST Token Support**: Pre-configured with 12 popular liquid staking tokens
- **Market Events**: Publishes real-time market data to NATS JetStream
- **Production Observability**: Structured logging, Prometheus metrics, and OpenTelemetry tracing

### Event Types Published

The quote-service now publishes the following event types to NATS:

#### PriceUpdateEvent
Published when token price changes are detected.

**Subject:** `market.price.update`

**Example:**
```json
{
  "eventType": "PriceUpdate",
  "timestamp": "2025-12-09T10:30:45Z",
  "inputMint": "So11111111111111111111111111111111111111112",
  "outputMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "protocol": "RaydiumAmm",
  "poolId": "58oQChx4yWmvKdwLLZzBi4ChoCc2fqCUWBkwMihLYQo2",
  "oldPrice": "195500000000",
  "newPrice": "195750000000",
  "priceChange": "250000000",
  "percentChange": 0.128,
  "inputSymbol": "SOL",
  "outputSymbol": "USDC",
  "slot": 285432156
}
```

#### SpreadUpdateEvent
Published when price spread between DEXes exceeds threshold (default: 1.0%).

**Subject:** `market.spread.update`

#### LargeTradeEvent
Published when a trade exceeds the configured USD threshold (default: $10,000).

**Subject:** `market.trade.large`

#### LiquidityUpdateEvent
Published when pool liquidity is scanned or changes.

**Subject:** `market.liquidity.update`

#### VolumeSpikeEvent
Published when update frequency exceeds threshold (indicates high trading activity).

**Subject:** `market.volume.spike`

**Default Threshold:** 10 updates/minute

#### PoolStateChangeEvent
Published when pool state changes (new pool, update, removal).

**Subject:** `market.pool.statechange`

#### SlotUpdateEvent
Published Solana slot updates (every 100 slots to reduce noise).

**Subject:** `market.slot.update`

### Grafana Dashboard: Events by Type

To monitor the event publishing system, I've set up a comprehensive Grafana dashboard showing events by type:

![Events by Type Dashboard](../images/event-by-type.png)

The dashboard visualizes:
- **Total events published** by event type (PriceUpdate, SpreadUpdate, LiquidityUpdate, etc.)
- **Publishing errors** for troubleshooting
- **Real-time event rate** to monitor system health
- **Event distribution** to understand market activity patterns

### Configuration

**Command-line flags:**
```bash
./quote-service \
  -nats="nats://localhost:4222" \
  -largeTradeThreshold=10000 \
  -volumeSpikeThreshold=10 \
  -spreadThreshold=1.0
```

| Flag | Description | Default |
|------|-------------|---------|
| `-nats` | NATS server URL (empty to disable) | `nats://localhost:4222` |
| `-largeTradeThreshold` | Large trade threshold in USD | `10000` |
| `-volumeSpikeThreshold` | Updates per minute for spike detection | `10` |
| `-spreadThreshold` | Spread alert threshold (percent) | `1.0` |

**Disable market events:**
```bash
./quote-service -nats=""
```

### Monitoring Endpoints

**Event stats endpoint:**
```bash
GET /events/stats
```

**Response:**
```json
{
  "enabled": true,
  "nats_connected": true,
  "nats_servers": 1,
  "nats_url": "nats://localhost:4222",
  "tracked_pairs": 24,
  "last_slot": 285432245,
  "large_trade_threshold_usd": 10000,
  "volume_spike_threshold": 10,
  "spread_threshold_percent": 1.0
}
```

**Prometheus metrics:**
- `market_events_published_total{event_type}` - Total events published
- `market_events_publish_errors_total{event_type}` - Publishing errors
- `market_events_marshal_errors_total{event_type}` - JSON marshaling errors
- `market_events_enabled` - Whether events are enabled (0 or 1)
- `nats_connections_total{status}` - NATS connection attempts

### Performance Characteristics

- **Async Publishing**: Events published in goroutines to avoid blocking quote requests
- **Zero Latency Impact**: Event publishing doesn't affect quote response times
- **Memory Storage**: Uses NATS memory storage for sub-millisecond event delivery
- **Automatic Reconnection**: NATS client automatically reconnects on connection loss

## 2. TypeScript Scanner Service Foundation

### Overview

Created a new TypeScript service stub at `C:\Trading\solana-trading-system\ts\apps\scanner-service` that will consume market events from NATS and scan for trading opportunities.

### Why TypeScript First?

The scanner-service will be implemented in TypeScript initially for several reasons:

1. **Rapid Development**: TypeScript allows for faster iteration during development
2. **Rich Ecosystem**: Access to extensive npm packages for market analysis
3. **Easy Integration**: Seamless integration with existing TypeScript services
4. **Prototyping**: Test trading strategies and algorithms quickly
5. **Team Collaboration**: Easier for team members to contribute and review

### Rust Version Planned

Once the TypeScript version is stable and trading strategies are validated, we'll implement a high-performance Rust version for production:

1. **Performance**: Sub-millisecond latency for opportunity detection
2. **Concurrency**: Efficient parallel processing of market events
3. **Memory Efficiency**: Lower resource usage for long-running processes
4. **Type Safety**: Compile-time guarantees for critical trading logic
5. **System Integration**: Direct integration with Solana RPC and DEX protocols

### Scanner Service Responsibilities

The scanner-service will:

1. **Consume Market Events**: Subscribe to NATS JetStream for real-time market data
2. **Detect Opportunities**: Analyze spreads, price movements, and liquidity changes
3. **Strategy Execution**: Implement various trading strategies:
   - **Arbitrage**: Cross-DEX price differences
   - **Market Making**: Provide liquidity and earn fees
   - **Trend Following**: Momentum-based trading
   - **Mean Reversion**: Price deviation from moving averages
4. **Risk Management**: Position sizing, stop-loss, and exposure limits
5. **Performance Tracking**: Monitor strategy profitability and execution quality

### Integration with System Architecture

```
┌─────────────────────────────────────────────────────────┐
│              system-initializer                         │
│  - Creates NATS JetStream streams (EVENTS_*, MARKET)   │
│  - Sets up priority-based event infrastructure         │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│                 quote-service                           │
│  - Fetches LST prices from Pyth oracles                │
│  - Publishes market events to NATS                     │
│  - Provides instant cached quotes                      │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼ (publishes events)
┌─────────────────────────────────────────────────────────┐
│               NATS JetStream                            │
│  - market.price.update                                 │
│  - market.spread.update                                │
│  - market.trade.large                                  │
│  - market.liquidity.update                             │
│  - market.volume.spike                                 │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼ (subscribes to events)
┌─────────────────────────────────────────────────────────┐
│         scanner-service (TypeScript)                    │
│  - Subscribe to all market.* events                    │
│  - Detect arbitrage opportunities                      │
│  - Identify trading signals                            │
│  - Publish opportunity events                          │
│  - Monitor strategy performance                        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│            Trading Execution                            │
│  - Execute trades via Solana transactions              │
│  - Manage positions and risk                           │
│  - Track P&L and performance                           │
└─────────────────────────────────────────────────────────┘
```

### Development Roadmap

**Phase 1: TypeScript Implementation (Current)**
- ✅ Service stub created
- 🚧 NATS event subscription
- 🚧 Basic arbitrage detection
- 🚧 Strategy framework
- 🚧 Performance monitoring

**Phase 2: Strategy Development**
- Cross-DEX arbitrage
- LST premium trading
- Market making algorithms
- Risk management system

**Phase 3: Production Hardening**
- Error handling and recovery
- Performance optimization
- Comprehensive testing
- Production deployment

**Phase 4: Rust Migration**
- Port validated strategies to Rust
- Performance benchmarking
- Gradual rollout
- Side-by-side comparison

## Impact and Next Steps

### What This Enables

Today's work creates the foundation for event-driven trading:

1. **Real-Time Market Data**: Quote-service broadcasts all market changes via NATS
2. **Decoupled Architecture**: Scanner-service operates independently, consuming events
3. **Scalable Design**: Multiple scanner instances can process different strategies
4. **Observable System**: Grafana dashboards provide full visibility into event flow

### Integration Benefits

The event publishing system provides several key benefits:

```
┌──────────────────────────────────────────────────────┐
│           Benefits of Event Architecture             │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. Decoupling: Services communicate via events     │
│  2. Scalability: Add consumers without changing     │
│     producers                                       │
│  3. Reliability: NATS provides guaranteed delivery  │
│  4. Flexibility: Easy to add new event types       │
│  5. Observability: Monitor entire data flow        │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Next Steps

1. **Complete Scanner TypeScript Implementation**:
   - NATS event subscription and processing
   - Arbitrage opportunity detection algorithm
   - Strategy pattern implementation
   - Performance metrics and monitoring

2. **Strategy Development**:
   - Cross-DEX arbitrage strategy
   - LST premium capture strategy
   - Volume spike momentum strategy
   - Mean reversion strategy

3. **Testing and Validation**:
   - Backtest strategies on historical data
   - Paper trading with live events
   - Performance analysis and optimization
   - Risk management validation

4. **Production Deployment**:
   - Deploy TypeScript scanner to production
   - Monitor strategy performance
   - Iterate and improve based on real data
   - Plan Rust migration for proven strategies

5. **Rust High-Performance Version**:
   - Design Rust scanner architecture
   - Port validated TypeScript strategies
   - Performance benchmarking
   - Gradual migration to production

## Conclusion

Today's work represents a significant milestone in building the Solana trading system:

- **Event Publishing**: Quote-service now broadcasts real-time market data to NATS with full observability
- **Scanner Foundation**: Created TypeScript scanner-service stub as the foundation for trading opportunity detection
- **Development Strategy**: TypeScript-first approach for rapid iteration, followed by Rust for production performance

The system now has a complete event-driven architecture enabling sophisticated trading strategies with real-time market data and comprehensive monitoring.

---

**Related Posts:**
- [Getting Started: Building a Solana Trading System](/posts/2025/12/getting-started-building-solana-trading-system/) - Project overview
- [System Initializer Rename, LST Support, and Market Events](/posts/2025/12/system-initializer-lst-support-market-events/) - Previous market events work
- [Building a Cross-Language Event System](/posts/2025/12/cross-language-event-system-for-solana-trading/) - NATS JetStream implementation

**Technical Documentation:**
- [Quote Service README](https://github.com/guidebee/solana-trading-system/tree/main/go/cmd/quote-service)
- [Quote Service Market Events Documentation](https://github.com/guidebee/solana-trading-system/blob/main/go/cmd/quote-service/MARKET_EVENTS.md)
- [Scanner Service README](https://github.com/guidebee/solana-trading-system/tree/main/ts/apps/scanner-service)

## Connect

- **GitHub:** [github.com/guidebee](https://github.com/guidebee)
- **LinkedIn:** [linkedin.com/in/guidebee](https://linkedin.com/in/guidebee)

---

*This is post #6 in the Solana Trading System development series. Follow along as I document the journey from working prototypes to production HFT system.*
