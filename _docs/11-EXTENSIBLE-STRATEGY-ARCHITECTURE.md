---
layout: single
title: "Extensible Multi-Strategy Trading Architecture"
permalink: /docs/11-extensible-strategy-architecture/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Extensible Multi-Strategy Trading Architecture

## Overview

This document outlines a **flexible, extensible architecture** that supports multiple trading strategies (arbitrage, DCA, grid trading, long/short) using a common event-driven framework.

**Key Principles:**
1. **Generic Scanner Events** - Reusable across all strategies
2. **Strategy Plugin System** - Easy to add new strategies
3. **Event-Driven Architecture** - NATS for loose coupling
4. **Common Interfaces** - Standardized strategy API
5. **Rust Performance** - Critical path in Rust, strategies can be Rust or TypeScript

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    MARKET DATA LAYER (Rust)                      │
│  Shredstream → Generic Event Emitter → NATS Subjects           │
└─────────────────────────────────────────────────────────────────┘
                              ↓ publishes to
┌─────────────────────────────────────────────────────────────────┐
│                      NATS JETSTREAM                              │
│  market.price.{token}                                           │
│  market.liquidity.{pool}                                        │
│  market.trade.{size}                                            │
│  market.spread.{pair}                                           │
│  market.volume.{token}                                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓ subscribes from
┌─────────────────────────────────────────────────────────────────┐
│                   STRATEGY ENGINES (Pluggable)                   │
│  ┌──────────────────┐  ┌──────────────────┐                   │
│  │ Arbitrage Engine │  │ DCA Engine       │                   │
│  │ (Rust)           │  │ (TypeScript)     │                   │
│  │ - LST 2-hop      │  │ - Time-based     │                   │
│  │ - LST 3-hop      │  │ - Price-based    │                   │
│  │ - USDC 2-hop     │  │ - RSI-based      │                   │
│  └──────────────────┘  └──────────────────┘                   │
│  ┌──────────────────┐  ┌──────────────────┐                   │
│  │ Grid Trading     │  │ Long/Short       │                   │
│  │ (TypeScript)     │  │ (Rust)           │                   │
│  │ - Range-bound    │  │ - Trend-based    │                   │
│  │ - Volatility     │  │ - Mean-reversion │                   │
│  └──────────────────┘  └──────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓ emits to
┌─────────────────────────────────────────────────────────────────┐
│                      NATS JETSTREAM                              │
│  trade.opportunity.{strategy}.{priority}                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓ subscribes from
┌─────────────────────────────────────────────────────────────────┐
│                   PLANNED LAYER (Rust)                         │
│  Priority Queue → Risk Check → Build TX → Simulate → Execute   │
└─────────────────────────────────────────────────────────────────┘
                              ↓ results to
┌─────────────────────────────────────────────────────────────────┐
│                   ANALYTICS LAYER (TypeScript)                   │
│  PnL per Strategy → Performance Analysis → Dashboards           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Generic Market Events (Reusable)

### Event Schema Design

**Create:** `rust/market-events/src/events.rs`

