# Local Business Intelligence with Thunderbit

Build a self-updating local-business dataset, audit NAP (Name / Address / Phone) consistency across maps and directories, and monitor ratings over time — all from public listings, using one AI web-data engine reachable from MCP, HTTP, or CLI.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## The data problem in local

Local business intelligence is a data-plumbing problem disguised as a marketing problem. The pains are concrete:

- **Fragmented listings.** The same coffee shop exists on Google Maps, on Yelp, in a chamber-of-commerce directory, and on three niche local directories — each with its own HTML, its own field names, and its own version of the truth. There is no single clean feed for "every café in this city, with rating and phone."
- **NAP drift kills local rankings.** Search engines treat consistent **N**ame / **A**ddress / **P**hone signals across the web as a trust factor for local SEO. When one directory still lists an old suite number or a disconnected phone, rankings and click-throughs quietly erode. The damage is invisible until you diff the sources side by side.
- **Manual audits don't scale.** Checking NAP by hand means opening a dozen tabs per business and eyeballing strings that differ only by `St` vs `Street` or `(512) 555-0142` vs `512.555.0142`. It's slow, error-prone, and impossible to schedule across hundreds of locations.
- **JS-rendered map UIs.** Modern map and directory front-ends render listings client-side, lazy-load result cards as you pan or scroll, and hide hours and the full address behind a detail panel. A naive HTTP `GET` returns an empty shell, so you need a real headless browser to see what a human sees.

The fix is a small, repeatable pipeline: discover the schema once, extract the search-results page into a list of businesses, follow each business's detail link in a batch, normalize and diff the NAP fields across sources, store the result, and compare it against the previous run. Thunderbit gives you the extraction engine; this tutorial gives you the pipeline.

---

## What you'll build

- A **local-business dataset** with `business_name`, `category`, `address`, `phone`, `rating`, `review_count`, `website`, and `hours` — sourced from public map and directory listings.
- A **NAP consistency audit** that extracts the same business from two or three public sources, normalizes name / address / phone, diffs them, and flags mismatches that hurt local SEO.
- A **review monitor** that snapshots `rating` and `review_count` per run so you can surface rating drops and review spikes over time.
- A two-stage pipeline: **(1)** a map/directory search-results page → `suggest_fields` → extract the list of businesses with detail URLs; **(2)** **batch-extract** the detail pages for website, hours, full address, and phone; **(3)** normalize + cross-source NAP diff + store + diff against the previous run.
- The same workflow, no-code, via the Thunderbit Chrome extension for non-developer teammates.

This is overwhelmingly **business-level** data (storefront name, public phone, opening hours). The one edge to watch is owner / proprietor names that some directories publish — those are personal data; the "Responsible use" section covers the GDPR/CCPA caution.

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
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and ad-hoc audits. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. Best for the scheduled NAP-audit and review-monitor pipeline. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, saving/reusing schemas. Best for prototyping and glue scripts. |

A good real-world split: **explore** in MCP, **freeze a schema** with the CLI, **run it weekly** over the HTTP API.

---

## Step 1 — discover the schema

Don't guess field names. Let the AI inspect the page and propose a schema. Throughout this tutorial, treat `https://www.example-localdir.com/search?q=coffee&city=austin-tx` as **a public local directory you are authorized to access**. The same pattern works on any public map/directory listings page — Google Maps/Search public business listings, Yelp public pages, public local directories — but read the "Responsible use" section first, and check the site's `robots.txt` and Terms of Service before you point a scraper at it.

`suggest_fields` costs **1 credit** and returns an array of field descriptors you can drop straight into an extract schema.

### (a) MCP — let the assistant call the tool

Prompt your assistant:

> Use Thunderbit to suggest fields for `https://www.example-localdir.com/search?q=coffee&city=austin-tx`. Focus on each business result card: business name, category, address, phone, star rating, number of reviews, website, and the link to the detail page.

