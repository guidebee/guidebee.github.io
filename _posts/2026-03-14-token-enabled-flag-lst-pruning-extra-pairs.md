---
layout: single
title: "Token Configuration Overhaul: Pruning 9 Dead LSTs and Adding Extra Token Pairs"
date: 2026-03-14
permalink: /posts/2026/03/token-enabled-flag-lst-pruning-extra-pairs/
categories:
  - blog
tags:
  - solana
  - trading
  - scanner-service
  - quote-service
  - arbitrage
  - architecture
  - performance
  - testing
  - market-data
excerpt: "Data-driven token config cleanup: disabled 9 LSTs that contributed zero arb signals across 373,000+ events, kept the 5 that matter, and expanded monitoring to 6 extra token pairs (BONK, WIF, JUP, RAY, ORCA, PYTH) via a new enabled flag system. Results to follow."
---

## TL;DR

- **Data-driven LST pruning**: Disabled 9 of 14 LSTs that produced **zero arbitrage signals** across 373,000+ arb events — cutting gRPC quote streams from 14 → 5 (64% reduction)
- **New `enabled` flag**: Tokens stay in the registry for logging and future re-evaluation; they're just excluded from live quote streams — no code changes needed to re-enable
- **Extra token pairs added**: 6 non-LST tokens (BONK, WIF, JUP, RAY, ORCA, PYTH) now monitored via a new `extra_token_pairs.json` config with its own three-layer enable gate
- **Testing underway**: The new pair configuration is live — arb data will be reported back in a couple of days

---

## Background: When 14 Streams Only 3 Do Any Work