```rust
use serde::{Deserialize, Serialize};
use solana_sdk::pubkey::Pubkey;

/// Generic market event that can be used by any strategy
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum MarketEvent {
    /// Price update for a token
    PriceUpdate(PriceUpdateEvent),

    /// Liquidity change in a pool
    LiquidityUpdate(LiquidityUpdateEvent),

    /// Large trade detected
    LargeTrade(LargeTradeEvent),

    /// Spread change between DEXes
    SpreadUpdate(SpreadUpdateEvent),

    /// Volume spike
    VolumeSpike(VolumeSpikeEvent),

    /// Pool state change (reserves, fees, etc.)
    PoolStateChange(PoolStateChangeEvent),

    /// New block/slot
    SlotUpdate(SlotUpdateEvent),
}

/// Price update event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PriceUpdateEvent {
    pub token: Pubkey,
    pub symbol: String,
    pub price_usd: f64,
    pub price_sol: f64,
    pub source: String,        // "raydium_amm", "orca", "pyth"
    pub timestamp: u64,
    pub slot: u64,
}

/// Liquidity update event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LiquidityUpdateEvent {
    pub pool_id: Pubkey,
    pub dex: String,
    pub token_a: Pubkey,
    pub token_b: Pubkey,
    pub reserve_a: u64,
    pub reserve_b: u64,
    pub liquidity_usd: f64,
    pub timestamp: u64,
    pub slot: u64,
}

/// Large trade event (useful for backrunning, momentum strategies)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LargeTradeEvent {
    pub signature: String,
    pub pool_id: Pubkey,
    pub dex: String,
    pub token_in: Pubkey,
    pub token_out: Pubkey,
    pub amount_in: u64,
    pub amount_out: u64,
    pub price_impact_bps: u16,
    pub trader: Pubkey,
    pub timestamp: u64,
    pub slot: u64,
}

/// Spread update event (useful for arbitrage)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SpreadUpdateEvent {
    pub token_a: Pubkey,
    pub token_b: Pubkey,
    pub buy_dex: String,
    pub buy_price: f64,
    pub sell_dex: String,
    pub sell_price: f64,
    pub spread_bps: u16,
    pub timestamp: u64,
}

/// Volume spike event (useful for momentum, volatility strategies)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VolumeSpikeEvent {
    pub token: Pubkey,
    pub symbol: String,
    pub volume_1m: f64,        // Last 1 minute
    pub volume_5m: f64,        // Last 5 minutes
    pub average_volume: f64,   // Average volume
    pub spike_ratio: f64,      // Current / Average
    pub timestamp: u64,
}

/// Pool state change event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PoolStateChangeEvent {
    pub pool_id: Pubkey,
    pub dex: String,
    pub token_a: Pubkey,
    pub token_b: Pubkey,
    pub previous_state: PoolState,
    pub current_state: PoolState,
    pub timestamp: u64,
    pub slot: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PoolState {
    pub reserve_a: u64,
    pub reserve_b: u64,
    pub fee_rate: u16,
    pub liquidity_usd: f64,
}

/// Slot update event (for timing-sensitive strategies)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SlotUpdateEvent {
    pub slot: u64,
    pub parent: u64,
    pub timestamp: u64,
    pub leader: Option<Pubkey>,
}

impl MarketEvent {
    /// Get NATS subject for this event
    pub fn nats_subject(&self) -> String {
        match self {
            MarketEvent::PriceUpdate(e) => {
                format!("market.price.{}", e.symbol)
            }
            MarketEvent::LiquidityUpdate(e) => {
                format!("market.liquidity.{}", e.pool_id)
            }
            MarketEvent::LargeTrade(e) => {
                format!("market.trade.large.{}", e.dex)
            }
            MarketEvent::SpreadUpdate(e) => {
                format!("market.spread.{}.{}", e.buy_dex, e.sell_dex)
            }
            MarketEvent::VolumeSpike(e) => {
                format!("market.volume.spike.{}", e.symbol)
            }
            MarketEvent::PoolStateChange(e) => {
                format!("market.pool.{}.{}", e.dex, e.pool_id)
            }
            MarketEvent::SlotUpdate(e) => {
                format!("market.slot.{}", e.slot)
            }
        }
    }

    /// Get event priority (for filtering)
    pub fn priority(&self) -> EventPriority {
        match self {
            MarketEvent::PriceUpdate(_) => EventPriority::Normal,
            MarketEvent::LiquidityUpdate(_) => EventPriority::Normal,
            MarketEvent::LargeTrade(_) => EventPriority::High,
            MarketEvent::SpreadUpdate(_) => EventPriority::High,
            MarketEvent::VolumeSpike(_) => EventPriority::High,
            MarketEvent::PoolStateChange(_) => EventPriority::Normal,
            MarketEvent::SlotUpdate(_) => EventPriority::Low,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum EventPriority {
    Low = 1,
    Normal = 2,
    High = 3,
}
```

### Event Publisher (Scanner)

**Create:** `rust/market-scanner/src/publisher.rs`

```rust
use async_nats::jetstream;
use market_events::MarketEvent;

pub struct EventPublisher {
    js: jetstream::Context,
}

impl EventPublisher {
    pub async fn new(nats_url: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let client = async_nats::connect(nats_url).await?;
        let js = jetstream::new(client);

        // Create streams for market events
        let _ = js
            .create_stream(jetstream::stream::Config {
                name: "MARKET_EVENTS".to_string(),
                subjects: vec!["market.>".to_string()],
                max_age: std::time::Duration::from_secs(3600), // 1 hour retention
                storage: jetstream::stream::StorageType::Memory,
                ..Default::default()
            })
            .await;

        Ok(Self { js })
    }

    /// Publish a market event
    pub async fn publish(&self, event: &MarketEvent) -> Result<(), Box<dyn std::error::Error>> {
        let subject = event.nats_subject();
        let payload = serde_json::to_vec(event)?;

        self.js
            .publish(subject, payload.into())
            .await?
            .await?;

        Ok(())
    }

    /// Publish multiple events (batch)
    pub async fn publish_batch(
        &self,
        events: Vec<MarketEvent>,
    ) -> Result<(), Box<dyn std::error::Error>> {
        for event in events {
            self.publish(&event).await?;
        }
        Ok(())
    }
}
```

---

## 2. Strategy Plugin System

### Strategy Interface

**Create:** `rust/strategy-framework/src/lib.rs`

