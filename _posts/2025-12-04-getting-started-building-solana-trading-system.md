---
layout: single
title: "Getting Started: Building a Solana Trading System from Prototype to Production"
date: 2025-12-04
categories:
  - solana
  - trading
  - development
tags:
  - solana
  - dex
  - arbitrage
  - golang
  - typescript
  - hft
excerpt: "Starting a journey to build a production-grade High-Frequency Trading (HFT) system on Solana. Currently have working arbitrage bots (~1.7s execution), aiming for sub-500ms execution with local pool math, Shredstream integration, and flash loans."
---

## TL;DR

Starting a journey to build a production-grade High-Frequency Trading (HFT) system on Solana. Currently have working arbitrage bots (~1.7s execution), aiming for sub-500ms execution with local pool math, Shredstream integration, and flash loans. This post covers the project structure, technology choices, and immediate goals.

## Context

I've been experimenting with Solana DEX arbitrage and have several working prototypes that successfully find and execute arbitrage opportunities. However, they're too slow for competitive HFT:

- **Current execution time:** 1.7-2.5 seconds end-to-end
- **Main bottleneck:** Quote latency via Jupiter API (100-300ms per quote)
- **Competition:** Need sub-500ms to catch profitable opportunities

The prototypes work, but I need to optimize and productionize them. This blog will document that journey.

## The Challenge

Transform working but slow arbitrage bots into a competitive HFT system:

**Current State:**
- ✅ Working arbitrage via Jupiter quote/swap
- ✅ Flash loan support (Kamino) for capital efficiency
- ✅ Jito Shredstream support (fast slot notifications)
- ✅ Local pool math started (Go SolRoute SDK)
- ⏱️ Too slow to compete (1.7-2.5s execution)

**Target State:**
- 🎯 Sub-500ms execution (< 200ms ideal)
- 🎯 Local pool math for all quotes (< 10ms)
- 🎯 Shredstream integration (400ms early alpha)
- 🎯 Multi-wallet parallel execution
- 🎯 95%+ Jito bundle landing rate

## Project Structure

This is a polyglot monorepo with three main technology stacks:

### 1. Go Stack (`go/`)
High-performance DEX routing and quote generation using the **SolRoute SDK**:
- Direct blockchain interaction (no third-party APIs)
- Supports Raydium (AMM, CLMM, CPMM), PumpSwap, Meteora DLMM
- Quote service returns cached quotes in milliseconds
- Uses Go workspace for module management

**Key Commands:**
```bash
# Run quote service
go run ./go/cmd/quote-service/main.go -port 8080

# Get a quote via CLI
go run ./go/cmd/quote/main.go -input SOL -output USDC -amount 1000000000

# Build everything
go build ./...
```

### 2. TypeScript Stack (`ts/`)
Turborepo monorepo with Next.js apps and shared packages:
- `web` - Monitoring dashboard (Next.js 16)
- `scanners` - Market data collection and event monitoring
- `@repo/ui` - Shared React components
- Uses pnpm workspaces for package management

**Key Commands:**
```bash
cd ts
pnpm install
pnpm dev          # Run all apps
pnpm build        # Build with Turborepo caching
```

### 3. Rust Stack (`rust/`)
Performance-critical services:
- `solana-rpc-proxy` - High-performance RPC proxy with load balancing
- `instruction-plans` - Transaction planning and optimization
- `jupiter/aggregator` - Jupiter program bindings
- Uses Cargo workspace

**Key Commands:**
```bash
cd rust
cargo build --release
cargo run -p solana-rpc-proxy --release
```

### 4. Infrastructure (`deployment/docker/`)
Complete observability stack:
- **Data:** Redis, PostgreSQL, NATS JetStream
- **Monitoring:** Prometheus, Grafana, Jaeger, Loki
- **Services:** Quote service, RPC proxy

```bash
cd deployment/docker
docker-compose up -d
```

## Technology Choices for HFT

After analyzing the prototypes and requirements, here's the stack:

| Component | Technology | Why |
|-----------|-----------|-----|
| **Quote Engine** | Go (SolRoute SDK) | < 10ms local pool math, concurrent calculations |
| **Market Data** | Rust (Shredstream) | Zero-copy parsing, 400ms early alpha |
| **Execution** | TypeScript | Jito SDK, flash loan integration, rapid iteration |
| **Cache** | Redis in-memory | Sub-millisecond access for hot data |
| **Message Bus** | NATS JetStream | Event-driven, exactly-once delivery |
| **Metrics** | Prometheus + Grafana | Time-series monitoring |
| **Tracing** | Jaeger | Distributed tracing for bottleneck analysis |

## My Immediate Plan

Following the optimization guide documented in my project, I'm tackling this in phases:

### Phase 1: Quick Wins (This Week)

Target: 1.7s → 800ms (2x improvement in 4 hours)

