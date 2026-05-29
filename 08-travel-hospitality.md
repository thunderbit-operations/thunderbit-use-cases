# Travel & Hospitality Rate Intelligence with Thunderbit

Track hotel and flight rates across public listing pages, watch availability and sellouts change by the day, and roll up guest-review sentiment — all from public travel pages, using one AI web-data engine reachable from MCP, HTTP, or CLI.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## The data problem in travel

Travel intelligence looks like an analysis problem — "is our property priced right for next weekend?" — but it's really a data-plumbing problem. The pains are concrete:

- **Rates are a moving target.** A nightly rate isn't one number. It changes by *date* (a Tuesday in March is not a Saturday in July), by *market* (the same room costs differently to a visitor from the US vs. Germany), and by *currency* (USD vs. EUR vs. GBP). Yesterday's rate-shop is wrong by this afternoon.
- **Fragmented OTAs.** Inventory and pricing are scattered across dozens of online travel agencies (OTAs), brand sites, and metasearch pages, each with its own HTML, field names, and quirks. There is no single clean feed for "every 3-star property in this city for these dates."
- **JS-heavy date-picker SPAs.** Modern booking pages render prices client-side *after* you pick check-in/check-out dates. A naive HTTP `GET` returns an empty shell with no rates in it — you need a real headless browser that loads the page the way a human's browser does.
- **Manual rate-shopping doesn't scale.** Revenue managers and analysts copy nightly rates, availability, and review scores into spreadsheets one tab at a time. It's slow, error-prone, can't run overnight, and can't watch fifty properties at once.

The fix is a small, repeatable pipeline: discover the schema once, extract the search-results page into a list of properties with their rates, follow each property's detail link in a batch, store the result as a dated snapshot, and diff it against yesterday. Thunderbit gives you the extraction engine; this tutorial gives you the pipeline.

---

## What you'll build

- A **rate tracker** with property name, room type, nightly rate, currency, check-in/check-out dates, star/guest rating, review count, and detail-page URL.
- An **availability / sellout monitor** — a daily diff that flags *rate drops*, *rate hikes*, *new inventory*, and *sellouts* (a property that was bookable yesterday and is gone today) for a given date range and market.
- A **review-sentiment digest** — distill public review pages into clean Markdown, then feed that text to an LLM for themes and sentiment.
- A multi-stage pipeline: **(1)** a search-results page → `suggest_fields` → extract the list of properties with detail URLs; **(2)** **batch-extract** the detail pages for full rate breakdowns; **(3)** store a dated snapshot + diff against the previous run.
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
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and ad-hoc rate-shops. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. Best for the automated daily rate tracker. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, saving/reusing schemas. Best for prototyping and glue scripts. |

A good real-world split: **explore** in MCP, **freeze a schema** with the CLI, **run it nightly** over the HTTP API — one job per market.

---

## Step 1 — discover the schema

Don't guess field names. Let the AI inspect the page and propose a schema. Throughout this tutorial, treat `https://www.example-travel.com/hotels?city=lisbon&checkin=2026-06-12&checkout=2026-06-14` as **a public travel listings page you are authorized to access**. The same pattern works on any public OTA, hotel, or airline listing page — but read the "Responsible use" section first. **Many OTAs explicitly restrict automated access**, so check the site's `robots.txt` and Terms of Service before you point a scraper at it.

`suggest_fields` costs **1 credit** and returns an array of field descriptors you can drop straight into an extract schema.

### (a) MCP — let the assistant call the tool

Prompt your assistant:

> Use Thunderbit to suggest fields for `https://www.example-travel.com/hotels?city=lisbon&checkin=2026-06-12&checkout=2026-06-14`. Focus on each property card: property name, room type, nightly rate, currency, guest rating, review count, and the link to the property's detail page.

The model issues a `thunderbit_suggest_fields` call with arguments like:

```json
{
  "url": "https://www.example-travel.com/hotels?city=lisbon&checkin=2026-06-12&checkout=2026-06-14",
  "prompt": "Extract each property card: property name, room type, nightly rate, currency, rating, review count, detail-page link",
  "countryCode": "PT"
}
```

