# Building Production Apps on the Thunderbit API

Ship a real product feature — "paste a URL, get back structured data or clean Markdown" — without leaking your API key, getting SSRF'd, or burning credits on duplicate work. This is a developer-integration guide: architecture, security, and resilience patterns for embedding Thunderbit's web-data engine inside your own backend.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## What we're building

A small full-stack feature you can drop into any product: a user pastes a URL, your service fetches it through Thunderbit, and you return either **structured fields** (via `/extract`) or a **clean Markdown copy** (via `/distill`) that you also persist for later.

The data flow, in words:

```
[ Browser / mobile app ]
        │  POST /api/capture { "url": "https://..." }   (your own auth, e.g. session cookie / JWT)
        ▼
[ Your backend ]  ← holds THUNDERBIT_API_KEY (server-side secret)
        │  1. authenticate + authorize the user
        │  2. validate & sanitize the URL  (SSRF guard)
        │  3. check cache  → hit? return stored result
        │  4. miss? POST https://openapi.thunderbit.com/openapi/v1/{extract|distill}
        │  5. store result, increment per-user usage counter
        ▼
[ Thunderbit Open API ]  ── returns { "data": ... }
        │
        ▼
[ Your database / object store ]  → result returned to the client as JSON
```

The single most important property of this design: **the browser never talks to Thunderbit directly.** It talks to *your* backend, which holds the secret and enforces your rules. Everything else in this guide hangs off that one decision.

---

## Architecture: never call Thunderbit from the browser

Your Thunderbit API key (`tb_...`) is a **bearer credential that spends money**. Anyone who has it can run extracts against your account until your credits are gone and your billing alert fires. Treat it exactly like a database password.

That means:

- **Never** put `tb_...` in frontend JavaScript, a React/Vue bundle, a mobile app binary, or a `NEXT_PUBLIC_*` / `VITE_*` env var. Anything shipped to the client is readable — minification is not encryption. Anyone can open DevTools or decompile the APK and read it.
- **Never** proxy the key by echoing it into HTML, a meta tag, or a config endpoint the browser can fetch.
- **Do** keep it server-side only, loaded from an environment variable or a secret manager (GCP Secret Manager, AWS Secrets Manager, Vault, Doppler). The reference is explicit: store the key in `THUNDERBIT_API_KEY` and never hard-code it.

```bash
# Server environment only — never in a client-shipped bundle
export THUNDERBIT_API_KEY=tb_your_api_key_here
```

The correct topology is a thin proxy you own:

```
browser  ──auth──▶  your backend  ──Bearer tb_...──▶  Thunderbit
```

Your backend is where you authenticate the user, validate the URL, enforce quotas, cache, and log spend. If you find yourself wanting to "just call it from the frontend to skip a hop," stop — that hop is the entire security boundary.

---

## A backend endpoint (Node / Express or Next.js route)

Here's the synchronous capture endpoint. It authenticates (left as a stub — wire your own session/JWT check), validates the URL, calls Thunderbit, and returns JSON. We use the thin `fetch` wrapper pattern from the reference.

### Express

```js
// capture.js  — Node 18+ (global fetch). npm i express
import express from "express";
import { assertSafePublicUrl } from "./url-safety.js"; // built in the next section

const BASE = "https://openapi.thunderbit.com";
const KEY = process.env.THUNDERBIT_API_KEY;
if (!KEY) throw new Error("THUNDERBIT_API_KEY is not set");

async function tb(path, body) {
  const res = await fetch(BASE + path, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${KEY}`,
      "Content-Type": "application/json",
      Accept: "application/json",
    },
    body: JSON.stringify(body),
  });
  const text = await res.text();
  if (!res.ok) {
    const err = new Error(`Thunderbit ${res.status}: ${text}`);
    err.status = res.status;
    throw err;
  }
  return JSON.parse(text);
}

