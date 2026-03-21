---
layout: single
title: "Planner Validation: Testing the Scanner→Planner→Executor Pipeline End-to-End"
date: 2026-03-21
permalink: /posts/2026/03/planner-validation-arb-pipeline-route-merge-simulation-execution/
categories:
  - blog
tags:
  - solana
  - trading
  - architecture
  - typescript
  - jupiter
  - arbitrage
  - testing
  - infrastructure
excerpt: "A systematic validation campaign for the planner (strategy service) layer: subscribing to live arb events, converting to Jupiter swap instructions, merging two-hop route instructions into a single atomic transaction, simulation, and on-chain execution — including confirmed DEX compatibility, token ledger chaining, and the Jito bundle rate-limit wall we hit."
---

## TL;DR

- **6-step test harness** validates the full scanner→planner→executor pipeline from live NATS events to on-chain execution
- **3 local protocols confirmed** execution-compatible (orca_whirlpool 41%, meteora_damm_v1 30%, raydium_amm 21%) — covering ~92% of all arb quotes
- **Route-merge mechanism validated**: all 9 same/cross-DEX combinations of the 3 local protocols simulate and execute on-chain; plus **15 external Jupiter DEXes** confirmed compatible in mixed cross-source tests (81/85 pairs passed)
- **SOL↔USDC is the focused token pair** — all 57k+ arb events target the SOL/USDC circular route; other token pairs were out of scope for this validation campaign
- **Jito bundles deferred**: infrastructure works but bundles don't land at minimum tip — rate-limit issues persist even with `JITO_UUID`; will revisit with competitive tip strategy

---

## Background: What We're Validating

The trading system is structured as a three-stage pipeline:

```
scanner-service  →  planner (strategy-service)  →  executor-service
     │                       │                            │
  Detects arb            Decides HOW to               Submits tx
  opportunities          execute (route-merge          on-chain
  via pool quotes         vs token-ledger,
  → NATS events           which DEXes, amounts)
```

