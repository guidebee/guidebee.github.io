---
layout: single
title: "Production System Architecture Plan"
permalink: /docs/02-production-architecture-plan/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Production System Architecture Plan

## Overview

This document outlines the architecture for a production-grade Solana trading system based on the scanner → planner → executor pattern, incorporating lessons learned from both prototype systems.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         MONITORS                                 │
│  (Prometheus, Grafana, Jaeger, Loki, Custom Metrics)            │
└─────────────────────────────────────────────────────────────────┘
                              ↓ observability
┌─────────────────────────────────────────────────────────────────┐
│                        PREPARERS                                 │
│  (Wallet Management, Balance Sync, Market Data Cache)           │
└─────────────────────────────────────────────────────────────────┘
                              ↓ initialization
┌──────────────┐      ┌──────────────┐      ┌──────────────────┐
│   SCANNERS   │ ───→ │   PLANNERS   │ ───→ │    EXECUTORS    │
│              │      │              │      │                  │
│ - Market     │      │ - Arbitrage  │      │ - Jito Bundles  │
│   Scanners   │      │ - Grid Trade │      │ - TPU Direct    │
│ - Account    │      │ - DCA        │      │ - Transaction   │
│   Watchers   │      │ - AI Analysis│      │   Confirmation  │
│ - Price Feed │      │ - Quote      │      │ - Error Handler │
│ - Volume     │      │   Optimizer  │      │                  │
│   Monitor    │      │              │      │                  │
└──────────────┘      └──────────────┘      └──────────────────┘
       ↓                     ↓                       ↓
┌─────────────────────────────────────────────────────────────────┐
│                    EVENT BUS (NATS JetStream)                    │
│  Topics: market.events, trade.opportunities, execution.orders   │
└─────────────────────────────────────────────────────────────────┘
       ↓                     ↓                       ↓
┌─────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                               │
│  Redis (hot data) | PostgreSQL (persistent) | TimescaleDB (time)│
└─────────────────────────────────────────────────────────────────┘
```

## System Components

### 1. SCANNERS (Data Acquisition Layer)

**Purpose:** Monitor blockchain state and market conditions to detect trading opportunities.

#### 1.1 Market Event Scanner (TypeScript)
**Technology:** TypeScript + Solana Web3.js
**Input:** Blockchain events via Shredstream or WebSocket
**Output:** NATS events to `market.events.*`

**Responsibilities:**
- Subscribe to account changes (filtered by relevant addresses)
- Monitor DEX pool state changes (Raydium, Meteora, etc.)
- Detect large transactions and unusual volume
- Track token price movements
- Emit normalized events to event bus

**Key Features:**
- Multiple subscription modes (Shredstream primary, WebSocket backup)
- Client-side filtering to reduce processing overhead (99% reduction)
- Event deduplication via Redis
- Latency tracking (target: 100-200ms)

**Configuration:**
```typescript
{
  subscriptionMode: "shredstream" | "websocket" | "rpc-polling",
  accountsToWatch: Address[],
  eventBufferSize: 1000,
  dedupWindow: 5000 // ms
}
```

#### 1.2 Price Feed Scanner (Go)
**Technology:** Go + concurrent workers
**Input:** DEX pool state, oracle feeds
**Output:** NATS events to `market.prices.*`

**Responsibilities:**
- Real-time price aggregation across multiple DEXes
- Oracle price fetching (Pyth, Switchboard)
- Spread calculation between venues
- Price impact estimation for various amounts

**Key Features:**
- Concurrent pool queries (goroutines per protocol)
- Sub-10ms response time for cached prices
- 5-minute TTL for price cache
- Automatic failover to backup data sources

#### 1.3 Volume Monitor (TypeScript)
**Technology:** TypeScript + database queries
**Input:** Historical transaction data
**Output:** NATS events to `market.volume.*`

**Responsibilities:**
- Track 24h trading volume per token
- Identify volume spikes (potential opportunities)
- Calculate average trade size
- Monitor liquidity changes

**Key Features:**
- 12-hour cache in Redis
- Background refresh every hour
- Anomaly detection for volume spikes

#### 1.4 Wallet Balance Scanner (TypeScript)
**Technology:** TypeScript + RPC batch queries
**Input:** Configured wallet addresses
**Output:** Internal state + alerts

**Responsibilities:**
- Monitor all managed wallet balances
- Compare expected vs. actual balances
- Trigger rebalancing when thresholds exceeded
- Alert on unexpected balance changes

**Key Features:**
- Batch RPC calls to minimize requests
- Expected balance validation
- Automatic rebalancing triggers

### 2. PLANNERS (Strategy & Decision Layer)

**Purpose:** Analyze scanner data to identify profitable trades and create execution plans.

#### 2.1 Arbitrage Planner (TypeScript)
**Technology:** TypeScript + business logic
**Input:** NATS `market.prices.*`, `market.events.*`
**Output:** NATS `trade.opportunities.arbitrage`

**Responsibilities:**
- Receive price update events from scanners
- Calculate profit potential: `outAmount - inAmount - fees - tips`
- Verify profitability threshold (configurable per wallet tier)
- Generate swap route plan with flash loan wrapping
- Emit trade opportunity if profitable

**Strategy Logic:**
```typescript
for each (tokenA, tokenB) pair:
  quoteSell = getQuote(tokenA → tokenB, amount)
  quoteBuy = getQuote(tokenB → tokenA, quoteSell.outAmount)

  profit = quoteBuy.outAmount - amount - flashLoanFee - jitoTip

  if profit > threshold:
    emit TradeOpportunity {
      strategy: "arbitrage",
      route: [sellRoute, buyRoute],
      expectedProfit: profit,
      priority: calculatePriority(profit, latency)
    }