// A fixed, server-owned schema. Do NOT let the client send an arbitrary schema
// unless you intend to expose extract's full power (and pay for it).
const CAPTURE_SCHEMA = {
  type: "array",
  items: {
    type: "object",
    properties: {
      title:       { type: "TEXT", instruction: "The main page or article title" },
      summary:     { type: "TEXT", instruction: "A one-paragraph summary of the page" },
      published:   { type: "DATE", instruction: "Publish date, if shown" },
      canonical_url:{ type: "URL", instruction: "The canonical link of the page" },
    },
  },
};

const app = express();
app.use(express.json({ limit: "16kb" })); // small bodies only; we expect just a URL

app.post("/api/capture", async (req, res) => {
  // 1. AUTH — replace with your real check
  const user = req.user; // e.g. set by an auth middleware
  if (!user) return res.status(401).json({ error: "unauthenticated" });

  // 2. VALIDATE INPUT
  const { url, mode = "extract" } = req.body ?? {};
  if (typeof url !== "string" || url.length > 2048) {
    return res.status(400).json({ error: "url is required and must be a string" });
  }
  let safeUrl;
  try {
    safeUrl = await assertSafePublicUrl(url); // throws on SSRF / bad scheme
  } catch (e) {
    return res.status(400).json({ error: `rejected url: ${e.message}` });
  }

  // 3. CALL THUNDERBIT
  try {
    if (mode === "distill") {
      const { data } = await tb("/openapi/v1/distill", {
        url: safeUrl,
        renderMode: "full",
        excludeTags: ["nav", "footer", "aside"],
        timeout: 30000,
      });
      return res.json({ kind: "markdown", markdown: data.markdown, url: data.url });
    }
    const { data } = await tb("/openapi/v1/extract", {
      url: safeUrl,
      schema: CAPTURE_SCHEMA,
      renderMode: "full",
      timeout: 60000,
    });
    return res.json({ kind: "fields", rows: data });
  } catch (e) {
    if (e.status === 401) return res.status(500).json({ error: "server misconfigured" }); // never expose key errors to user
    if (e.status === 402) return res.status(503).json({ error: "capture temporarily unavailable" }); // out of credits
    if (e.status === 429) return res.status(429).json({ error: "busy, retry shortly" });
    return res.status(502).json({ error: "capture failed" });
  }
});

app.listen(3000, () => console.log("capture service on :3000"));
```

### Next.js (App Router) route handler

The same logic as a route handler — keep it on the server, never in a client component:

```ts
// app/api/capture/route.ts
import { NextRequest, NextResponse } from "next/server";
import { assertSafePublicUrl } from "@/lib/url-safety";

export const runtime = "nodejs"; // need Node APIs (dns) for the SSRF check below

const BASE = "https://openapi.thunderbit.com";
const KEY = process.env.THUNDERBIT_API_KEY!; // server-only; NOT NEXT_PUBLIC_*

