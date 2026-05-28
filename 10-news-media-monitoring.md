# News & Media Monitoring with Thunderbit

Track every public story about your brand, a competitor, or a topic — aggregate the headlines, distill the full articles, tag sentiment and entities with your own LLM, and watch the coverage trend for spikes — all from public news sources, using one AI web-data engine reachable from MCP, HTTP, or CLI.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## The data problem in media monitoring

PR and comms teams live and die by one question: *what is the world saying about us right now, and is the tone moving?* Answering it from raw web data is harder than it looks:

- **Coverage is scattered across thousands of outlets.** A single product launch surfaces in national papers, trade blogs, regional sites, newsletters, and press-release wires — each with its own HTML, its own date format, and its own snippet layout. There is no single clean feed of "everything written about us this week."
- **Paid monitoring tools are expensive and opaque.** Enterprise media-intelligence suites cost five or six figures a year, lock your data behind their UI, and still miss the long tail of smaller outlets. You often can't see *why* a story was scored positive or negative, and you can't plug in your own model.
- **You need fresh, full-text, and structured at the same time.** A headline plus a snippet tells you a story exists; it does not tell you the *narrative*. Real sentiment, theme, and entity analysis needs the **full article body** — and it needs to be clean (no nav, no ad rails, no cookie banners) before you hand it to an LLM. Stale weekly digests miss the spike that matters today.

The fix is a small, repeatable pipeline: discover a schema once on a news search-results page, **extract** the article list (headline, source, date, URL, snippet), **distill** each article URL into clean Markdown, hand that Markdown to *your* LLM for sentiment/theme/entity tagging, and append the result to a time series you can chart and alert on. Thunderbit supplies the extraction and distillation engine; this tutorial supplies the pipeline. Crucially, **`distill` is the workhorse here** — it is 20× cheaper than `extract` and gives the LLM exactly what it needs.

---

## What you'll build

- An **article aggregator** — a structured list of coverage for any query: headline (TEXT), source (TEXT), published date (DATE), article URL (URL), and snippet (TEXT).
- A **sentiment & entity tagger** — for each article, distill the full body to Markdown, then run a pluggable `analyze(markdown)` LLM step that returns sentiment (`+` / `0` / `-`), themes, named entities, and an event type (launch, lawsuit, outage, funding, etc.).
- A **coverage-trend monitor** — per-article `{date, sentiment, source}` rolled up into a daily volume + average-sentiment time series, stored in SQLite, with a **spike alert** that fires when today's volume jumps above a rolling baseline.
- The same workflow, no-code, via the Thunderbit Chrome extension for non-developer comms teammates.

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

4. **An LLM for the analysis step.** This is **pluggable** — Thunderbit gives you the clean text; *you* choose the model. The `analyze(markdown)` function in this tutorial is a stub you point at any provider (OpenAI, Anthropic, a local Llama, anything that takes text and returns JSON). Thunderbit does not score sentiment for you; it gives the model the clean input it needs to do so well.

> Want **semantic search** over the article corpus you build here ("show me every story that framed us as a privacy risk")? The distilled Markdown is already the perfect input for an embedding index. See the [RAG tutorial](07-rag-knowledge-base.md) for the embedding-and-storage patterns; this tutorial focuses on sentiment/volume time series, but the two compose cleanly — distill once, use the Markdown for both.

For full endpoint, schema, pricing, and error details, keep the [API reference](docs/thunderbit-api-reference.md) open in a tab.

---

## Three ways to call Thunderbit

All three surfaces hit the identical HTTP API underneath, so schemas and field instructions transfer verbatim between them.

| Surface | Entry point | When to use it |
|---------|-------------|----------------|
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and ad-hoc "what's being said today" pulls. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. Best for the automated hourly/daily monitoring sweep. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, saving/reusing schemas. Best for prototyping and glue scripts. |

A good real-world split: **explore** a news query in MCP, **freeze the article-list schema** with the CLI, **run the sweep hourly** over the HTTP API.