> The MCP tool uses `countryCode` (camelCase). The raw HTTP body uses `country_code` (snake_case) — the MCP server translates for you. Set it to the market whose rates and currency you want: `PT` for a Lisbon shopper paying in EUR, `US` for the US-market price, and so on.

### (b) HTTP — `curl`

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/suggest_fields \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.example-travel.com/hotels?city=lisbon&checkin=2026-06-12&checkout=2026-06-14",
    "prompt": "Extract each property card: property name, room type, nightly rate, currency, rating, review count, detail-page link",
    "country_code": "PT"
  }'
```

### (c) CLI

```bash
thunderbit suggest-fields "https://www.example-travel.com/hotels?city=lisbon&checkin=2026-06-12&checkout=2026-06-14" \
  --prompt "Extract each property card: property name, room type, nightly rate, currency, rating, review count, detail-page link" \
  --country-code PT
```

### The suggested-fields response

The fields come back at `data` (one array of descriptors):

```json
{
  "data": [
    { "name": "property_name", "type": "TEXT",   "instruction": "Hotel or property name shown on the card" },
    { "name": "room_type",     "type": "TEXT",   "instruction": "Lowest-priced room/rate plan name shown, e.g. 'Standard Double'" },
    { "name": "nightly_rate",  "type": "NUMBER", "instruction": "Per-night price, digits only, no currency symbol or commas" },
    { "name": "currency",      "type": "TEXT",   "instruction": "ISO currency code or symbol shown next to the price, e.g. EUR or €" },
    { "name": "rating",        "type": "NUMBER", "instruction": "Guest review score as shown (e.g. 8.6 or 4.5)" },
    { "name": "review_count",  "type": "NUMBER", "instruction": "Number of reviews, digits only" },
    { "name": "detail_url",    "type": "URL",    "instruction": "Absolute link to the property's detail page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;` (JS) or `fields = res["data"].get("fields", res["data"])` (Python).

Treat the suggestion as a starting point. You'll usually tighten the `instruction` strings — the single biggest lever on extraction quality (especially for rate and currency parsing).

---

## Step 2 — extract the results page

Now turn the search-results page into structured rows. The schema is **not** raw JSON Schema; it's the Thunderbit-flavored shape: an array of objects, where each property has an UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction`. Extract costs **20 credits** per page.

For a search/results page we want enough to identify each property and its headline rate, plus the `detail_url` we'll follow in Step 3. We also carry the `check_in`/`check_out` we requested so every row is anchored to a date range.

### (a) HTTP — `curl`

Booking pages compute rates client-side after the date picker resolves, so set `renderMode:"full"` (headless browser) and give the date-driven content a moment with `waitFor`. Use `country_code` so the rates and currency match the market you care about.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.example-travel.com/hotels?city=lisbon&checkin=2026-06-12&checkout=2026-06-14",
    "renderMode": "full",
    "waitFor": 3000,
    "timeout": 90000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "property_name": { "type": "TEXT",   "instruction": "Hotel or property name shown on the card" },
          "room_type":     { "type": "TEXT",   "instruction": "Lowest-priced room/rate plan name shown" },
          "nightly_rate":  { "type": "NUMBER", "instruction": "Per-night price, digits only, no currency symbol or commas" },
          "currency":      { "type": "TEXT",   "instruction": "ISO currency code or symbol next to the price, e.g. EUR or €" },
          "check_in":      { "type": "DATE",   "instruction": "Check-in date for this rate in YYYY-MM-DD; use 2026-06-12 if not shown per-card" },
          "check_out":     { "type": "DATE",   "instruction": "Check-out date for this rate in YYYY-MM-DD; use 2026-06-14 if not shown per-card" },
          "rating":        { "type": "NUMBER", "instruction": "Guest review score as shown (e.g. 8.6 or 4.5)" },
          "review_count":  { "type": "NUMBER", "instruction": "Number of reviews, digits only" },
          "detail_url":    { "type": "URL",    "instruction": "Absolute link to this property's detail page" }
        }
      }
    }
  }'
