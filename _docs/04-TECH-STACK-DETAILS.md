---
layout: single
title: "Technology Stack Details"
permalink: /docs/04-tech-stack-details/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Technology Stack Details

## Overview

This document provides detailed justification and implementation guidance for each technology choice in the production system.

## Programming Languages

### TypeScript

**Usage:** Scanners, Planners, Executors, Business Logic

**Justification:**
- Rich Solana ecosystem (`@solana/web3.js`, `@coral-xyz/anchor`)
- Fast iteration for business logic
- Strong typing for correctness
- Large talent pool
- Excellent tooling (VS Code, ESLint, Prettier)

**Best Practices:**
- Use strict mode (`"strict": true` in tsconfig.json)
- Leverage discriminated unions for type safety
- Use Zod for runtime validation
- Prefer async/await over callbacks
- Use pnpm for package management
- Enable source maps for debugging

**Key Libraries:**
```json
{
  "@solana/web3.js": "^2.0.0",
  "@coral-xyz/anchor": "^0.30.0",
  "@jup-ag/api": "^6.0.0",
  "@kamino-finance/klend-sdk": "^0.5.0",
  "nats": "^2.20.0",
  "ioredis": "^5.3.0",
  "pg": "^8.11.0",
  "pino": "^8.19.0",
  "zod": "^3.22.0"
}
```

**Monorepo Structure:**
- Nx or Turborepo for workspace management
- Shared libraries (`@trading-system/types`, `@trading-system/rpc`)
- Consistent tsconfig across services

---

### Go

**Usage:** Quote Service, RPC Client, High-Performance Services

**Justification:**
- Excellent concurrency (goroutines)
- Fast compilation and startup
- Low memory footprint
- Static binaries for easy deployment
- 2-10ms quote latency (proven in prototype)

**Best Practices:**
- Use contexts for cancellation and timeouts
- Leverage `cosmossdk.io/math.Int` for precision
- Implement graceful shutdown
- Use structured logging (zerolog, zap)
- Profile with pprof for optimization
- Use Go modules for dependency management

**Key Libraries:**
```go
require (
    github.com/gagliardetto/solana-go v1.12.0
    github.com/gin-gonic/gin v1.10.0
    github.com/redis/go-redis/v9 v9.5.0
    cosmossdk.io/math v1.5.3
    github.com/prometheus/client_golang v1.19.0
    go.uber.org/zap v1.27.0
)
```

**Service Structure:**
```go
cmd/
  quote-service/
    main.go              # Entry point
    server.go            # HTTP server
    handlers.go          # Request handlers

pkg/
  api/
    types.go             # API types
    validation.go        # Request validation
  quote/
    engine.go            # Quote engine
    cache.go             # Redis caching
  dex/
    raydium/
    meteora/
    pump/
```

---

### Rust

**Usage:** RPC Proxy, Transaction Builder, Performance-Critical Paths

**Justification:**
- Maximum performance (zero-cost abstractions)
- Memory safety without garbage collection
- Excellent async runtime (Tokio)
- Solana SDK in Rust (native)
- Zero-copy serialization (Borsh)

**Best Practices:**
- Use async/await with Tokio
- Leverage `Result<T, E>` for error handling
- Use `Arc` and `Mutex` for shared state
- Profile with `cargo flamegraph`
- Use `tracing` for instrumentation
- Pin dependency versions in production

**Key Libraries:**
```toml
[dependencies]
tokio = { version = "1.36", features = ["full"] }
solana-client = "1.18"
solana-sdk = "1.18"
axum = "0.7"
tower = "0.4"
serde = { version = "1.0", features = ["derive"] }
borsh = "1.3"
tracing = "0.1"
tracing-subscriber = "0.3"
redis = { version = "0.25", features = ["tokio-comp"] }
```

**Service Structure:**
```
src/
  main.rs              # Entry point
  server.rs            # HTTP/WS server
  proxy/
    mod.rs
    connection.rs      # Connection pooling
    load_balancer.rs   # Round-robin LB
    health.rs          # Health checks
  metrics/
    mod.rs
    prometheus.rs      # Metrics exporter
  config.rs            # Configuration
  error.rs             # Error types
```

