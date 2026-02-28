---
layout: single
title: "Scanner → Planner → Executor Architecture"
permalink: /docs/05-scanner-planner-executor-design/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Scanner → Planner → Executor Architecture

## Overview

This document provides detailed design patterns for the core Scanner → Planner → Executor architecture, with concrete examples and event schemas.

## Architecture Principles

### 1. Separation of Concerns
- **Scanners:** Observe and collect data (no decision-making)
- **Planners:** Analyze data and make decisions (no execution)
- **Executors:** Execute decisions (no strategy logic)

### 2. Event-Driven Communication
- All components communicate via NATS events
- Loose coupling enables independent scaling
- Event replay for debugging and analysis

### 3. Idempotency
- All operations designed to be safely retried
- Event deduplication where needed
- Transaction IDs for tracking

### 4. Observable
- All events logged and traced
- Metrics at every boundary
- Clear error propagation

## SCANNERS Layer

### Purpose
Continuously monitor blockchain state and external data sources, emitting normalized events when relevant changes occur.

### Design Patterns

#### Pattern 1: Polling Scanner
```typescript
abstract class PollingScanner {
  abstract pollInterval: number;
  abstract natsSubject: string;

  async start() {
    while (true) {
      try {
        const data = await this.fetch();
        const events = await this.process(data);

        for (const event of events) {
          await this.publish(event);
        }
      } catch (error) {
        this.logger.error({ error }, "Polling failed");
      }

      await sleep(this.pollInterval);
    }
  }

  abstract fetch(): Promise<any>;
  abstract process(data: any): Promise<Event[]>;

  async publish(event: Event) {
    await this.nats.publish(this.natsSubject, JSON.stringify(event));
    this.metrics.eventsPublished.inc();
  }
}
```

**Example: Price Feed Scanner**
```typescript
class PriceFeedScanner extends PollingScanner {
  pollInterval = 5000; // 5 seconds
  natsSubject = "market.prices.update";

  async fetch() {
    // Fetch prices from multiple DEXes concurrently
    const [raydiumPrices, meteoraPrices, jupiterPrices] = await Promise.all([
      this.fetchRaydiumPrices(),
      this.fetchMeteoraPrices(),
      this.fetchJupiterPrices(),
    ]);

    return { raydium: raydiumPrices, meteora: meteoraPrices, jupiter: jupiterPrices };
  }

  async process(data: any): Promise<PriceUpdateEvent[]> {
    const events: PriceUpdateEvent[] = [];

    for (const token of this.watchedTokens) {
      const prices = {
        raydium: data.raydium[token] ?? null,
        meteora: data.meteora[token] ?? null,
        jupiter: data.jupiter[token] ?? null,
      };

      // Calculate best bid/ask
      const validPrices = Object.values(prices).filter(p => p !== null);
      if (validPrices.length === 0) continue;

      const bestBid = Math.max(...validPrices);
      const bestAsk = Math.min(...validPrices);
      const spread = ((bestAsk - bestBid) / bestBid) * 100;

      events.push({
        type: "price_update",
        token,
        prices,
        bestBid,
        bestAsk,
        spread,
        timestamp: Date.now(),
      });
    }

    return events;
  }
}
```

#### Pattern 2: Subscription Scanner
```typescript
abstract class SubscriptionScanner {
  abstract natsSubject: string;

  async start() {
    const subscription = await this.subscribe();

    for await (const update of subscription) {
      try {
        const events = await this.process(update);

        for (const event of events) {
          // Deduplicate
          const isDuplicate = await this.checkDuplicate(event);
          if (isDuplicate) continue;

          await this.publish(event);
        }
      } catch (error) {
        this.logger.error({ error, update }, "Processing failed");
      }
    }
  }

  abstract subscribe(): Promise<AsyncIterable<any>>;
  abstract process(update: any): Promise<Event[]>;

  async checkDuplicate(event: Event): Promise<boolean> {
    const key = `dedup:${event.type}:${this.getEventId(event)}`;
    const exists = await this.redis.set(key, "1", "EX", 5, "NX");
    return exists === null; // null = key already existed
  }

  abstract getEventId(event: Event): string;
}
```

