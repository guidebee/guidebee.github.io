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

- **Five years** of Solana development experience building NFT platforms and DApps
- **Three deliberate prototypes** over two years, each answering specific questions:
  1. **Next.js Platform**: Explored problem space with 150+ CLI tools, learned CLI-first > web UI
  2. **TypeScript Bot**: Proved 5+ strategies work together, but Jupiter API (100-300ms) too slow
  3. **Polyglot System**: **Breakthrough** - local routing (2-10ms) is 10-30x faster, HFT possible
- **Production plan** combines: P1's CLI patterns + P2's strategy architecture + P3's performance
- **The real value**: Guaranteed learning and skill development, potential profit is upside

The production architecture outlined in my recent posts isn't aspirational - it's synthesizing proven patterns from three working prototypes built in evolutionary order.

## Five Years on Solana

I started working with Solana blockchain five years ago, building various projects including NFT platforms and other decentralized applications. These experiences gave me deep understanding of Solana's architecture, transaction anatomy, account models, and the broader ecosystem.

Over the past two years specifically, I've focused on trading systems, building three working prototypes. Each successfully executed real trades. Each revealed critical insights about performance, architecture, and what it takes to compete in Solana DeFi:

1. **Full-stack Next.js platform** - Web UI with 150+ CLI tools, NFT and OpenBook DEX support
2. **Mature TypeScript trading bot** - 18+ modules, 5+ trading strategies, 42+ CLI commands
3. **Polyglot arbitrage system** (TypeScript, Go, Rust) - 666+ files, microservices architecture

This was a deliberate evolution. I started with a web-first approach to explore the problem space, then built a dedicated trading bot with multiple strategies, and finally focused on performance with a polyglot system testing local routing. Each iteration answered specific questions and revealed what was needed for production:

![Arb Transactions](../images/arb-transactions.png)

**Prototype 1: Can I build a trading platform with web UI?**
- Answer: Yes, but web framework overhead isn't needed for trading core
- Key learning: CLI tools are more valuable than web UI for operations

**Prototype 2: Can multiple strategies work together effectively?**
- Answer: Yes, but monolithic TypeScript is too slow for competitive trading
- Key learning: Queue systems and wallet segregation are essential patterns

**Prototype 3: Can I achieve competitive performance with local routing?**
- Answer: Yes, using Go's SolRoute SDK (2-10ms vs 100-300ms Jupiter API)
- Key learning: Polyglot microservices unlock performance at critical paths

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

## The Chronological Evolution

Understanding the order matters because each prototype was a response to lessons learned:

**Phase 1 - Exploration (Next.js Platform):**
Started with web UI to support NFT minting and OpenBook DEX trading. Built 150+ CLI commands thinking web UI would be valuable. Reality: CLI tools became the real value, web framework added overhead. Trading operations need speed and automation, not dashboards.

**Phase 2 - Refinement (TypeScript Bot):**
Stripped away web framework, built dedicated trading bot with 5+ strategies (arbitrage, grid trading, DCA, limit orders, AI analysis). Implemented queue-based execution and multi-tier wallet segregation. Proved the architecture works. But Jupiter API latency (100-300ms per quote) made competitive HFT impossible.

**Phase 3 - Performance (Polyglot System):**
Built focused arbitrage prototype to test THE critical question: Can local routing change the game? Used Go's SolRoute SDK for local DEX math. Result: **2-10ms vs 100-300ms = 10-30x faster**. This proved sub-500ms execution is achievable. Intentionally limited to single strategy to isolate performance testing.

**Phase 4 - Production (Current):**
Now combining proven patterns: P1's CLI infrastructure + P2's multi-strategy architecture + P3's local routing performance = Production HFT system with competitive performance potential.

## The Three Trading Prototypes in Detail

### Prototype 1: Full-Stack Next.js Platform