export async function POST(req: NextRequest) {
  const { url } = await req.json();
  if (typeof url !== "string" || url.length > 2048) {
    return NextResponse.json({ error: "bad url" }, { status: 400 });
  }
  let safeUrl: string;
  try {
    safeUrl = await assertSafePublicUrl(url);
  } catch (e: any) {
    return NextResponse.json({ error: `rejected url: ${e.message}` }, { status: 400 });
  }

  const res = await fetch(`${BASE}/openapi/v1/distill`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${KEY}`,
      "Content-Type": "application/json",
      Accept: "application/json",
    },
    body: JSON.stringify({ url: safeUrl, renderMode: "full", timeout: 30000 }),
  });
  if (!res.ok) {
    if (res.status === 402) return NextResponse.json({ error: "unavailable" }, { status: 503 });
    return NextResponse.json({ error: "capture failed" }, { status: 502 });
  }
  const { data } = await res.json();
  return NextResponse.json({ markdown: data.markdown, url: data.url });
}
```

Note `runtime = "nodejs"` — the SSRF helper needs DNS resolution, which the Edge runtime doesn't provide.

---

## SSRF & input safety

The moment you let users supply the URL you fetch, you've built a potential **Server-Side Request Forgery (SSRF)** weapon. An attacker submits `http://169.254.169.254/latest/meta-data/` (the cloud metadata endpoint) or `http://localhost:6379` and tries to get your infrastructure to reach internal services. Thunderbit fetches the page from its own infrastructure, which mitigates a lot — but you should still refuse to *forward* dangerous targets, because (a) you may add your own pre-fetch (favicon, OpenGraph preview) and (b) you don't want to become an open proxy that launders requests.

A robust validator does five things:

1. **Scheme allowlist.** Only `http` and `https`. Reject `file:`, `gopher:`, `ftp:`, `data:`, `javascript:`.
2. **Reject credentials in the URL** (`http://user:pass@host`) — a common parser-confusion trick.
3. **Resolve the hostname and block private/reserved ranges** — loopback (`127.0.0.0/8`, `::1`), private (`10/8`, `172.16/12`, `192.168/16`), link-local (`169.254/16`, including the cloud metadata IP `169.254.169.254`), unique-local IPv6 (`fc00::/7`), and `0.0.0.0`.
4. **Block obvious metadata hostnames** like `metadata.google.internal`.
5. **Optionally allowlist domains** — the strongest control if your product only needs to capture a known set of sites.

```js
// url-safety.js  — Node 18+
import dns from "node:dns/promises";
import net from "node:net";

const ALLOWED_SCHEMES = new Set(["http:", "https:"]);

// Optional: if set, ONLY these registrable domains are allowed (strongest guard).
const DOMAIN_ALLOWLIST = null; // e.g. new Set(["example.com", "wikipedia.org"])

const BLOCKED_HOSTNAMES = new Set([
  "localhost", "metadata", "metadata.google.internal",
]);

function isPrivateIp(ip) {
  if (net.isIPv4(ip)) {
    const [a, b] = ip.split(".").map(Number);
    if (a === 0 || a === 127 || a === 10) return true;            // this-host, loopback, private
    if (a === 169 && b === 254) return true;                       // link-local + cloud metadata
    if (a === 172 && b >= 16 && b <= 31) return true;              // private
    if (a === 192 && b === 168) return true;                       // private
    if (a === 100 && b >= 64 && b <= 127) return true;             // CGNAT
    return false;
  }
  if (net.isIPv6(ip)) {
    const low = ip.toLowerCase();
    if (low === "::1" || low === "::") return true;                // loopback / unspecified
    if (low.startsWith("fc") || low.startsWith("fd")) return true; // unique-local fc00::/7
    if (low.startsWith("fe80")) return true;                       // link-local
    if (low.startsWith("::ffff:")) return isPrivateIp(low.split(":").pop()); // IPv4-mapped
    return false;
  }
  return false;
}

export async function assertSafePublicUrl(raw) {
  let u;
  try {
    u = new URL(raw);
  } catch {
    throw new Error("not a valid URL");
  }
  if (!ALLOWED_SCHEMES.has(u.protocol)) throw new Error("only http/https allowed");
  if (u.username || u.password) throw new Error("credentials in URL not allowed");

  const host = u.hostname.toLowerCase().replace(/\.$/, ""); // strip trailing dot
  if (BLOCKED_HOSTNAMES.has(host)) throw new Error("blocked hostname");

  if (DOMAIN_ALLOWLIST) {
    const ok = [...DOMAIN_ALLOWLIST].some((d) => host === d || host.endsWith("." + d));
    if (!ok) throw new Error("domain not on allowlist");
  }

  // Resolve and check EVERY address the hostname maps to (defends against
  // DNS-rebinding-style targets that return a private IP).
  let addrs;
  if (net.isIP(host)) {
    addrs = [host];
  } else {
    const records = await dns.lookup(host, { all: true });
    addrs = records.map((r) => r.address);
  }
  if (addrs.length === 0) throw new Error("hostname does not resolve");
  for (const ip of addrs) {
    if (isPrivateIp(ip)) throw new Error("resolves to a private/reserved address");
  }

  return u.toString();
}
```

Two caveats. First, a determined attacker can use **DNS rebinding** (return a public IP at validation time, a private IP at fetch time). Because the actual page fetch happens on Thunderbit's infrastructure, your exposure is lower than a naive in-process fetcher — but if you *also* fetch the URL yourself anywhere (preview images, link unfurls), resolve once and connect to the resolved IP. Second, **the allowlist is the only control that fully closes the class** — if your product can enumerate the sites it supports, prefer it.

---

## A Python / FastAPI variant

The same endpoint for teams on Python. Uses `requests` (per the reference) for the Thunderbit call and a synchronous SSRF check mirroring the Node one.

```python
# main.py  — pip install fastapi uvicorn requests
import os, ipaddress, socket
from urllib.parse import urlparse

import requests
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]  # server-only secret
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json",
           "Accept": "application/json"}

ALLOWED_SCHEMES = {"http", "https"}
BLOCKED_HOSTS = {"localhost", "metadata", "metadata.google.internal"}
DOMAIN_ALLOWLIST: set[str] | None = None  # e.g. {"example.com"} for the strongest guard

CAPTURE_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "title":   {"type": "TEXT", "instruction": "The main page or article title"},
        "summary": {"type": "TEXT", "instruction": "A one-paragraph summary of the page"},
        "published": {"type": "DATE", "instruction": "Publish date, if shown"},
    }},
}

def assert_safe_public_url(raw: str) -> str:
    if len(raw) > 2048:
        raise ValueError("url too long")
    u = urlparse(raw)
    if u.scheme not in ALLOWED_SCHEMES:
        raise ValueError("only http/https allowed")
    if u.username or u.password:
        raise ValueError("credentials in URL not allowed")
    host = (u.hostname or "").lower().rstrip(".")
    if not host or host in BLOCKED_HOSTS:
        raise ValueError("blocked or empty hostname")
    if DOMAIN_ALLOWLIST is not None:
        if not any(host == d or host.endswith("." + d) for d in DOMAIN_ALLOWLIST):
            raise ValueError("domain not on allowlist")
    # Resolve every address and reject private/reserved ranges.
    infos = socket.getaddrinfo(host, None)
    addrs = {info[4][0] for info in infos}
    if not addrs:
        raise ValueError("hostname does not resolve")
    for ip in addrs:
        addr = ipaddress.ip_address(ip)
        if (addr.is_private or addr.is_loopback or addr.is_link_local
                or addr.is_reserved or addr.is_multicast or addr.is_unspecified):
            raise ValueError("resolves to a private/reserved address")
    return u.geturl()

app = FastAPI()

class CaptureBody(BaseModel):
    url: str = Field(..., max_length=2048)
    mode: str = "extract"

@app.post("/api/capture")
def capture(body: CaptureBody):
    # AUTH: wire your real dependency here (OAuth2 / session) before this point.
    try:
        safe_url = assert_safe_public_url(body.url)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=f"rejected url: {e}")

    if body.mode == "distill":
        payload = {"url": safe_url, "renderMode": "full", "timeout": 30000,
                   "excludeTags": ["nav", "footer", "aside"]}
        path = "/openapi/v1/distill"
    else:
        payload = {"url": safe_url, "schema": CAPTURE_SCHEMA,
                   "renderMode": "full", "timeout": 60000}
        path = "/openapi/v1/extract"

    r = requests.post(f"{BASE}{path}", headers=HEADERS, json=payload, timeout=130)
    if r.status_code == 401:
        raise HTTPException(status_code=500, detail="server misconfigured")
    if r.status_code == 402:
        raise HTTPException(status_code=503, detail="capture temporarily unavailable")
    if r.status_code == 429:
        raise HTTPException(status_code=429, detail="busy, retry shortly")
    if not r.ok:
        raise HTTPException(status_code=502, detail="capture failed")

    data = r.json()["data"]
    if body.mode == "distill":
        return {"kind": "markdown", "markdown": data["markdown"], "url": data["url"]}
    return {"kind": "fields", "rows": data}
```

Run it with `uvicorn main:app`. The `requests` call uses a 130s client timeout — slightly above the extract `timeout` ceiling of 120000 ms — so the HTTP client doesn't give up before Thunderbit does.

---

## Caching & dedupe to control credits

Extract costs **20 credits** per page; distill costs **1**. If two users capture the same URL with the same schema, paying twice is pure waste. Cache by a key of **`url` + a hash of the schema** (and the mode), with a TTL that reflects how fast the source page changes.

The cost impact is direct: a cache hit costs **0 credits** instead of 20. For any product where popular URLs get captured repeatedly (shared articles, trending product pages), a cache routinely cuts spend by an order of magnitude.

SQLite version (good for a single instance or low volume):

```js
// cache.js  — npm i better-sqlite3
import Database from "better-sqlite3";
import crypto from "node:crypto";

const db = new Database("capture-cache.sqlite");
db.exec(`CREATE TABLE IF NOT EXISTS cache (
  key TEXT PRIMARY KEY, payload TEXT NOT NULL, expires_at INTEGER NOT NULL)`);

function cacheKey(url, mode, schema) {
  const h = crypto.createHash("sha256")
    .update(JSON.stringify({ url, mode, schema: schema ?? null }))
    .digest("hex");
  return h;
}

export function cacheGet(url, mode, schema) {
  const row = db.prepare("SELECT payload, expires_at FROM cache WHERE key = ?")
    .get(cacheKey(url, mode, schema));
  if (!row) return null;
  if (row.expires_at < Date.now()) return null; // expired; treat as miss
  return JSON.parse(row.payload);
}

export function cacheSet(url, mode, schema, payload, ttlSeconds = 86400) {
  db.prepare("INSERT OR REPLACE INTO cache (key, payload, expires_at) VALUES (?,?,?)")
    .run(cacheKey(url, mode, schema), JSON.stringify(payload), Date.now() + ttlSeconds * 1000);
}
```

Wire it into the endpoint so a hit short-circuits the paid call:

```js
const cached = cacheGet(safeUrl, mode, schema);
if (cached) return res.json({ ...cached, cached: true }); // 0 credits

const result = await tb("/openapi/v1/extract", { url: safeUrl, schema, renderMode: "full" });
const payload = { kind: "fields", rows: result.data };
cacheSet(safeUrl, mode, schema, payload, 86400); // 24h TTL — tune per source freshness
return res.json(payload);
```

Redis is the same idea for multi-instance deployments — `SET key <json> EX <ttl>` on miss, `GET key` first:

```
GET  capture:<sha256(url|mode|schema)>            # hit → return, 0 credits
SET  capture:<sha256(...)>  <json>  EX 86400      # miss → store after the paid call
```

Pick the TTL deliberately: a news article's structured fields are stable for hours; a live price might warrant minutes; an evergreen reference page, days. The schema hash is essential — two users asking for *different* fields from the same URL are genuinely different captures and must not share a cache entry.

---

## Long jobs: async + webhooks

A JS-heavy SPA can take 60–90 seconds to render. Holding an HTTP request open that long is fragile (load balancers, proxies, and browsers all time out) and ties up a worker. For slow single pages use the **async** endpoints; for many URLs use **batch**. Both can notify you via **webhook** so you don't poll.

The pattern: enqueue the job, immediately return `202 Accepted` plus a job id to the client, then finish the work out-of-band and notify the client (the client polls your status endpoint, or you push over a websocket).

```js
// Enqueue: submit to Thunderbit's batch endpoint with a webhook, return 202.
app.post("/api/capture/async", async (req, res) => {
  const { url } = req.body ?? {};
  let safeUrl;
  try { safeUrl = await assertSafePublicUrl(url); }
  catch (e) { return res.status(400).json({ error: e.message }); }

  const job = await tb("/openapi/v1/batch/extract", {
    urls: [safeUrl],
    schema: CAPTURE_SCHEMA,
    timeout: 90000,
    webhook: {
      url: "https://your-server.example.com/thunderbit/callback",
      secret: process.env.THUNDERBIT_WEBHOOK_SECRET, // a signing secret YOU choose
    },
  });

  const jobId = job.data.id;
  // Persist {jobId → user, status:'pending'} so the callback can find the owner.
  saveJob(jobId, { user: req.user.id, status: "pending", url: safeUrl });
  return res.status(202).json({ jobId, status: "pending" });
});
```

The webhook handler must **verify the secret** before trusting the payload — anyone who learns your callback URL could otherwise POST fake results. We compare in constant time and we keep the raw body so the comparison is exact.

```js
// Webhook receiver — verify the shared secret, then persist + notify.
app.post(
  "/thunderbit/callback",
  express.json({
    limit: "5mb",
    verify: (req, _res, buf) => { req.rawBody = buf; }, // keep raw bytes if you later switch to HMAC
  }),
  (req, res) => {
    const body = req.body ?? {};
    const provided = body?.webhook?.secret ?? req.get("x-thunderbit-secret");
    const expected = process.env.THUNDERBIT_WEBHOOK_SECRET;

    // Constant-time comparison; reject anything that doesn't match.
    const ok =
      typeof provided === "string" &&
      provided.length === expected.length &&
      crypto.timingSafeEqual(Buffer.from(provided), Buffer.from(expected));
    if (!ok) return res.status(401).json({ error: "bad signature" });

    const job = body.data ?? body;
    const result = (job.results || [])[0] || {};
    if (result.status === "completed") {
      const rows = result.data || [];
      persistResult(job.id, rows);                  // write to your DB / object store
      notifyClient(job.id, { status: "done", rows }); // push over websocket / mark for polling
    } else {
      notifyClient(job.id, { status: "failed", error: result.error });
    }
    return res.json({ ok: true }); // 2xx so Thunderbit doesn't retry
  }
);
```

The client either polls `GET /api/capture/status/:jobId` (which reads your stored job row) or listens on a websocket channel keyed by `jobId`. Either way, the original request returned in milliseconds and no HTTP connection was held open for 90 seconds.

For a single slow page without a webhook, the async submit/poll endpoints (`POST /openapi/v1/async/extract` → `GET /openapi/v1/async/extract/{jobId}` every ~3s) work too; polling is free.

---

## Resilience: retries, rate limits, circuit-breaking

A production client must handle the three failure codes from the reference distinctly:

- **429 (rate-limited):** retry with **exponential backoff + jitter**.
- **5xx (server):** same — retry with backoff.
- **402 (out of credits):** **hard stop.** Don't retry; alert ops and degrade the user flow gracefully (return a friendly "temporarily unavailable," don't crash).
- **401 (bad key):** **never retry** — it will never succeed. Alert ops; it means your key is wrong or rotated.

```js
// tb-client.js — resilient wrapper around the raw call.
class OutOfCredits extends Error {}
class AuthError extends Error {}

async function tbResilient(path, body, { attempts = 5 } = {}) {
  for (let i = 0; i < attempts; i++) {
    const res = await fetch(BASE + path, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${KEY}`,
        "Content-Type": "application/json",
        Accept: "application/json",
      },
      body: JSON.stringify(body),
    });

    if (res.ok) return res.json();

    if (res.status === 401) {
      alertOps("Thunderbit 401 — API key invalid/rotated");
      throw new AuthError("401"); // never retry
    }
    if (res.status === 402) {
      alertOps("Thunderbit 402 — out of credits, top up at thunderbit.com/billing");
      tripCircuitBreaker(); // see below
      throw new OutOfCredits("402"); // hard stop, no retry
    }
    if (res.status === 429 || res.status >= 500) {
      const backoff = Math.min(30000, 2 ** i * 500) + Math.random() * 500; // + jitter
      await new Promise((r) => setTimeout(r, backoff));
      continue;
    }
    throw new Error(`Thunderbit ${res.status}: ${await res.text()}`);
  }
  throw new Error(`gave up on ${path} after ${attempts} attempts`);
}
```

A simple **circuit breaker** prevents stampeding Thunderbit (and your bill) when something is already broken. After a 402 or a run of failures, "open" the breaker for a cooldown so new requests fail fast and cheaply instead of all retrying:

```js
let breakerOpenUntil = 0;
function tripCircuitBreaker(ms = 5 * 60 * 1000) { breakerOpenUntil = Date.now() + ms; }
function breakerOpen() { return Date.now() < breakerOpenUntil; }