**Example: Account Change Scanner**
```typescript
class AccountChangeScanner extends SubscriptionScanner {
  natsSubject = "market.events.account_change";

  async subscribe() {
    // Subscribe to Shredstream or WebSocket
    return this.solana.onAccountChange(
      this.watchedAccounts,
      { commitment: "confirmed" }
    );
  }

  async process(update: AccountUpdate): Promise<AccountChangeEvent[]> {
    const account = update.accountId;
    const data = update.accountInfo.data;

    // Decode based on account type
    if (this.isRaydiumPool(account)) {
      const pool = decodeRaydiumPool(data);
      return [{
        type: "pool_update",
        protocol: "raydium",
        poolAddress: account,
        tokenA: pool.baseMint,
        tokenB: pool.quoteMint,
        reserveA: pool.baseReserve,
        reserveB: pool.quoteReserve,
        timestamp: Date.now(),
        slot: update.context.slot,
      }];
    }

    // Other account types...
    return [];
  }

  getEventId(event: AccountChangeEvent): string {
    return `${event.poolAddress}:${event.slot}`;
  }
}
```

### Scanner Event Schemas

#### PriceUpdateEvent
```typescript
interface PriceUpdateEvent {
  type: "price_update";
  token: Address;
  prices: {
    [dex: string]: number | null;
  };
  bestBid: number;
  bestAsk: number;
  spread: number; // percentage
  volume24h?: number;
  liquidity?: number;
  timestamp: number;
}

// NATS subject: market.prices.update
```

#### AccountChangeEvent
```typescript
interface AccountChangeEvent {
  type: "pool_update" | "large_swap" | "liquidity_change";
  protocol: string;
  poolAddress: Address;
  tokenA: Address;
  tokenB: Address;
  reserveA?: bigint;
  reserveB?: bigint;
  priceImpact?: number;
  timestamp: number;
  slot: number;
  data: Record<string, any>;
}

// NATS subject: market.events.account_change
```

#### VolumeUpdateEvent
```typescript
interface VolumeUpdateEvent {
  type: "volume_update";
  token: Address;
  volume24h: number;
  volumeChange: number; // percentage change from previous period
  avgTradeSize: number;
  tradeCount: number;
  timestamp: number;
}

// NATS subject: market.volume.update
```

---

## PLANNERS Layer

### Purpose
Subscribe to scanner events, analyze data, identify trading opportunities, and emit execution plans.

### Design Patterns

#### Pattern 1: Opportunity Detector
```typescript
abstract class OpportunityDetector {
  abstract strategy: string;
  abstract subscribeToSubjects: string[];
  abstract publishSubject: string;

  async start() {
    const nc = await this.nats.connect();
    const js = nc.jetstream();

    // Subscribe to multiple event types
    for (const subject of this.subscribeToSubjects) {
      const consumer = await js.consumers.get(subject, `${this.strategy}-planner`);
      const messages = await consumer.consume();

      this.processMessages(messages);
    }
  }

  async processMessages(messages: AsyncIterable<Msg>) {
    for await (const msg of messages) {
      try {
        const event = JSON.parse(msg.data.toString());
        const opportunities = await this.analyze(event);

        for (const opportunity of opportunities) {
          await this.publish(opportunity);
        }

        msg.ack();
      } catch (error) {
        this.logger.error({ error }, "Analysis failed");
        msg.nak();
      }
    }
  }

  abstract analyze(event: any): Promise<TradeOpportunity[]>;

  async publish(opportunity: TradeOpportunity) {
    await this.nats.publish(this.publishSubject, JSON.stringify(opportunity));
    this.metrics.opportunitiesDetected.inc({ strategy: this.strategy });
  }
}
```