---

## Infrastructure

### NATS JetStream

**Usage:** Event Bus, Pub/Sub, Message Persistence

**Justification:**
- High throughput (millions of messages/sec)
- Built-in persistence (JetStream)
- Excellent Go and TypeScript clients
- Replay capability (essential for debugging)
- Dead letter queues support
- Lightweight and fast

**Configuration:**
```typescript
// streams.ts
export const STREAMS = {
  MARKET_EVENTS: {
    name: "market.events",
    subjects: ["market.events.>"],
    retention: "limits",
    maxAge: 7 * 24 * 60 * 60 * 1_000_000_000, // 7 days (nanoseconds)
    maxBytes: 10 * 1024 * 1024 * 1024, // 10 GB
    storage: "file",
    replicas: 3,
  },
  TRADE_OPPORTUNITIES: {
    name: "trade.opportunities",
    subjects: ["trade.opportunities.>"],
    retention: "limits",
    maxAge: 24 * 60 * 60 * 1_000_000_000, // 1 day
    maxBytes: 5 * 1024 * 1024 * 1024, // 5 GB
    storage: "file",
    replicas: 3,
  },
  PLANNED_ORDERS: {
    name: "execution.orders",
    subjects: ["execution.orders.>"],
    retention: "limits",
    maxAge: 30 * 24 * 60 * 60 * 1_000_000_000, // 30 days
    maxBytes: 20 * 1024 * 1024 * 1024, // 20 GB
    storage: "file",
    replicas: 3,
  },
};
```

**TypeScript Client Usage:**
```typescript
import { connect, JSONCodec } from "nats";

const nc = await connect({ servers: process.env.NATS_URL });
const js = nc.jetstream();
const codec = JSONCodec();

// Publish
await js.publish("market.events.raydium.pool_update", codec.encode(event));

// Subscribe
const consumer = await js.consumers.get("market.events", "scanner-1");
const messages = await consumer.consume();

for await (const msg of messages) {
  const data = codec.decode(msg.data);
  await processEvent(data);
  msg.ack();
}
```

---

### Redis

**Usage:** Hot Data Cache, Pub/Sub, Distributed Locks

**Justification:**
- Sub-millisecond latency
- Rich data structures (hash, set, sorted set)
- Pub/sub for real-time updates
- Distributed locking (RedLock)
- Excellent client libraries
- Cluster mode for horizontal scaling

**Configuration:**
```yaml
# Redis configuration
bind 0.0.0.0
port 6379
protected-mode yes
requirepass your_redis_password
maxmemory 4gb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
save 60 10000
```

**TypeScript Client:**
```typescript
import Redis from "ioredis";

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: parseInt(process.env.REDIS_PORT),
  password: process.env.REDIS_PASSWORD,
  db: 0,
  retryStrategy: (times) => Math.min(times * 50, 2000),
});

// Set with TTL
await redis.setex(`price:${token}`, 300, JSON.stringify(priceData));

// Hash operations
await redis.hset("wallet:balances", address, balance.toString());

// Distributed lock
const lock = await redis.set(
  `lock:executor:${walletId}`,
  txId,
  "EX", 30,
  "NX"
);
```

---

### PostgreSQL

**Usage:** Persistent Data, Relational Data, Transactions

**Justification:**
- ACID transactions (critical for financial data)
- Rich query capabilities (joins, aggregations)
- Mature and battle-tested
- Excellent indexing
- JSON support for semi-structured data
- Read replicas for scalability

