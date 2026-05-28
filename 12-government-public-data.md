# Government & Public Data: Tender, Grant & Procurement Tracking with Thunderbit

Monitor public procurement and grant opportunities across federal, EU, and national portals; turn long solicitation documents into LLM-ready summaries; and get a daily alert the moment a new tender matching your keywords is posted — all from public government sources, using one AI web-data engine reachable from MCP, HTTP, or CLI.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## The data problem in public procurement

Bid and grant teams lose deals to plumbing, not strategy. The pains are concrete:

- **Opportunities are scattered across dozens of portals.** A single company might track SAM.gov (US federal contracting), Grants.gov (US federal grants), TED (EU tenders), plus a long tail of state, municipal, and national procurement portals — each with its own HTML, its own field names, and its own search UI. There is no single clean feed for "every new IT-services RFP under $5M in my categories."
- **Tight, unforgiving deadlines.** Solicitations open and close on fixed dates. A response window can be as short as 10 business days. Missing a posting by 48 hours can mean missing the bid entirely — there is no extension for "we didn't see it."
- **Long, dense solicitation documents.** A single RFP can be a 120-page PDF or a sprawling HTML page of eligibility rules, scope, evaluation criteria, and submission instructions. Reading every one by hand to decide "is this worth bidding?" doesn't scale.
- **JS-heavy, paginated search UIs.** Many modern portals render results client-side, lazy-load rows, and hide the real detail behind notice pages. A naive HTTP `GET` returns an empty shell, so you need a real headless browser to see what a human sees.
- **No memory.** Without a stored snapshot, you re-read the same notices every day and still can't reliably answer "what is *new* since yesterday?"

The fix is a small, repeatable pipeline: discover the schema once, extract the search-results page into a list of opportunities, follow each notice's detail link in a batch, distill the long solicitation text into a summary, store the result, and diff it against yesterday to surface what's new. Thunderbit gives you the extraction engine; this tutorial gives you the pipeline.

---

## What you'll build

- An **opportunity tracker** — a structured dataset of tenders/RFPs and grants with title, issuing agency, reference ID, category, value, currency, deadline, posted date, and detail-page URL.
- A **solicitation summarizer** — distill long notice pages into clean Markdown, then feed that to an LLM to summarize scope, eligibility, and submission requirements (the cheap path for RAG and triage; see the [RAG tutorial](07-rag-knowledge-base.md)).
- A **new-opportunity alert** — a daily (or twice-daily) diff that compares today's list against a stored set keyed by `reference_id`/`detail_url`, surfaces only the genuinely new postings, and emits them with a deadline countdown so the bid team can triage in minutes.
- A two-stage pipeline: **(1)** a tender/grant search-results page → `suggest_fields` → extract the list of opportunities with detail URLs; **(2)** **batch-extract** the detail pages for structured fields and **batch-distill** the long solicitation text for summaries; **(3)** store + diff against the previous run → alert.
- The same workflow, no-code, via the Thunderbit Chrome extension for non-developer teammates on the bid desk.

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

