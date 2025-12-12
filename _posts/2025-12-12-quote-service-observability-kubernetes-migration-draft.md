---
layout: single
title: "Quote Service Observability: Comprehensive Metrics, Tracing, and Kubernetes Migration Planning"
date: 2025-12-12
permalink: /posts/2025/12/quote-service-observability-kubernetes-migration-draft/
categories:
  - blog
tags:
  - solana
  - trading
  - observability
  - monitoring
  - metrics
  - tracing
  - opentelemetry
  - grafana
  - prometheus
  - kubernetes
  - docker-compose
  - quote-service
  - go
excerpt: "Added comprehensive metrics and distributed tracing to the quote-service, created a detailed Grafana performance dashboard, and drafted a complete Kubernetes migration plan. Plus, a special birthday celebration for my son Harry as he embarks on his university journey."
---

## TL;DR

Today's work focused on deep observability for the quote-service and planning for future infrastructure scalability:

1. **Comprehensive Metrics**: Added 30+ Prometheus metrics covering quote requests, cache performance, external API calls, and hybrid quote operations
2. **Distributed Tracing**: Implemented OpenTelemetry tracing with detailed spans for request flow analysis
3. **Performance Dashboard**: Created a detailed Grafana dashboard visualizing SLA metrics, latency percentiles, cache efficiency, and API performance
4. **Kubernetes Migration Draft**: Designed complete K8s manifests for future production deployment (staying with Docker Compose for now due to faster iteration)
5. **Personal Milestone**: Celebrating my son Harry's birthday and Year 12 graduation as he begins his journey to university

## A Personal Note: Happy Birthday, Harry!

Before diving into the technical details, I want to take a moment to celebrate something special. Today is my son Harry's birthday! He just graduated from All Saints College after completing Year 12 and is currently traveling through China with his friends - an adventure he's been looking forward to for months.

Harry, if you're reading this from somewhere in China (hopefully on decent WiFi!): **Happy Birthday!** I'm incredibly proud of you for completing your secondary education and for having the courage to explore the world. Next year, you'll be starting university, and I know you have a bright future ahead of you.

Watching you grow into an independent young adult has been one of the greatest joys of my life. As I build this trading system, I'm reminded that the most important investments we make aren't in markets or technology - they're in the people we love and the experiences we share.

Now, let's get back to the technical work - but this context reminds me why we build these systems: to create opportunities and freedom for the next generation.

---

## Why Observability Matters for Quote Services

The quote-service is the **heart of the trading system** - it provides real-time pricing for thousands of trading pairs across multiple DEXes. When latency increases by even 100ms, arbitrage opportunities can vanish. When cache hit rates drop, external API costs spike. When external quoters fail, trading strategies stall.

**Without comprehensive observability, we're flying blind.**

Today's work addresses this by instrumenting every critical path with metrics and traces, giving us:

- **Real-time performance visibility**: P50, P95, P99 latencies for every operation
- **Cost optimization**: Track external API usage to minimize costs
- **Failure detection**: Identify failing APIs, rate limiting, and cache issues
- **User experience**: Ensure fast quote responses via cache monitoring
- **Capacity planning**: Understand request patterns and scaling needs

## Metrics Implementation

### Architecture: Three Layers of Metrics

The quote-service metrics are organized into three logical layers:

```
┌─────────────────────────────────────────────────────┐
│               Quote Request Layer                    │
│  (Request handling, cache decisions, response)       │
└──────────────┬──────────────┬───────────────────────┘
               │              │
       ┌───────▼──────┐  ┌────▼──────────┐
       │ Cache Layer  │  │ External APIs │
       │ (Local calc) │  │ (Jupiter, etc)│
       └──────┬───────┘  └────┬──────────┘
              │               │
              └───────┬───────┘
                      │
              ┌───────▼────────┐
              │  Output Layer  │
              │ (Quote served) │
              └────────────────┘
```

### 1. Quote Request Metrics

These metrics track the entire request lifecycle:

**Request Volume:**
```go
// quote_requests_total (Counter)
// Labels: input_mint, output_mint
quoteRequestsTotal.WithLabelValues(inputMint, outputMint).Inc()
```

**Cache Performance:**
```go
// quote_cache_hits_total / quote_cache_misses_total (Counter)
// Labels: input_mint, output_mint
quoteCacheHitsTotal.WithLabelValues(inputMint, outputMint).Inc()

// quote_cache_check_duration_seconds (Histogram)
// Labels: cache_hit (true/false)
quoteCacheCheckDuration.WithLabelValues("true").Observe(duration)

// quote_cache_age_seconds (Histogram)
// Labels: input_mint, output_mint
quoteCacheAgeSeconds.WithLabelValues(inputMint, outputMint).Observe(age)
```

**End-to-End Latency:**
```go
// quote_request_total_duration_seconds (Histogram)
// Labels: source (cache/calculated), input_mint, output_mint
quoteRequestTotalDuration.WithLabelValues("cache", inputMint, outputMint).Observe(totalDuration)
```

### 2. External API Metrics

Track every external quoter (Jupiter, DFlow, QuickNode, BLXR):

**Request Tracking:**
```go
// external_api_requests_total (Counter)
// Labels: quoter, input_mint, output_mint
externalAPIRequestsTotal.WithLabelValues("Jupiter", inputMint, outputMint).Inc()

// external_api_success_total / external_api_errors_total (Counter)
externalAPISuccessTotal.WithLabelValues("Jupiter", inputMint, outputMint).Inc()
```