```rust
use async_trait::async_trait;
use market_events::MarketEvent;
use serde::{Deserialize, Serialize};
use solana_sdk::pubkey::Pubkey;

/// Common trait for all trading strategies
#[async_trait]
pub trait TradingStrategy: Send + Sync {
    /// Strategy name
    fn name(&self) -> &str;

    /// Strategy type
    fn strategy_type(&self) -> StrategyType;

    /// Which market events does this strategy care about?
    fn subscribed_events(&self) -> Vec<String>;

    /// Process a market event and potentially generate opportunities
    async fn process_event(
        &mut self,
        event: &MarketEvent,
    ) -> Result<Vec<TradeOpportunity>, Box<dyn std::error::Error>>;

    /// Strategy configuration
    fn config(&self) -> &StrategyConfig;

    /// Update strategy configuration (hot reload)
    async fn update_config(&mut self, config: StrategyConfig);

    /// Start strategy (initialization, subscriptions, etc.)
    async fn start(&mut self) -> Result<(), Box<dyn std::error::Error>>;

    /// Stop strategy (cleanup)
    async fn stop(&mut self) -> Result<(), Box<dyn std::error::Error>>;

    /// Get strategy health/status
    fn health(&self) -> StrategyHealth;
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum StrategyType {
    Arbitrage,
    DCA,
    GridTrading,
    LongShort,
    MarketMaking,
    Momentum,
    MeanReversion,
    Custom(String),
}

/// Trade opportunity generated by a strategy
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TradeOpportunity {
    pub id: String,
    pub strategy: String,
    pub strategy_type: StrategyType,
    pub priority: OpportunityPriority,

    // Trade details
    pub trade_type: TradeType,
    pub token_in: Pubkey,
    pub token_out: Pubkey,
    pub amount_in: u64,
    pub expected_amount_out: u64,

    // For multi-step trades (arbitrage, etc.)
    pub steps: Vec<TradeStep>,

    // Expected profit
    pub expected_profit: i64,
    pub expected_profit_pct: f64,

    // Risk parameters
    pub require_simulation: bool,
    pub max_slippage_bps: u16,
    pub deadline: Option<u64>, // Timestamp deadline

    // Metadata
    pub detected_at: u64,
    pub metadata: serde_json::Value,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, PartialOrd, Ord)]
pub enum OpportunityPriority {
    Low = 1,
    Medium = 2,
    High = 3,
    Critical = 4,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TradeType {
    Swap,           // Simple swap
    Arbitrage,      // Multi-step arbitrage
    DCA,            // Dollar-cost averaging
    GridOrder,      // Grid trading order
    LongPosition,   // Open long position
    ShortPosition,  // Open short position
    ClosePosition,  // Close existing position
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TradeStep {
    pub from_token: Pubkey,
    pub to_token: Pubkey,
    pub dex: String,
    pub pool_id: Pubkey,
    pub input_amount: u64,
    pub expected_output: u64,
}

/// Strategy configuration
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StrategyConfig {
    pub enabled: bool,
    pub min_profit_bps: u16,
    pub max_position_size: u64,
    pub require_simulation: bool,
    pub custom_params: serde_json::Value,
}

/// Strategy health status
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StrategyHealth {
    pub is_healthy: bool,
    pub events_processed: u64,
    pub opportunities_generated: u64,
    pub last_event_timestamp: u64,
    pub error_count: u64,
    pub last_error: Option<String>,
}

impl TradeOpportunity {
    /// Get NATS subject for publishing this opportunity
    pub fn nats_subject(&self) -> String {
        format!(
            "trade.opportunity.{}.{}",
            self.strategy,
            match self.priority {
                OpportunityPriority::Critical => "critical",
                OpportunityPriority::High => "high",
                OpportunityPriority::Medium => "medium",
                OpportunityPriority::Low => "low",
            }
        )
    }
}
```

---

## 3. Strategy Implementations

### 3.1 Arbitrage Strategy (Rust)

**Create:** `rust/strategies/arbitrage/src/lib.rs`

```rust
use async_trait::async_trait;
use market_events::{MarketEvent, SpreadUpdateEvent};
use strategy_framework::*;

pub struct ArbitrageStrategy {
    name: String,
    config: StrategyConfig,
    health: StrategyHealth,

    // Internal state
    quote_cache: std::collections::HashMap<String, Quote>,
}

impl ArbitrageStrategy {
    pub fn new(config: StrategyConfig) -> Self {
        Self {
            name: "arbitrage".to_string(),
            config,
            health: StrategyHealth {
                is_healthy: true,
                events_processed: 0,
                opportunities_generated: 0,
                last_event_timestamp: 0,
                error_count: 0,
                last_error: None,
            },
            quote_cache: std::collections::HashMap::new(),
        }
    }

    /// Detect LST two-hop arbitrage
    fn detect_lst_two_hop(&self, event: &SpreadUpdateEvent) -> Option<TradeOpportunity> {
        // Implementation from previous docs
        None
    }

    /// Detect LST three-hop triangular arbitrage
    fn detect_lst_three_hop(&self, event: &SpreadUpdateEvent) -> Option<TradeOpportunity> {
        // Implementation from previous docs
        None
    }
}

#[async_trait]
impl TradingStrategy for ArbitrageStrategy {
    fn name(&self) -> &str {
        &self.name
    }

    fn strategy_type(&self) -> StrategyType {
        StrategyType::Arbitrage
    }

    fn subscribed_events(&self) -> Vec<String> {
        vec![
            "market.spread.>".to_string(),
            "market.price.>".to_string(),
            "market.liquidity.>".to_string(),
        ]
    }

    async fn process_event(
        &mut self,
        event: &MarketEvent,
    ) -> Result<Vec<TradeOpportunity>, Box<dyn std::error::Error>> {
        self.health.events_processed += 1;
        self.health.last_event_timestamp = current_time_micros();

        let mut opportunities = Vec::new();

        match event {
            MarketEvent::SpreadUpdate(spread) => {
                // Detect arbitrage opportunities
                if let Some(opp) = self.detect_lst_two_hop(spread) {
                    opportunities.push(opp);
                }

                if let Some(opp) = self.detect_lst_three_hop(spread) {
                    opportunities.push(opp);
                }
            }
            MarketEvent::PriceUpdate(price) => {
                // Update quote cache for future calculations
                // ...
            }
            _ => {}
        }

        self.health.opportunities_generated += opportunities.len() as u64;

        Ok(opportunities)
    }

    fn config(&self) -> &StrategyConfig {
        &self.config
    }

    async fn update_config(&mut self, config: StrategyConfig) {
        self.config = config;
    }

    async fn start(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        println!("Starting Arbitrage Strategy");
        Ok(())
    }

    async fn stop(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        println!("Stopping Arbitrage Strategy");
        Ok(())
    }

    fn health(&self) -> StrategyHealth {
        self.health.clone()
    }
}

#[derive(Clone)]
struct Quote {
    input_mint: solana_sdk::pubkey::Pubkey,
    output_mint: solana_sdk::pubkey::Pubkey,
    amount: u64,
    output: u64,
}

fn current_time_micros() -> u64 {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_micros() as u64
}
```

