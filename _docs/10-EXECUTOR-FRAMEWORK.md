---
layout: single
title: "Executor Framework Guide - Transaction Execution Layer"
permalink: /docs/10-executor-framework/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Executor Framework Guide - Transaction Execution Layer

**Document Version**: 2.0 (Consolidated)
**Date**: 2025-12-20
**Status**: Framework Ready, Rust Migration Planned
**Author**: Solution Architect

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Executor Role in HFT Pipeline](#executor-role-in-hft-pipeline)
3. [TypeScript Executor Framework](#typescript-executor-framework)
4. [Transaction Building](#transaction-building)
5. [Execution Methods](#execution-methods)
6. [Wallet Management](#wallet-management)
7. [Rust Migration Strategy](#rust-migration-strategy)
8. [Integration with Pipeline](#integration-with-pipeline)
9. [Performance Optimization](#performance-optimization)
10. [Best Practices](#best-practices)
11. [Deployment Guide](#deployment-guide)

---

## Executive Summary

### Executor's Responsibility

The Executor is the **final stage** in the HFT trading pipeline, responsible for **dumb submission** of pre-validated transactions. It receives execution plans from the Planner (via NATS PLANNED stream) and submits them to Solana as quickly as possible.

**Key Principle**: Executor trusts the Planner's validation and does **NO** additional simulation or profitability checks. Its job is to **sign and send** as fast as possible.

### Performance Targets

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| **Transaction Building** | < 50ms | Framework ready | ✅ |
| **Wallet Selection** | < 5ms | ✅ Implemented | ✅ |
| **Jito Bundle Submission** | < 200ms | SDK integration pending | ⏳ |
| **TPU Submission** | < 100ms | Not implemented | ❌ |
| **Total Execution** | < 500ms | Framework ready | 🎯 Goal |

### Technology Stack

**Current (TypeScript)**:
- Package: `@repo/executor-framework`
- Base: BaseExecutor abstract class
- Implementations: JitoExecutor (core), TPUExecutor (future), RPCExecutor (future), FlashLoanExecutor (future)
- Dependencies: `@solana/kit`, `jito-ts`, `@kamino-finance/klend-sdk`

**Future (Rust)**:
- **30-40% faster execution** (525-925ms vs 750-1,500ms TypeScript)
- Shredstream integration for 400ms early alpha
- Zero-copy transaction building
- Parallel execution with multiple wallets

---

## Executor Role in HFT Pipeline

### Pipeline Architecture

From [18-HFT_PIPELINE_ARCHITECTURE.md](./18-HFT_PIPELINE_ARCHITECTURE.md):

```
┌─────────────────────────────────────────────────────────────┐
│ STAGE 0: QUOTE-SERVICE (Go)                                │
│ • Generates quotes, caches, streams via gRPC               │
│ • Latency: <10ms per quote                                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STAGE 1: SCANNER (Fast Pre-Filter)                         │
│ • Detects arbitrage patterns                                │
│ • No RPC calls, simple math only                           │
│ • Publishes: OPPORTUNITIES stream                          │
│ • Latency: <10ms                                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STAGE 2: PLANNER (Deep Validation)                         │
│ • RPC simulation, profitability validation                 │
│ • Chooses execution method (Jito/TPU/RPC)                  │
│ • Publishes: PLANNED stream                              │
│ • Latency: 50-100ms                                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ STAGE 3: EXECUTOR (Dumb Submission) ← YOU ARE HERE         │
│ • Build transaction from plan                               │
│ • Sign with wallet                                          │
│ • Submit via Jito/TPU/RPC                                   │
│ • NO validation, NO simulation                              │
│ • Publishes: EXECUTED stream                                │
│ • Latency: <20ms                                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
                   NATS: executed.completed.*
                            ↓
              PnL Service, Monitor, Analytics
```

### Separation of Concerns

**Why Executor Does NOT Validate**:

| Stage | Responsibility | Why |
|-------|---------------|-----|
| **Scanner** | Fast pre-filter | Eliminate 98% of noise (1-5ms) |
| **Planner** | Deep validation | RPC simulation is expensive (50-100ms) |
| **Executor** | Dumb submission | Speed is critical, trust Planner's work |

**Benefits**:
- ✅ **Speed**: Executor spends <20ms on execution, not 50-100ms on validation
- ✅ **Simplicity**: Executor logic is straightforward (build → sign → submit)
- ✅ **Scalability**: Can run multiple executor instances without RPC bottlenecks
- ✅ **Reliability**: Planner handles complex logic once, Executor just executes

### Executor Responsibilities

**MUST DO**:
- ✅ Build transaction from ExecutionPlanEvent
- ✅ Sign with selected wallet
- ✅ Submit via specified method (Jito/TPU/RPC)
- ✅ Wait for confirmation
- ✅ Publish ExecutionResultEvent to EXECUTED stream
- ✅ Handle SYSTEM stream events (kill switch, shutdown)

**MUST NOT DO**:
- ❌ Simulate transaction (Planner already did this)
- ❌ Validate profitability (Planner already did this)
- ❌ Re-fetch quotes (Planner used fresh quotes)
- ❌ Make strategy decisions (that's Planner's job)
- ❌ Retry failed transactions with different params (let Planner decide)

---

## TypeScript Executor Framework

### Package Structure

**Location**: `ts/packages/executor-framework/`

```
executor-framework/
├── src/
│   ├── types.ts                  # Type definitions
│   ├── base-executor.ts          # Abstract base class
│   ├── jito-executor.ts          # Jito implementation
│   ├── tpu-executor.ts           # TPU implementation (future)
│   ├── rpc-executor.ts           # RPC implementation (future)
│   ├── flash-loan-executor.ts    # Flash loan wrapper (future)
│   ├── transaction-builder.ts    # Transaction utilities
│   ├── wallet-manager.ts         # Wallet pool management
│   ├── executor-selector.ts      # Smart routing
│   └── index.ts                  # Public exports
├── package.json
├── tsconfig.json
├── tsdown.config.ts
└── README.md
```

### BaseExecutor Abstract Class

**Responsibilities**:
- Common lifecycle management (build → sign → submit → confirm)
- Retry logic with exponential backoff
- Metrics tracking
- Health monitoring
- Error handling

**Implementation**:

```typescript
export abstract class BaseExecutor {
  protected config: ExecutorConfig;
  protected metrics: MetricsCollector;
  protected logger: Logger;

  constructor(config: ExecutorConfig, logger?: Logger) {
    this.config = config;
    this.logger = logger || pino({ name: this.name() });
    this.metrics = new MetricsCollector(this.name());
  }

  // Abstract methods (must implement)
  abstract name(): string;
  abstract execute(request: ExecutionRequest): Promise<ExecutionResult>;

  // Template methods (can override)
  protected abstract buildTransaction(request: ExecutionRequest): Promise<VersionedTransaction>;
  protected abstract submitTransaction(tx: VersionedTransaction): Promise<string>;
  protected abstract confirmTransaction(signature: string, timeout: number): Promise<boolean>;

  // Helper methods (built-in)
  protected async signTransaction(
    tx: VersionedTransaction,
    signers: KeyPairSigner[]
  ): Promise<VersionedTransaction> {
    for (const signer of signers) {
      await tx.sign([signer]);
    }
    return tx;
  }

  protected async validateTransaction(tx: VersionedTransaction): Promise<void> {
    const serialized = tx.serialize();
    if (serialized.length > 1232) {
      throw new Error(`Transaction too large: ${serialized.length} bytes`);
    }
  }

  protected async handleExecutionError(error: Error, request: ExecutionRequest): Promise<void> {
    this.logger.error({ error, opportunity: request.opportunity.id }, 'Execution failed');
    this.metrics.incrementExecutionErrors(error.message);
  }

  // Retry logic
  protected async executeWithRetry<T>(
    fn: () => Promise<T>,
    maxRetries: number = 3
  ): Promise<T> {
    let lastError: Error;

    for (let i = 0; i < maxRetries; i++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error as Error;
        const delay = Math.pow(2, i) * 1000; // Exponential backoff
        await sleep(delay);
      }
    }

    throw lastError!;
  }
}
```

### Type Definitions

```typescript
export interface ExecutorConfig {
  name: string;
  enabled: boolean;

  // RPC configuration
  rpcEndpoint: string;
  rpcBackups: string[];

  // Jito configuration
  jitoBlockEngineUrl?: string;
  jitoUuid?: string;
  jitoTipBase?: number;
  jitoTipStep?: number;

  // TPU configuration
  tpuTimeout?: number;
  tpuRetryCount?: number;
  tpuFanoutSlots?: number;

  // Flash loan configuration
  kaminoMarketAddress?: string;

  // Transaction building
  computeUnitLimit: number;
  computeUnitPrice: number;

  // Confirmation
  confirmationTimeout: number;
  confirmationCommitment: 'processed' | 'confirmed' | 'finalized';

  // Error handling
  maxRetries: number;
  retryDelayMs: number;

  // Metrics
  metricsEnabled: boolean;
}

export interface ExecutionRequest {
  opportunity: TradeOpportunity;
  wallet: KeyPairSigner;
  timeout: number;
}

export interface ExecutionResult {
  opportunityId: string;
  status: 'success' | 'failed';
  signature?: string;
  error?: string;
  expectedProfit?: number;
  actualProfit?: number;
  latency: number;
  timestamp: number;
  metadata?: Record<string, unknown>;
}
```

---

## Transaction Building

### TransactionBuilder Class

**Purpose**: Fluent API for constructing Solana transactions with compute budget optimization and ALT compression.

**Implementation**:

```typescript
export class TransactionBuilder {
  private instructions: TransactionInstruction[] = [];
  private signers: KeyPairSigner[] = [];
  private feePayer: Address | null = null;
  private recentBlockhash: string | null = null;
  private addressLookupTables: AddressLookupTableAccount[] = [];

  addInstruction(ix: TransactionInstruction): this {
    this.instructions.push(ix);
    return this;
  }

  addInstructions(ixs: TransactionInstruction[]): this {
    this.instructions.push(...ixs);
    return this;
  }

  addComputeBudget(opts: { units: number; microLamports: number }): this {
    this.instructions.unshift(
      ComputeBudgetProgram.setComputeUnitLimit({ units: opts.units }),
      ComputeBudgetProgram.setComputeUnitPrice({ microLamports: opts.microLamports })
    );
    return this;
  }

  setFeePayer(feePayer: Address): this {
    this.feePayer = feePayer;
    return this;
  }

  setRecentBlockhash(blockhash: string): this {
    this.recentBlockhash = blockhash;
    return this;
  }

  addAddressLookupTable(alt: AddressLookupTableAccount): this {
    this.addressLookupTables.push(alt);
    return this;
  }

  async build(): Promise<VersionedTransaction> {
    if (!this.feePayer) throw new Error('Fee payer not set');
    if (!this.recentBlockhash) throw new Error('Recent blockhash not set');

    const message = MessageV0.compile({
      payerKey: this.feePayer,
      instructions: this.instructions,
      recentBlockhash: this.recentBlockhash,
      addressLookupTableAccounts: this.addressLookupTables
    });

    return new VersionedTransaction(message);
  }
}
```

### Usage Example

```typescript
// Build arbitrage transaction
const builder = new TransactionBuilder();

// Add compute budget (MUST be first)
builder.addComputeBudget({
  units: 400_000,
  microLamports: 1_000_000 // 1000 lamports priority fee
});

// Add swap instructions
for (const hop of opportunity.path) {
  const swapIxs = await buildSwapInstructions(hop, wallet);
  builder.addInstructions(swapIxs);
}

// Add Jito tip (if using Jito)
if (executionMethod === 'jito_bundle') {
  builder.addInstruction(
    SystemProgram.transfer({
      fromPubkey: wallet.address,
      toPubkey: jitoTipAccount,
      lamports: 10_000n // 0.00001 SOL tip
    })
  );
}

// Set transaction metadata
const blockhash = await connection.getLatestBlockhash();
builder.setFeePayer(wallet.address);
builder.setRecentBlockhash(blockhash.blockhash);

// Compress with ALT (optional, if transaction > 1232 bytes)
if (needsCompression) {
  builder.addAddressLookupTable(altAccount);
}

// Build transaction
const tx = await builder.build();

// Sign transaction
await tx.sign([wallet]);

// Validate size
const serialized = tx.serialize();
console.log(`Transaction size: ${serialized.length} bytes`); // Must be <= 1232
```

---

## Execution Methods

### 1. Jito Bundle Executor

**Purpose**: Submit transactions via Jito Block Engine for MEV protection and guaranteed execution.

**Use Cases**:
- High-value opportunities (> $0.10 profit)
- Critical priority trades
- Flash loan transactions

**Implementation**:

```typescript
export class JitoExecutor extends BaseExecutor {
  name = 'jito';

  private jitoClient: JitoJsonRpcClient;
  private tipAccounts: string[];

  constructor(config: ExecutorConfig, logger?: Logger) {
    super(config, logger);
    this.jitoClient = new JitoJsonRpcClient(
      config.jitoBlockEngineUrl!,
      config.jitoUuid!
    );
    this.tipAccounts = JITO_TIP_ACCOUNTS;
  }

  async execute(request: ExecutionRequest): Promise<ExecutionResult> {
    const startTime = Date.now();
    const tracer = trace.getTracer('executor-framework');

    return await tracer.startActiveSpan('jito_execution', async (span) => {
      try {
        // 1. Build transaction
        span.addEvent('building_transaction');
        const tx = await this.buildTransaction(request);

        // 2. Add tip instruction
        const tipAccount = this.selectTipAccount();
        const tipAmount = this.calculateTip(request.opportunity);
        await this.addTipInstruction(tx, tipAccount, tipAmount);

        // 3. Sign transaction
        span.addEvent('signing_transaction');
        const signed = await this.signTransaction(tx, [request.wallet]);

        // 4. Validate
        await this.validateTransaction(signed);

        // 5. Submit bundle
        span.addEvent('submitting_bundle');
        const bundleId = await this.submitTransaction(signed);

        // 6. Monitor confirmation
        span.addEvent('monitoring_confirmation');
        const confirmed = await this.confirmTransaction(bundleId, request.timeout);

        if (!confirmed) {
          throw new Error('Bundle confirmation timeout');
        }

        // 7. Parse results
        const actualProfit = await this.parseProfit(bundleId);

        span.setStatus({ code: SpanStatusCode.OK });

        return {
          opportunityId: request.opportunity.id,
          status: 'success',
          signature: bundleId,
          expectedProfit: request.opportunity.expectedProfitUsd,
          actualProfit,
          latency: Date.now() - startTime,
          timestamp: Date.now(),
          metadata: {
            executor: 'jito',
            tipAmount,
            tipAccount
          }
        };
      } catch (error) {
        span.recordException(error as Error);
        span.setStatus({ code: SpanStatusCode.ERROR, message: (error as Error).message });

        return {
          opportunityId: request.opportunity.id,
          status: 'failed',
          error: (error as Error).message,
          latency: Date.now() - startTime,
          timestamp: Date.now(),
          metadata: { executor: 'jito' }
        };
      } finally {
        span.end();
      }
    });
  }

  protected async buildTransaction(request: ExecutionRequest): Promise<VersionedTransaction> {
    const builder = new TransactionBuilder();

    // Add swap instructions based on opportunity routes
    for (const route of request.opportunity.routes) {
      const instructions = await this.buildSwapInstructions(route, request.wallet);
      builder.addInstructions(instructions);
    }

    // Add compute budget
    builder.addComputeBudget({
      units: 400_000,
      microLamports: 1_000_000 // 1000 lamports
    });

    // Set fee payer and blockhash
    const blockhash = await this.getRecentBlockhash();
    builder.setFeePayer(request.wallet.address);
    builder.setRecentBlockhash(blockhash);

    // Compress with ALT if needed
    const tx = await builder.build();
    return await this.compressWithALT(tx);
  }

  protected async submitTransaction(tx: VersionedTransaction): Promise<string> {
    const bundleId = await this.jitoClient.sendBundle([tx], {
      skipPreflight: false
    });

    this.logger.info({ bundleId }, 'Bundle submitted');
    return bundleId;
  }

  protected async confirmTransaction(bundleId: string, timeout: number): Promise<boolean> {
    const startTime = Date.now();

    while (Date.now() - startTime < timeout) {
      const status = await this.jitoClient.getBundleStatuses([bundleId]);

      if (status.value[0]?.confirmation_status === 'confirmed') {
        return true;
      }

      await sleep(2000); // Poll every 2 seconds
    }

    return false;
  }

  private selectTipAccount(): string {
    // Random selection or round-robin
    return this.tipAccounts[Math.floor(Math.random() * this.tipAccounts.length)];
  }

  private calculateTip(opportunity: TradeOpportunity): number {
    // Dynamic tip based on profit
    const baseTip = this.config.jitoTipBase || 5000;
    const profitBasedTip = Math.floor(opportunity.expectedProfitUsd * 0.01); // 1% of profit
    return Math.max(baseTip, profitBasedTip);
  }
}
```

**Jito Tip Accounts** (use random selection):
```typescript
const JITO_TIP_ACCOUNTS = [
  '96gYZGLnJYVFmbjzopPSU6QiEV5fGqZNyN9nmNhvrZU5',
  'HFqU5x63VTqvQss8hp11i4wVV8bD44PvwucfZ2bU7gRe',
  'Cw8CFyM9FkoMi7K7Crf6HNQqf4uEMzpKw6QNghXLvLkY',
  'ADaUMid9yfUytqMBgopwjb2DTLSokTSzL1zt6iGPaS49',
  'DfXygSm4jCyNCybVYYK6DwvWqjKee8pbDmJGcLWNDXjh'
];
```

### 2. TPU Direct Executor (Future)

**Purpose**: Submit transactions directly to validators via QUIC for lower latency.

**Use Cases**:
- Medium-value opportunities (> $0.03 profit)
- Normal priority trades
- No MEV risk

**Expected Performance**:
- Latency: 50-100ms (vs 100-200ms Jito)
- Cost: No tips required
- Success rate: 80-90% (vs 95%+ Jito)

**Implementation** (conceptual):

```typescript
export class TPUExecutor extends BaseExecutor {
  name = 'tpu';

  private tpuClient: QUICTPUClient;

  constructor(config: ExecutorConfig) {
    super(config);
    this.tpuClient = new QUICTPUClient({
      timeout: 5000,
      retryCount: 2,
      fanoutSlots: 4
    });
  }

  async execute(request: ExecutionRequest): Promise<ExecutionResult> {
    // Build and sign (same as Jito)
    const tx = await this.buildTransaction(request);
    const signed = await this.signTransaction(tx, [request.wallet]);
    await this.validateTransaction(signed);

    // Submit via TPU
    const signature = await this.submitTransaction(signed);

    // Confirm via RPC
    const confirmed = await this.confirmTransaction(signature, request.timeout);

    if (!confirmed) {
      throw new Error('Transaction confirmation timeout');
    }

    return {
      opportunityId: request.opportunity.id,
      status: 'success',
      signature,
      latency: Date.now() - startTime,
      timestamp: Date.now(),
      metadata: { executor: 'tpu' }
    };
  }

  protected async submitTransaction(tx: VersionedTransaction): Promise<string> {
    const serialized = tx.serialize();
    const signature = await this.tpuClient.send(serialized);

    this.logger.info({ signature }, 'Transaction submitted via TPU');
    return signature;
  }

  protected async confirmTransaction(signature: string, timeout: number): Promise<boolean> {
    const connection = new Connection(this.config.rpcEndpoint);

    try {
      const result = await connection.confirmTransaction(signature, 'confirmed', timeout);
      return !result.value.err;
    } catch {
      return false;
    }
  }
}
```

### 3. Flash Loan Executor (Future)

**Purpose**: Wrap any executor with Kamino flash loan instructions.

**Use Cases**:
- Large position sizes (20-100 SOL)
- Capital-efficient arbitrage
- No upfront capital required

**Implementation** (conceptual):

```typescript
export class FlashLoanExecutor extends BaseExecutor {
  name = 'flash-loan';

  private kaminoMarket: KaminoMarket;
  private baseExecutor: BaseExecutor;

  constructor(config: ExecutorConfig, baseExecutor: BaseExecutor) {
    super(config);
    this.baseExecutor = baseExecutor;
    this.kaminoMarket = new KaminoMarket(
      config.kaminoMarketAddress!,
      new Connection(config.rpcEndpoint)
    );
  }

  async execute(request: ExecutionRequest): Promise<ExecutionResult> {
    // Build flash loan transaction
    const tx = await this.buildFlashLoanTransaction(request);
    const signed = await this.signTransaction(tx, [request.wallet]);
    await this.validateTransaction(signed);

    // Submit via base executor
    const signature = await this.baseExecutor.submitTransaction(signed);
    const confirmed = await this.baseExecutor.confirmTransaction(signature, request.timeout);

    if (!confirmed) {
      throw new Error('Flash loan transaction timeout');
    }

    return {
      opportunityId: request.opportunity.id,
      status: 'success',
      signature,
      latency: Date.now() - startTime,
      timestamp: Date.now(),
      metadata: {
        executor: 'flash-loan',
        baseExecutor: this.baseExecutor.name()
      }
    };
  }

  protected async buildFlashLoanTransaction(request: ExecutionRequest): Promise<VersionedTransaction> {
    const builder = new TransactionBuilder();

    // 1. Flash loan borrow
    const borrowAmount = BigInt(request.opportunity.amount);
    const borrowIx = await this.kaminoMarket.getBorrowInstruction({
      owner: request.wallet.address,
      reserve: this.getReserveForToken(request.opportunity.tokenA),
      amount: borrowAmount
    });
    builder.addInstruction(borrowIx);

    // 2. Swap instructions
    for (const route of request.opportunity.routes) {
      const swapIxs = await this.buildSwapInstructions(route, request.wallet);
      builder.addInstructions(swapIxs);
    }

    // 3. Flash loan repay
    const repayIx = await this.kaminoMarket.getRepayInstruction({
      owner: request.wallet.address,
      reserve: this.getReserveForToken(request.opportunity.tokenA),
      amount: borrowAmount
    });
    builder.addInstruction(repayIx);

    // 4. Set transaction metadata
    const blockhash = await this.getRecentBlockhash();
    builder.setFeePayer(request.wallet.address);
    builder.setRecentBlockhash(blockhash);

    // 5. Add compute budget (flash loans need more CU)
    builder.addComputeBudget({
      units: 600_000, // Flash loans are more expensive
      microLamports: 2_000_000 // Higher priority
    });

    return await builder.build();
  }

  private getReserveForToken(tokenMint: string): string {
    // Map token mint to Kamino reserve address
    const RESERVES = {
      'So11111111111111111111111111111111111111112': 'SOL_RESERVE_ADDRESS',
      'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v': 'USDC_RESERVE_ADDRESS'
    };
    return RESERVES[tokenMint] || '';
  }
}
```

**Flash Loan Flow**:
```
1. Borrow 10 SOL from Kamino (no collateral)
   ↓
2. Swap 10 SOL → USDC on Raydium
   ↓
3. Swap USDC → SOL on Orca (profit: 10.05 SOL)
   ↓
4. Repay 10 SOL to Kamino
   ↓
5. Keep 0.05 SOL profit
```

---

## Wallet Management

### WalletManager Class

**Purpose**: Manage a pool of wallets for parallel execution and rotation.

**Implementation**:

```typescript
export class WalletManager {
  private wallets: KeyPairSigner[] = [];
  private walletStates: Map<string, WalletState> = new Map();
  private currentIndex = 0;

  constructor(private config: WalletConfig) {}

  async initialize(): Promise<void> {
    // Load wallets from configuration
    for (const privateKey of this.config.privateKeys) {
      const wallet = await this.loadWallet(privateKey);
      this.wallets.push(wallet);
      this.walletStates.set(wallet.address, {
        balance: 0,
        inFlight: 0,
        lastUsed: 0,
        errors: 0
      });
    }

    // Fetch initial balances
    await this.refreshBalances();
  }

  /**
   * Get next available wallet (round-robin with health checks)
   */
  getNextWallet(): KeyPairSigner {
    const maxAttempts = this.wallets.length;
    let attempts = 0;

    while (attempts < maxAttempts) {
      const wallet = this.wallets[this.currentIndex];
      const state = this.walletStates.get(wallet.address)!;

      // Advance index
      this.currentIndex = (this.currentIndex + 1) % this.wallets.length;
      attempts++;

      // Check wallet health
      if (this.isWalletHealthy(state)) {
        return wallet;
      }
    }

    throw new Error('No healthy wallets available');
  }

  /**
   * Get wallet with sufficient balance
   */
  getWalletWithBalance(minBalance: number): KeyPairSigner | null {
    for (const wallet of this.wallets) {
      const state = this.walletStates.get(wallet.address)!;
      if (state.balance >= minBalance && this.isWalletHealthy(state)) {
        return wallet;
      }
    }
    return null;
  }

  /**
   * Mark wallet as in-flight (transaction pending)
   */
  markInFlight(wallet: KeyPairSigner): void {
    const state = this.walletStates.get(wallet.address)!;
    state.inFlight++;
    state.lastUsed = Date.now();
  }

  /**
   * Mark wallet as completed (transaction confirmed)
   */
  markCompleted(wallet: KeyPairSigner, success: boolean): void {
    const state = this.walletStates.get(wallet.address)!;
    state.inFlight = Math.max(0, state.inFlight - 1);

    if (!success) {
      state.errors++;
    } else {
      state.errors = 0; // Reset error counter on success
    }
  }

  /**
   * Check wallet health
   */
  private isWalletHealthy(state: WalletState): boolean {
    // Too many errors
    if (state.errors >= 5) return false;

    // Too many in-flight transactions
    if (state.inFlight >= 3) return false;

    // Low balance
    if (state.balance < 0.1) return false;

    return true;
  }

  /**
   * Refresh wallet balances
   */
  async refreshBalances(): Promise<void> {
    const connection = new Connection(this.config.rpcEndpoint);

    const promises = this.wallets.map(async (wallet) => {
      const balance = await connection.getBalance(wallet.address);
      const state = this.walletStates.get(wallet.address)!;
      state.balance = balance / 1e9; // lamports to SOL
    });

    await Promise.all(promises);
  }

  /**
   * Get wallet statistics
   */
  getStats(): WalletStats {
    const stats: WalletStats = {
      total: this.wallets.length,
      healthy: 0,
      lowBalance: 0,
      highInFlight: 0,
      errored: 0
    };

    for (const state of this.walletStates.values()) {
      if (this.isWalletHealthy(state)) stats.healthy++;
      if (state.balance < 0.1) stats.lowBalance++;
      if (state.inFlight >= 3) stats.highInFlight++;
      if (state.errors >= 5) stats.errored++;
    }

    return stats;
  }
}

interface WalletState {
  balance: number;
  inFlight: number;
  lastUsed: number;
  errors: number;
}

interface WalletStats {
  total: number;
  healthy: number;
  lowBalance: number;
  highInFlight: number;
  errored: number;
}
```

### Usage Example

```typescript
// Initialize wallet manager
const walletManager = new WalletManager({
  privateKeys: [
    process.env.WALLET_1!,
    process.env.WALLET_2!,
    process.env.WALLET_3!,
    process.env.WALLET_4!,
    process.env.WALLET_5!
  ],
  rpcEndpoint: 'https://api.mainnet-beta.solana.com'
});

await walletManager.initialize();

// Get next wallet for execution
const wallet = walletManager.getNextWallet();

// Execute trade
walletManager.markInFlight(wallet);
try {
  await executor.execute({ opportunity, wallet, timeout: 10000 });
  walletManager.markCompleted(wallet, true);
} catch (error) {
  walletManager.markCompleted(wallet, false);
}

// Monitor wallet health
setInterval(async () => {
  await walletManager.refreshBalances();
  const stats = walletManager.getStats();
  console.log('Wallet stats:', stats);
}, 30000); // Every 30 seconds
```

---

## Rust Migration Strategy

### Why Migrate to Rust?

**Performance Benefits**:
- **30-40% faster execution**: 525-925ms (Rust) vs 750-1,500ms (TypeScript)
- **Zero-copy transaction parsing**: No serialization overhead
- **Better memory efficiency**: Lower GC pauses
- **Parallel execution**: Native async with Tokio

**When to Migrate**:
1. ✅ **Immediate**: Start with RPC proxy (already in Rust)
2. ⏳ **Month 2-3**: Transaction builder and Jito client
3. 🎯 **Month 4-6**: Full executor framework in Rust
4. 🚀 **Production**: TypeScript → Rust gradual rollout

### Rust Executor Architecture

**Crate Structure** (`rust/executor-framework/`):

```
executor-framework/
├── Cargo.toml
├── src/
│   ├── lib.rs                    # Public API
│   ├── base_executor.rs          # BaseExecutor trait
│   ├── jito_executor.rs          # Jito implementation
│   ├── tpu_executor.rs           # TPU implementation
│   ├── flash_loan_executor.rs    # Flash loan wrapper
│   ├── transaction_builder.rs    # Transaction utilities
│   ├── wallet_manager.rs         # Wallet pool
│   └── types.rs                  # Type definitions
└── examples/
    └── arbitrage_executor.rs
```

### BaseExecutor Trait

```rust
use async_trait::async_trait;
use solana_sdk::{
    signature::{Keypair, Signature},
    transaction::VersionedTransaction
};

#[async_trait]
pub trait BaseExecutor: Send + Sync {
    fn name(&self) -> &str;

    async fn execute(&self, request: ExecutionRequest) -> Result<ExecutionResult, ExecutorError>;

    async fn build_transaction(&self, request: &ExecutionRequest) -> Result<VersionedTransaction, ExecutorError>;

    async fn submit_transaction(&self, tx: &VersionedTransaction) -> Result<Signature, ExecutorError>;

    async fn confirm_transaction(&self, signature: &Signature, timeout_ms: u64) -> Result<bool, ExecutorError>;
}

pub struct ExecutionRequest {
    pub opportunity: TradeOpportunity,
    pub wallet: Keypair,
    pub timeout_ms: u64,
}

pub struct ExecutionResult {
    pub opportunity_id: String,
    pub status: ExecutionStatus,
    pub signature: Option<Signature>,
    pub error: Option<String>,
    pub expected_profit: Option<f64>,
    pub actual_profit: Option<f64>,
    pub latency_ms: u64,
    pub timestamp: i64,
}

#[derive(Debug, Clone, PartialEq)]
pub enum ExecutionStatus {
    Success,
    Failed,
    Timeout,
}
```

### Jito Executor (Rust)

```rust
pub struct JitoExecutor {
    config: ExecutorConfig,
    jito_client: JitoClient,
    tip_accounts: Vec<Pubkey>,
    metrics: Arc<Metrics>,
}

#[async_trait]
impl BaseExecutor for JitoExecutor {
    fn name(&self) -> &str {
        "jito"
    }

    async fn execute(&self, request: ExecutionRequest) -> Result<ExecutionResult, ExecutorError> {
        let start = Instant::now();

        // Build transaction
        let mut tx = self.build_transaction(&request).await?;

        // Add tip instruction
        let tip_account = self.select_tip_account();
        let tip_amount = self.calculate_tip(&request.opportunity);
        self.add_tip_instruction(&mut tx, tip_account, tip_amount)?;

        // Sign transaction
        tx.sign(&[&request.wallet], Hash::default());

        // Submit bundle
        let bundle_id = self.submit_transaction(&tx).await?;

        // Confirm
        let confirmed = self.confirm_transaction(&bundle_id, request.timeout_ms).await?;

        if !confirmed {
            return Err(ExecutorError::Timeout);
        }

        Ok(ExecutionResult {
            opportunity_id: request.opportunity.id.clone(),
            status: ExecutionStatus::Success,
            signature: Some(bundle_id),
            latency_ms: start.elapsed().as_millis() as u64,
            timestamp: chrono::Utc::now().timestamp(),
            ..Default::default()
        })
    }

    async fn submit_transaction(&self, tx: &VersionedTransaction) -> Result<Signature, ExecutorError> {
        let serialized = bincode::serialize(tx)?;

        let response = self.jito_client
            .send_bundle(&[serialized])
            .await?;

        Ok(response.bundle_id)
    }

    async fn confirm_transaction(&self, signature: &Signature, timeout_ms: u64) -> Result<bool, ExecutorError> {
        let start = Instant::now();

        while start.elapsed().as_millis() < timeout_ms as u128 {
            let status = self.jito_client
                .get_bundle_statuses(&[signature.to_string()])
                .await?;

            if status.value[0].confirmation_status == "confirmed" {
                return Ok(true);
            }

            tokio::time::sleep(Duration::from_millis(2000)).await;
        }

        Ok(false)
    }
}
```

### Performance Comparison

| Operation | TypeScript | Rust | Improvement |
|-----------|-----------|------|-------------|
| Transaction Build | 50-100ms | 20-40ms | **2x faster** |
| Signature Verify | 10-20ms | 2-5ms | **4x faster** |
| Bundle Submit | 100-200ms | 80-150ms | **20% faster** |
| Total Execution | 750-1,500ms | 525-925ms | **30-40% faster** |

### Shredstream Integration (Rust-Only)

**Key Advantage**: Get slot notifications 400ms before blocktime (early alpha).

```rust
use shredstream::ShredstreamClient;

pub struct ShredstreamScanner {
    client: ShredstreamClient,
    executor: Arc<dyn BaseExecutor>,
}

impl ShredstreamScanner {
    pub async fn subscribe_slots(&self) -> Result<(), Error> {
        let mut stream = self.client.subscribe_slots().await?;

        while let Some(slot) = stream.next().await {
            // 400ms early warning
            info!("New slot detected: {}, preparing execution", slot.slot);

            // Pre-compute quotes for pending opportunities
            self.prepare_executions(slot.slot).await?;
        }

        Ok(())
    }

    async fn prepare_executions(&self, slot: u64) -> Result<(), Error> {
        // Fetch pending opportunities
        let opportunities = self.fetch_pending_opportunities().await?;

        for opp in opportunities {
            // Pre-sign transaction (hot path)
            let tx = self.executor.build_transaction(&opp).await?;

            // Cache in memory
            self.cache_prepared_transaction(opp.id, tx).await;
        }

        Ok(())
    }
}
```

---

## Integration with Pipeline

### NATS Event Flow

```
Scanner → OPPORTUNITIES stream
           ↓
        Planner
           ↓
    PLANNED stream (ExecutionPlanEvent)
           ↓
        EXECUTOR ← YOU ARE HERE
           ↓
    EXECUTED stream (ExecutionResultEvent)
           ↓
    PnL Service, Monitor, Analytics
```

### Event Schemas

**Input**: ExecutionPlanEvent (from PLANNED stream)

```typescript
interface ExecutionPlanEvent {
  planId: string;
  opportunityId: string;
  strategy: 'arbitrage' | 'triangular' | 'market_making';
  executionMethod: 'jito_bundle' | 'tpu_direct' | 'rpc';

  // Transaction details
  routes: SwapRoute[];
  inputToken: string;
  outputToken: string;
  inputAmount: number;
  expectedProfit: number;

  // Validation results
  simulationSuccess: boolean;
  gasEstimate: number;
  slippage: number;

  // Execution constraints
  maxLatency: number;
  deadline: number;
  priorityFee: number;

  // Flash loan (optional)
  flashLoan?: {
    provider: 'kamino';
    amount: number;
    reserve: string;
  };

  timestamp: number;
}
```

**Output**: ExecutionResultEvent (to EXECUTED stream)

```typescript
interface ExecutionResultEvent {
  executionId: string;
  planId: string;
  opportunityId: string;
  status: 'success' | 'failed' | 'timeout';

  // Transaction results
  signature?: string;
  blockSlot?: number;
  confirmationTime?: number;

  // Financial results
  expectedProfit: number;
  actualProfit?: number;
  executionCost: number;
  netProfit?: number;

  // Performance metrics
  latency: number;
  executionMethod: string;

  // Error details (if failed)
  error?: string;
  errorCode?: string;

  timestamp: number;
}
```

### NATS Subscriber (TypeScript)

```typescript
import { connect, JSONCodec } from 'nats';

export class ExecutorService {
  private nc: NatsConnection;
  private js: JetStreamClient;
  private executor: BaseExecutor;
  private walletManager: WalletManager;

  async start(): Promise<void> {
    // Connect to NATS
    this.nc = await connect({ servers: 'nats://localhost:4222' });
    this.js = this.nc.jetstream();

    // Subscribe to PLANNED stream
    const sub = await this.js.subscribe('execution.planned.*', {
      stream: 'PLANNED',
      durable: 'executor-consumer',
      ack_policy: 'explicit'
    });

    // Subscribe to SYSTEM stream (kill switch)
    const systemSub = await this.js.subscribe('system.killswitch', {
      stream: 'SYSTEM'
    });

    // Process execution plans
    (async () => {
      for await (const msg of sub) {
        await this.handleExecutionPlan(msg);
      }
    })();

    // Monitor kill switch
    (async () => {
      for await (const msg of systemSub) {
        await this.handleKillSwitch(msg);
      }
    })();

    this.logger.info('Executor service started');
  }

  private async handleExecutionPlan(msg: JsMsg): Promise<void> {
    const jc = JSONCodec<ExecutionPlanEvent>();
    const plan = jc.decode(msg.data);

    try {
      // Select wallet
      const wallet = this.walletManager.getNextWallet();

      // Execute
      const result = await this.executor.execute({
        opportunity: plan,
        wallet,
        timeout: plan.maxLatency
      });

      // Publish result
      await this.publishResult(result);

      // Ack message
      msg.ack();
    } catch (error) {
      this.logger.error({ error, plan }, 'Execution failed');

      // Publish failure
      await this.publishResult({
        executionId: crypto.randomUUID(),
        planId: plan.planId,
        opportunityId: plan.opportunityId,
        status: 'failed',
        error: error.message,
        latency: 0,
        timestamp: Date.now()
      });

      // Ack message (don't reprocess)
      msg.ack();
    }
  }

  private async publishResult(result: ExecutionResultEvent): Promise<void> {
    const jc = JSONCodec<ExecutionResultEvent>();

    await this.js.publish(
      `executed.${result.status}.${result.opportunityId}`,
      jc.encode(result)
    );
  }

  private async handleKillSwitch(msg: JsMsg): Promise<void> {
    this.logger.error('Kill switch triggered, stopping executor');

    // Stop processing new messages
    await this.stop();

    // Publish shutdown metric
    await this.js.publish('metrics.executor.stopped', JSONCodec().encode({
      reason: 'killswitch',
      timestamp: Date.now()
    }));
  }

  async stop(): Promise<void> {
    await this.nc.drain();
    await this.nc.close();
    this.logger.info('Executor service stopped');
  }
}
```

---

## Performance Optimization

### 1. Cache Blockhash (50ms saved)

```typescript
export class BlockhashCache {
  private cachedBlockhash: { blockhash: string; lastValidBlockHeight: number } | null = null;
  private lastFetch = 0;

  async getBlockhash(connection: Connection): Promise<string> {
    const now = Date.now();

    // Refresh every 30 seconds (blockhash valid for 150 seconds)
    if (!this.cachedBlockhash || now - this.lastFetch > 30_000) {
      this.cachedBlockhash = await connection.getLatestBlockhash();
      this.lastFetch = now;
    }

    return this.cachedBlockhash.blockhash;
  }
}
```

### 2. Parallel RPC Calls

```typescript
// ❌ Sequential (slow)
const balance = await connection.getBalance(wallet);
const tokenAccounts = await connection.getTokenAccountsByOwner(wallet, { programId: TOKEN_PROGRAM_ID });

// ✅ Parallel (2x faster)
const [balance, tokenAccounts] = await Promise.all([
  connection.getBalance(wallet),
  connection.getTokenAccountsByOwner(wallet, { programId: TOKEN_PROGRAM_ID })
]);
```

### 3. Pre-warm Connections

```typescript
export class RPCConnectionPool {
  private connections: Connection[] = [];

  constructor(endpoints: string[]) {
    // Create connection pool
    this.connections = endpoints.map(url => new Connection(url, {
      commitment: 'confirmed',
      wsEndpoint: url.replace('https', 'wss')
    }));

    // Pre-warm connections
    this.warmup();
  }

  private async warmup(): Promise<void> {
    await Promise.all(
      this.connections.map(conn => conn.getSlot())
    );
  }

  getConnection(): Connection {
    // Round-robin selection
    return this.connections[Math.floor(Math.random() * this.connections.length)];
  }
}
```

### 4. Optimize Transaction Size

```typescript
// Use Address Lookup Tables (ALT) to compress transactions
export async function compressWithALT(
  tx: VersionedTransaction,
  alt: AddressLookupTableAccount
): Promise<VersionedTransaction> {
  const message = MessageV0.compile({
    payerKey: tx.message.staticAccountKeys[0],
    instructions: TransactionMessage.decompile(tx.message).instructions,
    recentBlockhash: tx.message.recentBlockhash,
    addressLookupTableAccounts: [alt]
  });

  return new VersionedTransaction(message);
}
```

### 5. Dynamic Priority Fees

```typescript
export async function calculatePriorityFee(
  opportunity: TradeOpportunity,
  congestion: 'low' | 'medium' | 'high'
): Promise<number> {
  // Base fee
  let microLamports = 1_000_000; // 1000 lamports

  // Adjust for congestion
  if (congestion === 'medium') microLamports *= 2;
  if (congestion === 'high') microLamports *= 5;

  // Adjust for opportunity value
  const profitBased = Math.floor(opportunity.expectedProfitUsd * 1_000_000);
  microLamports = Math.max(microLamports, profitBased);

  return microLamports;
}
```

---

## Best Practices

### 1. Error Handling

```typescript
try {
  const result = await executor.execute(request);
} catch (error) {
  if (error instanceof InsufficientFundsError) {
    // Wallet balance too low
    await walletManager.markCompleted(wallet, false);
    await fundWallet(wallet);
  } else if (error instanceof TransactionExpiredError) {
    // Blockhash expired, retry with fresh blockhash
    request.blockhash = await getLatestBlockhash();
    await executor.execute(request);
  } else if (error instanceof SimulationFailedError) {
    // Don't retry, opportunity is stale
    logger.warn('Simulation failed, skipping opportunity');
  } else {
    // Unknown error, log and skip
    logger.error({ error }, 'Unknown execution error');
  }
}
```

### 2. Circuit Breaker

```typescript
export class CircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      // Check if we should try half-open
      if (Date.now() - this.lastFailure > 60_000) {
        this.state = 'half-open';
      } else {
        throw new Error('Circuit breaker is open');
      }
    }

    try {
      const result = await fn();

      // Success, reset
      this.failures = 0;
      this.state = 'closed';

      return result;
    } catch (error) {
      this.failures++;
      this.lastFailure = Date.now();

      // Open circuit after 5 failures
      if (this.failures >= 5) {
        this.state = 'open';
        logger.error('Circuit breaker opened after 5 failures');
      }

      throw error;
    }
  }
}
```

### 3. Rate Limiting

```typescript
export class RateLimiter {
  private requests: number[] = [];

  async acquire(maxPerSecond: number): Promise<void> {
    const now = Date.now();

    // Remove requests older than 1 second
    this.requests = this.requests.filter(ts => now - ts < 1000);

    // Check rate limit
    if (this.requests.length >= maxPerSecond) {
      const oldestRequest = this.requests[0];
      const waitTime = 1000 - (now - oldestRequest);
      await sleep(waitTime);
      return this.acquire(maxPerSecond);
    }

    this.requests.push(now);
  }
}
```

### 4. Metrics Collection

```typescript
export class ExecutorMetrics {
  private registry = new Registry();

  executionsTotal = new Counter({
    name: 'executor_executions_total',
    help: 'Total number of executions',
    labelNames: ['status', 'method'],
    registers: [this.registry]
  });

  executionDuration = new Histogram({
    name: 'executor_execution_duration_ms',
    help: 'Execution duration in milliseconds',
    labelNames: ['method'],
    buckets: [10, 50, 100, 200, 500, 1000, 2000, 5000],
    registers: [this.registry]
  });

  profitTotal = new Counter({
    name: 'executor_profit_usd_total',
    help: 'Total profit in USD',
    labelNames: ['method'],
    registers: [this.registry]
  });

  recordExecution(result: ExecutionResult): void {
    this.executionsTotal.inc({
      status: result.status,
      method: result.metadata?.executor
    });

    this.executionDuration.observe(
      { method: result.metadata?.executor },
      result.latency
    );

    if (result.actualProfit) {
      this.profitTotal.inc(
        { method: result.metadata?.executor },
        result.actualProfit
      );
    }
  }
}
```

---

## Deployment Guide

### Environment Variables

```bash
# Executor Configuration
EXECUTOR_NAME=jito-executor-1
EXECUTOR_ENABLED=true

# RPC Endpoints (multiple for redundancy)
RPC_ENDPOINT=https://api.mainnet-beta.solana.com
RPC_BACKUPS=https://api.helius.xyz/...,https://rpc.ankr.com/solana

# Jito Configuration
JITO_BLOCK_ENGINE_URL=https://mainnet.block-engine.jito.wtf
JITO_UUID=your-jito-uuid
JITO_TIP_BASE=5000
JITO_TIP_STEP=1000

# Flash Loan Configuration
KAMINO_MARKET_ADDRESS=your-kamino-market-address

# Transaction Settings
COMPUTE_UNIT_LIMIT=400000
COMPUTE_UNIT_PRICE=1000000
CONFIRMATION_TIMEOUT=30000
CONFIRMATION_COMMITMENT=confirmed

# Wallet Configuration (private keys)
WALLET_1=base58-private-key-1
WALLET_2=base58-private-key-2
WALLET_3=base58-private-key-3
WALLET_4=base58-private-key-4
WALLET_5=base58-private-key-5

# NATS Configuration
NATS_URL=nats://localhost:4222
NATS_STREAM_EXECUTION=PLANNED
NATS_STREAM_EXECUTED=EXECUTED
NATS_STREAM_SYSTEM=SYSTEM

# Metrics Configuration
METRICS_ENABLED=true
METRICS_PORT=9090
```

### Docker Deployment

**Dockerfile** (`deployment/docker/Dockerfile.executor`):

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY ts/package.json ts/pnpm-lock.yaml ./
COPY ts/packages/executor-framework ./packages/executor-framework
COPY ts/apps/executor-service ./apps/executor-service

# Install dependencies
RUN npm install -g pnpm@9.0.0
RUN pnpm install --frozen-lockfile

# Build
RUN pnpm build --filter=executor-service

# Expose metrics port
EXPOSE 9090

# Run
CMD ["node", "apps/executor-service/dist/index.js"]
```

**Docker Compose** (`deployment/docker/docker-compose.yml`):

```yaml
services:
  executor-jito-1:
    build:
      context: ../../
      dockerfile: deployment/docker/Dockerfile.executor
    environment:
      - EXECUTOR_NAME=jito-executor-1
      - EXECUTOR_ENABLED=true
      - RPC_ENDPOINT=${RPC_ENDPOINT}
      - JITO_BLOCK_ENGINE_URL=${JITO_BLOCK_ENGINE_URL}
      - JITO_UUID=${JITO_UUID}
      - NATS_URL=nats://nats:4222
    depends_on:
      - nats
    restart: unless-stopped
    networks:
      - trading-system

  executor-jito-2:
    build:
      context: ../../
      dockerfile: deployment/docker/Dockerfile.executor
    environment:
      - EXECUTOR_NAME=jito-executor-2
      - EXECUTOR_ENABLED=true
      - RPC_ENDPOINT=${RPC_ENDPOINT}
      - JITO_BLOCK_ENGINE_URL=${JITO_BLOCK_ENGINE_URL}
      - JITO_UUID=${JITO_UUID}
      - NATS_URL=nats://nats:4222
    depends_on:
      - nats
    restart: unless-stopped
    networks:
      - trading-system

networks:
  trading-system:
    external: true
```

### Monitoring

**Grafana Dashboard** (executor metrics):

```json
{
  "dashboard": {
    "title": "Executor Performance",
    "panels": [
      {
        "title": "Executions per Second",
        "targets": [
          {
            "expr": "rate(executor_executions_total[1m])",
            "legendFormat": "{{method}} - {{status}}"
          }
        ]
      },
      {
        "title": "Execution Latency (p50, p95, p99)",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, executor_execution_duration_ms)",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.95, executor_execution_duration_ms)",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, executor_execution_duration_ms)",
            "legendFormat": "p99"
          }
        ]
      },
      {
        "title": "Profit per Hour",
        "targets": [
          {
            "expr": "rate(executor_profit_usd_total[1h]) * 3600",
            "legendFormat": "{{method}}"
          }
        ]
      }
    ]
  }
}
```

---

## Summary

**Executor Framework Checklist**:

| Component | Status | Priority | Complexity |
|-----------|--------|----------|------------|
| BaseExecutor class | ✅ Ready | High | Low |
| TransactionBuilder | ✅ Ready | High | Low |
| JitoExecutor | ⏳ SDK pending | High | Medium |
| WalletManager | ✅ Ready | High | Low |
| TPUExecutor | ❌ Not started | Medium | High |
| FlashLoanExecutor | ❌ Not started | Low | High |
| NATS integration | ✅ Ready | High | Low |
| Metrics | ✅ Ready | Medium | Low |
| Rust migration | 🎯 Planned | Medium | High |
| Shredstream | 🎯 Planned | High | Very High |

**Next Steps**:

1. **Week 1**: Implement JitoExecutor with jito-ts SDK
2. **Week 2**: Deploy executor service with NATS integration
3. **Week 3**: Add WalletManager for parallel execution
4. **Week 4**: Implement metrics and monitoring
5. **Month 2**: Start Rust migration for 30-40% performance gain
6. **Month 3**: Add Shredstream for 400ms early alpha

**Performance Target**: < 500ms total execution (Scanner 10ms + Planner 100ms + Executor 20ms + Confirmation 200ms)

---

**References**:
- [18-HFT_PIPELINE_ARCHITECTURE.md](./18-HFT_PIPELINE_ARCHITECTURE.md) - Complete pipeline architecture
- [08-optimization-guide.md](./08-optimization-guide.md) - Week-by-week optimization roadmap
- [07-hft-architecture.md](./07-hft-architecture.md) - Sub-500ms execution strategies