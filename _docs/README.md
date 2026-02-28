---
layout: single
title: "Solana Trading System Documentation"
permalink: /docs/readme/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Solana Trading System Documentation

## Overview

This directory contains comprehensive documentation for building a production-ready Solana trading system, from prototype analysis to deployment architecture.

---

## 📚 Documentation Index

### Getting Started

**Start here if you're new:**

**🎯 [12-consolidated-production-plan.md](12-consolidated-production-plan.md)** ⭐⭐⭐ **START HERE - COMPLETE PLAN**
   - **Consolidated plan combining docs 9, 10, 11**
   - High-level design (no code samples)
   - Realistic timeline for solo developer, part-time
   - Accounts for holidays and breaks
   - 9 months calendar time, 340-680 hours
   - Phase-by-phase implementation (5 phases)
   - Target: $15k-45k/month by September 2026
   - **Best for:** Understanding the complete journey

---

**Detailed Architecture Docs (for deep dives):**

1. **[11-extensible-strategy-architecture.md](11-extensible-strategy-architecture.md)** - **MULTI-STRATEGY ARCHITECTURE**
   - **Multi-strategy extensible architecture** with plugin system
   - Generic market events reusable across strategies
   - Arbitrage, DCA, Grid Trading, Long/Short strategies
   - Event-driven via NATS JetStream
   - Rust + TypeScript hybrid (use each where it's best)
   - Easy to add new strategies
   - **Best for:** Building a complete trading platform

2. **[10-rust-execution-architecture.md](10-rust-execution-architecture.md)** ⭐ **RUST HFT FOCUS**
   - **Rust-first execution architecture** for sub-200ms latency
   - Shredstream integration (400ms early alpha)
   - LST triangular arbitrage selection logic (jitoSOL ↔ mSOL scoring)
   - SOL/USDC as low-priority conservative strategy
   - Dark pool on-chain simulation integration
   - TypeScript for analytics/PnL only
   - **Best for:** Pure arbitrage HFT system with maximum performance

2. **[09-lst-arbitrage-production-plan.md](09-lst-arbitrage-production-plan.md)** ⭐ **RECOMMENDED START (TYPESCRIPT)**
   - Complete production development plan for LST arbitrage
   - Week-by-week implementation guide (12 weeks total)
   - TypeScript execution layer (easier to develop)
   - Code examples and architecture
   - Performance targets and success metrics
   - **Focus:** Winnable LST markets (jitoSOL, mSOL, bSOL) instead of SOL/USDC
   - **Best for:** Faster development, 750ms → 300ms execution

### Optimization & Performance

2. **[08-optimization-guide.md](08-optimization-guide.md)** ⭐ **OPTIMIZE EXISTING BOT**
   - Transform 1.7s → 200ms execution
   - Week-by-week optimization roadmap
   - Quick wins (this week): 2-3x improvement
   - Bottleneck analysis: Jupiter API, RPC calls, blockhash fetching
   - **Use this:** If you have a working bot that needs speed improvements

3. **[07-hft-architecture.md](07-hft-architecture.md)** ⭐ **FOR SUB-500MS HFT**
   - Sub-500ms execution architecture
   - Shredstream integration (400ms early alpha)
   - Local pool math in Go/Rust (< 10ms quotes)
   - Flash loan optimization (zero capital)
   - Advanced techniques: triangular arbitrage, market making
   - Memory-mapped state, SIMD calculations
   - **Use this:** For high-frequency trading (HFT) requirements

### Integration Guides

**🔥 [17-shredstream-architecture-design.md](17-shredstream-architecture-design.md)** ⭐⭐⭐ **NEW - SHREDSTREAM INTEGRATION**
   - **Complete architectural design for Jito Shredstream integration**
   - Analyzes 3 architectural options (microservice, monolithic, hybrid)
   - **Recommended:** Separate TypeScript Scanner + Go Quote Service via NATS
   - 300-800ms head start on pool price changes
   - Detailed implementation specs with code examples
   - Performance analysis: 70-90ms end-to-end latency
   - Docker Compose configuration
   - Monitoring & observability setup
   - **Expected impact:** 1.7s → 800ms (Week 1), → 200ms (Week 4)
   - **Best for:** Implementing Shredstream for early market data

**📋 [17a-shredstream-implementation-checklist.md](17a-shredstream-implementation-checklist.md)** - **IMPLEMENTATION CHECKLIST**
   - **Step-by-step implementation guide** for Shredstream integration
   - 8-day roadmap with daily tasks
   - Checkboxes for tracking progress
   - Troubleshooting guide
   - Success criteria per phase
   - **Use this:** As your daily checklist while implementing Shredstream

### Roadmaps & Implementation

4. **[06-solo-developer-roadmap.md](06-solo-developer-roadmap.md)** ⭐ **GENERAL SOLO DEV PLAN**
   - Realistic roadmap for solo developer
   - MVP-first approach (4 weeks to first trade)
   - Part-time: 8-10 months | Full-time: 4-5 months
   - Phase-by-phase with time estimates
   - Success metrics and milestones
   - **Use this:** For general trading system (not LST-specific)

5. **[03-implementation-roadmap.md](03-implementation-roadmap.md)**
   - Original team-based implementation plan
   - 8 phases, ~6 months timeline
   - Detailed component specifications
   - **Use this:** For reference on full-scale team implementation

### Architecture & Design

6. **[02-production-architecture-plan.md](02-production-architecture-plan.md)**
   - Complete production system architecture
   - Scanner → Planner → Executor pattern
   - Event-driven communication via NATS JetStream
   - Component specifications for all subsystems
   - Data flow examples and integration patterns
   - **Use this:** For overall system architecture understanding

7. **[05-scanner-planner-executor-design.md](05-scanner-planner-executor-design.md)**
   - Concrete design patterns for Scanner → Planner → Executor
   - Event schemas and data flow examples
   - Complete arbitrage trade walkthrough
   - Testing strategies (unit, integration, end-to-end)
   - Best practices and idempotency patterns
   - **Use this:** For detailed component design

8. **[04-tech-stack-details.md](04-tech-stack-details.md)**
   - Detailed justification for each technology choice
   - TypeScript, Go, and Rust usage patterns
   - Infrastructure setup (NATS, Redis, PostgreSQL)
   - Observability stack (Prometheus, Grafana, Jaeger, Loki)
   - Code examples and configuration templates
   - **Use this:** For technology selection and setup

### Analysis

9. **[01-prototype-analysis.md](01-prototype-analysis.md)**
   - Analysis of prototype systems in `references/` directory
   - Comparison matrix of features and architectures
   - Strengths and weaknesses of each approach
   - Recommendations for production system
   - **Use this:** To understand the prototype code in `references/`

---

## 🎯 Quick Decision Guide

### "I'm a solo developer working part-time and want a complete plan"
→ **[12-consolidated-production-plan.md](12-consolidated-production-plan.md)** ⭐⭐⭐ **START HERE**
- Combines best of docs 9, 10, 11
- Realistic 9-month timeline (340-680 hours)
- Accounts for holidays and breaks (Dec 23-Jan 5, Jan 26-Feb 20)
- Phase-by-phase: Foundation → Market Data → Arbitrage → Multi-Strategy
- First trade by March 2026, $15k/month by September 2026
- High-level only (no code samples, just architecture and plans)
- **Best for:** Part-time hobby project, sustainable pace

### "I want to build a multi-strategy trading platform (arbitrage + DCA + grid trading + more)"
→ Start with **[11-extensible-strategy-architecture.md](11-extensible-strategy-architecture.md)** ⭐ **NEW**
- Extensible plugin architecture
- Multiple strategies working simultaneously
- Generic market events (reusable across strategies)
- Event-driven via NATS JetStream
- Rust for performance-critical strategies, TypeScript for rapid development
- Easy to add new strategies (implement one trait/interface)
- **Best for:** Building a complete trading platform with multiple strategies

### "I want to build a high-performance Rust-based LST arbitrage system"
→ Start with **[10-rust-execution-architecture.md](10-rust-execution-architecture.md)**
- Rust execution layer (100-200ms latency)
- Shredstream integration (400ms early market data)
- LST triangular arbitrage (jitoSOL ↔ mSOL ↔ bSOL)
- SOL/USDC as low-priority fallback
- Dark pool on-chain simulation
- TypeScript for analytics only
- **Best for:** Maximum performance, already have Shredstream prototype

### "I want to build an LST arbitrage bot from scratch (TypeScript)"
→ Start with **[09-lst-arbitrage-production-plan.md](09-lst-arbitrage-production-plan.md)**
- Complete 12-week plan
- LST-specific strategy (jitoSOL, mSOL, bSOL)
- TypeScript execution layer (300-500ms)
- 60-75% success rate target
- $15k-30k/month profit target
- **Best for:** Faster development, easier debugging

### "I have a working bot but it's too slow (1-3 seconds)"
→ Start with **[08-optimization-guide.md](08-optimization-guide.md)**
- Optimize 1.7s → 200ms
- Week-by-week improvements
- Quick wins in first week

### "I want to integrate Jito Shredstream for 400ms early market data" 🔥 **NEW**
→ Start with **[17-shredstream-architecture-design.md](17-shredstream-architecture-design.md)** + **[17a-shredstream-implementation-checklist.md](17a-shredstream-implementation-checklist.md)**
- **Complete architectural design** with 3 options analyzed
- **Recommended:** Separate Scanner service + Go Quote Service via NATS
- **Impact:** 300-800ms head start on pool price changes
- **Timeline:** 8 days implementation
- **Expected outcome:** 1.7s → 800ms (Week 1), → 200ms (Week 4)
- **Step-by-step checklist** with daily tasks
- **Best for:** Already have Go Quote Service or working bot, need early market data

### "I want sub-500ms HFT execution"
→ Start with **[07-hft-architecture.md](07-hft-architecture.md)**
- Shredstream integration
- Local pool math (Go/Rust)
- Memory-mapped state
- SIMD calculations

### "I want to build a general trading system (not just LST)"
→ Start with **[06-solo-developer-roadmap.md](06-solo-developer-roadmap.md)** + **[02-production-architecture-plan.md](02-production-architecture-plan.md)**
- Multiple strategies (arbitrage, grid trading, DCA, AI analysis)
- Full Scanner → Planner → Executor architecture
- 8-10 months part-time

### "I need to understand the system architecture"
→ Read **[02-production-architecture-plan.md](02-production-architecture-plan.md)** + **[05-scanner-planner-executor-design.md](05-scanner-planner-executor-design.md)**
- Complete architecture overview
- Component interactions
- Event flows

### "I need to choose my tech stack"
→ Read **[04-tech-stack-details.md](04-tech-stack-details.md)**
- TypeScript, Go, Rust usage patterns
- Infrastructure components
- Configuration examples

---

## 📖 Reading Order by Goal

### Goal: Build Production LST Arbitrage System (Fastest Path to Profit)

**Reading Order:**
1. [09-lst-arbitrage-production-plan.md](09-lst-arbitrage-production-plan.md) - Complete implementation plan
2. [08-optimization-guide.md](08-optimization-guide.md) - Performance optimization techniques
3. [04-tech-stack-details.md](04-tech-stack-details.md) - Technology setup details
4. [07-hft-architecture.md](07-hft-architecture.md) - Advanced optimizations (optional)

**Timeline:** 12 weeks (3 months) to production

### Goal: Build General Multi-Strategy Trading System

**Reading Order:**
1. [06-solo-developer-roadmap.md](06-solo-developer-roadmap.md) - Overall roadmap
2. [02-production-architecture-plan.md](02-production-architecture-plan.md) - System architecture
3. [05-scanner-planner-executor-design.md](05-scanner-planner-executor-design.md) - Component design
4. [04-tech-stack-details.md](04-tech-stack-details.md) - Technology details
5. [01-prototype-analysis.md](01-prototype-analysis.md) - Learn from prototypes

**Timeline:** 8-10 months part-time (4-5 months full-time)

### Goal: Optimize Existing Bot

**Reading Order:**
1. [08-optimization-guide.md](08-optimization-guide.md) - Week-by-week optimizations
2. [07-hft-architecture.md](07-hft-architecture.md) - Advanced HFT techniques
3. [09-lst-arbitrage-production-plan.md](09-lst-arbitrage-production-plan.md) - LST strategy (if switching pairs)

**Timeline:** 4-8 weeks to sub-500ms execution

### Goal: Integrate Shredstream for Early Market Data 🔥 **NEW**

**Reading Order:**
1. [17-shredstream-architecture-design.md](17-shredstream-architecture-design.md) - Complete architecture design
2. [17a-shredstream-implementation-checklist.md](17a-shredstream-implementation-checklist.md) - Day-by-day tasks
3. [08-optimization-guide.md](08-optimization-guide.md) - Week 2 guidance (after Shredstream)
4. [07-hft-architecture.md](07-hft-architecture.md) - Advanced optimizations (Week 3-4)

**Timeline:** 8 days implementation, 1.7s → 800ms (Week 1), → 200ms (Week 4)

---

## 🔑 Key Insights Across All Docs

### Market Strategy
- **SOL/USDC is unwinnable** for solo developers (100+ bots, <20% success rate)
- **LST pairs (jitoSOL, mSOL, bSOL)** offer 60-75% success rates
- **Two-hop arbitrage** (SOL → LST → SOL) is simpler and more profitable than three-hop
- **Spreads:** LST pairs have 0.3-1% spreads vs 0.05-0.15% for SOL/USDC

### Performance
- **Quote latency:** Jupiter API (100-300ms) vs local Go service (2-10ms)
- **Execution target:** <500ms end-to-end (<200ms for HFT)
- **Bottlenecks:** Quote fetching, RPC calls, blockhash fetching, confirmation polling
- **Quick wins:** Cache blockhash (50ms saved), parallel quotes (2x faster), batch RPC (10x faster)

### Architecture
- **Scanner → Planner → Executor** pattern for scalability
- **Event-driven** via NATS JetStream for decoupling
- **Redis** for hot data (quotes, routes, balances)
- **PostgreSQL** for persistent data (trades, history)
- **Prometheus + Grafana** for observability

### Technology
- **TypeScript:** Business logic, scanners, planners, executors
- **Go:** High-performance quote service (2-10ms), pool math
- **Rust:** RPC proxy, transaction builder (optional for extreme performance)
- **Jito bundles:** MEV protection, 95%+ landing rate
- **Flash loans:** Zero capital arbitrage (Kamino)

### Risk Management
- **Circuit breakers:** Stop after 3 consecutive failures
- **Position limits:** Max 100 SOL per trade, max 50 SOL per LST
- **Minimum profit:** 0.1-0.3% after fees
- **Simulation required:** Pre-flight validation before every trade

---

## 📊 Performance Targets Summary

| Phase | Timeline | Opportunities/Day | Success Rate | Execution Latency | Daily Profit |
|-------|----------|-------------------|--------------|-------------------|--------------|
| MVP | Week 4 | 50-100 | 50-60% | 750-1,000ms | $30-80 |
| Optimized | Week 8 | 150-250 | 70-75% | 300-500ms | $100-250 |
| Advanced | Week 12 | 400-800 | 75-80% | 200-300ms | $500-1,200 |
| Production | Month 6 | 800-1,500 | 80-85% | 100-200ms | $1,000-2,500 |

**Monthly Profit (Production):** $30,000-75,000

---

## 💰 Infrastructure Costs Summary

| Phase | Instance Type | Monthly Cost |
|-------|---------------|--------------|
| MVP (Months 1-3) | t3.medium + managed DBs | $60-80 |
| Production (Months 4-6) | c6i.xlarge + managed DBs | $635 |
| High-Volume (Months 6+) | c6i.2xlarge or bare metal | $500-1,350 |

**Recommendation:** Start with MVP tier, scale up as profitable

---

## 🚀 Getting Started Checklist

**Before you start coding:**
- [ ] Read [09-lst-arbitrage-production-plan.md](09-lst-arbitrage-production-plan.md) (if doing LST arbitrage)
- [ ] OR read [06-solo-developer-roadmap.md](06-solo-developer-roadmap.md) (if doing general system)
- [ ] Review [04-tech-stack-details.md](04-tech-stack-details.md) for tech setup
- [ ] Review [01-prototype-analysis.md](01-prototype-analysis.md) to understand prototypes

**Development environment:**
- [ ] Set up Docker Compose (Redis, PostgreSQL, NATS, Prometheus, Grafana)
- [ ] Initialize pnpm workspace for TypeScript services
- [ ] Set up Go workspace for quote service
- [ ] Configure environment variables (.env files)

**First milestone (Week 4 for LST, Week 4 for general):**
- [ ] Working quote service responding < 10ms
- [ ] Scanner detecting opportunities
- [ ] Executor building and simulating transactions
- [ ] **Execute first profitable trade**

---

## 📝 Document Change Log

- **2025-01-02:** Added [09-lst-arbitrage-production-plan.md](09-lst-arbitrage-production-plan.md) - LST-focused production plan
- **2025-01-02:** Created README.md - Documentation index and navigation guide

---

## 🤝 Contributing

This documentation is living and should be updated as the system evolves. When adding new documentation:

1. Add entry to this README
2. Update relevant reading orders
3. Link to/from related documents
4. Update performance targets if applicable

---

## 📧 Support

For questions about the documentation or implementation:
- Review the relevant doc(s) based on your goal
- Check code examples in `references/` directory for proven patterns
- Refer to prototype analysis for implementation details

---

**Last Updated:** December 2, 2025