---

## Step 1 — discover the schema

Don't guess field names. Let the AI inspect the page and propose a schema. Throughout this tutorial, the canonical example is a **Google News query URL** — a public search-results page. For a brand like "Thunderbit" you'd point at:

```
https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en
```

The same pattern works on any public news source: a publisher's own search/section page, or the public index of a press-release wire such as **PR Newswire** or **Business Wire**. Read the "Responsible use" section first, and check each site's `robots.txt` and Terms of Service before you point a scraper at it.

`suggest_fields` costs **1 credit** and returns an array of field descriptors you can drop straight into an extract schema.

### (a) MCP — let the assistant call the tool

Prompt your assistant:

> Use Thunderbit to suggest fields for `https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en`. Focus on each result: headline, source/publisher name, published date, the link to the article, and the snippet/summary text.

The model issues a `thunderbit_suggest_fields` call with arguments like:

```json
{
  "url": "https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en",
  "prompt": "Extract each news result: headline, source/publisher, published date, article link, snippet",
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
    "url": "https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en",
    "prompt": "Extract each news result: headline, source/publisher, published date, article link, snippet",
    "country_code": "US"
  }'
```

### (c) CLI

```bash
thunderbit suggest-fields "https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en" \
  --prompt "Extract each news result: headline, source/publisher, published date, article link, snippet" \
  --country-code US
```

### The suggested-fields response

The fields come back at `data` (one array of descriptors):

```json
{
  "data": [
    { "name": "headline",  "type": "TEXT", "instruction": "The article headline/title text" },
    { "name": "source",    "type": "TEXT", "instruction": "Name of the publisher or outlet" },
    { "name": "published", "type": "DATE", "instruction": "Date the article was published" },
    { "name": "url",       "type": "URL",  "instruction": "Absolute link to the full article" },
    { "name": "snippet",   "type": "TEXT", "instruction": "Short summary/teaser text shown under the headline" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;` (JS) or `fields = res["data"].get("fields", res["data"])` (Python).

Treat the suggestion as a starting point. You'll usually tighten the `instruction` strings — the single biggest lever on extraction quality, especially the `published` DATE field (see the date-parsing tip in Step 2).

---

## Step 2 — extract the article list

Now turn the news search-results page into structured rows. The schema is **not** raw JSON Schema; it's the Thunderbit-flavored shape: an array of objects, where each property has an UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction`. Extract costs **20 credits** per page — but you only run it **once per query page**, on the *list*. The article *bodies* are distilled (1 credit each) in Step 3.

For an article-list page we want enough to identify each story plus the `url` we'll distill in Step 3:

### (a) HTTP — `curl`

Google News and most modern publisher result pages render client-side, so set `renderMode:"full"` (headless browser) and give lazy content a moment with `waitFor`. Use `country_code` to pull the regional edition of the news.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en",
    "renderMode": "full",
    "waitFor": 2500,
    "timeout": 90000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "headline":  { "type": "TEXT", "instruction": "The article headline/title text" },
          "source":    { "type": "TEXT", "instruction": "Name of the publisher or outlet (e.g. TechCrunch)" },
          "published": { "type": "DATE", "instruction": "Publication date. If shown as relative time like \"2 hours ago\" or \"yesterday\", resolve it to an absolute YYYY-MM-DD date" },
          "url":       { "type": "URL",  "instruction": "Absolute link to the full article page" },
          "snippet":   { "type": "TEXT", "instruction": "Short summary/teaser text shown under the headline" }
        }
      }
    }
  }'