### 3.2 DCA Strategy (TypeScript)

**Create:** `apps/strategies/dca-strategy/src/index.ts`

```typescript
import { TradingStrategy, StrategyConfig, TradeOpportunity, MarketEvent } from '@trading-system/strategy-framework';
import { PublicKey } from '@solana/web3.js';

interface DCAConfig extends StrategyConfig {
  token: string;
  intervalSeconds: number;
  amountPerInterval: number;
  maxPrice?: number;  // Only buy if price < maxPrice
  rsiThreshold?: number;  // Only buy if RSI < threshold
}

export class DCAStrategy implements TradingStrategy {
  private config: DCAConfig;
  private lastBuyTimestamp: number = 0;
  private health = {
    isHealthy: true,
    eventsProcessed: 0,
    opportunitiesGenerated: 0,
    lastEventTimestamp: 0,
    errorCount: 0,
    lastError: undefined as string | undefined,
  };

  constructor(config: DCAConfig) {
    this.config = config;
  }

  name(): string {
    return `dca-${this.config.token}`;
  }

  strategyType(): string {
    return 'DCA';
  }

  subscribedEvents(): string[] {
    return [
      `market.price.${this.config.token}`,
      'market.slot.*',  // For time-based triggers
    ];
  }

  async processEvent(event: MarketEvent): Promise<TradeOpportunity[]> {
    this.health.eventsProcessed++;
    this.health.lastEventTimestamp = Date.now();

    const opportunities: TradeOpportunity[] = [];

    if (event.type === 'PriceUpdate') {
      const now = Date.now() / 1000;
      const timeSinceLastBuy = now - this.lastBuyTimestamp;

      // Check if it's time to buy
      if (timeSinceLastBuy >= this.config.intervalSeconds) {
        // Check price condition
        if (this.config.maxPrice && event.priceUsd > this.config.maxPrice) {
          return opportunities;  // Price too high, skip
        }

        // Check RSI condition (if configured)
        if (this.config.rsiThreshold) {
          const rsi = await this.calculateRSI(event.token);
          if (rsi > this.config.rsiThreshold) {
            return opportunities;  // RSI too high (overbought), skip
          }
        }

        // Generate DCA buy opportunity
        opportunities.push({
          id: `dca-${this.config.token}-${now}`,
          strategy: this.name(),
          strategyType: 'DCA',
          priority: 'Low',
          tradeType: 'DCA',
          tokenIn: new PublicKey('EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v'), // USDC
          tokenOut: new PublicKey(event.token),
          amountIn: this.config.amountPerInterval,
          expectedAmountOut: 0,  // Will be calculated by executor
          steps: [],
          expectedProfit: 0,  // DCA is not profit-seeking
          expectedProfitPct: 0,
          requireSimulation: true,
          maxSlippageBps: 100,  // 1% max slippage for DCA
          deadline: undefined,
          detectedAt: now,
          metadata: {
            dcaInterval: this.config.intervalSeconds,
            currentPrice: event.priceUsd,
          },
        });

        this.lastBuyTimestamp = now;
        this.health.opportunitiesGenerated++;
      }
    }

    return opportunities;
  }

  private async calculateRSI(token: string): Promise<number> {
    // Fetch historical prices and calculate RSI
    // Implementation details...
    return 50;  // Mock
  }

  config(): StrategyConfig {
    return this.config;
  }

  async updateConfig(config: StrategyConfig): Promise<void> {
    this.config = config as DCAConfig;
  }

  async start(): Promise<void> {
    console.log(`Starting DCA Strategy for ${this.config.token}`);
  }

  async stop(): Promise<void> {
    console.log(`Stopping DCA Strategy for ${this.config.token}`);
  }

  health() {
    return this.health;
  }
}
```

### 3.3 Grid Trading Strategy (TypeScript)

**Create:** `apps/strategies/grid-trading/src/index.ts`