4. **Prefer official open data where it exists.** Government data is largely public and open, and many portals publish *better* access paths than their HTML: SAM.gov has a [Get Opportunities API](https://open.gsa.gov/api/get-opportunities-public-api/) and daily bulk extracts, Grants.gov offers a Search2 API and XML extracts, and TED publishes [bulk notice downloads and an API](https://docs.ted.europa.eu/). When an official API or bulk download exists, use it — it's more stable, faster, and the intended channel. Reach for Thunderbit to extract portals that have *no* usable API, to normalize many heterogeneous portals into one schema, and to distill long solicitation HTML/PDFs into LLM-ready text. For solicitation-document RAG, see the [RAG tutorial](07-rag-knowledge-base.md).

For full endpoint, schema, pricing, and error details, keep the [API reference](docs/thunderbit-api-reference.md) open in a tab.

---

## Three ways to call Thunderbit

All three surfaces hit the identical HTTP API underneath, so schemas and field instructions transfer verbatim between them.

| Surface | Entry point | When to use it |
|---------|-------------|----------------|
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and ad-hoc pulls. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. Best for the automated daily monitor. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, saving/reusing schemas. Best for prototyping and glue scripts. |

A good real-world split: **explore** a new portal in MCP, **freeze a schema** with the CLI, **run it twice a day** over the HTTP API.

---

## Step 1 — discover the schema

Don't guess field names. Let the AI inspect the search-results page and propose a schema. Throughout this tutorial, treat `https://www.example-procurement.gov/search?q=cloud+services` as **a public procurement portal you are authorized to access** (SAM.gov, Grants.gov, TED, and similar national portals are real public examples of this kind of page). The same pattern works on any public tender or grant search page — but read the "Responsible use" section first, prefer an official API where one exists, and check the site's `robots.txt` and Terms of Service before you point a scraper at it.

`suggest_fields` costs **1 credit** and returns an array of field descriptors you can drop straight into an extract schema.

### (a) MCP — let the assistant call the tool

Prompt your assistant:

> Use Thunderbit to suggest fields for `https://www.example-procurement.gov/search?q=cloud+services`. Focus on each opportunity row: notice title, issuing agency, reference/solicitation ID, category, estimated value, currency, response deadline, posted date, and the link to the notice's detail page.

The model issues a `thunderbit_suggest_fields` call with arguments like:

```json
{
  "url": "https://www.example-procurement.gov/search?q=cloud+services",
  "prompt": "Extract each opportunity row: title, agency, reference id, category, value, currency, deadline, posted date, detail-page link",
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
    "url": "https://www.example-procurement.gov/search?q=cloud+services",
    "prompt": "Extract each opportunity row: title, agency, reference id, category, value, currency, deadline, posted date, detail-page link",
    "country_code": "US"
  }'
```

### (c) CLI

```bash
thunderbit suggest-fields "https://www.example-procurement.gov/search?q=cloud+services" \
  --prompt "Extract each opportunity row: title, agency, reference id, category, value, currency, deadline, posted date, detail-page link" \
  --country-code US
```

### The suggested-fields response

The fields come back at `data` (one array of descriptors):

```json
{
  "data": [
    { "name": "title",       "type": "TEXT",   "instruction": "The notice / solicitation title as shown on the row" },
    { "name": "agency",      "type": "TEXT",   "instruction": "The issuing department or contracting agency name" },
    { "name": "reference_id","type": "TEXT",   "instruction": "The solicitation, notice, or opportunity reference number, exactly as printed" },
    { "name": "category",    "type": "TEXT",   "instruction": "Procurement category, NAICS/CPV label, or set-aside type if shown" },
    { "name": "value",       "type": "NUMBER", "instruction": "Estimated contract value, digits only, no currency symbol or commas" },
    { "name": "currency",    "type": "TEXT",   "instruction": "ISO currency code of the value, e.g. USD or EUR" },
    { "name": "deadline",    "type": "DATE",   "instruction": "Response/closing deadline date" },
    { "name": "posted_date", "type": "DATE",   "instruction": "Date the notice was posted" },
    { "name": "detail_url",  "type": "URL",    "instruction": "Absolute link to the notice's detail page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;` (JS) or `fields = res["data"].get("fields", res["data"])` (Python).

Treat the suggestion as a starting point. You'll usually tighten the `instruction` strings — the single biggest lever on extraction quality. For government data, be especially explicit about *dates* and *money* (formats vary wildly between portals and locales).

---

## Step 2 — extract the opportunity list

Now turn the search-results page into structured rows. The schema is **not** raw JSON Schema; it's the Thunderbit-flavored shape: an array of objects, where each property has an UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction`. Extract costs **20 credits** per page.

For a tender/grant search page we want enough to identify, value, and time each opportunity — plus the `detail_url` we'll follow in Step 3:

### (a) HTTP — `curl`

Many procurement portals render results client-side and paginate via query string, so set `renderMode:"full"` (headless browser), give lazy content a moment with `waitFor`, and `country_code` for regional portals (e.g. `DE` for a German portal, `GB` for a UK one).

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.example-procurement.gov/search?q=cloud+services",
    "renderMode": "full",
    "waitFor": 2500,
    "timeout": 90000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "title":        { "type": "TEXT",   "instruction": "The notice / solicitation title as shown on the row" },
          "agency":       { "type": "TEXT",   "instruction": "The issuing department or contracting agency name" },
          "reference_id": { "type": "TEXT",   "instruction": "Solicitation / notice / opportunity reference number, exactly as printed" },
          "category":     { "type": "TEXT",   "instruction": "Procurement category, NAICS/CPV label, or set-aside type if shown" },
          "value":        { "type": "NUMBER", "instruction": "Estimated contract value, digits only, no currency symbol or commas; null if not shown" },
          "currency":     { "type": "TEXT",   "instruction": "ISO currency code of the value, e.g. USD or EUR; infer from symbol if needed" },
          "deadline":     { "type": "DATE",   "instruction": "Response/closing deadline; normalize to YYYY-MM-DD" },
          "posted_date":  { "type": "DATE",   "instruction": "Date the notice was posted; normalize to YYYY-MM-DD" },
          "detail_url":   { "type": "URL",    "instruction": "Absolute link to this notice'"'"'s detail page" }
        }
      }
    }
  }'
