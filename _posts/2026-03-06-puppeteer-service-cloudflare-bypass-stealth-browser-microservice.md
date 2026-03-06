---
layout: single
title: "Puppeteer Service: A Stealth Browser Microservice for Bot-Protected APIs"
date: 2026-03-06
permalink: /posts/2026/03/puppeteer-service-cloudflare-bypass-stealth-browser-microservice/
categories:
  - blog
tags:
  - solana
  - trading
  - infrastructure
  - typescript
  - architecture
  - puppeteer
  - microservice
excerpt: "How we built a headless Chromium microservice with puppeteer-extra-plugin-stealth and residential proxy support to reliably fetch data from bot-protected DeFi APIs — and the non-obvious design decisions that made it work."
---

## Disclaimer

The techniques described in this post are shared purely for **technical education and personal research purposes**. Understanding how browser fingerprinting and bot protection work is valuable knowledge for any developer building systems that interact with web APIs.

If you are using these techniques commercially or in production against third-party services, you are responsible for reviewing and complying with those services' **Terms of Service**. Many APIs explicitly prohibit automated access, scraping, or circumventing rate limits and bot protection — regardless of the technical method used. This post does not encourage or endorse violating any website's Terms of Service.

Use this knowledge responsibly.

---

## TL;DR

- Built a small TypeScript HTTP microservice (`puppeteer-service`) that routes any URL fetch through a stealth headless Chromium browser
- Solves bot protection that blocks all conventional HTTP clients
- Uses `puppeteer-extra-plugin-stealth` to defeat browser fingerprinting, and CDP `page.authenticate()` to supply residential proxy credentials (the only method Chrome actually honours)
- Supports both GET and POST methods via a clean `POST /fetch` JSON API
- Deploys as a Docker sidecar — any service in the stack can delegate a protected fetch to it

---

## The Problem: Bot Protection vs. Direct HTTP Clients

Many DeFi data APIs sit behind bot protection systems that run JavaScript fingerprinting challenges. These systems detect:

- The `navigator.webdriver` flag set by Selenium/Puppeteer
- Missing browser plugins expected in a normal user session
- Canvas and WebGL fingerprint anomalies typical of headless environments
- IP reputation (datacenter IPs are flagged; residential IPs are allowed)

A plain `fetch()` or `axios` call from a Node.js process gets blocked immediately. Even off-the-shelf browser automation tools fail when combined with proxy services, because Chrome's `--proxy-server` flag silently ignores embedded `user:pass@` credentials in newer versions — producing `ERR_NO_SUPPORTED_PROXIES` with no useful error message.

We needed a reliable, reusable solution that any service in the trading stack could delegate to.

---

## Architecture

The service is a minimal Node.js HTTP server. It accepts a `POST /fetch` request describing what to fetch, launches a fresh Chromium instance to fetch it, and returns the parsed JSON response.

```
Caller (Go / TypeScript / curl)
        │
        │  POST /fetch  { url, method, headers, body }
        ▼
┌─────────────────────────────────────────┐
│          puppeteer-service :3001        │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  Chromium (headless, stealth)   │   │
│  │                                 │   │
│  │  page.authenticate(user, pass)  │   │  ← CDP credential injection
│  │  page.setExtraHTTPHeaders(...)  │   │  ← Custom headers per request
│  │                                 │   │
│  │  GET  → page.goto(url)          │   │
│  │  POST → page.evaluate(fetch())  │   │
│  └──────────────┬──────────────────┘   │
│                 │                       │
└─────────────────┼───────────────────────┘
                  │  via residential proxy
                  ▼
        Protected API endpoint
```

The caller never deals with browsers, proxies, or bot protection. It just calls `POST /fetch`.

---

## Dependencies

```json
{
  "dependencies": {
    "dotenv": "^16.4.5",
    "puppeteer": "^22.6.0",
    "puppeteer-extra": "^3.3.6",
    "puppeteer-extra-plugin-stealth": "^2.11.2"
  }
}
```

The key libraries:

- **`puppeteer`** — controls headless Chromium via the Chrome DevTools Protocol (CDP)
- **`puppeteer-extra`** — a wrapper that supports plugins
- **`puppeteer-extra-plugin-stealth`** — patches all known browser fingerprinting detection vectors so the headless browser is indistinguishable from a real user session

---

## Core Implementation

### Setup: Stealth Plugin and Configuration

