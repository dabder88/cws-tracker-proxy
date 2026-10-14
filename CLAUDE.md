# CLAUDE.md - CWS Tracker Proxy

Cloudflare Worker that fetches Chrome Web Store pages on behalf of the extension, bypassing Chrome's restriction on extension access to CWS domains.

**This is a separate npm package** with its own `package.json`, `tsconfig.json`, and test config. It shares NO code with the main extension.

## Architecture

Single-file Worker (`src/index.ts`) handling all routing, auth, rate limiting, caching, and CWS fetching. No framework - plain Request/Response handlers.

### Endpoints

| Endpoint | Purpose | Key Params |
|----------|---------|------------|
| `GET /detail` | Fetch extension detail page | `id` (32-char CWS ID), `hl` (locale) |
| `GET /search` | Fetch search results (page 1 + pagination) | `q` (query), `hl` (locale), `token` (pagination cursor, optional) |
| `GET /autocomplete` | Fetch search suggestions | `q` (query), `hl` (locale) |
| `GET /reviews` | Fetch extension reviews (up to `limit` newest, one call) | `id` (32-char CWS ID), `limit` (1–500, default 50), `hl` (locale) |
| `GET /health` | Health check (no auth) | none |

### Security

- **Auth:** API key required via `X-API-Key` header or `?key=` query param. `API_KEYS` is a **secret** (comma-separated list), NOT a committed `wrangler.toml` var. Set prod with `npx wrangler secret put API_KEYS`; set local dev via `.dev.vars`. `/health` is the only unauthenticated endpoint.
- **CORS:** Restricted to `chrome-extension://` origins (plus `http://localhost` for dev).
- **Rate limiting:** 30 requests/minute per API key. In-memory sliding window (per-isolate, resets on cold start).

### Caching

- 5-minute TTL using the Cache API for `/detail` and page-1 `/search`.
- Session params (`bl` build label, `sid`, `at` CSRF token) for batchexecute cached for 1 hour.
- **Paginated `/search` (`?token=`), `/autocomplete`, and `/reviews` bypass the cache** — results are transient (and reviews should be fresh for change detection), and the shared CDN cache may ignore query-string differences.

### Pagination

Search results beyond page 1 use the CWS `batchexecute` RPC endpoint (no URL-based paging). The Worker extracts session params (build label, session ID, CSRF token via `extractSessionParams`) from the initial search-page HTML, then POSTs a batchexecute request with the `zTyKYc` RPC method. Page size is 10 results. `/autocomplete` uses the same endpoint with the `QcU9bc` method (needs only `bl`, no `sid`/`at`). Responses are wrapped in synthetic `AF_initDataCallback` HTML so the extension's page-1 parser handles them unchanged.

**Reviews** use the `x1DgCd` RPC method (discovered via the reviews page's `AF_dataServiceRequests` map). Unlike search, there is **no token/offset pagination** — the request payload `[extensionId, [pageSize], 2, null, null, null, 0]` takes the desired page size in slot `[1]`, and CWS returns up to that many reviews in a **single** call (verified to 500). So `/reviews?limit=N` is one POST with `[N]` as the page size (clamped 1–500, default 50); no `bl`/`sid`/`at` needed (public read-only, like autocomplete). The RPC payload is the same `[[token], [reviews…], textReviewCount]` array the extension's `reviews-v1` parser consumes, returned as `{ data }`.

## Tech Stack

- **Runtime:** Cloudflare Workers (V8 isolates, NOT Node.js). Only Cloudflare-specific API used is `caches.default`; everything else is web-standard, so it also self-hosts on `workerd`, `wrangler dev`, or a Node/Deno process — see README "Self-hosting without Cloudflare".
- **Deploy:** Wrangler CLI (always use `npx wrangler`, never bare `wrangler`)
- **TypeScript:** Strict mode, ES2022, `@cloudflare/workers-types`
- **Testing:** Vitest with `@cloudflare/vitest-pool-workers` (runs tests inside Workers runtime via Miniflare)

## Commands

```bash
cp .dev.vars.example .dev.vars   # First-time local setup: holds API_KEYS secret (gitignored)
npm test              # Run all tests (auth, CORS, validation, routing, detail/search/autocomplete/reviews, caching)
npm run test:watch    # Watch mode
npx wrangler dev      # Local dev server on port 8787 (auto-loads .dev.vars)
npx wrangler deploy   # Deploy to production
```

## Testing

Tests run inside a real Workers runtime (Miniflare), not Node.js. The `vitest.config.mts` injects test bindings:
- `API_KEYS`: `'test-key-1,test-key-2'`
- `ENVIRONMENT`: `'test'`

Tests use `SELF.fetch()` to make requests to the Worker (Workers pool auto-binds it).

## Key Constants

```
CWS_FETCH_TIMEOUT_MS    = 15,000ms
RATE_LIMIT_MAX          = 30 req/min/key
CACHE_TTL               = 300s (5 min)
BUILD_LABEL_CACHE_TTL   = 3,600s (1 hour)   # session params
EXTENSION_ID_REGEX      = /^[a-z]{32}$/
MAX_SEARCH_QUERY        = 200 chars
SEARCH_PAGE_SIZE        = 10
SEARCH_RPC_METHOD       = zTyKYc            # /search pagination
AUTOCOMPLETE_RPC_METHOD = QcU9bc            # /autocomplete
REVIEWS_RPC_METHOD      = x1DgCd            # /reviews
REVIEWS_DEFAULT_LIMIT   = 50               # /reviews page size default
REVIEWS_MAX_LIMIT       = 500              # /reviews page size cap
```

## Rules

- No shared imports from `src/` of the main extension. This is a standalone package.
- `API_KEYS` is a secret — never write it into `wrangler.toml` or any committed file. Use `wrangler secret` (prod) or `.dev.vars` (local, gitignored).
- Always validate extension IDs with the 32-char lowercase regex before forwarding to CWS.
- CWS fetches have a 15s timeout via `AbortController`.
- User-Agent header is spoofed to look like a real browser.
- Never expose internal error details to clients - return generic error messages.
- Rate limit is per-isolate (in-memory Map). It resets on cold starts - this is acceptable.
