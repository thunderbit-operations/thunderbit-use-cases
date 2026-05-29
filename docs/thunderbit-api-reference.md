# Thunderbit Open API / MCP / SDK — Canonical Reference

> This is the shared source of truth for every tutorial in this repository.
> All endpoints, request bodies, and response envelopes below are taken from the
> official open-source clients (`@thunderbit/mcp-server` and `@thunderbit/thunderbit-cli`)
> and the Thunderbit developer docs at <https://docs.thunderbit.com>.
>
> If a code sample anywhere in this repo disagrees with this file, **this file wins.**

---

## 1. What Thunderbit ships

Thunderbit exposes the same high-fidelity web-data engine through three surfaces. They
all call the identical HTTP API underneath, so you can mix and match them freely.

| Surface | Package / entry point | Best for |
|---------|-----------------------|----------|
| **HTTP API** | `https://openapi.thunderbit.com` | Any language, servers, cron jobs, CI, webhooks |
| **MCP server** | `@thunderbit/mcp-server` (npx) | AI assistants — Claude Desktop/Code, Cursor, Cline, ChatGPT |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` (npm) | Local scripting, shell pipelines, programmatic Node usage |

There are three core capabilities, plus batch and async variants of each:

1. **Distill** — turn any web page into clean, LLM-ready Markdown.
2. **Extract** — pull structured JSON from a page using a field schema.
3. **Suggest fields** — let the AI inspect a page and propose an extraction schema.

---

## 2. Authentication

Every request authenticates with a bearer token. Get a key from
<https://app.thunderbit.com/console/api-keys> (the docs portal links it as
[thunderbit.com/open-api](https://thunderbit.com/open-api)). Keys are prefixed `tb_`.
Full developer documentation lives at <https://docs.thunderbit.com>.

```
Authorization: Bearer tb_your_api_key_here
```

Store the key in an environment variable — never hard-code it:

```bash
export THUNDERBIT_API_KEY=tb_your_api_key_here
```

Both the MCP server and the CLI read `THUNDERBIT_API_KEY` automatically. Key
resolution order is: explicit `apiKey` argument → `THUNDERBIT_API_KEY` env var → error.

---

## 3. HTTP API

**Base URL:** `https://openapi.thunderbit.com`
**API prefix:** `/openapi/v1`
**Override base URL** (self-host / staging): set `THUNDERBIT_API_BASE_URL`.

All POST bodies are JSON; send `Content-Type: application/json` and `Accept: application/json`.

### 3.1 Endpoint map

| Method | Path | Purpose | Cost |
|--------|------|---------|------|
| `POST` | `/openapi/v1/distill` | Page → Markdown (synchronous) | 1 credit |
| `POST` | `/openapi/v1/extract` | Page → structured JSON (synchronous) | 20 credits |
| `POST` | `/openapi/v1/suggest_fields` | Page → suggested schema | 1 credit |
| `POST` | `/openapi/v1/async/distill` | Submit async distill job | 1 credit |
| `GET`  | `/openapi/v1/async/distill/{jobId}` | Poll async distill | free |
| `POST` | `/openapi/v1/async/extract` | Submit async extract job | 20 credits |
| `GET`  | `/openapi/v1/async/extract/{jobId}` | Poll async extract | free |
| `POST` | `/openapi/v1/batch/distill` | Distill up to 100 URLs | 1 credit / URL |
| `GET`  | `/openapi/v1/batch/distill/{jobId}` | Poll batch distill (`?page=&pageSize=`) | free |
| `POST` | `/openapi/v1/batch/extract` | Extract from up to 100 URLs | 20 credits / URL |
| `GET`  | `/openapi/v1/batch/extract/{jobId}` | Poll batch extract (`?page=&pageSize=`) | free |

> **Rule of thumb:** use the synchronous `/distill` and `/extract` for single pages in
> interactive flows; use `/async/*` when a single page may be slow (heavy SPA) and you
> don't want to hold a connection open; use `/batch/*` whenever you have more than a
> handful of URLs.

### 3.2 Common request parameters

| Field | Type | Endpoints | Notes |
|-------|------|-----------|-------|
| `url` | string (URL) | distill, extract, suggest_fields, async | The page to process. |
| `urls` | string[] (1–100) | batch | The pages to process. |
| `schema` | object | extract, batch/extract, async/extract | Extraction schema (see §5). |
| `prompt` | string | suggest_fields | Natural-language guidance for which fields to propose. |
| `renderMode` | `"none"` \| `"basic"` \| `"full"` | distill, extract, async | `none` = raw HTML fetch (fastest/cheapest to render); `basic` = light JS; `full` = headless browser for JS-heavy SPAs. |
| `timeout` | integer (ms) | all | distill 5000–60000 (default 30000); extract 5000–120000 (default 60000). |
| `waitFor` | integer (ms, 0–10000) | distill, extract | Extra wait after load for lazy/dynamic content. |
| `country_code` | string (ISO-2) | distill, suggest_fields | Geo-route the request (e.g. `US`, `DE`, `GB`). Default `US`. |
| `includeTags` | string[] | distill | Keep only these HTML tags. |
| `excludeTags` | string[] | distill | Drop these HTML tags (e.g. `["nav","footer","aside"]`). |