**Schema Design:**
```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Wallets
CREATE TABLE wallets (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  tier VARCHAR(20) NOT NULL,
  address VARCHAR(44) NOT NULL UNIQUE,
  encrypted_private_key TEXT NOT NULL,
  expected_balance JSONB NOT NULL DEFAULT '{}',
  actual_balance JSONB NOT NULL DEFAULT '{}',
  last_sync TIMESTAMP NOT NULL DEFAULT NOW(),
  enabled BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_wallets_tier ON wallets(tier);
CREATE INDEX idx_wallets_enabled ON wallets(enabled);

-- Trades
CREATE TABLE trades (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  strategy VARCHAR(50) NOT NULL,
  wallet_id UUID NOT NULL REFERENCES wallets(id),
  signature VARCHAR(88) NOT NULL UNIQUE,
  expected_profit BIGINT NOT NULL,
  actual_profit BIGINT,
  status VARCHAR(20) NOT NULL,
  error_message TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  confirmed_at TIMESTAMP,
  slot BIGINT
);

CREATE INDEX idx_trades_strategy ON trades(strategy);
CREATE INDEX idx_trades_status ON trades(status);
CREATE INDEX idx_trades_created_at ON trades(created_at DESC);

-- Trade routes
CREATE TABLE trade_routes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trade_id UUID NOT NULL REFERENCES trades(id) ON DELETE CASCADE,
  step INTEGER NOT NULL,
  protocol VARCHAR(50) NOT NULL,
  pool VARCHAR(44) NOT NULL,
  input_mint VARCHAR(44) NOT NULL,
  output_mint VARCHAR(44) NOT NULL,
  input_amount BIGINT NOT NULL,
  output_amount BIGINT NOT NULL
);

CREATE INDEX idx_trade_routes_trade_id ON trade_routes(trade_id);
```

**TypeScript Client:**
```typescript
import { Pool } from "pg";

const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Transaction example
const client = await pool.connect();
try {
  await client.query("BEGIN");

  const tradeResult = await client.query(
    "INSERT INTO trades (strategy, wallet_id, signature, expected_profit, status) VALUES ($1, $2, $3, $4, $5) RETURNING id",
    [strategy, walletId, signature, expectedProfit, "pending"]
  );

  for (const route of routes) {
    await client.query(
      "INSERT INTO trade_routes (trade_id, step, protocol, pool, input_mint, output_mint, input_amount, output_amount) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)",
      [tradeResult.rows[0].id, route.step, route.protocol, route.pool, route.inputMint, route.outputMint, route.inputAmount, route.outputAmount]
    );
  }

  await client.query("COMMIT");
} catch (e) {
  await client.query("ROLLBACK");
  throw e;
} finally {
  client.release();
}
```

---

### TimescaleDB

**Usage:** Time-Series Metrics, Historical Analysis

**Justification:**
- PostgreSQL extension (familiar interface)
- Optimized for time-series data
- Automatic data partitioning
- Continuous aggregates (materialized views)
- Excellent compression

**Setup:**
```sql
-- Enable TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Create hypertable
CREATE TABLE scanner_metrics (
  time TIMESTAMPTZ NOT NULL,
  scanner_id VARCHAR(50) NOT NULL,
  events_processed INTEGER NOT NULL,
  latency_p50 REAL NOT NULL,
  latency_p99 REAL NOT NULL,
  error_count INTEGER NOT NULL
);

SELECT create_hypertable('scanner_metrics', 'time');

-- Create continuous aggregate (1-hour buckets)
CREATE MATERIALIZED VIEW scanner_metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS bucket,
  scanner_id,
  SUM(events_processed) as total_events,
  AVG(latency_p50) as avg_latency_p50,
  AVG(latency_p99) as avg_latency_p99,
  SUM(error_count) as total_errors
FROM scanner_metrics
GROUP BY bucket, scanner_id;
```

---

## Observability

### Prometheus

**Usage:** Metrics Collection, Alerting

**Configuration:**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'scanners'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['trading-system']
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: scanner
        action: keep
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: instance

  - job_name: 'planners'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['trading-system']
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: planner
        action: keep

  - job_name: 'executors'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['trading-system']
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: executor
        action: keep

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

**TypeScript Metrics:**
```typescript
import { register, Counter, Histogram, Gauge } from "prom-client";

// Counters
export const opportunitiesDetected = new Counter({
  name: "opportunities_detected_total",
  help: "Total opportunities detected",
  labelNames: ["strategy"],
});

export const tradesExecuted = new Counter({
  name: "trades_executed_total",
  help: "Total trades executed",
  labelNames: ["strategy", "status"],
});

// Histograms
export const quoteLatency = new Histogram({
  name: "quote_latency_seconds",
  help: "Quote latency in seconds",
  labelNames: ["source"],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
});

// Gauges
export const activeWallets = new Gauge({
  name: "active_wallets",
  help: "Number of active wallets",
  labelNames: ["tier"],
});

// Usage
opportunitiesDetected.inc({ strategy: "arbitrage" });
quoteLatency.observe({ source: "solroute" }, 0.008);
```