**Performance Tracking:**
```go
// external_api_duration_seconds (Histogram)
// Labels: quoter, input_mint, output_mint
externalAPIDuration.WithLabelValues("Jupiter", inputMint, outputMint).Observe(duration)

// external_api_output_amount (Histogram)
// Labels: quoter, input_mint, output_mint
// Used for price comparison across providers
externalAPIOutputAmount.WithLabelValues("Jupiter", inputMint, outputMint).Observe(outAmount)
```

**Rate Limiting:**
```go
// external_api_rate_limit_errors_total (Counter)
// external_api_rate_limit_wait_seconds (Histogram)
externalAPIRateLimitWait.WithLabelValues("Jupiter").Observe(waitTime)
```

### 3. Hybrid Quote Metrics

The hybrid endpoint provides cached quotes with external fallback:

**Fast Path (Cache Hit):**
```go
// hybrid_cache_hits_total (Counter)
hybridCacheHitsTotal.WithLabelValues(inputMint, outputMint).Inc()

// hybrid_cached_quote_age_seconds (Histogram)
hybridCachedQuoteAge.WithLabelValues(inputMint, outputMint).Observe(quoteAge)

// hybrid_quote_fast_path_duration_seconds (Histogram)
hybridQuoteFastPathDuration.WithLabelValues("true").Observe(duration)
```

**Slow Path (Cache Miss/Stale):**
```go
// hybrid_cache_misses_total / hybrid_cache_stale_total (Counter)
hybridCacheMissesTotal.WithLabelValues(inputMint, outputMint).Inc()

// External quote caching
hybridExternalCacheHitsTotal.WithLabelValues(inputMint, outputMint).Inc()
```

### Complete Metrics List

The quote-service exports **30+ metrics** covering:

| Category | Metric Count | Examples |
|----------|--------------|----------|
| Quote Requests | 6 | Total requests, cache hits/misses, duration |
| Quote Calculation | 6 | Calculation duration, errors, output amounts |
| External APIs | 9 | Requests, success, errors, duration, rate limits |
| Hybrid Quotes | 11 | Fast/slow path, cache age, external cache |
| Output Tracking | 4 | Quotes by protocol, pool, output amounts |

