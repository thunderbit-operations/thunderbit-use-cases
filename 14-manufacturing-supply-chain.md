# Manufacturing & Supply Chain Intelligence with Thunderbit

Build a self-updating supplier database, extract component spec sheets at scale, and monitor unit prices and lead times daily — all from public B2B catalogs and supplier directories, using one AI web-data engine reachable from MCP, HTTP, or CLI.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## The data problem in sourcing

Sourcing and procurement intelligence is a data-plumbing problem disguised as a negotiation problem. The pains are concrete:

- **Fragmented catalogs.** Suppliers and parts live across dozens of B2B marketplaces, distributor catalogs, and supplier directories, each with its own HTML, its own field names, and its own units. There is no single clean feed for "every authorized distributor of this stepper motor with MOQ, unit price, and lead time."
- **Price and lead-time volatility.** A unit price quoted on Monday and a "ships in 2 weeks" promise can both be stale by Friday. Component pricing moves with demand, FX, and allocation; lead times stretch and compress weekly. A static RFQ spreadsheet decays fast.
- **Spec sheets in HTML and PDF.** Technical attributes — voltage, torque, tolerance, package, operating temperature — are buried in detail pages, spec tables, and linked datasheet PDFs. Comparing parts across vendors means reading dozens of layouts by hand.
- **Manual RFQ research doesn't scale.** A buyer copying part number, MOQ, unit price, currency, and certifications into a spreadsheet one listing at a time is slow, error-prone, and impossible to schedule across hundreds of SKUs.

The fix is a small, repeatable pipeline: discover the schema once, extract the catalog search-results page into a list of suppliers and products, follow each listing's detail/spec page in a batch, store the result, and diff it against yesterday for price and lead-time movement. Thunderbit gives you the extraction engine; this tutorial gives you the pipeline.

---

## What you'll build

- A **supplier dataset** with supplier name, product, location, minimum order quantity, unit price, currency, lead time, certifications, and the supplier/listing URL.
- A **spec-sheet extractor** that pulls part numbers and key technical specs (numbers and text) from component detail pages, plus the linked datasheet URL — with an option to distill full datasheet text cheaply.
- A **price & lead-time monitor** that stores a daily snapshot and flags unit-price increases/decreases and lead-time changes since the previous run.
- A two-stage pipeline: **(1)** a catalog search-results page → `suggest_fields` → extract the supplier/product list with detail URLs; **(2)** **batch-extract** the spec/detail pages for full attributes; **(3)** store + diff against the previous run.
- The same workflow, no-code, via the Thunderbit Chrome extension for non-developer teammates on the sourcing team.

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
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and ad-hoc supplier lookups. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. Best for the automated daily price/lead-time monitor. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, saving/reusing schemas. Best for prototyping and glue scripts. |

A good real-world split: **explore** a new catalog in MCP, **freeze a schema** with the CLI, **run it nightly** over the HTTP API.

---

## Step 1 — discover the schema

Don't guess field names. Let the AI inspect the page and propose a schema. Throughout this tutorial, treat `https://www.example-bsupply.com/search?q=stepper-motor` as **a public B2B catalog you are authorized to access**. The same pattern works on any public supplier directory or marketplace listing page (public ThomasNet pages, public distributor/component-search catalogs, public marketplace listing pages) — but read the "Responsible use" section first, and check the site's `robots.txt` and Terms of Service before you point a scraper at it.

`suggest_fields` costs **1 credit** and returns an array of field descriptors you can drop straight into an extract schema.

### (a) MCP — let the assistant call the tool

Prompt your assistant:

> Use Thunderbit to suggest fields for `https://www.example-bsupply.com/search?q=stepper-motor`. Focus on each supplier/product listing: supplier name, product, location, minimum order quantity, unit price, currency, lead time, certifications, and the link to the supplier/detail page.

The model issues a `thunderbit_suggest_fields` call with arguments like:

```json
{
  "url": "https://www.example-bsupply.com/search?q=stepper-motor",
  "prompt": "Extract each supplier/product listing: supplier name, product, location, MOQ, unit price, currency, lead time, certifications, detail-page link",
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
    "url": "https://www.example-bsupply.com/search?q=stepper-motor",
    "prompt": "Extract each supplier/product listing: supplier name, product, location, MOQ, unit price, currency, lead time, certifications, detail-page link",
    "country_code": "US"
  }'
```

