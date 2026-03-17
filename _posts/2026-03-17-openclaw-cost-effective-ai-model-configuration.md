---
layout: single
title: "Running OpenClaw Cost-Effectively: Azure AI, GitHub Copilot, and Free Local Models"
date: 2026-03-17
permalink: /posts/2026/03/openclaw-cost-effective-ai-model-configuration/
categories:
  - blog
tags:
  - openclaw
  - ai
  - azure
  - github-copilot
  - devops
  - infrastructure
  - llm
  - cost-optimization
excerpt: "A practical guide to configuring OpenClaw with multiple AI providers — Azure AI Platform (GPT-4.1 and GPT-5), GitHub Copilot's free Claude Sonnet 4.6, MiniMax as a budget fallback, and zero-cost local models via Ollama — so you're never paying more than you need to."
---

## TL;DR

- **Visual Studio Professional** gives you $50/month in Azure credits — enough to run GPT-4.1 and GPT-5 as your primary OpenClaw models at zero marginal cost
- **GitHub Copilot** (separate subscription) lets you use **Claude Sonnet 4.6** for free within the subscription — excellent for chat-heavy workflows
- **MiniMax M2.5** (~$0.30/$1.20 per M tokens) is the best cheap fallback when Azure credits are exhausted
- **Ollama + DeepSeek/Qwen** on a Mac mini or local server gives you completely free, private inference — ideal for heartbeats and background tasks
- OpenClaw's `models.json` lets you mix all these providers in one config with zero code changes

---

## The Cost Problem with AI Agents

OpenClaw agents run continuously. Unlike a one-off ChatGPT query, an always-on monitoring agent fires LLM calls for:

- **Heartbeats** — checking service health every hour
- **Context compaction** — summarising long sessions automatically
- **Reactive tasks** — responding to alerts, analysing logs, answering Telegram messages

At OpenAI list prices, GPT-4.1 costs $2/$8 per million input/output tokens. A busy monitoring agent with hourly heartbeats and a few Telegram interactions a day can easily consume $30–50/month. That's before you add development tasks.

The good news: with the right subscription stacking, you can run all of this for **effectively zero additional cost**.

---

## Tier 1: Azure AI Platform — $50/Month Free with VS Professional

Microsoft Visual Studio Professional includes **$50/month of Azure credits**. With Azure AI Foundry (formerly Azure OpenAI Service), those credits cover real OpenAI models — including GPT-4.1 and GPT-5 — at standard pay-as-you-go rates.

### What $50 buys you

At Azure's GPT-4.1 pricing ($2 input / $8 output per million tokens):
- 10M input tokens + 2.5M output tokens = ~$25
- A monitoring agent running 24/7 with hourly heartbeats uses roughly 200K tokens/day → **~6M tokens/month** → well under $50

GPT-5 (`gpt-5-chat` in preview) currently has $0 metered cost during the preview window, making it effectively free even without credits.

### Setting up Azure AI deployments