```

**Key Features:**
- Hybrid quoting (SolRoute primary, Jupiter fallback)
- Rate limiting (60 Jupiter API calls/min)
- Route template caching with hash-based deduplication
- Dynamic profit threshold adjustment

#### 2.2 Grid Trading Planner (TypeScript)
**Technology:** TypeScript + order book management
**Input:** NATS `market.prices.*`
**Output:** NATS `trade.opportunities.grid`

**Responsibilities:**
- Maintain grid order book (buy/sell levels)
- Monitor price movements against grid
- Trigger buy orders when price drops to grid level
- Trigger sell orders when price rises to grid level
- Calculate P&L and rebalance grid

**Strategy Logic:**
```typescript
gridLevels = generateGridLevels(basePrice, gridCount, spacing)

for each priceUpdate:
  for each gridLevel:
    if currentPrice <= gridLevel.buyPrice && !gridLevel.buyFilled:
      emit BuyOrder(gridLevel)

    if currentPrice >= gridLevel.sellPrice && !gridLevel.sellFilled:
      emit SellOrder(gridLevel)
```

**Key Features:**
- Configurable grid spacing (percentage or fixed)
- Order TTL management (default 12 hours)
- Automatic grid rebalancing on price moves
- Split orders for large sizes

#### 2.3 DCA Planner (TypeScript)
**Technology:** TypeScript + time-based triggers
**Input:** Time intervals + `market.prices.*`
**Output:** NATS `trade.opportunities.dca`

**Responsibilities:**
- Schedule recurring buy/sell orders
- Average entry price calculation
- Position size management
- Stop-loss/take-profit monitoring

**Strategy Logic:**
```typescript
every interval (e.g., 1 hour):
  if shouldExecute(token, currentPrice, constraints):
    emit BuyOrder({
      amount: calculateDCAAmount(position, budget),
      maxPriceImpact: 0.5%,
      urgency: "low"
    })
```

**Key Features:**
- Configurable intervals (minutes to days)
- Price limit orders (buy only below X)
- Position size limits
- Automatic position tracking

#### 2.4 AI Analysis Planner (TypeScript)
**Technology:** TypeScript + OpenAI API
**Input:** Chart data, technical indicators
**Output:** NATS `trade.opportunities.ai`

**Responsibilities:**
- Generate TradingView chart screenshots
- Send to ChatGPT for analysis
- Parse AI recommendations
- Convert to actionable trade signals
- Weight recommendations with other signals

**Strategy Logic:**
```typescript
periodic or on-demand:
  chartUrl = generateTradingViewChart(token, timeframe)
  analysis = await chatGPT.analyzeChart(chartUrl, prompt)

  if analysis.recommendation == "BUY":
    emit TradeSignal({
      direction: "long",
      confidence: analysis.confidence,
      reasoning: analysis.explanation
    })
