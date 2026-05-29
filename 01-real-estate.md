# Real Estate Market Intelligence with Thunderbit

Build a self-updating comparables database, track new listings and price drops daily, and roll up neighborhood stats — all from public listing portals, using one AI web-data engine reachable from MCP, HTTP, or CLI.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## The data problem in real estate

Real estate intelligence is a data-plumbing problem disguised as an analysis problem. The pains are concrete:

- **Fragmented portals.** Inventory is spread across dozens of regional and national listing sites, each with its own HTML, its own field names, and its own quirks. There is no single clean feed for "every active 3-bed in this ZIP."
- **Stale comps.** A comparable-sales ("comps") workbook built by hand on Monday is wrong by Friday. Prices change, listings go pending, new inventory appears. A static spreadsheet decays fast.
- **Manual copy-paste.** Analysts and agents routinely copy price, beds, baths, square footage, and lot size into spreadsheets one card at a time. It is slow, error-prone, and impossible to schedule.
- **JS-heavy, map-driven UIs.** Modern portals render listings client-side, lazy-load cards as you scroll a map, and hide attributes behind detail pages. A naive HTTP `GET` returns an empty shell, so you need a real headless browser to see the data a human sees.

The fix is a small, repeatable pipeline: discover the schema once, extract the search-results page into a list of listings, follow each listing's detail link in a batch, store the result, and diff it against yesterday. Thunderbit gives you the extraction engine; this tutorial gives you the pipeline.

---

## What you'll build

- A **comps dataset** with price, beds, baths, sqft, lot size, year built, days-on-market, address, listing-agent name/phone (only where publicly posted), photos URL, and detail-page URL.
- A **daily diff** job that flags *new listings* and *price changes* (especially price drops) since the previous run.
- **Neighborhood roll-ups** — median price and median price-per-sqft per area.
- A two-stage pipeline: **(1)** a search-results page → `suggest_fields` → extract the list of listings with detail URLs; **(2)** **batch-extract** the detail pages for full attributes; **(3)** store + diff against the previous run.
- The same workflow, no-code, via the Thunderbit Chrome extension for non-developer teammates.

---

## Prerequisites

1. **An API key.** Create one at <https://app.thunderbit.com/console/api-keys>. Keys are prefixed `tb_`.
2. **Export it** so the CLI, MCP server, and your scripts all pick it up automatically:

   ```bash
   export THUNDERBIT_API_KEY=tb_your_api_key_here
   ```

3. **Install a surface** (pick what fits your workflow):

   ```bash
   # CLI + SDK (local scripting / shell)
   npm i -g @thunderbit/thunderbit-cli

   # MCP server (AI assistants) — add to your MCP client config; npx pulls it on demand
   npx -y @thunderbit/mcp-server
   ```

For full endpoint, schema, pricing, and error details, keep the [API reference](docs/thunderbit-api-reference.md) open in a tab.

---

## Three ways to call Thunderbit

All three surfaces hit the identical HTTP API underneath, so schemas and field instructions transfer verbatim between them.

| Surface | Entry point | When to use it |
|---------|-------------|----------------|
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and ad-hoc pulls. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. Best for the automated daily pipeline. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, saving/reusing schemas. Best for prototyping and glue scripts. |

A good real-world split: **explore** in MCP, **freeze a schema** with the CLI, **run it nightly** over the HTTP API.

---

## Step 1 — discover the schema

Don't guess field names. Let the AI inspect the page and propose a schema. Throughout this tutorial, treat `https://www.example-realty.com/listings?city=austin-tx` as **a public listings portal you are authorized to access**. The same pattern works on any public listings page — but read the "Responsible use" section first, and check the site's `robots.txt` and Terms of Service before you point a scraper at it.

`suggest_fields` costs **1 credit** and returns an array of field descriptors you can drop straight into an extract schema.

### (a) MCP — let the assistant call the tool

Prompt your assistant:

> Use Thunderbit to suggest fields for `https://www.example-realty.com/listings?city=austin-tx`. Focus on each listing card: price, beds, baths, square footage, address, and the link to the listing's detail page.

The model issues a `thunderbit_suggest_fields` call with arguments like:

```json
{
  "url": "https://www.example-realty.com/listings?city=austin-tx",
  "prompt": "Extract each listing card: price, beds, baths, sqft, address, detail-page link",
  "countryCode": "US"
}
```

> The MCP tool uses `countryCode` (camelCase). The raw HTTP body uses `country_code` (snake_case) — the MCP server translates for you.

