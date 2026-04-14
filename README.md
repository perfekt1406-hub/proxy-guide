# Guide: Building a Cloudflare Worker Reverse Proxy

This guide explains how to build and deploy a Cloudflare Worker reverse proxy. This architecture allows you to proxy requests to an origin server through Cloudflare's edge network. This is useful for accessing resources across restricted network environments or acting as a secure intermediary for apps, IoT devices, or web clients.

## The Problem: How Wi-Fi Blocks Work

Most restrictive Wi-Fi networks (like those in schools, workplaces, or public hotspots) block content using one of two main mechanisms:

1. **DNS/IP Blocklists:** Known domains (e.g., games, social media, entertainment) are blocked at the network level. When your device tries to look up the IP address for a blocked site, the network refuses to resolve it.
2. **SSL Inspection (MITM):** The network installs a trusted certificate on your device to decrypt your HTTPS traffic. This allows the network administrators to inspect the exact URLs you are visiting in plaintext, even on secure connections.

## The Solution: How This Proxy Bypasses Filters

To get around these restrictions, we introduce a middleman: a Cloudflare Worker. Here is why this specific approach works:

- **Defeating DNS/IP Blocks:** Your client (whether it's a browser, a mobile app, or an IoT device like an Arduino) never actually tries to connect to the blocked domain. Instead, it connects directly to your Cloudflare Worker. Because Cloudflare is a massive, trusted network that hosts millions of legitimate internet services, networks rarely block Cloudflare infrastructure wholesale. 
- **Defeating SSL Inspection:** Even if the network is intercepting and reading your HTTPS traffic, they won't see the blocked domain name in the URL. We disguise the target hostname by encoding it (using base64) directly into the URL path (e.g., `/proxy/ZXhhbXBsZS5jb20/...`). Most network filters aren't sophisticated enough to actively decode and re-evaluate every single path segment; to them, it just looks like your device is requesting a harmless, random CDN asset.

### High-Level Architecture

```text
Client (Web/App/IoT) → https://your-worker-name.workers.dev/proxy/{b64(host)}/path → Cloudflare Edge → Origin
                     ←                                           + CORS headers    ←                 ← Payload/Assets
```

The client sends all requests to the Worker. The Worker decodes the true destination from the URL, fetches the data from the real origin server on your behalf, cleans up the network headers, and streams the response back to you.

---

## Implementation Guide

This guide focuses on the core concepts you need to implement. While Cloudflare Workers are written in JavaScript or TypeScript, these concepts apply to any client you might be building—from a React web app to a native iOS app, or even an Arduino written in C++.

### Step 1: Prerequisites

Before you begin, you will need:
- A free Cloudflare account — [dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up)
- Node.js 18 or higher installed on your computer to run the Cloudflare deployment tools.

### Step 2: Initialize the Project

Start by creating a new directory for your project and initializing it with Wrangler (Cloudflare's official command-line tool):

```bash
mkdir your-proxy-project
cd your-proxy-project
npm init -y
npm install -D wrangler
```

### Step 3: Configure Wrangler

Create a `wrangler.toml` file in the root of your project. This file tells Cloudflare how to deploy your code. 

**Why naming matters:** The `name` you choose in this file will become the first part of your Cloudflare URL (e.g., `your-worker-name.workers.dev`). If you are trying to bypass a filter, you should choose a neutral, boring name that looks like standard web infrastructure. Avoid words like "proxy", "bypass", or "game". Instead, use names like `static-assets`, `media-cache`, or `cdn-relay`.

```toml
name = "static-assets"
main = "src/index.ts"
compatibility_date = "2025-01-01"
```

### Step 4: Worker Architecture & Core Concepts

Create a `src` directory and an `index.ts` file inside it. Your proxy needs to handle five core responsibilities.

#### 1. URL Encoding & Decoding

**Why we do this:** We need to tell the proxy where to go, but we can't put the blocked domain in plaintext, or the network filter will catch it. Furthermore, we only encode the *hostname* (like `example.com`), rather than the whole URL. If we encoded the whole URL into one massive blob, any relative links on the target webpage (like `<img src="/logo.png">`) would break. By keeping the resource path intact, standard web routing still works flawlessly.

**How to do it:**
We use `base64url` encoding. It's just like standard base64, but it replaces `+` and `/` with URL-safe characters (`-` and `_`) so it doesn't accidentally break the URL structure.

- **On your Client (App, Website, Arduino):** Before making a request, your client must encode the target hostname. In a web app, you use `btoa()` and replace the unsafe characters. In an Arduino/C++ project, you would use a Base64 library.
- **On the Worker:** Your worker intercepts the request, extracts the base64 string from the path, and decodes it back into the original hostname to figure out where to forward the traffic.

#### 2. Path Routing

**Why we do this:** The worker needs to cleanly separate the encoded hostname from the rest of the requested file path.

**How to do it:** You will typically use a Regular Expression (Regex) to parse the incoming URL. It is crucial that your regex stops capturing characters as soon as it hits the first `/` after the base64 string. If your regex is too greedy, it might accidentally swallow part of the file path, causing the request to fail.

#### 3. Host Allowlist

**Why we do this:** Security is paramount. If your proxy forwards *any* request it receives, you have created an "open proxy." Malicious actors scan the internet for open proxies to route illegal traffic, launch attacks, or spam websites. If your worker is used for this, Cloudflare will ban your account.

**How to do it:** You must hardcode an explicit "allowlist" (a list of approved domains) into your worker. After your worker decodes the requested hostname, it must check if the host is on the allowlist. If someone tries to proxy a domain not on the list, the worker must immediately block the request and return a `403 Forbidden` error.

#### 4. Header Modification

**Why we do this:** HTTP requests include "headers" — metadata about the request. When you insert a proxy into the middle of a connection, you have to carefully manipulate these headers so neither the client nor the server gets confused.

**How to do it:**
- **When forwarding to the Origin Server:** The worker must strip "hop-by-hop" headers (like `connection` or `keep-alive`), because these are only meant for a single network connection. More importantly, the worker must overwrite the `Host`, `Origin`, and `Referer` headers to match the target origin. If you don't do this, the destination server will think the request is meant for your proxy and will likely reject it.
- **When returning data to the Client:** The worker should strip headers that might prevent you from using the data. For example, if you are building a web portal, you should delete `X-Frame-Options` and `Content-Security-Policy` so the origin site doesn't refuse to load inside an iframe. Finally, if your client is a web app making cross-origin requests, your worker must inject `Access-Control-Allow-Origin` (CORS) headers to satisfy the browser's security model. (Note: Native mobile apps and IoT devices ignore CORS, so injecting these headers is harmless for them).

#### 5. WebSocket Support (Optional)

**Why we do this:** Traditional HTTP is a one-way street (client asks, server answers). Modern apps and IoT devices often rely on WebSockets for real-time, two-way communication (like chat apps or live sensor data). 

**How to do it:** Cloudflare Workers support WebSockets natively. Your worker just needs to check if the incoming request includes an `Upgrade: websocket` header. If it does, the worker instructs Cloudflare to establish a persistent WebSocket tunnel to the origin server instead of a standard HTTP request.

### Step 5: Deploy

Once your logic is written, authenticate with Cloudflare and deploy your worker:

```bash
npx wrangler login   # Opens your browser to authenticate
npx wrangler deploy  # Deploys your code and prints your worker URL
```

*(Note: If you are building WebSocket functionality, it will not work fully in local `wrangler dev` environments. You must deploy to Cloudflare's actual edge network to test it).*

### Step 6: Custom Domain (Optional but Recommended)

**Why we do this:** By default, Cloudflare hosts your project on a `*.workers.dev` subdomain. Because so many people use `workers.dev` to host proxies and VPNs, extremely aggressive network filters (like Securly or Cisco Umbrella) sometimes blanket-block the entire `workers.dev` domain.

**How to do it:** You can sidestep this completely by routing your worker through a custom domain name that hasn't been flagged yet.
1. Add your custom domain to your Cloudflare account (the free tier works perfectly).
2. Add a routing rule to your `wrangler.toml` file, telling it to route traffic from `cdn.yourdomain.com/*` to your worker.
3. Redeploy your worker, and update your client application to point to your new custom domain.

### Step 7: Testing & Client Integration

Once deployed, you need to integrate the proxy into your client application. Here is how the complete flow looks in practice:

**Example Flow (e.g., an Arduino fetching API data):**
1. Your Arduino needs to fetch a temperature reading from `api.example.com/sensor-data`.
2. The Arduino code takes `api.example.com` and base64url-encodes it, resulting in `YXBpLmV4YW1wbGUuY29t`.
3. Instead of calling the API directly, the Arduino constructs a new URL and makes an HTTP GET request to `https://cdn.yourdomain.com/proxy/YXBpLmV4YW1wbGUuY29t/sensor-data`.
4. Your Cloudflare Worker intercepts the request. It extracts `YXBpLmV4YW1wbGUuY29t`, decodes it back to `api.example.com`, and verifies it against the allowlist.
5. The Worker reaches out to `https://api.example.com/sensor-data` on the Arduino's behalf.
6. The Worker receives the JSON data from the server, strips away any problematic headers, and streams the exact payload back down to the Arduino.

---

## Troubleshooting

If things aren't working, check these common pitfalls:

| Symptom | Cause | Fix |
|---|---|---|
| **403 Forbidden on all requests** | Host not in Allowlist | Ensure the decoded host matches your allowlist exactly. Be careful with `www.` prefixes; `example.com` and `www.example.com` are technically different hosts. |
| **403 + Wrong Host Decoded** | Parsing Error | Your URL router is likely too greedy. Ensure your path routing logic isn't accidentally consuming the rest of the URL path alongside the base64 string. |
| **CORS Errors (Browser Only)** | Origin Mismatch | If you are testing in a web browser, ensure the worker is successfully injecting the `Access-Control-Allow-Origin` header into the response. |
| **Connection Freezes** | Local Environment | If you are testing WebSockets, ensure you are testing against the deployed Cloudflare worker, not `wrangler dev`. |
| **Complete Block / Timeout** | Network Filter | The network may be blocking all `workers.dev` traffic. Set up a custom domain (Step 6) to bypass this. |