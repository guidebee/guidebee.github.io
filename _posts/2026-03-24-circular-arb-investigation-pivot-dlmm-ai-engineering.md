---
layout: single
title: "Honest Post-Mortem: Why Circular Arbitrage Fails Without Proper Infrastructure Investment"
date: 2026-03-24
permalink: /posts/2026/03/circular-arb-investigation-pivot-dlmm-ai-engineering/
categories:
  - blog
tags:
  - solana
  - trading
  - arbitrage
  - dlmm
  - pivot
  - reflection
  - ai-engineering
  - career
excerpt: "After extended testing of our circular arb engine and a bot-tracing investigation that revealed a competitor likely running co-located infrastructure, it's becoming clear: circular arbitrage on Solana seems very difficult to achieve with home settings — and the infrastructure investment required to compete changes the economics entirely."
---

## TL;DR

- **The early prototype worked** — a few dollars a day last year was enough to spark this whole project
- **Market conditions changed** — circular arb is now dominated by bots with co-location infrastructure; without that investment, it's not viable
- **Difficult without proper infrastructure** — live testing suggests circular arb with home settings is very hard; speed-sensitive opportunities seem to require co-location to be competitive
- **Bot-tracing reveals the competition** — we tracked one effective bot and found its profit margin is ~$30/month; it likely co-locates with an RPC node
- **Not a wasted effort** — the infrastructure, DEX math, and execution pipeline are directly reusable for better strategies
- **Pivoting to DLMM LP + funding rate arb**, while also revisiting HFT research and pursuing AI engineering credentials (Azure AI-102, AWS MLA-C01)

---

## How This All Started: The $2/Day Prototype

This project didn't begin with a grand vision. It started with a prototype built in 2024 that traded the **SOL/USDC pair** — not LST routes — executing circular arb trades via Jupiter and racing submissions through Jito bundles, direct RPC, and TPU simultaneously. The system had flash loan wrapping (Kamino), 95+ RPC endpoint round-robin, route template caching in Redis, and Address Lookup Tables for transaction size reduction. It wasn't crude — it was a genuine attempt at production MEV.

And it made money. Not much — a couple of dollars on a good day — but real, on-chain profit. Enough to prove the concept was sound. Enough to make you think: *if this can work today, what could a purpose-built production system targeting less-contested LST pairs do?*

That question became this project.

---

## What We Built

The system that grew from that prototype is a polyglot monorepo — Go, TypeScript, Rust — with a microservices architecture designed around the Scanner → Planner → Executor pipeline, connected by NATS JetStream with FlatBuffers serialisation for zero-copy event passing.

**Four-stage pipeline:**
```
Quote Service (Go)  →  Scanner (TS)  →  Planner (TS)  →  Executor (TS)
  <10ms/quote          <10ms detect      50-100ms sim      <20ms submit
```

**Quote service** (`go/internal/local-quote-service` + gRPC streaming):
- On-chain pool state cached in memory, refreshed from RPC every 5–30s
- Local DEX math for Orca Whirlpool (CLMM), Raydium AMM V4, Raydium CLMM, Meteora DLMM, PumpSwap — no external API
- Pool registry sourced from Redis (populated by pool-discovery-service)
- Oracle price validation via Pyth over NATS
- gRPC streaming for sub-millisecond delivery to scanner

**Scanner service** (`ts/services/scanner`):
- Consumes gRPC quote stream, detects two-hop and triangular arb patterns
- Published 9.4M+ quotes across multi-DEX pairs over 52-day validation run
- Correct arb signal detection across Orca, Raydium, Meteora pool types
- FlatBuffers events published to NATS OPPORTUNITIES stream

**Jupiter-mode arb engine** (`ts/tests/jupiter-arb-test`):
- Parallel quote fetching across 3 LSTs × 10 trade sizes per cycle
- Multi-variant Jito bundle submission (tip sweep: 0.3× / 0.5× / 1.0× quoted profit)
- Simulation-adjusted `quotedOut` buffer to reduce 6001 slippage failures
- `minSimProfitLamports` gate — rejects cycles where simulated profit can't survive pipeline latency
- Dark pool exclusion (`BisonFi`, `TaurusFi`) that produce unexecutable instruction formats