> Note the snake_case `country_code` in HTTP bodies. The MCP tool exposes it as
> `countryCode` and translates it for you.

### 3.3 Distill — request & response

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/article",
    "renderMode": "full",
    "country_code": "US",
    "excludeTags": ["nav", "footer", "aside"],
    "timeout": 30000
  }'
```

Response:

```json
{
  "data": {
    "markdown": "# Article title\n\nClean body text in Markdown...",
    "url": "https://example.com/article"
  }
}
```

The Markdown lives at `data.markdown`.

### 3.4 Suggest fields — request & response

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/suggest_fields \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/products",
    "prompt": "Extract each product card",
    "country_code": "US"
  }'
```

Response — an array of field descriptors at `data` (each ready to drop into an extract schema):

```json
{
  "data": [
    { "name": "product_name", "type": "TEXT",   "instruction": "The product title" },
    { "name": "price",        "type": "NUMBER", "instruction": "Current price in USD" },
    { "name": "rating",       "type": "NUMBER", "instruction": "Star rating 0-5" },
    { "name": "detail_url",   "type": "URL",    "instruction": "Link to the product page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;`

### 3.5 Extract — request & response

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/products",
    "renderMode": "none",
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "product_name": { "type": "TEXT",   "instruction": "The product title" },
          "price":        { "type": "NUMBER", "instruction": "Current price in USD" },
          "in_stock":     { "type": "TEXT",   "instruction": "Availability text" },
          "detail_url":   { "type": "URL",    "instruction": "Link to the product page" }
        }
      }
    }
  }'
```

Response — rows at `data` (one object per matched record):

```json
{
  "data": [
    { "product_name": "Widget A", "price": 19.99, "in_stock": "In stock", "detail_url": "https://example.com/p/a" },
    { "product_name": "Widget B", "price": 24.50, "in_stock": "2 left",   "detail_url": "https://example.com/p/b" }
  ]
}
```

### 3.6 Async single page (submit + poll)

Use this when a single page render may exceed your HTTP client's patience.

```bash
# 1. Submit
curl -sS https://openapi.thunderbit.com/openapi/v1/async/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "url": "https://heavy-spa.example.com", "schema": { ... } }'
# → { "data": { "id": "job_abc123", "status": "pending" } }

# 2. Poll until status === "completed"
curl -sS https://openapi.thunderbit.com/openapi/v1/async/extract/job_abc123 \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
# → { "data": { "id": "job_abc123", "status": "completed", "data": [ ...rows ] } }
```

`status` transitions through `pending` → `processing` → `completed` | `failed`.
Poll every ~3 seconds. Polling is free.

### 3.7 Batch (up to 100 URLs)

```bash
# 1. Create the job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": ["https://a.example.com", "https://b.example.com"],
    "schema": { "type": "array", "items": { "type": "object", "properties": { ... } } },
    "timeout": 60000
  }'
# → { "data": { "id": "batch_xyz789", "status": "pending", "total": 2 } }

# 2. Poll for results (paginated)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/extract/batch_xyz789?page=0&pageSize=20" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

Batch status response:

```json
{
  "data": {
    "id": "batch_xyz789",
    "status": "completed",
    "total": 2,
    "completed": 2,
    "results": [
      { "url": "https://a.example.com", "status": "completed", "data": [ { } ] },
      { "url": "https://b.example.com", "status": "failed", "error": "timeout" }
    ]
  }
}
```

- `status` (job level): `pending` | `processing` | `completed` | `failed` | `cancelled`.
- Per-URL `results[].status` lets you retry only the failures.
- `page` is **0-based**; `pageSize` defaults to 20, max 100.

**Batch webhooks** — instead of polling, supply a callback so Thunderbit notifies you
when the job finishes:

```json
{
  "urls": ["https://a.example.com", "..."],
  "schema": { },
  "webhook": {
    "url": "https://your-server.example.com/thunderbit/callback",
    "secret": "whatever-signing-secret-you-choose"
  }
}
```

Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call.

---

## 4. Errors

Errors come back with a non-2xx status and a JSON body whose message is in
`error.message`, `message`, or `error`.

| HTTP status | Meaning | What to do |
|-------------|---------|-----------|
| `401` | Invalid / missing API key | Check `THUNDERBIT_API_KEY`; re-issue at the console. |
| `402` | Out of credits | Top up at <https://thunderbit.com/billing>. |
| `429` | Rate-limited | Back off and retry (exponential backoff). |
| `timeout` (client) | Page too slow | Raise `timeout`, set `renderMode:"full"`, or add `waitFor`. |