```typescript
import puppeteer from "puppeteer-extra";
import StealthPlugin from "puppeteer-extra-plugin-stealth";

puppeteer.use(StealthPlugin());

const PORT = Number(process.env.PUPPETEER_SERVICE_PORT ?? 3001);

const PROXY_HOST = process.env.PROXY_HOST ?? "";
const PROXY_PORT = Number(process.env.PROXY_PORT ?? 823);
const PROXY_USER = process.env.PROXY_USER ?? "";
const PROXY_PASS = process.env.PROXY_PASS ?? "";
const CHROME_EXECUTABLE = process.env.PUPPETEER_EXECUTABLE_PATH;

const PROXY_ENABLED = PROXY_HOST !== "";
```

Calling `puppeteer.use(StealthPlugin())` once at startup is all that's needed. Every subsequent browser launch inherits the stealth patches.

### The Request Type

```typescript
interface FetchRequest {
  url: string;
  method?: "GET" | "POST";
  headers?: Record<string, string>;
  body?: unknown;
}
```

Simple and generic — the caller describes what it wants, the service handles the browser mechanics.

### The Core Fetch Function

This is the heart of the service. A few design decisions are non-obvious and worth explaining:

```typescript
async function fetchViaProxy(req: FetchRequest): Promise<unknown> {
  const method = req.method ?? "GET";

  const launchArgs = [
    "--no-sandbox",
    "--disable-setuid-sandbox",
    "--disable-dev-shm-usage",
  ];

  if (PROXY_ENABLED) {
    launchArgs.push(`--proxy-server=http://${PROXY_HOST}:${PROXY_PORT}`);
  }

  const browser = await puppeteer.launch({
    headless: true,
    ...(CHROME_EXECUTABLE ? { executablePath: CHROME_EXECUTABLE } : {}),
    args: launchArgs,
  });

  try {
    const page = await browser.newPage();

    // Authenticate with the proxy via CDP — NOT via URL credentials
    if (PROXY_ENABLED && PROXY_USER) {
      await page.authenticate({ username: PROXY_USER, password: PROXY_PASS });
    }

    if (req.headers && Object.keys(req.headers).length > 0) {
      await page.setExtraHTTPHeaders(req.headers);
    }

    // ... GET or POST handling
  } finally {
    await browser.close();
  }
}
```

**Key insight — `page.authenticate()` vs URL credentials**: Chrome's `--proxy-server` flag accepts a hostname and port, but embedding `user:pass@host:port` credentials in the URL is silently ignored in Chrome 120+. The only reliable way to supply proxy credentials is through the CDP `Network.authenticate` method, which Puppeteer exposes as `page.authenticate()`. This was the root cause of the FlareSolverr failure we encountered.

### GET vs POST: Different Browser Strategies

The two HTTP methods need fundamentally different approaches because `page.goto()` cannot send a request body.

**GET requests** — use `page.goto()` which triggers the full page lifecycle, including any JavaScript challenge resolution:

```typescript
if (method === "GET") {
  const response = await page.goto(req.url, {
    waitUntil: "networkidle0",
    timeout: 60_000,
  });

  const status = response.status();
  if (status !== 200) {
    const body = await page.content();
    throw new Error(`HTTP ${status}: ${body.substring(0, 300)}`);
  }

  // Chrome's built-in JSON viewer renders JSON as text inside <body>
  const bodyText = await page.evaluate(
    () => (document.body as HTMLBodyElement).innerText
  );
  responseBody = JSON.parse(bodyText);
}
```

The `waitUntil: "networkidle0"` option is important — it waits until the network is idle, which gives any challenge scripts enough time to complete before we read the response.

**POST requests** — navigate to `about:blank` first (to initialise the browser context with the proxy and stealth patches), then call `fetch()` from inside the browser via `page.evaluate()`:

```typescript
} else {
  await page.goto("about:blank");

  const bodyStr =
    req.body === undefined
      ? undefined
      : typeof req.body === "string"
        ? req.body
        : JSON.stringify(req.body);

  const result = await page.evaluate(
    async (url: string, headers: Record<string, string>, body: string | undefined) => {
      const response = await fetch(url, {
        method: "POST",
        headers,
        ...(body !== undefined ? { body } : {}),
      });
      const text = await response.text();
      return { status: response.status, text };
    },
    req.url,
    (req.headers ?? {}) as Record<string, string>,
    bodyStr as string | undefined
  );

  responseBody = JSON.parse(result.text);
}
```

Executing `fetch()` inside `page.evaluate()` means the request goes out through the browser's networking stack — the residential proxy is in effect, and the stealth plugin has already patched the browser context. The caller gets a POST with a body, through the proxy, with full bot protection bypass.

### The HTTP Server

A minimal Node.js `http` server exposes two endpoints:

```typescript
const server = http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://localhost:${PORT}`);

  // Health check for Docker and dependent services
  if (url.pathname === "/health") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ status: "ok" }));
    return;
  }

  // Generic proxy endpoint
  if (url.pathname === "/fetch" && req.method === "POST") {
    const raw = await readBody(req);
    const fetchReq: FetchRequest = JSON.parse(raw);

    try {
      const data = await fetchViaProxy(fetchReq);
      res.writeHead(200, { "Content-Type": "application/json" });
      res.end(JSON.stringify(data));
    } catch (err) {
      const message = err instanceof Error ? err.message : String(err);
      res.writeHead(500, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ error: message }));
    }
    return;
  }
});