```

Rows come back at `data`:

```json
{
  "data": [
    { "headline": "Thunderbit launches AI web-data API for developers", "source": "TechCrunch", "published": "2026-05-27", "url": "https://techcrunch.com/2026/05/27/thunderbit-api", "snippet": "The company opened its scraping engine to programmatic access..." },
    { "headline": "How PR teams use AI to monitor brand sentiment",      "source": "PR Daily",   "published": "2026-05-26", "url": "https://prdaily.com/ai-brand-sentiment", "snippet": "A new wave of tools lets comms teams track narrative in near real time..." }
  ]
}
```

> **Date parsing tip.** News listings love relative dates ("3 hours ago", "yesterday"). The `instruction` on the `published` DATE field is what coerces those into an absolute date — brief it explicitly, as above. If a source still returns mixed formats, normalize them in your own code (the pipeline below falls back to today's date when parsing fails).

### (b) Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

ARTICLE_LIST_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "headline":  {"type": "TEXT", "instruction": "The article headline/title text"},
            "source":    {"type": "TEXT", "instruction": "Name of the publisher or outlet"},
            "published": {"type": "DATE", "instruction": "Publication date; resolve relative times like '2 hours ago' to an absolute YYYY-MM-DD date"},
            "url":       {"type": "URL",  "instruction": "Absolute link to the full article page"},
            "snippet":   {"type": "TEXT", "instruction": "Short summary/teaser text shown under the headline"},
        },
    },
}

def extract(url, schema, render_mode="full", timeout=90000, wait_for=2500, country_code="US"):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema,
        "renderMode": render_mode, "timeout": timeout, "waitFor": wait_for,
        "country_code": country_code,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

QUERY_URL = "https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en"
articles = extract(QUERY_URL, ARTICLE_LIST_SCHEMA)
print(len(articles), "articles")
```

### (c) CLI — save the schema, then reuse it

Save the schema to a file once, then re-run it any time:

```bash
# Save the article-list schema to articles.schema.json (an array-of-objects schema as above),
# then extract with it:
thunderbit extract "https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en" \
  --schema articles.schema.json \
  --render-mode full \
  --timeout 90000 \
  --country-code US \
  -f json > articles.json
```

> **`renderMode` cheat sheet:** `none` = raw HTML fetch (fastest/cheapest render; fine for server-rendered pages and many press-release wires); `basic` = light JS; `full` = headless browser for JS-heavy SPAs like Google News. If a page comes back empty, the first thing to try is `renderMode:"full"` plus a larger `waitFor`. The valid `waitFor` range is `0–10000` ms; `extract` `timeout` is `5000–120000` ms (default `60000`).
>
> **Regional news.** Set `country_code` (HTTP/CLI) or `countryCode` (MCP) to pull the edition that a given market sees — `US`, `GB`, `DE`, `FR`, etc. Pair it with the matching Google News `gl`/`ceid` query params for consistent results.

---

## Step 3 — distill full articles for analysis

This is the heart of media monitoring. A headline and snippet tell you a story exists; the **narrative** lives in the body. Take the `url` values from Step 2 and run them through **batch distill**, which handles **up to 100 URLs per job** at **1 credit per URL** — and returns clean, LLM-ready Markdown with the nav, ads, and cookie banners stripped out.

**Why distill, not extract, for the body?** You don't need fixed structured fields out of an article — you need the *prose*, so your own model can reason over it. Distill is **1 credit/URL**; extract is **20 credits/URL**. For a sweep of 100 articles that's **100 credits vs 2,000** — a 20× difference for exactly the input the LLM wants. Reserve extract for the *list* (Step 2); use distill for the *bodies*.

### (a) HTTP — create the batch-distill job, then poll

