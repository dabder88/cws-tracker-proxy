# CWS Tracker Proxy - Cloudflare Worker

Cloudflare Worker that proxies Chrome Web Store requests for the CWS Tracker extension.

Chrome blocks extensions from accessing CWS domains directly (CORS, content script injection, scripting API all restricted). This proxy fetches CWS pages server-side and returns content to the extension.

---

## Deploy your own (one click) — recommended

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/shapito27/cws-tracker-proxy)

Deploy this proxy to **your own** free Cloudflare account (100,000 requests/day — far more than the extension needs). No CLI required.

1. **Click the button.** Cloudflare asks you to sign in (or create a free account) and to **connect GitHub or GitLab and grant repository access** — it clones this repo into *your* account so future pushes auto-redeploy.

2. **Keep every build setting at its default** and continue. For reference, here is what each field means:

   | Field | Default — keep it | Why |
   |-------|-------------------|-----|
   | Environment | `production` | Just a label the Worker reads. |
   | Build command | *(empty)* | No build step — Wrangler bundles on deploy. |
   | Deploy command | `npm run deploy` | Runs `wrangler deploy`. |
   | Non-production branch deploy command | `npx wrangler versions upload` | Only used for other branches — ignore it. |
   | Path | `/` | The repo root *is* the Worker. |
   | API token | *(pre-filled)* | Cloudflare auto-creates this token scoped to **your own** account to run the deploy. It's yours, not the repo author's — keep it. |

3. **Set your API key.** When prompted for the `API_KEYS` secret, enter a long random string you invent (e.g. run `openssl rand -hex 32`). This is a shared password between your proxy and your extension — **save it.**

4. **Wait for the deploy to finish**, then copy your Worker URL — it looks like `https://cws-tracker-proxy.<your-subdomain>.workers.dev`.

5. **Configure the extension** → open **Settings** and paste:
   - **Proxy URL** = your Worker URL
   - **Proxy API Key** = the same `API_KEYS` value from step 3

Run a scan — rankings should come back through your own proxy. Prefer the CLI? See **Manual deployment** below.

---

## Prerequisites

- A **Cloudflare account** (free plan is sufficient)
- **Node.js** 18+ installed locally
- **npm** installed

---

## Manual deployment (advanced)

Prefer the CLI over the one-click button above? Follow these steps.

### Step 1: Create a Cloudflare Account