**Supporting infrastructure:**
- **Pool discovery service** (`go/internal/pool-discovery-service`) — detects new Raydium, Meteora, Orca pools on-chain in real time; 1,336 pools indexed in Redis
- **RPC proxy** (`rust/solana-rpc-proxy`) — connection pooling, load balancing, request routing across multiple RPC endpoints
- **Pool enricher** — Solscan API + Puppeteer for enriching pool metadata (TVL, volume, fee tier)
- **Meteora LP manager** (`ts/tests/meteora-lp-manager`) — position setup, withdrawal, and rebalancing for DLMM strategies
- **Grafana LGTM+ stack** — Loki, Mimir, Tempo, Pyroscope; complete distributed tracing, metrics, logs, and continuous profiling
- **OpenClaw AI gateway** — DeepSeek via Ollama, Telegram bot for real-time alerts and natural language queries about system state
- **Puppeteer service** — stealth headless Chromium microservice for fetching from Cloudflare-protected DeFi APIs

```
Configuration (.env):
MIN_SIM_PROFIT_LAMPORTS=50000    # ~$0.008 minimum after simulation
TIP_FRACTION=0.30
SLIPPAGE_BPS=50
SUBMISSION_MODE=race             # Jito + RPC simultaneously
```

All of this was built with Claude Code as primary development agent — four specialised skill roles (Solution Architect, Go Developer, TypeScript Developer, Rust Developer) with persistent memory across sessions, and multiple frontier LLMs (ChatGPT, Grok, Qwen, DeepSeek) for architecture review at each phase.

---

## The Market Condition Problem

One of the key practices in this project has been using **multiple AI platforms as independent reviewers** — not just for code, but for architectural and strategic decisions. ChatGPT, Grok, Qwen3-Max, and DeepSeek were each consulted at different phases to validate design choices and flag blind spots.

Mid-project, one consistent signal emerged across these reviews: **revisit the market regime before committing further to circular arb**. Qwen3-Max in particular flagged market regime awareness as a critical gap — pointing out that strategy viability can shift significantly with changes in volatility, liquidity concentration, and bot density on-chain.

That warning turned out to be correct.

LSTs like mSOL, JitoSOL, and bSOL have a **deterministic exchange rate** with SOL — staking yield accrual is predictable to four decimal places. In 2024, small pricing inefficiencies persisted long enough for even a slow scanner to catch them. By 2025-26, those gaps close in milliseconds. The pool count per pair has also collapsed — the local quote service found only 1-2 pools per LST direction. With a single pool, "circular arb" is just paying fees twice.

---

## The Pipeline Challenge: Speed and Dark Pools

After running the production engine for an extended test period, the results pointed to some fundamental difficulties with a home-settings setup.

The pipeline latency with a typical home/cloud setup looks like this:

```
Average total latency (sim-pass cases):  ~2,300ms

  Quote fetch (parallel): ~400ms
  Swap build:             ~200ms
  Simulation:             ~300ms
  Submission:             ~100ms
  Block inclusion:        ~1,300ms
```

At the price velocity typical in active SOL pools, a ~2,300ms pipeline means prices may have moved significantly between quote and execution — much of the simulated profit can be eroded by the time the transaction lands. The simulation pass rate was very low, and on-chain success in the test run was limited.

In contrast, a bot with co-location infrastructure — Shredstream access, a validator-adjacent RPC node, pre-signed transactions — operates on a completely different timescale, likely in the tens of milliseconds. The infrastructure gap is substantial, and it isn't something you can close with better code alone.

There's also the dark pool problem. A large proportion of simulation failures were 6001 slippage errors from routes through pools like HumidiFi, GoonFi, ZeroFi, and SolFi — these use off-chain order books. Jupiter quotes them against current order book state, but on-chain simulation can't see it. Every route touching a dark pool fails simulation unconditionally, regardless of the quoted profit.

---

## Bot Tracing: Watching a Competitor in the Wild

Rather than just giving up, we built one more tool: **`bot-tracing-test`** — a passive wallet observer that polls `getSignaturesForAddress` every 2 seconds and decodes all Jupiter swap events from confirmed transactions.

```typescript
// Observation only. No simulation. No execution.
// Polls getSignaturesForAddress every 2s
// Fetches each full transaction, parses swap behaviour
// from pre/post token balances + Jupiter SwapEvent inner instructions
// Writes structured observations to JSONL log
```

We identified one wallet operating a circular arb bot that appeared genuinely profitable. The trace data from a single session showed consistent on-chain circular trades — SOL in, SOL out, positive `profitAmount` on each. Real executed profit, not just quoted profit. Typical examples from the logs:

```
0.3507 SOL → [USDT → USDC → cbBTC → WBTC → JitoSOL] → 0.3507 SOL  (+70,537 lamports, 6 hops)
0.3760 SOL → [USDC → BONK → SOL]                                    (+48,854 lamports, 3 hops)
0.0940 SOL → [USDC → PYUSD → USDT → SOL]                           (+10,654 lamports, 5 hops)
0.0532 SOL → [USDC → USDT → SOL]                                    (+14,984 lamports, 3 hops)
```