In [Azure AI Foundry](https://ai.azure.com), create two deployments under your subscription:

| Deployment Name | Model | Use case |
|---|---|---|
| `gpt-4.1` | GPT-4.1 | Primary agent model, tool execution |
| `gpt-5-chat` | GPT-5 | Background tasks, compaction, heartbeat |

Both live under the same Azure AI resource. Note the endpoint URL pattern:

```
https://{resource-name}.cognitiveservices.azure.com/openai/deployments/{deployment-name}?api-version=2025-01-01-preview
```

### OpenClaw provider configuration

Each Azure deployment needs its own provider entry in `~/.openclaw/agents/main/agent/models.json` (or the `models.providers` section of `openclaw.json`). Azure uses a deployment-specific URL, so you need one provider per deployment — unlike OpenAI where one provider hosts all models.

```json
{
  "providers": {
    "azure-openai-gpt41": {
      "baseUrl": "https://YOUR-RESOURCE.cognitiveservices.azure.com/openai/deployments/gpt-4.1?api-version=2025-01-01-preview",
      "apiKey": "YOUR_AZURE_API_KEY",
      "api": "openai-completions",
      "headers": {
        "api-key": "YOUR_AZURE_API_KEY"
      },
      "models": [
        {
          "id": "gpt-4.1",
          "name": "GPT-4.1 (Azure)",
          "api": "openai-completions",
          "reasoning": false,
          "input": ["text", "image"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 1047576,
          "maxTokens": 32768
        }
      ]
    },

    "azure-openai": {
      "baseUrl": "https://YOUR-RESOURCE.cognitiveservices.azure.com/openai/deployments/gpt-5-chat?api-version=2025-01-01-preview",
      "apiKey": "YOUR_AZURE_API_KEY",
      "api": "openai-completions",
      "headers": {
        "api-key": "YOUR_AZURE_API_KEY"
      },
      "models": [
        {
          "id": "gpt-5-chat",
          "name": "GPT-5 (Azure)",
          "api": "openai-completions",
          "reasoning": false,
          "input": ["text", "image"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 128000,
          "maxTokens": 16384,
          "compat": {
            "supportsTools": true
          }
        }
      ]
    }
  }
}
```

**Key details:**

- **Two provider entries, one per deployment** — Azure bakes the deployment name into the URL
- **`headers: { "api-key": "..." }`** — Azure requires the key as a custom header, not just Bearer auth
- **`compat.supportsTools: true`** on GPT-5 — this flag tells OpenClaw's gateway to enable tool calls for models that don't advertise tool support via standard capability detection. GPT-4.1 doesn't need it (native support); GPT-5 chat preview does.
- **`cost: { input: 0, output: 0 }`** — set costs to zero since these are covered by subscription credits. This prevents OpenClaw's cost tracker from falsely reporting spend.

### Referencing Azure models in agent config

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "azure-openai-gpt41/gpt-4.1"
      },
      "compaction": {
        "mode": "safeguard",
        "model": "azure-openai-gpt41/gpt-4.1"
      },
      "heartbeat": {
        "every": "1h",
        "model": "azure-openai-gpt41/gpt-4.1"
      }
    }
  }
}
```

Using `azure-openai-gpt41/gpt-4.1` for both compaction and heartbeat means all background token usage stays within your Azure credit budget.

---

## Tier 2: GitHub Copilot — Free Claude Sonnet 4.6

If you have a **GitHub Copilot subscription**, OpenClaw can use it as an API backend, which means you get **Claude Sonnet 4.6** (a frontier model) at zero extra cost — entirely within your existing Copilot subscription.

### How it works

GitHub Copilot exposes an OpenAI-compatible API at `https://api.individual.githubcopilot.com`. OpenClaw authenticates with your Copilot token and routes requests through it. The models available include Claude Sonnet 4.5 and 4.6.

### Provider config

```json
{
  "providers": {
    "github-copilot": {
      "baseUrl": "https://api.individual.githubcopilot.com",
      "api": "openai-completions",
      "headers": {
        "Editor-Version": "vscode/1.85.0",
        "Copilot-Integration-Id": "vscode-chat"
      },
      "models": [
        {
          "id": "claude-sonnet-4.6",
          "name": "Claude Sonnet 4.6 (Copilot)",
          "reasoning": true,
          "input": ["text"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 200000,
          "maxTokens": 32000,
          "api": "openai-completions"
        },
        {
          "id": "claude-sonnet-4.5",
          "name": "Claude Sonnet 4.5 (Copilot)",
          "reasoning": true,
          "input": ["text"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 200000,
          "maxTokens": 32000,
          "api": "openai-completions"
        }
      ],
      "apiKey": "YOUR_COPILOT_TOKEN"
    }
  }
}
```

To get your Copilot token, run:
```bash
openclaw auth login --provider github-copilot
```

Or check `~/.openclaw/credentials/github-copilot.token.json` if you've already authenticated.

### When to use Copilot vs Azure