// At the top of your endpoint:
if (breakerOpen()) return res.status(503).json({ error: "capture temporarily unavailable" });
```

**Per-user quotas** cap blast radius and spend. Before each paid call, increment a counter (Redis `INCR` with a daily-expiring key works well) and reject over-limit users with `429` — your own, not Thunderbit's:

```js
async function checkUserQuota(userId, dailyLimit = 50) {
  const key = `quota:${userId}:${new Date().toISOString().slice(0, 10)}`;
  const used = await redis.incr(key);
  if (used === 1) await redis.expire(key, 86400);
  if (used > dailyLimit) throw new Error("daily capture limit reached");
}
```

---

## Background jobs & queues

For bulk ingestion — a user uploads a CSV of 5,000 URLs, or a nightly crawl — don't fan out 5,000 synchronous calls. Push the URLs onto a **queue** and have a worker consume them, processing with the **batch** endpoint in **chunks of 100** (the batch ceiling). Store per-URL results from `results[]`.

The queue can be a jobs table, Redis list, or an SQS-style service. The shape is the same:

```js
// worker.js — consumes a jobs table, batch-extracts in chunks of 100.
async function processBatchJob(jobId, urls) {
  // Validate every URL first; drop the unsafe ones before spending a credit.
  const safe = [];
  for (const u of urls) {
    try { safe.push(await assertSafePublicUrl(u)); }
    catch { /* log + skip — never forward an SSRF target into a batch */ }
  }

  for (let start = 0; start < safe.length; start += 100) {
    const chunk = safe.slice(start, start + 100);
    const job = (await tbResilient("/openapi/v1/batch/extract", {
      urls: chunk,
      schema: CAPTURE_SCHEMA,
      timeout: 90000,
    })).data;

    // Poll until terminal (or use a webhook as in the previous section).
    let status;
    do {
      await new Promise((r) => setTimeout(r, 3000)); // polling is free
      status = (await tbGet(`/openapi/v1/batch/extract/${job.id}`,
                            { page: 0, pageSize: 100 })).data;
    } while (!["completed", "failed", "cancelled"].includes(status.status));

    for (const result of status.results || []) {
      if (result.status !== "completed") {
        recordFailure(jobId, result.url, result.error); // retry only these later
        continue;
      }
      const rows = result.data || [];
      storeRows(jobId, result.url, rows); // one DB write per source URL
    }
  }
}
```

Key practices: **validate before enqueuing** (an SSRF target in a batch is still an SSRF target), **chunk at 100**, **retry only the per-URL failures** (re-running the whole batch double-charges the rows that already succeeded), and keep batch concurrency modest so you don't hammer a single origin.

---

## Observability & cost controls

You can't control spend you can't see. Log credits-per-request and expose a metric so cost is visible in your dashboards.

```js
// Tag every paid call with its credit cost so you can sum spend per user / per day.
const COST = {
  "/openapi/v1/distill": 1,
  "/openapi/v1/suggest_fields": 1,
  "/openapi/v1/extract": 20,
  "/openapi/v1/batch/extract": 20, // per URL — multiply by urls.length
  "/openapi/v1/batch/distill": 1,  // per URL
};