The model issues a `thunderbit_suggest_fields` call with arguments like:

```json
{
  "url": "https://www.example-localdir.com/search?q=coffee&city=austin-tx",
  "prompt": "Extract each business card: business name, category, address, phone, rating, review count, website, detail-page link",
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
    "url": "https://www.example-localdir.com/search?q=coffee&city=austin-tx",
    "prompt": "Extract each business card: business name, category, address, phone, rating, review count, website, detail-page link",
    "country_code": "US"
  }'
```

### (c) CLI

```bash
thunderbit suggest-fields "https://www.example-localdir.com/search?q=coffee&city=austin-tx" \
  --prompt "Extract each business card: business name, category, address, phone, rating, review count, website, detail-page link" \
  --country-code US
```

### The suggested-fields response

The fields come back at `data` (one array of descriptors):

```json
{
  "data": [
    { "name": "business_name", "type": "TEXT",   "instruction": "The business name as shown on the result card" },
    { "name": "category",      "type": "TEXT",   "instruction": "Primary business category, e.g. Coffee Shop, Cafe" },
    { "name": "address",       "type": "TEXT",   "instruction": "Street address shown on the card" },
    { "name": "phone",         "type": "TEXT",   "instruction": "Public phone number as displayed" },
    { "name": "rating",        "type": "NUMBER", "instruction": "Average star rating, 0-5, one decimal" },
    { "name": "review_count",  "type": "NUMBER", "instruction": "Total number of reviews, digits only" },
    { "name": "website",       "type": "URL",    "instruction": "Business website link if shown" },
    { "name": "detail_url",    "type": "URL",    "instruction": "Absolute link to the business detail page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;` (JS) or `fields = res["data"].get("fields", res["data"])` (Python).

Treat the suggestion as a starting point. You'll usually tighten the `instruction` strings — the single biggest lever on extraction quality. Note that `phone` is a `TEXT` field, not `NUMBER`: phone numbers carry formatting (parentheses, dashes, country codes) you want to preserve and normalize later, not coerce into a bare integer.

---

## Step 2 — extract the listings

Now turn the search-results page into structured rows. The schema is **not** raw JSON Schema; it's the Thunderbit-flavored shape: an array of objects, where each property has an UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction`. Extract costs **20 credits** per page.

For a map/directory results page we want enough to identify each business, capture the review signals, and grab the `detail_url` we'll follow in Step 3:

### (a) HTTP — `curl`

Map-driven and SPA directories render their cards in the browser, so set `renderMode:"full"` (headless browser) and give lazy content a moment with `waitFor`. Set `country_code` to route the request from the right locale so you get the local-language and local-format listings.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.example-localdir.com/search?q=coffee&city=austin-tx",
    "renderMode": "full",
    "waitFor": 3000,
    "timeout": 90000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "business_name": { "type": "TEXT",   "instruction": "The business name exactly as shown on the result card" },
          "category":      { "type": "TEXT",   "instruction": "Primary business category, e.g. Coffee Shop, Cafe, Bakery" },
          "address":       { "type": "TEXT",   "instruction": "Street address shown on the card, as a single line" },
          "phone":         { "type": "TEXT",   "instruction": "Public phone number as displayed, keep original formatting" },
          "rating":        { "type": "NUMBER", "instruction": "Average star rating from 0 to 5, keep one decimal like 4.6" },
          "review_count":  { "type": "NUMBER", "instruction": "Total number of reviews, digits only, no commas" },
          "website":       { "type": "URL",    "instruction": "Business website link if shown on the card, else leave empty" },
          "detail_url":    { "type": "URL",    "instruction": "Absolute link to this business's detail page" }
        }
      }
    }
  }'