**Example: Arbitrage Planner**
```typescript
class ArbitragePlanner extends OpportunityDetector {
  strategy = "arbitrage";
  subscribeToSubjects = ["market.prices.update"];
  publishSubject = "trade.opportunities.arbitrage";

  async analyze(event: PriceUpdateEvent): Promise<TradeOpportunity[]> {
    const opportunities: TradeOpportunity[] = [];

    // Find all tradeable pairs with this token
    const pairs = this.generateTradingPairs(event.token);

    for (const pair of pairs) {
      // Try different trade amounts
      for (const amount of this.testAmounts) {
        const opportunity = await this.checkArbitrage(
          pair.tokenA,
          pair.tokenB,
          amount
        );

        if (opportunity && opportunity.expectedProfit > this.minProfit) {
          opportunities.push(opportunity);
        }
      }
    }

    return opportunities;
  }

  async checkArbitrage(
    tokenA: Address,
    tokenB: Address,
    amount: bigint
  ): Promise<TradeOpportunity | null> {
    // Get quote: tokenA → tokenB
    const quote1 = await this.quoteService.getQuote({
      inputMint: tokenA,
      outputMint: tokenB,
      amount: amount.toString(),
    });

    if (!quote1) return null;

    // Get quote: tokenB → tokenA
    const quote2 = await this.quoteService.getQuote({
      inputMint: tokenB,
      outputMint: tokenA,
      amount: quote1.outputAmount,
    });

    if (!quote2) return null;

    // Calculate profit
    const outAmount = BigInt(quote2.outputAmount);
    const profit = outAmount - amount;

    // Subtract fees
    const flashLoanFee = amount * 5n / 10000n; // 0.05%
    const jitoTip = 5000n + BigInt(Math.floor(Math.random() * 1000));
    const netProfit = profit - flashLoanFee - jitoTip;

    if (netProfit <= 0n) return null;

    // Check if we've seen this route before (cache)
    const routeHash = this.hashRoute([quote1.routePlan, quote2.routePlan]);
    const cached = await this.redis.get(`route:${routeHash}`);

    return {
      id: uuid(),
      strategy: "arbitrage",
      tokenA,
      tokenB,
      amount: amount.toString(),
      expectedProfit: netProfit.toString(),
      priority: this.calculatePriority(netProfit, amount),
      routes: [
        {
          inputMint: tokenA,
          outputMint: tokenB,
          routePlan: quote1.routePlan,
          expectedOutput: quote1.outputAmount,
        },
        {
          inputMint: tokenB,
          outputMint: tokenA,
          routePlan: quote2.routePlan,
          expectedOutput: quote2.outputAmount,
        },
      ],
      useFlashLoan: true,
      urgency: "high",
      expiresAt: Date.now() + 5000, // 5 seconds
      metadata: {
        quote1Latency: quote1.latency,
        quote2Latency: quote2.latency,
        quote1Source: quote1.source,
        quote2Source: quote2.source,
      },
    };
  }

  calculatePriority(profit: bigint, amount: bigint): number {
    // Higher profit = higher priority
    // Larger amounts = lower priority (more risk)
    const profitPercent = Number(profit) / Number(amount);
    return Math.min(100, Math.floor(profitPercent * 10000));
  }
}
```

