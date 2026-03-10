---
layout: single
title: "Scanner Implementation Guide - Multi-Strategy System"
permalink: /docs/09-scanner-implementation/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Scanner Implementation Guide - Multi-Strategy System

**Document Version**: 3.0 (Consolidated)
**Date**: 2025-12-20
**Status**: ✅ Production Ready
**Author**: Solution Architect

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Scanner Framework Design](#scanner-framework-design)
4. [Scanner Implementations](#scanner-implementations)
5. [Integration Guide](#integration-guide)
6. [Monitoring & Observability](#monitoring--observability)
7. [Deployment Guide](#deployment-guide)
8. [Best Practices](#best-practices)
9. [Extensibility Patterns](#extensibility-patterns)
10. [Performance Optimization](#performance-optimization)
11. [Troubleshooting](#troubleshooting)

---

## Executive Summary

### Multi-Strategy Scanner System

This document describes the production-ready scanner implementation for the HFT trading system. Unlike earlier versions focused solely on arbitrage, this system supports **multiple trading strategies** through an extensible framework architecture.

**Supported Strategies** (from [11-extensible-strategy-architecture.md](./11-extensible-strategy-architecture.md)):
- ✅ **Arbitrage** - Cross-DEX price discrepancies (Rust + TypeScript)
- ✅ **DCA** - Dollar-cost averaging for position building (TypeScript)
- ✅ **Grid Trading** - Range-bound automated trading (TypeScript)
- ✅ **Long/Short** - Directional position taking (Rust)
- ✅ **Limit Orders** - Price-based order execution
- ✅ **Liquidation Monitoring** - Lending protocol opportunities

### Key Capabilities

**Framework Features**:
- 🚀 **Sub-50ms detection latency** (real-time streaming)
- 🔌 **Pluggable architecture** (add new scanners in hours, not days)
- 📊 **Built-in observability** (Prometheus metrics, Loki logs, Jaeger traces)
- 🔄 **Automatic retry & recovery** (resilient error handling)
- 🎯 **Deduplication** (Redis-based event dedup)
- 📈 **Horizontal scalability** (NATS JetStream event bus)

**Technology Stack**:
- **Scanner Framework**: TypeScript (`@repo/scanner-framework`)
- **Event Bus**: NATS JetStream (6-stream architecture)
- **Observability**: Prometheus + Grafana + Loki + Jaeger
- **Data Sources**: gRPC (Go quote-service), Pyth oracle, Shredstream (future)
- **Language Support**: TypeScript (current), Rust (future migration)

### Performance Targets

| Metric | Prototype | Current | Target | Status |
|--------|-----------|---------|--------|--------|
| **Quote Fetch** | 800-1500ms | 10-20ms | < 10ms | ✅ Achieved |
| **Detection Latency** | N/A (polling) | 10-30ms | < 50ms | ✅ Achieved |
| **Total Execution** | ~1700ms | ~500ms | < 200ms | ⏳ In Progress |

---

## Architecture Overview

### Three-Layer Architecture

The scanner system follows the **Scanner → Planner → Executor** pattern:

```
┌─────────────────────────────────────────────────────────────────┐
│ SCANNER LAYER: Data Collection (Observation Only)               │
├─────────────────────────────────────────────────────────────────┤
│ Framework: @repo/scanner-framework                              │
│ - BaseScanner: Abstract base class                             │
│ - PollingScanner: Periodic data fetching (5-30s intervals)     │
│ - SubscriptionScanner: Real-time streaming (Shredstream, WS)   │
│ - Built-in: Error handling, metrics, retries, backpressure     │
├─────────────────────────────────────────────────────────────────┤
│ Implementations:                                                 │
│ - ArbitrageQuoteScanner: gRPC quote stream → arbitrage          │
│ - DCAScanner: Price monitoring → DCA orders                     │
│ - LimitOrderScanner: Price monitoring → limit order triggers    │
│ - LiquidationScanner: Protocol monitoring → liquidations        │
│ - PythPriceScanner: Pyth oracle → price updates                │
└─────────────────────────────────────────────────────────────────┘
                           ↓ NATS Events (MARKET_DATA, OPPORTUNITIES)
┌─────────────────────────────────────────────────────────────────┐
│ PLANNER LAYER: Strategy Logic (Decision Making)                │
├─────────────────────────────────────────────────────────────────┤
│ Framework: @repo/strategy-framework (✅ ALREADY EXISTS)         │
│ - TradingStrategy interface                                     │
│ - HealthMonitor for strategy tracking                          │
│ - TradeOpportunity types                                        │
├─────────────────────────────────────────────────────────────────┤
│ Service: planner-service (stub for pipeline testing)           │
│ - Strategy orchestration                                        │
│ - Multi-strategy coordination                                   │
│ - Opportunity validation and prioritization                     │
└─────────────────────────────────────────────────────────────────┘
                           ↓ NATS Events (PLANNED)
┌─────────────────────────────────────────────────────────────────┐
│ EXECUTOR LAYER: Transaction Execution (No Strategy Logic)      │
├─────────────────────────────────────────────────────────────────┤
│ Service: executor-service (stub for pipeline testing)          │
│ - Transaction building and signing                             │
│ - Jito bundle submission with MEV protection                   │
│ - Confirmation tracking and result publishing                  │
└─────────────────────────────────────────────────────────────────┘
```

### Design Principles

1. **Single Responsibility**: Each scanner focuses on one data source or strategy type
2. **Framework vs Application**: Framework handles infrastructure, scanners handle business logic
3. **Event-Driven**: Scanners publish to NATS, planners/executors subscribe
4. **Strategy Agnostic**: Same scanner framework supports all trading strategies
5. **Composability**: Mix and match scanners for complex strategies

**Example: Triangular Arbitrage**
- Uses **3 scanners**: ArbitrageQuoteScanner (SOL/USDC), ArbitrageQuoteScanner (USDC/USDT), ArbitrageQuoteScanner (USDT/SOL)
- Planner combines 3 quotes into triangular opportunity
- Same framework, different configuration

---

## Scanner Framework Design

### Core Abstractions

The scanner framework provides two base classes for all scanner implementations:

#### 1. BaseScanner (Abstract)

**Location**: `ts/packages/scanner-framework/src/base.ts`

**Responsibilities**:
- NATS connection and JetStream publishing
- Redis deduplication
- Prometheus metrics collection
- Error handling and retry logic
- Graceful lifecycle management

**Key Methods**:
```typescript
export abstract class BaseScanner {
  // Abstract methods (must implement)
  abstract name(): string;
  abstract start(): Promise<void>;
  abstract stop(): Promise<void>;

  // Template methods (can override)
  protected async initialize(): Promise<void>
  protected async cleanup(): Promise<void>
  protected async handleError(error: Error): Promise<void>

  // Publishing helpers (built-in)
  protected async publish(event: MarketEvent): Promise<void>
  protected async isDuplicate(event: MarketEvent): Promise<boolean>
  protected abstract getEventId(event: MarketEvent): string
}
```

#### 2. PollingScanner (Periodic Fetching)

**Use Cases**: Price feeds, Jupiter quotes, periodic checks (DCA, limit orders)

**Pattern**:
```typescript
export abstract class PollingScanner extends BaseScanner {
  abstract pollInterval: number; // Milliseconds

  // Implement these two methods
  protected abstract fetch(): Promise<any>;
  protected abstract process(data: any): Promise<MarketEvent[]>;
}
```

**Automatic Features**:
- Periodic polling with configurable interval
- Automatic error handling and retry
- Poll duration metrics
- Last successful poll timestamp

#### 3. SubscriptionScanner (Real-time Streaming)

**Use Cases**: gRPC streams, WebSocket subscriptions, Shredstream, high-frequency data

**Pattern**:
```typescript
export abstract class SubscriptionScanner extends BaseScanner {
  abstract maxConcurrency: number;
  abstract maxQueueSize: number;

  // Implement these two methods
  protected abstract subscribe(): Promise<AsyncIterator<any>>;
  protected abstract process(update: any): Promise<MarketEvent[]>;
}
```

**Automatic Features**:
- Backpressure queue with concurrency control
- Automatic stream reconnection
- Queue overflow handling
- Processing duration metrics

### Framework Metrics

**Scanner Metrics** (auto-collected):
- `scanner_events_published_total{scanner, event_type}`
- `scanner_errors_total{scanner, error_type}`
- `scanner_poll_duration_seconds{scanner}` (PollingScanner)
- `scanner_processing_duration_seconds{scanner}` (SubscriptionScanner)
- `scanner_queue_overflows_total{scanner}` (SubscriptionScanner)
- `scanner_events_deduplicated_total{scanner}`
- `scanner_last_poll_success_timestamp{scanner}`
- `scanner_queue_size{scanner}`

**Custom Metrics** (scanner-specific):
- Each scanner can add custom stats via `this.stats.customStats`
- Example: `dcaExecutions`, `opportunitiesDetected`, `quotesReceived`

---

## Scanner Implementations

### 1. Arbitrage Quote Scanner

**Purpose**: Real-time arbitrage detection via gRPC quote streaming from Go quote-service

**Type**: `SubscriptionScanner`
**Location**: `ts/apps/scanner-service/src/scanners/arbitrage-quote-scanner.ts`

**Architecture**:
```
Go Quote Service (Port 50051)
    ↓ gRPC stream (10-20ms per quote)
ArbitrageQuoteScanner
    ├─ Maintains local quote cache
    ├─ Detects arbitrage when both directions available
    ├─ Calculates round-trip profit
    └─ Publishes to NATS: opportunity.arbitrage.detected
```

**Configuration**:
```typescript
{
  tokenPairs: [
    { inputMint: 'SOL_MINT', outputMint: 'JITOSOL_MINT' },
    { inputMint: 'JITOSOL_MINT', outputMint: 'SOL_MINT' },
    // ... 14 LST pairs total
  ],
  amounts: [
    100_000_000,    // 0.1 SOL
    1_000_000_000,  // 1 SOL
    // ... 39 amounts total
  ],
  minProfitBps: 50,        // 0.5%
  maxSlippageBps: 50,      // 0.5%
  minConfidence: 0.8,      // 80%
}
```

**Data Flow**:
1. Go quote-service streams quotes via gRPC (546-624 quote subscriptions)
2. Scanner receives quote and caches it
3. Checks if reverse quote exists in cache
4. If both directions available: calculates round-trip profit
5. If profitable (> 50 bps): publishes `ArbitrageOpportunity` event

**Performance**:
- Quote latency: 10-20ms (vs 800-1500ms Jupiter API)
- Detection latency: < 30ms total
- Throughput: 100+ quotes/sec

**Profit Calculation**:
```typescript
// Round-trip calculation
inputAmount = 1 SOL
  ↓ Forward swap: SOL → JitoSOL
forwardOutput = 1.05 JitoSOL
  ↓ Reverse swap: JitoSOL → SOL
reverseOutput = 1.02 SOL

netProfit = reverseOutput - inputAmount - fees
profitBps = (netProfit / inputAmount) * 10000
// Example: (0.02 / 1.0) * 10000 = 200 bps (2%)
```

**Key Files**:
- Scanner: `scanners/arbitrage-quote-scanner.ts`
- gRPC Client: `clients/grpc-quote-client.ts`
- Profit Calculator: `utils/profit-calculator.ts`
- Quote Handler: `handlers/quote-stream.ts`
- Pyth Client: `clients/pyth-client.ts`

### 2. DCA Scanner

**Purpose**: Execute dollar-cost averaging orders at configured intervals

**Type**: `PollingScanner`
**Location**: `ts/apps/scanner-service/src/scanners/dca-scanner.ts`

**Architecture**:
```
PriceClient (Mock)
    ↓ Fetch price every N seconds
DCAScanner
    ├─ Checks interval timer
    ├─ Checks price condition (optional)
    ├─ Emits SwapRoute event
    └─ Publishes to NATS: market.swaps.route
```

**Configuration**:
```typescript
{
  tokenMint: 'So11111111111111111111111111111111111111112', // SOL
  quoteMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v', // USDC
  interval: 3600,          // 1 hour
  amount: 100_000_000,     // 100 USDC
  targetPrice: 100,        // Only buy if SOL <= $100
  pairName: 'SOL/USDC',
}
```

**Custom Statistics**:
- `dcaExecutions`: Total DCA orders executed
- `lastExecutionPrice`: Price of last execution
- `lastExecutionTime`: Timestamp of last execution
- `skippedDueToPrice`: Count of skips (price too high)
- `skippedDueToInterval`: Count of skips (interval not passed)

**Use Cases**:
- Accumulate SOL/LST over time
- Avoid timing risk with periodic buys
- Build positions without manual intervention

### 3. Limit Order Scanner

**Purpose**: Monitor prices and trigger limit orders when price targets are met

**Type**: `PollingScanner`
**Example**: `docs/13-scanner-framework-optimization.md` (lines 562-682)

**Architecture**:
```
PriceClient
    ↓ Poll prices every 5 seconds
LimitOrderScanner
    ├─ Maintains order book (Map<orderId, LimitOrder>)
    ├─ Checks each order against current price
    ├─ Triggers order when price condition met
    └─ Publishes to NATS: opportunity.limit_order.triggered
```

**Order Management**:
```typescript
addOrder(order: LimitOrder): void
cancelOrder(orderId: string): boolean
getActiveOrders(): LimitOrder[]
```

**Custom Statistics**:
- `ordersTriggered`: Total orders executed
- `activeOrders`: Current open orders

### 4. Liquidation Scanner

**Purpose**: Monitor lending protocols for liquidation opportunities

**Type**: `SubscriptionScanner`
**Example**: `docs/13-scanner-framework-optimization.md` (lines 686-780)

**Architecture**:
```
LendingProtocolClient (Kamino, Solend, etc.)
    ↓ Real-time position updates
LiquidationScanner
    ├─ Monitors position health factors
    ├─ Calculates liquidation profitability
    ├─ Publishes when health factor < threshold
    └─ NATS: opportunity.liquidation.detected
```

**Profitability Calculation**:
```typescript
collateralValue = collateral * collateralPrice
debtValue = debt * debtPrice
potentialProfit = collateralValue * 0.05  // 5% liquidation bonus
```

**Custom Statistics**:
- `liquidationOpportunities`: Total opportunities detected
- `totalPotentialProfit`: Cumulative potential profit

### 5. Pyth Price Scanner

**Purpose**: Stream real-time price updates from Pyth oracle

**Type**: `PollingScanner` (could also be `SubscriptionScanner` with SSE)
**Example**: `docs/11-scanner-service-framework-integration.md` (lines 219-271)

**Architecture**:
```
Pyth Hermes API
    ↓ Fetch prices every 5 seconds
PythPriceScanner
    ├─ Fetches latest price updates
    ├─ Converts to MarketEvent
    └─ Publishes to NATS: market.prices.update
```

**Supported Tokens**:
- SOL, USDC, USDT
- LST tokens: JitoSOL, mSOL, stSOL, bSOL, JupSOL, INF, bonkSOL

**Event Structure**:
```typescript
{
  type: 'price_update',
  source: 'pyth',
  token: priceData.id,
  price: priceData.price.price,
  confidence: priceData.price.conf,
  timestamp: priceData.price.publish_time,
  metadata: { expo: priceData.price.expo }
}
```

---

## Integration Guide

### Step 1: Install Dependencies

```powershell
cd ts
pnpm install
```

**Package**: `@repo/scanner-framework`

**Dependencies**:
- `@solana/kit` - Solana SDK (NOT @solana/web3.js)
- `nats` - NATS JetStream client
- `ioredis` - Redis client for deduplication
- `@repo/observability` - Metrics and logging
- `@repo/market-events` - Event type definitions

### Step 2: Create Scanner Class

**Example: Simple Price Monitor Scanner**

```typescript
import { PollingScanner, type PollingScannerConfig } from '@repo/scanner-framework';
import type { MarketEvent } from '@repo/market-events';

export class PriceMonitorScanner extends PollingScanner {
  name = 'price-monitor';
  pollInterval = 10000; // 10 seconds

  private apiClient: PriceAPI;

  constructor(config: PollingScannerConfig, apiUrl: string) {
    super(config);
    this.apiClient = new PriceAPI(apiUrl);
  }

  protected async fetch(): Promise<any> {
    return await this.apiClient.getPrices(['SOL', 'USDC']);
  }

  protected async process(data: any): Promise<MarketEvent[]> {
    const events: MarketEvent[] = [];

    for (const price of data.prices) {
      events.push({
        type: 'price_update',
        source: 'api',
        token: price.symbol,
        price: price.usd,
        confidence: 1.0,
        timestamp: Date.now(),
        slot: 0,
        metadata: {}
      });
    }

    return events;
  }

  protected getEventId(event: MarketEvent): string {
    return `${event.token}:${event.timestamp}`;
  }
}
```

### Step 3: Configure and Start Scanner

```typescript
import { ScannerType } from '@repo/scanner-framework';
import { PriceMonitorScanner } from './scanners/price-monitor';

const config = {
  name: 'price-monitor',
  enabled: true,
  type: ScannerType.Polling,
  natsServers: ['nats://localhost:4222'],
  redisUrl: 'redis://localhost:6379',
  deduplicationTtl: 5,
  maxRetries: 3,
  retryDelayMs: 1000,
  metricsEnabled: true,
  deduplicationEnabled: true,
  customConfig: {},
  pollInterval: 10000,
};

const scanner = new PriceMonitorScanner(config, 'https://api.prices.com');
await scanner.start();

// Graceful shutdown
process.on('SIGINT', async () => {
  await scanner.stop();
  process.exit(0);
});
```

### Step 4: Monitor Metrics

**Prometheus Metrics**:
```
http://localhost:9094/metrics
```

**Key Metrics to Watch**:
- `scanner_events_published_total{scanner="price-monitor"}`
- `scanner_errors_total{scanner="price-monitor"}`
- `scanner_poll_duration_seconds{scanner="price-monitor"}`

**Grafana Dashboard**:
- Access: http://localhost:3000/d/ts-scanner-service
- Panels: Events published, error rate, poll duration, queue size

---

## Monitoring & Observability

### Service Identity

**Service Name**: `ts-scanner-service`
**Container**: `ts-scanner-service`
**Metrics Port**: `9094`
**Health Endpoint**: `http://localhost:9094/health`

### LGTM+ Stack Integration

**Logs (Loki)**:
- All logs sent to Loki via Alloy
- Structured JSON logs with service name
- Query: `{service="ts-scanner-service"}`

**Metrics (Mimir + Prometheus)**:
- Scraped every 15 seconds
- Long-term storage in Mimir
- Custom metrics exported (see Framework Metrics section)

**Traces (Tempo)**:
- Distributed tracing for cross-service requests
- gRPC call traces
- NATS message flow traces

**Profiles (Pyroscope)** (Future):
- Continuous profiling
- CPU flame graphs
- Memory allocation tracking

### Grafana Dashboard

**Dashboard ID**: `ts-scanner-service`
**URL**: http://localhost:3000/d/ts-scanner-service

**21 Panels across 6 Rows**:

**Row 1: Overview**
1. Service Status (UP/DOWN)
2. Uptime (seconds)
3. NATS Connected
4. gRPC Connected

**Row 2: Quote Streaming**
5. Quotes Received Rate
6. Active Token Pairs
7. Quote Latency (p95, p99)

**Row 3: Arbitrage Opportunities**
8. Opportunities Detected
9. Opportunities Published
10. Average Profit (BPS)
11. Rejection Rate

**Row 4: Oracle Prices**
12. Token Prices Table
13. Price Update Rate
14. Price Staleness

**Row 5: Performance**
15. Memory Usage
16. CPU Usage
17. Event Processing Time (p95, p99)
18. Event Queue Size

**Row 6: Errors & Logs**
19. Error Rate
20. Errors by Level
21. Recent Logs (Loki stream)

### Alert Rules

**File**: `deployment/monitoring/prometheus/rules/ts-scanner-service-alerts.yml`

**Alert Categories**:
1. **Service Health**: Service down, high restart rate
2. **NATS Connection**: Disconnections, reconnect rate
3. **gRPC Connection**: Connection loss, high error rate, no quotes received
4. **Arbitrage Detection**: No opportunities, high rejection rate
5. **Performance**: Memory usage, CPU usage, slow processing, event backlog
6. **Oracle Prices**: Stale prices, high deviation
7. **Errors**: High error rate, critical errors

---

## Deployment Guide

### Option 1: Docker Deployment (Production)

#### Step 1: Build Scanner Service

```powershell
cd deployment\docker
docker-compose build ts-scanner-service
```

#### Step 2: Start Full Stack

```powershell
docker-compose up -d
```

**Services Started**:
- NATS JetStream (port 4222, 8222, 6222)
- Redis (port 6379)
- PostgreSQL (port 5432)
- Go Quote Service (port 8080, 50051)
- TypeScript Scanner Service (port 9094)
- Monitoring stack (Grafana, Prometheus, Loki, Tempo, Mimir, Alloy)

#### Step 3: Verify Services

```powershell
# Check scanner service health
curl http://localhost:9094/health

# Check scanner logs
docker logs ts-scanner-service -f

# Check metrics
curl http://localhost:9094/metrics
```

### Option 2: Local Development

#### Step 1: Start Infrastructure

```powershell
cd deployment\docker
docker-compose up -d nats redis postgres
```

#### Step 2: Start Go Quote Service

```powershell
cd go
.\run-quote-service-with-logging.ps1 -Port 8080 -GrpcPort 50051
```

#### Step 3: Start Scanner Service

```powershell
cd ts\apps\scanner-service
pnpm start:dev
```

**Expected Output**:
```
[Scanner] Starting scanner service...
[Scanner] Pyth client initialized
[Scanner] Subscribing to 14 pairs with 39 amounts
[gRPC] New subscription: <id> with 14 pairs, 39 amounts
[Scanner] Scanner service started successfully
```

### Environment Configuration

**File**: `ts/apps/scanner-service/.env`

```env
# Service Identity
NODE_ENV=production
SERVICE_NAME=ts-scanner-service
SERVICE_VERSION=0.1.0

# gRPC Client
QUOTE_SERVICE_GRPC_HOST=localhost
QUOTE_SERVICE_GRPC_PORT=50051
GRPC_KEEPALIVE_MS=10000
GRPC_RECONNECT_MS=5000

# Token Pairs
MONITOR_LST_PAIRS=true
MONITOR_STABLECOIN_PAIRS=true

# Arbitrage Thresholds
MIN_PROFIT_BPS=50
MAX_SLIPPAGE_BPS=50
MIN_CONFIDENCE=0.8

# NATS
NATS_SERVERS=nats://localhost:4222

# Redis
REDIS_URL=redis://localhost:6379
```

---

## Best Practices

### 1. Scanner Design

**DO**:
- ✅ Keep scanners focused on data acquisition
- ✅ Use handlers for complex business logic
- ✅ Implement proper cleanup in `cleanup()` method
- ✅ Add custom metrics to track scanner-specific stats
- ✅ Use meaningful event IDs for deduplication

**DON'T**:
- ❌ Put business logic in scanners
- ❌ Forget to call `super.cleanup()` when overriding
- ❌ Block the event loop with long-running operations
- ❌ Create scanners that are too generic

### 2. Event ID Generation

```typescript
// ✅ GOOD - Stable, unique IDs
protected getEventId(event: MarketEvent): string {
  return `${event.type}:${event.accountId}:${event.slot}`;
}

// ❌ BAD - Includes volatile data
protected getEventId(event: MarketEvent): string {
  return `${event.type}:${event.price}`; // Price changes!
}

// ✅ GOOD - Don't deduplicate when each event is unique
protected getEventId(event: MarketEvent): string {
  return `${event.type}:${event.timestamp}:${Math.random()}`;
}
```

### 3. Custom Statistics

```typescript
protected async process(data: any): Promise<MarketEvent[]> {
  // ... processing logic

  // Update custom stats
  this.stats.customStats.itemsProcessed =
    (this.stats.customStats.itemsProcessed || 0) + data.items.length;
  this.stats.customStats.lastProcessedTimestamp = Date.now();

  return events;
}
```

### 4. Error Handling

```typescript
protected async process(data: any): Promise<MarketEvent[]> {
  try {
    // ... processing logic
    return events;
  } catch (error) {
    this.logger.error({ scanner: this.name, err: error }, 'Processing error');

    // Update error stats
    this.stats.customStats.processingErrors =
      (this.stats.customStats.processingErrors || 0) + 1;

    // Return empty array - framework will handle retry
    return [];
  }
}
```

---

## Extensibility Patterns

### Pattern 1: Simple Polling Scanner

**Use Case**: Periodic API polling (price feeds, Jupiter quotes, Pyth data)

See [Scanner Implementations](#scanner-implementations) section for examples.

### Pattern 2: Real-Time Subscription Scanner

**Use Case**: Streaming data (Shredstream, gRPC, WebSocket)

See [ArbitrageQuoteScanner](#1-arbitrage-quote-scanner) for full implementation.

### Pattern 3: Composite Scanner (Multiple Data Sources)

**Use Case**: Scanner that combines multiple data sources

```typescript
export class CompositeScanner extends SubscriptionScanner {
  private sourceA: SourceAClient;
  private sourceB: SourceBClient;
  private updateQueue: any[] = [];

  protected async subscribe(): Promise<AsyncIterator<any>> {
    // Connect both sources
    await Promise.all([
      this.sourceA.connect(),
      this.sourceB.connect(),
    ]);

    // Set up event handlers for both sources
    this.sourceA.on('data', (data) => this.enqueueUpdate({ source: 'A', data }));
    this.sourceB.on('data', (data) => this.enqueueUpdate({ source: 'B', data }));

    // Return async iterator that yields from queue
    return this.createIterator();
  }

  protected async process(update: any): Promise<MarketEvent[]> {
    if (update.source === 'A') {
      return this.processSourceA(update.data);
    } else {
      return this.processSourceB(update.data);
    }
  }
}
```

### Pattern 4: Scanner with External Handler

**Use Case**: Scanner that delegates complex processing to handlers

See [ArbitrageQuoteScanner](#1-arbitrage-quote-scanner) which uses `QuoteStreamHandler`.

---

## Performance Optimization

### Current Performance

**Build Performance**:
- scanner-framework: 73.72 kB (5.6s build)
- scanner-service: 2025.28 kB (4.8s build)

**Runtime Metrics**:
- Polling scanner overhead: < 1ms per poll
- Subscription scanner latency: < 5ms per event
- Deduplication check: < 1ms (Redis)
- Event publishing: < 10ms (NATS)

### Optimization Roadmap (from 08-optimization-guide.md)

**Week 1: Local Quote Engine** (1.7s → 800ms)
- ✅ Deploy Go quote service with 30s cache
- ✅ Hybrid quoting (Go → Jupiter fallback)
- ⏳ Cache blockhash (50ms saved)
- ⏳ Parallel quote fetching (2x faster)

**Week 2: Shredstream Integration** (800ms → 500ms)
- ⏳ Rust Shredstream scanner (400ms early alpha)
- ⏳ Pre-compute quotes on slot notification
- ⏳ Pre-sign transactions
- ⏳ Batch RPC account fetching

**Week 3: Flash Loan Optimization** (500ms → 300ms)
- ⏳ Optimize Kamino flash loan integration
- ⏳ Transaction packing with ALT
- ⏳ Compute budget optimization

**Week 4: Jito Bundle Optimization** (300ms → 200ms)
- ⏳ Optimize bundle submission
- ⏳ Dynamic tip calculation
- ⏳ Multi-wallet parallel execution (5-10)

### Performance Tuning

**Concurrency Limits**:
```typescript
// Scanner layer
MAX_CONCURRENT_PROCESSING = 10 // Per subscription scanner
POLL_INTERVAL = 5000 // 5 seconds for polling scanners

// Infrastructure
MAX_NATS_BATCH_SIZE = 100 // Events per batch
REDIS_CONNECTION_POOL = 10 // Connections
RPC_ENDPOINTS = 7+ // Multiple providers
```

---

## Troubleshooting

### Scanner Not Showing in Grafana

**Problem**: Service shows as `unknown-service` or doesn't appear

**Solution**:
1. Check environment variable:
   ```powershell
   docker exec ts-scanner-service env | grep SERVICE_NAME
   # Should show: SERVICE_NAME=ts-scanner-service
   ```

2. Check logs:
   ```powershell
   docker logs ts-scanner-service | grep service
   # Should show: "service":"ts-scanner-service"
   ```

3. Restart service:
   ```powershell
   docker-compose restart ts-scanner-service
   ```

### Metrics Not Showing

**Problem**: Dashboard shows "No data"

**Solution**:
1. Check Prometheus scraping:
   ```
   http://localhost:9090/targets
   # Should show ts-scanner-service target as UP
   ```

2. Check metrics endpoint:
   ```powershell
   curl http://localhost:9094/metrics
   # Should return metrics in Prometheus format
   ```

3. Check Mimir:
   ```promql
   # In Grafana, query:
   service_info{service="ts-scanner-service"}
   ```

### gRPC Connection Errors

**Problem**: Scanner can't connect to Go quote-service

**Solution**:
1. Verify quote-service is running:
   ```powershell
   netstat -an | findstr 50051
   # Should show LISTENING
   ```

2. Test gRPC endpoint:
   ```powershell
   # Use grpcurl or similar tool
   grpcurl -plaintext localhost:50051 list
   ```

3. Check Docker networking:
   ```powershell
   # If using Docker, ensure host.docker.internal is configured
   docker exec ts-scanner-service ping host.docker.internal
   ```

### Build Errors

**Problem**: `scanner-framework` build fails

**Solution** (Fixed in 13.a-scanner-optimization-summary.md):
1. Check metrics import:
   ```typescript
   // ❌ WRONG
   import { incrementCounter, observeHistogram } from '@repo/observability';

   // ✅ CORRECT
   import { metrics } from '@repo/observability';
   metrics.incrementCounter('metric_name', labels);
   ```

2. Rebuild:
   ```powershell
   cd ts/packages/scanner-framework
   pnpm build
   ```

### High Memory Usage

**Problem**: Scanner service using > 500MB memory

**Solution**:
1. Check quote cache size:
   ```typescript
   // In QuoteStreamHandler
   console.log('Cache size:', this.quoteCache.size);
   ```

2. Reduce cache TTL:
   ```typescript
   private maxCacheAgeMs: number = 3000; // 3 seconds instead of 5
   ```

3. Monitor cleanup:
   ```typescript
   // Ensure cleanupStaleQuotes() is running
   setInterval(() => this.cleanupStaleQuotes(), 5000); // Every 5s
   ```

---

## References

### Documentation

- [11-extensible-strategy-architecture.md](./11-extensible-strategy-architecture.md) - Multi-strategy architecture
- [18-HFT_PIPELINE_ARCHITECTURE.md](./18-HFT_PIPELINE_ARCHITECTURE.md) - Overall pipeline
- [08-optimization-guide.md](./08-optimization-guide.md) - Performance optimization
- [Scanner Framework README](../ts/packages/scanner-framework/README.md) - Framework API docs

### Code Locations

**Framework**:
- `ts/packages/scanner-framework/` - Scanner framework package

**Scanners**:
- `ts/apps/scanner-service/src/scanners/arbitrage-quote-scanner.ts` - Arbitrage scanner
- `ts/apps/scanner-service/src/scanners/dca-scanner.ts` - DCA scanner

**Supporting Packages**:
- `ts/packages/market-events/` - Event type definitions
- `ts/packages/observability/` - Metrics and logging
- `ts/packages/strategy-framework/` - Strategy abstractions

**Infrastructure**:
- `go/cmd/quote-service/` - Go quote service (gRPC server)
- `deployment/docker/docker-compose.yml` - Docker orchestration
- `deployment/monitoring/` - Grafana dashboards and Prometheus rules

### Related Systems

- **Go Quote Service**: gRPC streaming, pool math, DEX routing
- **NATS JetStream**: 6-stream event architecture
- **Pyth Oracle**: Real-time price feeds
- **Strategy Framework**: TradingStrategy interface, HealthMonitor
- **FlatBuffers**: Zero-copy event serialization (see 19-FLATBUFFERS-MIGRATION.md)

---

**Implementation Status**: ✅ Production Ready

This consolidated guide replaces the following individual documents:
- ~~09-arbitrage-scanner-implementation.md~~ (arbitrage-specific implementation)
- ~~09-scanner-executor-framework-design.md~~ (framework design sections incorporated)
- ~~11-scanner-service-framework-integration.md~~ (integration guide incorporated)
- ~~11-scanner-service-monitoring.md~~ (monitoring section incorporated)
- ~~13.a-scanner-optimization-summary.md~~ (optimization summary incorporated)
- ~~13-scanner-framework-optimization.md~~ (optimization patterns incorporated)

**Next Steps**:
1. Archive old scanner documentation files
2. Implement additional scanners (limit orders, liquidation monitoring)
3. Migrate to Rust for production deployment (see Rust WORKSPACE.md)
4. Integrate Shredstream for sub-500μs account updates