```

**Key Features:**
- Queue-based async processing
- Multi-language support
- Follow-up analysis
- Confidence scoring

#### 2.5 Quote Optimizer (Go)
**Technology:** Go service (high-performance)
**Input:** RPC requests from planners
**Output:** Optimized quotes with route details

**Responsibilities:**
- Interface between planners and quoting services
- Try SolRoute service first (2-10ms)
- Fallback to Jupiter API (100-300ms)
- Cache quotes in Redis (5-minute TTL)
- Health monitoring and automatic failover

**Key Features:**
- Concurrent quote requests across multiple DEXes
- Best route selection by output amount
- Route template generation for caching
- Performance metrics tracking

### 3. EXECUTORS (Transaction Execution Layer)

**Purpose:** Execute planned trades efficiently and reliably.

#### 3.1 Jito Bundle Executor (TypeScript)
**Technology:** TypeScript + Jito SDK
**Input:** NATS `trade.opportunities.*` (high priority)
**Output:** Transaction signatures + confirmation status

**Responsibilities:**
- Subscribe to high-priority trade opportunities
- Build transaction with instructions
- Add compute budget and priority fees
- Get Jito tip account
- Submit bundle with tip
- Monitor bundle status
- Emit execution results

**Transaction Assembly:**
```typescript
1. flashLoanBorrow (if needed)
2. setComputeUnitPrice (priority fee)
3. setComputeUnitLimit (compute budget)
4. swapInstruction(s) (from route plan)
5. flashLoanRepay (if needed)
6. Compress with Address Lookup Tables
7. Sign with appropriate wallet
8. Submit to Jito with tip
```

**Key Features:**
- Bundle composition with multiple transactions
- Dynamic tip calculation based on competition
- UUID-based bundle tracking
- Retry logic with exponential backoff
- Confirmation polling (max 30s)

#### 3.2 TPU Direct Executor (TypeScript)
**Technology:** TypeScript + Solana Web3.js
**Input:** NATS `trade.opportunities.*` (medium priority)
**Output:** Transaction signatures + confirmation status

**Responsibilities:**
- Alternative to Jito for non-MEV-critical trades
- Direct transaction submission to TPU
- Leader schedule awareness
- Confirmation monitoring

**Key Features:**
- Faster submission (no bundle overhead)
- Lower cost (no Jito tips)
- Suitable for non-competitive trades
- RPC failover on errors

#### 3.3 Transaction Coordinator (TypeScript)
**Technology:** TypeScript + state management
**Input:** All execution requests
**Output:** Routing decisions + execution tracking

**Responsibilities:**
- Select appropriate executor (Jito vs TPU vs Solayer)
- Manage execution queue
- Handle concurrent execution limits
- Track pending transactions
- Coordinate retries on failures
- Emit metrics and logs

**Routing Logic:**
```typescript
function selectExecutor(opportunity: TradeOpportunity): Executor {
  if (opportunity.expectedProfit > HIGH_PROFIT_THRESHOLD) {
    return jitoExecutor; // MEV protection
  }

  if (opportunity.strategy === "arbitrage") {
    return jitoExecutor; // Time-sensitive
  }

  if (opportunity.urgency === "low") {
    return tpuExecutor; // Save on tips
  }

  return jitoExecutor; // Default
}
```

**Key Features:**
- Priority-based routing
- Concurrent execution limits (configurable)
- Dead letter queue for failed transactions
- Execution metrics (success rate, latency)

#### 3.4 Confirmation Monitor (TypeScript)
**Technology:** TypeScript + RPC polling
**Input:** Pending transaction signatures
**Output:** NATS `execution.confirmed` or `execution.failed`

**Responsibilities:**
- Poll transaction status (getSignatureStatuses)
- Parse transaction logs for actual amounts
- Verify expected vs. actual profit
- Emit confirmation events
- Handle timeouts and resubmissions

**Key Features:**
- Batch status polling (up to 100 sigs)
- Exponential backoff on polling
- 30-second timeout (configurable)
- Event decoding for profit verification

### 4. PREPARERS (Initialization & Management)

**Purpose:** Setup and maintain system state before trading begins.

#### 4.1 Wallet Manager (TypeScript)
**Technology:** TypeScript + keypair management
**Input:** Configuration, treasure wallet
**Output:** Initialized wallets with balances

**Responsibilities:**
- Initialize wallet tiers (Proxy, Worker, Controller)
- Load private keys securely from secrets manager
- Create associated token accounts (ATAs)
- Initial balance distribution from treasure wallet
- Mask transfers for anonymity

**Wallet Tiers:**
- **Treasure Wallet:** Centralized funding source (hot wallet)
- **Controller Wallets:** Management operations (3-5 wallets)
- **Proxy Wallets:** External-facing for anonymity (10-20 wallets)
- **Worker Wallets:** Actual trading execution (20-50 wallets)

**Key Features:**
- Expected balance tracking in Redis
- Automatic rebalancing triggers
- Multi-hop transfers for masking
- ATA creation for all required tokens

#### 4.2 Market Data Initializer (Go)
**Technology:** Go service
**Input:** RPC endpoint, protocol configs
**Output:** Cached market data in Redis

**Responsibilities:**
- Fetch all relevant DEX pools (Raydium, Meteora, etc.)
- Load current pool reserves and prices
- Cache in Redis with 5-minute TTL
- Initialize Kamino lending markets
- Load Jupiter route templates from history

**Key Features:**
- Concurrent pool fetching (goroutines)
- Batch RPC calls for efficiency
- Warm cache before trading starts
- Health check on completion

#### 4.3 Config Validator (TypeScript)
**Technology:** TypeScript + Zod schemas
**Input:** Environment variables, config files
**Output:** Validated configuration or errors

**Responsibilities:**
- Validate all environment variables
- Check RPC endpoint connectivity
- Verify wallet keypairs are valid
- Test Redis/PostgreSQL connections
- Validate strategy parameters
- Generate default configs if missing

**Key Features:**
- Schema-based validation (Zod)
- Detailed error messages
- Connection testing
- Config file generation

#### 4.4 Historical Data Loader (TypeScript)
**Technology:** TypeScript + PostgreSQL
**Input:** Database connection
**Output:** Loaded historical data in memory/cache

**Responsibilities:**
- Load past trade history for analysis
- Cache profitable route templates
- Initialize strategy state from last run
- Load wallet balance history

**Key Features:**
- Efficient batch loading
- Selective caching (hot data only)
- State recovery on restart

### 5. MONITORS (Observability & Alerting)

**Purpose:** Monitor all subsystems, track performance, and alert on issues.

#### 5.1 Metrics Collector (Prometheus)
**Technology:** Prometheus + exporters
**Input:** Metrics from all services
**Output:** Time-series metrics database

**Responsibilities:**
- Scrape metrics endpoints from all services
- Store time-series data
- Provide query interface for Grafana

**Key Metrics:**
- Scanner: event rate, latency, error rate
- Planner: opportunities detected, profit potential, strategy distribution
- Executor: transaction success rate, confirmation time, profit realized
- System: CPU, memory, network, RPC calls

#### 5.2 Distributed Tracing (Jaeger)
**Technology:** Jaeger + OpenTelemetry
**Input:** Traces from all services
**Output:** Distributed trace visualization

**Responsibilities:**
- Collect traces from all components
- Visualize request flows across services
- Identify bottlenecks and errors
- Track latency breakdown

**Key Traces:**
- Market event → opportunity detection → execution → confirmation
- Quote request → SolRoute/Jupiter → response
- Transaction building → signing → submission → confirmation

#### 5.3 Log Aggregation (Loki)
**Technology:** Loki + Promtail
**Input:** Logs from all services
**Output:** Centralized log storage + queries

**Responsibilities:**
- Collect logs from all services
- Index and store efficiently
- Provide query interface
- Integrate with Grafana

**Log Levels:**
- ERROR: Critical failures requiring immediate attention
- WARN: Recoverable issues (RPC errors, quote failures)
- INFO: Normal operations (trades executed, balances updated)
- DEBUG: Detailed troubleshooting (quote details, route plans)

#### 5.4 Dashboard (Grafana)
**Technology:** Grafana
**Input:** Prometheus, Loki, Jaeger
**Output:** Unified dashboards

**Responsibilities:**
- Real-time system health dashboard
- Trading performance dashboard (P&L, success rate, volume)
- Strategy-specific dashboards (arbitrage, grid, DCA)
- Alert visualization
- Historical analysis

**Dashboards:**
- System Overview (all services health)
- Trading Performance (P&L, ROI, success rate)
- Strategy Analytics (per-strategy metrics)
- Wallet Management (balances, rebalancing events)
- RPC Health (latency, error rates, endpoint status)

#### 5.5 Alert Manager (Prometheus Alertmanager)
**Technology:** Alertmanager + notification channels
**Input:** Alert rules from Prometheus
**Output:** Notifications (Slack, PagerDuty, email)

**Responsibilities:**
- Evaluate alert rules
- Route alerts to appropriate channels
- Group and deduplicate alerts
- Escalation policies

**Alert Categories:**
- Critical: System down, wallet balance low, executor failures
- Warning: High error rates, slow confirmations, cache misses
- Info: Strategy completed, rebalancing triggered

## Data Flow Example: Arbitrage Trade

```
1. Market Event Scanner
   - Detects SOL/USDC pool update on Raydium
   - Emits: NATS market.events.raydium.pool_update