### (c) CLI

```bash
thunderbit suggest-fields "https://www.example-bsupply.com/search?q=stepper-motor" \
  --prompt "Extract each supplier/product listing: supplier name, product, location, MOQ, unit price, currency, lead time, certifications, detail-page link" \
  --country-code US
```

### The suggested-fields response

The fields come back at `data` (one array of descriptors):

```json
{
  "data": [
    { "name": "supplier_name",  "type": "TEXT",   "instruction": "Company/supplier name shown on the listing" },
    { "name": "product",        "type": "TEXT",   "instruction": "Product or component title" },
    { "name": "location",       "type": "TEXT",   "instruction": "Supplier location: city and/or country" },
    { "name": "min_order_qty",  "type": "NUMBER", "instruction": "Minimum order quantity (MOQ), digits only" },
    { "name": "unit_price",     "type": "NUMBER", "instruction": "Lowest listed unit price, digits only, no currency symbol" },
    { "name": "currency",       "type": "TEXT",   "instruction": "ISO currency code or symbol for the unit price (e.g. USD, EUR, CNY)" },
    { "name": "lead_time",      "type": "TEXT",   "instruction": "Quoted lead time / ship time text (e.g. '2-3 weeks')" },
    { "name": "certifications", "type": "TEXT",   "instruction": "Listed certifications (e.g. RoHS, CE, ISO 9001), comma-separated" },
    { "name": "supplier_url",   "type": "URL",    "instruction": "Absolute link to the supplier or product detail page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;` (JS) or `fields = res["data"].get("fields", res["data"])` (Python).

Treat the suggestion as a starting point. You'll usually tighten the `instruction` strings — the single biggest lever on extraction quality, and especially important for messy fields like price ("digits only, no currency symbol or commas") and lead time.

---

## Step 2 — extract the supplier/product list

Now turn the catalog search-results page into structured rows. The schema is **not** raw JSON Schema; it's the Thunderbit-flavored shape: an array of objects, where each property has an UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction`. Note that `min_order_qty` and `unit_price` are **NUMBER** so they store and diff cleanly. Extract costs **20 credits** per page.

For a catalog/search page we want enough to identify each supplier+product, plus the `supplier_url` we'll follow in Step 3:

### (a) HTTP — `curl`

Many modern B2B catalogs render listings client-side and lazy-load cards, so set `renderMode:"full"` (headless browser) and give lazy content a moment with `waitFor`.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.example-bsupply.com/search?q=stepper-motor",
    "renderMode": "full",
    "waitFor": 2500,
    "timeout": 90000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "supplier_name":  { "type": "TEXT",   "instruction": "Company/supplier name shown on the listing" },
          "product":        { "type": "TEXT",   "instruction": "Product or component title" },
          "location":       { "type": "TEXT",   "instruction": "Supplier location: city and/or country" },
          "min_order_qty":  { "type": "NUMBER", "instruction": "Minimum order quantity (MOQ), digits only" },
          "unit_price":     { "type": "NUMBER", "instruction": "Lowest listed unit price, digits only, no currency symbol or commas" },
          "currency":       { "type": "TEXT",   "instruction": "Currency code or symbol for the unit price, e.g. USD, EUR, CNY" },
          "lead_time":      { "type": "TEXT",   "instruction": "Quoted lead time / ship time text exactly as shown, e.g. '2-3 weeks'" },
          "certifications": { "type": "TEXT",   "instruction": "Listed certifications such as RoHS, CE, ISO 9001; comma-separated" },
          "supplier_url":   { "type": "URL",    "instruction": "Absolute link to this supplier or product detail page" }
        }
      }
    }
  }'