**[View showcase on LinkedIn →](https://www.linkedin.com/posts/solana-playground_solana-playround-activity-6904417601364078592-PkUM)**

**What it does:**
- Web application for NFT minting and OpenBook DEX trading
- 150+ CLI commands for comprehensive operations
- Multiple trading protocols with web UI control
- Wallet management and transaction monitoring

**Architecture:**
- Next.js 14 web application with React 18
- Firebase backend for data persistence
- Comprehensive CLI toolkit built within web framework
- Multi-protocol integration (NFT, OpenBook DEX, SPL tokens)

**Key features:**
- Transaction retry patterns with timeout-based fallback
- Clean wallet abstraction (KeypairWallet)
- 7+ Helius RPC endpoints with automatic failover
- 40+ reusable React components
- Redis caching with 10-minute TTL

**What I learned:**
- CLI tools matter more than web UI for trading operations
- Transaction patterns: retry logic, multi-RPC fallback, confirmation polling
- RPC redundancy is critical (7+ endpoints with round-robin)
- Wallet abstraction enables extensibility (hardware wallets, multi-sig)
- Modular CLI structure (150+ files) beats monolithic commands

**Pain points:**
- Next.js overhead unnecessary for backend trading system
- Firebase dependency when PostgreSQL would be better
- Redux complexity for simpler state needs
- Web framework when CLI-first would be more appropriate
- Still using Jupiter API (100-300ms latency) without local routing consideration

### Prototype 2: Mature TypeScript Trading Bot

**What it does:**
- Five trading strategies: arbitrage, grid trading, DCA, limit orders, AI chart analysis
- Internal order book system with queue management
- Sophisticated wallet segregation (proxy/worker/controller/treasure)
- 42+ CLI operational commands
- Dedicated trading focus without web framework overhead

**Architecture:**
- Focused TypeScript application (no web framework)
- Redis-backed queue system (Bee-queue) for distributed execution
- MySQL for historical data persistence
- ChatGPT integration for TradingView chart analysis
- Built from lessons learned in Prototype 1

**Key features:**
- Multi-tier wallet management for risk isolation
- Expected balance tracking with automatic rebalancing
- Queue-based order management with TTL
- CSV export for P&L analysis
- Comprehensive operational tooling refined from Prototype 1

**What I learned:**
- Multiple strategies working together need conflict resolution
- Queue systems decouple order generation from execution
- Historical tracking is crucial for optimization
- Wallet segregation significantly improves risk management
- CLI-first design is superior to web-first for trading systems

**Pain points:**
- Still too slow for competitive HFT (~100-300ms Jupiter API latency)
- Monolithic architecture limits independent scaling
- Single-language stack (TypeScript) constrains performance optimization
- Need to test local routing vs API-based quoting

### Prototype 3: Polyglot Arbitrage System

**What it does:**
- Flash loan arbitrage across Raydium, Meteora, PumpSwap
- Local routing with Go's SolRoute SDK instead of Jupiter API
- Jito bundle submission for MEV protection
- Shredstream integration for 400ms early slot notifications
- Focused on performance testing, not strategy diversity

**Architecture:**
- TypeScript CLI tools for orchestration
- Go service for local DEX quoting (SolRoute SDK)
- Rust services for RPC proxy and transaction planning
- Docker infrastructure (Redis, PostgreSQL, MongoDB)
- Microservices architecture for independent optimization

**Key metrics (measured, not estimated):**
- SolRoute quoting: **2-10ms** for known pools (vs 100-300ms Jupiter API)
- Jupiter API fallback: 100-300ms (kept for unknown pools)
- Shredstream latency: 100-200ms vs 500-1000ms websockets
- Transaction confirmation: 10-30s via Jito

**What I learned:**
- **Local routing changes the game**: 2-10ms vs 100-300ms is 10-30x faster
- Hybrid quoting (local + API fallback) provides best of both worlds
- Address Lookup Tables reduce transaction size by 30-40%
- Template caching with placeholders speeds up repeated routes
- Health monitoring with automatic failover is essential
- Polyglot architecture enables choosing the right tool per component
- Rate limiting is real (60 Jupiter API calls/minute, avoided with local)

**Pain points:**
- Single strategy focus (arbitrage only) - intentional for performance testing
- Needed to prove local routing performance before committing to full rebuild
- Complex deployment with multiple languages and services

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

## The Evolution: From Exploration to Performance

**The learning progression was deliberate:**

**Prototype 1 (Next.js Platform):** Started with web UI to explore NFT and DEX trading, built 150+ CLI tools. Learned that CLI-first design matters more than web UI for trading systems. Proved transaction patterns and RPC failover strategies work.

**Prototype 2 (TypeScript Bot):** Built dedicated trading bot without web framework overhead. Implemented 5+ strategies with queue-based execution and wallet segregation. Proved multiple strategies can work together, but discovered Jupiter API latency (100-300ms) was the bottleneck preventing competitive performance.

**Prototype 3 (Polyglot System):** Focused specifically on performance with local routing using Go's SolRoute SDK. Intentionally limited to arbitrage to isolate and measure performance improvements. Proved local routing (2-10ms) vs API (100-300ms) is 10-30x faster and competitive.

## What Makes This Time Different?

**Why not just optimize one of the existing prototypes?**

Because each prototype revealed fundamental architectural truths:

- Prototype 1: Great CLI patterns and transaction logic, but web framework overhead unnecessary
- Prototype 2: Great strategy diversity and architecture, but too slow without local routing
- Prototype 3: Great performance with local routing, but single strategy focus (intentional for testing)

The production system combines the best of all three in correct order:
- Comprehensive CLI tooling and transaction patterns (Prototype 1)
- Multiple strategies with queue-based execution and wallet segregation (Prototype 2)
- Polyglot microservices with local routing for performance (Prototype 3)

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

Each prototype contributed essential patterns to the production architecture, in evolutionary order:

**From Prototype 1 (Next.js Platform):**
- Transaction retry patterns with timeout-based fallback
- RPC endpoint redundancy strategies (7+ Helius endpoints)
- Clean wallet abstractions for extensibility
- Comprehensive CLI tooling foundation (150+ commands)
- Redis caching patterns
- Proof that web UI adds overhead without corresponding value for trading

**From Prototype 2 (TypeScript Bot):**
- Multiple trading strategies working together (5+ strategies)
- Queue-based execution for reliability and conflict resolution
- Wallet segregation architecture (proxy/worker/controller/treasure)
- Historical tracking and P&L analysis with MySQL
- Refined CLI tooling (42+ commands)
- Proof that monolithic TypeScript + Jupiter API can't achieve HFT speeds

**From Prototype 3 (Polyglot System):**
- **THE KEY INSIGHT:** Local routing (2-10ms) vs Jupiter API (100-300ms) = 10-30x faster
- Microservices architecture with language-specific optimization
- Go for high-performance quoting (SolRoute SDK)
- Rust for RPC proxy and transaction planning
- Docker-based infrastructure approach
- Prometheus + Grafana monitoring foundation
- Proof that sub-500ms execution is achievable with local routing

## The Comparison Matrix

| Capability | Prototype 1<br/>(Next.js) | Prototype 2<br/>(TypeScript Bot) | Prototype 3<br/>(Polyglot) | Production Plan |
|-----------|-------------|-------------|-------------|-----------------|
| **Quote speed** | 100-300ms (Jupiter) | 100-300ms (Jupiter) | **2-10ms (Go local)** | 2-10ms (Go) |
| **Strategies** | Multiple (web UI) | 5+ strategies | 1 (arb only) | Multiple + conflict resolution |
| **Wallet mgmt** | Abstraction | Multi-tier | Basic | Multi-tier + abstraction |
| **Data tracking** | Firebase | MySQL + CSV | None | PostgreSQL + analytics |
| **CLI tools** | 150+ | 42+ (refined) | Basic | Comprehensive + modular |
| **RPC failover** | 7+ Helius | Round-robin | 95+ endpoints | Multi-provider redundancy |
| **Retry logic** | Timeout fallback | Advanced | Basic | Advanced + multi-RPC |
| **Monitoring** | Firebase | CSV | Prom+Grafana | Prom+Grafana+Jaeger+Loki |
| **Architecture** | Full-stack web | Monolithic | Microservices | Event-driven microservices |
| **Event system** | None | None | None | NATS JetStream |
| **Focus** | Exploration | Strategy diversity | **Performance** | Production HFT |

**Key evolution:** Prototype 1 explored the problem space with web UI. Prototype 2 proved multiple strategies work together but was too slow. Prototype 3 proved local routing achieves 10-30x faster quotes (2-10ms vs 100-300ms), making HFT competitive performance possible.

The production system combines: CLI patterns (P1) + strategy diversity and wallet architecture (P2) + polyglot performance with local routing (P3) + new event-driven infrastructure (NATS).

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

The production system outlined in my recent posts isn't starting from scratch. It's synthesizing five years of Solana experience and proven patterns from three working prototypes, built in deliberate order:

**Proven from Prototype 1 (Next.js Platform):**
- Transaction retry patterns with timeout fallback
- Multi-RPC failover (7+ Helius endpoints)
- Clean wallet abstractions
- Comprehensive CLI tooling (150+ commands)
- **Lesson: Web UI overhead unnecessary, CLI-first is correct approach**

**Proven from Prototype 2 (TypeScript Bot):**
- Multiple strategies working together (5+ strategies)
- Queue-based execution with conflict resolution
- Wallet segregation (proxy/worker/controller/treasure)
- Historical tracking and P&L analysis
- **Lesson: Architecture works, but Jupiter API latency prevents HFT**

**Proven from Prototype 3 (Polyglot System):**
- **THE GAME CHANGER:** Local routing 2-10ms (Go SolRoute SDK) vs 100-300ms (Jupiter API)
- Flash loan arbitrage with capital efficiency
- Jito bundles for MEV protection
- Microservices architecture for independent optimization
- **Lesson: Local routing makes sub-500ms execution achievable**

**New additions for production:**
- Event-driven architecture (NATS JetStream) for service coordination
- Comprehensive observability (Prometheus, Grafana, Jaeger, Loki)
- Unified cross-language patterns (Rust, Go, TypeScript)
- Kill switch infrastructure for safety
- Proper testing and CI/CD

**The synthesis:** Combine Prototype 1's CLI patterns + Prototype 2's strategy architecture + Prototype 3's performance breakthrough = Production HFT system that can compete.

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