The Jito tips ranged from 2,153 to 52,548 lamports — small, competitive bids consistent with a bot that knows exactly what opportunities are worth.

**But here's the thing:** when we extrapolated across the observed transaction frequency and average profit per trade, the estimated monthly earnings came to roughly **$30/month**.

That's not nothing — but it's also not a business. And critically, to achieve even that, the bot almost certainly **co-locates with a Solana RPC node**. The transaction timing patterns suggest block-level awareness that you only get from Shredstream or a validator-adjacent setup. The infrastructure cost to replicate that likely exceeds the $30/month in profit.

---

## What Went Wrong Technically

Beyond the latency problem, three specific bugs affected the local quote service:

**1. Meteora DAMM v1 pool math is wrong.** The implementation reads vault LP token balances as if they were direct token reserves. They're not — you need to look up the underlying vault to compute actual reserves:

```
Actual reserve A = vault_total_tokens × (pool_lp_balance / vault_lp_supply)
```

Without this, the pool showed a fake 2.8–3% SOL/USDC arb — not real.

**2. Oracle price feed was stale.** The NATS oracle published SOL at ~$91.72 (timestamp `1970-01-01`) when the real price was ~$160+. This blocked most pools via deviation filtering and required a workaround that weakened the quality gate.

**3. Pool loading coverage was only 14%.** Of 1,336 pools in Redis, only 187 loaded successfully. The remaining 1,149 hit "not found in any supported protocol" — mostly pancakeswapv3 (not implemented), meteora_damm_v1 (wrong math), and various variant mismatches.

---

## What's Worth Saving

The infrastructure is not wasted — it's directly reusable:

| Component | Status | Reuse |
|-----------|--------|-------|
| Jupiter swap execution | Working | All strategies need fills |
| Jito bundle submission (multi-variant) | Working | Any MEV-sensitive execution |
| Simulation + 6001 error handling | Working | Any transaction submission |
| Local quote service (gRPC + HTTP) | Working (partial DEX coverage) | Fast pre-screening for LP rebalancer |
| Pool quality manager | Working | Pool health monitoring for LP strategies |
| Orca/Raydium pool math | Verified correct | DLMM rebalancer inputs |
| Scanner service (9.4M quotes validated) | Working | Opportunity detection for any strategy |
| Pool discovery service | Working | New pool detection for momentum strategy |
| RPC proxy (Rust) | Working | Low-latency RPC for any service |
| Meteora LP manager | Working | Foundation for DLMM rebalancer |
| bot-tracing-test | Working | Ongoing competitive intelligence |
| Grafana LGTM+ observability stack | Working | Monitor any new strategy |
| OpenClaw AI gateway | Working | Operational monitoring and alerts |

---

## Where We Go Next

### Strategy 1: DLMM Automated Liquidity Provision

Instead of racing bots, become the liquidity they trade against. Deploy capital into Meteora DLMM bins and earn fees from every swap through the range — no latency competition required.

```typescript
// Rebalancer logic (extending existing meteora-lp-manager):
1. Monitor current price vs position range
2. If price exits range → trigger rebalance
3. Select new range based on 1σ daily volatility (target 95% in-range time)
4. Auto-compound earned fees
5. Pause during extreme volatility (>5% in 1h)
```

Expected returns on active SOL/USDC pools: 50–200% APY on in-range capital. The existing `meteora-lp-manager` already handles position setup — the rebalancer is a 1–2 week build on top.

### Strategy 2: Funding Rate Arbitrage (Delta-Neutral)

Hold staked SOL (JitoSOL/mSOL) for ~8% staking APY, short SOL perpetuals on Drift Protocol. Net position is delta-neutral — no directional SOL exposure. In bull markets, longs pay shorts positive funding rates of 30–500% APY on top of staking yield.

```
Open position when:  drift_hourly_funding_rate > 0.01%  (= 87.6% APR)
Close position when: drift_hourly_funding_rate < 0.001% (market balanced)
```

No speed advantage needed — funding accrues hourly.

### Strategy 3: New Token Momentum

The pool-discovery-service already detects new Meteora/Raydium pool launches in real time. New pools have genuine price discovery — not pre-arbed by bots. Volume momentum in the first 30 minutes is observable on-chain and can be a profitable entry signal with strict position sizing (1–2% of capital per trade).

