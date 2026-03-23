---
layout: single
title: "How I Built a Solana Trading System with AI as My Co-Developer"
date: 2026-03-23
permalink: /posts/2026/03/ai-assisted-development-how-i-built-solana-trading-system/
categories:
  - blog
tags:
  - ai-tools
  - claude-code
  - github-copilot
  - solana
  - trading
  - reflection
  - ai-engineering
excerpt: "A candid look at how I used AI tools — Claude Code, GitHub Copilot, and multiple frontier LLMs — as genuine development partners to build a production-grade Solana HFT trading system. What actually works, what doesn't, and what it means for how software gets built."
---

## TL;DR

- **AI was not a shortcut** — it was a structured development methodology that let one developer build what would normally need a team of five
- **Four specialised AI roles** replaced four human specialists: solution architect, Go developer, TypeScript developer, Rust developer
- **Multiple LLMs as reviewers** provided architecture validation that previously required a design review committee
- **The system itself is agentic** — the AI development approach shaped the runtime architecture (observe → decide → act)

---

## A Different Kind of Solo Project

In late 2025, I started building a Solana high-frequency trading system. On paper it looked like a small project. In reality it turned out to be a polyglot monorepo spanning three languages — Go, TypeScript, and Rust — with a microservices architecture, a full observability stack, sub-500ms latency targets, and live on-chain execution.

That is the kind of project that normally requires a team of five or six engineers, six to twelve months, and a proper sprint board.

I built it in four months, mostly alone.

The difference was how I used AI tools — not as autocomplete, not as a Stack Overflow replacement, but as a genuine development team where I was the lead and the AI filled the specialist roles.