| Scenario | Best model |
|---|---|
| Telegram conversations, ad-hoc analysis | `github-copilot/claude-sonnet-4.6` |
| Coding tasks, complex reasoning | `github-copilot/claude-sonnet-4.6` |
| Background heartbeat / compaction | `azure-openai-gpt41/gpt-4.1` |
| Long-running agentic tasks | `azure-openai-gpt41/gpt-4.1` |

Claude Sonnet 4.6 is excellent for interactive chat and coding. Azure GPT-4.1 has a much larger context window (1M tokens) and is better suited to compaction and long agent sessions.

For the Telegram channel specifically, you can pin Claude Sonnet 4.6 per channel:

```json
{
  "channels": {
    "modelByChannel": {
      "telegram": {
        "YOUR_TELEGRAM_USER_ID": "github-copilot/claude-sonnet-4.6"
      }
    }
  }
}
```

---

## Tier 3: MiniMax — The Budget Fallback

When you're outside Azure credit or want a paid-but-cheap option for high-volume tasks, **MiniMax M2.5** is the best value in the current market:

- **Input**: $0.30/M tokens
- **Output**: $1.20/M tokens
- Supports reasoning, large context (131K tokens)
- OpenAI-compatible API

```json
{
  "providers": {
    "minimax": {
      "baseUrl": "https://api.minimaxi.chat/v1",
      "apiKey": "YOUR_MINIMAX_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "MiniMax-M2.5",
          "name": "MiniMax M2.5",
          "reasoning": true,
          "input": ["text"],
          "cost": { "input": 0.3, "output": 1.2, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 131072,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

**Tip**: Don't assign MiniMax to background processes like heartbeat or compaction — those run silently and will drain your balance without you noticing. Keep MiniMax for interactive tasks only, where you can see the token usage. Use the `mini` alias to call it explicitly from Telegram.

---

## Tier 4: Local Models via Ollama — Completely Free

If you have a **Mac mini (M-series)** or any reasonably specced local server (16GB+ RAM), you can run open-source models locally via [Ollama](https://ollama.ai) for zero API cost. This is ideal for:

- Hourly heartbeats (tiny prompts, no latency requirements)
- Draft summaries and low-stakes analysis
- Private data that shouldn't leave your network

### Install Ollama and pull models

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull models (choose based on your hardware)
ollama pull deepseek-r1:7b      # 8GB VRAM / 16GB RAM — good reasoning
ollama pull deepseek-r1:1.5b    # 4GB RAM — ultra-light, great for heartbeats
ollama pull qwen2.5-coder:7b    # Best for coding tasks
ollama pull qwen2.5:14b         # 16GB+ RAM — strong general model
```

### Hardware recommendations

| Hardware | Recommended model | Use case |
|---|---|---|
| Mac mini M4 (16GB) | `deepseek-r1:7b` or `qwen2.5:7b` | General agent tasks |
| Mac mini M4 Pro (24GB+) | `qwen2.5:14b` or `deepseek-r1:14b` | More capable reasoning |
| Mac Studio / Server (64GB+) | `qwen2.5:72b` or `deepseek-r1:70b` | Near-frontier quality |
| Any machine (8GB) | `deepseek-r1:1.5b` | Heartbeats, status checks only |

