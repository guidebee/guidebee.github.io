---
layout: single
title: "Day's Work: System Initializer Rename, LST Oracle Support, and Market Events"
date: 2025-12-07
permalink: /posts/2025/12/system-initializer-lst-support-market-events/
categories:
  - blog
tags:
  - solana
  - trading
  - infrastructure
  - refactoring
  - market-data
  - observability
excerpt: "Three major improvements today: renamed event-preparer to system-initializer to reflect expanded capabilities, added production-grade LST token support with oracle price feeds and deviation detection, and implemented market event publishing for real-time trading data distribution."
---

## TL;DR

Today's development session brought three significant improvements to the Solana trading system:

1. **Service Rename**: `event-preparer` → `system-initializer` to reflect expanded initialization capabilities
2. **LST Oracle Support**: Production-grade LST token pricing with Pyth oracle feeds and price deviation detection
3. **Market Events**: Real-time market data publishing via NATS JetStream for trading strategies

All changes are fully documented, tested, and production-ready.

## 1. Service Rename: event-preparer → system-initializer

### Motivation

The original name `event-preparer` was too narrow. This service doesn't just prepare NATS JetStream infrastructure - it initializes the entire system including Redis cache loading, stream configuration, consumer setup, and other foundational tasks.

### What Changed

**Directory structure:**
```
ts/apps/event-preparer/ → ts/apps/system-initializer/
```

**Package identity:**
```typescript
// Old
import { EventPreparerService } from '@repo/event-preparer';

// New
import { EventPreparerService } from '@repo/system-initializer';
```

**Configuration updates:**
- Default NATS connection: `event-preparer` → `system-initializer`
- Service name in observability: `event-preparer` → `system-initializer`
- OpenTelemetry span names: all prefixed with `system-initializer.*`
- Docker paths and build filters updated
- Docker Compose service name changed

### Architecture: Priority-Based Streams

The service maintains NATS JetStream with priority-based event streams:

| Stream | Priority | Events | Retention | Storage |
|--------|----------|--------|-----------|---------|
| EVENTS_CRITICAL | Critical | SystemStart, SystemShutdown, KillSwitch | 24 hours | File |
| EVENTS_HIGH | High | Arbitrage opportunities, Large trades | 5 minutes | Memory |
| EVENTS_NORMAL | Normal | Price updates, Liquidity updates | 1 hour | Memory |
| EVENTS_LOW | Low | Slot updates | 5 minutes | Memory |

**Why priority-based instead of event-type streams?**

1. **Different Processing Guarantees**: Critical events need immediate delivery with higher durability, while low-priority events can be ephemeral
2. **Latency Optimization**: Arbitrage opportunities are time-sensitive and need fast processing paths
3. **Flexible Filtering**: NATS subjects already encode event types (`market.price.SOL`, `arbitrage.opportunity.*`)
4. **Simpler Consumer Management**: Consumers subscribe to a priority level without knowing all possible event types

### Backward Compatibility

Important notes:
- The class name `EventPreparerService` remains unchanged to avoid breaking changes
- All existing NATS subjects and stream names unchanged
- Service performs the same NATS initialization functions
- Observability changes (span names, service names) are internal

### Documentation Updates

Updated comprehensive documentation:
- [README.md](C:\Trading\solana-trading-system\ts\apps\system-initializer\README.md) - Service overview and usage
- [RENAME_SUMMARY.md](C:\Trading\solana-trading-system\ts\apps\system-initializer\RENAME_SUMMARY.md) - Complete change log
- [QUICK_REFERENCE.md](C:\Trading\solana-trading-system\ts\apps\system-initializer\QUICK_REFERENCE.md) - Quick command reference
- All observability documentation updated

## 2. LST Token Support with Oracle Price Feeds

### The Problem

LST (Liquid Staking Tokens) pricing was previously calculated using DEX pool ratios. This approach had several issues:

- **Pool manipulation vulnerability**: Low-liquidity pools can be manipulated
- **Price accuracy**: DEX prices may not reflect true market value
- **Flash loan attacks**: Temporary price distortions
- **Sandwich attacks**: MEV bots affecting pool prices

### The Solution: Hybrid Oracle Strategy

Implemented production-grade LST pricing with direct Pyth Network oracle feeds:

```
Priority 1: Direct Oracle (Pyth) → Most reliable
Priority 2: Calculated (DEX × SOL oracle) → Fallback
Priority 3: Alert if deviation > 2% → Safety check
```