```

Rows come back at `data`:

```json
{
  "data": [
    { "supplier_name": "Acme Motion Co.", "product": "NEMA 17 Stepper Motor 1.8°", "location": "Shenzhen, CN", "min_order_qty": 100, "unit_price": 4.20, "currency": "USD", "lead_time": "2-3 weeks", "certifications": "RoHS, CE", "supplier_url": "https://www.example-bsupply.com/p/acme-nema17" },
    { "supplier_name": "Northwind Drives", "product": "NEMA 23 Stepper Motor 3.0A", "location": "Ohio, US", "min_order_qty": 50, "unit_price": 11.75, "currency": "USD", "lead_time": "in stock", "certifications": "RoHS, ISO 9001", "supplier_url": "https://www.example-bsupply.com/p/northwind-nema23" }
  ]
}
```

### (b) Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

SUPPLIER_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "supplier_name":  {"type": "TEXT",   "instruction": "Company/supplier name shown on the listing"},
            "product":        {"type": "TEXT",   "instruction": "Product or component title"},
            "location":       {"type": "TEXT",   "instruction": "Supplier location: city and/or country"},
            "min_order_qty":  {"type": "NUMBER", "instruction": "Minimum order quantity (MOQ), digits only"},
            "unit_price":     {"type": "NUMBER", "instruction": "Lowest listed unit price, digits only, no currency symbol or commas"},
            "currency":       {"type": "TEXT",   "instruction": "Currency code or symbol, e.g. USD, EUR, CNY"},
            "lead_time":      {"type": "TEXT",   "instruction": "Quoted lead time / ship time text exactly as shown"},
            "certifications": {"type": "TEXT",   "instruction": "Listed certifications such as RoHS, CE, ISO 9001; comma-separated"},
            "supplier_url":   {"type": "URL",    "instruction": "Absolute link to this supplier or product detail page"},
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

suppliers = extract("https://www.example-bsupply.com/search?q=stepper-motor", SUPPLIER_SCHEMA)
print(len(suppliers), "listings")
```

### (c) CLI — save the schema, then reuse it

Save the schema to a file once, then re-run it any time:

```bash
# Save the supplier schema to supplier.schema.json (an array-of-objects schema as above),
# then extract with it:
thunderbit extract "https://www.example-bsupply.com/search?q=stepper-motor" \
  --schema supplier.schema.json \
  --render-mode full \
  --timeout 90000 \
  -f json > suppliers.json
```

> **`renderMode` cheat sheet:** `none` = raw HTML fetch (fastest/cheapest render; fine for server-rendered catalogs); `basic` = light JS; `full` = headless browser for JS-heavy catalogs and faceted-search SPAs. If a page comes back empty, the first thing to try is `renderMode:"full"` plus a larger `waitFor`. The valid `waitFor` range is `0–10000` ms; `extract` `timeout` is `5000–120000` ms (default `60000`).

> **Regional catalogs.** A distributor that serves different inventory, MOQs, or currency by region needs the request routed from the right country. Set `country_code` on the HTTP/CLI call (`countryCode` in MCP) — e.g. `DE` for a European distributor catalog, `CN` for a marketplace whose default is region-gated. The default is `US`.

---

## Step 3 — batch-extract spec/detail pages

The search page rarely has everything — full part numbers, technical specs (voltage, current, torque, step angle, package, operating temperature), price-break tables, and the linked datasheet usually live on the detail/spec page. Collect the `supplier_url` values from Step 2 and process them with **batch extract**, which handles **up to 100 URLs per job** at **20 credits/URL**.

Define a richer spec schema. Use **NUMBER** for measurable specs so they compare cleanly, **TEXT** for categorical/unit-bearing fields, and **URL** for the datasheet link:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "part_number":     { "type": "TEXT",   "instruction": "Manufacturer part number (MPN) exactly as printed" },
      "supplier_name":   { "type": "TEXT",   "instruction": "Supplier or manufacturer name" },
      "rated_voltage_v": { "type": "NUMBER", "instruction": "Rated voltage in volts, digits only" },
      "rated_current_a": { "type": "NUMBER", "instruction": "Rated current per phase in amperes, digits only" },
      "step_angle_deg":  { "type": "NUMBER", "instruction": "Step angle in degrees, e.g. 1.8" },
      "holding_torque":  { "type": "TEXT",   "instruction": "Holding torque with its unit exactly as shown, e.g. '0.45 N·m'" },
      "package":         { "type": "TEXT",   "instruction": "Package / frame size, e.g. NEMA 17" },
      "unit_price":      { "type": "NUMBER", "instruction": "Lowest unit price on the detail page, digits only, no symbol" },
      "currency":        { "type": "TEXT",   "instruction": "Currency code or symbol for unit_price" },
      "min_order_qty":   { "type": "NUMBER", "instruction": "Minimum order quantity (MOQ), digits only" },
      "lead_time":       { "type": "TEXT",   "instruction": "Quoted lead time text exactly as shown" },
      "certifications":  { "type": "TEXT",   "instruction": "Certifications listed, comma-separated" },
      "datasheet_url":   { "type": "URL",    "instruction": "Absolute link to the PDF/HTML datasheet, if present" }
    }
  }
}
```

Because each detail page is one component, the engine returns a single record per URL (still inside the array shape).

### (a) HTTP — create the job, then poll

```bash
# 1. Create the batch job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://www.example-bsupply.com/p/acme-nema17",
      "https://www.example-bsupply.com/p/northwind-nema23"
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
      { "url": "https://www.example-bsupply.com/p/acme-nema17",     "status": "completed", "data": [ { "part_number": "AM-17HS4023", "rated_voltage_v": 12, "unit_price": 4.20, "lead_time": "2-3 weeks" } ] },
      { "url": "https://www.example-bsupply.com/p/northwind-nema23", "status": "failed",    "error": "timeout" }
    ]
  }
}
```

Job-level `status` is `pending | processing | completed | failed | cancelled`. Per-URL `results[].status` lets you re-submit only those URLs that came back `failed`.

### (b) CLI — batch from a file

`--file` reads one URL per line and ignores blank lines and lines starting with `#`.