```

Rows come back at `data`:

```json
{
  "data": [
    { "title": "Cloud Hosting & Managed Services", "agency": "Dept. of Administrative Services", "reference_id": "RFP-2026-0142", "category": "IT Services", "value": 4800000, "currency": "USD", "deadline": "2026-06-18", "posted_date": "2026-05-21", "detail_url": "https://www.example-procurement.gov/notice/RFP-2026-0142" },
    { "title": "Data Center Migration Support",    "agency": "State Health Authority",          "reference_id": "RFP-2026-0151", "category": "IT Services", "value": 1250000, "currency": "USD", "deadline": "2026-06-09", "posted_date": "2026-05-24", "detail_url": "https://www.example-procurement.gov/notice/RFP-2026-0151" }
  ]
}
```

### (b) Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

LIST_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "title":        {"type": "TEXT",   "instruction": "The notice / solicitation title as shown on the row"},
        "agency":       {"type": "TEXT",   "instruction": "The issuing department or contracting agency name"},
        "reference_id": {"type": "TEXT",   "instruction": "Solicitation / notice / opportunity reference number, exactly as printed"},
        "category":     {"type": "TEXT",   "instruction": "Procurement category, NAICS/CPV label, or set-aside type if shown"},
        "value":        {"type": "NUMBER", "instruction": "Estimated contract value, digits only, no symbol or commas; null if not shown"},
        "currency":     {"type": "TEXT",   "instruction": "ISO currency code of the value, e.g. USD or EUR; infer from symbol if needed"},
        "deadline":     {"type": "DATE",   "instruction": "Response/closing deadline; normalize to YYYY-MM-DD"},
        "posted_date":  {"type": "DATE",   "instruction": "Date the notice was posted; normalize to YYYY-MM-DD"},
        "detail_url":   {"type": "URL",    "instruction": "Absolute link to this notice's detail page"},
    }},
}

def extract(url, schema, render_mode="full", timeout=90000, wait_for=2500, country_code="US"):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema,
        "renderMode": render_mode, "timeout": timeout, "waitFor": wait_for,
        "country_code": country_code,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

opps = extract("https://www.example-procurement.gov/search?q=cloud+services", LIST_SCHEMA)
print(len(opps), "opportunities")
```

### (c) CLI — save the schema, then reuse it

Save the schema to a file once, then re-run it any time:

```bash
# Save the list schema to opportunities.schema.json (an array-of-objects schema as above),
# then extract with it:
thunderbit extract "https://www.example-procurement.gov/search?q=cloud+services" \
  --schema opportunities.schema.json \
  --render-mode full \
  --timeout 90000 \
  -f json > opportunities.json
```

> **`renderMode` cheat sheet:** `none` = raw HTML fetch (fastest/cheapest render; fine for server-rendered government pages, which many still are); `basic` = light JS; `full` = headless browser for JS-heavy SPAs. If a page comes back empty, the first thing to try is `renderMode:"full"` plus a larger `waitFor`. The valid `waitFor` range is `0–10000` ms; `extract` `timeout` is `5000–120000` ms (default `60000`).

> **Regional portals.** For a German tender portal set `country_code:"DE"` (HTTP/CLI) or `countryCode:"DE"` (MCP); for a UK one, `GB`. The default is `US`. This routes the request from the right region, which matters when a portal serves localized content or restricts traffic by geography.

---

## Step 3 — batch-extract detail pages + distill solicitations

The search row rarely has everything — full scope, eligibility, set-aside details, contact, and the actual solicitation text live on the notice detail page. You'll do **two** things with each detail URL:

1. **Batch-extract** structured fields you'll query and alert on (deadline, value, contact).
2. **Batch-distill** the long solicitation text into clean Markdown for an LLM to summarize — this is the *cheap* path (1 credit/URL vs 20 for extract).

### 3a. Batch-extract the structured detail

