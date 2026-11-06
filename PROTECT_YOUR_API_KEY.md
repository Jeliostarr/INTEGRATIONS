# Protect Your API Key

Your API key is like a password for your quota. If it ends up in a web page, a public repository or an app package, anyone can copy it, spend your daily limit, and get **your** key suspended.

The fix is simple: **keep the key on a server you control and let your website or app talk to that server.** That server is called a **gateway**. This guide gives you a ready-made one that runs for free on Cloudflare Workers, plus free alternatives.

---

## Contents

1. [How it works](#how-it-works)
2. [Option A: Cloudflare Worker (recommended)](#option-a-cloudflare-worker-recommended)
3. [Calling the gateway](#calling-the-gateway)
4. [Make the gateway harder to abuse](#make-the-gateway-harder-to-abuse)
5. [Free alternatives](#free-alternatives)
6. [Playing video](#playing-video)
7. [Checklist](#checklist)
8. [If your key leaks](#if-your-key-leaks)

---

## How it works

```
 Your website / app  ──▶  YOUR gateway (holds the key)  ──▶  YOCINEMA API
        ▲                                                         │
        └────────── titles + media links (contain no key) ◀───────┘
```

- The **key never reaches the browser or the app**. It lives in the gateway as a secret.
- The gateway only forwards a short list of endpoints, so it can't be used as a general-purpose proxy.
- Responses for lists and details are cached in the gateway, which also **saves your daily quota**.
- Media links in the responses (video, subtitles, images) are **not credentials**. They play directly on the viewer's device and never pass through your gateway.

### What each layer stops

| Layer | Stops |
|---|---|
| Key stored as a server secret | Copying your key from page source, network tab or an app package |
| Path and parameter allow-list | Using your gateway as an open proxy for other endpoints |
| `ALLOWED_ORIGINS` | Other websites calling your gateway from a browser |
| Rate limit | One visitor draining your quota |
| Cache | Repeated identical requests costing quota |
| Turnstile (optional) | Scripts and bots requesting play links |
| Signed sessions (optional) | Anyone without a login on your own site/app |
| Errors from the API hidden | Visitors learning your key's status |

> **Be realistic:** a script can fake the `Origin` header. `ALLOWED_ORIGINS` stops other *websites* and casual copying; the **rate limit**, **Turnstile** and **signed sessions** are what stop determined scripts. Use them if your project is popular or has paying users.

---

## Option A: Cloudflare Worker (recommended)

Workers have a free plan that is plenty for a small project (at the time of writing, 100,000 requests per day; check Cloudflare's current limits).

### 1. Create the project

```bash
mkdir my-yocinema-gateway && cd my-yocinema-gateway
mkdir src
# create src/index.js and wrangler.toml with the code below
npm install --save-dev wrangler
```

### 2. `wrangler.toml`

```toml
name = "my-yocinema-gateway"
main = "src/index.js"
compatibility_date = "2026-10-01"

[vars]
# Browsers: the exact origin(s) of YOUR site, comma separated, no trailing slash.
ALLOWED_ORIGINS = "https://yoursite.com"
# Apps / your own server send no Origin header. Set to "true" ONLY if you need that, and then
# protect the gateway with SESSION_SECRET (see PROTECT_YOUR_API_KEY.md).
# ALLOW_NO_ORIGIN = "true"

# Per-visitor rate limit: 60 requests per 60 seconds per IP. (period must be 10 or 60)
[[unsafe.bindings]]
name = "RL"
type = "ratelimit"
namespace_id = "2001"
simple = { limit = 60, period = 60 }

# Secrets are NOT stored in this file. Set them with:
#   npx wrangler secret put YOCINEMA_API_KEY        (required)
#   npx wrangler secret put TURNSTILE_SECRET        (optional, bot check on play endpoints)
#   npx wrangler secret put SESSION_SECRET          (optional, signed sessions for logged-in users)
```

### 3. `src/index.js`

```js
// YOCINEMA API gateway for Cloudflare Workers
//
// Your API key lives ONLY here (as a secret). Your website or app calls this Worker; the
// Worker adds the key and forwards the request. Visitors never see the key.
//
// What protects it (each layer is small; together they stop casual abuse):
//   1. Only GET, only the listed paths, only the listed query parameters.
//   2. Browser callers must come from YOUR site (ALLOWED_ORIGINS).
//   3. Per-visitor rate limit (RL binding).
//   4. Optional: a Cloudflare Turnstile check before the "play" endpoints (TURNSTILE_SECRET).
//   5. Optional: your own signed session token for logged-in users (SESSION_SECRET).
//   6. Metadata responses are cached, which also saves your daily quota.
//   7. Errors from the API are never passed through, so visitors can't learn about your key.

const API_BASE = "https://axis.yocinema.dpdns.org";

// Only these endpoints can be reached through your gateway. `ttl` = seconds to cache.
// `heavy` = endpoints that hand out play links (optionally protected by Turnstile).
const ROUTES = [
  { re: /^\/api\/(home|trending|search|catalog|browse)$/, ttl: 300 },
  { re: /^\/api\/(details|seasons)\/[A-Za-z0-9._~%-]{1,200}$/, ttl: 1800 },
  { re: /^\/api\/(movie|tv|captions)\/[A-Za-z0-9._~%-]{1,200}$/, ttl: 0, heavy: true },
  { re: /^\/api\/sports\/(leagues|live|schedule|media|replays)$/, ttl: 30 },
  { re: /^\/api\/sports\/events\/\d{1,20}$/, ttl: 0 },
];

const ALLOWED_PARAMS = new Set([
  "q", "mode", "limit", "page", "type", "genre", "country", "year", "sort", "filters",
  "rankingId", "query", "season", "episode", "cursor", "leagueId", "sport", "date",
  "from", "to", "days",
]);

const json = (status, body, headers = {}) =>
  new Response(JSON.stringify(body), {
    status,
    headers: { "Content-Type": "application/json", ...headers },
  });

function allowedOrigins(env) {
  return String(env.ALLOWED_ORIGINS || "").split(",").map((s) => s.trim()).filter(Boolean);
}

// Browsers always send Origin on cross-site requests. A request with NO Origin comes from a
// server, an app or a script: only accept it when you opt in (ALLOW_NO_ORIGIN="true").
function callerAllowed(origin, env) {
  if (origin) return allowedOrigins(env).includes(origin);
  return env.ALLOW_NO_ORIGIN === "true";
}

function corsHeaders(origin, env) {
  if (!origin || !allowedOrigins(env).includes(origin)) return {};
  return {
    "Access-Control-Allow-Origin": origin,
    "Access-Control-Allow-Headers": "Content-Type, X-Turnstile-Token, X-Gateway-Session",
    "Access-Control-Allow-Methods": "GET, OPTIONS",
    "Vary": "Origin",
  };
}

// Optional session token: "<expiryUnixSeconds>.<hex HMAC-SHA256 of the expiry>", issued by
// YOUR backend after your own login. See PROTECT_YOUR_API_KEY.md for the issuing code.
async function sessionValid(token, secret) {
  if (!token) return false;
  const [exp, sig] = String(token).split(".");
  if (!exp || !sig || Number(exp) < Date.now() / 1000) return false;
  const key = await crypto.subtle.importKey(
    "raw", new TextEncoder().encode(secret), { name: "HMAC", hash: "SHA-256" }, false, ["sign"],
  );
  const mac = await crypto.subtle.sign("HMAC", key, new TextEncoder().encode(exp));
  const expected = [...new Uint8Array(mac)].map((b) => b.toString(16).padStart(2, "0")).join("");
  if (expected.length !== sig.length) return false;
  let diff = 0;
  for (let i = 0; i < expected.length; i++) diff |= expected.charCodeAt(i) ^ sig.charCodeAt(i);
  return diff === 0;
}

async function turnstileOk(token, secret, ip) {
  if (!token) return false;
  const body = new FormData();
  body.append("secret", secret);
  body.append("response", token);
  if (ip) body.append("remoteip", ip);
  const r = await fetch("https://challenges.cloudflare.com/turnstile/v0/siteverify", { method: "POST", body });
  const data = await r.json().catch(() => ({}));
  return data.success === true;
}

export default {
  async fetch(request, env, ctx) {
    const origin = request.headers.get("Origin");
    const cors = corsHeaders(origin, env);

    if (request.method === "OPTIONS") {
      return origin && cors["Access-Control-Allow-Origin"]
        ? new Response(null, { status: 204, headers: cors })
        : json(403, { error: "Forbidden" });
    }
    if (request.method !== "GET") return json(405, { error: "Method not allowed" }, cors);
    if (!callerAllowed(origin, env)) return json(403, { error: "Forbidden" }, cors);

    // Optional: require a session issued by your own backend.
    if (env.SESSION_SECRET && !(await sessionValid(request.headers.get("X-Gateway-Session"), env.SESSION_SECRET))) {
      return json(401, { error: "Sign in required" }, cors);
    }

    // Per-visitor rate limit.
    const ip = request.headers.get("CF-Connecting-IP") || "unknown";
    if (env.RL) {
      const { success } = await env.RL.limit({ key: ip });
      if (!success) return json(429, { error: "Too many requests" }, { ...cors, "Retry-After": "30" });
    }

    // Route + parameter allow-list.
    const url = new URL(request.url);
    const route = ROUTES.find((r) => r.re.test(url.pathname));
    if (!route) return json(404, { error: "Not found" }, cors);

    const params = new URLSearchParams();
    for (const [k, v] of url.searchParams) {
      if (!ALLOWED_PARAMS.has(k) || v.length > 300) return json(400, { error: "Bad request" }, cors);
      params.append(k, v);
    }
    params.sort();
    const qs = params.toString();
    const target = API_BASE + url.pathname + (qs ? "?" + qs : "");

    // Optional bot check before handing out play links.
    if (route.heavy && env.TURNSTILE_SECRET) {
      const ok = await turnstileOk(request.headers.get("X-Turnstile-Token"), env.TURNSTILE_SECRET, ip);
      if (!ok) return json(403, { error: "Verification failed" }, cors);
    }

    // Cache metadata (never the play links).
    const cache = route.ttl > 0 && typeof caches !== "undefined" ? caches.default : null;
    const cacheKey = new Request(target, { method: "GET" });
    if (cache) {
      const hit = await cache.match(cacheKey);
      if (hit) return withHeaders(hit, { ...cors, "Cache-Control": `public, max-age=${route.ttl}` });
    }

    // Forward with YOUR key. Anything the visitor sent as a key is ignored.
    let upstream;
    try {
      upstream = await fetch(target, { headers: { "X-API-Key": env.YOCINEMA_API_KEY, Accept: "application/json" } });
    } catch (e) {
      return json(502, { error: "Service unavailable" }, cors);
    }

    // Never pass the API's auth errors to visitors: they describe YOUR key. Log them for you.
    if (upstream.status === 401 || upstream.status === 403) {
      console.error("API rejected the gateway key:", upstream.status, await upstream.text().catch(() => ""));
      return json(502, { error: "Service unavailable" }, cors);
    }
    if (upstream.status === 429) return json(429, { error: "Busy, try again later" }, { ...cors, "Retry-After": "30" });

    const out = new Response(upstream.body, {
      status: upstream.status,
      headers: { "Content-Type": upstream.headers.get("Content-Type") || "application/json" },
    });
    if (cache && upstream.ok) {
      const toStore = withHeaders(out.clone(), { "Cache-Control": `public, max-age=${route.ttl}` });
      ctx.waitUntil(cache.put(cacheKey, toStore));
    }
    return withHeaders(out, { ...cors, "Cache-Control": route.ttl ? `public, max-age=${route.ttl}` : "no-store" });
  },
};

function withHeaders(res, extra) {
  const r = new Response(res.body, res);
  for (const [k, v] of Object.entries(extra)) r.headers.set(k, v);
  return r;
}
```

### 4. Add your key as a secret and deploy

```bash
npx wrangler secret put YOCINEMA_API_KEY     # paste your key when asked
npx wrangler deploy
```

Wrangler prints your gateway address, e.g. `https://my-yocinema-gateway.<your-name>.workers.dev`.

**Never** put the key in `wrangler.toml`, in code, or in a repository. For local testing create a file named `.dev.vars` containing `YOCINEMA_API_KEY=...` and run `npx wrangler dev`. Add `.dev.vars` to `.gitignore`.

### 5. Test it

```bash
# works: your site's origin
curl -H "Origin: https://yoursite.com" "https://my-yocinema-gateway.<your-name>.workers.dev/api/search?q=test"

# blocked: another site
curl -i -H "Origin: https://other.com" "https://my-yocinema-gateway.<your-name>.workers.dev/api/search?q=test"
```

---

## Calling the gateway

The gateway exposes the same paths as the API, so only the base URL changes.

### Website

```js
const GATEWAY = "https://my-yocinema-gateway.<your-name>.workers.dev";

async function api(path) {
  const res = await fetch(GATEWAY + path);          // no key, no special headers
  if (!res.ok) throw new Error("HTTP " + res.status);
  return res.json();
}

const { results } = await api("/api/search?q=oppenheimer&limit=5");
const stream = await api("/api/movie/" + results[0].detailPath);
document.querySelector("video").src = stream.best_free.url;   // plays directly
```

### Android / iOS app

Set your gateway as the base URL and **send no key from the app**:

```kotlin
Retrofit.Builder()
    .baseUrl("https://my-yocinema-gateway.<your-name>.workers.dev/")
    // ...
```

Apps send no `Origin` header, so set `ALLOW_NO_ORIGIN = "true"` in `wrangler.toml` **and** turn on signed sessions (below), otherwise anyone who finds your gateway address could use it.

---

## Make the gateway harder to abuse

### Rate limit (already included)

The `RL` binding in `wrangler.toml` allows 60 requests per minute per visitor. Lower it if your site doesn't need that many.

### Turnstile bot check (free, optional)

Cloudflare Turnstile is a free, privacy-friendly alternative to CAPTCHA. With it on, the endpoints that hand out play links (`movie`, `tv`, `captions`) only answer requests that carry a valid Turnstile token.

1. Create a Turnstile widget in the Cloudflare dashboard and note the **site key** and **secret key**.
2. `npx wrangler secret put TURNSTILE_SECRET` and paste the secret key.
3. On your page:

```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
<div class="cf-turnstile" data-sitekey="YOUR_SITE_KEY" data-callback="onTurnstile"></div>
<script>
  let tsToken = "";
  function onTurnstile(t) { tsToken = t; }

  async function getStream(detailPath) {
    const res = await fetch(GATEWAY + "/api/movie/" + detailPath, {
      headers: { "X-Turnstile-Token": tsToken },
    });
    turnstile.reset();            // tokens are single-use: get a fresh one for the next request
    return res.json();
  }
</script>
```

### Signed sessions for logged-in users (optional, recommended for apps)

If your site or app has its own login, make your **own backend** issue a short-lived token after login, and have the gateway reject requests without one.

1. `npx wrangler secret put SESSION_SECRET` (any long random string; keep the same value in your backend).
2. In your backend, after the user logs in:

```js
import crypto from "node:crypto";

export function issueSession(secret, ttlSeconds = 900) {
  const exp = String(Math.floor(Date.now() / 1000) + ttlSeconds);
  const sig = crypto.createHmac("sha256", secret).update(exp).digest("hex");
  return exp + "." + sig;               // send this to the logged-in client
}
```

3. The client sends it with every gateway call:

```js
fetch(GATEWAY + "/api/home", { headers: { "X-Gateway-Session": sessionToken } });
```

Requests without a valid, unexpired token get `401`.

---

## Free alternatives

Any place that can run a little server-side code and keep a secret works. The logic is identical to the Worker: check the caller, check the path, add the key, forward, hide key errors.

| Platform | Free tier | Where the secret goes | Notes |
|---|---|---|---|
| **Cloudflare Workers** | Yes | `wrangler secret put` | Full example above (rate limit + cache built in) |
| **Cloudflare Pages Functions** | Yes | Project settings → Variables and Secrets | Same code as the Worker in `functions/` |
| **Vercel Functions** | Yes (hobby) | Project → Settings → Environment Variables | Example below |
| **Deno Deploy** | Yes | Project → Settings → Environment Variables | Example below |
| **Netlify Functions** | Yes | Site settings → Environment variables | Same logic as the Vercel example |
| **Supabase Edge Functions** | Yes | `supabase secrets set` | Same logic as the Deno example |
| **Render / Fly.io / Railway** | Limited or trial tiers | Dashboard secrets | Run the same handler in a tiny Node server |

Free tiers change, so check each provider's current limits.

### Vercel (Node) example

Save as `api/yocinema/[...path].js`. Call it as `/api/yocinema/search?q=test` from your site.

```js
const API = "https://axis.yocinema.dpdns.org/api/";
const ALLOWED = [
  /^(home|trending|search|catalog|browse)$/,
  /^(details|seasons|movie|tv|captions)\/[A-Za-z0-9._~%-]{1,200}$/,
  /^sports\/(leagues|live|schedule|media|replays)$/,
  /^sports\/events\/\d{1,20}$/,
];
const PARAMS = new Set(["q","mode","limit","page","type","genre","country","year","sort","filters",
  "rankingId","query","season","episode","cursor","leagueId","sport","date","from","to","days"]);
const ORIGINS = (process.env.ALLOWED_ORIGINS || "").split(",").map((s) => s.trim()).filter(Boolean);

export default async function handler(req, res) {
  const origin = req.headers.origin;
  if (req.method !== "GET" || !origin || !ORIGINS.includes(origin)) return res.status(403).json({ error: "Forbidden" });

  const path = [].concat(req.query.path || []).join("/");
  if (!ALLOWED.some((re) => re.test(path))) return res.status(404).json({ error: "Not found" });

  const qs = new URLSearchParams();
  for (const [k, v] of Object.entries(req.query)) {
    if (k === "path") continue;
    if (!PARAMS.has(k) || String(v).length > 300) return res.status(400).json({ error: "Bad request" });
    qs.set(k, String(v));
  }

  const r = await fetch(API + path + (qs.size ? "?" + qs : ""), {
    headers: { "X-API-Key": process.env.YOCINEMA_API_KEY, Accept: "application/json" },
  });
  if (r.status === 401 || r.status === 403) return res.status(502).json({ error: "Service unavailable" });

  const playLinks = /^(movie|tv|captions)\//.test(path);
  res.setHeader("Access-Control-Allow-Origin", origin);
  res.setHeader("Vary", "Origin");
  res.setHeader("Cache-Control", playLinks ? "no-store" : "public, s-maxage=300, stale-while-revalidate=600");
  res.status(r.status).setHeader("Content-Type", "application/json").send(await r.text());
}
```

Set `YOCINEMA_API_KEY` and `ALLOWED_ORIGINS` as environment variables. Vercel has no built-in per-visitor rate limit on the free plan, so add one (for example with a free Upstash Redis limiter, or the Vercel Firewall).

### Deno Deploy example

```ts
const API = "https://axis.yocinema.dpdns.org";
const ALLOWED = [
  /^\/api\/(home|trending|search|catalog|browse)$/,
  /^\/api\/(details|seasons|movie|tv|captions)\/[A-Za-z0-9._~%-]{1,200}$/,
  /^\/api\/sports\/(leagues|live|schedule|media|replays)$/,
  /^\/api\/sports\/events\/\d{1,20}$/,
];
const ORIGINS = (Deno.env.get("ALLOWED_ORIGINS") ?? "").split(",").map((s) => s.trim()).filter(Boolean);

Deno.serve(async (req) => {
  const origin = req.headers.get("origin") ?? "";
  const url = new URL(req.url);
  if (req.method !== "GET" || !ORIGINS.includes(origin)) return new Response("Forbidden", { status: 403 });
  if (!ALLOWED.some((re) => re.test(url.pathname))) return new Response("Not found", { status: 404 });

  const r = await fetch(API + url.pathname + url.search, {
    headers: { "X-API-Key": Deno.env.get("YOCINEMA_API_KEY")!, Accept: "application/json" },
  });
  if (r.status === 401 || r.status === 403) return new Response("Service unavailable", { status: 502 });

  return new Response(r.body, {
    status: r.status,
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": origin,
      "Vary": "Origin",
      "Cache-Control": "no-store",
    },
  });
});
```

This short version forwards the query string as-is. For the stricter parameter allow-list, copy the `ALLOWED_PARAMS` check from the Worker.

---

## Playing video

Use the links from the API responses **directly in the player**. Do not route video or subtitles through your gateway:

- It adds delay and uses your platform's bandwidth allowance.
- Hosting platforms' terms often restrict using their free functions as a video proxy.
- The links already work without any key and are designed to be opened by the viewer's device.

Request a link **when the viewer presses play** (one link per viewer), because links expire and are tied to your key. See *Media links* in the integration guide.

---

## Checklist

- [ ] The API key is stored only as a secret on the gateway, never in client code or Git
- [ ] `ALLOWED_ORIGINS` contains only your own site(s)
- [ ] The path and parameter allow-lists are unchanged (don't add `*`)
- [ ] The rate limit is on
- [ ] Turnstile and/or signed sessions are on if your project is public or paid
- [ ] `.dev.vars`, `.env` and similar files are in `.gitignore`
- [ ] Video is played from the returned links, not proxied
- [ ] You know how to rotate your key (dashboard) and update the secret

## If your key leaks

1. **Create a new key** in the dashboard (<https://dash.yocinema.dpdns.org>) and revoke the old one.
2. Update the gateway secret: `npx wrangler secret put YOCINEMA_API_KEY`, or the equivalent on your platform.
3. Check what leaked it: a public repository, a browser bundle, an app build.
4. Message us if you need help (see the integration guide's Support section).