**Example: Grid Trading Planner**
```typescript
class GridTradingPlanner extends OpportunityDetector {
  strategy = "grid_trading";
  subscribeToSubjects = ["market.prices.update"];
  publishSubject = "trade.opportunities.grid";

  private gridLevels: Map<Address, GridLevel[]> = new Map();

  async analyze(event: PriceUpdateEvent): Promise<TradeOpportunity[]> {
    const token = event.token;
    const currentPrice = event.bestBid;

    // Get or initialize grid for this token
    let levels = this.gridLevels.get(token);
    if (!levels) {
      levels = this.initializeGrid(token, currentPrice);
      this.gridLevels.set(token, levels);
    }

    const opportunities: TradeOpportunity[] = [];

    for (const level of levels) {
      // Check if price crossed buy level
      if (currentPrice <= level.buyPrice && !level.buyFilled) {
        opportunities.push({
          id: uuid(),
          strategy: "grid_trading",
          tokenA: this.quoteCurrency, // USDC
          tokenB: token,
          amount: level.amount.toString(),
          expectedProfit: "0", // Grid trading profits accumulate over time
          priority: 50,
          routes: [{
            inputMint: this.quoteCurrency,
            outputMint: token,
            routePlan: [], // Will be filled by executor
            expectedOutput: (BigInt(level.amount) * BigInt(Math.floor(currentPrice * 1e9)) / 1000000000n).toString(),
          }],
          useFlashLoan: false,
          urgency: "low",
          expiresAt: Date.now() + level.ttl,
          metadata: {
            gridLevel: level.index,
            buyPrice: level.buyPrice,
            sellPrice: level.sellPrice,
          },
        });

        level.buyFilled = true;
        level.sellFilled = false; // Reset sell for next cycle
      }

      // Check if price crossed sell level
      if (currentPrice >= level.sellPrice && !level.sellFilled && level.buyFilled) {
        opportunities.push({
          id: uuid(),
          strategy: "grid_trading",
          tokenA: token,
          tokenB: this.quoteCurrency,
          amount: level.amount.toString(),
          expectedProfit: ((level.sellPrice - level.buyPrice) * level.amount).toString(),
          priority: 60,
          routes: [{
            inputMint: token,
            outputMint: this.quoteCurrency,
            routePlan: [],
            expectedOutput: (BigInt(level.amount) * BigInt(Math.floor(currentPrice * 1e9)) / 1000000000n).toString(),
          }],
          useFlashLoan: false,
          urgency: "low",
          expiresAt: Date.now() + level.ttl,
          metadata: {
            gridLevel: level.index,
            buyPrice: level.buyPrice,
            sellPrice: level.sellPrice,
            profit: (level.sellPrice - level.buyPrice) * level.amount,
          },
        });

        level.sellFilled = true;
        level.buyFilled = false; // Reset buy for next cycle
      }
    }

    return opportunities;
  }

  initializeGrid(token: Address, basePrice: number): GridLevel[] {
    const gridConfig = this.config.grids[token] || this.config.defaultGrid;
    const levels: GridLevel[] = [];

    for (let i = 0; i < gridConfig.levelCount; i++) {
      const offset = (i - gridConfig.levelCount / 2) * gridConfig.spacing;
      const buyPrice = basePrice * (1 + offset / 100);
      const sellPrice = basePrice * (1 + (offset + gridConfig.spacing) / 100);

      levels.push({
        index: i,
        buyPrice,
        sellPrice,
        amount: gridConfig.amountPerLevel,
        buyFilled: false,
        sellFilled: false,
        ttl: gridConfig.orderTTL || 12 * 60 * 60 * 1000, // 12 hours
      });
    }

    return levels;
  }
}
```

### Planner Event Schemas

#### TradeOpportunity
```typescript
interface TradeOpportunity {
  id: string; // UUID
  strategy: "arbitrage" | "grid_trading" | "dca" | "ai_signal";
  tokenA: Address;
  tokenB: Address;
  amount: string; // bigint as string
  expectedProfit: string; // bigint as string
  priority: number; // 0-100, higher = more urgent
  routes: RouteStep[];
  useFlashLoan: boolean;
  urgency: "low" | "medium" | "high";
  expiresAt: number; // timestamp
  metadata: Record<string, any>;
}

interface RouteStep {
  inputMint: Address;
  outputMint: Address;
  routePlan: any[]; // Jupiter route plan or empty
  expectedOutput: string;
}

// NATS subject: trade.opportunities.{strategy}
```

---

## EXECUTORS Layer

### Purpose
Subscribe to trade opportunities, select appropriate execution method, build and submit transactions, monitor confirmations.

### Design Patterns