**Batch extract** handles **up to 100 URLs per job** at **20 credits/URL**. Define a richer detail schema:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "title":         { "type": "TEXT",   "instruction": "Full official notice title" },
      "agency":        { "type": "TEXT",   "instruction": "Issuing department / contracting office" },
      "reference_id":  { "type": "TEXT",   "instruction": "Solicitation / notice reference number, exactly as printed" },
      "category":      { "type": "TEXT",   "instruction": "Procurement category, NAICS/CPV code or label, set-aside type" },
      "value":         { "type": "NUMBER", "instruction": "Estimated or ceiling contract value, digits only, no symbol; null if not stated" },
      "currency":      { "type": "TEXT",   "instruction": "ISO currency code, e.g. USD, EUR, GBP" },
      "deadline":      { "type": "DATE",   "instruction": "Response/closing deadline; normalize to YYYY-MM-DD" },
      "posted_date":   { "type": "DATE",   "instruction": "Date the notice was posted; normalize to YYYY-MM-DD" },
      "place_of_perf": { "type": "TEXT",   "instruction": "Place of performance / delivery location if stated" },
      "contact_email": { "type": "EMAIL",  "instruction": "Contracting officer email ONLY if publicly published in the notice" },
      "detail_url":    { "type": "URL",    "instruction": "Canonical URL of this notice" }
    }
  }
}
```

> **Personal-data note.** A named contracting officer's email is borderline personal data. Government notices publish it as the official point of contact, which is generally a lawful basis, but collect it **only when publicly published in the notice**, use it only for the procurement it relates to, and drop the field if you don't need it. See "Responsible use."

#### HTTP — create the job, then poll

```bash
# 1. Create the batch job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://www.example-procurement.gov/notice/RFP-2026-0142",
      "https://www.example-procurement.gov/notice/RFP-2026-0151"
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
      { "url": "https://www.example-procurement.gov/notice/RFP-2026-0142", "status": "completed", "data": [ { "reference_id": "RFP-2026-0142", "value": 4800000, "deadline": "2026-06-18" } ] },
      { "url": "https://www.example-procurement.gov/notice/RFP-2026-0151", "status": "failed",    "error": "timeout" }
    ]
  }
}
```

Job-level `status` is `pending | processing | completed | failed | cancelled`. Per-URL `results[].status` lets you re-submit only those URLs that came back `failed` — don't re-run the whole batch, or you'll pay credits again for rows that already succeeded.

#### CLI — batch from a file

`--file` reads one URL per line and ignores blank lines and lines starting with `#`.

```bash
# detail_urls.txt has one notice URL per line
thunderbit batch extract --file detail_urls.txt \
  --schema detail.schema.json \
  --timeout 90000 \
  -f json > details.json
```

#### MCP

Ask your assistant: *"Use Thunderbit to batch-extract these notice URLs with this schema, then poll until done."* It calls `thunderbit_batch_extract_create` (args `urls`, `schema`, `timeout?`) and then `thunderbit_batch_extract_status` (args `jobId`, `page?`, `pageSize?`) until the job is `completed`.

### 3b. Batch-distill the solicitation text → LLM summary

To decide whether an opportunity is worth pursuing, you don't need structured fields from the body — you need the *text*: scope, eligibility, evaluation criteria, submission instructions. That's a job for **distill** (clean Markdown, **1 credit/URL**), not extract.

```bash
# Create a batch distill job over the same notice URLs
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://www.example-procurement.gov/notice/RFP-2026-0142",
      "https://www.example-procurement.gov/notice/RFP-2026-0151"
    ],
    "timeout": 60000
  }'
# → { "data": { "id": "batch_dist_abc", "status": "pending", "total": 2 } }

# Poll (paginated, page 0-based)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/distill/batch_dist_abc?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

Each completed result carries `data.markdown` (the clean solicitation text). Feed that to your LLM with a tight prompt to get a one-screen summary:

```python
SUMMARY_PROMPT = """You are a bid-desk analyst. From this public solicitation text,
produce a concise brief with exactly these sections:
- Scope (2-3 sentences)
- Eligibility / set-aside requirements
- Key dates (questions due, response deadline)
- Submission method
- Go/No-Go signal (one line: is this a fit for an IT-services vendor?)
Solicitation text:
---
{markdown}
"""
```

This is also the natural feed for a **retrieval-augmented** assistant over all your tracked solicitations — chunk the distilled Markdown, embed it, and let the bid team ask questions across every open opportunity. The full recipe is in the [RAG tutorial](07-rag-knowledge-base.md).

> **Why distill here, not extract?** Extract is for fields you'll *query or alert on* (deadline, value). Distill is for *reading and summarizing*. Distilling a notice is 20× cheaper than extracting it — see "Cost & scale."

---

## Alerting on new opportunities

The whole point is to catch *new* postings fast. The mechanism is a diff: compare today's extracted list against the set you stored last run, keyed by a stable identity. Use `reference_id` when present (it's the portal's own canonical key) and fall back to `detail_url`.

```python
def opportunity_key(row):
    """Stable identity: prefer the portal's reference id, fall back to the URL."""
    return (row.get("reference_id") or "").strip() or row.get("detail_url")

