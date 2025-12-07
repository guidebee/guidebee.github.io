---
layout: single
title: "The Research Behind the Plan: Two Years and Three Prototypes"
date: 2025-12-07
permalink: /posts/2025/12/research-behind-plan-prototypes/
categories:
  - blog
tags:
  - solana
  - trading
  - research
  - prototypes
  - architecture
  - lessons-learned
excerpt: "Five years of Solana development and three working trading prototypes inform the production system. This post provides context on that research journey, the lessons learned, and why the challenge itself is valuable regardless of outcome."
---

## TL;DR

The production architecture outlined in my [getting started post](/posts/2025/12/getting-started-building-solana-trading-system/), [event system design](/posts/2025/12/cross-language-event-system-for-solana-trading/), and [observability infrastructure](/posts/2025/12/observability-and-monitoring-solana-trading-system/) isn't aspirational - it's built on five years of Solana development experience and intensive research with three trading prototypes. This post provides context on that journey and why it's worthwhile regardless of the outcome.

## Five Years on Solana

I started working with Solana blockchain five years ago, building various projects including NFT platforms and other decentralized applications. These experiences gave me deep understanding of Solana's architecture, transaction anatomy, account models, and the broader ecosystem.

Over the past two years specifically, I've focused on trading systems, building three working prototypes. Each successfully executed real trades. Each revealed critical insights about performance, architecture, and what it takes to compete in Solana DeFi:

1. **Polyglot arbitrage system** (TypeScript, Go, Rust) - 666+ files, microservices architecture
2. **Mature TypeScript trading bot** - 18+ modules, 5+ trading strategies, 42+ CLI commands
3. **Full-stack Next.js platform** - 150+ CLI tools, comprehensive web UI, multiple protocols

This wasn't a linear journey. I started with one approach, learned from its limitations, built another, discovered new patterns, and evolved to a third. Each iteration answered specific questions:

![Arb Transactions](../images/arb-transactions.png)

**Prototype 1: Can I build a working arbitrage bot?**
- Answer: Yes, but it's too slow (~1.7-2.5s execution)
- Key learning: Jupiter API latency (100-300ms) kills competitiveness

**Prototype 2: Can I make it faster with local pool math?**
- Answer: Yes, using Go's SolRoute SDK (2-10ms quotes)
- Key learning: Language choice matters for performance-critical paths

**Prototype 3: Can I scale this to multiple strategies?**
- Answer: Yes, but architecture must support distributed execution
- Key learning: Queue systems and wallet segregation are essential

## The Foundation: Early Solana Projects

Before diving into trading systems, I spent three years building various Solana projects:

**NFT Projects:**
- Built NFT minting and marketplace platforms
- Learned Metaplex program interactions and token metadata standards
- Gained experience with high-traffic transaction processing
- Understood SPL token standards and account structures

**DeFi Explorations:**
- Integrated with major protocols (Raydium, Orca, Serum)
- Built wallet management systems
- Implemented RPC failover and retry strategies
- Learned transaction confirmation patterns

These projects weren't about trading, but they taught me fundamental Solana concepts that became crucial for building trading systems: account rent, program-derived addresses (PDAs), compute units, transaction size limits, and the importance of RPC infrastructure.

## The Three Trading Prototypes in Detail

### Prototype 1: Polyglot Arbitrage System

**What it does:**
- Flash loan arbitrage across Raydium, Meteora, PumpSwap
- Jito bundle submission for MEV protection
- Shredstream integration for 400ms early slot notifications

**Architecture:**
- TypeScript CLI tools for orchestration
- Go service for local DEX quoting (SolRoute)
- Rust services for RPC proxy and transaction planning
- Docker infrastructure (Redis, PostgreSQL, MongoDB)

**Key metrics (measured, not estimated):**
- SolRoute quoting: 2-10ms for known pools
- Jupiter API fallback: 100-300ms
- Shredstream latency: 100-200ms vs 500-1000ms websockets
- Transaction confirmation: 10-30s via Jito