```typescript
import { TradingStrategy, TradeOpportunity, MarketEvent } from '@trading-system/strategy-framework';
import { PublicKey } from '@solana/web3.js';

interface GridLevel {
  price: number;
  buyAmount: number;
  sellAmount: number;
  filled: boolean;
}

interface GridTradingConfig {
  enabled: boolean;
  token: string;
  priceRangeLow: number;  // $1.50
  priceRangeHigh: number; // $2.50
  gridLevels: number;     // 10 levels
  amountPerLevel: number; // 10 USDC per level
  rebalanceOnFill: boolean;
}

export class GridTradingStrategy implements TradingStrategy {
  private config: GridTradingConfig;
  private gridLevels: GridLevel[] = [];
  private currentPrice: number = 0;

  constructor(config: GridTradingConfig) {
    this.config = config;
    this.initializeGrid();
  }

  private initializeGrid() {
    const priceRange = this.config.priceRangeHigh - this.config.priceRangeLow;
    const priceStep = priceRange / this.config.gridLevels;

    for (let i = 0; i <= this.config.gridLevels; i++) {
      const price = this.config.priceRangeLow + (priceStep * i);
      this.gridLevels.push({
        price,
        buyAmount: this.config.amountPerLevel,
        sellAmount: this.config.amountPerLevel,
        filled: false,
      });
    }
  }

  name(): string {
    return `grid-${this.config.token}`;
  }

  strategyType(): string {
    return 'GridTrading';
  }

  subscribedEvents(): string[] {
    return [`market.price.${this.config.token}`];
  }

  async processEvent(event: MarketEvent): Promise<TradeOpportunity[]> {
    const opportunities: TradeOpportunity[] = [];

    if (event.type === 'PriceUpdate') {
      const newPrice = event.priceUsd;
      const oldPrice = this.currentPrice;
      this.currentPrice = newPrice;

      // Check if price crossed any grid levels
      for (const level of this.gridLevels) {
        // Price moved down through level → BUY
        if (oldPrice > level.price && newPrice <= level.price && !level.filled) {
          opportunities.push({
            id: `grid-buy-${level.price}-${Date.now()}`,
            strategy: this.name(),
            strategyType: 'GridTrading',
            priority: 'Medium',
            tradeType: 'GridOrder',
            tokenIn: new PublicKey('EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v'), // USDC
            tokenOut: new PublicKey(event.token),
            amountIn: level.buyAmount,
            expectedAmountOut: 0,
            steps: [],
            expectedProfit: 0,
            expectedProfitPct: 0,
            requireSimulation: true,
            maxSlippageBps: 50,
            deadline: undefined,
            detectedAt: Date.now() / 1000,
            metadata: {
              gridLevel: level.price,
              action: 'buy',
            },
          });

          level.filled = true;
        }

        // Price moved up through level → SELL
        if (oldPrice < level.price && newPrice >= level.price && level.filled) {
          opportunities.push({
            id: `grid-sell-${level.price}-${Date.now()}`,
            strategy: this.name(),
            strategyType: 'GridTrading',
            priority: 'Medium',
            tradeType: 'GridOrder',
            tokenIn: new PublicKey(event.token),
            tokenOut: new PublicKey('EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v'), // USDC
            amountIn: level.sellAmount,
            expectedAmountOut: 0,
            steps: [],
            expectedProfit: 0,
            expectedProfitPct: 0,
            requireSimulation: true,
            maxSlippageBps: 50,
            deadline: undefined,
            detectedAt: Date.now() / 1000,
            metadata: {
              gridLevel: level.price,
              action: 'sell',
            },
          });

          // Reset for rebalancing
          if (this.config.rebalanceOnFill) {
            level.filled = false;
          }
        }
      }
    }

    return opportunities;
  }

  // Implement other TradingStrategy methods...
  config() { return this.config; }
  async updateConfig(config: any) { this.config = config; }
  async start() { console.log('Starting Grid Trading Strategy'); }
  async stop() { console.log('Stopping Grid Trading Strategy'); }
  health() {
    return {
      isHealthy: true,
      eventsProcessed: 0,
      opportunitiesGenerated: 0,
      lastEventTimestamp: 0,
      errorCount: 0,
      lastError: undefined,
    };
  }
}
```

### 3.4 Long/Short Strategy (Rust)

**Create:** `rust/strategies/long-short/src/lib.rs`