def diff_new(prev_keys, rows):
    """Return only rows whose key was not in the previous run."""
    new = []
    for r in rows:
        key = opportunity_key(r)
        if key and key not in prev_keys:
            new.append(r)
    return new
```

For each new opportunity, compute a **deadline countdown** so the alert is immediately actionable:

```python
import datetime

def days_until(deadline_str):
    """Days from today to an ISO deadline (YYYY-MM-DD). None if unparseable."""
    try:
        d = datetime.date.fromisoformat(deadline_str[:10])
        return (d - datetime.date.today()).days
    except (TypeError, ValueError):
        return None

def format_alert(row):
    days = days_until(row.get("deadline"))
    when = f"{days} days left" if days is not None else "deadline unknown"
    if days is not None and days < 0:
        when = "CLOSED"
    elif days is not None and days <= 7:
        when = f"⚠ {days} days left"
    val = row.get("value")
    money = f"{val:,.0f} {row.get('currency') or ''}".strip() if val else "value n/a"
    return (f"NEW: {row.get('title')} [{row.get('reference_id') or '—'}]\n"
            f"     {row.get('agency')} · {money} · deadline {row.get('deadline')} ({when})\n"
            f"     {row.get('detail_url')}")
```

A posting with a deadline only a few days out and a value in your sweet spot is the alert that should page someone, not just land in a digest.

---

## End-to-end pipeline

Here's one cohesive, runnable Python script that ties it all together: extract the keyword search page, batch-extract notice details (in chunks of 100), batch-distill the same notices for summaries, write a SQLite snapshot, then diff against the previous run to surface **new opportunities** and print them with **deadline countdowns**. Error handling follows the reference: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Daily public-procurement monitor built on the Thunderbit Open API."""
import os, time, json, random, sqlite3, datetime
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
# A public procurement portal you're authorized to access (SAM.gov / Grants.gov / TED / national portal).
# Prefer the portal's official open-data API/bulk download where one exists — see Prerequisites.
SEARCH_URL = "https://www.example-procurement.gov/search?q=cloud+services"
COUNTRY = "US"   # set per portal region, e.g. "DE", "GB"
DB = "opportunities.sqlite"

LIST_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "title":        {"type": "TEXT",   "instruction": "The notice / solicitation title as shown on the row"},
        "agency":       {"type": "TEXT",   "instruction": "The issuing department or contracting agency name"},
        "reference_id": {"type": "TEXT",   "instruction": "Solicitation / notice reference number, exactly as printed"},
        "category":     {"type": "TEXT",   "instruction": "Procurement category, NAICS/CPV label, or set-aside type"},
        "value":        {"type": "NUMBER", "instruction": "Estimated value, digits only, no symbol or commas; null if absent"},
        "currency":     {"type": "TEXT",   "instruction": "ISO currency code, e.g. USD or EUR; infer from symbol if needed"},
        "deadline":     {"type": "DATE",   "instruction": "Response/closing deadline; normalize to YYYY-MM-DD"},
        "posted_date":  {"type": "DATE",   "instruction": "Date the notice was posted; normalize to YYYY-MM-DD"},
        "detail_url":   {"type": "URL",    "instruction": "Absolute link to this notice's detail page"},
    }},
}

DETAIL_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "title":         {"type": "TEXT",   "instruction": "Full official notice title"},
        "agency":        {"type": "TEXT",   "instruction": "Issuing department / contracting office"},
        "reference_id":  {"type": "TEXT",   "instruction": "Solicitation / notice reference number, exactly as printed"},
        "category":      {"type": "TEXT",   "instruction": "Procurement category, NAICS/CPV code or label, set-aside type"},
        "value":         {"type": "NUMBER", "instruction": "Estimated or ceiling value, digits only, no symbol; null if absent"},
        "currency":      {"type": "TEXT",   "instruction": "ISO currency code, e.g. USD, EUR, GBP"},
        "deadline":      {"type": "DATE",   "instruction": "Response/closing deadline; normalize to YYYY-MM-DD"},
        "posted_date":   {"type": "DATE",   "instruction": "Date the notice was posted; normalize to YYYY-MM-DD"},
        "place_of_perf": {"type": "TEXT",   "instruction": "Place of performance / delivery location if stated"},
        "contact_email": {"type": "EMAIL",  "instruction": "Contracting officer email ONLY if publicly published in the notice"},
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
            time.sleep(min(60, 2 ** i) + random.uniform(0, 1))  # exponential backoff + jitter
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"Gave up on {path} after {attempts} attempts")

def _get(path, params=None):
    r = requests.get(f"{BASE}{path}", headers=HEADERS, params=params or {}, timeout=60)
    r.raise_for_status()
    return r.json()

def extract_list():
    res = _post("/openapi/v1/extract", {
        "url": SEARCH_URL, "schema": LIST_SCHEMA,
        "renderMode": "full", "waitFor": 2500, "timeout": 90000, "country_code": COUNTRY,
    })
    return res["data"]

def _poll_batch(kind, job_id):
    """kind = 'extract' | 'distill'. Returns the completed job's results list."""
    while True:
        status = _get(f"/openapi/v1/batch/{kind}/{job_id}", {"page": 0, "pageSize": 100})["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            return status.get("results", [])
        time.sleep(3)  # polling is free

def batch_detail(urls):
    """Structured detail fields, keyed by URL. Chunks of 100 (the batch ceiling)."""
    out = {}
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/extract",
                    {"urls": chunk, "schema": DETAIL_SCHEMA, "timeout": 90000})["data"]
        for res in _poll_batch("extract", job["id"]):
            if res.get("status") != "completed":
                print(f"  ! detail extract failed: {res['url']} ({res.get('error')})")
                continue
            payload = res.get("data") or []
            record = payload[0] if isinstance(payload, list) and payload else (payload or {})
            record["detail_url"] = res["url"]
            out[res["url"]] = record
    return out

def batch_solicitation_text(urls):
    """Clean Markdown of the solicitation body, keyed by URL (1 credit/URL via distill)."""
    out = {}
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/distill", {"urls": chunk, "timeout": 60000})["data"]
        for res in _poll_batch("distill", job["id"]):
            if res.get("status") != "completed":
                print(f"  ! distill failed: {res['url']} ({res.get('error')})")
                continue
            md = (res.get("data") or {}).get("markdown", "")
            out[res["url"]] = md
    return out

def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS snapshots (
        run_date TEXT, key TEXT, reference_id TEXT, detail_url TEXT, title TEXT,
        agency TEXT, category TEXT, value REAL, currency TEXT, deadline TEXT,
        posted_date TEXT, place_of_perf TEXT, contact_email TEXT, solicitation_md TEXT,
        PRIMARY KEY (run_date, key))""")
    con.commit()
    return con

def previous_keys(con):
    cur = con.execute("SELECT key FROM snapshots")  # every key ever seen
    return {row[0] for row in cur.fetchall()}

def key_of(row):
    return (row.get("reference_id") or "").strip() or row.get("detail_url")

def save_snapshot(con, run_date, rows, texts):
    for r in rows:
        con.execute("INSERT OR REPLACE INTO snapshots VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?)", (
            run_date, key_of(r), r.get("reference_id"), r.get("detail_url"), r.get("title"),
            r.get("agency"), r.get("category"), r.get("value"), r.get("currency"),
            r.get("deadline"), r.get("posted_date"), r.get("place_of_perf"),
            r.get("contact_email"), texts.get(r.get("detail_url"), "")))
    con.commit()

def days_until(deadline_str):
    try:
        d = datetime.date.fromisoformat((deadline_str or "")[:10])
        return (d - datetime.date.today()).days
    except (TypeError, ValueError):
        return None

def alert_line(r):
    days = days_until(r.get("deadline"))
    if days is None:
        when = "deadline unknown"
    elif days < 0:
        when = "CLOSED"
    elif days <= 7:
        when = f"⚠ {days} days left"
    else:
        when = f"{days} days left"
    val = r.get("value")
    money = f"{val:,.0f} {r.get('currency') or ''}".strip() if val else "value n/a"
    return (f"NEW: {r.get('title')} [{r.get('reference_id') or '—'}]\n"
            f"     {r.get('agency')} · {money} · deadline {r.get('deadline')} ({when})\n"
            f"     {r.get('detail_url')}")

def main():
    run_date = datetime.date.today().isoformat()
    con = init_db()
    prev = previous_keys(con)

    listings = extract_list()
    urls = [l["detail_url"] for l in listings if l.get("detail_url")]
    print(f"Found {len(urls)} opportunities on the search page")

    details = batch_detail(urls)                    # 20 credits/URL — fields to alert on
    texts = batch_solicitation_text(urls)           # 1 credit/URL  — text to summarize/RAG

    # Merge: prefer detail-page fields, fall back to the list row.
    merged = []
    for l in listings:
        u = l.get("detail_url")
        row = {**l, **details.get(u, {})}
        merged.append(row)

    new = [r for r in merged if key_of(r) and key_of(r) not in prev]
    save_snapshot(con, run_date, merged, texts)

    print(f"\n{len(new)} NEW opportunities since the last run\n")
    for r in sorted(new, key=lambda x: days_until(x.get("deadline")) or 1e9):
        print(alert_line(r))
        # texts[r["detail_url"]] holds the distilled solicitation — feed to your LLM/RAG here.

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire this to your alerting (PagerDuty/Slack/email)
        raise
```