2. Arbitrage Planner
   - Receives pool update event
   - Gets quote: SOL → USDC (SolRoute: 2ms)
   - Gets quote: USDC → SOL (SolRoute: 2ms)
   - Calculates profit: 0.05 SOL (profitable!)
   - Emits: NATS trade.opportunities.arbitrage

3. Transaction Coordinator
   - Receives arbitrage opportunity
   - Selects Worker Wallet #5 (available)
   - Selects Jito Executor (high priority)
   - Routes to executor

4. Jito Bundle Executor
   - Builds transaction:
     [flashBorrow, setComputeBudget, swap1, swap2, flashRepay]
   - Signs with Worker Wallet #5
   - Submits bundle to Jito (UUID: abc-123)
   - Emits: NATS execution.submitted

5. Confirmation Monitor
   - Polls bundle status every 2s
   - Bundle confirmed in slot 246382819 (12s)
   - Parses logs: actual profit = 0.048 SOL
   - Emits: NATS execution.confirmed

6. Monitors
   - Metrics: arbitrage_profit_realized=0.048 SOL
   - Logs: "Arbitrage trade confirmed, profit 0.048 SOL"
   - Dashboard: Update P&L chart

7. Wallet Manager
   - Updates Worker Wallet #5 expected balance
   - Checks if rebalancing needed (no)