```rust
use async_trait::async_trait;
use market_events::{MarketEvent, PriceUpdateEvent, LargeTradeEvent};
use strategy_framework::*;

pub struct LongShortStrategy {
    name: String,
    config: StrategyConfig,
    health: StrategyHealth,

    // Internal state
    positions: Vec<Position>,
    price_history: std::collections::VecDeque<f64>,
}

#[derive(Debug, Clone)]
struct Position {
    token: solana_sdk::pubkey::Pubkey,
    side: PositionSide,
    entry_price: f64,
    size: u64,
    opened_at: u64,
}

#[derive(Debug, Clone)]
enum PositionSide {
    Long,
    Short,
}

impl LongShortStrategy {
    pub fn new(config: StrategyConfig) -> Self {
        Self {
            name: "long-short".to_string(),
            config,
            health: StrategyHealth {
                is_healthy: true,
                events_processed: 0,
                opportunities_generated: 0,
                last_event_timestamp: 0,
                error_count: 0,
                last_error: None,
            },
            positions: Vec::new(),
            price_history: std::collections::VecDeque::with_capacity(100),
        }
    }

    /// Detect trend-based long opportunity
    fn detect_long_entry(&self, price: f64) -> Option<TradeOpportunity> {
        // Calculate moving averages
        let ma_20 = self.calculate_ma(20);
        let ma_50 = self.calculate_ma(50);

        // Golden cross: MA20 crosses above MA50 = LONG signal
        if price > ma_20 && ma_20 > ma_50 {
            return Some(TradeOpportunity {
                id: format!("long-{}", current_time_micros()),
                strategy: self.name.clone(),
                strategy_type: StrategyType::LongShort,
                priority: OpportunityPriority::Medium,
                trade_type: TradeType::LongPosition,
                token_in: solana_sdk::pubkey::Pubkey::default(), // USDC
                token_out: solana_sdk::pubkey::Pubkey::default(), // Token
                amount_in: 10_000_000, // 10 USDC
                expected_amount_out: 0,
                steps: vec![],
                expected_profit: 0,
                expected_profit_pct: 0.0,
                require_simulation: true,
                max_slippage_bps: 100,
                deadline: None,
                detected_at: current_time_micros(),
                metadata: serde_json::json!({
                    "signal": "golden_cross",
                    "ma_20": ma_20,
                    "ma_50": ma_50,
                }),
            });
        }

        None
    }

    /// Detect trend-based short opportunity
    fn detect_short_entry(&self, price: f64) -> Option<TradeOpportunity> {
        let ma_20 = self.calculate_ma(20);
        let ma_50 = self.calculate_ma(50);

        // Death cross: MA20 crosses below MA50 = SHORT signal
        if price < ma_20 && ma_20 < ma_50 {
            return Some(TradeOpportunity {
                id: format!("short-{}", current_time_micros()),
                strategy: self.name.clone(),
                strategy_type: StrategyType::LongShort,
                priority: OpportunityPriority::Medium,
                trade_type: TradeType::ShortPosition,
                token_in: solana_sdk::pubkey::Pubkey::default(), // Token
                token_out: solana_sdk::pubkey::Pubkey::default(), // USDC
                amount_in: 10_000_000,
                expected_amount_out: 0,
                steps: vec![],
                expected_profit: 0,
                expected_profit_pct: 0.0,
                require_simulation: true,
                max_slippage_bps: 100,
                deadline: None,
                detected_at: current_time_micros(),
                metadata: serde_json::json!({
                    "signal": "death_cross",
                    "ma_20": ma_20,
                    "ma_50": ma_50,
                }),
            });
        }

        None
    }

    fn calculate_ma(&self, periods: usize) -> f64 {
        if self.price_history.len() < periods {
            return 0.0;
        }

        let sum: f64 = self.price_history.iter().rev().take(periods).sum();
        sum / periods as f64
    }
}

#[async_trait]
impl TradingStrategy for LongShortStrategy {
    fn name(&self) -> &str {
        &self.name
    }

    fn strategy_type(&self) -> StrategyType {
        StrategyType::LongShort
    }

    fn subscribed_events(&self) -> Vec<String> {
        vec![
            "market.price.>".to_string(),
            "market.trade.large.>".to_string(),
        ]
    }

    async fn process_event(
        &mut self,
        event: &MarketEvent,
    ) -> Result<Vec<TradeOpportunity>, Box<dyn std::error::Error>> {
        self.health.events_processed += 1;
        self.health.last_event_timestamp = current_time_micros();

        let mut opportunities = Vec::new();

        match event {
            MarketEvent::PriceUpdate(price_event) => {
                // Update price history
                self.price_history.push_back(price_event.price_usd);
                if self.price_history.len() > 100 {
                    self.price_history.pop_front();
                }

                // Detect long/short opportunities
                if let Some(opp) = self.detect_long_entry(price_event.price_usd) {
                    opportunities.push(opp);
                }

                if let Some(opp) = self.detect_short_entry(price_event.price_usd) {
                    opportunities.push(opp);
                }
            }
            MarketEvent::LargeTrade(trade) => {
                // Large trade could indicate momentum
                // Could trigger backrunning or momentum-following trades
            }
            _ => {}
        }

        self.health.opportunities_generated += opportunities.len() as u64;

        Ok(opportunities)
    }

    fn config(&self) -> &StrategyConfig {
        &self.config
    }

    async fn update_config(&mut self, config: StrategyConfig) {
        self.config = config;
    }

    async fn start(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        println!("Starting Long/Short Strategy");
        Ok(())
    }

    async fn stop(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        println!("Stopping Long/Short Strategy");
        Ok(())
    }

    fn health(&self) -> StrategyHealth {
        self.health.clone()
    }
}

fn current_time_micros() -> u64 {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_micros() as u64
}
```

---

## 4. Strategy Manager (Orchestration)

**Create:** `rust/strategy-manager/src/lib.rs`