```bash
# spec_urls.txt has one detail-page URL per line
thunderbit batch extract --file spec_urls.txt \
  --schema spec.schema.json \
  --timeout 90000 \
  -f json > specs.json
```

### (c) MCP

Ask your assistant: *"Use Thunderbit to batch-extract these spec-page URLs with this schema, then poll until done."* It calls `thunderbit_batch_extract_create` (args `urls`, `schema`, `timeout?`) and then `thunderbit_batch_extract_status` (args `jobId`, `page?`, `pageSize?`) until the job is `completed`.

### Datasheet PDFs — distill for full text, cheaply

When a spec page links a full **datasheet** and you want the complete technical text (for a RAG index, a spec-compare LLM, or a searchable archive), reach for **distill** instead of extract — it returns clean Markdown at **1 credit/URL**, a 20× saving over extract. Point distill at the **datasheet page URL** (Thunderbit renders the page; for a PDF served inline, pass the page URL and raise `timeout`/use the async path if it's large):

```bash
# One datasheet → Markdown (1 credit)
thunderbit distill "https://www.example-bsupply.com/datasheets/AM-17HS4023" -f markdown

# Many datasheets → batch distill (1 credit/URL)
thunderbit batch distill --file datasheet_urls.txt --timeout 90000 -f json > datasheets.json
```

Use **extract** for the structured fields you'll query and diff (price, lead time, voltage); use **distill** for the long-form datasheet text you'll only read or embed.

---

## Price & lead-time monitoring

The whole point of running this nightly is to catch movement. Two signals matter most to a buyer: **unit-price changes** (a hike is a budget/quote risk; a drop is a re-source opportunity) and **lead-time changes** (a stretch is a supply risk that may need a second source). The pattern is the same as any change-detection pipeline: store a daily snapshot keyed by a stable identity, then diff today's row against the most recent prior row.

- **Identity key.** Use `part_number` when present (it's the most stable cross-vendor key) and fall back to `supplier_url` for listings without a clean MPN. Diff like-for-like.
- **Price diff.** Compare `unit_price` (a NUMBER) against the previous snapshot's value for the same key. Flag increases and decreases, and compute the percentage delta. Only compare when `currency` matches — a currency switch is a data issue, not a price move.
- **Lead-time diff.** `lead_time` is free text ("in stock", "2-3 weeks", "8-10 weeks"). Compare the stored string; when it changes, flag it. If you want numeric thresholds, normalize to weeks with a small parser (or add a NUMBER `lead_time_weeks` field with an `instruction` like *"lead time converted to whole weeks; 'in stock' = 0"*).

```python
def diff_prices_and_leadtime(prev, rows, key="part_number"):
    """prev: {key: {"unit_price","currency","lead_time"}}. Returns alerts."""
    price_changes, leadtime_changes, new_parts = [], [], []
    for r in rows:
        k = r.get(key) or r.get("supplier_url")
        if not k:
            continue
        before = prev.get(k)
        if before is None:
            new_parts.append(r)
            continue
        # price (only compare same currency)
        p_old, p_new = before.get("unit_price"), r.get("unit_price")
        if (p_old not in (None, 0) and p_new is not None
                and before.get("currency") == r.get("currency") and p_new != p_old):
            pct = round((p_new - p_old) / p_old * 100, 1)
            price_changes.append({"key": k, "product": r.get("product"),
                                  "old": p_old, "new": p_new, "pct": pct,
                                  "direction": "increase" if p_new > p_old else "decrease"})
        # lead time (string compare)
        lt_old, lt_new = before.get("lead_time"), r.get("lead_time")
        if lt_old and lt_new and lt_old != lt_new:
            leadtime_changes.append({"key": k, "product": r.get("product"),
                                     "old": lt_old, "new": lt_new})
    return new_parts, price_changes, leadtime_changes
```

On day two this prints exactly which parts moved in price (and by how much) and which lead times stretched or shortened since the previous snapshot.

---

## End-to-end pipeline

Here's one cohesive, runnable Python script that ties it all together: extract the catalog search page, batch-extract spec/detail pages (in chunks of 100), write to **both** CSV and SQLite, then diff against the previous run to surface **new parts**, **price changes**, and **lead-time changes**. Error handling follows the reference: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Daily supplier / component price & lead-time monitor built on the Thunderbit Open API."""
import os, time, csv, json, random, sqlite3, datetime
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
SEARCH_URL = "https://www.example-bsupply.com/search?q=stepper-motor"  # a public B2B catalog you're authorized to access
DB = "supply.sqlite"

SUPPLIER_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "supplier_name":  {"type": "TEXT",   "instruction": "Company/supplier name shown on the listing"},
        "product":        {"type": "TEXT",   "instruction": "Product or component title"},
        "location":       {"type": "TEXT",   "instruction": "Supplier location: city and/or country"},
        "min_order_qty":  {"type": "NUMBER", "instruction": "Minimum order quantity (MOQ), digits only"},
        "unit_price":     {"type": "NUMBER", "instruction": "Lowest listed unit price, digits only, no symbol or commas"},
        "currency":       {"type": "TEXT",   "instruction": "Currency code or symbol, e.g. USD, EUR, CNY"},
        "lead_time":      {"type": "TEXT",   "instruction": "Quoted lead time text exactly as shown"},
        "certifications": {"type": "TEXT",   "instruction": "Certifications such as RoHS, CE, ISO 9001; comma-separated"},
        "supplier_url":   {"type": "URL",    "instruction": "Absolute link to this supplier or product detail page"},
    }},
}