```

Rows come back at `data`:

```json
{
  "data": [
    { "property_name": "Hotel Alfama Vista", "room_type": "Standard Double", "nightly_rate": 142, "currency": "EUR", "check_in": "2026-06-12", "check_out": "2026-06-14", "rating": 8.6, "review_count": 1284, "detail_url": "https://www.example-travel.com/h/alfama-vista" },
    { "property_name": "Baixa Boutique Inn",  "room_type": "Queen Room",      "nightly_rate": 98,  "currency": "EUR", "check_in": "2026-06-12", "check_out": "2026-06-14", "rating": 9.1, "review_count": 642,  "detail_url": "https://www.example-travel.com/h/baixa-boutique" }
  ]
}
```

### (b) Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

RESULTS_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "property_name": {"type": "TEXT",   "instruction": "Hotel or property name shown on the card"},
            "room_type":     {"type": "TEXT",   "instruction": "Lowest-priced room/rate plan name shown"},
            "nightly_rate":  {"type": "NUMBER", "instruction": "Per-night price, digits only, no currency symbol or commas"},
            "currency":      {"type": "TEXT",   "instruction": "ISO currency code or symbol next to the price, e.g. EUR or €"},
            "check_in":      {"type": "DATE",   "instruction": "Check-in date in YYYY-MM-DD; use 2026-06-12 if not shown per-card"},
            "check_out":     {"type": "DATE",   "instruction": "Check-out date in YYYY-MM-DD; use 2026-06-14 if not shown per-card"},
            "rating":        {"type": "NUMBER", "instruction": "Guest review score as shown (e.g. 8.6 or 4.5)"},
            "review_count":  {"type": "NUMBER", "instruction": "Number of reviews, digits only"},
            "detail_url":    {"type": "URL",    "instruction": "Absolute link to this property's detail page"},
        },
    },
}

def extract(url, schema, render_mode="full", timeout=90000, wait_for=3000):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema,
        "renderMode": render_mode, "timeout": timeout, "waitFor": wait_for,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

url = "https://www.example-travel.com/hotels?city=lisbon&checkin=2026-06-12&checkout=2026-06-14"
properties = extract(url, RESULTS_SCHEMA)
print(len(properties), "properties")
```

### (c) CLI — save the schema, then reuse it

Save the schema to a file once, then re-run it any time (for example, once per market):

```bash
# Save the results schema to rates.schema.json (an array-of-objects schema as above),
# then extract with it:
thunderbit extract "https://www.example-travel.com/hotels?city=lisbon&checkin=2026-06-12&checkout=2026-06-14" \
  --schema rates.schema.json \
  --render-mode full \
  --timeout 90000 \
  -f json > rates.json
```

> **`renderMode` cheat sheet:** `none` = raw HTML fetch (fastest/cheapest render; fine for server-rendered pages); `basic` = light JS; `full` = headless browser for JS-heavy date-picker SPAs. If a page comes back empty, the first thing to try is `renderMode:"full"` plus a larger `waitFor`. The valid `waitFor` range is `0–10000` ms; `extract` `timeout` is `5000–120000` ms (default `60000`). Booking pages almost always need `full`.

---

## Step 3 — batch-extract detail pages

The results page rarely has everything — the full room-type breakdown, taxes and fees, cancellation terms, exact availability ("only 2 rooms left"), and the canonical review count usually live on the property detail page. Collect the `detail_url` values from Step 2 and process them with **batch extract**, which handles **up to 100 URLs per job** at **20 credits/URL**.