- [x] Set up blog to document progress
- [ ] Cache blockhash (saves 50ms)
- [ ] Parallel quote fetching (2x faster)
- [ ] Batch RPC calls for pool data (10x faster)

### Phase 2: Core Optimizations (Weeks 1-4)

Target: 800ms → 200ms (8x total improvement)

**Week 1:** Local quote engine (Go/Rust)
- Replace Jupiter API with local pool math
- Expected: 300ms → 10ms per quote

**Week 2:** Shredstream integration
- Get slot notifications 400ms early
- Build transaction immediately on slot

**Week 3:** Flash loan optimization
- Zero capital requirements
- Atomic arbitrage trades

**Week 4:** Jito bundle optimization
- Tip calculation strategies
- 95%+ landing rate

### Phase 3: Advanced Strategies (Weeks 5-8)

- Triangular arbitrage (3-leg trades)
- Cross-DEX arbitrage
- Statistical arbitrage
- Optional: Market making

## Project Documentation

The project includes comprehensive documentation covering:

### Core Documentation Areas

1. **Prototype Analysis** - Analysis of existing prototypes and what works
2. **Production Architecture Plan** - Scanner → Planner → Executor architecture
3. **Solo Developer Roadmap** - Realistic roadmap (MVP in 4 weeks)
4. **HFT Architecture** - Sub-500ms execution architecture design
5. **Optimization Guide** - Week-by-week optimization strategies

### Reference Implementations

The project includes three working prototype systems:

- Polyglot arbitrage system with Go quote service
- Mature TypeScript trading bot with proven patterns
- Full-stack Next.js app with 150+ CLI tools

These prototypes demonstrate proven patterns for transaction building, retry logic, wallet management, and more.

## Development Setup

Getting everything running is straightforward:

```bash
# 1. Install dependencies
go work sync              # Go workspace
cd ts && pnpm install    # TypeScript packages
cd rust && cargo check   # Rust dependencies

# 2. Configure environment
cp .env.example .env     # Main config
cp go/.env.example go/.env  # Go service config

# 3. Start infrastructure
cd deployment/docker
docker-compose up -d

# 4. Access services
# Grafana: http://localhost:3000
# Prometheus: http://localhost:9090
# Jaeger: http://localhost:16686
# Quote Service: http://localhost:8080
```

## Key Learnings So Far

1. **Prototypes are valuable** - Three different prototype approaches provided insights into what works and what doesn't
2. **Monorepo with native tooling** - Go workspace, pnpm/Turborepo, Cargo workspace - each stack uses its native tools
3. **Observability from day one** - Having Prometheus, Grafana, and Jaeger set up early makes optimization data-driven
4. **Local pool math is critical** - 300ms API latency kills HFT competitiveness; local calculations can hit sub-10ms

## Challenges Ahead

**Technical Challenges:**
- Achieving sub-10ms quote latency with local pool math
- Integrating Shredstream for 400ms early alpha
- Optimizing Jito bundle submission (current: ~50% landing, target: 95%+)
- Managing multiple wallets in parallel without nonce collisions

**Learning Challenges:**
- Deep dive into Solana transaction anatomy and MEV strategies
- Understanding AMM math across different DEX types (constant product, CLMM, DLMM)
- Mastering account derivation and PDAs
- Binary encoding (Borsh, little-endian) for on-chain data

**Operational Challenges:**
- Risk management and circuit breakers
- Cost management (~$300-650/month infrastructure)
- Monitoring and alerting for 24/7 operation

## Next Steps

This week I'm focusing on:

1. **[Today]** Set up this blog structure ✅
2. **[Tomorrow]** Profile current bot to identify bottlenecks
3. **[This week]** Implement quick wins (cache blockhash, parallel quotes, batch RPC)
4. **[Next post]** Document baseline performance and first optimizations

## Resources I'm Using

**Documentation:**
- [Solana Cookbook](https://solanacookbook.com/) - Patterns and best practices
- [Raydium SDK](https://github.com/raydium-io/raydium-sdk) - AMM implementation details
- [Jupiter Docs](https://docs.jup.ag/) - Swap aggregation reference
- [Jito Labs](https://www.jito.wtf/docs/) - MEV protection and bundles

**Tools:**
- [Solana Explorer](https://explorer.solana.com/) - Transaction debugging
- [Solscan](https://solscan.io/) - Account inspection
- [Birdeye](https://birdeye.so/) - Price charts and analytics

**Communities:**
- Solana Discord - Technical questions
- Jito Discord - Bundle submission help
- Twitter/X - Following MEV researchers

---

**Next Post:** Week 1 - Profiling and Quick Wins

## Connect

- **GitHub:** [github.com/guidebee](https://github.com/guidebee)
- **LinkedIn:** [linkedin.com/in/guidebee](https://linkedin.com/in/guidebee)

---

*This is post #1 in the Solana Trading System development series. Follow along as I document the journey from working prototype to production HFT system.*
