# Bypassing Network Restrictions with a Cloudflare Worker Reverse Proxy

Most institutional networks (schools, workplaces) block content using one of two mechanisms:

- **DNS/IP blocklists** — known game/entertainment domains are blocked at the network level
- **SSL inspection** — a MITM proxy with a trusted CA cert decrypts HTTPS traffic and inspects URLs in plaintext

A Cloudflare Worker reverse proxy defeats the first completely and significantly hardens against the second. This document explains the architecture, the evasion design decisions, and how to deploy your own.

---

## How It Works

```
Browser → https://your-worker.dev/proxy/{b64(host)}/path → Cloudflare Edge → Origin
        ←                          + CORS headers        ←                 ← HTML/assets
```

The front-end never requests blocked domains directly. All requests go to your Worker's hostname (a Cloudflare edge node). The Worker decodes the target host from the path, fetches the resource from the real origin server-side, strips framing/CSP headers that would prevent iframe embedding, injects CORS headers, and streams the response back.

Because the Worker runs at Cloudflare's edge, the origin sees a Cloudflare IP — not the client's. The client only ever talks to your Worker hostname.

---

## Why This Gets Past Filters

### DNS/IP blocking
The browser makes no DNS lookup for the blocked domain. Every TCP connection is to Cloudflare's IP range. Most filters have no grounds to block `*.workers.dev` or your custom domain since they're legitimate Cloudflare infrastructure used by thousands of real services.

### SSL inspection
Even on networks that perform TLS MITM, the inspector only sees:

```
GET /proxy/d2F0Y2hkb2N1bWVudGFyaWVzLmNvbQ/wp-content/uploads/games/2048/ HTTP/2
Host: static-assets.r3ynard.workers.dev
```

The hostname (`watchdocumentaries.com` in this example) is base64url-encoded in the path. A filter would need to actively decode and re-check every path segment to identify the target — most don't. The path looks like a CDN asset token.

### Why only the hostname is encoded (not the full URL)
Encoding the full URL into one opaque blob breaks relative URL resolution inside the proxied page. When a game page at `/proxy/{blob}` loads `js/game.js` relatively, the browser resolves it as `/proxy/js/game.js` — which the Worker can't decode. By encoding only the hostname and keeping the path verbatim, sub-resources resolve to `/proxy/{b64host}/js/game.js`, which the Worker can correctly route.

### What doesn't get hidden
On networks with SSL inspection and aggressive path scanning, a determined admin can decode base64 paths. The real protection here is that most institutional filters are signature-based, not forensic. The custom domain advice below is the stronger mitigation for aggressive environments.

---

## URL Format

| Original | Proxied |
|---|---|
| `https://watchdocumentaries.com/wp-content/uploads/games/2048/` | `https://your-worker/proxy/d2F0Y2hkb2N1bWVudGFyaWVzLmNvbQ/wp-content/uploads/games/2048/` |

`d2F0Y2hkb2N1bWVudGFyaWVzLmNvbQ` is `watchdocumentaries.com` in base64url (no padding).

Encoding in JavaScript (build-time, client, or Worker):
```js
const b64Host = btoa(host).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
```

Decoding in the Worker:
```js
function decodeB64(encoded) {
  const b64 = encoded.replace(/-/g, '+').replace(/_/g, '/');
  const padded = b64 + '='.repeat((4 - b64.length % 4) % 4);
  return atob(padded);
}
```

---

## Worker Architecture

### Path routing
```
/proxy/{base64url(host)}/{path...}?{query}
```
The regex that splits these must not include `/` in the base64 character class, otherwise it greedily matches into the path:
```ts
// Correct — stops at first /
url.pathname.match(/^\/proxy\/([A-Za-z0-9_-]+)(\/.*)?$/)

// Wrong — base64 char class includes / and eats the path
url.pathname.match(/^\/proxy\/([A-Za-z0-9+/=_-]+)(\/.*)?$/)
```

### Host allowlist
Never operate an open proxy. Maintain an explicit allowlist of hostable domains:
```ts
const ALLOWED_HOSTS = new Set([
  "watchdocumentaries.com",
  "www.watchdocumentaries.com",
  // ...
]);
```
Validate after decoding the host, before fetching anything.

### Headers to strip from upstream responses
```ts
patched.headers.delete("X-Frame-Options");       // allows iframe embedding
patched.headers.delete("Content-Security-Policy"); // removes frame-ancestors restrictions
```

### Headers NOT to expose to clients
Strip any headers that fingerprint the proxy:
- Do **not** forward `X-RateLimit-*` on success responses — they reveal proxy infrastructure
- Do **not** return `{"status":"ok"}` on `GET /` — return a blank HTML page or a plausible-looking response