```bash
# 1. Create the batch-distill job (up to 100 URLs)
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://techcrunch.com/2026/05/27/thunderbit-api",
      "https://prdaily.com/ai-brand-sentiment"
    ],
    "timeout": 60000
  }'
# → { "data": { "id": "batch_news123", "status": "pending", "total": 2 } }

# 2. Poll for results (page is 0-based; pageSize default 20, max 100)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/distill/batch_news123?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

The batch status response carries job-level status plus a per-URL `results` array, so you can retry only the failures. Each distilled body lives at `results[].data.markdown`:

```json
{
  "data": {
    "id": "batch_news123",
    "status": "completed",
    "total": 2,
    "completed": 2,
    "results": [
      { "url": "https://techcrunch.com/2026/05/27/thunderbit-api", "status": "completed", "data": { "markdown": "# Thunderbit launches AI web-data API\n\nThe company today opened..." } },
      { "url": "https://prdaily.com/ai-brand-sentiment",          "status": "completed", "data": { "markdown": "# How PR teams use AI\n\nA new wave of tools..." } }
    ]
  }
}
```

Job-level `status` is `pending | processing | completed | failed | cancelled`. Per-URL `results[].status` lets you re-submit only those URLs that came back `failed` — and **skip** the ones that come back empty (a common signal of a paywall; see Troubleshooting).

### (b) CLI — batch from a file

`--file` reads one URL per line and ignores blank lines and lines starting with `#`.

```bash
# article_urls.txt has one article URL per line
thunderbit batch distill --file article_urls.txt \
  --timeout 60000 \
  -f json > bodies.json
```

### (c) MCP

Ask your assistant: *"Use Thunderbit to batch-distill these article URLs, then poll until done and give me the Markdown for each."* It calls `thunderbit_batch_distill_create` (args `urls`, `timeout?`) and then `thunderbit_batch_distill_status` (args `jobId`, `page?`, `pageSize?`) until the job is `completed`.

### The pluggable LLM analysis step

Thunderbit hands you clean Markdown. Now *your* model turns it into sentiment, themes, entities, and an event type. Keep this swappable — the rest of the pipeline only cares that `analyze(markdown)` returns a dict:

```python
import json

# ----------------------------------------------------------------------------
# PLUGGABLE LLM STEP — swap in any provider (OpenAI, Anthropic, local Llama...).
# Thunderbit gives you the clean article text; your model does the judgement.
# The contract: take Markdown, return a dict with these keys.
# ----------------------------------------------------------------------------
ANALYSIS_PROMPT = """You are a media-analysis assistant. Read the news article below and return STRICT JSON:
{
  "sentiment": "+" | "0" | "-",          // tone toward the subject brand/topic
  "themes": ["short", "theme", "tags"],   // 1-4 topical themes
  "entities": ["Org or Person names"],    // named entities mentioned
  "event_type": "launch|funding|lawsuit|outage|partnership|hire|award|opinion|other"
}
Return ONLY the JSON object, no prose.

ARTICLE:
"""

def analyze(markdown: str) -> dict:
    """Return {sentiment, themes, entities, event_type} for one article.

    Replace the body with a real LLM call. Example shapes:

        # OpenAI
        from openai import OpenAI
        client = OpenAI()
        resp = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": ANALYSIS_PROMPT + markdown[:12000]}],
            response_format={"type": "json_object"},
        )
        return json.loads(resp.choices[0].message.content)

        # Anthropic
        import anthropic
        msg = anthropic.Anthropic().messages.create(
            model="claude-3-5-haiku-latest", max_tokens=512,
            messages=[{"role": "user", "content": ANALYSIS_PROMPT + markdown[:12000]}],
        )
        return json.loads(msg.content[0].text)
    """
    # --- Offline stub so this file runs without an LLM configured ---
    text = markdown.lower()
    if any(w in text for w in ("lawsuit", "outage", "breach", "criticism", "decline")):
        sentiment = "-"
    elif any(w in text for w in ("launch", "raises", "growth", "award", "wins", "partnership")):
        sentiment = "+"
    else:
        sentiment = "0"
    return {"sentiment": sentiment, "themes": [], "entities": [], "event_type": "other"}
```

Truncating to ~12k characters (`markdown[:12000]`) keeps token cost predictable; lead and first few paragraphs carry most of the sentiment signal. Tune to your model's context window and budget.

---

## Tracking trends & spikes

One article is an anecdote; the **time series** is the signal. For every analyzed article, store a compact record — `{date, sentiment, source, url, headline}` — then roll it up daily:

- **Daily volume** = number of articles published per day for the query.
- **Average sentiment** = map `+ → +1`, `0 → 0`, `- → -1`, then average per day. A drift toward negative is your early-warning gauge.
- **Spike alert** = compare today's volume to a **rolling baseline** (e.g. the mean of the prior 7 days). When today exceeds `baseline + k × stdev` (or simply `> 2 × baseline`), something is happening — fire an alert.

```python
import sqlite3, statistics, datetime

def daily_rollup(con):
    """Return [(date, volume, avg_sentiment)] ordered by date."""
    cur = con.execute("""
        SELECT published_date,
               COUNT(*)                                   AS volume,
               AVG(CASE sentiment WHEN '+' THEN 1 WHEN '-' THEN -1 ELSE 0 END) AS avg_sent
        FROM coverage
        GROUP BY published_date
        ORDER BY published_date
    """)
    return cur.fetchall()

def detect_spike(series, window=7, factor=2.0):
    """series = [(date, volume, avg_sent)]. Flag the latest day if its volume
    exceeds factor× the mean volume of the preceding `window` days."""
    if len(series) < window + 1:
        return None
    *history, latest = series
    baseline = statistics.mean(v for _d, v, _s in history[-window:])
    date, volume, avg_sent = latest
    if baseline > 0 and volume > factor * baseline:
        return {"date": date, "volume": volume, "baseline": round(baseline, 1),
                "avg_sentiment": round(avg_sent, 2)}
    return None
```

A spike that arrives with **negative** average sentiment is the call-the-CEO scenario; a spike with positive sentiment is amplification fuel for the comms team. Both are worth an alert.

---

## End-to-end pipeline

Here's one cohesive, runnable Python script that ties it all together: extract the article list for a query, batch-**distill** the article bodies (in chunks of 100), run the pluggable `analyze()` LLM step on each, append the result to a SQLite time series (deduped by canonical `url`), then roll up the daily volume/sentiment and detect a spike. Error handling follows the reference: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Hourly/daily news & media monitoring pipeline built on the Thunderbit Open API.

Pipeline: news query -> extract article list -> batch-distill bodies
          -> pluggable LLM tag -> append to SQLite time series -> detect spike.
"""
import os, time, json, random, sqlite3, statistics, datetime
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
# A Google News query URL — a PUBLIC search-results page. Swap the q= term for your brand/topic.
QUERY_URL = "https://news.google.com/search?q=Thunderbit&hl=en-US&gl=US&ceid=US:en"
COUNTRY = "US"
DB = "coverage.sqlite"

ARTICLE_LIST_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "headline":  {"type": "TEXT", "instruction": "The article headline/title text"},
        "source":    {"type": "TEXT", "instruction": "Name of the publisher or outlet"},
        "published": {"type": "DATE", "instruction": "Publication date; resolve relative times like '2 hours ago' to an absolute YYYY-MM-DD date"},
        "url":       {"type": "URL",  "instruction": "Absolute link to the full article page"},
        "snippet":   {"type": "TEXT", "instruction": "Short summary/teaser text shown under the headline"},
    }},
}

class OutOfCredits(Exception):
    """402 — stop the run and alert a human."""

# --- HTTP helpers with backoff per the API reference §4 ----------------------
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

# --- Step 2: extract the article list ----------------------------------------
def extract_articles():
    res = _post("/openapi/v1/extract", {
        "url": QUERY_URL, "schema": ARTICLE_LIST_SCHEMA,
        "renderMode": "full", "waitFor": 2500, "timeout": 90000,
        "country_code": COUNTRY,
    })
    return res["data"]

# --- Step 3: batch-distill article bodies (1 credit/URL) ----------------------
def batch_distill(urls):
    """Distill article URLs in chunks of 100. Returns {url: markdown}."""
    out = {}
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/distill", {"urls": chunk, "timeout": 60000})["data"]
        job_id = job["id"]
        while True:
            status = _get(f"/openapi/v1/batch/distill/{job_id}",
                          {"page": 0, "pageSize": 100})["data"]
            if status["status"] in ("completed", "failed", "cancelled"):
                break
            time.sleep(3)  # polling is free
        for result in status.get("results", []):
            if result.get("status") != "completed":
                print(f"  ! distill failed: {result['url']} ({result.get('error')})")
                continue
            md = (result.get("data") or {}).get("markdown", "")
            if not md.strip():
                print(f"  ~ empty body (likely paywall) — skipping: {result['url']}")
                continue  # never try to bypass a paywall; just skip it
            out[result["url"]] = md
    return out

# --- Pluggable LLM analysis (see prose for real-provider variants) ------------
def analyze(markdown):
    text = markdown.lower()
    if any(w in text for w in ("lawsuit", "outage", "breach", "criticism", "decline")):
        sentiment = "-"
    elif any(w in text for w in ("launch", "raises", "growth", "award", "wins", "partnership")):
        sentiment = "+"
    else:
        sentiment = "0"
    return {"sentiment": sentiment, "themes": [], "entities": [], "event_type": "other"}

# --- Storage: one row per canonical url (dedupe) ------------------------------
def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS coverage (
        url TEXT PRIMARY KEY, headline TEXT, source TEXT, published_date TEXT,
        sentiment TEXT, themes TEXT, entities TEXT, event_type TEXT, seen_at TEXT)""")
    con.commit()
    return con