### (b) HTTP — `curl`

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/suggest_fields \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.example-realty.com/listings?city=austin-tx",
    "prompt": "Extract each listing card: price, beds, baths, sqft, address, detail-page link",
    "country_code": "US"
  }'
```

### (c) CLI

```bash
thunderbit suggest-fields "https://www.example-realty.com/listings?city=austin-tx" \
  --prompt "Extract each listing card: price, beds, baths, sqft, address, detail-page link" \
  --country-code US
```

### The suggested-fields response

The fields come back at `data` (one array of descriptors):

```json
{
  "data": [
    { "name": "price",        "type": "NUMBER", "instruction": "Asking price in USD, digits only, no currency symbol or commas" },
    { "name": "beds",         "type": "NUMBER", "instruction": "Number of bedrooms" },
    { "name": "baths",        "type": "NUMBER", "instruction": "Number of bathrooms (may be a decimal like 2.5)" },
    { "name": "sqft",         "type": "NUMBER", "instruction": "Interior living area in square feet, digits only" },
    { "name": "address",      "type": "TEXT",   "instruction": "Full street address shown on the card" },
    { "name": "detail_url",   "type": "URL",    "instruction": "Absolute link to the listing's detail page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;` (JS) or `fields = res["data"].get("fields", res["data"])` (Python).

Treat the suggestion as a starting point. You'll usually tighten the `instruction` strings — the single biggest lever on extraction quality.

---

## Step 2 — extract the listings page

Now turn the search-results page into structured rows. The schema is **not** raw JSON Schema; it's the Thunderbit-flavored shape: an array of objects, where each property has an UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction`. Extract costs **20 credits** per page.

For a listings/search page we want enough to identify and locate each property, plus the `detail_url` we'll follow in Step 3:

### (a) HTTP — `curl`

Map-driven and SPA portals render their cards in the browser, so set `renderMode:"full"` (headless browser) and give lazy content a moment with `waitFor`.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.example-realty.com/listings?city=austin-tx",
    "renderMode": "full",
    "waitFor": 2500,
    "timeout": 90000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "price":      { "type": "NUMBER", "instruction": "Asking price in USD, digits only, no currency symbol or commas" },
          "beds":       { "type": "NUMBER", "instruction": "Number of bedrooms" },
          "baths":      { "type": "NUMBER", "instruction": "Number of bathrooms; keep decimals like 2.5" },
          "sqft":       { "type": "NUMBER", "instruction": "Interior living area in square feet, digits only" },
          "address":    { "type": "TEXT",   "instruction": "Full street address shown on the listing card" },
          "detail_url": { "type": "URL",    "instruction": "Absolute link to this listing's detail page" }
        }
      }
    }
  }'
```

Rows come back at `data`:

```json
{
  "data": [
    { "price": 549000, "beds": 3, "baths": 2,   "sqft": 1820, "address": "1204 Maple Ave, Austin, TX 78704", "detail_url": "https://www.example-realty.com/p/1204-maple-ave" },
    { "price": 725000, "beds": 4, "baths": 2.5, "sqft": 2410, "address": "88 Cedar St, Austin, TX 78702",    "detail_url": "https://www.example-realty.com/p/88-cedar-st" }
  ]
}
```

### (b) Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

LISTINGS_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "price":      {"type": "NUMBER", "instruction": "Asking price in USD, digits only, no currency symbol or commas"},
            "beds":       {"type": "NUMBER", "instruction": "Number of bedrooms"},
            "baths":      {"type": "NUMBER", "instruction": "Number of bathrooms; keep decimals like 2.5"},
            "sqft":       {"type": "NUMBER", "instruction": "Interior living area in square feet, digits only"},
            "address":    {"type": "TEXT",   "instruction": "Full street address shown on the listing card"},
            "detail_url": {"type": "URL",    "instruction": "Absolute link to this listing's detail page"},
        },
    },
}

def extract(url, schema, render_mode="full", timeout=90000, wait_for=2500):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema,
        "renderMode": render_mode, "timeout": timeout, "waitFor": wait_for,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

listings = extract("https://www.example-realty.com/listings?city=austin-tx", LISTINGS_SCHEMA)
print(len(listings), "listings")
```

### (c) CLI — save the schema, then reuse it

Save the schema to a file once, then re-run it any time:

```bash
# Save the listings schema to listings.schema.json (an array-of-objects schema as above),
# then extract with it:
thunderbit extract "https://www.example-realty.com/listings?city=austin-tx" \
  --schema listings.schema.json \
  --render-mode full \
  --timeout 90000 \
  -f json > listings.json
```