server.listen(PORT, "0.0.0.0");
```

No Express, no Fastify — just the standard `http` module. For a service with two endpoints and no middleware needs, the overhead isn't worth it.

---

## API Reference

### `GET /health`

```json
{ "status": "ok" }
```

Used by Docker's `HEALTHCHECK` and by dependent services that wait for the service to be ready before starting.

### `POST /fetch`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | `string` | Yes | Full URL to fetch |
| `method` | `"GET"` \| `"POST"` | No | Default: `"GET"` |
| `headers` | `Record<string,string>` | No | HTTP headers injected via `setExtraHTTPHeaders` |
| `body` | `string` \| `object` | No | Request body for POST; objects are JSON-serialised automatically |

**Response `200`** — parsed JSON body from the target URL

**Response `500`** — browser error or non-200 from target: `{ "error": "<message>" }`

---

## Calling the Service

### TypeScript Client

```typescript
interface PuppeteerFetchRequest {
  url: string;
  method?: "GET" | "POST";
  headers?: Record<string, string>;
  body?: unknown;
}

async function fetchViaChrome(req: PuppeteerFetchRequest): Promise<unknown> {
  const serviceURL = process.env.PUPPETEER_SERVICE_URL ?? "http://localhost:3001";

  const res = await fetch(`${serviceURL}/fetch`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(req),
  });

  if (!res.ok) {
    const err = await res.json() as { error: string };
    throw new Error(`puppeteer-service: ${err.error}`);
  }

  return res.json();
}

// Example: POST to a GraphQL API behind a protected endpoint
const data = await fetchViaChrome({
  url: "https://some-protected-api.io/graphql",
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Origin":       "https://some-protected-api.io",
    "Referer":      "https://some-protected-api.io/",
  },
  body: {
    query: "{ pools { id liquidity } }",
  },
});
```

### Go Client

```go
type PuppeteerRequest struct {
    URL     string            `json:"url"`
    Method  string            `json:"method,omitempty"`
    Headers map[string]string `json:"headers,omitempty"`
    Body    interface{}       `json:"body,omitempty"`
}

func (c *PuppeteerClient) Fetch(
    ctx context.Context,
    url, method string,
    headers map[string]string,
    body interface{},
) ([]byte, error) {
    reqBody, _ := json.Marshal(PuppeteerRequest{
        URL: url, Method: method, Headers: headers, Body: body,
    })

    resp, err := http.PostWithContext(ctx, c.serviceURL+"/fetch",
        "application/json", bytes.NewReader(reqBody))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}
```

### curl (Manual Testing)

```bash
# Health check
curl http://localhost:3001/health

# GET a bot-protected JSON endpoint
curl -s -X POST http://localhost:3001/fetch \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://protected-api.io/v2/data?id=abc123",
    "method": "GET",
    "headers": {
      "Accept": "application/json",
      "Origin": "https://protected-api.io",
      "Referer": "https://protected-api.io/"
    }
  }' | jq .

# POST with a JSON body
curl -s -X POST http://localhost:3001/fetch \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://price.jup.ag/v6/price",
    "method": "POST",
    "headers": {
      "Content-Type": "application/json",
      "Origin": "https://jup.ag",
      "Referer": "https://jup.ag/"
    },
    "body": { "ids": ["So11111111111111111111111111111111111111112"] }
  }' | jq .
```

---

## Docker Deployment

### Dockerfile

The Dockerfile has a few non-obvious choices worth documenting:

```dockerfile
FROM node:20-slim