The scanner service ([post #24](/posts/2026/03/scanner-service-production-validation-9m-quotes-arb-signals/)) has been running in production, emitting `TwoHopArbitrageEvent` messages over NATS JetStream. Before building the full planner service, we needed to validate every assumption about execution: Can we convert scanner events into executable transactions? Which DEX combinations merge cleanly? Does simulation match execution? What is the actual execution overhead?

This post documents that validation campaign — two test packages (`swap-test` and `jupiter-test`) run against mainnet over ~1 week.

---

## The Test Architecture

### swap-test: 6-Step Pipeline Harness

`swap-test` is a step-by-step integration harness where each step builds on the previous one. You run it with `STEP=N pnpm start:dev`:

```
NATS Stream (TwoHopArbitrageEvents)
        │
        ▼
Step 1  Subscribe & inspect live arb events
        │
        ▼
Step 2  Convert events → Jupiter QuoteResponse format
        │
        ▼
Step 3  Call /swap-instructions, decode & merge Route instructions
        │
        ▼
Step 4  Save reusable transaction template to disk
        │
        ▼
Step 5  Simulate the merged swap transaction via RPC (no SOL spent)
        │
        ▼
Step 6  Execute the swap transaction on-chain
```

Steps 1–4 require a live NATS connection and the scanner service. Steps 5–6 only need the saved template file and an RPC endpoint — no NATS required.

### jupiter-test: DEX Compatibility Matrix

`jupiter-test` runs controlled experiments independent of the live event stream, covering:

- **`local-dex-merge-test`**: all 9 combinations of the 3 local protocols in both directions
- **`mixed-dex-merge-test`**: 90 cross-source pairs (3 local × 15 Jupiter DEXes × 2 directions)
- **`dex-discovery-test`**: scanned all 76 IDL `Swap` variant names against Jupiter's `dexes=` filter
- **`bundle-test`**: token-ledger chaining (two separate instructions in one transaction)
- **`jito-bundle-test`**: Jito bundle submission (two independent transactions as a bundle)

---

## Step 1: Confirming the Arb Event Stream

The first thing to validate was that the scanner is publishing well-formed events and that the quality is production-grade.

Step 1 subscribes to two JetStream consumers in parallel — one for `LOCAL` events (Go quote cache wins) and one for `EXTERNAL` events (Jupiter API wins) — and prints a rolling key-findings table every 50 events:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Key Findings — Step 1 (Arb Events)                                           │
├─────────────────────────────────┬────────────────────────┬───────────────────┤
│ Metric                          │ Value                  │ Assessment        │
├─────────────────────────────────┼────────────────────────┼───────────────────┤
│ Total Events                    │ 57,551                 │ ✅ Excellent       │
│ Test Duration                   │ ~52 hours              │                   │
│ Events/Second                   │ ~0.31                  │ ✅ Consistent      │
│ Valid Metadata                  │ 100.0%  (57551/57551)  │ ✅ PERFECT        │
│ Valid Routes                    │ 100.0%  (57551/57551)  │ ✅ PERFECT        │
│ High Price Impact (>5%)         │ 0 events  (0.0%)       │ ✅ ZERO CRITICAL  │
│ Expired on Receipt              │ 0 events  (0.0%)       │ ✅ FRESH          │
├─────────────────────────────────┼────────────────────────┼───────────────────┤
│ Local Arbs                      │ ~46% (26,473)          │ Balanced ✅       │
│ External Arbs                   │ ~54% (31,078)          │ Slight ext edge   │
├─────────────────────────────────┼────────────────────────┼───────────────────┤
│ Local Avg Profit                │ ~244 bps (days 1,3)    │ ✅ STRONG         │
│ External Avg Profit             │ ~8–12 bps (days 2,4)   │ ⚠️  LOW           │
│ Oracle Price Coverage           │ 100.0%                 │ ✅ PERFECT        │
└─────────────────────────────────┴────────────────────────┴───────────────────┘
```

**Focused on SOL/USDC.** This validation campaign targets the SOL/USDC circular route (SOL→USDC→SOL) exclusively. All 57,551 events processed over 4 days are SOL/USDC arb opportunities. Other token pairs in the configuration (JitoSOL, mSOL, sSOL, bSOL, INF, JUP, BONK, USDT) were out of scope for this phase — the planner validation was designed to prove the execution mechanics on the highest-liquidity pair first before expanding token coverage.

The source-tag pattern is also notable: LOCAL dominates on some days (up to 0.607 SOL quoted profit, 450+ bps average), while EXTERNAL dominates on others with much smaller spreads (~10K lamports, ~8 bps). This alternates roughly daily, correlating with quote service restart cycles rather than market conditions.

---

## Step 2: Event → QuoteResponse Conversion

The scanner emits `TwoHopArbitrageEvent` FlatBuffer messages containing the pool IDs, amounts, and protocol labels for both legs. Step 2 validates that these can be converted into Jupiter `QuoteResponse` objects — the format required by `/swap-instructions`.

The key field is `ammKey = pool_id` from the local service. Jupiter's `/swap-instructions` uses this to look up the pool on-chain and Borsh-encode the instruction. This means we never call Jupiter's `/quote` endpoint — the local service drives all price discovery and pool selection. Jupiter acts purely as an **instruction serializer**.

```typescript
// Synthetic QuoteResponse built from scanner event data
const quote: QuoteResponse = {
    inputMint,
    inAmount: String(inAmount),
    outputMint,
    outAmount: String(outAmount),
    routePlan: [{
        swapInfo: {
            ammKey: pool_id,   // ← local service pool ID, not Jupiter's routing
            label,
            inputMint,
            outputMint,
            inAmount: String(inAmount),
            outAmount: String(outAmount),
            feeAmount: String(Math.floor(inAmount * feeBps / 10_000)),
            feeMint: inputMint,
        },
        percent: 100,
    }],
    // ...
};
```

Step 2 outputs `logs/step2-*.jsonl` with conversion status per event. 100% conversion success rate was observed for events from the 3 confirmed protocols.

---

## Step 3: The Route-Merge Mechanism

This is the core of the planner's execution strategy. The idea: instead of two separate swap transactions, merge both legs of the circular arb into a **single atomic instruction**. Either both legs execute atomically or neither does — no partial execution, no intermediate USDC held between blocks.

### How the Merge Works

```
Leg 1 (SOL→USDC): POST /swap-instructions  → SwapInstructionsResponse (25 accounts, Route ix)
Leg 2 (USDC→SOL): POST /swap-instructions  → SwapInstructionsResponse (25 accounts, Route ix)
                                                │
                              mergeRouteInstructions(instr1, instr2)
                                                │
                              Combined: 25 + 16 = ~41 accounts
                              routePlan: [leg1 steps..., leg2 steps...]  (reindexed)
                              lastStep.outputIndex = 0  (circular: return to source ATA)
```

The merge algorithm:

```typescript
// 1. Combined accounts: all of leg1 + leg2 DEX accounts only (skip first 9 fixed)
combinedAccounts = [
    ...instr1.swapInstruction.accounts,           // all 25 (9 fixed + 16 DEX)
    ...instr2.swapInstruction.accounts.slice(9),  // leg2 DEX accounts only
];

// 2. Reindex leg2 routePlan to avoid collision with leg1 indices
lastOutputIndex = leg1.routePlan.last.outputIndex;  // typically 1
leg2.routePlan.forEach(step => {
    step.inputIndex  += lastOutputIndex;
    step.outputIndex += lastOutputIndex;
});

// 3. Force circular: last step output returns to index 0 (source ATA)
mergedRoutePlan.last.outputIndex = 0;
```

### The Critical Account Fix

After merging, Jupiter's fixed accounts from leg1 contain the wrong values for a circular arb (they assume USDC as the final destination). Without correction, Jupiter rejects with **Custom(6010) InvalidRoutePlan**:

| Account | Jupiter Returns (leg1) | Required for Circular Arb |
|---------|----------------------|--------------------------|
| `accounts[3]` userDestination | USDC ATA | **wSOL ATA** (circular final dest) |
| `accounts[5]` destinationMint | USDC mint | **wSOL mint** |

The fix is straightforward — detect the marker (`accounts[4] === Jupiter Program ID`, which is only present when `useSharedAccounts: false`) and overwrite:

```typescript
if (accounts[4]?.pubkey === JUPITER_PROGRAM_ID) {
    accounts[3] = { pubkey: wsolAta, isWritable: true,  isSigner: false };
    accounts[5] = { pubkey: SOL_MINT, isWritable: false, isSigner: false };
    // accounts[4] stays as Jupiter Program ID — DO NOT change it
}
```

The error progression during debugging shows how each fix layers in:

| State | Error | Root Cause |
|-------|-------|------------|
| No fix | `6010 InvalidRoutePlan` | accounts[3]=USDC ATA ≠ last step wSOL output |
| accounts[3]=wSOL, [4]=USDC ATA, [5]=wSOL | `Custom(3) MintMismatch` | accounts[4].mint (USDC) ≠ accounts[5] (wSOL) |
| accounts[3]=wSOL, [4]=unchanged, [5]=wSOL | `6001 SlippageExceeded` | Structure valid, quoted profit too stale |
| Above + `outAmt=1n` | **✅ SUCCESS** | Simulation passes |

The `/swap-instructions` call parameters matter: `useSharedAccounts: false` is required — it is what produces `accounts[4] = Jupiter Program ID` as the detectable marker for the fix.

---

## Step 4–5: Template Save and Simulation

Step 4 saves a fully-decoded transaction template to disk — all Borsh-decoded `RouteInstructionData`, combined accounts, and ALT (Address Lookup Table) addresses. Steps 5 and 6 load this file directly; no NATS connection or Jupiter calls needed.

Step 5 simulation tests 5 amount variations (100%, 99.99%, 100.01%, 95%, 90% of `inAmount`) and stops at the first that passes:

```
  VARIATION 1/5  inAmount=500000000  quotedOut=500010000
  tx size          : 847 bytes ✅
  compute units    : 142831
  simulation       : ✅ SUCCESS

  SIMULATION PASSED ✅
  Amount           : 500000000
  Quoted out       : 500010000
  Compute units    : 142831
  TX size          : 847 bytes
```

The template must be fresh (seconds old) for simulation to reflect current on-chain state. Stale templates fail with `Custom(6001)` when the opportunity has passed.

---

## DEX Compatibility: local-dex-merge-test

Before running the live pipeline, we tested all 9 combinations of the 3 local protocols in isolation using `local-dex-merge-test`. These are the protocols that account for ~92% of all scanner arb events:

| Protocol | Jupiter DEX Label | IDL `__kind` | Quote Share |
|---|---|---|---|
| `orca_whirlpool` | Whirlpool | `Whirlpool` / `WhirlpoolSwapV2` | 41% |
| `meteora_damm_v1` | Meteora | `Meteora` | 30% |
| `raydium_amm` | Raydium | `Raydium` | 21% |

**All 9 cross-DEX combinations passed simulation AND on-chain execution:**

| Leg 1 (SOL→USDC) | Leg 2 (USDC→SOL) | Kind | Sim | Execute |
|---|---|---|---|---|
| orca_whirlpool | orca_whirlpool | same-dex | ✅ | ✅ |
| meteora_damm_v1 | meteora_damm_v1 | same-dex | ✅ | ✅ |
| raydium_amm | raydium_amm | same-dex | ✅ | ✅ |
| orca_whirlpool | meteora_damm_v1 | **cross-dex** | ✅ | ✅ |
| orca_whirlpool | raydium_amm | **cross-dex** | ✅ | ✅ |
| meteora_damm_v1 | orca_whirlpool | **cross-dex** | ✅ | ✅ |
| meteora_damm_v1 | raydium_amm | **cross-dex** | ✅ | ✅ |
| raydium_amm | orca_whirlpool | **cross-dex** | ✅ | ✅ |
| raydium_amm | meteora_damm_v1 | **cross-dex** | ✅ | ✅ |

One pool compatibility issue was discovered during this run: two Whirlpool pools had exited their tick range after SOL crossed $84 and were rejected by Jupiter with `MARKET_NOT_FOUND`. Both were added to the pool blocklist in `go/pkg/config/filteredpools.go`. After the blocklist update, all 9/9 pairs passed.

**Why Meteora DAMM v1 works but Meteora DLMM does not**: DAMM v1 uses a constant-product curve — accounts are stateless relative to the swap path. DLMM uses bin arrays that are path-dependent; forcing `outputIndex=0` in the merged routePlan creates an invalid bin traversal that Jupiter rejects at route plan validation.

---

## DEX Compatibility: mixed-dex-merge-test

The mixed test validated cross-source pairs where one leg comes from the local quote service and the other comes directly from Jupiter's routing across 15 confirmed external DEXes:

```
# 15 confirmed external DEXes (dex-discovery-test 2026-03-21)
Raydium, Aldrin, Whirlpool, Invariant, Meteora, Phoenix, Perps,
Woofi, TesseraV, SolFi V2, DefiTuna, WhaleStreet, Manifest, Quantum, Scorch
```

**Test matrix**: 3 local protocols × 15 external DEXes × 2 directions = 90 pairs.
**Result: 81/85 passed** (5 skipped for transient no-route, 4 failed — all transient):

| Direction | Pairs | Passed | Failed | Skipped |
|---|---|---|---|---|
| local → jupiter | 45 | 43 | 2 | 0 |
| jupiter → local | 45 | 38 | 2 | 5 |
| **Total** | **90** | **81** | **4** | **5** |

All 4 failures were transient (pool state changed between simulation and execution, or edge pool condition). The same direction with a different local protocol passes, confirming no structural incompatibility.

The key architectural confirmation: **mixed-source route-merge works**. The local service provides the pool ID and amounts for one leg; Jupiter provides the other leg's routing. Both legs are serialized via `/swap-instructions` and merged. The resulting transaction is no different from a same-source merge — Jupiter doesn't know or care that one ammKey came from a local cache.

---

## Token Ledger Chaining: An Alternative Execution Path

The route-merge approach is the primary execution method — single instruction, minimal CU, proven across all 9 local combinations and 15 external DEXes. But we also tested a second approach: **token-ledger chaining**, where two separate swap instructions are placed sequentially in one transaction.

The mechanism: `useTokenLedger: true` on leg 2, combined with a `tokenLedgerInstruction` that snapshots the USDC balance before leg 1 runs. Leg 2 uses the delta as its exact input amount, eliminating slippage between the legs.

Test matrix: 40 cases across `bundle-test`. **12 pairs confirmed stable across two independent sessions:**

```
# Stable local×local
meteora_damm_v1 → raydium_amm
raydium_amm     → orca_whirlpool

# Stable local→jupiter
raydium_amm     → Whirlpool / Phoenix / Woofi / Aldrin
meteora_damm_v1 → Aldrin / Woofi / DefiTuna / Quantum

# Stable jupiter→local
jupiter(Phoenix)  → local(orca_whirlpool)
jupiter(DefiTuna) → local(raydium_amm)
```

**Execution method decision tree for the Planner:**

```
Is protocol pair in confirmed route-merge list?
  YES → use route-merge  (lower CU, all 9 local combinations confirmed)
  NO  → Is pair in confirmed token-ledger stable list?
    YES → use token-ledger
    NO  → skip / Jupiter-only fallback
```

Route-merge is preferred — it uses fewer compute units and is structurally simpler. Token-ledger is the fallback for combinations where route-merge has issues.

---

## Jito Bundles: Deferred

We also tested Jito bundle submission — two independent signed transactions sent as a bundle for atomic MEV protection without route merging.

**What worked:**
- Both transactions simulate successfully on-chain
- `JitoClient.sendBundle` reaches the block engine and receives a valid `bundle_id`
- Bundle status polling via `getInflightBundleStatuses` works

**What did not work:**
- **Bundles consistently fail to land** — status never reaches `Landed`; bundles expire or show `Failed`
- Root cause: tip too low. Even at 1000 lamports (the documented minimum), bundles don't win slots during normal network activity

Three infrastructure issues were resolved along the way:

| Issue | Root Cause | Fix |
|---|---|---|
| `Transaction is missing signatures` | `setTransactionMessageFeePayer(address)` sets fee payer as static address; signer must be embedded in an instruction | Added 1-lamport self-transfer (`source: wallet`) to each tx |
| `-32097 globally rate limited` on all endpoints | 30s HTTP timeout × 8 endpoints = saturating rate buckets | Reduced timeout to 8s; added 600ms delay between retries |
| UUID not bypassing rate limit | UUID must be sent as **both** `x-jito-auth` header **and** `?uuid=` query param | Updated `JitoClient` to send both simultaneously |

The Jito rate limits with UUID: 5 req/s per IP per region (8 regions are independent). The client code is complete and functional. The bundle submission path is deferred until we implement dynamic tip calculation — competitive tips (10K–100K lamports) are likely the primary factor for landing rate.

---

## Protocol Execution Compatibility Architecture

A key architectural decision from this testing campaign: how to handle the ~25 protocols in `pools.toml` where most cannot actually execute?

The failure modes break down cleanly by layer:

| Protocol Group | Quote | Execute | Root Cause |
|---|---|---|---|
| `orca_whirlpool`, `meteora_damm_v1`, `raydium_amm` | ✅ | ✅ | Fully compatible |
| `meteora_dlmm` | ✅ | ❌ | Bin arrays path-dependent; `outputIndex=0` invalid |
| `pump_amm`, `pump_bonding_curve` | ✅ | ❌ | Meme token scope only, not SOL/USDC |
| `humidifi`, `goonfi`, `zerofi` | ❌ (no local) | ✅ via Jupiter | Dark pools — Jupiter API only |
| All others | varies | ❌ | No SOL/USDC liquidity or Jupiter `MARKET_NOT_FOUND` |

The chosen solution: add an `execution_compatible` flag per protocol in `pools.toml`. Pool discovery remains broad — all protocols are still tracked for analytics and monitoring. The quote service reads this flag and only generates arb events for compatible protocols. The planner never sees an event it cannot execute.

```toml
[pools.orca_whirlpool]
execution_compatible = true   # Confirmed: route-merge + token-ledger

[pools.meteora_dlmm]
execution_compatible = false  # Bin arrays incompatible with circular arb merge

[pools.raydium_clmm]
execution_compatible = false  # Untested — promote after validation run
```

This keeps the discovery layer maximally broad while giving the execution layer a clean, single-source-of-truth gate. As new protocols are validated, they get promoted to `true` with a 30-minute test run.

---

## Step 6: On-Chain Execution

Step 6 loads a template, simulates the lead variation, then submits up to 8 amount variations (100%, 99.99%, 100.01%, 95%, 90%… of `inAmount`) via `sendTransaction` to maximise landing probability.

Execution flow:
1. Load template from disk
2. Simulate lead variation — **abort if simulation fails** (safety gate)
3. Update compute unit limit from simulation result (+10% buffer)
4. Submit up to 8 signed variations with `sendTransaction`
5. Print confirmation for each submitted transaction

The template must be fresh (seconds old) for the transaction to land profitably. The step is intentionally not automated in the test harness — each execution requires an explicit run, making it safe to test without burning SOL on stale opportunities.

---

## What Comes Next

This test campaign answers all the key questions for building the planner service:

1. **Which protocols to use**: orca_whirlpool, meteora_damm_v1, raydium_amm — the `execution_compatible` flag formalises this
2. **Which execution method**: route-merge is primary (all 9 combinations confirmed); token-ledger is secondary for specific pairs
3. **Which external DEXes**: 15 confirmed for mixed cross-source merge; full list in `circular-arb-merge-findings.md`
4. **Token pairs**: SOL/USDC for this validation phase — other token pairs are out of scope until execution mechanics are fully proven
5. **Jito**: deferred until competitive tip strategy is built; the infrastructure is ready

The next step is wiring this validated execution logic into the actual planner service — consuming live `TwoHopArbitrageEvent` messages from NATS, routing to the correct execution method, and submitting transactions with appropriate compute budget and priority fees.

---

## Conclusion

The planner validation campaign confirmed that the route-merge approach is solid: merge two single-hop Jupiter swap instructions into one atomic circular-arb instruction, apply the circular-arb account fix, simulate, execute. It works across all 9 local DEX combinations, all 15 confirmed external DEXes in cross-source mode, and produces clean on-chain transactions without a ledger account.

57,551 events over 4 days, all SOL/USDC. The deliberate focus on one pair kept the validation clean — prove the execution mechanics end-to-end before expanding token coverage. One pair, three protocols, one merge method. The complexity budget goes into latency, not breadth.

---

## Related Posts

- [Scanner Service Production Validation: 9M Quotes and Real Arb Signals](/posts/2026/03/scanner-service-production-validation-9m-quotes-arb-signals/) — post #24
- [Pool Enricher Deep Dive: Decoding Solscan's Binary API and the Preflight Pattern](/posts/2026/03/pool-enricher-solscan-api-decoding-puppeteer-preflight/) — post #26
- [Quote Service Production Validation: Chinese New Year Test Run](/posts/2026/02/quote-service-production-validation-chinese-new-year/) — post #23
- [Project Milestone: Complete Infrastructure Ready for Arbitrage Phase](/posts/2026/01/project-milestone-complete-infrastructure-ready-for-arbitrage-phase/) — post #20

## Technical Documentation

- [swap-test harness](https://github.com/guidebee/solana-trading-system/tree/main/ts/tests/swap-test)
- [jupiter-test harness](https://github.com/guidebee/solana-trading-system/tree/main/ts/tests/jupiter-test)
- [Circular Arb Merge Findings](https://github.com/guidebee/solana-trading-system/blob/main/ts/tests/jupiter-test/docs/circular-arb-merge-findings.md)
- [Execution Protocol Strategy](https://github.com/guidebee/solana-trading-system/blob/main/ts/tests/jupiter-test/docs/execution-protocol-strategy.md)

---

*This is post #27 in the Solana Trading System development series. Follow along on [GitHub](https://github.com/guidebee) or [LinkedIn](https://linkedin.com/in/guidebee).*