> **`renderMode` cheat sheet:** `none` = raw HTML fetch (fastest/cheapest render; fine for server-rendered pages); `basic` = light JS; `full` = headless browser for JS-heavy/map SPAs. If a page comes back empty, the first thing to try is `renderMode:"full"` plus a larger `waitFor`. The valid `waitFor` range is `0–10000` ms; `extract` `timeout` is `5000–120000` ms (default `60000`).

---

## Step 3 — batch-extract detail pages

The listings page rarely has everything — lot size, year built, days-on-market, agent contact, and the photos URL usually live on the detail page. Collect the `detail_url` values from Step 2 and process them with **batch extract**, which handles **up to 100 URLs per job** at **20 credits/URL**.

Define a richer detail schema:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "price":         { "type": "NUMBER", "instruction": "Current asking price in USD, digits only" },
      "beds":          { "type": "NUMBER", "instruction": "Number of bedrooms" },
      "baths":         { "type": "NUMBER", "instruction": "Number of bathrooms; keep decimals like 2.5" },
      "sqft":          { "type": "NUMBER", "instruction": "Interior living area in square feet, digits only" },
      "lot_size_sqft": { "type": "NUMBER", "instruction": "Lot size in square feet; convert acres to sqft if shown in acres" },
      "year_built":    { "type": "NUMBER", "instruction": "Four-digit year the home was built" },
      "days_on_market":{ "type": "NUMBER", "instruction": "Days on market / days since listed, digits only" },
      "address":       { "type": "TEXT",   "instruction": "Full street address including city, state, ZIP" },
      "agent_name":    { "type": "TEXT",   "instruction": "Listing agent name ONLY if publicly displayed on the page" },
      "agent_phone":   { "type": "TEXT",   "instruction": "Listing agent phone ONLY if publicly displayed on the page" },
      "photos_url":    { "type": "URL",    "instruction": "URL of the primary/hero listing photo" },
      "listed_date":   { "type": "DATE",   "instruction": "Date the listing was posted, if shown" }
    }
  }
}
```

Because each detail page is one property, the engine returns a single record per URL (still inside the array shape).

### (a) HTTP — create the job, then poll

```bash
# 1. Create the batch job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://www.example-realty.com/p/1204-maple-ave",
      "https://www.example-realty.com/p/88-cedar-st"
    ],
    "schema": { "type": "array", "items": { "type": "object", "properties": { } } },
    "timeout": 90000
  }'
# → { "data": { "id": "batch_xyz789", "status": "pending", "total": 2 } }

# 2. Poll for results (page is 0-based; pageSize default 20, max 100)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/extract/batch_xyz789?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

The batch status response carries job-level status plus a per-URL `results` array, so you can retry only the failures:

```json
{
  "data": {
    "id": "batch_xyz789",
    "status": "completed",
    "total": 2,
    "completed": 2,
    "results": [
      { "url": "https://www.example-realty.com/p/1204-maple-ave", "status": "completed", "data": [ { "price": 549000, "year_built": 1998 } ] },
      { "url": "https://www.example-realty.com/p/88-cedar-st",    "status": "failed",    "error": "timeout" }
    ]
  }
}
```

Job-level `status` is `pending | processing | completed | failed | cancelled`. Per-URL `results[].status` lets you re-submit only those URLs that came back `failed`.

### (b) CLI — batch from a file

`--file` reads one URL per line and ignores blank lines and lines starting with `#`.

```bash
# detail_urls.txt has one detail-page URL per line
thunderbit batch extract --file detail_urls.txt \
  --schema detail.schema.json \
  --timeout 90000 \
  -f json > details.json
```

### (c) MCP

Ask your assistant: *"Use Thunderbit to batch-extract these detail URLs with this schema, then poll until done."* It calls `thunderbit_batch_extract_create` (args `urls`, `schema`, `timeout?`) and then `thunderbit_batch_extract_status` (args `jobId`, `page?`, `pageSize?`) until the job is `completed`.

---

## End-to-end pipeline

Here's one cohesive, runnable Python script that ties it all together: extract the search page, batch-extract details (in chunks of 100), write to **both** CSV and SQLite, then diff against the previous run to surface **new listings** and **price drops**. Error handling follows the reference: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Daily real-estate comps pipeline built on the Thunderbit Open API."""
import os, time, csv, json, random, sqlite3, datetime
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
SEARCH_URL = "https://www.example-realty.com/listings?city=austin-tx"  # a public portal you're authorized to access
DB = "comps.sqlite"