SPEC_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "part_number":     {"type": "TEXT",   "instruction": "Manufacturer part number (MPN) exactly as printed"},
        "supplier_name":   {"type": "TEXT",   "instruction": "Supplier or manufacturer name"},
        "product":         {"type": "TEXT",   "instruction": "Product or component title"},
        "rated_voltage_v": {"type": "NUMBER", "instruction": "Rated voltage in volts, digits only"},
        "rated_current_a": {"type": "NUMBER", "instruction": "Rated current per phase in amperes, digits only"},
        "step_angle_deg":  {"type": "NUMBER", "instruction": "Step angle in degrees, e.g. 1.8"},
        "holding_torque":  {"type": "TEXT",   "instruction": "Holding torque with unit, e.g. '0.45 N·m'"},
        "package":         {"type": "TEXT",   "instruction": "Package / frame size, e.g. NEMA 17"},
        "unit_price":      {"type": "NUMBER", "instruction": "Lowest unit price, digits only, no symbol"},
        "currency":        {"type": "TEXT",   "instruction": "Currency code or symbol for unit_price"},
        "min_order_qty":   {"type": "NUMBER", "instruction": "Minimum order quantity (MOQ), digits only"},
        "lead_time":       {"type": "TEXT",   "instruction": "Quoted lead time text exactly as shown"},
        "certifications":  {"type": "TEXT",   "instruction": "Certifications listed, comma-separated"},
        "datasheet_url":   {"type": "URL",    "instruction": "Absolute link to the PDF/HTML datasheet, if present"},
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

def extract_suppliers():
    res = _post("/openapi/v1/extract", {
        "url": SEARCH_URL, "schema": SUPPLIER_SCHEMA,
        "renderMode": "full", "waitFor": 2500, "timeout": 90000,
    })
    return res["data"]