function recordSpend(userId, path, urlCount = 1) {
  const credits = (COST[path] || 0) * urlCount;
  metrics.counter("thunderbit_credits_total", credits, { path, user: userId });
  logger.info({ event: "tb_spend", userId, path, urlCount, credits });
  return credits;
}
```

Then:

- **Set spend alerts.** Alert when daily credits cross a budget threshold; a runaway loop or a popular URL with a broken cache shows up here first.
- **Expose a counter** (`thunderbit_credits_total`) in Prometheus/StatsD so cost per feature is graphable.
- **Tie cost to the distill-vs-extract decision.** This is the biggest lever: extract is **20 credits**, distill is **1** — a 20× difference. If a feature only needs clean text (RAG, summarization, a saved readable copy), use distill. Reserve extract for the structured fields you'll actually query. The reference's worked example: 1,000 pages is 20,000 credits with extract vs 1,000 with distill.
- **Count cache hits.** A `cache_hit_ratio` metric directly translates to credits saved.

---

## MCP vs API in your app

Two surfaces, two jobs:

- **HTTP API** — use this for **backend product features**, exactly as in this guide. It's deterministic, language-agnostic, runs on your servers, and you control auth, validation, caching, and spend. This is what ships inside your app.
- **MCP server** (`@thunderbit/mcp-server`) — this is for **AI assistants and agents** (Claude Desktop/Code, Cursor, Cline). You don't write request bodies; the model calls `thunderbit_*` tools in response to natural language. Great for exploration, internal tooling, and agentic workflows — but it's not how you back a user-facing feature with strict SLAs and cost controls.

Rule of thumb: if a deterministic server endpoint is making the call, use the HTTP API; if an LLM is deciding what to call, use MCP. For building agents on top of Thunderbit, see [AI Agent development](16-ai-agent-development.md).

---

## Responsible use

These guardrails are part of shipping, not an afterthought:

- **Public data only.** Thunderbit targets publicly accessible pages. Never point your feature at login-walled platforms or private/authenticated areas.
- **Respect `robots.txt` and Terms of Service** for any site your users target. Honor disallowed paths and crawl-rate directives.
- **Don't become an open proxy / SSRF vector.** This is the developer-specific obligation: validate and limit user-supplied targets (scheme, private IPs, allowlist) as shown above. An unvalidated "fetch any URL" endpoint is a liability, not a feature.
- **Per-user rate limits.** Cap how many captures each user can run so one account can't drain your credits or hammer a third-party origin.
- **GDPR/CCPA for any personal data.** If your users capture pages containing personal data, you need a lawful basis to process it. Prefer non-personal, company-level public data; drop personal fields you don't need.
- **Throttle.** Use batch with reasonable concurrency and add jitter to scheduled jobs; don't hammer a single origin.
- **Cache and dedupe** to avoid re-fetching unchanged pages — it saves credits and reduces load on the source.

---

## Troubleshooting & FAQ

**My `tb_` key leaked into the client bundle.** Treat it as compromised immediately: **rotate it** at <https://app.thunderbit.com/console/api-keys>, deploy the backend-proxy pattern from this guide, and audit billing for unexpected spend. A leaked key can be used until rotated or out of credits. Never embed it in frontend code, `NEXT_PUBLIC_*`/`VITE_*` vars, or a mobile binary again.

**Extraction returns an empty array.** The page is client-rendered. Set `renderMode:"full"` (headless browser) and raise `waitFor` (up to its 10000 ms ceiling) so lazy content loads; raise `timeout` (extract allows up to 120000 ms). `renderMode:"none"` against an SPA is the usual cause.

**Requests time out.** For a slow single page, switch to the async endpoint (`POST /openapi/v1/async/extract` then poll `GET /openapi/v1/async/extract/{jobId}` every ~3s) so you don't hold an HTTP connection open and trip your load balancer. Raise `timeout` toward the 120000 ms max and add `waitFor`.

**Got a 402 mid-flow.** You're out of credits. Treat it as a hard stop — don't retry, alert ops, top up at <https://thunderbit.com/billing>, and degrade gracefully (return "temporarily unavailable," don't crash the user's request). Trip the circuit breaker so you fail fast during the outage.

**The same URL is costing us twice.** You're missing the cache. Cache by `url` + schema hash + mode with a TTL (see "Caching & dedupe"); a hit costs 0 credits instead of 20. Make sure the cache key includes the schema hash, or different-schema captures will collide.

**Someone is submitting `http://169.254.169.254/` or `http://localhost`.** That's an SSRF attempt. Your `assertSafePublicUrl` helper rejects it (private/link-local/metadata ranges + scheme allowlist). If you don't have that helper in the request path yet, add it before anything else — it's the difference between a feature and a vulnerability.

**401 errors in production.** Bad or missing key. Check `THUNDERBIT_API_KEY` is set in the server's environment/secret manager (not the client), and that it wasn't rotated out from under you. Never retry a 401 — it can't succeed.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status` and re-submit only the `failed` URLs in a fresh batch. Re-running the whole batch double-charges the rows that already succeeded. See the [API reference](docs/thunderbit-api-reference.md) §3.7.