```

Rows come back at `data`:

```json
{
  "data": [
    { "business_name": "Maple Street Coffee", "category": "Coffee Shop", "address": "1204 Maple Ave, Austin, TX 78704", "phone": "(512) 555-0142", "rating": 4.6, "review_count": 318, "website": "https://maplestreetcoffee.example", "detail_url": "https://www.example-localdir.com/b/maple-street-coffee" },
    { "business_name": "Cedar Roasters",      "category": "Cafe",        "address": "88 Cedar St, Austin, TX 78702",    "phone": "512-555-0199",   "rating": 4.3, "review_count": 142, "website": "",                                  "detail_url": "https://www.example-localdir.com/b/cedar-roasters" }
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
            "business_name": {"type": "TEXT",   "instruction": "The business name exactly as shown on the result card"},
            "category":      {"type": "TEXT",   "instruction": "Primary business category, e.g. Coffee Shop, Cafe, Bakery"},
            "address":       {"type": "TEXT",   "instruction": "Street address shown on the card, as a single line"},
            "phone":         {"type": "TEXT",   "instruction": "Public phone number as displayed, keep original formatting"},
            "rating":        {"type": "NUMBER", "instruction": "Average star rating from 0 to 5, keep one decimal like 4.6"},
            "review_count":  {"type": "NUMBER", "instruction": "Total number of reviews, digits only, no commas"},
            "website":       {"type": "URL",    "instruction": "Business website link if shown on the card, else leave empty"},
            "detail_url":    {"type": "URL",    "instruction": "Absolute link to this business's detail page"},
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

businesses = extract("https://www.example-localdir.com/search?q=coffee&city=austin-tx", LISTINGS_SCHEMA)
print(len(businesses), "businesses")
```

### (c) CLI — save the schema, then reuse it

Save the schema to a file once, then re-run it any time:

```bash
# Save the listings schema to listings.schema.json (an array-of-objects schema as above),
# then extract with it:
thunderbit extract "https://www.example-localdir.com/search?q=coffee&city=austin-tx" \
  --schema listings.schema.json \
  --render-mode full \
  --timeout 90000 \
  -f json > businesses.json
```

> **`renderMode` cheat sheet:** `none` = raw HTML fetch (fastest/cheapest render; fine for server-rendered pages); `basic` = light JS; `full` = headless browser for JS-heavy/map SPAs. If a page comes back empty, the first thing to try is `renderMode:"full"` plus a larger `waitFor`. The valid `waitFor` range is `0–10000` ms; `extract` `timeout` is `5000–120000` ms (default `60000`).

---

## Step 3 — batch-extract detail pages

The results card rarely has everything — full street address (with suite), opening hours, the canonical website, and sometimes a cleaner phone usually live on the business detail page. Collect the `detail_url` values from Step 2 and process them with **batch extract**, which handles **up to 100 URLs per job** at **20 credits/URL**.

Define a richer detail schema. This is the schema whose `address` / `phone` / `business_name` fields feed the NAP audit, so the instructions are deliberately precise:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "business_name": { "type": "TEXT",   "instruction": "Official business name as displayed on the detail page header" },
      "category":      { "type": "TEXT",   "instruction": "Primary business category or type" },
      "address":       { "type": "TEXT",   "instruction": "Full street address on one line including suite, city, state, ZIP" },
      "phone":         { "type": "TEXT",   "instruction": "Primary public phone number, keep original formatting" },
      "rating":        { "type": "NUMBER", "instruction": "Average star rating from 0 to 5, keep one decimal" },
      "review_count":  { "type": "NUMBER", "instruction": "Total number of reviews, digits only" },
      "website":       { "type": "URL",    "instruction": "Canonical business website URL if publicly linked" },
      "hours":         { "type": "TEXT",   "instruction": "Opening hours for each day as one line, e.g. 'Mon-Fri 7am-6pm; Sat-Sun 8am-4pm'" }
    }
  }
}
```

Because each detail page is one business, the engine returns a single record per URL (still inside the array shape).

### (a) HTTP — create the job, then poll

```bash
# 1. Create the batch job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://www.example-localdir.com/b/maple-street-coffee",
      "https://www.example-localdir.com/b/cedar-roasters"
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
      { "url": "https://www.example-localdir.com/b/maple-street-coffee", "status": "completed", "data": [ { "business_name": "Maple Street Coffee", "address": "1204 Maple Ave, Suite B, Austin, TX 78704", "phone": "(512) 555-0142", "hours": "Mon-Fri 7am-6pm; Sat-Sun 8am-4pm" } ] },
      { "url": "https://www.example-localdir.com/b/cedar-roasters",      "status": "failed",    "error": "timeout" }
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

## NAP consistency audit

This is the heart of local-SEO hygiene. The same business appears on multiple public sources, and each may format — or get wrong — the **N**ame, **A**ddress, and **P**hone. Search engines reward consistency, so your job is to extract the same business from two or three public sources, **normalize** each NAP field to a canonical form, **diff** them, and **flag** the mismatches.

The trick is that most mismatches are *cosmetic* (`St` vs `Street`, `Ste` vs `Suite`, `(512) 555-0142` vs `512.555.0142`) and harmless, while a few are *real* (a stale phone, a moved suite number, a misspelled name). Normalization separates the two so you only chase the real ones.

### Extract the business from each source

Run the same single-record schema against each source URL for one business. Use a flat object schema (one business per page) — or the array form, which still returns one record per URL:

```python
NAP_SCHEMA = {
    "type": "object",
    "properties": {
        "business_name": {"type": "TEXT", "instruction": "Official business name as displayed, no taglines"},
        "address":       {"type": "TEXT", "instruction": "Full street address on one line including suite, city, state, ZIP"},
        "phone":         {"type": "TEXT", "instruction": "Primary public phone number, keep original formatting"},
    },
}

sources = {
    "directory": "https://www.example-localdir.com/b/maple-street-coffee",
    "yelp":      "https://www.yelp.com/biz/maple-street-coffee-austin",   # public page
    "maps":      "https://www.google.com/maps/place/Maple+Street+Coffee", # public listing
}
```

> Confirm each source's `robots.txt`/ToS allows automated access before adding it. Yelp public pages, Google Maps/Search public business listings, and public local directories are the kinds of public sources this audit targets.

### Normalize + compare

The runnable normalize-and-diff core. Pure functions, easy to unit-test:

```python
import re

# Common US street-suffix and unit abbreviations → canonical form.
_STREET = {
    "street": "st", "avenue": "ave", "boulevard": "blvd", "drive": "dr",
    "road": "rd", "lane": "ln", "court": "ct", "suite": "ste", "unit": "ste",
    "north": "n", "south": "s", "east": "e", "west": "w",
}

def norm_name(name: str) -> str:
    """Lowercase, strip punctuation and common suffixes, collapse whitespace."""
    if not name:
        return ""
    s = name.lower()
    s = re.sub(r"\b(inc|llc|ltd|co|corp|the)\b", " ", s)
    s = re.sub(r"[^a-z0-9 ]", " ", s)
    return re.sub(r"\s+", " ", s).strip()

def norm_phone(phone: str) -> str:
    """Keep digits only; drop a leading US country code so formats compare equal."""
    if not phone:
        return ""
    digits = re.sub(r"\D", "", phone)
    if len(digits) == 11 and digits.startswith("1"):
        digits = digits[1:]
    return digits

def norm_address(addr: str) -> str:
    """Lowercase, expand/standardize suffixes, drop punctuation, collapse whitespace."""
    if not addr:
        return ""
    s = addr.lower()
    s = re.sub(r"[.,#]", " ", s)
    tokens = [_STREET.get(t, t) for t in s.split()]
    return re.sub(r"\s+", " ", " ".join(tokens)).strip()

def nap_audit(by_source: dict) -> dict:
    """by_source: {source_name: {business_name, address, phone}}.
    Returns canonical values, per-field consensus, and the list of mismatches."""
    fields = {
        "name":    (norm_name,    "business_name"),
        "address": (norm_address, "address"),
        "phone":   (norm_phone,   "phone"),
    }
    report, mismatches = {}, []
    for field, (fn, key) in fields.items():
        normalized = {src: fn(rec.get(key, "")) for src, rec in by_source.items()}
        distinct = set(v for v in normalized.values() if v)
        consistent = len(distinct) <= 1
        report[field] = {
            "raw":        {src: rec.get(key, "") for src, rec in by_source.items()},
            "normalized": normalized,
            "consistent": consistent,
        }
        if not consistent:
            mismatches.append({"field": field, "values": normalized})
    report["nap_consistent"] = len(mismatches) == 0
    report["mismatches"] = mismatches
    return report
```

A worked result: if the directory says `1204 Maple Ave, Suite B` and Yelp says `1204 Maple Avenue Ste B`, both normalize to `1204 maple ave ste b` → **consistent** (cosmetic difference, no action). But if Maps still shows phone `512-555-0100` while the directory and Yelp agree on `512-555-0142`, the phone field is flagged as a **real mismatch** worth fixing — exactly the kind of stale NAP signal that drags on local rankings.

---

## End-to-end pipeline

Here's one cohesive, runnable Python script that ties it all together: extract the directory search page, batch-extract details (in chunks of 100), run the cross-source NAP diff for each business, write a SQLite snapshot, and diff the review counts against the previous run to surface **rating drops** and **review spikes**. Error handling follows the reference: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Weekly local-business intelligence pipeline built on the Thunderbit Open API.
   Builds a dataset, audits NAP across public sources, and monitors review trends."""
import os, re, time, json, random, sqlite3, datetime
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
SEARCH_URL = "https://www.example-localdir.com/search?q=coffee&city=austin-tx"  # a public directory you're authorized to access
DB = "localbiz.sqlite"

LISTINGS_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "business_name": {"type": "TEXT",   "instruction": "Business name exactly as shown on the result card"},
        "category":      {"type": "TEXT",   "instruction": "Primary business category, e.g. Coffee Shop"},
        "address":       {"type": "TEXT",   "instruction": "Street address shown on the card, single line"},
        "phone":         {"type": "TEXT",   "instruction": "Public phone number as displayed, keep formatting"},
        "rating":        {"type": "NUMBER", "instruction": "Average star rating 0-5, one decimal"},
        "review_count":  {"type": "NUMBER", "instruction": "Total number of reviews, digits only"},
        "website":       {"type": "URL",    "instruction": "Business website link if shown, else empty"},
        "detail_url":    {"type": "URL",    "instruction": "Absolute link to this business's detail page"},
    }},
}

DETAIL_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "business_name": {"type": "TEXT",   "instruction": "Official business name on the detail page header"},
        "category":      {"type": "TEXT",   "instruction": "Primary business category or type"},
        "address":       {"type": "TEXT",   "instruction": "Full street address on one line incl suite, city, state, ZIP"},
        "phone":         {"type": "TEXT",   "instruction": "Primary public phone number, keep original formatting"},
        "rating":        {"type": "NUMBER", "instruction": "Average star rating 0-5, one decimal"},
        "review_count":  {"type": "NUMBER", "instruction": "Total number of reviews, digits only"},
        "website":       {"type": "URL",    "instruction": "Canonical business website URL if publicly linked"},
        "hours":         {"type": "TEXT",   "instruction": "Opening hours as one line, e.g. 'Mon-Fri 7am-6pm; Sat 8am-4pm'"},
    }},
}

# --- NAP normalization (see the audit section) ---
_STREET = {"street":"st","avenue":"ave","boulevard":"blvd","drive":"dr","road":"rd",
           "lane":"ln","court":"ct","suite":"ste","unit":"ste",
           "north":"n","south":"s","east":"e","west":"w"}

def norm_name(s):
    if not s: return ""
    s = re.sub(r"\b(inc|llc|ltd|co|corp|the)\b", " ", s.lower())
    return re.sub(r"\s+", " ", re.sub(r"[^a-z0-9 ]", " ", s)).strip()

def norm_phone(s):
    d = re.sub(r"\D", "", s or "")
    return d[1:] if len(d) == 11 and d.startswith("1") else d

def norm_address(s):
    if not s: return ""
    s = re.sub(r"[.,#]", " ", s.lower())
    return re.sub(r"\s+", " ", " ".join(_STREET.get(t, t) for t in s.split())).strip()

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
            time.sleep(min(60, 2 ** i) + random.uniform(0, 1))  # exponential backoff + jitter
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
        "renderMode": "full", "waitFor": 3000, "timeout": 90000, "country_code": "US",
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

def nap_audit_one(detail_row, extra_sources):
    """Diff this business's NAP across the directory record + any extra public sources.
       extra_sources: list of detail rows for the SAME business from other sources."""
    by_source = {"directory": detail_row}
    for i, rec in enumerate(extra_sources):
        by_source[rec.get("_source", f"source{i}")] = rec
    fns = {"name": (norm_name, "business_name"),
           "address": (norm_address, "address"),
           "phone": (norm_phone, "phone")}
    mismatches = []
    for field, (fn, key) in fns.items():
        distinct = {fn(r.get(key, "")) for r in by_source.values() if r.get(key)}
        if len(distinct) > 1:
            mismatches.append({"field": field,
                               "values": {s: r.get(key, "") for s, r in by_source.items()}})
    return mismatches

def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS snapshots (
        run_date TEXT, detail_url TEXT, business_name TEXT, category TEXT,
        address TEXT, phone TEXT, rating REAL, review_count REAL,
        website TEXT, hours TEXT, nap_mismatches TEXT,
        PRIMARY KEY (run_date, detail_url))""")
    con.commit()
    return con

def previous_reviews(con):
    """Most recent prior run's rating + review_count keyed by detail_url."""
    cur = con.execute("SELECT detail_url, rating, review_count FROM snapshots "
                      "WHERE run_date = (SELECT MAX(run_date) FROM snapshots)")
    return {url: (rating, rc) for url, rating, rc in cur.fetchall()}

def save_snapshot(con, run_date, rows):
    for r in rows:
        con.execute("INSERT OR REPLACE INTO snapshots VALUES (?,?,?,?,?,?,?,?,?,?,?)", (
            run_date, r.get("detail_url"), r.get("business_name"), r.get("category"),
            r.get("address"), r.get("phone"), r.get("rating"), r.get("review_count"),
            r.get("website"), r.get("hours"), json.dumps(r.get("nap_mismatches", []))))
    con.commit()

def review_diff(prev, rows):
    rating_drops, review_spikes = [], []
    for r in rows:
        url = r.get("detail_url")
        if url not in prev:
            continue
        old_rating, old_rc = prev[url]
        new_rating, new_rc = r.get("rating"), r.get("review_count")
        if old_rating and new_rating and new_rating < old_rating:
            rating_drops.append({"business": r.get("business_name"),
                                 "old": old_rating, "new": new_rating, "url": url})
        if old_rc and new_rc and new_rc - old_rc >= 10:
            review_spikes.append({"business": r.get("business_name"),
                                  "delta": new_rc - old_rc, "url": url})
    return rating_drops, review_spikes

def main():
    run_date = datetime.date.today().isoformat()
    con = init_db()
    prev = previous_reviews(con)

    listings = extract_listings()
    detail_urls = [b["detail_url"] for b in listings if b.get("detail_url")]
    print(f"Found {len(detail_urls)} businesses on the search page")

    rows = batch_extract_details(detail_urls)
    print(f"Extracted {len(rows)} detail pages")

    # NAP audit. In production, extra_sources would be batch-extracted Yelp/Maps
    # records matched to the same business (by normalized name + address).
    flagged = 0
    for r in rows:
        r["nap_mismatches"] = nap_audit_one(r, extra_sources=[])
        if r["nap_mismatches"]:
            flagged += 1
    print(f"{flagged} businesses have NAP mismatches across sources")

    save_snapshot(con, run_date, rows)

    rating_drops, review_spikes = review_diff(prev, rows)
    print(f"\n{len(rating_drops)} RATING DROPS, {len(review_spikes)} REVIEW SPIKES")
    for d in rating_drops:
        print(f"  ↓ {d['business']}: {d['old']} → {d['new']} ({d['url']})")

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire this to your alerting (PagerDuty/Slack/email)
        raise
```

Run it, and on week two it will print exactly which businesses lost rating, which gained a burst of reviews, and which carry NAP mismatches worth fixing.

---

## No-code version: the Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper** Chrome extension. Non-developer teammates — local-SEO specialists, agency account managers — can run the identical workflow without writing code, and the schema they build transfers verbatim to the API.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **AI Suggest Fields.** Open the public directory or map results page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns (business name, category, address, phone, rating, review count, website, detail link).
3. **Edit columns** in plain language. Rename, drop, or add columns; tune each column's instruction exactly like the `instruction` strings in the API schema (e.g. "phone as displayed, keep formatting").
4. **Scrape**, then **Scrape Subpages** to follow each row's `detail_url` and pull hours, full address, and the canonical website — the no-code equivalent of the two-stage list → detail batch pipeline in Steps 2–3.
5. **Export** to Excel, Google Sheets, Airtable, or Notion, or **schedule** recurring runs. Run the same scrape against each public source, then compare the exported sheets to spot NAP drift.

The recommended flow: design and validate the schema interactively in the extension, then port the field names and instructions to the CLI/API for automation at scale. They map one-to-one.

---

## Automate it

- **Cron.** Run the pipeline weekly. The review-diff logic already compares against the most recent prior snapshot stored in SQLite:

  ```cron
  # Every Monday at 06:15, run the local-business pipeline and append a log
  15 6 * * 1  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/localbiz/pipeline.py >> /var/log/localbiz.log 2>&1
  ```

- **Batch webhooks instead of polling.** For server-side jobs, supply a callback so Thunderbit notifies you when the batch finishes — no open connection needed:

  ```json
  {
    "urls": ["https://www.example-localdir.com/b/maple-street-coffee", "..."],
    "schema": { "type": "array", "items": { "type": "object", "properties": { } } },
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "whatever-signing-secret-you-choose"
    }
  }
  ```

  Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call, then write the results to your store.

- **Store snapshots.** Keep one row per `(run_date, detail_url)` (as the script does). That history powers NAP-drift detection, rating-trend charts, and review-velocity tracking over time.
- **Alert on NAP drift or rating drops.** Pipe the flagged `nap_mismatches` and `rating_drops` lists to Slack/email. A newly introduced phone mismatch on a high-traffic location, or a rating that slipped below a threshold, is worth a same-day notification.

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit** *(some docs list it as free — confirm against your plan)*, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**Worked example.** A weekly run over **300 businesses**, auditing each across **3 public sources**:

```
300 businesses × 3 sources × 20 credits = 18,000 credits / week
+ 1 directory search-page extract × 20  =     20 credits
+ 1 suggest_fields (one-time)           =      1 credit
≈ 18,020 credits / week
```

**Extract vs distill — a 20× decision.** Extract (structured JSON) is 20× the price of distill (clean Markdown). If you only need *clean text* — say, to summarize the latest reviews for sentiment analysis — distill those review pages for **1 credit each** instead of 20. Reserve extract for the structured NAP fields you'll actually diff (name, address, phone, rating, review_count).

**Throttle & cache to cut both cost and load:**

- **Dedupe by `(name + address)`.** The same business often appears multiple times across sources and across paginated result pages. Key your dataset on normalized `(business_name + address)` so you extract each business detail once per source, not repeatedly.
- **Stage cheaply.** The search-page extract is one call that yields many `detail_url`s; only batch-extract the details you don't already have fresh in your snapshot.
- **Reasonable concurrency.** Batch caps at 100 URLs/job; chunk larger sets and don't fire many jobs at one origin simultaneously. Add jitter between scheduled runs.

---

## Responsible use

These guardrails apply to every workflow in this repo — local business intelligence included.

- **Check `robots.txt` and Terms of Service first.** Before pointing a scraper at any map, directory, or review site, confirm it permits automated access to the pages you target. Honor disallowed paths and crawl-rate directives.
- **Public data only.** Thunderbit targets publicly accessible pages. Google Maps/Search public business listings, Yelp public pages, and public local directories are fair game; never build workflows against login-walled platforms or private/authenticated areas.
- **Prefer business-level data.** Storefront name, public phone, address, hours, rating, and review counts are business facts, not personal data — that's what this audit is built around. Keep the schema focused on those.
- **Personal-data caution.** Some directories publish an **owner / proprietor name** or a personal mobile number. Those are personal data. Collect them **only when publicly posted**, and only if you have a lawful basis to process them (GDPR/CCPA). If you don't need them, drop those fields from the schema entirely — the NAP audit doesn't require them.
- **Throttle.** Use batch with reasonable concurrency; don't hammer a single origin. Add jitter to scheduled runs.
- **Cache and dedupe** to avoid re-scraping unchanged pages — it saves credits and reduces load on the source.

---

## Troubleshooting & FAQ

**Extraction returns an empty array.** Map and directory UIs are almost always client-rendered. Set `renderMode:"full"` (headless browser) and raise `waitFor` (up to its `10000` ms ceiling) so lazy map pins and result cards load. Also raise `timeout` (extract allows up to `120000` ms). If you're on `renderMode:"none"` against a map SPA, that's the cause.

**Addresses won't match in the NAP diff even though they look the same.** They differ only cosmetically (`St` vs `Street`, `Ste` vs `Suite`, trailing punctuation). Two fixes that stack: (1) tighten the extract `instruction` to normalize at the source — e.g. "Full street address on one line; expand abbreviations to full words; include suite, city, state, ZIP"; and (2) run the `norm_address` function before comparing. Source-side `instruction` normalization plus client-side normalization catches the long tail.

**Phone numbers compare as different across sources.** Strip everything but digits and drop a leading US `1` before comparing (the `norm_phone` function does this). Keep the raw value in the dataset for display; compare on the normalized digits only.

**Pagination / "load more" on the search page.** Most directories paginate via a query parameter (`?page=2`) or a map viewport. Enumerate the page URLs and feed them as a list to **batch extract** with the listings schema, then merge. For infinite-scroll pages, increasing `waitFor` loads more cards, but URL-based pagination is more reliable than scrolling.

**Duplicate businesses in the dataset.** The same shop appears on multiple result pages and across sources. Dedupe by normalized `(business_name + address)` — `norm_name(b["business_name"]) + "|" + norm_address(b["address"])` makes a stable key. This is also what lets you align the same business across sources for the NAP audit.

**Locale / regional listings look wrong.** Set `country_code` (HTTP/CLI) or `countryCode` (MCP) to route the request from the right region — e.g. `US`, `GB`, `DE`. The default is `US`. This matters when a directory serves different businesses, languages, or phone formats by visitor location.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status`; re-submit only the `failed` URLs in a fresh batch job. Don't re-run the whole batch — you'd pay credits again for the rows that already succeeded.

**Ratings or review counts come back as strings (e.g. "4.6 stars", "1,204 reviews").** Tighten the `instruction`: "average star rating 0–5, one decimal, digits only" and "total reviews, digits only, no commas." For NUMBER fields, the instruction is what coerces the value — brief it like you'd brief a human assistant.

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.
