---
layout: single
title: "Environment Configuration Architecture"
permalink: /docs/27-env-configrations/
author_profile: true
toc: true
toc_label: "On This Page"
---

# Environment Configuration Architecture

**Version:** 1.0
**Last Updated:** 2025-12-27
**Author:** Solution Architect

## Table of Contents

1. [Overview](#overview)
2. [Architecture Rationale](#architecture-rationale)
3. [Configuration Structure](#configuration-structure)
4. [Configuration Categories](#configuration-categories)
5. [Loading Priority](#loading-priority)
6. [Setup Guide](#setup-guide)
7. [Docker Compose Integration](#docker-compose-integration)
8. [Usage Examples](#usage-examples)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Migration Guide](#migration-guide)

---

## Overview

This HFT trading system uses a **hybrid distributed configuration architecture** that balances convenience with service independence. Configuration is organized into:

- **Shared Infrastructure** (`.env.shared`) - Common services (NATS, Redis, Postgres, observability)
- **Service-Specific Configs** (`go/.env`, `rust/.env`, `ts/.env`) - Per-stack configuration
- **Application Secrets** (`.env`) - API keys, credentials, notifications

### Key Principles

✅ **Production-Only Environment** - All services use `ENVIRONMENT=production` (no dev/staging modes)
✅ **Service Independence** - Each service can be deployed with only its required config
✅ **Security (Least Privilege)** - Services only access secrets they need
✅ **Clear Ownership** - Each technology stack owns its configuration
✅ **Docker-Native** - Multiple `env_file` entries for layered config loading

---

## Architecture Rationale

### Why Distributed Configuration?

**✅ Advantages:**

1. **Service Independence** - Deploy quote-service without needing scanner configs
2. **Security Isolation** - Go service doesn't need TypeScript API keys
3. **Team Scalability** - Go, Rust, and TypeScript developers work independently
4. **Selective Deployment** - Deploy services to different environments with different configs
5. **Clear Ownership** - Each stack owns and maintains its config
6. **Version Control Safety** - No merge conflicts on single config file

**❌ Problems with Single Root `.env`:**

1. **Monolithic Coupling** - All services depend on one massive config file
2. **Security Risk** - All secrets exposed to all services (violates least privilege)
3. **Deployment Complexity** - Can't deploy services independently
4. **Merge Conflicts** - Multiple developers editing same file
5. **Configuration Bloat** - 500+ line config files that are hard to maintain

### Why NOT Multiple `.env` Files Per Service?

Some projects use one `.env` per microservice (e.g., `quote-service/.env`, `scanner-service/.env`). We don't do this because:

- **Duplication** - Infrastructure config (NATS, Redis) repeated in every service
- **Inconsistency Risk** - Different services using different NATS URLs
- **Maintenance Burden** - Update 10 files to change one Redis password
- **Docker Complexity** - Need complex volume mounts for each service

Our hybrid approach gives **shared infrastructure + service-specific configs** without duplication.

---

## Configuration Structure

```
solana-trading-system/
├── .env.shared              # ✅ Shared infrastructure (NATS, Redis, Postgres, RPC, observability)
├── .env.shared.example      # Template for shared infrastructure
├── .env                     # ✅ Application secrets (API keys, email, wallet keys)
├── .env.example             # Template for application secrets
│
├── go/
│   ├── .env                 # ✅ Go service config (quote service, quoters, rate limits)
│   ├── .env.example         # Template for Go service
│   └── .env.example.quoters # Detailed quoter configuration reference
│
├── rust/
│   ├── .env                 # ✅ Rust service config (RPC proxy ports, Shredstream)
│   └── .env.example         # Template for Rust service
│
├── ts/
│   ├── .env                 # ✅ TypeScript service config (scanners, planners, executors)
│   ├── .env.example         # Template for TypeScript services
│   └── apps/
│       ├── grpc-test-client/.env.example     # Reference to ../../.env
│       ├── scanner-service/.env.example      # Reference to ../../.env
│       └── system-initializer/.env.example   # Reference to ../../.env
│
└── deployment/
    └── docker/
        ├── docker-compose.yml                # Uses multiple env_file entries
        └── docker-compose.quote-service.yml  # Uses multiple env_file entries
```

### File Sizes (Approximate)

| File | Lines | Purpose |
|------|-------|---------|
| `.env.shared` | ~80 | Infrastructure baseline |
| `.env` | ~40 | Application secrets |
| `go/.env` | ~175 | Quote service + quoters |
| `rust/.env` | ~15 | RPC proxy config |
| `ts/.env` | ~50 | TypeScript services |

---

## Configuration Categories

### 1. Shared Infrastructure (`.env.shared`)

**Purpose:** Configuration shared by ALL services (infrastructure baseline)

**Contents:**
```bash
# PostgreSQL Database
POSTGRES_USER=postgres
POSTGRES_PASSWORD=trading1234
POSTGRES_DB=trading_system
POSTGRES_HOST=localhost
POSTGRES_PORT=5432

# Redis Cache
REDIS_HOST=localhost
REDIS_PORT=6379

# NATS Message Streaming
NATS_URL=nats://localhost:4222
NATS_SERVERS=nats://localhost:4222
NATS_CLUSTER_ID=trading-cluster
NATS_MAX_RECONNECT=-1
NATS_RECONNECT_WAIT=2000
NATS_TIMEOUT=5000

# Observability Stack
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTLP_ENDPOINT=http://localhost:4318
JAEGER_ENDPOINT=http://localhost:14268/api/traces
PROMETHEUS_ENDPOINT=http://localhost:9090
LOKI_ENDPOINT=http://localhost:3100
ENVIRONMENT=production
LOG_LEVEL=info

# Solana RPC Endpoints
SOLANA_RPC_PRIMARY=https://api.mainnet-beta.solana.com
RPC_RATE_LIMIT=20

# Grafana & PGAdmin
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=your_password
PGADMIN_DEFAULT_EMAIL=admin@example.com
PGADMIN_DEFAULT_PASSWORD=your_password

# Traefik
TRAEFIK_DOMAIN=localhost
```

**Who Uses This:**
- ✅ All Docker services (postgres, redis, nats, grafana, etc.)
- ✅ Go quote service (for NATS, observability)
- ✅ Rust RPC proxy (for observability)
- ✅ TypeScript services (for NATS, observability)

---

### 2. Application Secrets (`.env`)

**Purpose:** Application-wide secrets and API keys

**Contents:**
```bash
# Solana Wallet Private Keys (NEVER commit!)
# WALLET_PRIVATE_KEY=your_base58_private_key_here

# External API Keys
JUPITER_API_KEY=your_jupiter_api_key
# HELIUS_API_KEY=your_helius_api_key
# QUICKNODE_API_KEY=your_quicknode_api_key
# JITO_API_KEY=your_jito_api_key

# Email Notification Service
EMAIL_ENABLED=true
EMAIL_SERVICE=gmail
EMAIL_USER=your_email@gmail.com
EMAIL_PASSWORD=your_app_password
EMAIL_FROM=Solana Trading System
EMAIL_DEFAULT_RECIPIENT=your_email@example.com

# Notification Rules
NOTIFY_SYSTEM_LIFECYCLE=true
NOTIFY_CRITICAL_HEALTH=true
NOTIFY_ARBITRAGE_THRESHOLD_BPS=100  # 1% minimum profit to notify
NOTIFY_DAILY_SUMMARY=true
NOTIFY_DAILY_SUMMARY_TIME=09:00
```

**Who Uses This:**
- ✅ Notification service (email config)
- ✅ Go quote service (Jupiter API key if using external quoters)
- ✅ TypeScript executors (wallet keys, Jito API)
- ✅ Any service needing external API keys

---

### 3. Go Service Config (`go/.env`)

**Purpose:** Quote service and external quoter configuration

**Contents:**
```bash
# Service Identification
SERVICE_NAME=quote-service
SERVICE_VERSION=1.0.0
ENVIRONMENT=production
LOG_LEVEL=info

# OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

# RPC Endpoints (can override shared)
RPC_ENDPOINTS=https://mainnet.helius-rpc.com/?api-key=YOUR_KEY

# External Quoters
QUOTERS_ENABLED=true
QUOTERS_TIMEOUT=30
QUOTERS_USE_RATE_LIMIT=true

# Quoter Providers
QUOTERS_USE_JUPITER=true
QUOTERS_USE_JUPITER_ULTRA=false
QUOTERS_USE_DFLOW=true
QUOTERS_USE_OKX=true
QUOTERS_USE_BLXR=true
QUOTERS_USE_HUMIDIFI=true  # Dark pool
QUOTERS_USE_GOONFI=true    # Dark pool
QUOTERS_USE_ZEROFI=true    # Dark pool

# Rate Limits (requests per second)
# CRITICAL: Jupiter API = 1 RPS total shared across all Jupiter-based quoters
QUOTERS_JUPITER_RATE_LIMIT=0.16         # Jupiter shares 1 RPS pool
QUOTERS_JUPITER_ULTRA_RATE_LIMIT=0.16   # Shares Jupiter's rate limit
QUOTERS_HUMIDIFI_RATE_LIMIT=0.16        # Dark pool (uses Jupiter API)
QUOTERS_GOONFI_RATE_LIMIT=0.16          # Dark pool (uses Jupiter API)
QUOTERS_ZEROFI_RATE_LIMIT=0.16          # Dark pool (uses Jupiter API)
QUOTERS_DFLOW_RATE_LIMIT=5.0            # Separate infrastructure
QUOTERS_OKX_RATE_LIMIT=10.0             # Separate infrastructure
QUOTERS_BLXR_RATE_LIMIT=5.0             # Separate infrastructure

# OKX API Credentials
OKX_API_KEY=your_okx_api_key
OKX_SECRET_KEY=your_okx_secret_key
OKX_API_PASSPHRASE=your_okx_passphrase

# BLXR API Credentials
BLXR_API_BASE=https://ny.solana.dex.blxrbdn.com
BLXR_API_AUTH=your_blxr_auth_token

# Proxy Configuration (optional)
QUOTERS_PROXY_ENABLED=false
QUOTERS_JUPITER_PROXY_ENABLED=false
```

**Who Uses This:**
- ✅ Go quote service (`go/cmd/quote-service`)
- ✅ Go event logger service (`go/cmd/event-logger-service`)
- ✅ Go pool discovery service (`go/cmd/pool-discovery-service`)

**See Also:** `go/.env.example.quoters` for comprehensive quoter documentation

---

### 4. Rust Service Config (`rust/.env`)

**Purpose:** Rust service configuration (RPC proxy, Shredstream)

**Contents:**
```bash
# Service Identification
SERVICE_NAME=rust-services
SERVICE_VERSION=1.0.0
ENVIRONMENT=production
LOG_LEVEL=info

# RPC Proxy Configuration
HTTP_PROXY_PORT=3030
WS_PROXY_PORT=8900
RPC_PROXY_POOL_SIZE=100
RPC_PROXY_TIMEOUT_MS=5000

# Shredstream Configuration (optional)
# SHREDSTREAM_ENDPOINT=tcp://127.0.0.1:5555
# SHREDSTREAM_BATCH_SIZE=1000

# Observability (inherits from .env.shared, can override)
# OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
# LOKI_ENDPOINT=http://localhost:3100
```

**Who Uses This:**
- ✅ Solana RPC Proxy (`rust/solana-rpc-proxy`)
- ✅ Shredstream integration (future)
- ✅ Transaction planning services (future)

---

### 5. TypeScript Service Config (`ts/.env`)

**Purpose:** TypeScript services configuration (scanners, planners, executors)

**Contents:**
```bash
# Service Environment
NODE_ENV=production  # CRITICAL: Production-only (see CLAUDE.md)
SERVICE_VERSION=0.1.0
LOG_LEVEL=info
PRETTY_LOGS=true

# gRPC Configuration (Quote Service Client)
GRPC_HOST=localhost
GRPC_PORT=50051
GRPC_KEEPALIVE_MS=10000
GRPC_RECONNECT_DELAY_MS=5000

QUOTE_SERVICE_HOST=localhost
QUOTE_SERVICE_PORT=50051

# Scanner Service Configuration
MONITOR_LST_PAIRS=true
MONITOR_STABLECOIN_PAIRS=false

# Arbitrage Configuration
ARB_MIN_PROFIT_BPS=50              # 0.5% minimum profit
ARB_MAX_SLIPPAGE_BPS=50            # 0.5% max slippage
ARB_MIN_CONFIDENCE=0.8             # 80% minimum confidence

# Executor Service Configuration
PRIORITY_FEE=10000                 # Priority fee (lamports) = 0.00001 SOL
JITO_TIP=100000                    # Jito tip (lamports) = 0.0001 SOL
SWAP_FEE_BPS=25                    # DEX swap fee = 0.25%

JITO_BLOCK_ENGINE_URL=https://mainnet.block-engine.jito.wtf

# Test Configuration
TOKEN_PAIRS=So11111111111111111111111111111111111111112:EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
TEST_AMOUNTS=100000000,1000000000,10000000000  # 0.1 SOL, 1 SOL, 10 SOL
SLIPPAGE_BPS=50
LOG_DIR=./logs

# Observability
METRICS_PORT=9092
```

**Who Uses This:**
- ✅ Scanner service (`ts/apps/scanner-service`)
- ✅ Strategy service (planner)
- ✅ Executor service
- ✅ System manager
- ✅ System auditor
- ✅ Notification service
- ✅ gRPC test client

---

## Loading Priority

### Docker Compose Loading Order

Services load env files in this order (later files override earlier ones):

```yaml
services:
  quote-service:
    env_file:
      - ../../.env.shared    # 1. Infrastructure baseline (loaded first)
      - ../../go/.env        # 2. Service-specific config
      - ../../.env           # 3. Application secrets (loaded last, can override)
```

**Example Override:**

If `.env.shared` has `LOG_LEVEL=info` but `go/.env` has `LOG_LEVEL=debug`, the Go service will use `debug`.

### Environment Variable Precedence

**Highest Priority (wins):**
1. `environment:` section in `docker-compose.yml` (direct overrides)
2. `.env` file (application secrets)
3. `<service>/.env` file (service-specific)
4. `.env.shared` file (infrastructure baseline)
5. Service defaults (if variable not set)

**Example:**

```yaml
services:
  quote-service:
    env_file:
      - ../../.env.shared    # LOG_LEVEL=info
      - ../../go/.env        # LOG_LEVEL=debug (overrides shared)
      - ../../.env           # (no LOG_LEVEL, doesn't override)
    environment:
      - LOG_LEVEL=trace      # Final value: trace (highest priority)
```

**Result:** `LOG_LEVEL=trace`

---

## Setup Guide

### Initial Setup (New Installation)

```powershell
# 1. Navigate to repository root
cd C:\Trading\solana-trading-system

# 2. Copy shared infrastructure config
Copy-Item .env.shared.example .env.shared

# 3. Copy application secrets
Copy-Item .env.example .env

# 4. Copy service-specific configs
Copy-Item go\.env.example go\.env
Copy-Item rust\.env.example rust\.env
Copy-Item ts\.env.example ts\.env

# 5. Edit configuration files (in this order)
# Start with infrastructure baseline
code .env.shared

# Then application secrets
code .env

# Then service-specific configs
code go\.env
code rust\.env
code ts\.env
```

### Configuration Checklist

**`.env.shared` - Infrastructure:**
- [ ] Update `POSTGRES_PASSWORD`
- [ ] Update `PGADMIN_DEFAULT_PASSWORD`
- [ ] Update `GF_SECURITY_ADMIN_PASSWORD`
- [ ] Add premium RPC endpoints to `SOLANA_RPC_BACKUP`
- [ ] Verify NATS, Redis, observability endpoints

**`.env` - Application Secrets:**
- [ ] Add `JUPITER_API_KEY` (if using external quoters)
- [ ] Add `HELIUS_API_KEY`, `QUICKNODE_API_KEY` (if using)
- [ ] Configure `EMAIL_USER` and `EMAIL_PASSWORD` (Gmail app password)
- [ ] Set `EMAIL_DEFAULT_RECIPIENT`
- [ ] Configure notification thresholds

**`go/.env` - Quote Service:**
- [ ] Enable desired external quoters (`QUOTERS_USE_JUPITER=true`, etc.)
- [ ] Configure rate limits (Jupiter = 0.16 req/s to stay under 1 RPS limit)
- [ ] Add `OKX_API_KEY`, `OKX_SECRET_KEY`, `OKX_API_PASSPHRASE` (if using OKX)
- [ ] Add `BLXR_API_BASE`, `BLXR_API_AUTH` (if using BLXR)
- [ ] Add RPC endpoints with API keys

**`rust/.env` - Rust Services:**
- [ ] Verify `HTTP_PROXY_PORT=3030` (avoid port conflicts)
- [ ] Verify `WS_PROXY_PORT=8900` (avoid port conflicts)

**`ts/.env` - TypeScript Services:**
- [ ] Set arbitrage thresholds (`ARB_MIN_PROFIT_BPS`)
- [ ] Configure Jito tip and priority fees
- [ ] Add token pairs to monitor

### Verification

```powershell
# Check that all required files exist
ls .env.shared, .env, go\.env, rust\.env, ts\.env

# Test Docker Compose config (dry run)
cd deployment\docker
docker-compose config

# Start infrastructure services
docker-compose up -d redis postgres nats

# Check service logs
docker-compose logs -f
```

---

## Docker Compose Integration

### Service Configuration Examples

#### Infrastructure Service (Redis)

```yaml
services:
  redis:
    image: redis:7.2-alpine
    container_name: trading-redis
    # No env_file needed - uses default config
    ports:
      - "${REDIS_PORT:-6379}:6379"
    # REDIS_PORT comes from .env.shared (loaded by compose)
```

#### Database Service (Postgres)

```yaml
services:
  postgres:
    image: timescale/timescaledb:latest-pg16
    container_name: trading-postgres
    env_file:
      - ../../.env.shared    # POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
      - ../../.env           # (can override if needed)
    environment:
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8"
      TIMESCALEDB_TELEMETRY: "off"
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
```

#### Go Service (Quote Service)

```yaml
services:
  quote-service:
    build:
      context: ../../go
      dockerfile: Dockerfile.quote-service
    container_name: quote-service
    env_file:
      - ../../.env.shared    # Infrastructure (NATS, observability)
      - ../../go/.env        # Quote service config (quoters, rate limits)
      - ../../.env           # Application secrets (Jupiter API key)
    environment:
      # Direct overrides (highest priority)
      - PORT=8080
      - ENVIRONMENT=production
    ports:
      - "8080:8080"
```

#### TypeScript Service (Scanner)

```yaml
services:
  ts-scanner-service:
    build:
      context: ../../
      dockerfile: deployment/docker/scanner-service/Dockerfile.ts-scanner-service
    container_name: ts-scanner-service
    env_file:
      - ../../.env.shared    # Infrastructure (NATS, observability)
      - ../../ts/.env        # TypeScript service config (arbitrage thresholds)
      - ../../.env           # Application secrets
    environment:
      - NODE_ENV=${NODE_ENV:-production}
      - SERVICE_NAME=ts-scanner-service
      - NATS_SERVERS=nats://nats:4222  # Override for Docker network
    ports:
      - "9096:9096"
```

### Running Services

```powershell
cd deployment\docker

# Start all infrastructure
docker-compose up -d

# Start specific service
docker-compose up -d quote-service

# View logs
docker-compose logs -f quote-service

# Check environment variables (debug)
docker-compose exec quote-service env | grep -E "QUOTERS|NATS|ENVIRONMENT"

# Restart service after config change
docker-compose restart quote-service
```

---

## Usage Examples

### Example 1: Adding a New External Quoter

**Scenario:** Enable DFlow quoter for the quote service

**Steps:**

1. Edit `go/.env`:
```bash
# Change from false to true
QUOTERS_USE_DFLOW=true

# Set rate limit (5 req/s for DFlow)
QUOTERS_DFLOW_RATE_LIMIT=5.0
```

2. Restart quote service:
```powershell
cd deployment\docker
docker-compose restart quote-service
```

3. Verify quoter is enabled:
```powershell
curl "http://localhost:8080/quoters/stats"
```

**Result:** DFlow quoter now provides quotes alongside Jupiter, OKX, etc.

---

### Example 2: Changing Database Password

**Scenario:** Update Postgres password for security

**Steps:**

1. Edit `.env.shared`:
```bash
POSTGRES_PASSWORD=new_secure_password_123
```

2. Update database (requires recreation):
```powershell
cd deployment\docker

# Stop postgres
docker-compose stop postgres

# Remove old volume (WARNING: deletes data!)
docker volume rm docker_postgres_data

# Recreate with new password
docker-compose up -d postgres
```

3. Update PGAdmin connection (manual step in UI)

**Result:** Postgres now uses new password. All services automatically use updated password from `.env.shared`.

---

### Example 3: Deploying Quote Service to Remote Server

**Scenario:** Deploy quote service to dedicated server with low latency to RPC endpoints

**Steps:**

1. Copy required env files to server:
```powershell
# On your dev machine
scp .env.shared user@quote-server:/opt/trading/
scp go/.env user@quote-server:/opt/trading/go/
scp .env user@quote-server:/opt/trading/
```

2. On remote server, override RPC endpoints for low latency:
```bash
# Edit go/.env on remote server
RPC_ENDPOINTS=https://low-latency-rpc.local:8899,https://backup-rpc.local:8899
```

3. Run quote service (standalone):
```bash
cd /opt/trading/go
./bin/quote-service -port 8080 -refresh 30 -slippage 50 -ratelimit 20
```

**Result:** Quote service runs on dedicated server with optimized RPC endpoints, using the same config structure.

---

### Example 4: Development vs Production Configuration

**Scenario:** Run development version with verbose logging

**Important:** This system uses **production-only environment** (see CLAUDE.md). There is no `NODE_ENV=development` mode. For testing:

**Steps:**

1. Create a separate config set (don't modify existing):
```powershell
Copy-Item .env.shared dev.env.shared
Copy-Item go\.env go\dev.env
```

2. Edit `dev.env.shared`:
```bash
# Use local infrastructure (not production)
POSTGRES_DB=trading_system_dev
REDIS_PORT=6380  # Different port to avoid conflict

# Verbose logging for debugging
LOG_LEVEL=debug
```

3. Edit `go/dev.env`:
```bash
# More verbose logging
LOG_LEVEL=debug

# Disable external quoters (faster testing)
QUOTERS_ENABLED=false
```

4. Run with dev config:
```powershell
# Manually load dev configs (Docker Compose doesn't support this directly)
# Better: Use separate docker-compose.dev.yml file
docker-compose -f docker-compose.dev.yml up -d
```

**Note:** Even in "dev" config, use `ENVIRONMENT=production` (per CLAUDE.md). The difference is in infrastructure targets (local vs cloud) and log levels, NOT in runtime mode.

---

### Example 5: Testing with Mock/Stub Services

**Scenario:** Test scanner service without real NATS

**Steps:**

1. Create test override file `ts/test.env`:
```bash
# Override NATS URL to point to mock
NATS_URL=nats://localhost:14222

# Enable verbose logging
LOG_LEVEL=debug
PRETTY_LOGS=true

# Reduce thresholds for faster testing
ARB_MIN_PROFIT_BPS=10  # 0.1% (vs 0.5% in prod)
```

2. Start mock NATS server:
```powershell
# In separate terminal
docker run -p 14222:4222 nats:latest
```

3. Run scanner with test config:
```powershell
cd ts/apps/scanner-service

# Manually set env vars (or use dotenv loader)
$env:NATS_URL="nats://localhost:14222"
$env:LOG_LEVEL="debug"

pnpm dev
```

**Result:** Scanner connects to mock NATS on port 14222 for isolated testing.

---

## Best Practices

### 1. **Never Commit Secrets**

❌ **Bad:**
```bash
# .env (committed to Git)
POSTGRES_PASSWORD=mysecretpassword123
OKX_API_KEY=real-api-key-here
```

✅ **Good:**
```bash
# .env.example (committed to Git)
POSTGRES_PASSWORD=your_postgres_password_here
OKX_API_KEY=your_okx_api_key

# .env (in .gitignore, NOT committed)
POSTGRES_PASSWORD=actual-secret-password
OKX_API_KEY=actual-api-key
```

**Check `.gitignore`:**
```gitignore
# Environment files (secrets)
.env
.env.shared
go/.env
rust/.env
ts/.env

# Keep templates
!.env.example
!.env.shared.example
!go/.env.example
!rust/.env.example
!ts/.env.example
```

---

### 2. **Use Environment-Specific Overrides Sparingly**

❌ **Bad - Overriding too much in docker-compose.yml:**
```yaml
services:
  quote-service:
    environment:
      - QUOTERS_ENABLED=true
      - QUOTERS_USE_JUPITER=true
      - QUOTERS_USE_DFLOW=true
      - QUOTERS_USE_OKX=true
      # ... 50 more lines
```

✅ **Good - Use env files, override only deployment-specific values:**
```yaml
services:
  quote-service:
    env_file:
      - ../../.env.shared
      - ../../go/.env
      - ../../.env
    environment:
      - NATS_SERVERS=nats://nats:4222  # Docker network override
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318  # Docker override
```

**Rule:** Only override in `environment:` section if the value is specific to Docker deployment (e.g., Docker network service names).

---

### 3. **Document Required Variables**

Add comments to `.env.example` files explaining what each variable does:

✅ **Good:**
```bash
# =============================================================================
# External Quoters Configuration
# =============================================================================

# Enable external quote providers (Jupiter, OKX, DFlow, etc.)
# Set to false to use only local pool calculations (faster, no API dependency)
QUOTERS_ENABLED=true

# Timeout for external quote requests (seconds)
# Increase if experiencing timeout errors, decrease for faster failover
QUOTERS_TIMEOUT=30

# Jupiter API rate limit (requests per second)
# CRITICAL: Jupiter API has 1 RPS limit shared across ALL Jupiter-based quoters
# With Jupiter Oracle using ~0.2 req/s, set this to 0.16 to stay under limit
QUOTERS_JUPITER_RATE_LIMIT=0.16
```

---

### 4. **Validate Configuration on Startup**

Services should validate required env vars on startup:

```go
// Go example
func validateConfig() error {
    required := []string{"NATS_URL", "REDIS_HOST", "POSTGRES_DB"}
    for _, key := range required {
        if os.Getenv(key) == "" {
            return fmt.Errorf("missing required env var: %s", key)
        }
    }
    return nil
}
```

```typescript
// TypeScript example
function validateConfig() {
  const required = ['NATS_URL', 'GRPC_HOST', 'ARB_MIN_PROFIT_BPS'];
  const missing = required.filter(key => !process.env[key]);
  if (missing.length > 0) {
    throw new Error(`Missing required env vars: ${missing.join(', ')}`);
  }
}
```

---

### 5. **Use Defaults for Non-Critical Values**

```bash
# Good - Provide sensible defaults
QUOTERS_TIMEOUT=${QUOTERS_TIMEOUT:-30}
LOG_LEVEL=${LOG_LEVEL:-info}
RETRY_MAX_ATTEMPTS=${RETRY_MAX_ATTEMPTS:-3}

# Bad - Force user to set everything
QUOTERS_TIMEOUT=
LOG_LEVEL=
RETRY_MAX_ATTEMPTS=
```

---

### 6. **Keep Secrets in `.env`, Config in Service Files**

**Secrets** (`.env`):
- API keys, passwords, wallet private keys
- Anything that changes per deployment environment
- High-security values

**Configuration** (`go/.env`, `ts/.env`):
- Feature flags (`QUOTERS_ENABLED`)
- Thresholds (`ARB_MIN_PROFIT_BPS`)
- Timeouts, rate limits
- Service behavior settings

**Infrastructure** (`.env.shared`):
- Connection strings (NATS, Redis, Postgres)
- Observability endpoints
- Common RPC endpoints

---

## Troubleshooting

### Issue 1: Service Can't Connect to NATS

**Symptom:**
```
Error: connect ECONNREFUSED 127.0.0.1:4222
```

**Debug Steps:**

1. Check if NATS is running:
```powershell
docker-compose ps nats
```

2. Check NATS_URL in service:
```powershell
docker-compose exec quote-service env | grep NATS
```

3. Verify `.env.shared` has correct NATS config:
```bash
# For Docker services, use Docker network name
NATS_URL=nats://nats:4222  # NOT localhost

# For host-based services, use localhost
NATS_URL=nats://localhost:4222
```

4. Check Docker network:
```powershell
docker network inspect docker_trading-system
```

**Solution:**

For **Docker services**, use Docker service name:
```yaml
environment:
  - NATS_SERVERS=nats://nats:4222
```

For **host-based services** (running outside Docker), use localhost:
```bash
NATS_URL=nats://localhost:4222
```

---

### Issue 2: Environment Variables Not Loading

**Symptom:**
```
Service using default values instead of .env values
```

**Debug Steps:**

1. Check if .env files exist:
```powershell
ls .env.shared, .env, go\.env, rust\.env, ts\.env
```

2. Check Docker Compose config (shows resolved values):
```powershell
cd deployment\docker
docker-compose config | grep -A5 "quote-service"
```

3. Check running container env vars:
```powershell
docker-compose exec quote-service env | sort
```

4. Verify env_file paths in docker-compose.yml:
```yaml
# Paths are relative to docker-compose.yml location
env_file:
  - ../../.env.shared    # Resolves to project root
  - ../../go/.env        # Resolves to go/.env
```

**Solution:**

Ensure `env_file` paths are correct (relative to `docker-compose.yml` location, not project root):
```yaml
# deployment/docker/docker-compose.yml
services:
  quote-service:
    env_file:
      - ../../.env.shared    # ✅ Correct: goes up 2 levels to root
      - ../go/.env           # ❌ Wrong: should be ../../go/.env
```

---

### Issue 3: Quoter Rate Limiting

**Symptom:**
```
Jupiter API error: 429 Too Many Requests
```

**Debug Steps:**

1. Check quoter stats:
```powershell
curl "http://localhost:8080/quoters/stats"
```

2. Verify rate limits in `go/.env`:
```bash
# Jupiter has 1 RPS limit shared across ALL Jupiter-based quoters
QUOTERS_JUPITER_RATE_LIMIT=0.16      # Too high if multiple quoters enabled
QUOTERS_HUMIDIFI_RATE_LIMIT=0.16
QUOTERS_GOONFI_RATE_LIMIT=0.16
QUOTERS_ZEROFI_RATE_LIMIT=0.16
```

**Solution:**

See CLAUDE.md **External API Rate Limiting** section for detailed explanation.

**Quick fix** - Reduce concurrent Jupiter-based quoters:
```bash
# Disable some dark pools to reduce API usage
QUOTERS_USE_HUMIDIFI=false
QUOTERS_USE_GOONFI=false
QUOTERS_USE_ZEROFI=false

# Or increase rate limit if you have premium API key
QUOTERS_JUPITER_RATE_LIMIT=10.0  # For premium tier
```

---

### Issue 4: Docker Compose Can't Find .env Files

**Symptom:**
```
ERROR: Couldn't find env file: /path/to/.env.shared
```

**Debug Steps:**

1. Check current directory:
```powershell
pwd
# Should be: C:\Trading\solana-trading-system\deployment\docker
```

2. Check if .env.shared exists (relative to docker-compose.yml):
```powershell
ls ..\..\env.shared
```

3. Check docker-compose.yml paths:
```yaml
env_file:
  - ../../.env.shared    # Relative to docker-compose.yml location
```

**Solution:**

Always run `docker-compose` from the directory containing `docker-compose.yml`:
```powershell
# ✅ Correct
cd deployment\docker
docker-compose up -d

# ❌ Wrong
cd C:\Trading\solana-trading-system
docker-compose -f deployment\docker\docker-compose.yml up -d
# (This changes relative paths)
```

---

### Issue 5: Secrets Not Loading in Docker

**Symptom:**
```
Missing API key, using default behavior
```

**Debug Steps:**

1. Verify .env file exists and has values:
```powershell
cat .env | grep JUPITER_API_KEY
```

2. Check if .env is in docker-compose.yml env_file list:
```yaml
env_file:
  - ../../.env.shared
  - ../../go/.env
  - ../../.env           # ✅ Must include this for secrets
```

3. Check container has the variable:
```powershell
docker-compose exec quote-service env | grep JUPITER
```

**Solution:**

Ensure `.env` is included in `env_file` list for services that need secrets:
```yaml
services:
  quote-service:
    env_file:
      - ../../.env.shared
      - ../../go/.env
      - ../../.env       # Application secrets (API keys, etc.)
```

---

### Issue 6: Production-Only Requirement Violated

**Symptom:**
```
Service started with NODE_ENV=development
```

**Fix:**

Per CLAUDE.md, this system **only uses production environment**:

```bash
# ❌ Wrong
NODE_ENV=development

# ✅ Correct
NODE_ENV=production
ENVIRONMENT=production
```

Update `ts/.env`:
```bash
NODE_ENV=production  # CRITICAL: Production-only (see CLAUDE.md)
```

**Rationale:** HFT systems operate in real-time with live capital. Testing should be done in separate infrastructure with different capital allocation, NOT different runtime modes.

---

## Migration Guide

### Migrating from Old Single .env Structure

**Old Structure:**
```
solana-trading-system/
├── .env  (1000+ lines with everything)
└── deployment/docker/.env  (duplicate config)
```

**New Structure:**
```
solana-trading-system/
├── .env.shared              # Infrastructure
├── .env                     # Secrets only
├── go/.env                  # Go config
├── rust/.env                # Rust config
└── ts/.env                  # TypeScript config
```

**Migration Steps:**

1. **Backup existing config:**
```powershell
Copy-Item .env .env.backup
Copy-Item deployment\docker\.env deployment\docker\.env.backup
```

2. **Create new structure:**
```powershell
Copy-Item .env.shared.example .env.shared
Copy-Item .env.example .env
Copy-Item go\.env.example go\.env
Copy-Item rust\.env.example rust\.env
Copy-Item ts\.env.example ts\.env
```

3. **Migrate values from old .env:**

Extract infrastructure values → `.env.shared`:
```bash
# From old .env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=trading1234
NATS_URL=nats://localhost:4222
# ... copy to .env.shared
```

Extract secrets → new `.env`:
```bash
# From old .env
JUPITER_API_KEY=your-key
EMAIL_USER=your-email
# ... copy to .env
```

Extract Go config → `go/.env`:
```bash
# From old .env
QUOTERS_ENABLED=true
OKX_API_KEY=your-key
# ... copy to go/.env
```

Extract TypeScript config → `ts/.env`:
```bash
# From old .env
ARB_MIN_PROFIT_BPS=50
JITO_TIP=100000
# ... copy to ts/.env
```

4. **Update docker-compose.yml:**

```yaml
# Old (single env file)
services:
  quote-service:
    env_file:
      - ../../.env

# New (multiple env files)
services:
  quote-service:
    env_file:
      - ../../.env.shared
      - ../../go/.env
      - ../../.env
```

5. **Test migration:**
```powershell
cd deployment\docker
docker-compose config  # Validate syntax
docker-compose up -d   # Start services
docker-compose logs -f # Check for errors
```

6. **Delete old files (after verification):**
```powershell
rm .env.backup
rm deployment\docker\.env
```

---

## Summary

### Quick Reference

| File | Purpose | Used By | Size |
|------|---------|---------|------|
| `.env.shared` | Infrastructure (NATS, Redis, Postgres, observability) | All services | ~80 lines |
| `.env` | Application secrets (API keys, email, wallet keys) | Services needing secrets | ~40 lines |
| `go/.env` | Quote service, external quoters, rate limits | Go services | ~175 lines |
| `rust/.env` | RPC proxy ports, Shredstream config | Rust services | ~15 lines |
| `ts/.env` | Scanners, planners, executors config | TypeScript services | ~50 lines |

### Key Takeaways

✅ **DO:**
- Use distributed config for service independence
- Keep secrets in `.env`, infrastructure in `.env.shared`
- Use `ENVIRONMENT=production` for all services
- Document all required variables in `.env.example` files
- Validate config on service startup

❌ **DON'T:**
- Commit `.env` files with secrets to Git
- Put all config in one monolithic file
- Use `NODE_ENV=development` (production-only system)
- Override everything in `docker-compose.yml` `environment:` section
- Duplicate infrastructure config across services

### Support

For issues or questions:
1. Check this documentation
2. Review CLAUDE.md Configuration Architecture section
3. Check service-specific `<stack>/CLAUDE.md` files
4. Inspect `.env.example` files for variable documentation
5. Use `docker-compose config` to debug Docker Compose loading

---

**Document Version:** 1.0
**Last Updated:** 2025-12-27
**Maintained By:** Solution Architect