Run it, and on day two it prints exactly which opportunities are new since the previous snapshot, sorted by how soon they close — with the distilled solicitation text already on hand to summarize or index.

---

## No-code version: the Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper** Chrome extension. Non-developer teammates on the bid desk can run the identical workflow without writing code, and the schema they build transfers verbatim to the API.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **AI Suggest Fields.** Open the public tender/grant search page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns (title, agency, reference id, category, value, deadline, posted date, detail link).
3. **Edit columns** in plain language. Rename, drop, or add columns; tune each column's instruction exactly like the `instruction` strings in the API schema — be explicit about date and currency formats.
4. **Scrape**, then **Scrape Subpages** to follow each row's `detail_url` and pull the full notice fields — the no-code equivalent of the two-stage list → detail batch pipeline in Steps 2–3.
5. **Export** to Excel, Google Sheets, Airtable, or Notion, or **schedule** recurring runs so the bid team sees new postings in a shared sheet each morning.

The recommended flow: design and validate the schema interactively in the extension, then port the field names and instructions to the CLI/API for automation at scale. They map one-to-one.

---

## Automate it

- **Cron — daily or twice-daily.** Procurement windows are short; running twice a day (morning + afternoon) shrinks the gap between a posting going live and your team seeing it. The diff logic already compares against every key stored in SQLite:

  ```cron
  # 07:00 and 14:00 every weekday: run the monitor and append a log
  0 7,14 * * 1-5  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/gov/monitor.py >> /var/log/gov.log 2>&1
  ```

