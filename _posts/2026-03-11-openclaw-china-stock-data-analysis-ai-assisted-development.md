---
layout: single
title: "OpenClaw Beyond Trading Bots: AI-Assisted China Stock Data Retrieval and Analysis"
date: 2026-03-11
permalink: /posts/2026/03/openclaw-china-stock-data-analysis-ai-assisted-development/
categories:
  - blog
tags:
  - openclaw
  - ai
  - trading
  - china
  - stock-market
  - telegram
  - data
  - typescript
  - claude-code
  - devops
excerpt: "OpenClaw isn't just for monitoring trading bots — it's a general-purpose AI agent gateway. I used it with Claude Code to rapidly build a China A-share stock data pipeline and TradingView-compatible charting service, then wired OpenClaw to analyse stocks and send chart summaries straight to Telegram."
---

## TL;DR

- Built **china_stock_tradingview** — a TradingView-compatible data feed server for all China A/B/STAR/BSE listed stocks, backed by SQLite with multi-resolution OHLCV caching
- Published **[china-stock-data](https://github.com/guidebee/china-stock-data)** — a free, auto-updated public repo of daily OHLCV data for 5,568 China A-share companies across SSE, SZSE and BSE
- Set up **cron jobs** to keep the data current after each trading day — automatically committed to GitHub
- Did the whole thing **in a couple of days with Claude Code + OpenClaw** — versus a similar ASX project ([asx_data_daily](https://github.com/asxgym/asx_data_daily)) that took me weeks to build solo a few years ago
- Wired **OpenClaw** to pull charts and deliver AI-powered technical analysis directly to Telegram

---

## OpenClaw is a General-Purpose AI Agent Gateway

In [the previous post](/posts/2026/03/openclaw-setup-ai-monitoring-solana-trading-bot/) I set up OpenClaw to monitor and control Solana HFT Docker services via Telegram. That's a narrow use case — but OpenClaw is not a narrow tool. It's a general-purpose AI agent gateway: any task you can describe in a system prompt, and any data you can pipe through a shell command or HTTP call, is fair game.

The China stock analysis project is a good illustration of this. The same `openclaw-bot` daemon that watches Docker containers can equally well receive a stock symbol, fetch a chart, and return a structured technical analysis — all triggered from a Telegram message.

---

## The ASX Comparison: Weeks vs Hours

A few years ago I built [asx_data_daily](https://github.com/asxgym/asx_data_daily) — a similar daily OHLCV data pipeline for the Australian Securities Exchange. Same idea: scrape end-of-day prices, commit structured CSV files to GitHub, keep it updated automatically. I did it entirely on my own, no AI assistance. It took me weeks of evenings: figuring out the data formats, handling edge cases, wiring up the cron automation, debugging the git commit flow.

This time, I did the same thing for China A-shares. Same structure, harder problem (5,568 companies across three exchanges, Chinese character handling, multiple market segments). With **Claude Code and OpenClaw**, the whole pipeline — data retrieval code, repository structure, README, cron jobs — was up and running in a couple of days. Some of the individual pieces took only hours.

The difference isn't just speed. It's the texture of the work. Instead of grinding through documentation and trial-and-error, I described what I wanted and iterated on real output. Claude Code handled the boilerplate; I focused on the design decisions.

---

## china_stock_tradingview: A TradingView-Compatible Data Feed

The [china_stock_tradingview](https://github.com/guidebee/china_stock_tradingview) project is a TypeScript/Node.js service that exposes a REST API compatible with the TradingView Charting Library's UDF datafeed protocol.

### Features

- **Multi-resolution OHLCV caching** — stores 5-min, 15-min, 30-min, 60-min and 240-min bars in SQLite; serves cached data immediately while refreshing in the background
- **Symbol search** — `/search/:keyword` returns matching stocks by symbol or code across all exchanges
- **Symbol listing** — `/symbols` returns the full universe of listed companies with name, exchange and type
- **Time-range queries** — `/data/:symbol?resolution=&from=&to=&limit=` returns bars for any window
- **Background delta fetch** — on every request, a background job checks whether newer data is available and fetches only the delta, keeping latency low for the next caller
- **SQLite persistence** — company metadata and price data stored locally; no external database dependency

### API surface

```
GET /data/:symbol?resolution=5&from=<unix>&to=<unix>&limit=500
  → array of OHLCV bars

GET /search/:keyword
  → [{ symbol, full_name, description, exchange, type }, ...]

GET /symbols
  → full company universe in TradingView UDF format
```

Resolutions supported: `5`, `15`, `30`, `60`, `240` (minutes).

### Data model

Each bar stored in SQLite:

```typescript
interface StockMarketData {
  timeStamp: number;   // Unix ms
  open:      number;
  close:     number;
  high:      number;
  low:       number;
  volume:    number;
}
```

---

## china-stock-data: Free Daily OHLCV for 5,568 Companies

The companion public dataset, [china-stock-data](https://github.com/guidebee/china-stock-data), is updated automatically after each trading day and follows the same flat-file convention as [asx_data_daily](https://github.com/asxgym/asx_data_daily).

### Repository layout

```
data/
├── company/
│   └── companies.json          # All listed companies with metadata
└── price/
    └── YYYY/
        └── MM/
            └── stock_price_YYYY_MM_DD.csv
```

### Company metadata (`companies.json`)

```json
[
  {
    "symbol":        "sh600000",
    "code":          "600000",
    "name":          "浦发银行",
    "stock_type":    "sh_a",
    "trade":         10.07,
    "mktcap":        33538979.1681,
    "nmc":           33538979.1681,
    "turnoverratio": 0.14922
  }
]
```

| Field | Description |
|-------|-------------|
| `symbol` | Full symbol with exchange prefix (`sh600000`, `sz000001`, `bj920000`) |
| `code` | Bare stock code |
| `name` | Company name (Chinese) |
| `stock_type` | Market segment (see below) |
| `mktcap` | Total market cap (CNY × 1000) |
| `nmc` | Circulating market cap (CNY × 1000) |
| `turnoverratio` | Daily turnover ratio (%) |

Market segments covered:

| `stock_type` | Exchange | Count |
|---|---|---|
| `sh_a` | Shanghai A-shares (上交所) | 1,703 |
| `sz_a` | Shenzhen A-shares (深交所) | 2,884 |
| `sh_b` | Shanghai B-shares | 41 |
| `sz_b` | Shenzhen B-shares | 38 |
| `kcb` | STAR Market / Sci-Tech Board (科创板) | 604 |
| `hs_bjs` | Beijing Stock Exchange (北交所) | 298 |

**Total: 5,568 companies**

### Daily price files (CSV, no header)

```
symbol, date, open, close, high, low, volume, amount
```

Example (`stock_price_2026_03_11.csv`):

```
sh600000,2026-03-11,9.97,10.07,10.09,9.85,62853742,625372487.23
sz000001,2026-03-11,10.82,10.85,10.91,10.76,104882300,1137291048.5
sz000002,2026-03-11,4.64,4.66,4.69,4.61,87345200,406852341.8
sh688001,2026-03-11,37.1,37.28,37.85,36.9,5923100,221847320.4
bj920000,2026-03-11,17.9,18.07,18.2,17.87,413986,7453468
```

Loading in Python is straightforward:

```python
import pandas as pd

cols = ['symbol', 'date', 'open', 'close', 'high', 'low', 'volume', 'amount']
df = pd.read_csv('data/price/2026/03/stock_price_2026_03_11.csv',
                 header=None, names=cols, parse_dates=['date'])
```

---

## Keeping the Data Fresh: Cron Jobs

After the initial backfill, keeping the dataset current is a two-step cron:

1. **Fetch** — runs after market close (15:30 CST) on each trading day, calls the data retrieval script, writes the new CSV under `data/price/YYYY/MM/`
2. **Commit and push** — stages the new file, creates a timestamped commit, pushes to GitHub

```bash
# Example crontab (server time = UTC+8)
# Fetch daily prices at 15:45 on weekdays
45 15 * * 1-5 /home/james/bin/fetch-china-prices.sh >> /var/log/china-prices.log 2>&1

# Commit and push at 16:00
0 16 * * 1-5 cd /opt/china-stock-data && git add data/price && \
  git commit -m "Automated price update $(date +\%Y-\%m-\%d)" && git push
```

Claude Code helped wire these up and handle the edge cases — holiday detection, partial trading days, retry logic on network errors. What would have been an afternoon of bash debugging took about thirty minutes.

---

## OpenClaw: Stock Analysis in Telegram

With the data feed running and the daily pipeline in place, the natural next step was connecting it to OpenClaw. The same agent that monitors Docker containers can equally well receive a stock symbol and return a technical analysis.

The setup is minimal. OpenClaw's `exec` tool calls a script that fetches recent price data, renders a chart snapshot, and returns it to the agent. The agent then produces a structured summary and sends it back to Telegram.

Here's what that looks like in practice:

![OpenClaw delivering China stock technical analysis via Telegram](../images/china_stock_analysis.png)

The agent's output follows a consistent structure:

```
Key Patterns Observed:
1. Major Drop       — sharp selloff, volume spike
2. Recovery         — two-day bounce toward prior levels
3. Volume Spike     — elevated volume at the low, accumulation signal
4. MA Crossover     — MA5 crossed below MA20 during the drop; now stabilising

Technical Outlook:
  Signal     │ Bearish recovery
  Trend      │ ▲ Tentative (watch X35 level)
  Resistance │ X37–38 range (previous pivot)
  Support    │ X34–35 range
  Volatility │ High (2.5–3× avg)

Short-term: Bouncing but still below key MA. Watch X35 as
critical support — if broken, likely retests prior lows.
```

The same Telegram interface used to restart a Solana Docker service now delivers stock chart analysis — same daemon, same `SOUL.md` identity, completely different task domain. That's the point: OpenClaw is a general-purpose gateway, not a single-purpose monitor.

---

## What Claude Code Enabled

On the pure development side, Claude Code accelerated every phase:

- **Data feed server** — generated the Express/SQLite scaffold, the UDF endpoint contracts, and the background refresh logic from a brief description
- **Repository structure** — suggested the `data/company/` + `data/price/YYYY/MM/` layout (matching the established ASX convention) and wrote the README from the schema
- **Cron pipeline** — handled the fetch/commit/push automation, date arithmetic, and retry handling
- **OpenClaw agent config** — extended the existing `openclaw.json` and `SOUL.md` to add the stock analysis workflow without touching the Solana monitoring setup

The ASX project ([asx_data_daily](https://github.com/asxgym/asx_data_daily)) was built entirely solo a few years ago — a legitimate engineering exercise that taught me a lot. But the China stock version, which is objectively more complex (3× the companies, multiple exchanges, Chinese character handling, a full intraday charting service on top), was done faster and with less friction. That gap is entirely attributable to AI tooling.

---

## What's Next

A natural evolution is adding sector and index-level analysis — grouping stocks by `stock_type` or industry and letting the agent spot market-wide patterns rather than single-stock moves. The company metadata already includes market cap, so weighting by `mktcap` for index approximations is straightforward.

The OpenClaw side could also be extended to pro-actively scan the daily data after market close — similar to how `HEARTBEAT.md` monitors Docker services — and push a Telegram summary of notable movers without any manual query.

---

## Related Posts

- [OpenClaw: AI-Powered Monitoring for My Solana HFT Trading Bot](/posts/2026/03/openclaw-setup-ai-monitoring-solana-trading-bot/)
- [Puppeteer Service: A Stealth Browser Microservice](/posts/2026/03/puppeteer-service-cloudflare-bypass-stealth-browser-microservice/)
- [Scanner Service Architecture & AI Tools Reflection](/posts/2026/01/scanner-service-architecture-ai-tools-reflection-australia-day/)

## Technical Documentation

- [china-stock-data on GitHub](https://github.com/guidebee/china-stock-data)
- [asx_data_daily on GitHub](https://github.com/asxgym/asx_data_daily)
- [OpenClaw Documentation](https://openclaw.ai/docs)

---

**Connect**: [GitHub](https://github.com/guidebee) · [LinkedIn](https://linkedin.com/in/jamesshenjava)

*This is post #28 in the Solana Trading System development series. This post demonstrates OpenClaw as a general-purpose AI agent gateway — the same daemon used for Solana HFT monitoring repurposed to deliver China stock technical analysis via Telegram, built with Claude Code in a fraction of the time a comparable project took without AI assistance.*