Define a richer detail schema:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "property_name": { "type": "TEXT",   "instruction": "Full property name as shown on the detail page" },
      "room_type":     { "type": "TEXT",   "instruction": "Name of the cheapest available room/rate plan for the selected dates" },
      "nightly_rate":  { "type": "NUMBER", "instruction": "Lowest per-night rate for the selected dates, digits only, no symbol or commas" },
      "currency":      { "type": "TEXT",   "instruction": "ISO currency code or symbol shown next to the rate, e.g. EUR or €" },
      "total_price":   { "type": "NUMBER", "instruction": "Total stay price for the date range incl. taxes/fees if shown, digits only" },
      "check_in":      { "type": "DATE",   "instruction": "Check-in date in YYYY-MM-DD" },
      "check_out":     { "type": "DATE",   "instruction": "Check-out date in YYYY-MM-DD" },
      "rooms_left":    { "type": "NUMBER", "instruction": "Number of rooms left if a scarcity message like 'only 2 left' is shown; else leave empty" },
      "availability":  { "type": "TEXT",   "instruction": "Availability status text, e.g. 'Available', 'Sold out', 'Only 2 rooms left'" },
      "rating":        { "type": "NUMBER", "instruction": "Overall guest review score as shown (e.g. 8.6 or 4.5)" },
      "review_count":  { "type": "NUMBER", "instruction": "Total number of guest reviews, digits only" },
      "free_cancellation": { "type": "TEXT", "instruction": "Yes/No or the free-cancellation policy text if displayed" }
    }
  }
}
```

Because each detail page is one property for one date range, the engine returns a single record per URL (still inside the array shape).

### (a) HTTP — create the job, then poll

```bash
# 1. Create the batch job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://www.example-travel.com/h/alfama-vista?checkin=2026-06-12&checkout=2026-06-14",
      "https://www.example-travel.com/h/baixa-boutique?checkin=2026-06-12&checkout=2026-06-14"
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
      { "url": "https://www.example-travel.com/h/alfama-vista?checkin=2026-06-12&checkout=2026-06-14", "status": "completed", "data": [ { "nightly_rate": 142, "rooms_left": 2, "availability": "Only 2 rooms left" } ] },
      { "url": "https://www.example-travel.com/h/baixa-boutique?checkin=2026-06-12&checkout=2026-06-14", "status": "failed", "error": "timeout" }
    ]
  }
}
```

Job-level `status` is `pending | processing | completed | failed | cancelled`. Per-URL `results[].status` lets you re-submit only those URLs that came back `failed` — never re-run the whole batch, or you'll pay credits again for rows that already succeeded.

### (b) CLI — batch from a file

`--file` reads one URL per line and ignores blank lines and lines starting with `#`.

```bash
# detail_urls.txt has one detail-page URL (with date params) per line
thunderbit batch extract --file detail_urls.txt \
  --schema detail.schema.json \
  --timeout 90000 \
  -f json > details.json
```

### (c) MCP

Ask your assistant: *"Use Thunderbit to batch-extract these property detail URLs with this schema, then poll until done."* It calls `thunderbit_batch_extract_create` (args `urls`, `schema`, `timeout?`) and then `thunderbit_batch_extract_status` (args `jobId`, `page?`, `pageSize?`) until the job is `completed`.

---

## Review mining with distill

Rates tell you *what* to charge; reviews tell you *why* guests pick or avoid a property. You don't need structured fields for free-text reviews — you need clean text. That's what **distill** is for: it turns a page into LLM-ready Markdown for **1 credit** — **20× cheaper than extract**. Run distill over public review pages (e.g. a public TripAdvisor or Trustpilot property page you're authorized to access), then hand the Markdown to your own LLM for sentiment and themes.

### (a) HTTP — distill a public review page

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.example-reviews.com/hotel/alfama-vista",
    "renderMode": "full",
    "country_code": "PT",
    "excludeTags": ["nav", "footer", "aside"],
    "waitFor": 2500,
    "timeout": 45000
  }'
```

The clean Markdown comes back at `data.markdown`:

```json
{
  "data": {
    "markdown": "# Hotel Alfama Vista — Guest Reviews\n\n**8.6 / 10** · 1,284 reviews\n\n> \"Spotless room and the rooftop view was unreal...\"\n\n> \"Walls are thin — could hear the street until 2am...\"",
    "url": "https://www.example-reviews.com/hotel/alfama-vista"
  }
}
```

### (b) Distill at scale + LLM sentiment

Batch-distill many review pages (1 credit/URL), then summarize each with the LLM you already use:

```python
import os, time, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

