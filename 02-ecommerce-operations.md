# E-commerce Operations with Thunderbit: Price & Competitor Monitoring at Scale

Turn any public storefront or marketplace category page into a daily, diff-able price-and-assortment feed — using one AI web-data engine you can drive from an MCP client, plain HTTP, or the CLI.

## The data problem in e-commerce ops

E-commerce operators live and die by data they don't own. Competitor prices change multiple times a day. A minimum-advertised-price (MAP) violation by a reseller can erode your margin before a human notices. Competitors quietly add SKUs to a category you thought you owned, or drop products that signal a discontinued line. Stockouts on the other side of the market are your demand-capture opportunity — but only if you catch them within hours.

The manual version of this job is a person with 40 browser tabs, a spreadsheet, and a caffeine problem. It doesn't scale past a handful of SKUs, it's error-prone, and it's never timely. Worse, modern storefronts render prices, stock badges, and "sale" flags with client-side JavaScript, so a naive HTTP fetch returns an empty `<div>` where the price should be. You need a renderer that executes the page, an AI layer that maps messy markup to clean fields, and a pipeline that snapshots and diffs over time.

That's exactly the shape Thunderbit fits. This tutorial builds the whole loop.

## What you'll build

By the end you'll have:

- A **price tracker** — extract `product_name`, `price`, `sale_price`, `currency`, availability, `sku`, `rating`, `review_count`, `image_url`, and `product_url` from competitor and marketplace pages.
- An **assortment diff** — detect new SKUs competitors added, SKUs they removed, and gaps where they carry products you don't.
- A **review digest** — distill public review pages into clean Markdown and summarize sentiment/themes with an LLM, for ~1/20th the cost of structured extraction.
- **Alerting** — emit price-drop, stockout, and new-SKU events from a daily snapshot diff, wired to Slack/email or a webhook.

Everything runs against the public scraping sandbox `https://books.toscrape.com` (a real, runnable, ToS-friendly demo with a category page → product-detail structure). The *identical pattern* applies to any public storefront you're authorized to monitor — generic Shopify-style stores, public marketplace category pages, etc. Always honor `robots.txt` and Terms of Service (see [Responsible use](#responsible-use)).

## Prerequisites

1. A Thunderbit API key (prefixed `tb_`). Get one at <https://app.thunderbit.com/console/api-keys>.
2. Export it — never hard-code it:

   ```bash
   export THUNDERBIT_API_KEY=tb_your_api_key_here
   ```

3. For the CLI/MCP examples:

   ```bash
   # CLI + SDK
   npm i -g @thunderbit/thunderbit-cli

   # MCP server (run on demand via npx; no global install needed)
   npx -y @thunderbit/mcp-server
   ```

4. For the Python pipeline: Python 3.9+ and `pip install requests`.

Full endpoint, schema, and error semantics live in the [API reference](docs/thunderbit-api-reference.md) — when this tutorial and that file disagree, the reference wins.

## Three ways to call Thunderbit

All three surfaces hit the same HTTP engine, so schemas and field instructions transfer verbatim between them. Pick by context:

| Surface | Entry point | Best for | When to use it |
|---------|-------------|----------|----------------|
| **MCP server** | `@thunderbit/mcp-server` | AI assistants (Claude Desktop/Code, Cursor, Cline) | You're an agent or working inside one; you want to describe the task in natural language and let the model call tools. |
| **HTTP API** | `https://openapi.thunderbit.com` | Any language, servers, cron, CI, webhooks | Production pipelines, scheduled jobs, anything unattended. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell scripts, local prototyping | Quick interactive schema design, one-off pulls, pipeline glue. |

Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and every request authenticates with `Authorization: Bearer tb_...`.

## Step 1 — discover the schema

Don't hand-write field lists. Let the AI inspect the page and propose them (`suggest_fields` costs 1 credit). We'll discover the schema for the books.toscrape.com category listing across all three surfaces.

### Via MCP (AI assistant)

Inside an MCP-enabled assistant, you don't write code — you describe the task and the model calls `thunderbit_suggest_fields`:

> "Use Thunderbit to suggest extraction fields for this product category page: `https://books.toscrape.com/catalogue/category/books_1/index.html`. Treat each product card as a row."