# Chrome shared libraries for Debian Bookworm (slim)
RUN apt-get update && apt-get install -y \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libdrm2 libgbm1 \
    libxkbcommon0 libxcomposite1 libxdamage1 libxfixes3 libxrandr2 \
    libasound2 libpango-1.0-0 libcairo2 libnspr4 libnss3 \
    libx11-xcb1 libxss1 \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY ts/apps/puppeteer-service/package.json ./

# Strip devDependencies and add tsx as a runtime dep
RUN node -e " \
    const p = JSON.parse(require('fs').readFileSync('./package.json', 'utf8')); \
    delete p.devDependencies; \
    p.dependencies.tsx = '^4.19.0'; \
    require('fs').writeFileSync('./package.json', JSON.stringify(p, null, 2)); \
    "

# Install without running postinstall (pnpm v10 blocks Puppeteer's Chrome download)
RUN npm install --ignore-scripts

# Manually trigger Puppeteer's Chrome download after npm install
RUN node_modules/.bin/puppeteer browsers install chrome

COPY ts/apps/puppeteer-service/src/ ./src/

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=10s --start-period=90s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3001/health',(r)=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"

CMD ["node_modules/.bin/tsx", "src/index.ts"]
```

**Three non-obvious gotchas:**

1. **`libasound2` not `libasound2t64`** — The `node:20-slim` base image is Debian Bookworm. Ubuntu 24.04 renamed the package to `libasound2t64`; using the Ubuntu name on Debian produces a `Package not found` error that's easy to misdiagnose.

2. **`pnpm v10` blocks Puppeteer's postinstall Chrome download** — The monorepo uses pnpm v10, which enforces `onlyBuiltDependencies` restrictions. In Docker, we break out of the monorepo context entirely and use `npm install --ignore-scripts` followed by an explicit `puppeteer browsers install chrome` to download the Chrome binary in a separate step.

3. **`start_period: 90s`** — The first cold start in an environment without a pre-warmed image layer needs time for the Chrome binary download. A shorter `start_period` will cause the container to be marked unhealthy before it's actually ready.

### docker-compose

```yaml
puppeteer-service:
  build:
    context: ../../
    dockerfile: ts/apps/puppeteer-service/Dockerfile
  container_name: puppeteer-service
  ports:
    - "3001:3001"
  env_file:
    - ../../go/.env
  environment:
    - PUPPETEER_SERVICE_PORT=3001
  healthcheck:
    test: ["CMD", "node", "-e",
           "require('http').get('http://localhost:3001/health',(r)=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 90s

# Consuming service
pool-discovery-service:
  depends_on:
    - puppeteer-service
  environment:
    - PUPPETEER_SERVICE_URL=http://puppeteer-service:3001
```

---

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PUPPETEER_SERVICE_PORT` | `3001` | Port the HTTP server listens on |
| `PROXY_HOST` | `""` | Residential proxy hostname. Empty = direct connection |
| `PROXY_PORT` | `823` | Residential proxy port |
| `PROXY_USER` | `""` | Proxy username (injected via CDP, not URL) |
| `PROXY_PASS` | `""` | Proxy password |
| `PUPPETEER_EXECUTABLE_PATH` | _(bundled Chrome)_ | Override Chrome binary path |

---

## Key Design Decisions

### Per-Request Browser Launch

Each `POST /fetch` call launches a fresh browser and closes it when done. This trades throughput for correctness:

- **Clean cookie jar** — no session state leaks between requests
- **No fingerprint accumulation** — the browser starts fresh every time, which is harder to fingerprint over time
- **Isolation** — one browser crash doesn't affect other requests

The ~1–2 second launch overhead is acceptable for the use case this service was built for: data enrichment at pool discovery time, not real-time HFT. For a high-throughput scenario you could maintain a browser pool, but that adds significant complexity for no current benefit.

### Why `document.body.innerText` for GET Responses

When Chrome navigates to a URL that returns `Content-Type: application/json`, it renders a built-in JSON viewer — the raw JSON is available as `document.body.innerText`. This is simpler than intercepting the network response at the CDP level, and it works reliably whether a bot protection challenge fired or not.

### Why `puppeteer-extra-plugin-stealth`

Bot protection systems typically check for:
- `navigator.webdriver === true` (set by default in Puppeteer)
- Absence of browser plugins (headless has none)
- Canvas/WebGL fingerprint anomalies
- Chrome's `languages` property being empty

The stealth plugin patches all of these to match a real user session. Without it, the protection layer blocks the request even through a residential proxy.

### Proxy Credentials via CDP

As mentioned earlier — embedding `user:pass@host:port` in `--proxy-server` is silently broken in Chrome 120+. The CDP-based `page.authenticate({ username, password })` is the only supported path. This is easy to miss because Chrome gives no error; it just returns `ERR_NO_SUPPORTED_PROXIES` as though the proxy itself is broken.

---

## Onboarding a New Protected API

When adding a new bot-protected data source:

1. **Identify the correct headers** — open browser DevTools, filter Network by XHR/Fetch, find the real API call, copy `Origin`, `Referer`, `Accept`, and any auth headers.

2. **Test with curl first**:
   ```bash
   curl -s -X POST http://localhost:3001/fetch \
     -H "Content-Type: application/json" \
     -d '{"url":"<API_URL>","headers":{"Origin":"<origin>","Referer":"<referer>"}}'
   ```

3. **Add a typed wrapper** in the consuming service so callers never deal with raw header maps.

4. **Update docker-compose** — add `depends_on: puppeteer-service` and the service URL env var.

---

## Error Handling

| HTTP Status | Meaning | Common Cause |
|-------------|---------|--------------|
| `200` | Success | Parsed JSON returned |
| `400` | Bad request | Missing `url` field or malformed JSON |
| `404` | Not found | Wrong path |
| `500` | Proxy error | Protected API 403/429, JSON parse failure, browser crash |

Transient `403` responses do occur under concurrent load — the bot protection challenge occasionally slips through. The consuming service should treat these as retriable failures and handle them at the retry/enrichment layer rather than in the puppeteer-service itself. An ~89% success rate under normal concurrent load is typical and acceptable.

---

## Impact and Next Steps

### What This Service Enables

- Any service in the stack can fetch data from bot-protected APIs without implementing browser automation itself
- The complexity of proxy credential injection, stealth patching, and GET/POST strategy is encapsulated in one ~230-line file
- Adding a new protected data source is a matter of identifying headers and adding a thin wrapper — no changes to the service itself

### What's Next

With the data layer infrastructure solidified, the focus shifts to strategy execution:

1. **Strategy Service** — define profit thresholds, slippage tolerance, and trade sizing
2. **Executor Service** — Jito bundle submission for MEV-protected trade execution
3. **Paper Trading** — full pipeline validation without real capital
4. **Live Trading** — controlled small-capital runs to measure real-world execution quality

---

## Conclusion

The puppeteer-service is a good example of the "boring infrastructure" that makes complex systems work. Bot protection is a real operational challenge for any system that depends on DeFi data APIs, and trying to fight it with conventional HTTP clients is a losing battle. A stealth headless browser with residential proxy support, packaged as a simple HTTP sidecar, is the right level of complexity: it solves the problem cleanly, stays reusable, and keeps the details out of every consumer.

The three non-obvious lessons that cost the most debugging time: `page.authenticate()` over URL credentials, `libasound2` vs `libasound2t64`, and the pnpm postinstall block. Hopefully this post saves someone else the same discovery.

---

## Related Posts

- [Quote Service Production Validation: Zero Critical Deviations & a Chinese New Year Milestone](/posts/2026/02/quote-service-production-validation-chinese-new-year/) - Production validation of the quote layer
- [Pool Discovery Service: Architecture and Implementation](/posts/2025/12/pool-discovery-service-rpc-proxy-architecture/) - The pool discovery service that consumes puppeteer-service
- [Project Milestone Complete: Infrastructure Ready for Arbitrage Phase](/posts/2026/01/project-milestone-complete-infrastructure-ready-for-arbitrage-phase/) - Overall project status

---

## Technical Documentation

- [Puppeteer Service README](https://github.com/guidebee/solana-trading-system/blob/master/ts/apps/puppeteer-service/docs/README.md)
- [Integration Guide](https://github.com/guidebee/solana-trading-system/blob/master/ts/apps/puppeteer-service/docs/usage.md)
- [Project Status and Feasibility](https://github.com/guidebee/solana-trading-system/blob/master/docs/08-PROJECT-STATUS-AND-FEASIBILITY.md)

---

## Connect

- **GitHub**: [guidebee/solana-trading-system](https://github.com/guidebee/solana-trading-system)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

*This is post #26 in the Solana Trading System development series. A deep dive into the puppeteer-service: a stealth headless Chromium microservice that enables any service in the trading stack to fetch data from bot-protected DeFi APIs through a clean HTTP sidecar.*