```rust
use async_nats::{jetstream, Client};
use market_events::MarketEvent;
use strategy_framework::{TradingStrategy, TradeOpportunity};
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct StrategyManager {
    strategies: Arc<RwLock<HashMap<String, Box<dyn TradingStrategy>>>>,
    nats_client: Client,
    js: jetstream::Context,
}

impl StrategyManager {
    pub async fn new(nats_url: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let client = async_nats::connect(nats_url).await?;
        let js = jetstream::new(client.clone());

        Ok(Self {
            strategies: Arc::new(RwLock::new(HashMap::new())),
            nats_client: client,
            js,
        })
    }

    /// Register a strategy
    pub async fn register_strategy(&self, strategy: Box<dyn TradingStrategy>) {
        let name = strategy.name().to_string();
        let mut strategies = self.strategies.write().await;
        strategies.insert(name.clone(), strategy);
        println!("Registered strategy: {}", name);
    }

    /// Start all strategies
    pub async fn start_all(&self) -> Result<(), Box<dyn std::error::Error>> {
        let mut strategies = self.strategies.write().await;

        for (name, strategy) in strategies.iter_mut() {
            if strategy.config().enabled {
                strategy.start().await?;
                println!("Started strategy: {}", name);
            }
        }

        Ok(())
    }

    /// Subscribe to market events and route to strategies
    pub async fn run(&self) -> Result<(), Box<dyn std::error::Error>> {
        let consumer = self
            .js
            .get_or_create_consumer("MARKET_EVENTS", jetstream::consumer::pull::Config {
                durable_name: Some("strategy-manager".to_string()),
                filter_subject: "market.>".to_string(),
                ..Default::default()
            })
            .await?;

        let messages = consumer.messages().await?;

        while let Some(msg) = messages.next().await {
            let msg = msg?;

            // Deserialize market event
            let event: MarketEvent = serde_json::from_slice(&msg.payload)?;

            // Route event to interested strategies
            self.route_event(event).await?;

            // Acknowledge message
            msg.ack().await?;
        }

        Ok(())
    }

    /// Route event to strategies that subscribed to it
    async fn route_event(&self, event: MarketEvent) -> Result<(), Box<dyn std::error::Error>> {
        let subject = event.nats_subject();
        let strategies = self.strategies.write().await;

        for (name, strategy) in strategies.iter() {
            // Check if strategy subscribed to this event type
            let subscriptions = strategy.subscribed_events();
            let matches = subscriptions.iter().any(|pattern| {
                self.subject_matches(&subject, pattern)
            });

            if matches && strategy.config().enabled {
                // Process event and generate opportunities
                match strategy.process_event(&event).await {
                    Ok(opportunities) => {
                        // Publish opportunities to NATS
                        for opp in opportunities {
                            self.publish_opportunity(&opp).await?;
                        }
                    }
                    Err(e) => {
                        eprintln!("Strategy {} error: {}", name, e);
                    }
                }
            }
        }

        Ok(())
    }

    /// Publish trade opportunity to NATS
    async fn publish_opportunity(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let subject = opportunity.nats_subject();
        let payload = serde_json::to_vec(opportunity)?;

        self.js
            .publish(subject, payload.into())
            .await?
            .await?;

        println!("Published opportunity: {} ({})", opportunity.id, opportunity.strategy);

        Ok(())
    }

    /// Simple wildcard matching for NATS subjects
    fn subject_matches(&self, subject: &str, pattern: &str) -> bool {
        if pattern.ends_with(".>") {
            let prefix = pattern.trim_end_matches(".>");
            subject.starts_with(prefix)
        } else if pattern == "*" {
            true
        } else {
            subject == pattern
        }
    }

    /// Get strategy health status
    pub async fn get_health(&self) -> HashMap<String, strategy_framework::StrategyHealth> {
        let strategies = self.strategies.read().await;
        strategies
            .iter()
            .map(|(name, strategy)| (name.clone(), strategy.health()))
            .collect()
    }
}
```

---

## 5. Universal Executor (Handles All Trade Types)

**Create:** `rust/universal-executor/src/lib.rs`

```rust
use strategy_framework::{TradeOpportunity, TradeType};
use solana_sdk::transaction::Transaction;

pub struct UniversalExecutor {
    rpc_client: solana_client::nonblocking::rpc_client::RpcClient,
}

impl UniversalExecutor {
    pub fn new(rpc_url: String) -> Self {
        Self {
            rpc_client: solana_client::nonblocking::rpc_client::RpcClient::new(rpc_url),
        }
    }

    /// Execute any trade opportunity
    pub async fn execute(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<String, Box<dyn std::error::Error>> {
        // 1. Risk checks (circuit breakers, position limits, etc.)
        self.check_risk(opportunity)?;

        // 2. Build transaction based on trade type
        let tx = self.build_transaction(opportunity).await?;

        // 3. Simulate if required
        if opportunity.require_simulation {
            let simulation_result = self.simulate_transaction(&tx).await?;
            if !simulation_result.profitable {
                return Err("Simulation shows unprofitable trade".into());
            }
        }

        // 4. Execute
        let signature = self.submit_transaction(tx).await?;

        Ok(signature)
    }

    async fn build_transaction(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<Transaction, Box<dyn std::error::Error>> {
        match opportunity.trade_type {
            TradeType::Arbitrage => {
                // Build multi-step arbitrage transaction
                self.build_arbitrage_tx(opportunity).await
            }
            TradeType::Swap | TradeType::DCA | TradeType::GridOrder => {
                // Build simple swap transaction
                self.build_swap_tx(opportunity).await
            }
            TradeType::LongPosition => {
                // Build leveraged long position (use Kamino, Drift, etc.)
                self.build_long_position_tx(opportunity).await
            }
            TradeType::ShortPosition => {
                // Build leveraged short position
                self.build_short_position_tx(opportunity).await
            }
            TradeType::ClosePosition => {
                // Close existing position
                self.build_close_position_tx(opportunity).await
            }
        }
    }

    async fn build_arbitrage_tx(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<Transaction, Box<dyn std::error::Error>> {
        // Implementation from previous Rust executor docs
        todo!()
    }

    async fn build_swap_tx(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<Transaction, Box<dyn std::error::Error>> {
        // Build simple swap via Jupiter or direct DEX
        todo!()
    }

    async fn build_long_position_tx(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<Transaction, Box<dyn std::error::Error>> {
        // Build leveraged long via Kamino, Drift, etc.
        todo!()
    }

    async fn build_short_position_tx(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<Transaction, Box<dyn std::error::Error>> {
        // Build leveraged short
        todo!()
    }

    async fn build_close_position_tx(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<Transaction, Box<dyn std::error::Error>> {
        // Close position
        todo!()
    }

    fn check_risk(
        &self,
        opportunity: &TradeOpportunity,
    ) -> Result<(), Box<dyn std::error::Error>> {
        // Circuit breaker checks
        // Position limit checks
        // Daily loss limit checks
        Ok(())
    }

    async fn simulate_transaction(
        &self,
        tx: &Transaction,
    ) -> Result<SimulationResult, Box<dyn std::error::Error>> {
        // Simulation logic
        Ok(SimulationResult { profitable: true, estimated_profit: 0 })
    }

    async fn submit_transaction(
        &self,
        tx: Transaction,
    ) -> Result<String, Box<dyn std::error::Error>> {
        // Jito bundle submission or direct TPU
        Ok("signature".to_string())
    }
}

struct SimulationResult {
    profitable: bool,
    estimated_profit: i64,
}
```