#### Pattern 1: Transaction Coordinator
```typescript
class TransactionCoordinator {
  private semaphore: Semaphore;
  private executors: Map<string, Executor>;

  async start() {
    this.semaphore = new Semaphore(this.config.maxConcurrentTrades);

    const nc = await this.nats.connect();
    const js = nc.jetstream();

    // Subscribe to all opportunity types
    const consumer = await js.consumers.get("trade.opportunities", "executor-coordinator");
    const messages = await consumer.consume();

    for await (const msg of messages) {
      // Don't block on execution
      this.execute(msg).catch(error => {
        this.logger.error({ error }, "Execution failed");
      });
    }
  }

  async execute(msg: Msg) {
    try {
      const opportunity: TradeOpportunity = JSON.parse(msg.data.toString());

      // Check if expired
      if (Date.now() > opportunity.expiresAt) {
        this.logger.warn({ opportunityId: opportunity.id }, "Opportunity expired");
        msg.ack();
        return;
      }

      // Acquire semaphore (limit concurrency)
      await this.semaphore.acquire();

      try {
        // Select wallet
        const wallet = await this.walletManager.selectWallet({
          requiredAmount: BigInt(opportunity.amount),
          requiredToken: opportunity.tokenA,
          tier: "worker",
        });

        if (!wallet) {
          this.logger.warn({ opportunity }, "No wallet available");
          msg.nak();
          return;
        }

        // Select executor
        const executor = this.selectExecutor(opportunity);

        // Execute
        const result = await executor.execute({
          opportunity,
          wallet,
          timeout: 30000,
        });

        // Publish result
        await this.publishResult(result);

        msg.ack();
      } finally {
        this.semaphore.release();
      }
    } catch (error) {
      this.logger.error({ error }, "Execution error");
      msg.nak();
    }
  }

  selectExecutor(opportunity: TradeOpportunity): Executor {
    // High profit = use Jito for MEV protection
    if (Number(opportunity.expectedProfit) > 0.1 * LAMPORTS_PER_SOL) {
      return this.executors.get("jito")!;
    }

    // Time-sensitive strategies = use Jito
    if (opportunity.urgency === "high") {
      return this.executors.get("jito")!;
    }

    // Low urgency = use TPU (save on tips)
    return this.executors.get("tpu")!;
  }

  async publishResult(result: ExecutionResult) {
    await this.nats.publish("execution.results", JSON.stringify(result));
    this.metrics.tradesExecuted.inc({
      strategy: result.strategy,
      status: result.status,
    });
  }
}
```

#### Pattern 2: Jito Executor
```typescript
class JitoExecutor implements Executor {
  async execute(request: ExecutionRequest): Promise<ExecutionResult> {
    const startTime = Date.now();
    const span = this.tracer.startSpan("jito_execution");

    try {
      // Build transaction
      span.addEvent("building_transaction");
      const tx = await this.buildTransaction(request);

      // Sign
      span.addEvent("signing_transaction");
      const signedTx = await this.signTransaction(tx, request.wallet);

      // Submit to Jito
      span.addEvent("submitting_bundle");
      const bundleId = await this.submitBundle(signedTx);

      // Monitor confirmation
      span.addEvent("monitoring_confirmation");
      const confirmation = await this.monitorBundle(bundleId, request.timeout);

      // Parse actual profit from logs
      const actualProfit = await this.parseProfit(confirmation);

      span.setStatus({ code: SpanStatusCode.OK });

      return {
        id: request.opportunity.id,
        strategy: request.opportunity.strategy,
        status: "success",
        signature: confirmation.signature,
        bundleId,
        expectedProfit: request.opportunity.expectedProfit,
        actualProfit: actualProfit.toString(),
        latency: Date.now() - startTime,
        timestamp: Date.now(),
        metadata: {
          slot: confirmation.slot,
          executor: "jito",
        },
      };
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });

      return {
        id: request.opportunity.id,
        strategy: request.opportunity.strategy,
        status: "failed",
        error: error.message,
        latency: Date.now() - startTime,
        timestamp: Date.now(),
        metadata: {
          executor: "jito",
        },
      };
    } finally {
      span.end();
    }
  }

  async buildTransaction(request: ExecutionRequest): Promise<Transaction> {
    const instructions: TransactionInstruction[] = [];

    // Flash loan borrow (if needed)
    if (request.opportunity.useFlashLoan) {
      const flashBorrowIx = await this.buildFlashLoanBorrowIx(
        request.wallet.address,
        request.opportunity.tokenA,
        BigInt(request.opportunity.amount)
      );
      instructions.push(flashBorrowIx);
    }

    // Compute budget
    instructions.push(
      ComputeBudgetProgram.setComputeUnitPrice({
        microLamports: 1_000_000, // 1M micro-lamports = 1000 lamports
      })
    );

    instructions.push(
      ComputeBudgetProgram.setComputeUnitLimit({
        units: 400_000,
      })
    );

    // Swap instructions
    for (const route of request.opportunity.routes) {
      const swapIxs = await this.buildSwapInstructions(
        request.wallet.address,
        route
      );
      instructions.push(...swapIxs);
    }

    // Flash loan repay (if needed)
    if (request.opportunity.useFlashLoan) {
      const flashRepayIx = await this.buildFlashLoanRepayIx(
        request.wallet.address,
        request.opportunity.tokenA,
        BigInt(request.opportunity.amount)
      );
      instructions.push(flashRepayIx);
    }

    // Build transaction
    const tx = new Transaction();
    tx.add(...instructions);

    // Fetch recent blockhash
    const { blockhash } = await this.connection.getLatestBlockhash();
    tx.recentBlockhash = blockhash;
    tx.feePayer = request.wallet.publicKey;

    // Compress with ALT
    const compressedTx = await this.compressWithALT(tx);

    return compressedTx;
  }

  async submitBundle(tx: Transaction): Promise<string> {
    const tipAccount = await this.jitoClient.getRandomTipAccount();
    const tip = this.config.jitoTipBase + Math.floor(Math.random() * this.config.jitoTipStep);

    const bundleId = await this.jitoClient.sendBundle({
      transactions: [tx],
      skipPreflightValidation: false,
    });

    this.logger.info({ bundleId, tip }, "Bundle submitted");

    return bundleId;
  }

  async monitorBundle(bundleId: string, timeout: number): Promise<Confirmation> {
    const startTime = Date.now();

    while (Date.now() - startTime < timeout) {
      const status = await this.jitoClient.getBundleStatus(bundleId);

      if (status.confirmed) {
        return {
          signature: status.transactions[0],
          slot: status.slot,
          confirmed: true,
        };
      }

      await sleep(2000); // Poll every 2 seconds
    }

    throw new Error(`Bundle confirmation timeout after ${timeout}ms`);
  }
}
```