def batch_extract_specs(urls):
    """Process detail/spec URLs in chunks of 100 (the batch ceiling). Returns flat rows."""
    rows = []
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/extract", {
            "urls": chunk, "schema": SPEC_SCHEMA, "timeout": 90000,
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
                print(f"  ! spec failed: {result['url']} ({result.get('error')})")
                continue
            payload = result.get("data") or []
            record = payload[0] if isinstance(payload, list) and payload else (payload or {})
            record["supplier_url"] = result["url"]
            rows.append(record)
    return rows

def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS snapshots (
        run_date TEXT, part_key TEXT, supplier_url TEXT, supplier_name TEXT, product TEXT,
        part_number TEXT, location TEXT, min_order_qty REAL, unit_price REAL, currency TEXT,
        lead_time TEXT, certifications TEXT, datasheet_url TEXT,
        rated_voltage_v REAL, rated_current_a REAL, step_angle_deg REAL,
        holding_torque TEXT, package TEXT,
        PRIMARY KEY (run_date, part_key))""")
    con.commit()
    return con

def part_key(r):
    """Stable identity: prefer part_number, fall back to supplier_url."""
    return (r.get("part_number") or "").strip() or r.get("supplier_url")

def previous_snapshot(con):
    """Most recent prior run keyed by part_key → {unit_price, currency, lead_time}."""
    cur = con.execute("SELECT part_key, unit_price, currency, lead_time FROM snapshots "
                      "WHERE run_date = (SELECT MAX(run_date) FROM snapshots)")
    return {k: {"unit_price": p, "currency": c, "lead_time": lt}
            for k, p, c, lt in cur.fetchall()}

def save_snapshot(con, run_date, rows):
    for r in rows:
        con.execute("""INSERT OR REPLACE INTO snapshots VALUES
            (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)""", (
            run_date, part_key(r), r.get("supplier_url"), r.get("supplier_name"), r.get("product"),
            r.get("part_number"), r.get("location"), r.get("min_order_qty"), r.get("unit_price"),
            r.get("currency"), r.get("lead_time"), r.get("certifications"), r.get("datasheet_url"),
            r.get("rated_voltage_v"), r.get("rated_current_a"), r.get("step_angle_deg"),
            r.get("holding_torque"), r.get("package")))
    con.commit()

def write_csv(run_date, rows):
    fields = ["part_number","supplier_name","product","location","min_order_qty","unit_price",
              "currency","lead_time","certifications","datasheet_url","rated_voltage_v",
              "rated_current_a","step_angle_deg","holding_torque","package","supplier_url"]
    with open(f"supply_{run_date}.csv", "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=fields, extrasaction="ignore")
        w.writeheader()
        w.writerows(rows)

def diff(prev, rows):
    new_parts, price_changes, leadtime_changes = [], [], []
    for r in rows:
        k = part_key(r)
        if not k:
            continue
        before = prev.get(k)
        if before is None:
            new_parts.append(r)
            continue
        p_old, p_new = before.get("unit_price"), r.get("unit_price")
        if (p_old not in (None, 0) and p_new is not None
                and before.get("currency") == r.get("currency") and p_new != p_old):
            pct = round((p_new - p_old) / p_old * 100, 1)
            price_changes.append({"key": k, "product": r.get("product"),
                                  "old": p_old, "new": p_new, "pct": pct,
                                  "direction": "increase" if p_new > p_old else "decrease"})
        lt_old, lt_new = before.get("lead_time"), r.get("lead_time")
        if lt_old and lt_new and lt_old != lt_new:
            leadtime_changes.append({"key": k, "product": r.get("product"),
                                     "old": lt_old, "new": lt_new})
    return new_parts, price_changes, leadtime_changes

def main():
    run_date = datetime.date.today().isoformat()
    con = init_db()
    prev = previous_snapshot(con)

    suppliers = extract_suppliers()
    detail_urls = [s["supplier_url"] for s in suppliers if s.get("supplier_url")]
    print(f"Found {len(detail_urls)} listings on the search page")

    rows = batch_extract_specs(detail_urls)
    print(f"Extracted {len(rows)} spec/detail pages")

    write_csv(run_date, rows)
    save_snapshot(con, run_date, rows)

    new_parts, price_changes, leadtime_changes = diff(prev, rows)
    print(f"\n{len(new_parts)} NEW parts, {len(price_changes)} PRICE changes, "
          f"{len(leadtime_changes)} LEAD-TIME changes")
    for c in price_changes:
        arrow = "↑" if c["direction"] == "increase" else "↓"
        print(f"  {arrow} {c['product']}: {c['old']} → {c['new']} ({c['pct']:+}%) [{c['key']}]")
    for c in leadtime_changes:
        print(f"  ⏱ {c['product']}: '{c['old']}' → '{c['new']}' [{c['key']}]")

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire this to your alerting (PagerDuty/Slack/email)
        raise
```

Run it, and on day two it will print exactly which parts are new, which unit prices moved (and by what percentage), and which lead times stretched or shortened since the previous snapshot.

---

## No-code version: the Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper** Chrome extension. Non-developer teammates on the sourcing or procurement team can run the identical workflow without writing code, and the schema they build transfers verbatim to the API.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **AI Suggest Fields.** Open the public B2B catalog search page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns (supplier name, product, MOQ, unit price, currency, lead time, certifications, detail link).
3. **Edit columns** in plain language. Rename, drop, or add columns; tune each column's instruction exactly like the `instruction` strings in the API schema (e.g. "unit price, digits only, no currency symbol").
4. **Scrape**, then **Scrape Subpages** to follow each row's `supplier_url` and pull the full part numbers and specs — the no-code equivalent of the two-stage list → detail batch pipeline in Steps 2–3.
5. **Export** to Excel, Google Sheets, Airtable, or Notion, or **schedule** recurring runs so the supplier sheet refreshes daily.

The recommended flow: design and validate the schema interactively in the extension, then port the field names and instructions to the CLI/API for automation at scale. They map one-to-one.

---

## Automate it

- **Cron.** Run the monitor nightly. The diff logic already compares against the most recent prior snapshot stored in SQLite:

  ```cron
  # Every day at 05:30, run the supply monitor and append a daily log
  30 5 * * *  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/supply/pipeline.py >> /var/log/supply.log 2>&1
  ```

- **Batch webhooks instead of polling.** For server-side jobs, supply a callback so Thunderbit notifies you when the batch finishes — no open connection needed:

  ```json
  {
    "urls": ["https://www.example-bsupply.com/p/acme-nema17", "..."],
    "schema": { "type": "array", "items": { "type": "object", "properties": { } } },
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "whatever-signing-secret-you-choose"
    }
  }
  ```

  Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call, then write the results to your store and run the diff.

- **Store snapshots.** Keep one row per `(run_date, part_key)` (as the script does). That history powers price-trend charts, lead-time stretch alerts, and supplier-reliability scoring over time.
- **Alert on changes.** Pipe `price_changes` and `leadtime_changes` to Slack/email. A price hike on a single-sourced critical part, or a lead time jumping from "in stock" to "8-10 weeks," is exactly the early-warning signal a buyer wants before it becomes a line-down event.

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit** *(some docs list it as free — confirm against your plan)*, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**Worked example.** A nightly run over **300 spec/detail pages**:

```
300 spec pages × 20 credits   = 6,000 credits / night
+ 1 search-page extract × 20   =    20 credits
+ 1 suggest_fields (one-time)  =     1 credit
≈ 6,020 credits / night
```

**Extract vs distill — a 20× decision.** Extract (structured JSON) is 20× the price of distill (clean Markdown). If you only need the *full datasheet text* — to build a spec-search RAG index or let an LLM compare parts — distill 300 datasheet pages for **300 credits** instead of 6,000. Reserve **extract** for the structured fields you'll actually query and diff (part number, unit price, lead time, voltage); use **distill** for the long-form text you'll only read or embed.

**Throttle & dedupe to cut both cost and load:**

- **Dedupe by `part_number` (or `supplier_url`).** The same MPN often appears under multiple listings and across catalogs. Collapse on the part key before batch-extracting so you don't pay 20 credits twice for the same component.
- **Skip unchanged parts.** If a part's price, lead time, and key specs matched yesterday and the listing has no "updated" signal, skip re-extracting it — your SQLite history already knows it. Only re-extract what moved or is new.
- **Stage cheaply.** The search-page extract is one call that yields many `supplier_url`s; only batch-extract the detail pages you don't already have fresh.
- **Reasonable concurrency.** Batch caps at 100 URLs/job; chunk larger sets and don't fire many jobs at one origin simultaneously.

---

## Responsible use

These guardrails apply to every workflow in this repo — supplier and component intelligence included.

- **Check `robots.txt` and Terms of Service first.** Before pointing a scraper at any B2B catalog, marketplace, or supplier directory, confirm the site permits automated access to the pages you target. Honor disallowed paths and crawl-rate directives.
- **Public data only.** Thunderbit targets publicly accessible pages — public marketplace listings, public distributor catalogs, public supplier directory pages. Never build workflows against login-walled portals, gated buyer accounts, or private/authenticated quote systems. Use only the public catalog pages you are authorized to access.
- **Business/product-level data.** Supplier company names, product specs, MOQs, list prices, and lead times are business and product facts, not personal data — that's the right scope here. Avoid collecting named individuals' contact details unless they're publicly posted and you have a lawful basis (GDPR/CCPA); prefer the company-level contact a supplier publishes for inquiries.
- **Throttle.** Use batch with reasonable concurrency; don't hammer a single origin. Add jitter to scheduled runs.
- **Respect catalog ToS on redistribution.** Some catalogs permit access for your own sourcing research but restrict re-publishing their pricing or datasheets. Use the data for internal sourcing decisions; check the source's terms before redistributing prices, spec tables, or datasheet content externally.
- **Cache and dedupe** to avoid re-scraping unchanged pages — it saves credits and reduces load on the source.

---

## Troubleshooting & FAQ

**Extraction returns an empty array.** The catalog is almost certainly client-rendered or behind faceted-search JavaScript. Set `renderMode:"full"` (headless browser) and raise `waitFor` (up to its `10000` ms ceiling) so lazy listing cards load. Also raise `timeout` (extract allows up to `120000` ms). If you're on `renderMode:"none"` against an SPA catalog, that's the cause.

**Prices or quantities come back as strings (e.g. "$4.20 / pc", "MOQ 1,000 pcs").** Tighten the `instruction` on the NUMBER field: "digits only, no currency symbol, commas, or unit suffix." For `unit_price` and `min_order_qty`, the instruction is what coerces the value — brief it like you'd brief a human assistant. Keep `currency` as a separate TEXT field so the number stays clean.

**Pagination / "load more" on the search page.** Most catalogs paginate via a query parameter (`?page=2`, `&offset=50`) or a faceted-search state. Enumerate the page URLs and feed them as a list to **batch extract** with the supplier schema, then merge. For infinite-scroll catalogs, increasing `waitFor` loads more cards, but URL-based pagination is more reliable than scrolling.

**Duplicate parts across listings.** The same MPN appears under several suppliers and on multiple catalogs. Dedupe on `part_number` after extraction (fall back to `supplier_url` when the MPN is missing); the pipeline's `part_key` already does this. Dedup before batch-extracting to save 20 credits per avoided page.

**Datasheet PDFs.** When the spec lives in a linked datasheet, point **distill** at the datasheet *page URL* (1 credit) rather than extracting it (20 credits). Large or slow PDFs can exceed a synchronous call — raise `timeout` toward the distill max (`60000` ms) and, for a single very slow document, use the async path (`POST /openapi/v1/async/distill` then poll `GET /openapi/v1/async/distill/{jobId}` every ~3s) so you don't hold an HTTP connection open.

**Regional catalogs / wrong currency or inventory.** A distributor that serves different MOQs, stock, or currency by region needs the request routed correctly. Set `country_code` (HTTP/CLI) or `countryCode` (MCP) — e.g. `DE`, `GB`, `CN`. The default is `US`. This matters when a catalog shows EUR pricing to European visitors and USD to everyone else.

**Lead time won't diff cleanly.** `lead_time` is free text and vendors phrase it differently ("in stock" vs "ships immediately"). For numeric thresholds, add a NUMBER `lead_time_weeks` field with an `instruction` like *"lead time converted to whole weeks; 'in stock' or 'ships now' = 0"*, and diff that instead of the raw string.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status`; re-submit only the `failed` URLs in a fresh batch job. Don't re-run the whole batch — you'd pay credits again for the rows that already succeeded.

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.