def normalize_url(url):
    """Strip query/fragment so the same article via different trackers dedupes to one row."""
    from urllib.parse import urlsplit, urlunsplit
    s = urlsplit(url)
    return urlunsplit((s.scheme, s.netloc, s.path.rstrip("/"), "", ""))

def upsert(con, row):
    con.execute("""INSERT OR IGNORE INTO coverage
        (url, headline, source, published_date, sentiment, themes, entities, event_type, seen_at)
        VALUES (?,?,?,?,?,?,?,?,?)""", (
        row["url"], row.get("headline"), row.get("source"), row.get("published_date"),
        row["sentiment"], json.dumps(row.get("themes", [])),
        json.dumps(row.get("entities", [])), row.get("event_type"),
        datetime.datetime.utcnow().isoformat()))

def already_seen(con, url):
    return con.execute("SELECT 1 FROM coverage WHERE url = ?", (url,)).fetchone() is not None

# --- Trend & spike ------------------------------------------------------------
def daily_rollup(con):
    cur = con.execute("""
        SELECT published_date,
               COUNT(*) AS volume,
               AVG(CASE sentiment WHEN '+' THEN 1 WHEN '-' THEN -1 ELSE 0 END) AS avg_sent
        FROM coverage WHERE published_date IS NOT NULL
        GROUP BY published_date ORDER BY published_date""")
    return cur.fetchall()

def detect_spike(series, window=7, factor=2.0):
    if len(series) < window + 1:
        return None
    *history, latest = series
    baseline = statistics.mean(v for _d, v, _s in history[-window:])
    date, volume, avg_sent = latest
    if baseline > 0 and volume > factor * baseline:
        return {"date": date, "volume": volume, "baseline": round(baseline, 1),
                "avg_sentiment": round(avg_sent or 0, 2)}
    return None