### OpenClaw provider config for Ollama

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://127.0.0.1:11434",
      "api": "ollama",
      "apiKey": "ollama",
      "models": [
        {
          "id": "deepseek-r1:7b",
          "name": "DeepSeek R1 7B (Local)",
          "reasoning": true,
          "input": ["text"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 131072,
          "maxTokens": 8192
        },
        {
          "id": "qwen2.5-coder:7b",
          "name": "Qwen 2.5 Coder 7B (Local)",
          "reasoning": false,
          "input": ["text"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 131072,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

If Ollama runs on a different machine on your local network, replace `127.0.0.1` with the server's local IP (e.g. `http://192.168.1.100:11434`).

---

## Putting It All Together: The Full Cost-Optimised Config

Here's how the tiers translate into a complete agent defaults configuration:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "azure-openai-gpt41/gpt-4.1",
        "fallbacks": []
      },
      "models": {
        "github-copilot/claude-sonnet-4.6": { "alias": "sonnet" },
        "azure-openai-gpt41/gpt-4.1":       { "alias": "azure-gpt41" },
        "azure-openai/gpt-5-chat":           { "alias": "azure-gpt5" },
        "minimax/MiniMax-M2.5":              { "alias": "mini" },
        "ollama/deepseek-r1:7b":             { "alias": "local" }
      },
      "compaction": {
        "mode": "safeguard",
        "model": "azure-openai-gpt41/gpt-4.1"
      },
      "heartbeat": {
        "every": "1h",
        "model": "azure-openai-gpt41/gpt-4.1",
        "target": "last",
        "directPolicy": "allow"
      }
    }
  }
}
```

And a per-channel Telegram override so interactive conversations use Claude:

```json
{
  "channels": {
    "modelByChannel": {
      "telegram": {
        "YOUR_USER_ID": "github-copilot/claude-sonnet-4.6"
      }
    }
  }
}
```

---

## Cost Summary

```
┌──────────────────────────────────────────────────────────────────┐
│  Monthly Cost Breakdown (VS Professional subscriber)             │
├──────────────────────────┬──────────────────┬────────────────────┤
│  Provider                │ Monthly Cost     │ Best For           │
├──────────────────────────┼──────────────────┼────────────────────┤
│  Azure GPT-4.1           │ $0 (VS credits)  │ Primary agent,     │
│                          │                  │ compaction, HB     │
├──────────────────────────┼──────────────────┼────────────────────┤
│  Azure GPT-5 (preview)   │ $0 (preview)     │ Heavy reasoning    │
├──────────────────────────┼──────────────────┼────────────────────┤
│  Claude Sonnet 4.6       │ $0 (within       │ Telegram chat,     │
│  via GitHub Copilot      │ Copilot sub)     │ coding tasks       │
├──────────────────────────┼──────────────────┼────────────────────┤
│  MiniMax M2.5            │ ~$2–5            │ High-volume        │
│                          │ (if used)        │ interactive tasks  │
├──────────────────────────┼──────────────────┼────────────────────┤
│  Ollama (local)          │ $0               │ Low-priority bg    │
│  DeepSeek / Qwen         │                  │ tasks, heartbeats  │
├──────────────────────────┼──────────────────┼────────────────────┤
│  TOTAL                   │ ~$0–5/month      │                    │
└──────────────────────────┴──────────────────┴────────────────────┘
```

---

## A Note on the `compat.supportsTools` Flag

One gotcha I ran into with the Azure GPT-5 deployment: tool execution (`exec`, `read`, `write` tools in OpenClaw) was silently failing. The fix was adding `"compat": { "supportsTools": true }` to the model config.

This flag is an OpenClaw-specific compatibility override. When a model doesn't advertise standard tool/function calling support through capability negotiation, OpenClaw defaults to disabling tool routing for it. The `supportsTools: true` flag bypasses that check and forces tool calls through. GPT-4.1 doesn't need it (it advertises tool support natively); GPT-5 in the current Azure preview does.

If your Azure model is responding to chat messages fine but tools silently don't run, this is almost always the cause.

---

## Related Posts

- [OpenClaw: AI-Powered Monitoring for My Solana HFT Trading Bot](/posts/2026/03/openclaw-setup-ai-monitoring-solana-trading-bot/) — initial setup, security isolation, Telegram integration
- [OpenClaw Beyond Trading Bots: AI-Assisted China Stock Data](/posts/2026/03/openclaw-china-stock-data-analysis-ai-assisted-development/) — using OpenClaw for general-purpose AI workflows

---

*This is part of the OpenClaw + Solana Trading System development series. Follow along on [GitHub](https://github.com/guidebee) or connect on [LinkedIn](https://linkedin.com/in/guidebee).*