### Executor Event Schemas

#### ExecutionResult
```typescript
interface ExecutionResult {
  id: string; // Opportunity ID
  strategy: string;
  status: "success" | "failed" | "timeout";
  signature?: string;
  bundleId?: string;
  expectedProfit: string;
  actualProfit?: string;
  error?: string;
  latency: number;
  timestamp: number;
  metadata: Record<string, any>;
}

// NATS subject: execution.results
```

---

## Complete Data Flow Example

### Arbitrage Trade from Detection to Confirmation

```
┌─────────────────────────────────────────────────────────────┐
│ 1. SCANNER: Price Feed Scanner                              │
│    - Polls DEX prices every 5 seconds                       │
│    - Detects SOL price difference between Raydium/Meteora   │
│    - Publishes: market.prices.update                        │
└─────────────────────────────────────────────────────────────┘
                           ↓
{
  type: "price_update",
  token: "So11111111...",
  prices: {
    raydium: 137.5,
    meteora: 137.8
  },
  bestBid: 137.8,
  bestAsk: 137.5,
  spread: 0.22,
  timestamp: 1709876543210
}
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. PLANNER: Arbitrage Planner                               │
│    - Receives price update                                  │
│    - Calculates: buy SOL on Raydium, sell on Meteora       │
│    - Gets quotes (SolRoute: 3ms + 4ms)                      │
│    - Expected profit: 0.05 SOL (profitable!)               │
│    - Publishes: trade.opportunities.arbitrage               │
└─────────────────────────────────────────────────────────────┘
                           ↓
{
  id: "abc-123-def",
  strategy: "arbitrage",
  tokenA: "So111...",
  tokenB: "EPjFW...",
  amount: "1000000000",
  expectedProfit: "50000000",
  priority: 85,
  routes: [...],
  useFlashLoan: true,
  urgency: "high",
  expiresAt: 1709876548210
}
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. EXECUTOR: Transaction Coordinator                        │
│    - Receives opportunity                                   │
│    - Checks expiration (valid)                              │
│    - Selects Worker Wallet #7 (available)                  │
│    - Selects Jito Executor (high priority)                 │
│    - Routes to JitoExecutor                                 │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. EXECUTOR: Jito Executor                                  │
│    - Builds transaction with flash loan                     │
│    - Signs with Worker Wallet #7                            │
│    - Submits bundle to Jito (tip: 5,500 lamports)          │
│    - Monitors confirmation (polling every 2s)               │
│    - Confirmed in slot 246382819 (12s)                      │
│    - Parses actual profit: 0.048 SOL                        │
│    - Publishes: execution.results                           │
└─────────────────────────────────────────────────────────────┘
                           ↓
{
  id: "abc-123-def",
  strategy: "arbitrage",
  status: "success",
  signature: "5Kn3...",
  bundleId: "bundle-xyz",
  expectedProfit: "50000000",
  actualProfit: "48000000",
  latency: 12340,
  timestamp: 1709876555550,
  metadata: {
    slot: 246382819,
    executor: "jito"
  }
}
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. MONITORS: Metrics & Dashboards                           │
│    - Prometheus: arbitrage_profit_realized=0.048 SOL       │
│    - Grafana: Updates P&L chart                             │
│    - Jaeger: Traces full 12.3s latency                      │
│    - Loki: Logs "Arbitrage success, profit 0.048 SOL"      │
└─────────────────────────────────────────────────────────────┘
```