We may revisit circular arb later, with better infrastructure — Shredstream access, a validator-adjacent RPC, and pre-signed transactions. The knowledge gained from this investigation will directly inform that effort.

---

## Stepping Back: The Bigger Picture

At the same time as pivoting the trading strategy, it's also worth stepping back to look at the broader career direction this project points toward.

The traditional software development job market is contracting. AI is accelerating that shift. But it's also opening a different kind of opportunity: **AI engineering** — deploying AI systems in production environments, building agentic pipelines, integrating LLMs with real infrastructure. As described in my recent LinkedIn article [*From Full-Stack Developer to AI Engineer: Is Now the Right Time to Make the Move?*](https://www.linkedin.com/pulse/from-full-stack-developer-ai-engineer-now-right-time-make-james-shen-pshcc/), this is the #1 fastest-growing job category in Australia for 2026, and the window for leveraging senior engineering experience at premium rates is estimated at 12–18 months before the market saturates.

This project has been an accidental portfolio for exactly that credential: agentic systems, multi-LLM architecture validation, polyglot microservices, production observability. The next step is to make it explicit — pursuing **Azure AI-102** (AI Engineer Associate) and **AWS MLA-C01** (ML Engineer Associate) certifications, which align directly with Australian employer requirements.

Alongside that, continuing to research HFT and DeFi market structure — understanding Shredstream, validator co-location economics, and order flow dynamics — builds a knowledge base that would make a future re-entry into arb strategies more informed.

---

## Lessons Learned

**1. Test market conditions before going deep.** The 2024 prototype worked because conditions allowed it. A structured market regime assessment mid-project — as the AI reviewers suggested — should have happened sooner.

**2. Infrastructure gap is real.** The speed difference between a home setup and a co-located bot can't be closed with better code alone. It requires different infrastructure (Shredstream, co-location) or a different strategy (LP, funding rate arb) where speed isn't the deciding factor.

**3. Tracking competitors gives real data.** The bot-tracing experiment yielded concrete numbers — $30/month at co-location. That's more actionable than any theoretical analysis.

**4. Infrastructure compounds.** The quote service, execution engine, Jito submission, observability stack, pool math — none of this is wasted. Every component transfers directly to the next strategy.

**5. A pivot is not a failure.** The goal was to learn and build something real. Both happened. The system exists, it works, and the knowledge it generated is genuinely hard to acquire any other way.

---

## Conclusion

The early prototype making a couple of dollars a day was real. The ambition to scale that with a purpose-built production system was reasonable. But circular arbitrage on Solana isn't broken as a concept — it seems very difficult to make work with home settings. The infrastructure required to be competitive — Shredstream access, validator-adjacent RPC co-location, pre-signed transactions ready to fire in milliseconds — is a non-trivial investment that changes the economics entirely.

The bot we traced is making ~$30/month and likely co-locates with an RPC node. Whether that infrastructure cost makes it worth the effort is a separate question. What we can say is: attempting the same with a standard home/cloud setup seems unlikely to be viable, at least for now.

So we pivot. DLMM liquidity provision and funding rate arbitrage are the near-term focus — strategies where being a liquidity provider or delta-neutral yield earner requires no speed advantage. HFT research and AI engineering certification run in parallel. And the door to revisiting arb remains open — but only with a clear-eyed view of what the infrastructure investment actually requires.

Every line of code written for this project is still in the repo. None of it goes to waste.

---

## Related Posts

- [Planner Validation: Arb Pipeline Route Merge, Simulation and Execution](/posts/2026/03/planner-validation-arb-pipeline-route-merge-simulation-execution/)
- [How I Built a Solana Trading System with AI as My Co-Developer](/posts/2026/03/ai-assisted-development-how-i-built-solana-trading-system/)
- [Scanner Service Production Validation: 9M Quotes, Arb Signals](/posts/2026/03/scanner-service-production-validation-9m-quotes-arb-signals/)
- [Pool Enricher: Solscan API Decoding and Puppeteer Preflight](/posts/2026/03/pool-enricher-solscan-api-decoding-puppeteer-preflight/)

## Technical Documentation

- [Circular Arb Investigation and Pivot](https://github.com/guidebee/solana-trading-system)
- [AI-Assisted Development Overview](https://github.com/guidebee/solana-trading-system)

---

## Connect

**GitHub**: [github.com/guidebee](https://github.com/guidebee) | **LinkedIn**: [linkedin.com/in/guidebee](https://linkedin.com/in/guidebee)

*This is post #32 in the Solana Trading System development series — an honest account of what didn't work, what we learned, and where we're heading next.*