---

## 6. Configuration System

**Create:** `config/strategies.toml`

```toml
# Strategy configurations

[[strategies]]
name = "arbitrage"
type = "Arbitrage"
enabled = true
min_profit_bps = 30
require_simulation = false
language = "rust"

[[strategies]]
name = "arbitrage-usdc"
type = "Arbitrage"
enabled = true
min_profit_bps = 80  # Higher threshold for USDC
require_simulation = true  # Always simulate
language = "rust"

[[strategies]]
name = "dca-sol"
type = "DCA"
enabled = true
token = "SOL"
interval_seconds = 3600  # 1 hour
amount_per_interval = 10000000  # 10 USDC
max_price = 150.0  # Only buy if SOL < $150
language = "typescript"

[[strategies]]
name = "dca-jitosol"
type = "DCA"
enabled = true
token = "jitoSOL"
interval_seconds = 86400  # Daily
amount_per_interval = 50000000  # 50 USDC
rsi_threshold = 70  # Only buy if RSI < 70
language = "typescript"

[[strategies]]
name = "grid-jup"
type = "GridTrading"
enabled = true
token = "JUP"
price_range_low = 0.80
price_range_high = 1.20
grid_levels = 10
amount_per_level = 5000000  # 5 USDC per level
rebalance_on_fill = true
language = "typescript"

[[strategies]]
name = "long-short-sol"
type = "LongShort"
enabled = false  # Start disabled
token = "SOL"
max_leverage = 3
language = "rust"

[[strategies]]
name = "momentum-lst"
type = "Momentum"
enabled = false
tokens = ["jitoSOL", "mSOL", "bSOL"]
language = "rust"
```

---

## 7. Main Application

**Create:** `rust/trading-system/src/main.rs`

```rust
use strategy_manager::StrategyManager;
use strategies::{
    arbitrage::ArbitrageStrategy,
    long_short::LongShortStrategy,
};
use strategy_framework::StrategyConfig;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Load configuration
    let config = load_config("config/strategies.toml")?;

    // Initialize strategy manager
    let manager = StrategyManager::new("nats://localhost:4222").await?;

    // Register strategies
    for strategy_config in config.strategies {
        match strategy_config.strategy_type.as_str() {
            "Arbitrage" => {
                let strategy = Box::new(ArbitrageStrategy::new(strategy_config.into()));
                manager.register_strategy(strategy).await;
            }
            "LongShort" => {
                let strategy = Box::new(LongShortStrategy::new(strategy_config.into()));
                manager.register_strategy(strategy).await;
            }
            // TypeScript strategies loaded via separate process
            _ => {}
        }
    }

    // Start all strategies
    manager.start_all().await?;

    // Run event loop
    manager.run().await?;

    Ok(())
}

fn load_config(path: &str) -> Result<Config, Box<dyn std::error::Error>> {
    // Load TOML configuration
    todo!()
}

struct Config {
    strategies: Vec<StrategyConfigFile>,
}

struct StrategyConfigFile {
    name: String,
    strategy_type: String,
    enabled: bool,
    // ... other fields
}

impl From<StrategyConfigFile> for StrategyConfig {
    fn from(config: StrategyConfigFile) -> Self {
        StrategyConfig {
            enabled: config.enabled,
            min_profit_bps: 0,
            max_position_size: 0,
            require_simulation: false,
            custom_params: serde_json::Value::Null,
        }
    }
}
```

---

## Summary

This extensible architecture provides:

1. **Generic Market Events** - Reusable across all strategies
2. **Strategy Plugin System** - Easy to add new strategies (Rust or TypeScript)
3. **Event-Driven** - NATS for loose coupling
4. **Multiple Strategies** - Arbitrage, DCA, Grid Trading, Long/Short all working simultaneously
5. **Unified Executor** - Handles all trade types
6. **Hot Reloadable Config** - Update strategy parameters without restart
7. **Health Monitoring** - Per-strategy health and metrics

**Adding a new strategy:**
1. Implement `TradingStrategy` trait
2. Subscribe to relevant market events
3. Generate `TradeOpportunity` objects
4. Register with `StrategyManager`

**Timeline:**
- Core framework: 1-2 weeks
- Arbitrage strategy: Already done
- DCA strategy: 2-3 days
- Grid trading: 3-5 days
- Long/Short: 1 week

This gives you a production-grade extensible system that can grow with your needs!