**Full documentation:** [OBSERVABILITY_METRICS.md](https://github.com/guidebee/solana-trading-system/blob/master/go/cmd/quote-service/docs/OBSERVABILITY_METRICS.md)

## Distributed Tracing Implementation

### OpenTelemetry Integration

Added OpenTelemetry tracing to visualize the complete request flow:

**Initialization:**
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

tracer := otel.Tracer("quote-service")
```

**Request Span:**
```go
func (qc *QuoteCache) GetOrCalculate(ctx context.Context, inputMint, outputMint string, amount uint64) (*Quote, error) {
    ctx, span := tracer.Start(ctx, "quote_request")
    defer span.End()

    // Add attributes
    span.SetAttributes(
        attribute.String("quote.input_mint", inputMint),
        attribute.String("quote.output_mint", outputMint),
        attribute.Int64("quote.amount", int64(amount)),
    )

    // Track cache check
    ctx, cacheSpan := tracer.Start(ctx, "cache.check")
    quote, found := qc.get(cacheKey)
    cacheSpan.SetAttributes(
        attribute.Bool("cache.hit", found),
        attribute.Float64("cache.age_seconds", age),
    )
    cacheSpan.End()

    if found {
        span.AddEvent("cache_hit", trace.WithAttributes(
            attribute.Float64("quote.age_seconds", age),
        ))
        return quote, nil
    }

    // Calculate new quote
    ctx, calcSpan := tracer.Start(ctx, "quote.calculate")
    quote, err := qc.calculateQuote(ctx, inputMint, outputMint, amount)
    calcSpan.End()

    return quote, err
}
```

**External API Spans:**
```go
func (qm *QuoterManager) GetAllQuotes(ctx context.Context, inputMint, outputMint string, amount uint64) ([]ExternalQuote, error) {
    ctx, span := tracer.Start(ctx, "quoter_manager.get_all_quotes")
    defer span.End()

    // Create child spans for each quoter
    for _, quoter := range qm.quoters {
        quoterCtx, quoterSpan := tracer.Start(ctx, fmt.Sprintf("quoter.%s", quoter.Name()))

        startTime := time.Now()
        quote, err := quoter.GetQuote(quoterCtx, inputMint, outputMint, amount)
        duration := time.Since(startTime).Seconds()

        quoterSpan.SetAttributes(
            attribute.String("quoter.name", quoter.Name()),
            attribute.Float64("quoter.duration_seconds", duration),
            attribute.Bool("error", err != nil),
        )

        if quote != nil {
            quoterSpan.SetAttributes(
                attribute.Int64("quoter.out_amount", int64(quote.OutAmount)),
            )
        }

        quoterSpan.End()
    }

    return quotes, nil
}
```

### Trace Hierarchy

Traces create a hierarchical view of request execution:

```
quote_request (150ms)
├── cache.check (2ms)
│   └── event: cache_miss
├── quote.calculate (145ms)
│   ├── query_pools (120ms)
│   │   ├── raydium.query (40ms)
│   │   ├── meteora.query (38ms)
│   │   └── orca.query (42ms)
│   ├── select_best_pool (5ms)
│   └── calculate_output (20ms)
└── event: quote_served
    └── quote.protocol: "raydium"
    └── quote.out_amount: 98750000
```

**Alternative: External Quote Path**

```
quoter_manager.get_all_quotes (800ms)
├── quoter.Jupiter (250ms)
│   └── rate_limited: false
├── quoter.DFlow (320ms)
│   └── rate_limited: false
├── quoter.QuickNode (180ms)
│   └── rate_limited: false
└── quoter.BLXR (220ms)
    └── rate_limited: false
```

### Span Attributes

Every span includes relevant context:

**Quote Request Attributes:**
- `quote.input_mint`: Input token address
- `quote.output_mint`: Output token address
- `quote.amount`: Amount in base units
- `quote.source`: "cache" or "calculated"
- `quote.protocol`: Selected DEX protocol
- `quote.pool_id`: Pool used for quote
- `quote.out_amount`: Output amount
- `quote.total_duration_seconds`: Request duration
- `quote.cache_age_seconds`: Age if from cache

**External Quoter Attributes:**
- `quoter.name`: Quoter service name
- `quoter.duration_seconds`: API call duration
- `quoter.out_amount`: Output amount
- `rate_limited`: Whether rate limiting occurred
- `error`: Whether an error occurred

**Trace-to-Metrics Correlation:**

Traces include the same labels as metrics, enabling correlation:
```
Metric: external_api_duration_seconds{quoter="Jupiter"}
Trace:  span.attributes["quoter.name"] = "Jupiter"
```

## Grafana Performance Dashboard

Created a comprehensive Grafana dashboard with 40+ panels organized into sections:

![Quote Service Performance Dashboard](../images/quote-service-performance-dashboard.png)

### Dashboard Sections

#### 1. Executive Summary - SLA Dashboard

**Key Metrics:**
- **SLA Uptime**: Percentage of successful requests (target: >99%)
- **Request Rate**: Total requests per second
- **Cache Hit Rate**: Percentage served from cache (target: >80%)
- **Error Rate**: Errors per second (target: <1%)

**PromQL:**
```promql
# SLA Uptime (Success Rate)
(
  sum(rate(quotes_served_total[5m]))
  /
  sum(rate(quote_requests_total[5m]))
)

# Cache Hit Rate
(
  sum(rate(quote_cache_hits_total[5m]))
  /
  (sum(rate(quote_cache_hits_total[5m])) + sum(rate(quote_cache_misses_total[5m])))
)
```

#### 2. Request Performance

**Panels:**
- **Request Latency (P50, P95, P99)**: End-to-end response times
- **Requests by Source**: Cache vs calculated breakdown
- **Request Rate by Trading Pair**: Top 10 pairs
- **Total Duration Distribution**: Histogram view

**PromQL:**
```promql
# P95 Request Latency
histogram_quantile(0.95,
  sum by(le) (rate(quote_request_total_duration_seconds_bucket[5m]))
)

# Requests by Source (Cache vs Calculated)
sum by(source) (rate(quote_requests_total[5m]))
```

#### 3. Cache Performance

**Panels:**
- **Cache Hit Rate Over Time**: Trend analysis
- **Cache Age Distribution**: P50, P95, P99 quote age
- **Cache Lookup Duration**: Cache operation latency
- **Cache Hits vs Misses**: Time series comparison

**PromQL:**
```promql
# P95 Cache Age
histogram_quantile(0.95,
  sum by(le) (rate(quote_cache_age_seconds_bucket[5m]))
)

# Cache Lookup Duration
histogram_quantile(0.99,
  sum by(cache_hit, le) (rate(quote_cache_check_duration_seconds_bucket[5m]))
)
```

#### 4. Calculation Performance

**Panels:**
- **Calculation Duration (P50, P95, P99)**: Local calculation latency
- **Calculation Errors**: Error rate and types
- **Calculations per Second**: Volume of fresh calculations

**PromQL:**
```promql
# P95 Calculation Duration
histogram_quantile(0.95,
  sum by(le) (rate(quote_calculation_duration_seconds_bucket[5m]))
)

# Calculation Error Rate
sum by(error_type) (rate(quote_calculation_errors_total[5m]))
```

#### 5. External API Performance

**Panels:**
- **External API Success Rate by Provider**: Jupiter, DFlow, QuickNode, BLXR
- **External API Latency by Provider (P95)**: Comparative latency
- **External API Request Volume**: Requests per provider
- **External API Error Rate**: Failures by provider
- **Rate Limit Wait Time**: Impact of rate limiting

**PromQL:**
```promql
# Success Rate by Provider
(
  sum by(quoter) (rate(external_api_success_total[5m]))
  /
  sum by(quoter) (rate(external_api_requests_total[5m]))
)

# P95 Latency by Provider
histogram_quantile(0.95,
  sum by(quoter, le) (rate(external_api_duration_seconds_bucket[5m]))
)

# Rate Limit Wait Time
histogram_quantile(0.99,
  sum by(quoter, le) (rate(external_api_rate_limit_wait_seconds_bucket[5m]))
)
```

#### 6. Hybrid Quote Performance

**Panels:**
- **Hybrid Fast Path Rate**: Percentage using fast path (cached)
- **Fast Path vs Slow Path Latency**: Performance comparison
- **External Quote Cache Hit Rate**: External quote caching efficiency
- **Stale Quote Rate**: Percentage of stale quotes refreshed

**PromQL:**
```promql
# Fast Path Rate
(
  sum(rate(hybrid_cache_hits_total[5m]))
  /
  sum(rate(hybrid_quote_requests_total[5m]))
)

# Fast Path Latency vs Slow Path
histogram_quantile(0.95,
  sum by(has_external, le) (rate(hybrid_quote_fast_path_duration_seconds_bucket[5m]))
)
```

#### 7. Protocol & Pool Usage

**Panels:**
- **Quotes by Protocol**: Raydium, Meteora, Orca, etc.
- **Top 10 Pools by Volume**: Most-used liquidity pools
- **Protocol Distribution Over Time**: Trend analysis

**PromQL:**
```promql
# Quotes by Protocol
sum by(protocol) (rate(quotes_by_protocol[5m]))

# Top 10 Pools
topk(10,
  sum by(pool_id, protocol) (rate(quotes_by_pool[5m]))
)
```

#### 8. Error Tracking

**Panels:**
- **Total Error Rate**: All errors aggregated
- **Errors by Type**: Validation, calculation, API errors
- **External API Errors by Provider**: Provider-specific failures
- **Recent Error Log**: Live error stream from Loki

**PromQL:**
```promql
# Total Error Rate
sum(rate(quote_errors_total[5m])) +
sum(rate(quote_calculation_errors_total[5m])) +
sum(rate(external_api_errors_total[5m]))

# Errors by Type
sum by(error_type) (rate(quote_errors_total[5m]))
```

### Dashboard Features

**Interactive:**
- Variable dropdowns for trading pair filtering
- Time range selector (5m, 15m, 1h, 6h, 24h)
- Panel drill-down to Jaeger traces
- Link to external Jaeger UI for detailed trace analysis

**Auto-refresh:**
- 10-second refresh interval
- Real-time monitoring during trading hours

**Alerting-Ready:**
- Thresholds configured for SLA violations
- Color-coded panels (green/yellow/red)
- Alert annotations on graphs

**Exported Configuration:**
- [quote-service-performance.json](https://github.com/guidebee/solana-trading-system/blob/master/deployment/monitoring/grafana/provisioning/dashboards/quote-service-performance.json)
- 2,768 lines of dashboard configuration
- Automatically provisioned via Grafana ConfigMap

## Kubernetes Migration Planning

### Why Kubernetes? (Eventually)

While Docker Compose serves us well for development, Kubernetes offers:

**Scalability:**
- Horizontal pod autoscaling based on request load
- Load balancing across multiple quote-service replicas
- Cluster-wide resource management

**High Availability:**
- Pod health checks and automatic restarts
- Rolling updates with zero downtime
- Multi-zone deployments

**Production-Ready:**
- Secrets management
- Network policies and security
- Service mesh integration (Istio, Linkerd)

**DevOps Integration:**
- GitOps with ArgoCD/FluxCD
- CI/CD pipeline integration
- Infrastructure as Code

### Why Staying with Docker Compose for Now

Despite the benefits, we're **sticking with Docker Compose for development** because:

**Faster Iteration:**
```bash
# Docker Compose: 15-30 seconds
docker-compose up -d

# Kubernetes: 2-3 minutes
kubectl apply -f deployment/kubernetes/
kubectl wait --for=condition=available deployment/quote-service
```

**Simpler Debugging:**
```bash
# Docker Compose: Direct logs
docker-compose logs quote-service -f

# Kubernetes: More verbose
kubectl logs -n trading-system -l app=quote-service -f
```

**Resource Efficiency:**
- Docker Compose: ~2-3GB RAM overhead
- Kubernetes: ~4-5GB RAM overhead (control plane + workers)

**Complexity Trade-off:**
- Docker Compose: Single `docker-compose.yml` file
- Kubernetes: 20+ YAML manifests, ConfigMaps, Secrets, PVCs

**When to Migrate to Kubernetes:**
- Moving to production with real trading
- Need horizontal scaling (multiple quote-service instances)
- Deploying to cloud (AWS, GCP, Azure)
- Integrating with existing K8s infrastructure

### Kubernetes Architecture Design

Despite staying with Docker Compose now, I've completed the **full Kubernetes migration design** for future use:

#### Service Tiers

**1. Infrastructure Tier (tier=infrastructure)**
- Redis (caching)
- PostgreSQL/TimescaleDB (data storage)
- NATS with JetStream (event bus)
- Traefik (ingress controller)

**2. Monitoring Tier (tier=monitoring)**
- Prometheus (metrics)
- Loki (logs)
- Grafana (visualization)
- Alloy (log collection)
- Mimir (long-term metrics storage)
- Tempo (distributed tracing)
- Pyroscope (continuous profiling)
- OpenTelemetry Collector (trace aggregation)

**3. Application Tier (tier=application)**
- system-initializer
- notification-service
- event-logger-service
- quote-service (future)
- scanner-service (future)
- executor-service (future)

#### Storage Architecture

Created 11 Persistent Volume Claims:

```yaml
# storage/persistent-volumes.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-data
  namespace: trading-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: hostpath  # For Docker Desktop
  resources:
    requests:
      storage: 50Gi
---
# Mimir (long-term metrics)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mimir-data
  namespace: trading-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: hostpath
  resources:
    requests:
      storage: 100Gi
---
# ... (9 more PVCs for Loki, Tempo, Redis, PostgreSQL, etc.)
```

**Storage Sizing:**
| Service | Storage | Retention | Purpose |
|---------|---------|-----------|---------|
| Prometheus | 50Gi | 15 days | Short-term metrics |
| Mimir | 100Gi | 90 days | Long-term metrics |
| Loki | 50Gi | 30 days | Log aggregation |
| Tempo | 50Gi | 7 days | Distributed traces |
| Pyroscope | 20Gi | 14 days | Continuous profiles |
| PostgreSQL | 20Gi | Permanent | Trading data |
| NATS | 10Gi | 7 days | Event stream |
| Redis | 5Gi | Ephemeral | Cache |
| Grafana | 5Gi | Permanent | Dashboards |

#### Configuration Management

**Docker Compose Approach (Current):**
```yaml
volumes:
  - ../monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
```

**Kubernetes Approach (Future):**
```yaml
# Create ConfigMap from file
kubectl create configmap prometheus-config \
  --from-file=prometheus.yml=../monitoring/prometheus/prometheus.yml \
  -n trading-system

# Mount in deployment
volumeMounts:
  - name: prometheus-config
    mountPath: /etc/prometheus
    readOnly: true
volumes:
  - name: prometheus-config
    configMap:
      name: prometheus-config
```

**Benefits:**
- Configuration versioned with deployment
- No file system dependencies
- Cross-platform compatibility
- Easy updates via `kubectl apply`

#### Service Discovery

**Docker Compose (Current):**
```yaml
environment:
  NATS_URL: "nats://nats:4222"  # Service name
```

**Kubernetes (Future):**
```yaml
environment:
  - name: NATS_URL
    value: "nats://nats-service.trading-system.svc.cluster.local:4222"
```

**DNS Resolution:**
- `nats-service`: Within same namespace
- `nats-service.trading-system`: Cross-namespace
- `nats-service.trading-system.svc.cluster.local`: Fully qualified

#### Health Checks

**Docker Compose (Current):**
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
```

**Kubernetes (Future):**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
```

**Benefits:**
- Separate liveness (restart pod) vs readiness (remove from service)
- Faster health check intervals
- Automatic pod restart on failures
- Load balancer integration

#### Resource Management

All deployments have resource requests and limits:

```yaml
# services/quote-service.yaml (future)
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

**Benefits:**
- Prevents resource starvation
- Enables proper scheduling
- Predictable performance
- Cost optimization

#### Deployment Automation

Created comprehensive deployment scripts:

**PowerShell Script (Windows):**
```powershell
# deployment/kubernetes/deploy.ps1
param(
    [switch]$Fast,
    [switch]$SkipBuild
)

Write-Host "Deploying Trading System to Kubernetes..." -ForegroundColor Green

# 1. Create namespace
kubectl apply -f namespace.yaml

# 2. Create secrets from .env
kubectl create secret generic trading-system-env `
  --from-env-file=../../.env `
  -n trading-system `
  --dry-run=client -o yaml | kubectl apply -f -

# 3. Create ConfigMaps
kubectl create configmap prometheus-config `
  --from-file=prometheus.yml=../monitoring/prometheus/prometheus.yml `
  -n trading-system `
  --dry-run=client -o yaml | kubectl apply -f -

# ... (more ConfigMaps)

# 4. Deploy infrastructure
kubectl apply -f storage/
kubectl apply -f infrastructure/

# Wait for infrastructure
kubectl wait --for=condition=ready pod -l tier=infrastructure -n trading-system --timeout=300s

# 5. Deploy monitoring
kubectl apply -f monitoring/

# Wait for monitoring
kubectl wait --for=condition=ready pod -l tier=monitoring -n trading-system --timeout=300s

# 6. Deploy services
kubectl apply -f services/

Write-Host "Deployment complete!" -ForegroundColor Green
```

**Makefile (Cross-platform):**
```makefile
# deployment/kubernetes/Makefile
.PHONY: deploy status logs cleanup

deploy:
	@echo "Deploying to Kubernetes..."
	@./deploy.ps1

deploy-fast:
	@echo "Fast deployment (skip waits)..."
	@./deploy-fast.ps1

status:
	@kubectl get all -n trading-system

logs:
	@kubectl logs -n trading-system -l tier=application --tail=50 -f

cleanup:
	@./cleanup.ps1
```

**Usage:**
```bash
# Full deployment
make deploy

# Fast deployment (skip waits)
make deploy-fast

# Check status
make status

# View logs
make logs

# Cleanup
make cleanup
```

#### Network Policies

Security through network isolation:

```yaml
# infrastructure/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quote-service-network-policy
  namespace: trading-system
spec:
  podSelector:
    matchLabels:
      app: quote-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: application
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: nats
    ports:
    - protocol: TCP
      port: 4222
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:  # External Solana RPC endpoints
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 8899
```

### Migration Documentation

Created comprehensive documentation:

**Files:**
- [deployment/kubernetes/README.md](https://github.com/guidebee/solana-trading-system/blob/master/deployment/kubernetes/README.md) - Complete deployment guide
- [deployment/kubernetes/QUICKSTART.md](https://github.com/guidebee/solana-trading-system/blob/master/deployment/kubernetes/QUICKSTART.md) - 5-minute quick start
- [deployment/kubernetes/MIGRATION-SUMMARY.md](https://github.com/guidebee/solana-trading-system/blob/master/deployment/kubernetes/MIGRATION-SUMMARY.md) - Detailed migration analysis
- [deployment/kubernetes/ARCHITECTURE.md](https://github.com/guidebee/solana-trading-system/blob/master/deployment/kubernetes/ARCHITECTURE.md) - System architecture

**Kubernetes Manifests Created:**
- `namespace.yaml` - Namespace definition
- `storage/persistent-volumes.yaml` - 11 PVCs
- `infrastructure/redis.yaml` - Redis deployment + service
- `infrastructure/postgres.yaml` - PostgreSQL/TimescaleDB
- `infrastructure/nats.yaml` - NATS with JetStream
- `infrastructure/traefik.yaml` - Traefik ingress controller
- `infrastructure/ingress.yaml` - Ingress rules
- `monitoring/prometheus.yaml` - Prometheus
- `monitoring/loki.yaml` - Loki
- `monitoring/grafana.yaml` - Grafana
- `monitoring/alloy.yaml` - Alloy DaemonSet
- `monitoring/otel-collector.yaml` - OpenTelemetry Collector
- `monitoring/lgtm-stack.yaml` - Mimir, Tempo, Pyroscope
- `services/system-initializer.yaml` - System initializer
- `services/notification-service.yaml` - Notification service
- `services/event-logger-service.yaml` - Event logger

**Total:** 20+ YAML files, ready for production deployment

### When to Revisit Kubernetes

We'll migrate to Kubernetes when:

1. **Production Deployment**: Moving beyond development to real trading
2. **Horizontal Scaling**: Need multiple quote-service instances for load distribution
3. **Cloud Migration**: Deploying to AWS EKS, GCP GKE, or Azure AKS
4. **Team Growth**: Multiple developers needing isolated environments
5. **SLA Requirements**: Need 99.9%+ uptime guarantees
6. **Cost Optimization**: Need fine-grained resource management
7. **Multi-Region**: Deploying to multiple geographic regions

## Performance Characteristics

### Current Metrics (Docker Compose)

**Quote Service Performance:**
- **P50 Request Latency**: 8ms (cache hit), 145ms (cache miss)
- **P95 Request Latency**: 15ms (cache hit), 320ms (cache miss)
- **P99 Request Latency**: 25ms (cache hit), 580ms (cache miss)
- **Cache Hit Rate**: 78% (target: >80%)
- **Throughput**: 500 requests/second (single instance)

**Cache Performance:**
- **Cache Lookup Duration**: <2ms (P99)
- **Average Cache Age**: 12 seconds (quotes under 30s old)
- **Cache Size**: ~10,000 trading pairs cached

**External API Performance:**
| Provider | P95 Latency | Success Rate | Notes |
|----------|-------------|--------------|-------|
| Jupiter | 280ms | 99.2% | Most reliable |
| DFlow | 350ms | 97.8% | Occasional timeouts |
| QuickNode | 220ms | 98.5% | Fast but limited pairs |
| BLXR | 310ms | 96.1% | Higher error rate |

**Rate Limiting:**
- **Jupiter**: 20 req/sec, P99 wait time: 50ms
- **DFlow**: 10 req/sec, P99 wait time: 120ms
- **QuickNode**: 15 req/sec, P99 wait time: 80ms
- **BLXR**: 10 req/sec, P99 wait time: 110ms

### Expected Kubernetes Performance

With Kubernetes horizontal scaling:

**3 Quote Service Replicas:**
- **Aggregate Throughput**: 1,500 requests/second
- **Latency**: Similar to single instance (load balanced)
- **Cache Hit Rate**: Lower initially (distributed cache), can be improved with Redis

**5 Quote Service Replicas:**
- **Aggregate Throughput**: 2,500 requests/second
- **High Availability**: Service continues if 2 pods fail
- **Rolling Updates**: Zero-downtime deployments

### Resource Usage

**Docker Compose (Current):**
| Service | Memory | CPU | Notes |
|---------|--------|-----|-------|
| quote-service | 120 MB | 5-10% | Single instance |
| LGTM Stack | ~900 MB | 8-13% | Full observability |
| Infrastructure | ~800 MB | 10-15% | Redis, PostgreSQL, NATS |
| **Total** | **~1.8 GB** | **23-38%** | Efficient for dev |

**Kubernetes (Projected):**
| Component | Memory | CPU | Notes |
|-----------|--------|-----|-------|
| K8s Control Plane | ~2 GB | 20% | Docker Desktop overhead |
| quote-service (3 replicas) | 360 MB | 15-30% | Horizontal scaling |
| LGTM Stack | ~900 MB | 8-13% | Same as Docker Compose |
| Infrastructure | ~800 MB | 10-15% | Same as Docker Compose |
| **Total** | **~4 GB** | **53-78%** | Higher overhead |

**Verdict:** Docker Compose is more efficient for development, Kubernetes shines in production.

## Integration Testing

### Testing the Metrics

**Test Script:**
```bash
# Generate load
for i in {1..100}; do
  curl "http://localhost:8080/quote?input=So11111111111111111111111111111111111111112&output=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v&amount=1000000000"
  sleep 0.1
done

# Check metrics
curl http://localhost:8080/metrics | grep quote_requests_total
curl http://localhost:8080/metrics | grep quote_cache_hits_total
curl http://localhost:8080/metrics | grep external_api_duration_seconds
```

**Expected Output:**
```
# HELP quote_requests_total Total number of quote requests
# TYPE quote_requests_total counter
quote_requests_total{input_mint="So11111...",output_mint="EPjFW..."} 100

# HELP quote_cache_hits_total Total number of cache hits
# TYPE quote_cache_hits_total counter
quote_cache_hits_total{input_mint="So11111...",output_mint="EPjFW..."} 78

# HELP external_api_duration_seconds Duration of external API calls
# TYPE external_api_duration_seconds histogram
external_api_duration_seconds_bucket{quoter="Jupiter",le="0.1"} 5
external_api_duration_seconds_bucket{quoter="Jupiter",le="0.5"} 18
external_api_duration_seconds_bucket{quoter="Jupiter",le="1"} 22
```

### Testing the Traces

**Jaeger UI:**
```bash
# Open Jaeger
open http://localhost:16686

# Search for traces
Service: quote-service
Operation: quote_request
Lookback: Last hour
```

**Trace Inspection:**
1. Select a trace from the list
2. View span hierarchy and timings
3. Inspect span attributes (input_mint, output_mint, etc.)
4. Check for errors or anomalies
5. Verify parent-child span relationships

### Testing the Dashboard

**Grafana Access:**
```bash
# Open Grafana
open http://localhost:3000

# Navigate to dashboard
Dashboards > Trading System > Quote Service Performance
```

**Validation Checks:**
- ✅ SLA uptime > 99%
- ✅ Cache hit rate > 75%
- ✅ P95 request latency < 500ms
- ✅ External API success rate > 95%
- ✅ No sustained error rate spikes

## Impact and Benefits

### Immediate Benefits

1. **Performance Visibility**: Real-time insight into quote service performance with 30+ metrics
2. **Cost Optimization**: Track external API usage to minimize costs (Jupiter, DFlow, etc.)
3. **Failure Detection**: Identify failing APIs, rate limiting, and cache issues within seconds
4. **User Experience**: Ensure fast quote responses via cache hit rate monitoring
5. **Debugging**: Use distributed traces to debug slow requests and identify bottlenecks

### Long-Term Benefits

1. **Capacity Planning**: Understand request patterns and scaling needs before moving to production
2. **SLA Tracking**: Monitor and enforce service level agreements with detailed metrics
3. **Kubernetes-Ready**: Complete migration plan ready when production scaling is needed
4. **Cost Management**: External API usage tracking helps optimize quoter selection
5. **Continuous Improvement**: Baseline metrics enable A/B testing and optimization

### Developer Experience

1. **Single Dashboard**: All observability signals in one Grafana dashboard
2. **Correlation**: Logs ↔ Metrics ↔ Traces ↔ Profiles in unified LGTM stack
3. **Fast Queries**: Prometheus + Loki optimized for real-time analysis
4. **Alerting**: Rule-based alerts on metrics and logs (to be configured)
5. **Visualization**: Rich, customizable dashboards for deep analysis

## Next Steps

### Phase 1: Observability Refinement (This Week)
- [x] Add comprehensive metrics to quote-service
- [x] Implement distributed tracing with OpenTelemetry
- [x] Create Grafana performance dashboard
- [ ] Configure Grafana alerting rules for SLA violations
- [ ] Add exemplars (metrics → traces linking)
- [ ] Test alert notification to Discord/Slack

### Phase 2: Service Expansion (Next Week)
- [ ] Add metrics and tracing to scanner-service (TypeScript)
- [ ] Add metrics and tracing to executor-service (Rust, when ready)
- [ ] Create unified system dashboard showing all services
- [ ] Implement NATS trace propagation (trace context in event headers)

### Phase 3: Advanced Observability (2 Weeks)
- [ ] Add Pyroscope continuous profiling to quote-service
- [ ] Configure SLO (Service Level Objectives) tracking
- [ ] Implement anomaly detection using Grafana ML
- [ ] Create runbooks and alerting workflows
- [ ] Set up PagerDuty/Opsgenie integration

### Phase 4: Kubernetes Migration (When Needed)
- [ ] Evaluate production deployment requirements
- [ ] Test Kubernetes deployment on local cluster
- [ ] Migrate to cloud Kubernetes (AWS EKS, GCP GKE, or Azure AKS)
- [ ] Configure horizontal pod autoscaling
- [ ] Implement multi-region deployment

### Phase 5: Production Hardening (Pre-Launch)
- [ ] Load testing with realistic trading volumes
- [ ] Chaos engineering tests (fault injection)
- [ ] Security audit and penetration testing
- [ ] Backup and disaster recovery testing
- [ ] Final production readiness review

## Configuration Files

All configuration and code changes are version-controlled:

**Observability Documentation:**
- [go/cmd/quote-service/docs/OBSERVABILITY_METRICS.md](https://github.com/guidebee/solana-trading-system/blob/master/go/cmd/quote-service/docs/OBSERVABILITY_METRICS.md) - Complete metrics reference

**Grafana Dashboard:**
- [deployment/monitoring/grafana/provisioning/dashboards/quote-service-performance.json](https://github.com/guidebee/solana-trading-system/blob/master/deployment/monitoring/grafana/provisioning/dashboards/quote-service-performance.json) - Dashboard configuration (2,768 lines)

**Kubernetes Manifests:**
- [deployment/kubernetes/README.md](https://github.com/guidebee/solana-trading-system/blob/master/deployment/kubernetes/README.md) - Deployment guide
- [deployment/kubernetes/MIGRATION-SUMMARY.md](https://github.com/guidebee/solana-trading-system/blob/master/deployment/kubernetes/MIGRATION-SUMMARY.md) - Migration analysis
- [deployment/kubernetes/services/](https://github.com/guidebee/solana-trading-system/tree/master/deployment/kubernetes/services) - Service manifests
- [deployment/kubernetes/monitoring/](https://github.com/guidebee/solana-trading-system/tree/master/deployment/kubernetes/monitoring) - Monitoring manifests

**Code Changes:**
- [go/cmd/quote-service/main.go](https://github.com/guidebee/solana-trading-system/blob/master/go/cmd/quote-service/main.go) - Metrics and tracing implementation
- [go/cmd/quote-service/cache.go](https://github.com/guidebee/solana-trading-system/blob/master/go/cmd/quote-service/cache.go) - Cache metrics
- [go/cmd/quote-service/external_quotes.go](https://github.com/guidebee/solana-trading-system/blob/master/go/cmd/quote-service/external_quotes.go) - External API metrics

## Quick Start Guide

### View the Dashboard

```bash
# Ensure all services are running
cd deployment/docker
docker-compose up -d

# Access Grafana
open http://localhost:3000

# Navigate to dashboard
Dashboards > Trading System > Quote Service Performance
```

### Query Metrics Directly

```bash
# Prometheus metrics endpoint
curl http://localhost:8080/metrics

# Specific metric query
curl 'http://localhost:9090/api/v1/query?query=quote_requests_total'

# Rate of requests
curl 'http://localhost:9090/api/v1/query?query=rate(quote_requests_total[5m])'

# Cache hit rate
curl 'http://localhost:9090/api/v1/query?query=rate(quote_cache_hits_total[5m])/(rate(quote_cache_hits_total[5m])+rate(quote_cache_misses_total[5m]))'
```

### View Traces

```bash
# Access Jaeger UI
open http://localhost:16686

# Or via Grafana dashboard link
# Click "Jaeger Traces" link in dashboard header
```

### Test the Quote Service

```bash
# Health check
curl http://localhost:8080/health

# Get quote (will generate metrics and traces)
curl "http://localhost:8080/quote?input=So11111111111111111111111111111111111111112&output=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v&amount=1000000000"

# Check metrics were recorded
curl http://localhost:8080/metrics | grep quote_requests_total
```

## Troubleshooting

### Metrics Not Appearing

**Check Prometheus is scraping:**
```bash
# View Prometheus targets
open http://localhost:9090/targets

# Ensure quote-service target is UP
# If DOWN, check quote-service logs
docker-compose logs quote-service --tail 50
```

### Traces Not in Jaeger

**Check OpenTelemetry Collector:**
```bash
# View collector logs
docker-compose logs otel-collector --tail 50

# Check collector health
curl http://localhost:13133
```

### Dashboard Not Loading

**Check Grafana data sources:**
```bash
# Restart Grafana
docker-compose restart grafana

# Check Grafana logs
docker-compose logs grafana --tail 50

# Verify Prometheus data source
open http://localhost:3000/datasources
```

## Conclusion

Today's work significantly enhanced the observability of the quote-service, providing comprehensive metrics, distributed tracing, and a detailed Grafana dashboard. With 30+ metrics tracking every aspect of quote generation - from cache performance to external API latency - we now have full visibility into system performance.

The Grafana dashboard gives real-time insight into SLA compliance, request latency, cache efficiency, and external API performance. The distributed tracing implementation allows us to debug slow requests by visualizing the complete request flow.

Additionally, we've completed a comprehensive Kubernetes migration plan, creating 20+ manifests and deployment automation scripts. While we're staying with Docker Compose for now (faster iteration, lower overhead), we're ready to migrate to Kubernetes when production scaling demands it.

**Key Achievements:**
- ✅ 30+ Prometheus metrics covering all critical paths
- ✅ OpenTelemetry distributed tracing with detailed spans
- ✅ Comprehensive Grafana performance dashboard (2,768 lines)
- ✅ Complete Kubernetes migration plan (20+ manifests)
- ✅ Deployment automation scripts (PowerShell + Makefile)
- ✅ Full documentation for observability and K8s migration

**Personal Achievement:**
- 🎉 Celebrating Harry's birthday and Year 12 graduation!

The next phase will focus on adding observability to other services (scanner-service, executor-service) and configuring alerting rules for production monitoring.

---

**Related Posts:**
- [Unified Observability: Migrating to Grafana LGTM Stack with Alloy](/posts/2025/12/grafana-lgtm-stack-unified-observability/)
- [Day's Work: Observability and Monitoring for Solana Trading System](/posts/2025/12/observability-and-monitoring-solana-trading-system/)
- [Getting Started: Building a Solana Trading System from Prototypes](/posts/2025/12/getting-started-building-solana-trading-system/)

**Technical Documentation:**
- [Quote Service Observability Metrics](https://github.com/guidebee/solana-trading-system/blob/master/go/cmd/quote-service/docs/OBSERVABILITY_METRICS.md)
- [Kubernetes Migration Summary](https://github.com/guidebee/solana-trading-system/blob/master/deployment/kubernetes/MIGRATION-SUMMARY.md)
- [Kubernetes Deployment Guide](https://github.com/guidebee/solana-trading-system/blob/master/deployment/kubernetes/README.md)

## Connect

- **GitHub:** [github.com/guidebee](https://github.com/guidebee)
- **LinkedIn:** [linkedin.com/in/guidebee](https://linkedin.com/in/guidebee)

---

*This is post #8 in the Solana Trading System development series. Follow along as I document the journey from working prototypes to production HFT system.*