def batch_distill(urls, poll_every=3):
    job = requests.post(f"{BASE}/openapi/v1/batch/distill", headers=HEADERS,
                        json={"urls": urls}, timeout=60).json()["data"]
    job_id = job["id"]
    while True:
        status = requests.get(f"{BASE}/openapi/v1/batch/distill/{job_id}",
                              headers=HEADERS, params={"page": 0, "pageSize": 100},
                              timeout=60).json()["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            return status
        time.sleep(poll_every)

review_urls = [
    "https://www.example-reviews.com/hotel/alfama-vista",
    "https://www.example-reviews.com/hotel/baixa-boutique",
]
job = batch_distill(review_urls)
for result in job.get("results", []):
    if result.get("status") != "completed":
        continue
    markdown = (result.get("data") or {}).get("markdown", "")
    # Hand `markdown` to your LLM with a prompt like:
    #   "Summarize guest sentiment in 3 bullets, list the top 3 recurring complaints,
    #    and give an overall sentiment label (positive / mixed / negative)."
    print(result["url"], "→", len(markdown), "chars of review text")
```

This keeps cost low: distilling 100 review pages is **100 credits**, versus **2,000 credits** if you (needlessly) extracted them as structured fields. Reserve extract for the rate/availability fields you'll query numerically; use distill for everything you only need to *read*.

---

## End-to-end pipeline

Here's one cohesive, runnable Python script that ties it all together: extract the search page, batch-extract detail pages (in chunks of 100), write to **both** CSV and SQLite as a dated snapshot, then diff against the previous run to surface **rate drops**, **rate hikes**, **new inventory**, and **sellouts**. Error handling follows the reference: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Daily travel rate & availability pipeline built on the Thunderbit Open API."""
import os, time, csv, json, random, sqlite3, datetime
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

# One run per (market, date range). Currency follows the market via country_code.
MARKET = "PT"           # country_code: rates/currency for this market
CHECK_IN = "2026-06-12"
CHECK_OUT = "2026-06-14"
SEARCH_URL = (f"https://www.example-travel.com/hotels?city=lisbon"
              f"&checkin={CHECK_IN}&checkout={CHECK_OUT}")  # a public page you're authorized to access
DB = "rates.sqlite"

RESULTS_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "property_name": {"type": "TEXT",   "instruction": "Hotel or property name shown on the card"},
        "room_type":     {"type": "TEXT",   "instruction": "Lowest-priced room/rate plan name shown"},
        "nightly_rate":  {"type": "NUMBER", "instruction": "Per-night price, digits only, no symbol or commas"},
        "currency":      {"type": "TEXT",   "instruction": "ISO currency code or symbol next to the price"},
        "rating":        {"type": "NUMBER", "instruction": "Guest review score as shown (e.g. 8.6)"},
        "review_count":  {"type": "NUMBER", "instruction": "Number of reviews, digits only"},
        "detail_url":    {"type": "URL",    "instruction": "Absolute link to this property's detail page"},
    }},
}

DETAIL_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "property_name": {"type": "TEXT",   "instruction": "Full property name as shown on the detail page"},
        "room_type":     {"type": "TEXT",   "instruction": "Name of the cheapest available room/rate plan"},
        "nightly_rate":  {"type": "NUMBER", "instruction": "Lowest per-night rate for the dates, digits only"},
        "currency":      {"type": "TEXT",   "instruction": "ISO currency code or symbol next to the rate"},
        "total_price":   {"type": "NUMBER", "instruction": "Total stay price incl. taxes/fees if shown, digits only"},
        "rooms_left":    {"type": "NUMBER", "instruction": "Rooms left if a scarcity message is shown; else empty"},
        "availability":  {"type": "TEXT",   "instruction": "Availability status, e.g. 'Available', 'Sold out', 'Only 2 rooms left'"},
        "rating":        {"type": "NUMBER", "instruction": "Overall guest review score as shown"},
        "review_count":  {"type": "NUMBER", "instruction": "Total number of guest reviews, digits only"},
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

def extract_results():
    res = _post("/openapi/v1/extract", {
        "url": SEARCH_URL, "schema": RESULTS_SCHEMA,
        "renderMode": "full", "waitFor": 3000, "timeout": 90000,
        "country_code": MARKET,
    })
    return res["data"]

def with_dates(url):
    """Carry the date range onto each detail URL so the page prices the right stay."""
    sep = "&" if "?" in url else "?"
    return f"{url}{sep}checkin={CHECK_IN}&checkout={CHECK_OUT}"

def batch_extract_details(urls):
    """Process detail URLs in chunks of 100 (the batch ceiling). Returns flat rows."""
    rows = []
    for start in range(0, len(urls), 100):
        chunk = [with_dates(u) for u in urls[start:start + 100]]
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
        run_date TEXT, market TEXT, check_in TEXT, check_out TEXT, detail_url TEXT,
        property_name TEXT, room_type TEXT, nightly_rate REAL, currency TEXT,
        total_price REAL, rooms_left REAL, availability TEXT, rating REAL, review_count REAL,
        PRIMARY KEY (run_date, market, check_in, check_out, detail_url))""")
    con.commit()
    return con

def previous_rates(con):
    """Most recent prior run's rates for this market+date range, keyed by detail_url."""
    cur = con.execute(
        "SELECT detail_url, nightly_rate, availability FROM snapshots "
        "WHERE market = ? AND check_in = ? AND check_out = ? AND run_date = "
        "(SELECT MAX(run_date) FROM snapshots WHERE market = ? AND check_in = ? AND check_out = ?)",
        (MARKET, CHECK_IN, CHECK_OUT, MARKET, CHECK_IN, CHECK_OUT))
    return {url: {"rate": rate, "availability": avail} for url, rate, avail in cur.fetchall()}

def save_snapshot(con, run_date, rows):
    for r in rows:
        con.execute("""INSERT OR REPLACE INTO snapshots VALUES
            (?,?,?,?,?,?,?,?,?,?,?,?,?,?)""", (
            run_date, MARKET, CHECK_IN, CHECK_OUT, r.get("detail_url"),
            r.get("property_name"), r.get("room_type"), r.get("nightly_rate"), r.get("currency"),
            r.get("total_price"), r.get("rooms_left"), r.get("availability"),
            r.get("rating"), r.get("review_count")))
    con.commit()

def write_csv(run_date, rows):
    fields = ["detail_url","property_name","room_type","nightly_rate","currency","total_price",
              "rooms_left","availability","rating","review_count"]
    with open(f"rates_{MARKET}_{run_date}.csv", "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=fields, extrasaction="ignore")
        w.writeheader()
        w.writerows(rows)

def is_sold_out(row):
    avail = (row.get("availability") or "").lower()
    return "sold out" in avail or "no availability" in avail or row.get("nightly_rate") is None

def diff(prev, rows):
    new_inventory, rate_drops, rate_hikes, sellouts = [], [], [], []
    seen = set()
    for r in rows:
        url, rate = r.get("detail_url"), r.get("nightly_rate")
        seen.add(url)
        if url not in prev:
            new_inventory.append(r)
            continue
        old_rate = prev[url]["rate"]
        if rate is not None and old_rate is not None:
            if rate < old_rate:
                rate_drops.append({"property": r.get("property_name"), "old": old_rate, "new": rate, "detail_url": url})
            elif rate > old_rate:
                rate_hikes.append({"property": r.get("property_name"), "old": old_rate, "new": rate, "detail_url": url})
    # Sellouts: was present (and bookable) yesterday, gone or sold out today
    today_bookable = {r["detail_url"] for r in rows if not is_sold_out(r)}
    for url, info in prev.items():
        was_bookable = info["rate"] is not None and "sold out" not in (info["availability"] or "").lower()
        if was_bookable and url not in today_bookable:
            sellouts.append({"detail_url": url, "was_rate": info["rate"]})
    return new_inventory, rate_drops, rate_hikes, sellouts

def main():
    run_date = datetime.date.today().isoformat()
    con = init_db()
    prev = previous_rates(con)

    results = extract_results()
    detail_urls = [p["detail_url"] for p in results if p.get("detail_url")]
    print(f"Found {len(detail_urls)} properties on the search page ({MARKET}, {CHECK_IN}→{CHECK_OUT})")

    rows = batch_extract_details(detail_urls)
    print(f"Extracted {len(rows)} detail pages")

    write_csv(run_date, rows)
    save_snapshot(con, run_date, rows)

    new_inventory, rate_drops, rate_hikes, sellouts = diff(prev, rows)
    print(f"\n{len(new_inventory)} NEW, {len(rate_drops)} RATE DROPS, "
          f"{len(rate_hikes)} RATE HIKES, {len(sellouts)} SELLOUTS")
    for d in rate_drops:
        print(f"  ↓ {d['property']}: {d['old']} → {d['new']} ({d['detail_url']})")
    for s in sellouts:
        print(f"  ✗ SOLD OUT: {s['detail_url']} (was {s['was_rate']})")

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire this to your alerting (PagerDuty/Slack/email)
        raise
```

Run it, and on day two it will print exactly which properties dropped or raised their rate, which new inventory appeared, and which bookable rooms sold out for that market and date range since the previous snapshot.

---

## No-code version: the Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper** Chrome extension. Non-developer teammates — revenue managers, analysts — can run the identical workflow without writing code, and the schema they build transfers verbatim to the API.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **AI Suggest Fields.** Open the public results page *after* you've set the check-in/check-out dates in the site's date picker, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the rendered page and proposes columns (property name, room type, nightly rate, currency, rating, review count, detail link).
3. **Edit columns** in plain language. Rename, drop, or add columns; tune each column's instruction exactly like the `instruction` strings in the API schema (e.g. "digits only, no currency symbol").
4. **Scrape**, then **Scrape Subpages** to follow each row's `detail_url` and pull the full rate/availability breakdown — the no-code equivalent of the two-stage list → detail batch pipeline in Steps 2–3.
5. **Export** to Excel, Google Sheets, Airtable, or Notion, or **schedule** recurring runs so the rate sheet refreshes itself.

The recommended flow: design and validate the schema interactively in the extension, then port the field names and instructions to the CLI/API for automation at scale. They map one-to-one.

---

## Automate it

- **Cron.** Run the pipeline nightly, once per market. The diff logic already compares against the most recent prior snapshot for the same `(market, check_in, check_out)` stored in SQLite:

  ```cron
  # Every day at 05:30, refresh the Lisbon rate sheet for the target weekend
  30 5 * * *  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/rates/pipeline.py >> /var/log/rates.log 2>&1
  ```

- **Batch webhooks instead of polling.** For server-side jobs, supply a callback so Thunderbit notifies you when the batch finishes — no open connection needed:

  ```json
  {
    "urls": ["https://www.example-travel.com/h/alfama-vista?checkin=2026-06-12&checkout=2026-06-14", "..."],
    "schema": { "type": "array", "items": { "type": "object", "properties": { } } },
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "whatever-signing-secret-you-choose"
    }
  }
  ```

  Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call, then write the results to your store.

- **Store snapshots.** Keep one row per `(run_date, market, check_in, check_out, detail_url)` (as the script does). That history powers rate-change charts, lead-time curves (how rates move as the date approaches), and sellout patterns.
- **Alert on rate drops & sellouts.** Pipe the `rate_drops` and `sellouts` lists to Slack/email. A competitor dropping its rate for a high-demand weekend, or a comp set selling out, is an immediate revenue-management signal.

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit** *(some docs list it as free — confirm against your plan)*, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**Worked example.** A nightly run over **60 properties in one market**, for one date range:

```
60 detail pages × 20 credits   = 1,200 credits / night / market
+ 1 search-page extract × 20    =    20 credits
+ 1 suggest_fields (one-time)   =     1 credit
≈ 1,220 credits / night / market
```

Watch three markets and two date ranges and you're at roughly `1,220 × 6 ≈ 7,320` credits/night — so be deliberate about which markets and dates you actually need.

**Extract vs distill — a 20× decision.** Extract (structured JSON) is 20× the price of distill (clean Markdown). Rates and availability are *numbers you'll query*, so they belong in extract. Guest reviews are *text you'll read*, so distill them: 100 review pages cost **100 credits**, not 2,000. Always ask whether you need structured fields (extract) or just clean text (distill).

**Throttle & cache to cut both cost and load:**

- **One snapshot per market.** Don't re-extract the same property for the same dates twice in a day. The currency/rate is fixed by `country_code` + date range, so one snapshot per `(market, check_in, check_out)` is enough.
- **Dedupe.** Skip detail pages whose rate and availability are unchanged from the last snapshot — your SQLite history already knows them. Only re-extract what moved (or near a date with volatile pricing).
- **Stage cheaply.** The search-page extract is one call that yields many `detail_url`s; only batch-extract the details you don't already have fresh.
- **Reasonable concurrency.** Batch caps at 100 URLs/job; chunk larger sets and don't fire many jobs at one origin simultaneously.

---

## Responsible use

These guardrails apply to every workflow in this repo — travel included, where they matter more than most.

- **Check `robots.txt` and Terms of Service first — OTAs often restrict automated access.** Major online travel agencies and airlines frequently prohibit scraping in their ToS or block it in `robots.txt`. Confirm the site permits automated access to the pages you target *before* you run anything, honor disallowed paths and crawl-rate directives, and back off rather than evade if you're blocked.
- **Public data only.** Thunderbit targets publicly accessible pages. Never build workflows against login-walled platforms, members-only fare pages, or private/authenticated areas. Use only the public listing and review pages you are authorized to access.
- **No login-walled platforms.** If a rate or review is only visible after sign-in (loyalty-only fares, gated portals), it is out of scope — don't try to reach it.
- **Personal-data caution.** Reviewer names and profile details are personal data. Aggregate review *sentiment and themes*; avoid collecting or storing individual reviewers' personal information unless you have a lawful basis (GDPR/CCPA). Prefer property-level facts and aggregate review stats.
- **Throttle.** Use batch with reasonable concurrency; don't hammer a single origin. Add jitter to scheduled runs.
- **Cache and dedupe** to avoid re-scraping unchanged pages — it saves credits and reduces load on the source.

---

## Troubleshooting & FAQ

**Extraction returns an empty array.** The page is almost certainly a client-rendered, date-picker SPA that computes rates only after the dates resolve. Set `renderMode:"full"` (headless browser) and raise `waitFor` (up to its `10000` ms ceiling) so the rates load. Also raise `timeout` (extract allows up to `120000` ms). If you're on `renderMode:"none"` against a booking SPA, that's the cause.

**Rates show up but check-in/check-out look wrong or empty.** The dates must be baked into the URL (query params like `?checkin=2026-06-12&checkout=2026-06-14`) so the page prices the right stay before Thunderbit reads it. Enumerate the date ranges you care about, build one URL per range, and run them as separate extracts or a batch. In the schema, set the `check_in`/`check_out` `instruction` to fall back to the dates you requested if they aren't printed per-card.

**Pagination / "load more" on the results page.** Most OTAs paginate via a query parameter (`?page=2` or `&offset=25`) on top of the date params. Enumerate the page URLs and feed them as a list to **batch extract** with the results schema, then merge. For infinite-scroll pages, increasing `waitFor` loads more cards, but URL-based pagination is more reliable than scrolling.

**Currency comes back as part of the price (e.g. "€142" or "142 EUR").** Split it: keep `nightly_rate` as NUMBER with the instruction "digits only, no currency symbol or commas," and add a separate `currency` TEXT field with "ISO currency code or symbol next to the price." For NUMBER fields, the instruction is what coerces the value — brief it like you'd brief a human assistant.

**Rates differ from what I see in my browser / wrong currency.** Rates vary by market. Set `country_code` (HTTP/CLI) or `countryCode` (MCP) to route the request from the right region — e.g. `US`, `GB`, `DE`, `PT`. The default is `US`. Run one snapshot per market and store the `currency` alongside the rate so you never compare EUR to USD by accident.

**Timeouts on heavy detail pages.** Raise `timeout` toward the `120000` ms max and add `waitFor`. For a single very slow SPA page outside a batch, use the async endpoints (`POST /openapi/v1/async/extract` then poll `GET /openapi/v1/async/extract/{jobId}` every ~3s) so you don't hold an HTTP connection open.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status`; re-submit only the `failed` URLs in a fresh batch job. Don't re-run the whole batch — you'd pay credits again for the rows that already succeeded. Travel pages fail more often near peak load, so a modest retry of just the failures is usually all you need.

**Anti-bot / occasional blocks.** Throttle, add jitter between runs, and keep batch concurrency modest. If a specific OTA blocks automated access, that is a signal to re-check its ToS and `robots.txt` — back off rather than evade.

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.