A production client should: retry `429`/`5xx` with exponential backoff + jitter, treat
`402` as a hard stop (alert a human), and never retry `401`.

---

## 5. Extract schema format

The schema is **not** raw JSON Schema — it is a Thunderbit-flavored shape. The common
form is an array of objects (one object per record on the page):

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "<field_name>": {
        "type": "TEXT | NUMBER | URL | EMAIL | DATE",
        "instruction": "Natural-language description of how to find this field"
      }
    }
  }
}
```

Field types (case matters — use UPPERCASE):

| Type | Use for |
|------|---------|
| `TEXT` | Names, descriptions, free text, statuses |
| `NUMBER` | Prices, counts, ratings, sizes |
| `URL` | Links, image sources, detail-page hrefs |
| `EMAIL` | Email addresses |
| `DATE` | Published dates, deadlines, timestamps |

The `instruction` field is the most important lever for quality: write it the way you'd
brief a human assistant ("the asking price in USD, digits only, no currency symbol").

For a single record (e.g. one company profile page) you can use a flat object schema
instead of an array — `{ "type": "object", "properties": { ... } }`.

---

## 6. MCP server

The MCP server exposes 7 tools over stdio. Install by adding it to your MCP client config.

**Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json` on
macOS, `%APPDATA%\Claude\claude_desktop_config.json` on Windows), Cursor, and Cline all
use the same shape:

```json
{
  "mcpServers": {
    "thunderbit": {
      "command": "npx",
      "args": ["-y", "@thunderbit/mcp-server"],
      "env": { "THUNDERBIT_API_KEY": "tb_your_api_key_here" }
    }
  }
}
```

**Claude Code plugin** (bundles the MCP server + 4 skills):

```bash
claude plugin marketplace add thunderbit-open/thunderbit-mcp-server
claude plugin add thunderbit
export THUNDERBIT_API_KEY=tb_your_api_key_here
```

### 6.1 Tools

| Tool | Cost | Key arguments |
|------|------|---------------|
| `thunderbit_suggest_fields` | 1 credit | `url`, `prompt?`, `countryCode?`, `apiKey?` |
| `thunderbit_distill` | 1 credit | `url`, `renderMode?`, `timeout?`, `waitFor?`, `includeTags?`, `excludeTags?`, `countryCode?`, `apiKey?` |
| `thunderbit_extract` | 20 credits | `url`, `schema`, `renderMode?`, `timeout?`, `waitFor?`, `apiKey?` |
| `thunderbit_batch_distill_create` | 1 credit/URL | `urls`, `timeout?`, `apiKey?` |
| `thunderbit_batch_distill_status` | free | `jobId`, `page?`, `pageSize?`, `apiKey?` |
| `thunderbit_batch_extract_create` | 20 credits/URL | `urls`, `schema`, `timeout?`, `apiKey?` |
| `thunderbit_batch_extract_status` | free | `jobId`, `page?`, `pageSize?`, `apiKey?` |

> The MCP tool uses `countryCode` (camelCase); the raw HTTP body uses `country_code`.

### 6.2 Recommended agent workflow

```
1. thunderbit_suggest_fields   → discover what data the page has
2. thunderbit_extract          → pull it as structured JSON
   OR thunderbit_distill       → get the page as Markdown for RAG/reading
3. thunderbit_batch_*_create   → submit up to 100 URLs
4. thunderbit_batch_*_status   → poll until done
```

In an AI assistant you don't write code — you describe the task and the model calls these
tools. Example prompt: *"Use Thunderbit to suggest fields for this product category page,
then extract every product as a table."*

---

## 7. CLI

```bash
npm i -g @thunderbit/thunderbit-cli
export THUNDERBIT_API_KEY=tb_your_api_key_here
```

Global flags: `-k, --api-key`, `--base-url`, `-f, --format <json|table|markdown>`.

| Command | What it does |
|---------|--------------|
| `thunderbit distill <url> [--render-mode none\|basic\|full] [--timeout ms] [--country-code CC] [--sync]` | Page → Markdown. Async by default; `--sync` forces the synchronous endpoint. |
| `thunderbit suggest-fields <url> [--prompt text] [--country-code CC]` | Print suggested fields. |
| `thunderbit extract <url> [--schema <json-or-file>] [-i] [--prompt text] [--render-mode mode] [--timeout ms] [--save-schema file] [--sync]` | Extract structured data. `-i` opens an interactive field editor; omit `--schema` to auto-suggest first. |
| `thunderbit batch distill [urls...] [--file urls.txt] [--timeout ms] [--no-poll]` | Batch distill; reads one URL per line from `--file`. |
| `thunderbit batch extract [urls...] --schema <json-or-file> [--file urls.txt] [--timeout ms] [--no-poll]` | Batch extract. |