LISTINGS_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "price":      {"type": "NUMBER", "instruction": "Asking price in USD, digits only, no symbol or commas"},
        "beds":       {"type": "NUMBER", "instruction": "Number of bedrooms"},
        "baths":      {"type": "NUMBER", "instruction": "Number of bathrooms; keep decimals like 2.5"},
        "sqft":       {"type": "NUMBER", "instruction": "Interior living area in square feet, digits only"},
        "address":    {"type": "TEXT",   "instruction": "Full street address shown on the card"},
        "detail_url": {"type": "URL",    "instruction": "Absolute link to this listing's detail page"},
    }},
}

DETAIL_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "price":          {"type": "NUMBER", "instruction": "Current asking price in USD, digits only"},
        "beds":           {"type": "NUMBER", "instruction": "Number of bedrooms"},
        "baths":          {"type": "NUMBER", "instruction": "Number of bathrooms; keep decimals like 2.5"},
        "sqft":           {"type": "NUMBER", "instruction": "Interior living area in square feet, digits only"},
        "lot_size_sqft":  {"type": "NUMBER", "instruction": "Lot size in sqft; convert acres to sqft if shown in acres"},
        "year_built":     {"type": "NUMBER", "instruction": "Four-digit year the home was built"},
        "days_on_market": {"type": "NUMBER", "instruction": "Days on market, digits only"},
        "address":        {"type": "TEXT",   "instruction": "Full street address with city, state, ZIP"},
        "agent_name":     {"type": "TEXT",   "instruction": "Listing agent name ONLY if publicly displayed"},
        "agent_phone":    {"type": "TEXT",   "instruction": "Listing agent phone ONLY if publicly displayed"},
        "photos_url":     {"type": "URL",    "instruction": "URL of the primary listing photo"},
        "listed_date":    {"type": "DATE",   "instruction": "Date the listing was posted, if shown"},
    }},
}

class OutOfCredits(Exception):
    """402 — stop the run and alert a human."""

def _post(path, body, attempts=6):
    """POST with backoff on 429/5xx; hard-stop on 402; never retry 401."""
    for i in range(attempts):
        r = requests.post(f"{BASE}{path}", headers=HEADERS, json=body, timeout=140)
        if r.status_code == 401:
            raise RuntimeError("401 invalid/missing API key — check THUNDERBIT_API_KEY")
        if r.status_code == 402:
            raise OutOfCredits("402 out of credits — top up at https://thunderbit.com/billing")
        if r.status_code == 429 or r.status_code >= 500:
            sleep = min(60, 2 ** i) + random.uniform(0, 1)  # exponential backoff + jitter
            time.sleep(sleep)
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"Gave up on {path} after {attempts} attempts")

def _get(path, params=None):
    r = requests.get(f"{BASE}{path}", headers=HEADERS, params=params or {}, timeout=60)
    r.raise_for_status()
    return r.json()

def extract_listings():
    res = _post("/openapi/v1/extract", {
        "url": SEARCH_URL, "schema": LISTINGS_SCHEMA,
        "renderMode": "full", "waitFor": 2500, "timeout": 90000,
    })
    return res["data"]

def batch_extract_details(urls):
    """Process detail URLs in chunks of 100 (the batch ceiling). Returns flat rows."""
    rows = []
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/extract", {
            "urls": chunk, "schema": DETAIL_SCHEMA, "timeout": 90000,
        })["data"]
        job_id = job["id"]
        while True:
            status = _get(f"/openapi/v1/batch/extract/{job_id}",
                          {"page": 0, "pageSize": 100})["data"]
            if status["status"] in ("completed", "failed", "cancelled"):
                break
            time.sleep(3)  # polling is free
        for result in status.get("results", []):
            if result.get("status") != "completed":
                print(f"  ! detail failed: {result['url']} ({result.get('error')})")
                continue
            payload = result.get("data") or []
            record = payload[0] if isinstance(payload, list) and payload else (payload or {})
            record["detail_url"] = result["url"]
            rows.append(record)
    return rows