- **Batch webhooks instead of polling.** For server-side jobs, supply a callback so Thunderbit notifies you when the batch finishes — no open connection needed:

  ```json
  {
    "urls": ["https://www.example-procurement.gov/notice/RFP-2026-0142", "..."],
    "schema": { "type": "array", "items": { "type": "object", "properties": { } } },
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "whatever-signing-secret-you-choose"
    }
  }
  ```

  Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call, then write the results to your store.

- **Store snapshots.** Keep one row per `(run_date, key)` (as the script does). That history is what makes "new since last run" reliable, survives restarts, and lets you reconstruct when each opportunity first appeared and how its deadline/value changed.
- **Alert with deadline countdowns.** Pipe the `new` list to Slack/email. Lead each alert with the days-to-deadline and value so triage is instant; escalate (page someone) when a high-value fit closes within a week.

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit**, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**Worked example.** A run that tracks **200 notices**:

```
1 search-page extract                  =     20 credits
200 detail extracts × 20 credits       =  4,000 credits   (fields you alert on)
200 solicitation distills × 1 credit   =    200 credits   (text you summarize/RAG)
+ 1 suggest_fields (one-time)          =      1 credit
≈ 4,221 credits / run
```

**Extract vs distill — a 20× decision.** Extract (structured JSON) costs 20× distill (clean Markdown). The list page and the few fields you alert on (deadline, value, reference id) justify extract; the long solicitation *body* does not — distill it for 1 credit/URL and let an LLM read it. Distilling all 200 notices for **200 credits** instead of extracting them for 4,000 is the single biggest lever on cost.

**Throttle & dedupe to cut both cost and load:**