---

## Best Practices

### Event Naming
- Use hierarchical subjects: `{domain}.{entity}.{action}`
- Examples: `market.prices.update`, `trade.opportunities.arbitrage`, `execution.results`

### Event Versioning
```typescript
interface Event {
  version: string; // "v1", "v2", etc.
  type: string;
  data: any;
}
```

### Error Handling
- Scanners: Log and continue (don't crash on single error)
- Planners: NAK message for retry (up to 3 times)
- Executors: DLQ (dead letter queue) after 3 failures

### Idempotency
- Use UUIDs for opportunity IDs
- Check Redis for duplicate processing
- Use transaction signatures for deduplication

### Timeouts
- Scanner publishing: 5s
- Planner analysis: 10s
- Executor building: 5s
- Executor submission: 30s

### Metrics
- Every event published/received
- Every error
- Latency at each stage
- Success rates

---

## Testing

### Unit Tests
```typescript
describe("ArbitragePlanner", () => {
  it("should detect profitable arbitrage", async () => {
    const planner = new ArbitragePlanner(mockConfig);
    const priceUpdate: PriceUpdateEvent = {
      type: "price_update",
      token: SOL_MINT,
      prices: { raydium: 137.5, meteora: 138.0 },
      bestBid: 138.0,
      bestAsk: 137.5,
      spread: 0.36,
      timestamp: Date.now(),
    };

    const opportunities = await planner.analyze(priceUpdate);

    expect(opportunities).toHaveLength(1);
    expect(opportunities[0].expectedProfit).toBeGreaterThan("0");
  });
});
```

### Integration Tests
```typescript
describe("Scanner → Planner Integration", () => {
  it("should flow from scanner to planner", async () => {
    const nats = await connect({ servers: TEST_NATS_URL });
    const js = nats.jetstream();

    const scanner = new PriceFeedScanner(config);
    const planner = new ArbitragePlanner(config);

    // Start planner
    planner.start();

    // Emit price update from scanner
    await scanner.publish({
      type: "price_update",
      token: SOL_MINT,
      prices: { raydium: 137.5, meteora: 138.0 },
      bestBid: 138.0,
      bestAsk: 137.5,
      spread: 0.36,
      timestamp: Date.now(),
    });

    // Wait for opportunity
    const opportunity = await waitForEvent(
      js,
      "trade.opportunities.arbitrage",
      5000
    );

    expect(opportunity).toBeDefined();
    expect(opportunity.strategy).toBe("arbitrage");
  });
});
```

### End-to-End Tests
```typescript
describe("Full Trading Loop", () => {
  it("should execute profitable arbitrage", async () => {
    // Setup: Start all components
    const scanner = new PriceFeedScanner(config);
    const planner = new ArbitragePlanner(config);
    const coordinator = new TransactionCoordinator(config);

    await Promise.all([
      scanner.start(),
      planner.start(),
      coordinator.start(),
    ]);

    // Trigger: Emit price update
    await scanner.publish(mockPriceUpdate);

    // Assert: Wait for execution result
    const result = await waitForEvent(
      nats.jetstream(),
      "execution.results",
      30000
    );

    expect(result.status).toBe("success");
    expect(result.actualProfit).toBeGreaterThan("0");
  }, 60000); // 60s timeout
});
```

---

This architecture provides a scalable, maintainable, and observable system for automated trading on Solana.