def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS snapshots (
        run_date TEXT, detail_url TEXT, price REAL, beds REAL, baths REAL,
        sqft REAL, lot_size_sqft REAL, year_built REAL, days_on_market REAL,
        address TEXT, agent_name TEXT, agent_phone TEXT, photos_url TEXT, listed_date TEXT,
        PRIMARY KEY (run_date, detail_url))""")
    con.commit()
    return con

def previous_prices(con):
    """Most recent prior run's prices keyed by detail_url."""
    cur = con.execute("SELECT detail_url, price FROM snapshots WHERE run_date = "
                      "(SELECT MAX(run_date) FROM snapshots)")
    return {url: price for url, price in cur.fetchall()}

def save_snapshot(con, run_date, rows):
    for r in rows:
        con.execute("""INSERT OR REPLACE INTO snapshots VALUES
            (?,?,?,?,?,?,?,?,?,?,?,?,?,?)""", (
            run_date, r.get("detail_url"), r.get("price"), r.get("beds"), r.get("baths"),
            r.get("sqft"), r.get("lot_size_sqft"), r.get("year_built"), r.get("days_on_market"),
            r.get("address"), r.get("agent_name"), r.get("agent_phone"),
            r.get("photos_url"), r.get("listed_date")))
    con.commit()

def write_csv(run_date, rows):
    fields = ["detail_url","price","beds","baths","sqft","lot_size_sqft","year_built",
              "days_on_market","address","agent_name","agent_phone","photos_url","listed_date"]
    with open(f"comps_{run_date}.csv", "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=fields, extrasaction="ignore")
        w.writeheader()
        w.writerows(rows)

def diff(prev, rows):
    new_listings, price_drops = [], []
    for r in rows:
        url, price = r.get("detail_url"), r.get("price")
        if url not in prev:
            new_listings.append(r)
        elif price is not None and prev[url] is not None and price < prev[url]:
            price_drops.append({"detail_url": url, "address": r.get("address"),
                                "old": prev[url], "new": price})
    return new_listings, price_drops

def neighborhood_stats(rows):
    """Median price and median price/sqft across the run (simple market pulse)."""
    import statistics
    prices = [r["price"] for r in rows if r.get("price")]
    ppsf = [r["price"] / r["sqft"] for r in rows if r.get("price") and r.get("sqft")]
    return {
        "median_price": round(statistics.median(prices)) if prices else None,
        "median_price_per_sqft": round(statistics.median(ppsf), 2) if ppsf else None,
        "sample_size": len(rows),
    }

def main():
    run_date = datetime.date.today().isoformat()
    con = init_db()
    prev = previous_prices(con)

    listings = extract_listings()
    detail_urls = [l["detail_url"] for l in listings if l.get("detail_url")]
    print(f"Found {len(detail_urls)} listings on the search page")

    rows = batch_extract_details(detail_urls)
    print(f"Extracted {len(rows)} detail pages")

    write_csv(run_date, rows)
    save_snapshot(con, run_date, rows)

    new_listings, price_drops = diff(prev, rows)
    print(f"\n{len(new_listings)} NEW listings, {len(price_drops)} PRICE DROPS")
    for d in price_drops:
        print(f"  ↓ {d['address']}: {d['old']} → {d['new']} ({d['detail_url']})")
    print("\nNeighborhood stats:", json.dumps(neighborhood_stats(rows)))

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire this to your alerting (PagerDuty/Slack/email)
        raise
```

Run it, and on day two it will print exactly which listings are new and which have dropped in price since the previous snapshot.

---

## No-code version: the Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper** Chrome extension. Non-developer teammates can run the identical workflow without writing code, and the schema they build transfers verbatim to the API.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **AI Suggest Fields.** Open the public listings page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns (price, beds, baths, sqft, address, detail link).
3. **Edit columns** in plain language. Rename, drop, or add columns; tune each column's instruction exactly like the `instruction` strings in the API schema.
4. **Scrape**, then **Scrape Subpages** to follow each row's `detail_url` and pull the full attributes — the no-code equivalent of the two-stage list → detail batch pipeline in Steps 2–3.
5. **Export** to Excel, Google Sheets, Airtable, or Notion, or **schedule** recurring runs.

The recommended flow: design and validate the schema interactively in the extension, then port the field names and instructions to the CLI/API for automation at scale. They map one-to-one.

---

## Automate it

- **Cron.** Run the pipeline nightly. The diff logic already compares against the most recent prior snapshot stored in SQLite:

  ```cron
  # Every day at 06:15, run the comps pipeline and append a daily log
  15 6 * * *  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/comps/pipeline.py >> /var/log/comps.log 2>&1
  ```

- **Batch webhooks instead of polling.** For server-side jobs, supply a callback so Thunderbit notifies you when the batch finishes — no open connection needed:

  ```json
  {
    "urls": ["https://www.example-realty.com/p/1204-maple-ave", "..."],
    "schema": { "type": "array", "items": { "type": "object", "properties": { } } },
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "whatever-signing-secret-you-choose"
    }
  }
  ```

  Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call, then write the results to your store.

- **Store snapshots.** Keep one row per `(run_date, detail_url)` (as the script does). That history powers price-drop alerts, days-on-market trends, and longitudinal neighborhood charts.
- **Alert on price drops.** Pipe the `price_drops` list to Slack/email. A drop on a property that has been on-market for many `days_on_market` is a strong negotiation signal.

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit** *(some docs list it as free — confirm against your plan)*, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**Worked example.** A nightly run over **500 detail pages**:

```
500 detail pages × 20 credits  = 10,000 credits / night
+ 1 search-page extract × 20    =     20 credits
+ 1 suggest_fields (one-time)   =      1 credit
≈ 10,020 credits / night
```

**Extract vs distill — a 20× decision.** Extract (structured JSON) is 20× the price of distill (clean Markdown). If you only need *clean text* — say, to summarize listing descriptions for a RAG index — distill 500 pages for **500 credits** instead of 10,000. Reserve extract for the structured fields you'll actually query (price, sqft, etc.).

**Throttle & cache to cut both cost and load:**

- **Dedupe.** Skip detail pages whose price and key attributes are unchanged from the last snapshot — your SQLite history already knows them. Only re-extract what moved.
- **Stage cheaply.** The search-page extract is one call that yields many `detail_url`s; only batch-extract the details you don't already have fresh.
- **Reasonable concurrency.** Batch caps at 100 URLs/job; chunk larger sets and don't fire many jobs at one origin simultaneously.

---

## Responsible use

These guardrails apply to every workflow in this repo — real estate included.

- **Check `robots.txt` and Terms of Service first.** Before pointing a scraper at any portal, confirm the site permits automated access to the pages you target. Honor disallowed paths and crawl-rate directives.
- **Public data only.** Thunderbit targets publicly accessible pages. Never build workflows against login-walled platforms or private/authenticated areas. Use only the public listing pages you are authorized to access.
- **Personal-data caution.** Listing-agent **name and phone** are personal data. Collect them **only when publicly posted**, and only if you have a lawful basis to process them (GDPR/CCPA). Prefer property- and listing-level facts over personal contact data; if you don't need agent contact info, drop those fields from the schema.
- **Throttle.** Use batch with reasonable concurrency; don't hammer a single origin. Add jitter to scheduled runs.
- **Cache and dedupe** to avoid re-scraping unchanged pages — it saves credits and reduces load on the source.

---

## Troubleshooting & FAQ

**Extraction returns an empty array.** The page is almost certainly client-rendered. Set `renderMode:"full"` (headless browser) and raise `waitFor` (up to its `10000` ms ceiling) so lazy/map content loads. Also raise `timeout` (extract allows up to `120000` ms). If you're on `renderMode:"none"` against an SPA, that's the cause.

**Pagination / "load more" on the search page.** Most portals paginate via a query parameter (`?page=2`) or a map viewport. Enumerate the page URLs and feed them as a list to **batch extract** with the listings schema, then merge. For infinite-scroll pages, increasing `waitFor` loads more cards, but URL-based pagination is more reliable than scrolling.

**Anti-bot / occasional blocks.** Throttle, add jitter between runs, and keep batch concurrency modest. If a specific origin blocks automated access, that is a signal to re-check its ToS and `robots.txt` — back off rather than evade.

**Timeouts on heavy detail pages.** Raise `timeout` toward the `120000` ms max and add `waitFor`. For a single very slow SPA page outside a batch, use the async endpoints (`POST /openapi/v1/async/extract` then poll `GET /openapi/v1/async/extract/{jobId}` every ~3s) so you don't hold an HTTP connection open.

**Geo-specific content / regional portals.** Set `country_code` (HTTP/CLI) or `countryCode` (MCP) to route the request from the right region — e.g. `US`, `GB`, `DE`. The default is `US`. This matters when a portal serves different inventory or currency by visitor location.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status`; re-submit only the `failed` URLs in a fresh batch job. Don't re-run the whole batch — you'd pay credits again for the rows that already succeeded.

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.

**Numbers come back as strings (e.g. "$549,000").** Tighten the `instruction`: "digits only, no currency symbol or commas." For NUMBER fields, the instruction is what coerces the value — brief it like you'd brief a human assistant.