```

## Technology Stack

### Core Services

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Scanners** | TypeScript | Rich ecosystem, Web3.js integration |
| **Planners** | TypeScript | Business logic flexibility, fast iteration |
| **Executors** | TypeScript | Transaction signing, Solana SDK |
| **Quote Service** | Go | High performance, concurrency, 2-10ms latency |
| **RPC Proxy** | Rust | Maximum performance, connection pooling |
| **Transaction Builder** | Rust | Zero-copy serialization, speed |

### Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Event Bus** | NATS JetStream | Pub/sub, persistence, replay |
| **Hot Cache** | Redis | Sub-ms access, pub/sub |
| **Persistent DB** | PostgreSQL | ACID transactions, relational data |
| **Time-Series** | TimescaleDB | Historical metrics, optimized queries |
| **Secrets** | Vault / AWS Secrets | Secure key management |
| **Container Runtime** | Docker + Kubernetes | Orchestration, scaling |
| **Observability** | Prometheus, Grafana, Jaeger, Loki | Metrics, traces, logs |

### Communication Patterns

```
Service-to-Service: NATS pub/sub (async, decoupled)
Client-to-Service: REST API / gRPC (sync, request-response)
Cache: Redis (read-through pattern)
State: PostgreSQL (source of truth)
Events: NATS JetStream (persistent streams)
```

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer (Nginx)                 │
└─────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                       │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  Scanners   │  │  Planners   │  │  Executors  │     │
│  │  (3 pods)   │  │  (5 pods)   │  │  (3 pods)   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Quote Svc   │  │  RPC Proxy  │  │  Preparers  │     │
│  │  (Go-2pods) │  │  (Rust-3)   │  │  (2 pods)   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│                   Managed Services                        │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │   NATS   │  │  Redis   │  │PostgreSQL│  │ Grafana │ │
│  │JetStream │  │ Cluster  │  │ Primary  │  │  Cloud  │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
└──────────────────────────────────────────────────────────┘
```

### Scaling Strategy

**Scanners:** Scale horizontally, partition by account ranges
**Planners:** Scale horizontally, subscribe to all events
**Executors:** Scale horizontally, coordinate via NATS
**Quote Service:** Scale horizontally, cache in Redis
**RPC Proxy:** Scale horizontally, round-robin load balancing

## Next Steps

See `03-implementation-roadmap.md` for the phased implementation plan.