I wrote about the career dimension of this transition in [From Full-Stack Developer to AI Engineer: Is Now the Right Time to Make the Switch?](https://www.linkedin.com/pulse/from-full-stack-developer-ai-engineer-now-right-time-make-james-shen-pshcc/) — the short answer being: yes, now is exactly the right time. This post goes deeper into the technical reality of what that actually looks like day to day.

---

## The Team I Never Had to Hire

The first thing I changed was how I thought about Claude Code (Anthropic's AI coding CLI). Most developers use it like a very fast search engine — ask a question, get an answer, copy some code. I treated it differently: as a set of specialist colleagues who needed proper onboarding.

I created four distinct **skill roles**, each loaded with domain-specific expertise:

**Solution Architect** — responsible for system design, technology selection, and cross-service integration decisions. When I was unsure whether to use gRPC or NATS between services, the Architect role weighed the tradeoffs in the context of *this specific system's* latency requirements and operational complexity.

**Go Senior Developer** — owned the quote service, DEX routing logic, and the pool pricing algorithms. This role knew which patterns were forbidden (blocking calls inside a hot loop), which libraries were preferred, and what 10ms latency actually means in Go concurrency terms.

**TypeScript Senior Developer** — owned the scanner, strategy, and executor services. Critically, this role knew that `@solana/web3.js` was banned in favour of `@solana/kit` — a non-obvious constraint that matters enormously for correctness and performance.

**Rust Senior Developer** — owned the RPC proxy and the Shredstream integration. This role understood zero-copy parsing, Tokio async patterns, and why naïve Rust is often slower than correct Rust.

Each skill file contained not just area responsibilities but also explicit forbidden patterns, performance targets, and naming conventions. The result: Claude Code's output was consistent with production HFT requirements rather than with generic best practices learned from Stack Overflow posts written in 2019.

---

## Memory That Survives the Session

One of the hidden costs of AI-assisted development is the re-explanation problem. Every new conversation, you spend ten minutes re-establishing context: here is the system, here is the constraint, here is why we made that choice three weeks ago. It is exhausting and error-prone — you always leave something out, and the AI makes a suggestion that contradicts a decision you already made.

Claude Code's persistent memory system changed this. Architectural decisions, debugging outcomes, and design constraints are stored and recalled across sessions automatically. Some examples of what was stored:

- *"The scanner is a pre-filter only — `slippageBps: 0` is intentional, not a bug. Do not suggest adding slippage."*
- *"Test baseline run on March 14 showed 142ms median latency on RPC calls — use this as the benchmark when evaluating optimisation suggestions."*
- *"Jupiter Codama code generation requires `pnpm codama generate` from the root — do not suggest regenerating individual files."*

These sound like small things. Across a four-month project, they were the difference between coherent, consistent codebase evolution and a gradual drift towards contradictory implementations.

---

## Using Multiple AIs as a Review Committee

Here is a practice I have not seen described elsewhere: using multiple frontier LLMs as independent reviewers *before* implementing architecture.

For each major design decision, I would write up the proposal and send it to four different models:

| Model | What I asked it to focus on |
|-------|----------------------------|
| **ChatGPT (GPT-4)** | General correctness, race conditions, non-blocking patterns |
| **Grok (xAI)** | Performance optimisations and throughput bottlenecks |
| **Qwen3-Max** | Market regime awareness and operational resilience |
| **DeepSeek** | Risk management gaps and safety mechanisms |

The results were not just "AI says it looks good." GPT-4 scored one architecture at 9.3/10 and identified two specific torn-read risks. Grok's suggestions improved simulated success rates from 60–70% to 70–85%. DeepSeek found seven critical risk management gaps that I had not considered. Qwen3 added probability uplift of three to five percentage points through market regime detection.

This is what an architecture review committee does in a large engineering organisation. I ran one with AI instead of colleagues. It cost me an afternoon and a modest API bill. The alternative would have been shipping with those seven gaps undiscovered.

---

## The AI That Runs the Trading System Itself

There is a deeper layer to this story. The development approach shaped the runtime architecture.

The trading system itself is built as an agentic pipeline:

```
SCANNERS  →  PLANNERS  →  EXECUTORS
(Observe)    (Decide)      (Act)
```

Scanners continuously monitor live DEX pools and emit typed opportunity events over NATS. Planners validate those opportunities, calculate risk scores, and publish execution plans. Executors autonomously build, sign, and submit transactions.

The three-layer structure — observe, decide, act — is the same model used in modern AI agent frameworks. It was not a coincidence. After months of working with AI agents as development tools, the same architecture that made the AI tools reliable (separation of concerns, typed interfaces, explicit state) turned out to be the right architecture for autonomous trading too.

There was also an earlier prototype: an agent that connected to TradingView chart data to interpret technical analysis signals — trend direction, momentum indicators — and feed them into the trading strategy. A system where one AI-flavoured component communicated with another. That felt like a preview of where software is going.

---

## GitHub Copilot: The Other Half of the Equation

Claude Code handles the large, architectural, multi-file work. GitHub Copilot handles everything else.

While Claude Code is thinking about whether a service should use push or pull semantics over NATS, Copilot is completing the retry loop I am currently typing. While Claude Code is generating a Rust RPC proxy with correct connection pooling, Copilot is suggesting the next line of the protocol buffer definition I started.

They do not compete. They cover different time horizons: Copilot for seconds, Claude Code for minutes and hours. Together they produce a velocity that is qualitatively different from what either provides alone.

---

## OpenClaw: AI for Operations, Not Just Development

A few months in, I realised I needed AI not just to build the system but to run it.

I built an internal tool called **OpenClaw** — a local AI gateway that sits alongside the trading system in production. It connects a local DeepSeek model (via Ollama, running on my own hardware, not sent to any cloud API), GitHub Copilot's API, and a Telegram bot.

The result: I can send a message to a Telegram channel and ask "what was the average arbitrage profit margin in the last hour?" and get a natural language answer sourced from live Prometheus metrics. I can ask "are there any anomalies in the RPC latency today?" and get a structured analysis.

This is a different kind of AI use — not development, not code generation, but operational intelligence. The trading system runs autonomously; OpenClaw is the interface through which I observe and understand what it is doing.

---

## What Actually Changes

I want to be direct about something: this is not a story about AI making development easy. It is a story about AI making ambitious development *possible* with smaller teams.

The hard parts were still hard. I spent days debugging why Jupiter's `/swap-instructions` returns the wrong destination account for circular arbitrage routes. No AI solved that — I had to read the Borsh encoding, trace the account indexing through the merged route plan, and work out the fix. What AI did was handle the ninety percent of the work that is not that hard part: the boilerplate, the scaffolding, the known patterns, the consistency enforcement, the documentation. That freed my attention for the problems that actually required a human.

I also found that the quality of AI output depends almost entirely on the quality of constraints you give it. A vague prompt produces vague code. A prompt with explicit forbidden patterns, performance targets, and system-specific context produces code you can ship. The skill that matters is not knowing how to prompt — it is knowing your domain well enough to specify the constraints correctly.

---

## What Comes Next

I wrote in the LinkedIn article that the transition from full-stack developer to AI engineer is not about learning AI — it is about learning how to direct AI. That is what this project taught me in practice.

The Solana trading system is now in active execution, processing live arbitrage opportunities across three confirmed DEX protocols, with 57,000+ events validated and on-chain execution confirmed. It was built by one person with AI as the team.

I do not think that model is unique to me or to this project. I think it is how a significant fraction of software will be built in the next three years.

---

## Related Posts

- [Scanner Service: AI Tools Reflection](/posts/2026/01/scanner-service-architecture-ai-tools-reflection-australia-day/) — post #24: First reflection on two months of AI-assisted development
- [OpenClaw Setup: AI Monitoring for Solana Trading Bot](/posts/2026/03/openclaw-setup-ai-monitoring-solana-trading-bot/) — post #25: Building the operational AI layer
- [OpenClaw: Cost-Effective AI Model Configuration](/posts/2026/03/openclaw-cost-effective-ai-model-configuration/) — post #28: Balancing model cost and capability
- [Planner Validation: Testing the Scanner→Planner→Executor Pipeline](/posts/2026/03/planner-validation-arb-pipeline-route-merge-simulation-execution/) — post #27: What the AI-assisted architecture produced

## Further Reading

- [From Full-Stack Developer to AI Engineer: Is Now the Right Time?](https://www.linkedin.com/pulse/from-full-stack-developer-ai-engineer-now-right-time-make-james-shen-pshcc/) — LinkedIn article
- [AI-Assisted Development Overview](https://github.com/guidebee/solana-trading-system/blob/main/docs/AI-ASSISTED-DEVELOPMENT-OVERVIEW.md) — Technical documentation

---

*This is post #28 in the Solana Trading System development series. Follow along on [GitHub](https://github.com/guidebee) or [LinkedIn](https://linkedin.com/in/guidebee).*