def main():
    con = init_db()
    today = datetime.date.today().isoformat()

    articles = extract_articles()
    print(f"Found {len(articles)} articles on the query page")

    # Dedupe by canonical url; only distill bodies we haven't analyzed yet (saves credits).
    fresh = {}
    for a in articles:
        if not a.get("url"):
            continue
        canon = normalize_url(a["url"])
        if already_seen(con, canon):
            continue
        fresh[canon] = a
    print(f"{len(fresh)} new articles to analyze")

    bodies = batch_distill(list(fresh.keys()))
    print(f"Distilled {len(bodies)} bodies")

    analyzed = 0
    for canon, md in bodies.items():
        meta = fresh[canon]
        tags = analyze(md)
        upsert(con, {
            "url": canon,
            "headline": meta.get("headline"),
            "source": meta.get("source"),
            "published_date": meta.get("published") or today,  # fallback if date parse failed
            **tags,
        })
        analyzed += 1
    con.commit()
    print(f"Tagged and stored {analyzed} articles")

    series = daily_rollup(con)
    if series:
        d, v, s = series[-1]
        print(f"Latest day {d}: {v} articles, avg sentiment {round(s or 0, 2)}")
    spike = detect_spike(series)
    if spike:
        tone = "NEGATIVE" if spike["avg_sentiment"] < 0 else "positive"
        print(f"\n🔔 COVERAGE SPIKE on {spike['date']}: {spike['volume']} articles "
              f"(baseline {spike['baseline']}), avg sentiment {spike['avg_sentiment']} [{tone}]")
        # wire this to Slack/email — see "Automate it"

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire this to your alerting (PagerDuty/Slack/email)
        raise