**What I learned:**
- Hybrid quoting (local + API fallback) works
- Address Lookup Tables reduce transaction size by 30-40%
- Template caching with placeholders speeds up repeated routes
- Health monitoring with automatic failover is essential
- Rate limiting is a real constraint (60 Jupiter API calls/minute)

**Pain points:**
- Single strategy focus (arbitrage only)
- No historical performance tracking
- Hardcoded configuration values
- Limited operational tooling

### Prototype 2: Mature TypeScript Trading Bot

**What it does:**
- Five trading strategies: arbitrage, grid trading, DCA, limit orders, AI chart analysis
- Internal order book system with queue management
- Sophisticated wallet segregation (proxy/worker/controller/treasure)
- 42+ CLI operational commands

**Architecture:**
- Monolithic TypeScript application (666 files)
- Redis-backed queue system (Bee-queue) for distributed execution
- MySQL for historical data persistence
- ChatGPT integration for TradingView chart analysis

**Key features:**
- Multi-tier wallet management for risk isolation
- Expected balance tracking with automatic rebalancing
- Queue-based order management with TTL
- CSV export for P&L analysis
- Comprehensive operational tooling

**What I learned:**
- Multiple strategies need conflict resolution
- Queue systems decouple order generation from execution
- Historical tracking is crucial for optimization
- Wallet segregation significantly improves risk management
- CLI tooling is more valuable than initially thought

**Pain points:**
- Monolithic architecture makes scaling difficult
- No high-performance local quoting
- Limited monitoring and observability
- Single-language stack (TypeScript only)

### Prototype 3: Full-Stack Next.js Platform

**What it does:**
- Web application for trade monitoring and management
- 150+ CLI commands for comprehensive operations
- Multiple trading strategies with UI control
- OpenBook market maker implementation

**Architecture:**
- Next.js 14 web application with React 18
- Firebase backend for data persistence
- Comprehensive CLI toolkit
- Multi-chain protocol integration

**Key features:**
- Transaction retry patterns with timeout-based fallback
- Clean wallet abstraction (KeypairWallet)
- 7+ Helius RPC endpoints with automatic failover
- 40+ reusable React components
- Redis caching with 10-minute TTL

**What I learned:**
- Transaction patterns: retry logic, multi-RPC fallback, confirmation polling
- RPC redundancy is critical (7+ endpoints with round-robin)
- Wallet abstraction enables extensibility (hardware wallets, multi-sig)
- Web UI adds operational overhead but improves visibility
- Modular CLI structure (150+ files) beats monolithic commands

**Pain points:**
- Next.js overhead for backend trading system
- Firebase dependency when PostgreSQL might be better
- Redux complexity for simpler state needs

## The Research-Backed Performance Claims

When I mention sub-500ms execution, I'm not guessing. Here's the data from Prototype 1:

**Current measured latency breakdown (1.7-2.5s total):**
- Market event detection: 100-200ms (Shredstream)
- Quote generation: 100-300ms (Jupiter API) or 2-10ms (local SolRoute)
- Transaction building: 50-100ms
- Signing and submission: 10-30ms (Jito bundle)
- Confirmation monitoring: 10-30s (confirmation, not execution)

**Optimization path to sub-500ms:**
- Market event: 100-200ms (unchanged, Shredstream is optimal)
- Quote generation: 2-10ms (eliminate Jupiter API, use only local)
- Transaction building: 10-20ms (optimize with caching, templates)
- Signing and submission: 10-30ms (unchanged, network bound)
- **Total: 122-260ms** (conservative: 150-300ms range)

The sub-500ms claim isn't aspirational. It's what happens when you eliminate the Jupiter API bottleneck and optimize transaction building. I've measured every component.

## What Makes This Time Different?

**Why not just optimize one of the existing prototypes?**

Because each prototype revealed fundamental architectural limitations:

- Prototype 1: Great performance core, but single strategy focus
- Prototype 2: Great strategy diversity, but monolithic and slow
- Prototype 3: Great tooling and UX, but too much overhead

The production system combines the best of all three:
- Polyglot microservices (Prototype 1)
- Multiple strategies with queue-based execution (Prototype 2)
- Comprehensive CLI tooling and transaction patterns (Prototype 3)

Plus new capabilities none of them had:
- NATS JetStream for event-driven architecture
- Production observability (Prometheus, Grafana, Jaeger)
- Proper testing and CI/CD
- Documentation for maintainability

## The Real Value: Knowledge, Not Guaranteed Profit

Here's what I want to be crystal clear about: **I don't know if this will be profitable.**

The Solana trading landscape is brutally competitive. MEV bots with institutional resources operate at speeds I may never match. Market conditions change. Competition evolves.

**But here's what I do know:**

Even if this project doesn't generate the returns I hope for, I will have gained:

**Technical skills:**
- Deep understanding of Solana transaction anatomy
- Expertise in AMM math across DEX types (constant product, CLMM, DLMM)
- Production polyglot architecture experience
- High-frequency system design patterns
- Real-world distributed systems at scale

**Practical knowledge:**
- RPC optimization and failover strategies
- MEV protection and bundle submission techniques
- Wallet management and risk isolation
- Transaction retry and confirmation patterns
- Operational monitoring and alerting

**Career value:**
- Open-source portfolio demonstrating real-world skills
- Blog documenting detailed problem-solving
- Network connections in Solana ecosystem
- Understanding of DeFi protocols and trading systems

## Why These Prototypes Matter

Each prototype contributed essential patterns to the production architecture:

**From Prototype 1 (Polyglot System):**
- Microservices architecture with language-specific optimization
- Go for high-performance quoting (2-10ms)
- Rust for RPC proxy and transaction planning
- Docker-based infrastructure approach
- Prometheus + Grafana monitoring foundation

**From Prototype 2 (TypeScript Bot):**
- Multiple trading strategies working together
- Queue-based execution for reliability
- Wallet segregation (proxy/worker/controller/treasure)
- Historical tracking and P&L analysis
- Comprehensive CLI tooling (42+ commands)

**From Prototype 3 (Next.js Platform):**
- Transaction retry patterns with fallback
- RPC endpoint redundancy strategies
- Clean wallet abstractions
- Even more CLI tools (150+ commands)
- Web UI patterns (optional for monitoring)

## The Comparison Matrix

| Capability | Prototype 1 | Prototype 2 | Prototype 3 | Production Plan |
|-----------|-------------|-------------|-------------|-----------------|
| **Quote speed** | 2-10ms (Go) | 100-300ms | 100-300ms | 2-10ms (Go) |
| **Strategies** | 1 | 5+ | 6+ | Multiple + conflict resolution |
| **Wallet mgmt** | Basic | Multi-tier | Abstraction | Multi-tier + abstraction |
| **Data tracking** | None | MySQL + CSV | Firebase | PostgreSQL + analytics |
| **CLI tools** | Basic | 42+ | 150+ | Comprehensive + modular |
| **RPC failover** | 95+ endpoints | Round-robin | 7+ Helius | Multi-provider redundancy |
| **Retry logic** | Basic | Advanced | Timeout | Advanced + multi-RPC |
| **Monitoring** | Prom+Grafana | CSV | Firebase | Prom+Grafana+Jaeger+Loki |
| **Architecture** | Microservices | Monolithic | Full-stack | Event-driven microservices |
| **Event system** | None | None | None | NATS JetStream |

The production system combines the best capabilities from each prototype while adding new infrastructure like NATS JetStream for event-driven coordination.

## The Value Proposition

Here's the reality: I don't know if this will be profitable. Solana DeFi is brutally competitive, with institutional players operating at speeds and scales I may never match. Market conditions change. Competition evolves.

But profitability isn't the only measure of success. Even if this doesn't generate expected returns, the knowledge gained is substantial:

**Technical expertise:**
- Deep Solana protocol knowledge (5 years and counting)
- High-performance systems design across three languages
- Production polyglot microservices architecture
- Event-driven distributed systems
- Real-world MEV strategies and optimizations

**Practical experience:**
- Three working prototypes with real trade execution
- Measured performance data across components
- RPC optimization and multi-provider failover
- Transaction patterns that work in production
- Operational monitoring and alerting

**Career assets:**
- Open-source portfolio demonstrating capabilities
- Detailed blog documenting problem-solving approach
- Network connections in Solana ecosystem
- Comprehensive documentation of complex systems

**Personal satisfaction:**
- I genuinely enjoy tackling challenging problems
- The satisfaction of resolving difficult issues after sustained effort
- Building complex systems that work
- Learning through doing, not just reading

The learning is guaranteed. The profit is potential upside. And honestly, I find the challenge itself rewarding - there's real satisfaction in solving hard problems, optimizing latency from seconds to milliseconds, or debugging why a transaction fails. That intrinsic motivation matters more than any outcome.

## From Prototypes to Production

The production system outlined in my recent posts isn't starting from scratch. It's synthesizing five years of Solana experience and proven patterns from three working prototypes:

**Core infrastructure already proven:**
- Local pool math: 2-10ms quotes (Go SolRoute SDK)
- Flash loan arbitrage: Capital-efficient execution
- Jito bundles: MEV protection and prioritization
- Multiple strategies: Queue-based conflict resolution
- Wallet segregation: Risk management patterns
- Transaction retry: Multi-RPC fallback strategies

**New additions for production:**
- Event-driven architecture (NATS JetStream)
- Comprehensive observability (Prometheus, Grafana, Jaeger, Loki)
- Unified cross-language patterns (Rust, Go, TypeScript)
- Kill switch infrastructure for safety
- Proper testing and CI/CD

The challenge now is combining these proven components into a cohesive system that can compete in production DeFi markets.

## What's Next

The [getting started post](/posts/2025/12/getting-started-building-solana-trading-system/) outlined the project structure and goals. The [event system](/posts/2025/12/cross-language-event-system-for-solana-trading/) and [observability infrastructure](/posts/2025/12/observability-and-monitoring-solana-trading-system/) posts showed the foundation being built.

Future posts will document:
- Performance profiling and optimization results
- Scanner service implementation (market data ingestion)
- Executor service development (trade execution)
- Real operational challenges and solutions
- Lessons learned from production deployment

## Conclusion

This is what research-backed development looks like: five years of Solana experience, three working prototypes, measured performance data, and documented lessons learned. The production system combines proven patterns rather than inventing new ones.

Will it be profitable? The market will decide. Will I build something substantial and learn extensively along the way? Absolutely. And I'll enjoy solving the hard problems regardless of the outcome.

---

**Related Posts:**
- [Getting Started: Building a Solana Trading System](/posts/2025/12/getting-started-building-solana-trading-system/) - Project overview and goals
- [Building a Cross-Language Event System](/posts/2025/12/cross-language-event-system-for-solana-trading/) - NATS JetStream implementation
- [Adding Observability and Monitoring](/posts/2025/12/observability-and-monitoring-solana-trading-system/) - Prometheus, Grafana, Jaeger, Loki

**Technical Documentation:**
- [Prototype Analysis](https://github.com/guidebee/solana-trading-system/blob/main/docs/01-prototype-analysis.md) - Detailed technical breakdown
- [Production Architecture Plan](https://github.com/guidebee/solana-trading-system/blob/main/docs/02-production-architecture-plan.md) - System design


## Connect

- **GitHub:** [github.com/guidebee](https://github.com/guidebee)
- **LinkedIn:** [linkedin.com/in/guidebee](https://linkedin.com/in/guidebee)

---

*This is post #4 in the Solana Trading System development series. Follow along as I document the journey from working prototypes to production HFT system.*