- **Dedupe by `reference_id`.** Once you've extracted and distilled a notice, don't re-process it on the next run — your SQLite history already has it keyed. Only extract/distill notices whose `reference_id` is new (or whose deadline/value changed).
- **Stage cheaply.** The search-page extract is one call that yields many `detail_url`s; only batch the details you don't already have.
- **Reasonable concurrency.** Batch caps at 100 URLs/job; chunk larger sets and don't fire many jobs at one origin simultaneously. Add jitter to scheduled runs.

---

## Responsible use

Government procurement data is largely public and open — which is exactly why it's a good Thunderbit target — but the guardrails that apply to every workflow in this repo still apply here.

- **Prefer official open data.** SAM.gov, Grants.gov, and TED all publish official APIs and/or bulk downloads. When one exists, use it — it's the intended channel, it's more stable than scraping HTML, and it spares the portal load. Reserve Thunderbit for portals with no usable API, for normalizing many heterogeneous portals into one schema, and for distilling long solicitation HTML/PDFs into LLM-ready text.
- **Check `robots.txt` and Terms of Service first.** "Public" is not the same as "anything goes." Confirm the portal permits automated access to the pages you target, honor disallowed paths and crawl-rate directives, and back off if an origin blocks you rather than trying to evade.
- **Public, non-personal data by default.** Tender and grant facts (title, agency, value, dates, scope) are public and non-personal. The rare exception is a **named contact** — a contracting officer's name, email, or phone published as the point of contact. Government notices publish these as the official channel, which is generally a lawful basis, but collect them **only when publicly published in the notice**, use them only for the procurement they relate to, and drop the field if you don't need it (GDPR/CCPA). Don't build a contact-harvesting list.
- **Throttle.** Use batch with reasonable concurrency; don't hammer a single origin. Add jitter to scheduled runs.
- **Cache and dedupe** to avoid re-processing unchanged notices — it saves credits and reduces load on the source.

---

## Troubleshooting & FAQ

**Extraction returns an empty array.** The portal is almost certainly client-rendered. Set `renderMode:"full"` (headless browser) and raise `waitFor` (up to its `10000` ms ceiling) so lazy results load. Also raise `timeout` (extract allows up to `120000` ms). If you're on `renderMode:"none"` against an SPA, that's the cause. (Many government pages *are* still server-rendered — try `none` first; it's cheaper to render and often works.)

**Dates and values come back malformed** (e.g. `"18/06/2026"`, `"€4,8 Mio."`). This is the most common government-data snag because formats vary by portal and locale. Fix it in the `instruction`: for dates, `"normalize to YYYY-MM-DD"`; for money, `"digits only, no currency symbol, thousands separators, or abbreviations; convert millions to the full number"`. Keep `currency` a separate `TEXT` field so the `value` NUMBER stays clean. For NUMBER and DATE fields, the instruction is what coerces the value — brief it like you'd brief a human assistant.

**Pagination / "next page" on the search results.** Most portals paginate via a query parameter (`?page=2`, `?start=20`). Enumerate the page URLs and feed them as a list to **batch extract** with the list schema, then merge. For infinite-scroll results, raising `waitFor` loads more rows, but URL-based pagination is more reliable than scrolling. Always dedupe the merged rows by `reference_id`.

**Duplicate opportunities across runs or pages.** Dedupe by `reference_id` (the portal's own canonical key); fall back to `detail_url` only when no reference id is shown. Cross-portal duplicates (the same EU tender mirrored on a national portal) are best deduped by `reference_id` plus a normalized title.

**Big solicitation documents / timeouts.** Long notice pages and embedded PDFs render slowly. Raise `timeout` toward the `120000` ms max for extract (`60000` for distill) and add `waitFor`. For a single very large/slow notice outside a batch, use the async endpoints (`POST /openapi/v1/async/extract` or `async/distill`, then poll `GET /openapi/v1/async/.../{jobId}` every ~3s) so you don't hold an HTTP connection open. For reading-only, prefer `distill` — it's faster and 20× cheaper than extract.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status`; re-submit only the `failed` URLs in a fresh batch job. Don't re-run the whole batch — you'd pay credits again for the rows that already succeeded.

**Regional / non-US portals serve different content or block US traffic.** Set `country_code` (HTTP/CLI) or `countryCode` (MCP) to route the request from the right region — e.g. `DE` for a German portal, `GB` for a UK one, `FR` for France. The default is `US`. This matters when a portal serves localized content, currency, or restricts traffic by geography.

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.
