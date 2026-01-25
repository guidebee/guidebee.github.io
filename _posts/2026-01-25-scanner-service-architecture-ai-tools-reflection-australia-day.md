---
layout: single
title: "Scanner Service Deep Dive: Arbitrage Detection Engine with Full Observability"
date: 2026-01-25
permalink: /posts/2026/01/scanner-service-architecture-ai-tools-reflection-australia-day/
categories:
  - blog
tags:
  - solana
  - trading
  - scanner-service
  - typescript
  - architecture
  - observability
  - grafana
  - ai-tools
  - reflection
excerpt: "Deep dive into the TypeScript Scanner Service architecture - the arbitrage detection engine that processes gRPC quote streams, detects profitable opportunities, and publishes FlatBuffers events. Plus: reflections on two months of AI-assisted development and thoughts on the future of programming."
---

## TL;DR

- **Scanner Service Complete**: First production component in the HFT pipeline - detects arbitrage opportunities from dual quote streams with sub-10ms latency
- **Full Observability**: Comprehensive Grafana dashboard with 3.86M quotes processed and 878K opportunities detected
- **Two-Month Reflection**: AI tools (GitHub Copilot, Claude Code, Google's tools) are transforming development - traditional programming may be fundamentally changing
- **Happy Australia Day**: Taking a holiday break to visit family in China

---

## Happy Australia Day!

Before diving into technical content, I want to wish everyone a **Happy Australia Day**! It's January 26th here, and I'll be taking a well-deserved holiday break, heading back to China to visit my parents, brother, and friends. After two intense months of building this trading system, some time with family is exactly what I need.

---

## Table of Contents

1. [Scanner Service Architecture](#scanner-service-architecture)
2. [Data Flow Pipeline](#data-flow-pipeline)
3. [Grafana Dashboard](#grafana-dashboard)
4. [Two-Month Project Summary](#two-month-project-summary)
5. [Reflections on AI-Assisted Development](#reflections-on-ai-assisted-development)
6. [Conclusion](#conclusion)

---

## Scanner Service Architecture

The TypeScript Scanner Service is the **first production component** in the HFT trading pipeline. It serves as both a functional production system and a prototype for a future high-performance Rust implementation.

### System Context

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              HFT Trading Pipeline                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌──────────────────────┐                                                       │
│  │  Pool Discovery      │  Discovers DEX pools (Raydium, Meteora, PumpSwap)    │
│  │  Service (Go)        │  Subscribes to on-chain pool state changes            │
│  └──────────┬───────────┘                                                       │
│             │ Pool updates                                                      │
│             ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                        Quote Services Layer (Go)                          │  │
│  │  ┌─────────────────────────┐    ┌─────────────────────────────────────┐  │  │
│  │  │ Local Quote Service     │    │ External Quote Service              │  │  │
│  │  │ (port 50052)            │    │ (port 50053)                        │  │  │
│  │  │                         │    │                                     │  │  │
│  │  │ - On-chain pool quotes  │    │ - Jupiter aggregator API            │  │  │
│  │  │ - < 10ms latency        │    │ - Dark pools (HumidiFi, GoonFi)     │  │  │
│  │  │ - 30s refresh interval  │    │ - 10s refresh interval              │  │  │
│  │  │ - No rate limits        │    │ - 1 RPS Jupiter rate limit          │  │  │
│  │  └────────────┬────────────┘    └─────────────────┬───────────────────┘  │  │
│  └───────────────┼────────────────────────────────────┼──────────────────────┘  │
│                  │ gRPC Stream                        │ gRPC Stream              │
│                  │                                    │                          │
│  ┌───────────────▼────────────────────────────────────▼──────────────────────┐  │
│  │                    ★ SCANNER SERVICE (TypeScript) ★                       │  │
│  │                                                                           │  │
│  │  - Receives quotes from both services via gRPC streaming                  │  │
│  │  - Detects arbitrage opportunities (round-trip profit)                    │  │
│  │  - Publishes FlatBuffers events to NATS JetStream                         │  │
│  │  - Full observability (Prometheus, Loki, Tempo)                           │  │
│  └───────────────────────────────────────────┬───────────────────────────────┘  │
│                                              │ NATS JetStream                    │
│                                              │ (FlatBuffers)                     │
│  ┌───────────────────────────────────────────▼───────────────────────────────┐  │
│  │                        Strategy Service (TypeScript)                      │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Internal Components

The scanner service is built with a clean, layered architecture:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Scanner Service Internals                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                             Entry Point                                   │  │
│  │                           (src/index.ts)                                  │  │
│  │                                                                           │  │
│  │  - Initialize observability (ts-scanner-service prefix)                   │  │
│  │  - Start HTTP metrics server (:9092)                                      │  │
│  │  - Connect to NATS JetStream                                              │  │
│  │  - Create and start scanner components                                    │  │
│  │  - Handle graceful shutdown (SIGINT/SIGTERM)                              │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│                    ┌───────────────────┼───────────────────┐                    │
│                    │                   │                   │                    │
│                    ▼                   ▼                   ▼                    │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌────────────────────────┐  │
│  │   ScannerService    │  │ ArbitrageQuote      │  │    SystemHandler       │  │
│  │   (scanner.ts)      │  │ ScannerFlatBuf      │  │  (system-handler.ts)   │  │
│  │                     │  │                     │  │                        │  │
│  │ NATS MARKET_DATA    │  │ Main scanner engine │  │ SYSTEM stream listener │  │
│  │ consumer (legacy)   │  │ gRPC quote streams  │  │ - Kill switch          │  │
│  │                     │  │ FlatBuf publishing  │  │ - Graceful shutdown    │  │
│  └─────────────────────┘  └──────────┬──────────┘  └────────────────────────┘  │
│                                      │                                          │
│                                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │                         gRPC Client Layer                              │    │
│  │  ┌─────────────────────────────┐  ┌─────────────────────────────────┐  │    │
│  │  │   GrpcMultiQuoteClient      │  │   GrpcAggregatorClient          │  │    │
│  │  │   (Direct Mode - Default)   │  │   (Aggregator Mode)             │  │    │
│  │  │                             │  │                                 │  │    │
│  │  │  - Local stream (50052)     │  │  - Single connection (50051)    │  │    │
│  │  │  - External stream (50053)  │  │  - Pre-aggregated quotes        │  │    │
│  │  │  - Emits TaggedQuote        │  │  - Emits AggregatedQuote        │  │    │
│  │  │  - Independent reconnect    │  │  - Best source selection        │  │    │
│  │  └─────────────────────────────┘  └─────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

**1. Direct Mode as Default**

The scanner operates in "direct mode" by default, connecting to both local (50052) and external (50053) quote services independently. This provides:

- Independent processing of local and external quotes
- External arbitrage detection even when local "wins"
- Lower latency without aggregation overhead
- Separate profit thresholds per source

**2. FlatBuffers over JSON**

Events are serialized using FlatBuffers instead of JSON for:

- ~10x faster serialization/deserialization
- ~50% smaller payload size
- Schema-enforced type safety at compile time
- Zero-copy direct memory access

**3. Source-Specific Profit Thresholds**

- **LOCAL quotes**: 50 bps minimum (higher confidence, trusted on-chain data)
- **EXTERNAL quotes**: 10 bps minimum (API aggregation may have stale data, but Jupiter multi-hop routing can find better prices)

---

## Data Flow Pipeline

### Quote Processing Flow

```
1. QUOTE ACQUISITION
   ┌────────────────────────────────────────────────────────────────────┐
   │                                                                    │
   │  Local Quote Service ─────┐                                        │
   │  (50052)                  │                                        │
   │  - Raydium AMM/CLMM/CPMM │         ┌──────────────────────┐       │
   │  - Meteora DLMM          ├────────▶│  GrpcMultiQuoteClient │       │
   │  - PumpSwap              │         │                      │       │
   │                          │         │  TaggedQuote {       │       │
   │  External Quote Service ─┘         │    sourceTag: 'local'│       │
   │  (50053)                           │           | 'external'│      │
   │  - Jupiter API                     │  }                   │       │
   │  - Dark pools                      └──────────┬───────────┘       │
   │                                               │                    │
   └───────────────────────────────────────────────┼────────────────────┘
                                                   │
                                                   ▼
2. QUOTE CACHING & MATCHING
   ┌────────────────────────────────────────────────────────────────────┐
   │                         QuoteStreamHandler                         │
   │                                                                    │
   │  Quote Cache (Map<string, CachedQuote>)                           │
   │  ┌─────────────────────────────────────────────────────────────┐  │
   │  │  Key: `${pairId}:${inputMint}:${provider}`                  │  │
   │  │  Value: { quote, timestamp, source }                        │  │
   │  │                                                              │  │
   │  │  TTL:                                                        │  │
   │  │  - Local quotes: 5 seconds                                  │  │
   │  │  - External quotes: 30 seconds                              │  │
   │  └─────────────────────────────────────────────────────────────┘  │
   │                                                                    │
   │  Matching Logic:                                                   │
   │  ┌─────────────────────────────────────────────────────────────┐  │
   │  │  For each incoming quote (A → B):                           │  │
   │  │    1. Cache the quote                                       │  │
   │  │    2. Look for reverse quote (B → A) in cache               │  │
   │  │    3. If found → Calculate arbitrage profit                 │  │
   │  └─────────────────────────────────────────────────────────────┘  │
   │                                                                    │
   └───────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
3. PROFIT CALCULATION
   ┌────────────────────────────────────────────────────────────────────┐
   │                        profit-calculator.ts                        │
   │                                                                    │
   │  Two-Hop Arbitrage Formula:                                        │
   │  ┌─────────────────────────────────────────────────────────────┐  │
   │  │                                                              │  │
   │  │  Forward Quote: A → B                                        │  │
   │  │    inputAmount = 1 SOL (1_000_000_000 lamports)              │  │
   │  │    forwardOutput = X USDC                                   │  │
   │  │                                                              │  │
   │  │  Reverse Quote: B → A                                        │  │
   │  │    inputAmount = Y USDC (from previous quote)                │  │
   │  │    reverseOutput = Z SOL                                    │  │
   │  │                                                              │  │
   │  │  Profit:                                                     │  │
   │  │    grossProfit = adjustedOutput - initialInput               │  │
   │  │    networkFees = priorityFee + jitoTip (≈ 30,000 lamports)  │  │
   │  │    netProfit = grossProfit - networkFees                    │  │
   │  │    profitBps = (netProfit / initialInput) * 10000           │  │
   │  │                                                              │  │
   │  └─────────────────────────────────────────────────────────────┘  │
   │                                                                    │
   └───────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
4. EVENT PUBLISHING
   ┌────────────────────────────────────────────────────────────────────┐
   │                                                                    │
   │  FlatBuffers Serialization → NATS JetStream                       │
   │  ┌─────────────────────────────────────────────────────────────┐  │
   │  │  Stream: OPPORTUNITIES                                      │  │
   │  │  Subject: opportunity.arbitrage.two_hop.{token1}.{token2}   │  │
   │  │  Format: FlatBuffers binary (JSON fallback on error)        │  │
   │  └─────────────────────────────────────────────────────────────┘  │
   │                                                                    │
   └────────────────────────────────────────────────────────────────────┘
```

### Performance Characteristics

| Operation | Target | Actual |
|-----------|--------|--------|
| Quote receive (gRPC) | < 1ms | ~0.5ms |
| Quote caching | < 1ms | ~0.1ms |
| Quote matching | < 1ms | ~0.2ms |
| Profit calculation | < 1ms | ~0.3ms |
| FlatBuffers serialize | < 1ms | ~0.2ms |
| NATS publish | < 5ms | ~2ms |
| **Total per quote** | **< 10ms** | **~3-4ms** |

---

## Grafana Dashboard

The scanner service has a comprehensive Grafana dashboard for monitoring all aspects of service health and performance.

![Scanner Service Dashboard](../images/scanner-service.png)

### Dashboard Overview

| Section | Key Metrics |
|---------|-------------|
| **Overview Metrics** | Service Status, Uptime, NATS/gRPC Connection Status |
| **Quote Streaming** | Quotes Received Rate (3.86M total), Active Token Pairs, Quote Latency |
| **Quote Source Analysis** | Local vs External distribution, Opportunities by source |
| **Arbitrage Opportunities** | 878K total opportunities detected, Profit distribution |
| **Performance** | CPU usage, Event processing time, Queue size |
| **Errors & Logs** | Error rate tracking, Loki log streaming |

### Key Metrics

**Quote Processing**:
- `ts_scanner_service_quotes_received_total` - 3.86 Million quotes processed
- `ts_scanner_service_active_token_pairs` - Number of monitored pairs

**Arbitrage Detection**:
- `ts_scanner_service_arbitrage_opportunities_detected_total` - 878K opportunities found
- `ts_scanner_service_arbitrage_profit_bps` - Profit distribution in basis points

**Health Indicators**:
- `ts_scanner_service_nats_connected` - NATS connectivity (green = connected)
- `ts_scanner_service_grpc_connected` - gRPC stream health

### Alert Rules

**Critical (P0)**:
- Scanner service down for > 1 minute
- NATS disconnected for > 30 seconds
- gRPC disconnected for > 30 seconds

**Warning (P1)**:
- Error rate > 0.1/second for 5 minutes
- Queue backlog > 100 for 2 minutes
- No quotes received for 2 minutes

---

## Two-Month Project Summary

Looking back at the past two months of development (late November 2025 to late January 2026), the progress has been remarkable:

### Infrastructure Built (85% Complete)

**Go Codebase** (~62,000 lines across 206 files):
- Pool Discovery Service - DEX pool discovery with Redis caching
- Local Quote Service - 10 DEX protocols, sub-2ms selection
- External Quote Service - Jupiter/DFlow/OKX integration
- Quote Aggregator Service - Confidence scoring, oracle comparison
- Event Logger Service - 6-stream NATS, FlatBuffers, Loki integration

**TypeScript Services**:
- Scanner Service - Arbitrage detection engine (this post)
- Strategy Service - Opportunity validation (in progress)
- Executor Service - Transaction building/submission (planned)

**Infrastructure** (21 Docker services):
- Event Streaming: NATS JetStream with 6 configured streams
- Observability: Grafana LGTM+ stack (Prometheus, Loki, Tempo, Mimir, Pyroscope)
- Database: PostgreSQL + TimescaleDB
- Caching: Redis with pub/sub
- Networking: Traefik reverse proxy

### Milestones Achieved

| Date | Milestone |
|------|-----------|
| Dec 4 | Project kickoff, initial architecture |
| Dec 10 | Grafana LGTM stack unified observability |
| Dec 18 | FlatBuffers migration, HFT pipeline ready |
| Dec 22 | Quote service architecture established |
| Dec 28 | Pool discovery service complete |
| Jan 1 | Quote service three-way split |
| Jan 8 | Pool discovery refactored with testing |
| Jan 18 | Quote services ecosystem evolution |
| Jan 20 | Infrastructure milestone - 85% complete |
| Jan 25 | Scanner service deep dive (this post) |

---

## Reflections on AI-Assisted Development

This project has been my first deep experience using AI coding assistants for a substantial codebase. Here are my honest reflections:

### Tools Used

**GitHub Copilot**: Excellent for autocomplete and inline suggestions. Particularly helpful for boilerplate code, repetitive patterns, and when you know exactly what you want to write.

**Claude Code**: My primary tool for complex tasks. Particularly strong at:
- Understanding system architecture
- Writing comprehensive documentation
- Debugging complex issues
- Refactoring large codebases

**Google's AI Tools (Gemini/Bard)**: Good for research and exploring alternatives. Useful for getting different perspectives on architectural decisions.

### What Impressed Me

For **clearly defined tasks**, these tools can work as fast as (or faster than) experienced developers like myself. Examples:

- Writing unit tests for existing code
- Creating Grafana dashboards from metrics
- Implementing well-documented APIs
- Refactoring code to new patterns
- Writing documentation

The scanner service documentation you see in this post? Much of it was AI-assisted. The architecture diagrams, the metric tables, the alert configurations - tasks that would have taken hours were completed in minutes.

### Current Limitations

AI tools still need **experienced developer guidance**:

- Architectural decisions require human judgment
- Security considerations need careful review
- Performance optimization needs real-world testing
- Integration testing catches issues AI misses

The tools are assistants, not replacements. They amplify developer productivity but don't eliminate the need for expertise.

### The Future of Programming

Here's my honest prediction: **traditional programming as we know it may be fundamentally changing**.

For the past 30+ years, programming has been about translating human intent into precise syntax. But AI is rapidly closing the gap between "what I want" and "working code."

**Near term (1-2 years)**:
- AI handles 70-80% of routine coding tasks
- Developers focus on architecture, integration, and oversight
- Code review becomes more about intent than syntax

**Medium term (3-5 years)**:
- AI may form closed loops - improving its own code
- The role of "programmer" evolves to "system architect" and "AI supervisor"
- Entry-level coding jobs transform dramatically

**My advice for the next generation**: Learn to work WITH AI, not against it. The skills that matter will be:
- System thinking and architecture
- Understanding business requirements
- Quality assurance and testing
- Security and ethics
- Communication and collaboration

The developers who thrive will be those who can leverage AI to multiply their effectiveness, not those who try to compete with it on raw coding speed.

---

## Conclusion

The Scanner Service represents a significant milestone in the Solana Trading System - the first production component capable of detecting arbitrage opportunities in real-time. With 3.86M quotes processed and 878K opportunities detected, the system is proving its viability.

**Technical Achievements**:
- Sub-10ms end-to-end processing latency
- Dual quote source integration (local + external)
- FlatBuffers serialization for high performance
- Comprehensive observability with Grafana

**Personal Reflections**:
- AI tools have transformed my development workflow
- Two months of intense work has built a solid foundation
- The future of programming is exciting (and a bit uncertain)

**What's Next**:
After the holiday break, focus shifts to:
1. Strategy Service implementation
2. Executor Service with Jito bundle submission
3. Paper trading validation
4. Small capital testing

---

Happy Australia Day to everyone! I'll be spending quality time with family in China, recharging for the next phase of development. See you in February!

---

## Related Posts

- [Project Milestone Complete: Infrastructure Ready for Arbitrage Phase](/posts/2026/01/project-milestone-complete-infrastructure-ready-for-arbitrage-phase/) - Overall project status
- [Quote Services Evolution: Local Enhancements, External Integration, and Aggregator Architecture](/posts/2026/01/quote-services-local-enhancements-external-integration-aggregator-architecture/) - Quote service ecosystem
- [HFT Pipeline Architecture: FlatBuffers Performance Optimization](/posts/2025/12/hft-pipeline-architecture-flatbuffers-performance-optimization/) - Event serialization

---

## Technical Documentation

- [Scanner Service Architecture](https://github.com/guidebee/solana-trading-system/blob/master/ts/apps/scanner-service/docs/ARCHITECTURE.md)
- [Grafana Dashboard Documentation](https://github.com/guidebee/solana-trading-system/blob/master/ts/apps/scanner-service/docs/GRAFANA_DASHBOARD.md)
- [Project Status and Feasibility](https://github.com/guidebee/solana-trading-system/blob/master/docs/08-PROJECT-STATUS-AND-FEASIBILITY.md)

---

## Connect

- **GitHub**: [guidebee/solana-trading-system](https://github.com/guidebee/solana-trading-system)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

*This is post #24 in the Solana Trading System development series. The Scanner Service brings the HFT pipeline to life, detecting arbitrage opportunities with sub-10ms latency. As I take a holiday break for Australia Day, I reflect on two months of AI-assisted development and the changing landscape of programming. Happy holidays to all!*