Following the 9.4M-quote production validation ([post #29](/posts/2026/03/scanner-service-production-validation-9m-quotes-arb-signals/)), we had enough data to ask a harder question: of the 14 LST tokens being monitored, how many are actually contributing?

The answer came from analysing **373,000+ fresh arb events** across 6 sessions (March 2–3, 2026 data):

| Token | % of Arb Events |
|-------|----------------|
| sSOL  | 7.94% |
| mSOL  | 2.11% |
| JupSOL | < 0.01% |
| All other 11 LSTs | **0%** |

Eleven tokens. Zero contribution. Each one was consuming a gRPC quote stream, burning Jupiter API rate-limit budget, and occupying Go goroutines — for nothing.

---

## The Fix: `enabled` Flag in Token Config

Rather than deleting dead tokens from `tokens.json` (which would lose their metadata and make re-evaluation harder), we added an `"enabled"` boolean flag to each LST entry.

```json
{
  "symbol": "stSOL",
  "mint": "7dHbWXmci3dT8UFYWYZweBLXgycu7Y3iL6trKn1Y7ARj",
  "decimals": 9,
  "enabled": false
}
```

The `getLSTTokens()` function filters `enabled !== false` by default. Disabled tokens stay in the registry — accessible for admin tooling and dashboards via `getLSTTokens(true)` — but never enter the gRPC quote path.

No code changes are needed to re-enable a token: flip the flag, restart the scanner, done.

---

## Current LST Configuration

### Enabled (5 active quote streams)

| Token | Reason Kept |
|-------|-------------|
| JitoSOL | Largest LST by TVL (~$1B+); deep pools |
| mSOL | 2.11% of observed arb events; proven contributor |
| sSOL | 7.94% of observed arb events; top LST arb source |
| bSOL | Top-5 LST by TVL; active BlazeStake pools |
| INF | Infinity multi-LST basket; broad pool coverage |

### Disabled (9 config-only, no quotes)

| Token | Reason Disabled |
|-------|-----------------|
| JupSOL | < 0.01% arb events despite large TVL; spreads too tight |
| stSOL | Lido deprecated on Solana; pool liquidity has migrated |
| hyloSOL | Zero arb contribution; thin liquidity |
| bbSOL | Zero arb contribution; niche/institutional |
| bonkSOL | Zero arb contribution; thin liquidity |
| dSOL | Zero arb contribution; thin liquidity |
| BNSOL | Zero arb contribution; thin liquidity |
| mpSOL | Zero arb contribution; thin liquidity |
| BGSOL | Zero arb contribution; niche/institutional |

---

## Impact on Quote Stream Capacity

| Metric | Before | After |
|--------|--------|-------|
| Active LST streams | 14 | **5** |
| Go bi-directional pairs | 28 | **10** |
| Jupiter API budget freed | — | **~64%** |

That freed capacity goes directly to the pairs that matter: SOL/USDC, SOL/USDT, and now the new extra token pairs.

---

## New: Extra Token Pairs

With LST overhead cut, we expanded the scanner to monitor 6 non-LST tokens: **BONK, WIF, JUP, RAY, ORCA, PYTH**. These are meme coins, DeFi governance tokens, and infrastructure tokens — a different class of opportunity from LST arbitrage.

These are controlled by a separate file, `config/extra_token_pairs.json`, with its own enable hierarchy:

```
config/extra_token_pairs.json  ← per-token enabled flags
       ↓
loadExtraTokenPairs()          ← shared package; returns null if missing
       ↓
generateExtraTokenPairs()      ← resolves mints; skips disabled tokens
       ↓
getAllMonitoredPairs()          ← when MONITOR_EXTRA_PAIRS=true
       ↓
BatchQuoteRequest              ← same gRPC path as LST pairs
       ↓
Go quote-service               ← pools for extra pairs already in Redis
```

### Three-Layer Enable Gate

All three must pass for a token to be actively quoted:

1. **`POOL_DISCOVERY_ENABLE_EXTRA_PAIRS`** on `pool-discovery-service` — populates Redis with pool data for extra pairs
2. **`MONITOR_EXTRA_PAIRS`** on `ts-scanner-service` — enables the scanner to request quotes for extra pairs
3. **`"enabled"` flag** in `extra_token_pairs.json` — per-token selective disable

The Go quote services required no changes — `local-quote-service` reads pools from Redis (pool-discovery already handles this), and `external-quote-service` is purely reactive to whatever pairs the scanner requests.

---

## Updated Pair Counts

| Configuration | Pairs |
|---|---|
| LST pairs only | 5 |
| + SOL/stablecoin (default) | 7 |
| + extra pairs (`MONITOR_EXTRA_PAIRS=true`) | 19 |
| + stablecoin pairs | 20 |

---

## Data Flow: Before and After

```
BEFORE (14 LSTs, no extra pairs):
───────────────────────────────────
tokens.json (14 LSTs, no enabled flag)
       ↓
getLSTTokens()  →  28 bi-directional pairs
       ↓
BatchQuoteRequest  →  gRPC to Go quote-services
       ↓
11 of 14 LSTs: 0 arb events


AFTER (5 LSTs + 6 extra tokens):
───────────────────────────────────────
tokens.json (enabled: true/false)          extra_token_pairs.json (enabled: true/false)
       ↓                                              ↓
getLSTTokens()  →  10 pairs          generateExtraTokenPairs()  →  12 pairs
                        └──────────────────────┘
                                    ↓
                           getAllMonitoredPairs()
                                    ↓
                           BatchQuoteRequest (22 pairs total)
                                    ↓
                        Go quote-services (purely reactive)
```

---

## Re-enable Criteria

Disabled tokens aren't gone forever. The re-evaluation criteria:

**Re-enable an LST when:**
- A new high-liquidity pool launches (Raydium/Orca/Meteora)
- TVL exceeds $100M (indicates meaningful pool depth)
- A protocol incentive program creates temporary spread opportunities

**Disable an extra token when:**
- Zero arb signals over a 7-day observation window
- Pools deprecated or migrated by the issuing protocol

---

## What We're Testing

The new configuration (5 LSTs + 6 extra token pairs) is now live. The core questions:

1. Do any of the extra tokens (BONK, WIF, JUP, RAY, ORCA, PYTH) produce arb signals at meaningful frequency?
2. How do their deviation profiles compare to the LST pairs?
3. Does the freed Jupiter API budget improve quote quality for the retained pairs?

Results and updated token pair statistics will be reported in a follow-up post in a couple of days once we have enough session data to draw conclusions.

---

## Conclusion

The `enabled` flag is a small configuration change with a meaningful operational impact: 64% fewer gRPC streams, zero disruption to the pairs that actually produce arb signals, and a clean path to expand monitoring into new token categories without code changes.

The real test is whether the extra token pairs justify their quota. The data will tell us.

---

## Related Posts

- [Scanner Service Production Validation: 9.4M Quotes, 106-Hour Continuous Run](/posts/2026/03/scanner-service-production-validation-9m-quotes-arb-signals/) - The validation that surfaced the LST underperformance data
- [Quote Service Production Validation: Zero Critical Deviations & Chinese New Year Milestone](/posts/2026/02/quote-service-production-validation-chinese-new-year/) - Epoch 2 baseline
- [Project Milestone Complete: Infrastructure Ready for Arbitrage Phase](/posts/2026/01/project-milestone-complete-infrastructure-ready-for-arbitrage-phase/) - Overall project status

---

## Technical Documentation

- [Token Enabled Flag — Design Doc](https://github.com/guidebee/solana-trading-system/blob/master/docs/32-LST-TOKEN-ENABLED-FLAG.md)
- [Scanner Service](https://github.com/guidebee/solana-trading-system/blob/master/ts/apps/scanner-service/)
- [Shared Token Registry](https://github.com/guidebee/solana-trading-system/blob/master/ts/packages/shared/src/tokens/)

---

## Connect

- **GitHub**: [guidebee/solana-trading-system](https://github.com/guidebee/solana-trading-system)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

*This is post #30 in the Solana Trading System development series. Data from 373,000+ arb events drove a token configuration overhaul: 9 non-contributing LSTs disabled, 5 high-signal LSTs retained, and 6 extra token pairs (BONK, WIF, JUP, RAY, ORCA, PYTH) added for evaluation. Follow-up results coming in a couple of days.*