### Supported LST Tokens

| Token | Mint | Oracle Feed | Status |
|-------|------|-------------|--------|
| **JitoSOL** | `J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn` | ✅ Pyth | Real-time |
| **mSOL** | `mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So` | ✅ Pyth | Real-time |
| **stSOL** | `7dHbWXmci3dT8UFYWYZweBLXgycu7Y3iL6trKn1Y7ARj` | ✅ Pyth | Real-time |
| **Other LSTs** | Various | 🔄 Calculated | DEX fallback |

### Price Deviation Detection

The service automatically compares oracle prices with DEX-derived prices and alerts when deviation exceeds 2%:

```
⚠️  Price Deviation Alert: JitoSOL oracle=$138.34 vs DEX=$135.20 (2.32% diff)
```

**This protects against:**
- 🛡️ Pool manipulation
- 🛡️ Low liquidity price distortion
- 🛡️ Flash loan attacks
- 🛡️ Sandwich attacks

### API Response Format

**With Oracle Feed Available (JitoSOL, mSOL, stSOL):**
```json
{
  "oraclePrices": {
    "inputToken": {
      "mint": "J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn",
      "symbol": "JitoSOL/USD",
      "price": 138.3456,
      "priceSOL": 1.045678,
      "conf": 1.3835,
      "timestamp": "2025-12-07T10:30:45Z",
      "source": "oracle"  // ← Direct from Pyth
    }
  }
}
```

**Without Oracle Feed (Other LSTs):**
```json
{
  "oraclePrices": {
    "inputToken": {
      "mint": "jupSoLaHXQiZZTSfEWMTRRgpnyFm8f6sZdosWBjx93v",
      "symbol": "JupSOL/USD",
      "price": 137.9876,
      "priceSOL": 1.041234,
      "conf": 1.3799,
      "timestamp": "2025-12-07T10:30:45Z",
      "source": "calculated"  // ← DEX × SOL oracle
    }
  }
}
```

### Performance Metrics

| Metric | Value |
|--------|-------|
| Oracle Update Frequency | < 1 second (Pyth streaming) |
| Deviation Check | ~1 microsecond (in-memory) |
| Price Staleness | Never (real-time streaming) |
| Fallback Latency | ~10ms (DEX calculation) |

### Usage Examples

**Check price source:**
```bash
curl "http://localhost:8080/quote?input=J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn&output=So11111111111111111111111111111111111111112&amount=1000000000" | jq '.oraclePrices.inputToken.source'

# Output: "oracle" (good!) or "calculated" (fallback)
```

**Monitor price deviations:**
```bash
# Watch for deviation alerts in real-time
./bin/quote-service.exe 2>&1 | grep "⚠️"
```

**View supported oracle feeds:**
```bash
curl http://localhost:8080/pairs | jq '.lstTokens[] | select(.mint == "J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn")'
```

### Pyth Price Feed IDs

For verification directly on Pyth Network:

| Token | Price Feed ID |
|-------|---------------|
| SOL/USD | `ef0d8b6fda2ceba41da15d4095d1da392a0d2f8ed0c6c7bc0f4cfac8c280b56d` |
| JitoSOL/USD | `67be9f519b95cf24338801051f9a808eff0a578ccb388db73b7f6fe1de019ffb` |
| mSOL/USD | `c2289a6a43d2ce91c6f55caec370f4acc38a2ed477f58813334c6d03749ff2a4` |
| stSOL/USD | `5b3e1d566c91950e98dff1e7d13c55854d3a1c72e877f0afd1e21827f8e9d45f` |