The tool arguments resolve to:

```jsonc
// thunderbit_suggest_fields
{
  "url": "https://books.toscrape.com/catalogue/category/books_1/index.html",
  "prompt": "Extract each product card: title, price, availability, rating, detail link",
  "countryCode": "US"
}
```

### Via HTTP (`curl`)

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/suggest_fields \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://books.toscrape.com/catalogue/category/books_1/index.html",
    "prompt": "Extract each product card: title, price, availability, rating, detail link",
    "country_code": "US"
  }'
```

Note the snake_case `country_code` in the raw HTTP body (the MCP tool exposes it as `countryCode`).

### Via CLI

```bash
thunderbit suggest-fields \
  "https://books.toscrape.com/catalogue/category/books_1/index.html" \
  --prompt "Extract each product card: title, price, availability, rating, detail link" \
  --country-code US
```

### A realistic response

`suggest_fields` returns an array of field descriptors at `data`, each ready to drop into an extract schema:

```json
{
  "data": [
    { "name": "product_name", "type": "TEXT",   "instruction": "The book/product title" },
    { "name": "price",        "type": "NUMBER", "instruction": "Listed price, digits only, no currency symbol" },
    { "name": "availability", "type": "TEXT",   "instruction": "Stock status text, e.g. 'In stock'" },
    { "name": "rating",       "type": "NUMBER", "instruction": "Star rating 0-5" },
    { "name": "product_url",  "type": "URL",    "instruction": "Link to the product detail page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `fields = res["data"].get("fields", res["data"]) if isinstance(res["data"], dict) else res["data"]`.

## Step 2 — extract a category page

Now extend the suggested fields into the full e-commerce schema we want and run `extract` (20 credits). The schema is a Thunderbit-flavored shape: an `array` of `object`s, one object per product card. Field types are UPPERCASE — use `NUMBER` for anything you'll do math or comparisons on (prices, counts, ratings).

```jsonc
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "product_name":  { "type": "TEXT",   "instruction": "The product/book title, plain text" },
      "price":         { "type": "NUMBER", "instruction": "Current/listed price, digits and decimal only, no currency symbol" },
      "sale_price":    { "type": "NUMBER", "instruction": "Discounted price if shown, else leave empty; digits only" },
      "currency":      { "type": "TEXT",   "instruction": "ISO currency code or symbol shown near the price, e.g. GBP, USD, £" },
      "availability":  { "type": "TEXT",   "instruction": "Raw stock text, e.g. 'In stock', 'Out of stock', '2 available'" },
      "sku":           { "type": "TEXT",   "instruction": "Product SKU / UPC / catalogue code if present" },
      "rating":        { "type": "NUMBER", "instruction": "Star rating from 0 to 5" },
      "review_count":  { "type": "NUMBER", "instruction": "Number of reviews/ratings, integer" },
      "image_url":     { "type": "URL",    "instruction": "Absolute URL of the product thumbnail image" },
      "product_url":   { "type": "URL",    "instruction": "Absolute URL of the product detail page" }
    }
  }
}
```

### Via HTTP (`curl`)

books.toscrape.com is static HTML, so `renderMode: "none"` is fine and cheapest to render:

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://books.toscrape.com/catalogue/category/books_1/index.html",
    "renderMode": "none",
    "schema": { "type": "array", "items": { "type": "object", "properties": {
      "product_name": { "type": "TEXT",   "instruction": "The product/book title" },
      "price":        { "type": "NUMBER", "instruction": "Listed price, digits only" },
      "availability": { "type": "TEXT",   "instruction": "Raw stock text" },
      "rating":       { "type": "NUMBER", "instruction": "Star rating 0-5" },
      "product_url":  { "type": "URL",    "instruction": "Absolute URL of the product detail page" }
    } } }
  }'
```

Response — rows at `data`, one object per product:

```json
{
  "data": [
    { "product_name": "A Light in the Attic", "price": 51.77, "availability": "In stock", "rating": 3, "product_url": "https://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html" },
    { "product_name": "Tipping the Velvet",    "price": 53.74, "availability": "In stock", "rating": 1, "product_url": "https://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html" }
  ]
}
```

### JS-rendered prices

Real Shopify-style and marketplace stores inject prices/stock with client-side JS. For those, switch the renderer to a headless browser and give late-loading content a moment:

```jsonc
{
  "url": "https://some-shopify-store.example.com/collections/all",
  "renderMode": "full",   // headless browser for JS-heavy SPAs
  "waitFor": 2500,         // extra ms after load for lazy price/stock widgets
  "schema": { /* same array schema as above */ }
}
```

### Via Python (SDK pattern)

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

PRODUCT_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "product_name": {"type": "TEXT",   "instruction": "The product/book title"},
        "price":        {"type": "NUMBER", "instruction": "Listed price, digits only"},
        "availability": {"type": "TEXT",   "instruction": "Raw stock text"},
        "rating":       {"type": "NUMBER", "instruction": "Star rating 0-5"},
        "product_url":  {"type": "URL",    "instruction": "Absolute product detail URL"},
    }},
}

def extract(url, schema, render_mode="none", timeout=60000):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema, "renderMode": render_mode, "timeout": timeout,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

rows = extract("https://books.toscrape.com/catalogue/category/books_1/index.html", PRODUCT_SCHEMA)
print(len(rows), "products")
```

### Via CLI (with schema reuse)

Design the schema interactively once, save it, then re-run unattended:

```bash
# Discover + edit fields interactively, save the schema for reuse
thunderbit extract \
  "https://books.toscrape.com/catalogue/category/books_1/index.html" \
  -i --save-schema products.schema.json

# Re-run with the saved schema, render as a table
thunderbit extract \
  "https://books.toscrape.com/catalogue/category/books_1/index.html" \
  --schema products.schema.json -f table
```

## Step 3 — batch-extract product detail pages

Category pages give you the product list and each `product_url`. Detail pages give you the authoritative price, SKU, full availability, and review counts. Gather the URLs, then batch-extract up to 100 at a time (20 credits/URL).

### Via HTTP

```bash
# 1. Create the batch job (urls come from Step 2's product_url values)
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html",
      "https://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html"
    ],
    "schema": { "type": "array", "items": { "type": "object", "properties": {
      "product_name": { "type": "TEXT",   "instruction": "The product title on the detail page" },
      "price":        { "type": "NUMBER", "instruction": "Price incl. tax, digits only" },
      "sku":          { "type": "TEXT",   "instruction": "Product code / UPC" },
      "availability": { "type": "TEXT",   "instruction": "Stock text e.g. 'In stock (22 available)'" },
      "review_count": { "type": "NUMBER", "instruction": "Number of reviews, integer" }
    } } },
    "timeout": 60000
  }'
# → { "data": { "id": "batch_xyz789", "status": "pending", "total": 2 } }

# 2. Poll for results (page is 0-based; pageSize defaults to 20, max 100)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/extract/batch_xyz789?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

The status payload carries per-URL results so you can retry only failures:

```json
{
  "data": {
    "id": "batch_xyz789", "status": "completed", "total": 2, "completed": 2,
    "results": [
      { "url": "https://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html", "status": "completed", "data": [ { "product_name": "A Light in the Attic", "price": 51.77, "sku": "a897fe39b1053632", "availability": "In stock (22 available)", "review_count": 0 } ] },
      { "url": "https://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html",   "status": "failed", "error": "timeout" }
    ]
  }
}
```

### Via CLI

```bash
thunderbit batch extract --file urls.txt --schema detail.schema.json -f json > details.json
```

`--file` reads one URL per line and ignores blank lines and lines starting with `#`.

## End-to-end price-monitoring pipeline

This single runnable script does the full loop: extract the category list → batch-extract detail pages → write a daily snapshot to SQLite → diff against yesterday → emit price-drop, stockout, and new-SKU events. It includes the production error handling the [reference](docs/thunderbit-api-reference.md#4-errors) prescribes: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402`, never retry `401`.

```python
import os, time, random, sqlite3, datetime, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
DB = "competitor_prices.db"

LIST_URL = "https://books.toscrape.com/catalogue/category/books_1/index.html"

LIST_SCHEMA = {"type": "array", "items": {"type": "object", "properties": {
    "product_name": {"type": "TEXT",   "instruction": "The product/book title"},
    "price":        {"type": "NUMBER", "instruction": "Listed price, digits only"},
    "availability": {"type": "TEXT",   "instruction": "Raw stock text"},
    "product_url":  {"type": "URL",    "instruction": "Absolute product detail URL"},
}}}

DETAIL_SCHEMA = {"type": "array", "items": {"type": "object", "properties": {
    "product_name": {"type": "TEXT",   "instruction": "The product title"},
    "price":        {"type": "NUMBER", "instruction": "Price incl. tax, digits only"},
    "sku":          {"type": "TEXT",   "instruction": "Product code / UPC"},
    "availability": {"type": "TEXT",   "instruction": "Stock text"},
    "review_count": {"type": "NUMBER", "instruction": "Number of reviews, integer"},
}}}


class OutOfCredits(Exception): ...


def call(method, path, **kwargs):
    """POST/GET with backoff. 402 -> hard stop, 401 -> never retry, 429/5xx -> retry."""
    for attempt in range(6):
        r = requests.request(method, f"{BASE}{path}", headers=HEADERS, timeout=130, **kwargs)
        if r.status_code == 401:
            raise RuntimeError("401: bad/missing THUNDERBIT_API_KEY — do not retry")
        if r.status_code == 402:
            raise OutOfCredits("402: out of credits — top up at thunderbit.com/billing")
        if r.status_code == 429 or r.status_code >= 500:
            sleep = min(2 ** attempt + random.random(), 30)
            time.sleep(sleep)
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"{method} {path} failed after retries")


def extract_list(url):
    return call("POST", "/openapi/v1/extract",
                json={"url": url, "renderMode": "none", "schema": LIST_SCHEMA})["data"]


def batch_extract(urls, schema, poll_every=5):
    job = call("POST", "/openapi/v1/batch/extract",
               json={"urls": urls, "schema": schema, "timeout": 60000})["data"]
    job_id = job["id"]
    while True:
        status = call("GET", f"/openapi/v1/batch/extract/{job_id}",
                      params={"page": 0, "pageSize": 100})["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            return status
        time.sleep(poll_every)


def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS snapshots (
        snapshot_date TEXT, product_url TEXT, product_name TEXT,
        price REAL, availability TEXT, sku TEXT, review_count INTEGER,
        PRIMARY KEY (snapshot_date, product_url))""")
    con.commit()
    return con


def save_snapshot(con, day, rows):
    for r in rows:
        con.execute("INSERT OR REPLACE INTO snapshots VALUES (?,?,?,?,?,?,?)",
                    (day, r.get("product_url"), r.get("product_name"), r.get("price"),
                     r.get("availability"), r.get("sku"), r.get("review_count")))
    con.commit()


def load_day(con, day):
    cur = con.execute(
        "SELECT product_url, product_name, price, availability FROM snapshots WHERE snapshot_date=?",
        (day,))
    return {row[0]: {"product_name": row[1], "price": row[2], "availability": row[3]}
            for row in cur.fetchall()}


def in_stock(text):
    return bool(text) and "out of stock" not in text.lower()


def diff(today, yesterday):
    events = []
    for url, t in today.items():
        y = yesterday.get(url)
        if y is None:
            events.append(("NEW_SKU", url, t["product_name"], t["price"]))
            continue
        if y["price"] and t["price"] and t["price"] < y["price"]:
            drop = round((y["price"] - t["price"]) / y["price"] * 100, 1)
            events.append(("PRICE_DROP", url, t["product_name"], f"{y['price']} -> {t['price']} (-{drop}%)"))
        if in_stock(y["availability"]) and not in_stock(t["availability"]):
            events.append(("STOCKOUT", url, t["product_name"], t["availability"]))
    for url, y in yesterday.items():
        if url not in today:
            events.append(("REMOVED_SKU", url, y["product_name"], None))
    return events


def run():
    con = init_db()
    today = datetime.date.today().isoformat()
    yesterday = (datetime.date.today() - datetime.timedelta(days=1)).isoformat()

    listing = extract_list(LIST_URL)
    urls = [r["product_url"] for r in listing if r.get("product_url")][:100]

    # Authoritative detail pass (batch). Retry only the URLs that failed.
    status = batch_extract(urls, DETAIL_SCHEMA)
    rows, failed = [], []
    for res in status.get("results", []):
        if res["status"] == "completed" and res.get("data"):
            row = res["data"][0]
            row["product_url"] = res["url"]
            rows.append(row)
        else:
            failed.append(res["url"])
    if failed:
        retry = batch_extract(failed, DETAIL_SCHEMA)
        for res in retry.get("results", []):
            if res["status"] == "completed" and res.get("data"):
                row = res["data"][0]
                row["product_url"] = res["url"]
                rows.append(row)

    save_snapshot(con, today, rows)
    events = diff(load_day(con, today), load_day(con, yesterday))
    for kind, url, name, detail in events:
        print(f"[{kind}] {name} | {detail} | {url}")
    return events


if __name__ == "__main__":
    run()
```

Swap the SQLite sink for CSV, BigQuery, or your warehouse — the diff logic is sink-agnostic. The `events` list is what you forward to alerting (next section).

## Review mining with distill

For sentiment and theme analysis you don't need structured fields — you need clean text. That's `distill` (1 credit), which is **20× cheaper than `extract`** (20 credits). Distill a public review page to Markdown, then hand it to an LLM.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.trustpilot.com/review/example-store.com",
    "renderMode": "full",
    "country_code": "US",
    "excludeTags": ["nav", "footer", "aside"],
    "timeout": 30000
  }'
```

Response — Markdown at `data.markdown`:

```json
{ "data": { "markdown": "# Reviews for Example Store\n\n★★★★★ Fast shipping...\n\n★★☆☆☆ Item arrived damaged...", "url": "https://www.trustpilot.com/review/example-store.com" } }
```

Feed that Markdown to any LLM:

```python
md = call("POST", "/openapi/v1/distill",
          json={"url": review_url, "renderMode": "full",
                "excludeTags": ["nav", "footer", "aside"]})["data"]["markdown"]

prompt = (
    "You are an e-commerce analyst. From these public reviews, return: "
    "(1) overall sentiment, (2) top 3 praise themes, (3) top 3 complaint themes, "
    "(4) any recurring product-quality or shipping issues.\n\n" + md
)
# summary = your_llm.complete(prompt)
```

**Distill vs extract — decide by output shape.** If you need *rows with typed fields you'll compare/aggregate* (prices, SKUs, stock), use `extract`. If you need *clean prose to read or summarize* (reviews, descriptions, policies), use `distill` and save 95% of the credits. For batch review mining, `/batch/distill` runs up to 100 URLs at 1 credit each.

## No-code version: Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper Chrome extension**. Non-developers on your team can run the identical workflow without code:

1. Install **Thunderbit** from the Chrome Web Store and **sign in / create an account** (a free tier is available — new accounts get a monthly page allowance).
2. Open the competitor category page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns.
3. Edit columns in plain language (rename `price`, add `sale_price`, tweak instructions), then click **Scrape**.
4. Use **Scrape Subpages** to follow each row's `product_url` into the detail page — the no-code version of the Step 2 → Step 3 list→detail pipeline.
5. Export to **Excel, Google Sheets, Airtable, or Notion**, or **schedule recurring runs** for daily monitoring.

The field names and instructions transfer directly between the extension and the API/CLI — design a schema interactively in the extension, then port it to the pipeline for scale.

## Automate it

Wrap the pipeline in a scheduler and route the `events` to your team:

```bash
# crontab -e — run the monitor every morning at 06:30
30 6 * * *  THUNDERBIT_API_KEY=tb_... /usr/bin/python3 /opt/ecom/price_monitor.py >> /var/log/price_monitor.log 2>&1
```

For large jobs, prefer **batch webhooks** over polling — Thunderbit POSTs the completed job to your endpoint:

```jsonc
{
  "urls": ["https://store.example.com/p/1", "..."],
  "schema": { /* DETAIL_SCHEMA */ },
  "webhook": {
    "url": "https://your-server.example.com/thunderbit/callback",
    "secret": "your-signing-secret"
  }
}
```

Verify the `secret` on receipt before trusting the payload. Then turn diff events into alerts:

```python
import json, requests

def alert_slack(events, webhook_url):
    if not events:
        return
    lines = [f"*{k}* — {name}: {detail}" for k, _url, name, detail in events]
    requests.post(webhook_url, data=json.dumps({"text": "\n".join(lines)}),
                  headers={"Content-Type": "application/json"})
```

Store each day's snapshot (SQLite/warehouse) so the diff has a baseline and you can chart price history over time.

## Cost & scale

Credit math drives every design decision (extract = 20 credits, distill = 1, suggest_fields = 1, polling = free):

| Job | Volume | Endpoint | Daily credits |
|-----|--------|----------|---------------|
| Price/stock detail pass | 2,000 SKUs | `extract` (20) | 40,000 |
| Same, if you only need text | 2,000 SKUs | `distill` (1) | 2,000 |
| Review mining | 200 review pages | `batch/distill` (1) | 200 |
| Schema discovery | 1 page | `suggest_fields` (1) | 1 |

Practical levers:

- **Use distill where you can.** Reviews, descriptions, and policy pages don't need typed fields — that's a 20× saving.
- **Cache unchanged pages.** Skip re-extracting SKUs whose listing-page price and stock are byte-identical to yesterday; only spend `extract` credits on the detail pass for changed rows.
- **Throttle.** Batch caps at 100 URLs/job; run jobs sequentially rather than blasting one origin, and back off on `429`.
- **Tier your cadence.** Monitor your top-200 competitive SKUs hourly and the long tail daily, rather than everything at the same frequency.

## Responsible use

These guardrails apply to every workflow here:

- **Public data only.** Target publicly accessible pages. Do **not** build workflows against login-walled platforms (Facebook, Instagram, LinkedIn, private/authenticated areas). Public marketplaces, public storefronts, and public review sites (Trustpilot, Yelp public pages) are fair game.
- **Respect `robots.txt` and Terms of Service** for every site you monitor.
- **Throttle politely.** Don't hammer a single origin; use batch with reasonable concurrency.
- **Don't collect personal data you lack a lawful basis to process** (GDPR/CCPA). Prefer product- and company-level, non-personal data — prices, stock, SKUs, aggregate review themes, not individual reviewer identities.
- **Cache and dedupe** to avoid re-scraping unchanged pages — it saves credits and reduces load on the target.

## Troubleshooting & FAQ

**Prices come back empty / null.** The store renders prices with JavaScript. Set `renderMode: "full"` (headless browser) and add `waitFor: 2000`–`3000` (ms) so lazy price/stock widgets finish loading. If a single SPA page is slow, raise `timeout` (extract allows up to 120000 ms) or use the async endpoint `/openapi/v1/async/extract` and poll.

**Currency parsing is messy.** Keep `price` as `NUMBER` with the instruction "digits and decimal only, no currency symbol", and add a separate `currency` `TEXT` field for the symbol/code. This avoids `NUMBER` fields choking on "£51.77" and keeps math clean.

**Pagination / infinite scroll.** For numbered pagination, build the page URLs (`.../page-2.html`, `?p=2`) and feed them all into `batch/extract`. For infinite scroll, `renderMode: "full"` plus a generous `waitFor` captures the initially loaded set; if you need more, paginate via the site's underlying page/offset URLs rather than scrolling.

**Anti-bot blocks.** Lower your request rate, run batch jobs sequentially, and respect the site's `robots.txt`. If a site clearly forbids scraping in its ToS, don't monitor it — pick a source you're authorized to use.

**Geo-specific pricing.** Many marketplaces show different prices/availability by region. Set `country_code` (HTTP body) / `countryCode` (MCP) to the ISO-2 market you care about, e.g. `"DE"` or `"GB"`. Run one snapshot per market to track regional price differences.

**Some batch URLs fail with `timeout`.** Per-URL `results[].status` lets you retry only the failures (the pipeline above does this). Bump `timeout` and `renderMode` on the retry pass for the stubborn pages.

**401 / 402 / 429.** `401` = bad/missing `THUNDERBIT_API_KEY` (never retry — re-issue at the console). `402` = out of credits (hard stop; top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter). See [Errors](docs/thunderbit-api-reference.md#4-errors).