```

Run it hourly and the `coverage` table becomes a living record of your narrative: every new article tagged, the daily volume and sentiment trending, and a spike alert the moment coverage jumps above its baseline.

---

## No-code version: the Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper** Chrome extension. Non-developer comms teammates can run the identical aggregation workflow without writing code, and the schema they build transfers verbatim to the API.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **AI Suggest Fields.** Open the Google News query page (or a publisher's search page), click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns (headline, source, published, article link, snippet).
3. **Edit columns** in plain language. Rename, drop, or add columns; tune each column's instruction exactly like the `instruction` strings in the API schema — for example, telling the `published` column to resolve "2 hours ago" into a real date.
4. **Scrape**, then **Scrape Subpages** to follow each row's article link and pull the full body text — the no-code equivalent of the batch-distill step. You can then summarize or tag those bodies with your own tools.
5. **Export** to Excel, Google Sheets, Airtable, or Notion, or **schedule** recurring runs to feed a sentiment dashboard.

The recommended flow: design and validate the article-list schema interactively in the extension, then port the field names and instructions to the CLI/API for the automated sentiment pipeline. They map one-to-one.

---

## Automate it

- **Cron sweeps.** Run the pipeline hourly for fast-moving brands, or daily for slower topics. The dedupe-by-`url` logic means each run only distills articles it hasn't seen, so frequent runs stay cheap:

  ```cron
  # Every hour at :05, sweep the news query and update the time series
  5 * * * *  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/monitor/pipeline.py >> /var/log/monitor.log 2>&1
  ```

- **Batch webhooks instead of polling.** For server-side jobs, supply a callback so Thunderbit notifies you when the batch-distill job finishes — no open connection needed:

  ```json
  {
    "urls": ["https://techcrunch.com/2026/05/27/thunderbit-api", "..."],
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "whatever-signing-secret-you-choose"
    }
  }
  ```

  Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call, then run `analyze()` on each body and append to your store.

- **Time-series storage.** Keep one row per canonical `url` (as the script does) with its `published_date` and `sentiment`. That history powers the daily volume/sentiment chart, week-over-week comparisons, and the spike baseline.
- **Spike alerts to Slack/email.** Pipe the `detect_spike()` result to a Slack incoming webhook or an email. Include the day, volume, baseline, and average sentiment so the on-call comms person can triage in one glance — a negative-sentiment spike is a page-someone event.

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit**, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**Distill is the workhorse here.** You pay 20 credits *once* for the article-list extract, then 1 credit per article body. A typical hourly sweep that finds **30 new articles**:

```
1 article-list extract        =  20 credits
30 article bodies × 1 (distill) =  30 credits
≈ 50 credits / sweep
```

Run that hourly for a day and you're around **1,200 credits/day** — versus **~14,400** if you (wrongly) used `extract` on every body. **Always distill the bodies.** You'd only reach for `extract` on an article body if you needed fixed structured fields out of it (rare in monitoring).

**Throttle & cache to cut both cost and load:**

- **Dedupe by canonical `url`.** Strip tracking query params/fragments (the script's `normalize_url`) so the same story arriving via three different trackers counts once and is distilled once.
- **Skip what you've seen.** The pipeline only distills URLs not already in the `coverage` table — re-runs of the same sweep cost almost nothing.
- **Reasonable concurrency.** Batch caps at 100 URLs/job; chunk larger sets and don't fire many jobs at one origin simultaneously. Add jitter to scheduled runs.

---

## Responsible use

These guardrails apply to every workflow in this repo — media monitoring especially, because news content is copyrighted.

- **Public sources only.** Target publicly accessible news — Google News results, public news sites, and the public pages of press-release wires (PR Newswire, Business Wire). Never build workflows against login-walled or subscriber-only areas.
- **Respect publisher copyright.** Distill articles to **analyze them internally** — sentiment, themes, entities, trend lines. Do **not** redistribute or republish the full article text. Store your *derived* signals (sentiment, tags, your own summaries) and link back to the original; don't rehost the body.
- **Don't bypass paywalls.** If an article body comes back empty or truncated, that's usually a paywall. **Skip it** — never use tricks to get behind a paywall. The pipeline above explicitly drops empty bodies.
- **Check `robots.txt` and Terms of Service** for each source before you point a scraper at it. Honor disallowed paths and crawl-rate directives.
- **Throttle.** Use batch with reasonable concurrency and add jitter; don't hammer a single outlet's origin.
- **Personal-data caution.** Articles name people. You're analyzing public reporting, but keep your stored entities to what serves the monitoring purpose, and mind GDPR/CCPA if you retain anything about individuals.

---

## Troubleshooting & FAQ

**Distill returns an empty or near-empty body.** That's almost always a **paywall** or a consent-wall interstitial. Skip the URL — do **not** try to defeat the paywall. The pipeline drops empty `markdown` automatically. If a page is merely JS-heavy (not paywalled), the next two tips apply.

**The query page or an article comes back empty.** The page is client-rendered. For the *list* extract, set `renderMode:"full"` and raise `waitFor` (up to its `10000` ms ceiling) so lazy content loads; also raise `timeout` (extract allows up to `120000` ms). Google News in particular needs `renderMode:"full"`. Batch distill renders pages for you, but a single very slow page can still time out — raise the batch `timeout`.

**Dedupe: the same story shows up twice.** News aggregators append tracking params (`?utm_source=…`) and Google News wraps links in redirects, so the "same" article arrives under several URLs. Normalize to a **canonical url** (strip query + fragment, lowercase host, drop trailing slash — see `normalize_url`) and key your store on that. Where a publisher exposes a `<link rel="canonical">`, prefer it.

**Relative or missing dates.** Listings show "3 hours ago" or omit the year. Put the conversion in the `published` field's `instruction` ("resolve relative times to an absolute YYYY-MM-DD date"), and keep a code-side fallback (the pipeline defaults to today's date when parsing fails) so a bad date never drops an article from the time series.

**Regional / non-US coverage.** Set `country_code` (HTTP/CLI) or `countryCode` (MCP) to the market you care about — `GB`, `DE`, `FR`, `JP` — and match the Google News `gl`/`hl`/`ceid` query params. The default is `US`. This changes which edition and which outlets you see.

**Sentiment looks off.** Sentiment quality lives in *your* LLM and prompt, not in Thunderbit — Thunderbit only supplies the clean text. Tighten the `ANALYSIS_PROMPT`, give the model the full lead (not just the snippet), and consider few-shot examples for your domain. Truncate the body to a sensible length (`markdown[:12000]`) to control token cost.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status`; re-submit only the `failed` URLs in a fresh batch job. Don't re-run the whole batch — you'd pay credits again for the bodies that already succeeded.

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.

**I want semantic search over the corpus, not just trends.** The distilled Markdown is already an ideal embedding input. Run the same `batch/distill` step, then embed and store per the [RAG tutorial](07-rag-knowledge-base.md) — distill once, feed both the sentiment time series and the vector index.