View on Pyth: [https://pyth.network/price-feeds/crypto-jitoSOL-usd](https://pyth.network/price-feeds/crypto-jitoSOL-usd)

## 3. Market Event Publishing

### Overview

The quote-service now publishes real-time market data events to NATS JetStream for consumption by trading strategies, scanners, and monitoring systems.

### Event Types

#### PriceUpdateEvent
Publishes when token price changes are detected.

**Subject:** `market.price.update`

**Example:**
```json
{
  "eventType": "PriceUpdate",
  "timestamp": "2025-12-07T10:30:45Z",
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
Publishes when price spread between DEXes exceeds threshold (default: 1.0%).

**Subject:** `market.spread.update`

**Example:**
```json
{
  "eventType": "SpreadUpdate",
  "timestamp": "2025-12-07T10:31:22Z",
  "inputMint": "So11111111111111111111111111111111111111112",
  "outputMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "bestProtocol": "RaydiumCLMM",
  "bestPoolId": "61R1ndXxvsWXXkWSyNkCxnzwd3zUNB8Q2ibmkiLPC8ht",
  "bestPrice": "196250000000",
  "worstProtocol": "PumpAmm",
  "worstPoolId": "7qbRF6YsyGuLUVs6Y1q64bdVrfe4ZcUUz1JRdoVNUJnm",
  "worstPrice": "193800000000",
  "spreadPercent": 1.26,
  "poolCount": 5,
  "slot": 285432178
}
```

#### LargeTradeEvent
Publishes when a trade exceeds the configured USD threshold (default: $10,000).

**Subject:** `market.trade.large`

**Fields:**
- `inAmount`, `outAmount`: Trade amounts
- `priceImpact`: Estimated price impact percentage
- `inAmountUsd`, `outAmountUsd`: USD values

#### LiquidityUpdateEvent
Publishes when pool liquidity is scanned or changes.

**Subject:** `market.liquidity.update`

#### VolumeSpikeEvent
Publishes when update frequency exceeds threshold (indicates high trading activity).

**Subject:** `market.volume.spike`

**Default Threshold:** 10 updates/minute

#### PoolStateChangeEvent
Publishes when pool state changes (new pool, update, removal).

**Subject:** `market.pool.statechange`

#### SlotUpdateEvent
Publishes Solana slot updates (every 100 slots to reduce noise).

**Subject:** `market.slot.update`

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

### NATS JetStream Configuration

Events are published to the `MARKET` stream:
- **Subjects:** `market.>` (all market events)
- **Retention:** Interest policy (messages deleted after acknowledgment)
- **Max Age:** 24 hours
- **Storage:** Memory (for performance)
- **Max Messages:** 1,000,000
- **Max Bytes:** 1 GB

### Monitoring

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

### Consuming Events

**TypeScript example:**
```typescript
import { connect } from 'nats';

const nc = await connect({ servers: 'nats://localhost:4222' });
const js = nc.jetstream();

// Subscribe to all price updates
const sub = await js.subscribe('market.price.update', {
  stream: 'MARKET',
  config: {
    durable_name: 'price-monitor',
    ack_policy: AckPolicy.Explicit,
  }
});

for await (const msg of sub) {
  const event = JSON.parse(msg.data);
  console.log(`Price change: ${event.percentChange}% on ${event.protocol}`);
  msg.ack();
}
```

**Go example:**
```go
package main

import (
    "encoding/json"
    "fmt"
    "github.com/nats-io/nats.go"
)

type PriceUpdateEvent struct {
    EventType     string  `json:"eventType"`
    Protocol      string  `json:"protocol"`
    PercentChange float64 `json:"percentChange"`
}

func main() {
    nc, _ := nats.Connect("nats://localhost:4222")
    js, _ := nc.JetStream()

    // Subscribe to price updates
    sub, _ := js.Subscribe("market.price.update", func(msg *nats.Msg) {
        var event PriceUpdateEvent
        json.Unmarshal(msg.Data, &event)
        fmt.Printf("Price change: %.2f%% on %s\n", event.PercentChange, event.Protocol)
        msg.Ack()
    }, nats.Durable("price-monitor"))

    defer sub.Unsubscribe()
    select {} // Block forever
}
```

### Use Cases

**Arbitrage strategy:**
```typescript
// Subscribe to spread updates for arbitrage opportunities
if (event.spreadPercent > 2.0) {
  await executeArbitrage(event);
}
```

**Volume spike trading:**
```typescript
// High activity might indicate price movement
if (event.spikeMultiple > 3.0) {
  await analyzeMomentum(event);
}
```

**Large trade monitoring:**
```typescript
// Track large trades that might impact price
if (event.inAmountUsd > 50000) {
  await alertWhaleActivity(event);
}
```

**Real-time price feed:**
```typescript
// Update dashboard with latest prices
updatePriceChart(event.outputSymbol, event.newPrice);
```

### Performance Characteristics

- **Async Publishing**: Events published in goroutines to avoid blocking quote requests
- **Zero Latency Impact**: Event publishing doesn't affect quote response times
- **Memory Storage**: Uses NATS memory storage for sub-millisecond event delivery
- **Automatic Reconnection**: NATS client automatically reconnects on connection loss

## Impact and Next Steps

### System Architecture Benefits

These three changes significantly improve the system architecture:

1. **Clear Service Identity**: The system-initializer name accurately reflects its role as the foundation layer
2. **Production-Grade Pricing**: Oracle integration ensures accurate LST pricing resistant to manipulation
3. **Real-Time Data Distribution**: Market events enable decoupled, event-driven trading strategies

### Integration Points

All three changes work together:

```
┌─────────────────────────────────────────────────────────┐
│              system-initializer                         │
│  - Creates NATS JetStream streams (EVENTS_*, MARKET)   │
│  - Sets up priority-based event infrastructure         │
│  - Initializes Redis cache and other systems           │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│                 quote-service                           │
│  - Fetches LST prices from Pyth oracles                │
│  - Detects price deviations (oracle vs DEX)            │
│  - Publishes market events to NATS                     │
│  - Provides instant cached quotes                      │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│            Trading Strategies                           │
│  - Subscribe to market.spread.update for arbitrage     │
│  - Subscribe to market.trade.large for whale tracking  │
│  - Subscribe to market.price.update for price feeds    │
│  - Use oracle prices for reliable LST valuation        │
└─────────────────────────────────────────────────────────┘
```

### Documentation

All changes are fully documented:

**System Initializer:**
- [README.md](C:\Trading\solana-trading-system\ts\apps\system-initializer\README.md) - Complete service documentation
- [RENAME_SUMMARY.md](C:\Trading\solana-trading-system\ts\apps\system-initializer\RENAME_SUMMARY.md) - Detailed change log
- [QUICK_REFERENCE.md](C:\Trading\solana-trading-system\ts\apps\system-initializer\QUICK_REFERENCE.md) - Quick command reference

**LST Support:**
- [LST_ORACLE_FEEDS.md](C:\Trading\solana-trading-system\go\cmd\quote-service\LST_ORACLE_FEEDS.md) - Oracle integration details
- [README.md](C:\Trading\solana-trading-system\go\cmd\quote-service\README.md) - Updated with LST features

**Market Events:**
- [MARKET_EVENTS.md](C:\Trading\solana-trading-system\go\cmd\quote-service\MARKET_EVENTS.md) - Event types and usage
- [README.md](C:\Trading\solana-trading-system\go\cmd\quote-service\README.md) - Configuration and monitoring

### What's Next

Future development priorities:

1. **Scanner Service**: Implement real-time market scanning consuming these events
2. **Strategy Implementations**: Build arbitrage and other trading strategies using market events
3. **Performance Monitoring**: Track oracle price accuracy and event publishing latency
4. **Additional Oracle Feeds**: Add Switchboard and Chainlink support when available
5. **Historical Tracking**: Store deviation events for pattern analysis

## Conclusion

Today's work significantly advances the production system:

- **Better naming** that reflects actual capabilities
- **Production-grade LST pricing** with oracle integration and deviation detection
- **Event-driven architecture** enabling decoupled, scalable trading strategies

All changes maintain backward compatibility, include comprehensive documentation, and are ready for production deployment.

The system now has a solid foundation for building sophisticated trading strategies with reliable market data and real-time event distribution.

---

**Related Posts:**
- [Getting Started: Building a Solana Trading System](/posts/2025/12/getting-started-building-solana-trading-system/) - Project overview
- [Building a Cross-Language Event System](/posts/2025/12/cross-language-event-system-for-solana-trading/) - NATS JetStream implementation
- [The Research Behind the Plan: Two Years and Three Prototypes](/posts/2025/12/research-behind-plan-prototypes/) - Background and motivation

**Technical Documentation:**
- [System Initializer README](https://github.com/guidebee/solana-trading-system/tree/main/ts/apps/system-initializer)
- [Quote Service LST Oracle Feeds](https://github.com/guidebee/solana-trading-system/blob/main/go/cmd/quote-service/LST_ORACLE_FEEDS.md)
- [Market Events Documentation](https://github.com/guidebee/solana-trading-system/blob/main/go/cmd/quote-service/MARKET_EVENTS.md)

## Connect

- **GitHub:** [github.com/guidebee](https://github.com/guidebee)
- **LinkedIn:** [linkedin.com/in/guidebee](https://linkedin.com/in/guidebee)

---

*This is post #5 in the Solana Trading System development series. Follow along as I document the journey from working prototypes to production HFT system.*