---

### Grafana

**Usage:** Dashboards, Visualization

**Dashboard Examples:**

**System Overview:**
```json
{
  "dashboard": {
    "title": "Trading System Overview",
    "panels": [
      {
        "title": "Opportunities Detected",
        "targets": [
          {
            "expr": "rate(opportunities_detected_total[5m])",
            "legendFormat": "{{strategy}}"
          }
        ]
      },
      {
        "title": "Trade Success Rate",
        "targets": [
          {
            "expr": "rate(trades_executed_total{status=\"success\"}[5m]) / rate(trades_executed_total[5m])",
            "legendFormat": "Success Rate"
          }
        ]
      },
      {
        "title": "Quote Latency (p99)",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(quote_latency_seconds_bucket[5m]))",
            "legendFormat": "{{source}}"
          }
        ]
      }
    ]
  }
}
```

---

### Jaeger

**Usage:** Distributed Tracing

**TypeScript Integration:**
```typescript
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { JaegerExporter } from "@opentelemetry/exporter-jaeger";
import { registerInstrumentations } from "@opentelemetry/instrumentation";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";

const provider = new NodeTracerProvider();

const exporter = new JaegerExporter({
  endpoint: process.env.JAEGER_ENDPOINT,
});

provider.addSpanProcessor(
  new BatchSpanProcessor(exporter)
);

provider.register();

registerInstrumentations({
  instrumentations: [new HttpInstrumentation()],
});

// Usage
import { trace } from "@opentelemetry/api";

const tracer = trace.getTracer("arbitrage-planner");

async function analyzeOpportunity() {
  const span = tracer.startSpan("analyze_opportunity");

  try {
    const quote1 = await getQuote(); // auto-traced
    const quote2 = await getQuote(); // auto-traced

    span.addEvent("quotes_received");

    const profit = calculateProfit(quote1, quote2);
    span.setAttribute("profit", profit);

    return profit;
  } finally {
    span.end();
  }
}
```

---

### Loki

**Usage:** Log Aggregation

**TypeScript Logging:**
```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  transport: {
    target: "pino-loki",
    options: {
      batching: true,
      interval: 5,
      host: process.env.LOKI_URL,
      labels: {
        app: "trading-system",
        service: "arbitrage-planner",
      },
    },
  },
});

logger.info({ opportunity: {...} }, "Opportunity detected");
logger.error({ err, txId }, "Transaction failed");
```

---

## Deployment

### Kubernetes

**Deployment Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: arbitrage-planner
  namespace: trading-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: planner
      strategy: arbitrage
  template:
    metadata:
      labels:
        app: planner
        strategy: arbitrage
    spec:
      containers:
        - name: arbitrage-planner
          image: trading-system/arbitrage-planner:latest
          env:
            - name: NATS_URL
              valueFrom:
                secretKeyRef:
                  name: nats-creds
                  key: url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-creds
                  key: url
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: arbitrage-planner
  namespace: trading-system
spec:
  selector:
    app: planner
    strategy: arbitrage
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

---

## Development Tools

### Nx / Turborepo

**Monorepo Management:**

**Turborepo Configuration:**
```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false
    }
  }
}
```

### Docker Compose (Local Development)

```yaml
version: "3.9"

services:
  nats:
    image: nats:2.10
    command: ["-js", "-m", "8222"]
    ports:
      - "4222:4222"
      - "8222:8222"

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass dev_password
    ports:
      - "6379:6379"

  postgres:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: trading_system
    ports:
      - "5432:5432"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3000:3000"
```

---

## Summary

This tech stack provides:
- **High performance:** Go/Rust for critical paths, TypeScript for flexibility
- **Reliability:** NATS persistence, PostgreSQL ACID transactions
- **Scalability:** Horizontal scaling of all components
- **Observability:** Full metrics, traces, and logs
- **Developer experience:** Modern tooling, strong typing, fast iteration