### WebSocket support
Cloudflare Workers support WebSocket proxying natively. Forward upgrade requests using the Workers WebSocket API:
```ts
if (request.headers.get("Upgrade")?.toLowerCase() === "websocket") {
  return await fetch(targetUrl, {
    headers: forwardHeaders(request.headers),
    webSocket: true, // CF-specific
  });
}
```
Note: WebSocket proxying does **not** work in `wrangler dev` (local). It requires actual Cloudflare edge deployment.

### Hop-by-hop headers
Strip these before forwarding to the origin:
```
connection, keep-alive, proxy-authenticate, proxy-authorization,
te, trailer, transfer-encoding, upgrade, host
```
Set `Host`, `Origin`, and `Referer` to the target origin so it responds as if directly requested.

---

## Deployment

### Prerequisites
- Free Cloudflare account — [dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up)
- Node.js 18+

### Deploy
```bash
cd game-proxy-worker
npm install
npx wrangler login   # opens browser auth first time
npm run deploy       # prints your worker URL
```

`wrangler.toml` minimum config:
```toml
name = "static-assets"   # choose a neutral name — avoid "proxy", "bypass", "game"
main = "src/index.ts"
compatibility_date = "2025-01-01"
```

### Naming matters
Your Worker name becomes the first segment of your `workers.dev` URL:
```
https://{name}.{account-subdomain}.workers.dev
```
`static-assets`, `cdn-relay`, `media-cache` all look like infrastructure. `game-proxy` does not.

---

## Custom Domain (Strongly Recommended)

`workers.dev` is categorised as "proxy/anonymizer" by Securly, GoGuardian, and Cisco Umbrella and is blanket-blocked on many managed networks. A custom domain on neutral hosting sidesteps this entirely.

1. Add your domain to Cloudflare (free plan) and point nameservers at Cloudflare
2. Add a route in `wrangler.toml`:

```toml
routes = [
  { pattern = "cdn.yourdomain.com/*", zone_name = "yourdomain.com" }
]
```

3. Redeploy. Update your front-end env var to `https://cdn.yourdomain.com`.

Subdomain naming: `cdn.`, `static.`, `assets.`, `media.` all blend in. Avoid anything that reads as a circumvention tool. A newly registered domain that hasn't been categorised yet will pass most filters until it's reported.

---

## Rate Limiting

In-memory rate limiting (per Worker isolate) is sufficient for abuse prevention but degrades on high-traffic deployments since isolates are not shared across all edge nodes. For stronger guarantees use Cloudflare's built-in rate limiting rules (available on free plans) or Durable Objects for global state.

```ts
const RATE_WINDOW_MS = 60_000;
const RATE_LIMIT_PER_WINDOW = 120;
```

---

## Testing

Encode a host manually and test end-to-end:
```bash
ENCODED=$(echo -n "watchdocumentaries.com" | base64 | tr '+/' '-_' | tr -d '=')
curl "https://your-worker/proxy/${ENCODED}/wp-content/uploads/games/2048/"
# Expect: 200 with game HTML
curl "https://your-worker/proxy/${ENCODED}/wp-content/uploads/games/2048/js/application.js"
# Expect: 200 with JavaScript
```

Check that blocked domains 403:
```bash
EVIL=$(echo -n "evil.com" | base64 | tr '+/' '-_' | tr -d '=')
curl "https://your-worker/proxy/${EVIL}/"
# Expect: {"error":"Forbidden: host not in allowlist or invalid path"}
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| 403 on all requests | Host not in `ALLOWED_HOSTS` after decode | Check that decoded host matches the allowlist exactly (with/without `www.`) |
| 403 + wrong host decoded | Regex eating into path | Ensure base64 char class in regex is `[A-Za-z0-9_-]` with no `/` |
| Assets 404 after HTML loads | `<base href>` pointing off-proxy | Do not inject `<base href>` — keep paths verbatim so relative URLs resolve through the proxy |
| WebSocket games freeze | Running `wrangler dev` locally | WebSocket proxy requires real Cloudflare edge; deploy to test |
| CORS errors | `ALLOWED_ORIGINS` set but origin mismatch | Check `ALLOWED_ORIGINS` env var or remove it to fall back to `*` |
| `workers.dev` blocked on network | Filter has `workers.dev` category blocked | Set up a custom domain |

---

## Limits

- **Free plan:** 100,000 requests/day, 10ms CPU time per request
- CPU limit is per-request compute, not wall time — proxying is mostly I/O so this is rarely hit
- No persistent storage without Durable Objects or KV (not needed for a pure proxy)