1. Go to [https://dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up)
2. Sign up with email and password
3. No domain or credit card is required for Workers free tier

**Free tier limits:**
- 100,000 requests per day
- 10ms CPU time per request
- Unlimited Workers

This is more than enough for the CWS Tracker extension.

### Step 2: Install Dependencies

```bash
cd proxy
npm install
```

This installs:
- **wrangler** - Cloudflare's CLI tool for developing, testing, and deploying Workers
- **@cloudflare/workers-types** - TypeScript types for the Workers runtime
- **vitest** + **@cloudflare/vitest-pool-workers** - Testing framework

### Step 3: Authenticate Wrangler with Cloudflare

```bash
npx wrangler login
```

This opens your browser and asks you to authorize Wrangler to access your Cloudflare account. Click **Allow**.

After authorization, Wrangler stores a token locally at `~/.wrangler/config/default.toml`. You won't need to login again on this machine.

To verify it worked:

```bash
npx wrangler whoami
```

This should print your Cloudflare account name and ID.

### Step 4: Generate Your API Key

The proxy requires an API key to prevent unauthorized use. **You create this key yourself** - it's not provided by Cloudflare. It's a shared secret between your extension and your proxy.

Generate a random key:

```bash
# Option A: Using openssl
openssl rand -hex 32

# Option B: Using Node.js
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# Option C: Using Python
python3 -c "import secrets; print(secrets.token_hex(32))"
```

This produces something like:
```
a3f8b2c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1
```

**Save this key.** You'll need it in two places:
1. As a secret in Cloudflare Workers (Step 5)
2. In the extension's settings as the proxy API key (later)

### Step 5: Set the API Keys Secret

Secrets are encrypted environment variables that your Worker can read but that never appear in your code or dashboard in plain text.

```bash
npx wrangler secret put API_KEYS
```

When prompted, paste your API key and press Enter. If you want multiple keys (e.g., one for development, one for production), separate them with commas:

```
a3f8b2c4d5e6f7...key1,b4c9d0e1f2a3...key2
```

### Step 6: Deploy

```bash
npm run deploy
```

Wrangler will:
1. Bundle your TypeScript code
2. Upload it to Cloudflare's edge network
3. Print the URL of your deployed Worker

Output looks like:
```
Published cws-tracker-proxy (X.XX sec)
  https://cws-tracker-proxy.<your-subdomain>.workers.dev
```

**Save this URL.** This is your proxy endpoint.

### Step 7: Verify Deployment

Test the health endpoint (no auth needed):

```bash
curl https://cws-tracker-proxy.<your-subdomain>.workers.dev/health
```

Expected response:
```json
{"status":"ok","timestamp":"2026-02-05T...","version":"0.1.0"}
```

Test fetching an extension (auth required):

```bash
curl -H "X-API-Key: YOUR_API_KEY_HERE" \
  "https://cws-tracker-proxy.<your-subdomain>.workers.dev/detail?id=cjpalhdlnbpafiamejdnhcphjbkeiagm"
```

Expected: JSON response with `html` field containing the CWS page HTML.

---

## Local Development

### Run locally

```bash
npm run dev
```

Starts a local dev server at `http://localhost:8787`. For local dev, copy `.dev.vars.example` to `.dev.vars` and set your key there — `.dev.vars` is gitignored, so it never gets committed:

```
# .dev.vars
API_KEYS=dev-test-key
```

Wrangler loads `.dev.vars` automatically in `npm run dev`. For production the key is a Worker secret (`npx wrangler secret put API_KEYS`) or is entered during the one-click deploy — never put it in `wrangler.toml`.

### Run tests

```bash
npm test
```

24 tests covering auth, CORS, validation, routing, and error handling.

### Test locally with curl

```bash
# Health check
curl http://localhost:8787/health

# Fetch extension detail
curl -H "X-API-Key: dev-test-key" \
  "http://localhost:8787/detail?id=cjpalhdlnbpafiamejdnhcphjbkeiagm"

# Fetch search results
curl -H "X-API-Key: dev-test-key" \
  "http://localhost:8787/search?q=ad+blocker"

# Fetch with locale
curl -H "X-API-Key: dev-test-key" \
  "http://localhost:8787/detail?id=cjpalhdlnbpafiamejdnhcphjbkeiagm&hl=ja"
```

---

## Self-hosting without Cloudflare

You don't need a Cloudflare account to run this proxy. The Worker is built almost entirely on web-standard APIs (`fetch`, `Request`/`Response`, `URL`, `AbortController`); the only Cloudflare-specific dependency is the Cache API (`caches.default`), and caching is a best-effort optimization, not required for correctness.

Three ways to run it off Cloudflare:

| Option | Code changes | Caching | Best for | Production-grade |
|--------|-------------|---------|----------|------------------|
| **A. `workerd`** | none | off (needs cache stub) | faithful self-host, same runtime | ✅ |
| **B. `wrangler dev`** | none | in-memory (works) | quick personal use / testing | ⚠️ dev server |
| **C. Node / Deno** | one entry file | in-memory polyfill | a normal server process you operate | ✅ |

> **Two things every self-host needs that Cloudflare gives you for free:**
> 1. **HTTPS.** The extension calls the proxy over `https://`. Run a reverse proxy in front of the HTTP port — [Caddy](https://caddyserver.com) is easiest (automatic Let's Encrypt TLS). Plain `http://` works for `localhost` testing only.
> 2. **A process manager** (systemd, pm2, Docker) so it restarts on crash/reboot.
>
> One upside of single-process self-hosting: the in-memory **rate limiter and cache actually persist** across requests, instead of resetting per-isolate like on Cloudflare's edge.

### Option A — `workerd` (Cloudflare's open-source runtime)

[`workerd`](https://github.com/cloudflare/workerd) is the same runtime that powers Cloudflare Workers, released as a standalone open-source binary (Apache-2.0). It runs the **exact same `src/index.ts`** — no source changes.

```bash
# 1. Bundle the TypeScript Worker to a single JS module
npx wrangler deploy --dry-run --outdir dist     # -> dist/index.js

# 2. Install the runtime
npm install -D workerd
```

The Worker uses the Cache API (`caches.default`), and **standalone `workerd` throws `No Cache was configured` unless you give it a cache backend** — so you must wire one up or every authed request 500s. The simplest backend is a tiny "null cache": reads always miss, writes are discarded (caching is effectively off, but the Worker runs correctly). Create `nullcache.js`:

```js
// Minimal Cache API backend: every read misses, every write is discarded.
export default { async fetch() { return new Response(null, { status: 504 }); } };
```

Create `workerd.capnp` (keep it in the same directory as `nullcache.js` and `dist/` — `embed` paths resolve relative to the `.capnp` file):

```capnp
using Workerd = import "/workerd/workerd.capnp";

const config :Workerd.Config = (
  services = [
    (name = "main",      worker = .proxy),
    (name = "nullcache", worker = .nullcache),
  ],
  sockets = [ (name = "http", address = "*:8787", http = (), service = "main") ],
);

const proxy :Workerd.Worker = (
  modules = [ (name = "index.js", esModule = embed "dist/index.js") ],
  bindings = [
    (name = "API_KEYS",    fromEnvironment = "API_KEYS"),
    (name = "ENVIRONMENT", text = "production"),
  ],
  cacheApiOutbound   = "nullcache",          # required, else caches.default throws
  compatibilityDate  = "2024-12-30",
  compatibilityFlags = ["nodejs_compat"],
);

const nullcache :Workerd.Worker = (
  modules = [ (name = "nullcache.js", esModule = embed "nullcache.js") ],
  compatibilityDate = "2024-12-30",
);
```

Run it:

```bash
API_KEYS="$(openssl rand -hex 32)" npx workerd serve workerd.capnp
```

Outbound `fetch` to Chrome Web Store works out of the box. Want real caching instead of the stub? Point `cacheApiOutbound` at a proper cache backend — see the [workerd config samples](https://github.com/cloudflare/workerd/tree/main/samples).

### Option B — `wrangler dev` as a server (quickest)

The same dev server you use locally can listen on all interfaces. Zero code changes, and caching works (Miniflare implements the Cache API in memory).

```bash
# Put your key in .dev.vars (gitignored):  API_KEYS=your-long-random-secret
npx wrangler dev --ip 0.0.0.0 --port 8787
```

Convenient for personal use, but `wrangler dev` is a **development** server — no graceful restarts, not hardened for hostile traffic. Keep it behind Caddy/nginx + a firewall, or prefer Option A/C for anything serious.

### Option C — Node.js (or Deno) process

For a conventional server you already operate. The Worker's handler is already `(request, env, ctx) => Response`, so you only add a thin entry file that supplies `env` from `process.env`, a `ctx` shim, and a tiny `caches.default` polyfill (the one API Node lacks).

```bash
npm install @hono/node-server tsx   # web-standard HTTP server + TypeScript runner (Node 18+)
```

Create `server.node.ts`:

```ts
import { serve } from '@hono/node-server';
import worker from './src/index';

// Workers' `caches.default` isn't in Node — minimal in-memory polyfill.
// (Ignores TTL; swap for an LRU/Redis if you want real expiry.)
const store = new Map<string, Response>();
(globalThis as any).caches = {
  default: {
    async match(req: Request | string) {
      const hit = store.get(typeof req === 'string' ? req : req.url);
      return hit ? hit.clone() : undefined;
    },
    async put(req: Request | string, res: Response) {
      store.set(typeof req === 'string' ? req : req.url, res.clone());
    },
  },
};

const env = {
  API_KEYS: process.env.API_KEYS ?? '',
  ENVIRONMENT: process.env.ENVIRONMENT ?? 'production',
};
const ctx = {
  waitUntil: (p: Promise<unknown>) => void Promise.resolve(p).catch(() => {}),
  passThroughOnException() {},
};

const port = Number(process.env.PORT ?? 8787);
serve({ fetch: (request) => worker.fetch(request, env as any, ctx as any), port });
console.log(`CWS proxy listening on http://localhost:${port}`);
```

Run it (no build step — `tsx` runs TypeScript directly):

```bash
API_KEYS="$(openssl rand -hex 32)" npx tsx server.node.ts
```

**Deno** works the same way: keep the same polyfill and `env`/`ctx` shims, then `Deno.serve(req => worker.fetch(req, env, ctx))` and run with `deno run --allow-net --allow-env`.

Everything else — CWS fetching, batchexecute pagination, the in-memory rate limiter, CORS, auth, input validation — is runtime-agnostic and runs unchanged.

Whichever option you pick, point the extension's **Proxy URL** at your server's public HTTPS address and set **Proxy API Key** to your `API_KEYS` value, exactly as in the Cloudflare setup.

---

## API Reference

### Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/detail?id={extensionId}&hl={locale}` | Required | Fetch extension detail page |
| GET | `/search?q={query}&hl={locale}` | Required | Fetch search results page |
| GET | `/autocomplete?q={query}&hl={locale}` | Required | Fetch search autocomplete suggestions |
| GET | `/reviews?id={extensionId}&limit={n}&hl={locale}` | Required | Fetch up to `limit` newest reviews (1–500, default 50) |
| GET | `/health` | None | Health check |

### Authentication

Pass API key via **either** method:

```
# Header (recommended for extension)
X-API-Key: your-api-key

# Query parameter (useful for quick testing)
/detail?id=...&key=your-api-key
```

### Response Format

Successful response (200):

```json
{
  "url": "https://chromewebstore.google.com/detail/cjpalhdlnbpafiamejdnhcphjbkeiagm?hl=en",
  "status": 200,
  "html": "<!doctype html>...",
  "htmlLength": 650000,
  "fetchedAt": "2026-02-05T03:00:00.000Z"
}
```

Error response:

```json
{
  "error": "Description of what went wrong"
}
```

### Parameters

**`/detail` endpoint:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `id` | Yes | - | Extension ID (32 lowercase letters) |
| `hl` | No | `en` | Locale code (e.g., `ja`, `es`, `zh-CN`) |

**`/search` endpoint:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `q` | Yes | - | Search query (max 200 chars) |
| `hl` | No | `en` | Locale code |

### Error Codes

| Status | Meaning | Example |
|--------|---------|---------|
| 400 | Bad request | Missing `id`, invalid extension ID format, query too long |
| 401 | No API key | Missing `X-API-Key` header and `key` param |
| 403 | Bad API key | Key not in the configured `API_KEYS` list |
| 404 | Bad route | Called an endpoint that doesn't exist |
| 405 | Bad method | Used POST/PUT/etc instead of GET |
| 429 | Rate limited | Exceeded 30 requests/minute for this API key |
| 502 | CWS error | Proxy couldn't fetch from Chrome Web Store |
| 504 | Timeout | CWS didn't respond within 15 seconds |

---

## How it Works

```
Extension                    Cloudflare Worker              Chrome Web Store
   |                              |                              |
   |  GET /detail?id=xxx&key=yyy  |                              |
   |----------------------------->|                              |
   |                              |  GET /detail/xxx?hl=en       |
   |                              |----------------------------->|
   |                              |                              |
   |                              |  200 OK (HTML page)          |
   |                              |<-----------------------------|
   |                              |                              |
   |  200 OK { html: "..." }      |                              |
   |<-----------------------------|                              |
```

1. Extension sends request to proxy with API key
2. Proxy validates API key and checks rate limits
3. Proxy fetches the CWS page server-side (no CORS issues)
4. Proxy wraps the HTML in JSON and returns it with CORS headers
5. Extension parses the HTML to extract extension data

### Security

- **API key auth**: Prevents unauthorized use of your proxy
- **CORS restriction**: Only `chrome-extension://` origins and `localhost` can call the proxy
- **Rate limiting**: 30 req/min per API key (in-memory sliding window)
- **Caching**: 5-minute TTL reduces load on CWS and speeds up responses
- **Input validation**: Extension IDs must be exactly 32 lowercase letters

### Caching

Successful responses are cached for 5 minutes using Cloudflare's Cache API. The `X-Cache` header indicates cache status:
- `X-Cache: HIT` - Served from cache
- `X-Cache: MISS` - Fresh fetch from CWS

---

## Cloudflare Dashboard Management

After deployment, you can manage the Worker at:

```
https://dash.cloudflare.com > Workers & Pages > cws-tracker-proxy
```

From the dashboard you can:
- View request analytics (requests/day, errors, latency)
- View real-time logs (Logs tab > Begin log stream)
- Update environment variables and secrets (Settings > Variables)
- Set up custom domains (Settings > Triggers > Custom Domains)
- Configure additional rate limiting rules (Security > WAF)

### Custom Domain (Optional)

By default your Worker is at `cws-tracker-proxy.<subdomain>.workers.dev`. To use a custom domain:

1. Add a domain to Cloudflare (free plan works)
2. Go to Worker > Settings > Triggers > Custom Domains
3. Add your domain (e.g., `api.cws-tracker.com`)

---

## Updating the Worker

After making code changes:

```bash
npm test           # Verify tests pass
npm run deploy     # Deploy updated code
```

The update is instant and zero-downtime.

---

## Cost

**Free tier covers all expected usage:**

| Limit | Free Tier | Expected Usage |
|-------|-----------|----------------|
| Requests/day | 100,000 | ~100-500 (10 extensions x 20 keywords x daily scan + retries) |
| CPU time/request | 10ms | ~2-5ms (simple fetch relay) |
| Workers | Unlimited | 1 |

You would need the paid plan ($5/month) only if you exceed 100K requests/day, which would require ~200 active users running full scans simultaneously.