Examples:

```bash
# Markdown to stdout
thunderbit distill https://example.com/article -f markdown

# Discover fields, then save the schema for reuse
thunderbit extract https://example.com/products -i --save-schema products.schema.json

# Re-run extraction with the saved schema, output a table
thunderbit extract https://example.com/products --schema products.schema.json -f table

# Batch extract a list of URLs from a file, write JSON
thunderbit batch extract --file urls.txt --schema products.schema.json -f json > out.json
```

`--file` ignores blank lines and lines starting with `#`.

---

## 8. SDK (programmatic Node + thin clients in any language)

The CLI ships a small `ThunderbitClient` you can import, but because the API is plain
REST, a "SDK" in any language is a thin wrapper over `fetch`/`requests`. Patterns:

### 8.1 Node (programmatic, using fetch)

```js
const BASE = "https://openapi.thunderbit.com";
const KEY = process.env.THUNDERBIT_API_KEY;

async function tb(path, body) {
  const res = await fetch(BASE + path, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });
  if (!res.ok) throw new Error(`Thunderbit ${res.status}: ${await res.text()}`);
  return res.json();
}

const { data } = await tb("/openapi/v1/extract", {
  url: "https://example.com/products",
  schema: { type: "array", items: { type: "object", properties: {
    name:  { type: "TEXT",   instruction: "Product name" },
    price: { type: "NUMBER", instruction: "Price in USD, digits only" },
  } } },
});
console.log(data);
```

### 8.2 Python (requests)

```python
import os, time, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

def extract(url, schema, render_mode="none", timeout=60000):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema, "renderMode": render_mode, "timeout": timeout,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

def batch_extract(urls, schema, poll_every=5):
    job = requests.post(f"{BASE}/openapi/v1/batch/extract", headers=HEADERS,
                        json={"urls": urls, "schema": schema}).json()["data"]
    job_id = job["id"]
    while True:
        status = requests.get(f"{BASE}/openapi/v1/batch/extract/{job_id}",
                              headers=HEADERS, params={"page": 0, "pageSize": 100}
                             ).json()["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            return status
        time.sleep(poll_every)
```

The same thin-wrapper pattern works in Go, Ruby, PHP, Java — POST JSON, read `data`.

---

## 9. Pricing & free tier

- **Distill:** 1 credit. **Suggest fields:** 1 credit *(some Thunderbit docs list field
  suggestion as free — confirm against your plan)*. **Extract:** 20 credits.
- **Batch:** scales per URL (distill 1/URL, extract 20/URL).
- **Status polling:** free.
- **Free tier:** new accounts get a one-time allotment (~600 units) — enough to prototype
  distill and extract before topping up at <https://thunderbit.com/billing>.

**Cost math worked example:** extracting 1,000 product pages = 1,000 × 20 = 20,000
credits. If you only need the text (for RAG/summarization), distill is 20× cheaper:
1,000 × 1 = 1,000 credits. Always ask whether you need *structured fields* (extract) or
just *clean text* (distill) — it's a 20× cost difference.

---

## 10. The Thunderbit product (no-code Chrome extension)

The Open API is the programmatic surface of the same engine that powers the
**Thunderbit AI Web Scraper Chrome extension**. Non-developers on your team can run the
identical workflows without code:

1. Install the **Thunderbit** extension from the Chrome Web Store and create an account
   (a free tier is available — new accounts get a monthly page allowance).
2. Open the target page, click the extension, and hit **AI Suggest Fields** — this is the
   UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns.
3. Edit columns in plain language, then click **Scrape**. Use **Scrape Subpages** to
   follow each row's detail link (the no-code version of a two-stage list→detail pipeline).
4. Export to **Excel, Google Sheets, Airtable, or Notion**, or schedule recurring runs.

Use the extension to design and validate a schema interactively, then port that schema to
the API/CLI for automation at scale. The field names and instructions transfer directly.

---

## 11. Responsible-use guardrails (applies to every tutorial here)

- **Public data only.** Thunderbit targets publicly accessible pages. Do **not** build
  workflows against login-walled platforms (Facebook, Instagram, LinkedIn profile/company
  data, private/authenticated areas). Public review and directory sites
  (Trustpilot, Yelp public pages, Google News, Product Hunt, public marketplaces,
  public company sites, public job boards) are fair game.
- **Respect `robots.txt` and Terms of Service** for the sites you target.
- **Throttle.** Use batch with reasonable concurrency; don't hammer a single origin.
- **Don't collect personal data you don't have a lawful basis to process** (GDPR/CCPA).
  Prefer company-level and public, non-personal data.
- **Cache and dedupe** to avoid re-scraping unchanged pages (saves credits and load).
