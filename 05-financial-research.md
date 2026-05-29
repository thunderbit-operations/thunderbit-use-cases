# Financial & Investment Research: Fundamentals, Filings RAG & News Sentiment from Public Sources with Thunderbit

Turn scattered public filings, investor-relations pages, market-data quotes, and news into a fresh, structured research dataset and a retrieval-ready filings corpus — entirely from public sources.

---

## The data problem in investment research

Investment research is, underneath the analysis, a data-plumbing problem. The raw material an analyst needs is public — SEC filings, earnings transcripts, investor-relations decks, quote pages, press releases, news — but it lives in a dozen incompatible shapes. Fundamentals sit in HTML tables on quote pages. The story behind those numbers sits in 80-page 10-K PDFs and hour-long earnings transcripts. Sentiment sits in a firehose of news articles. None of it shares a schema, and all of it goes stale the moment a new filing or quote prints.

So researchers read manually: open a 10-K, skim 60 pages for the one risk-factor paragraph that changed, copy a few numbers into a spreadsheet, repeat for every name on the watchlist. That doesn't scale past a handful of tickers, and it produces datasets that were already out of date the day they were assembled.

What you actually need are two things working together: **structured, refreshable fundamentals** (numbers you can diff and alert on) and a **clean, machine-readable text corpus** of the underlying disclosures (so an LLM can answer "what changed in the risk factors this quarter?" with citations). Thunderbit gives you both — `extract` for the tables, `distill` for the prose — and `distill` is roughly 20× cheaper, which makes a large filings corpus genuinely affordable.

This guide builds on **public sources only**: SEC EDGAR (public filings, explicitly built for programmatic access), public company investor-relations pages, public market-data/quote pages, and public news. It is not investment advice, and it deliberately avoids paywalled or login-walled data. Every endpoint, schema, and response shape below comes from the canonical [API reference](docs/thunderbit-api-reference.md) — if a sample here ever disagrees with that file, the reference wins.

## What you'll build

- **A fundamentals dataset** — extract a structured table per ticker (ticker, company, price, market cap, P/E, EPS, dividend yield, as-of date) from public quote pages.
- **A filings RAG corpus** — batch-distill public 10-Ks, 10-Qs, earnings transcripts, and IR pages into clean Markdown, chunked and ready to embed for retrieval-augmented Q&A.
- **A news & sentiment monitor** — distill public news/press pages and run an LLM over the clean text for per-ticker sentiment and event tagging.
- **A watchlist tracker** — one runnable Python pipeline that refreshes fundamentals for a list of tickers, distills their latest filings into a versioned corpus, snapshots the result, and diffs against the last run to fire alerts.

## Prerequisites

1. A Thunderbit API key (prefixed `tb_`). Get one at <https://app.thunderbit.com/console/api-keys>.
2. Set it in your environment — never hard-code it:

```bash
export THUNDERBIT_API_KEY=tb_your_api_key_here
```

3. For the CLI / SDK route:

```bash
npm i -g @thunderbit/thunderbit-cli
```

4. For the Python pipeline: `pip install requests`.

New accounts get a one-time free allotment (~600 units) — enough to prototype distill and extract before topping up. See the [API reference](docs/thunderbit-api-reference.md) for auth, pricing, and error semantics.

## Three ways to call Thunderbit

The same web-data engine is exposed through three surfaces that all hit the identical HTTP API, so you can mix them freely.

| Surface | Entry point | Best for in financial research |
|---------|-------------|--------------------------------|
| **MCP server** | `@thunderbit/mcp-server` (npx) | An AI assistant (Claude, Cursor, Cline) doing interactive research: "distill this 10-K and summarize the risk factors," or "extract the fundamentals table from this quote page." No code. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs (pre-market refresh), webhooks, the production pipeline. Any language. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` (npm) | Local scripting, one-off corpus builds from a `urls.txt` of filing links, saving/reusing a fundamentals schema. |

Rule of thumb: prototype the schema in MCP or the CLI, then automate the watchlist at scale with the HTTP API.

## Step 1 — Discover the schema on a public quote page

Don't hand-write field names. Point `suggest_fields` at a public quote/summary page and let the AI propose a schema you can tweak. We'll use a representative public market-data quote page as the example — substitute the actual public quote page you're entitled to use, and honor its terms on redistribution of market data:

`https://example-finance.com/quote/AAPL`

### Via MCP (AI assistant)

In Claude Desktop / Cursor / Cline with the Thunderbit MCP server configured, just ask:

> Use `thunderbit_suggest_fields` on `https://example-finance.com/quote/AAPL` with the prompt "Extract the key fundamentals for this stock: ticker, company name, price, market cap, P/E ratio, EPS, dividend yield, and the as-of date."

The model calls the tool and shows you the proposed fields. `suggest_fields` costs 1 credit.

### Via HTTP API

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/suggest_fields \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example-finance.com/quote/AAPL",
    "prompt": "Extract the key fundamentals for this stock: ticker, company name, price, market cap, P/E ratio, EPS, dividend yield, and the as-of date.",
    "country_code": "US"
  }'
```

### Via CLI

```bash
thunderbit suggest-fields https://example-finance.com/quote/AAPL \
  --prompt "Key fundamentals: ticker, company, price, market cap, P/E, EPS, dividend yield, as-of date" \
  --country-code US
```

A realistic response (the array lives at `data`; some deployments wrap it as `data.fields`, so handle both):

```json
{
  "data": [
    { "name": "ticker",         "type": "TEXT",   "instruction": "The stock ticker symbol, e.g. AAPL" },
    { "name": "company",        "type": "TEXT",   "instruction": "The full company name" },
    { "name": "price",          "type": "NUMBER", "instruction": "Last traded price in USD, digits only" },
    { "name": "market_cap",     "type": "TEXT",   "instruction": "Market capitalization as shown, e.g. 2.95T" },
    { "name": "pe_ratio",       "type": "NUMBER", "instruction": "Price-to-earnings (TTM) ratio" },
    { "name": "eps",            "type": "NUMBER", "instruction": "Earnings per share (TTM) in USD" },
    { "name": "dividend_yield", "type": "NUMBER", "instruction": "Forward dividend yield as a percentage" },
    { "name": "as_of",          "type": "DATE",   "instruction": "The date/time the quote was last updated" }
  ]
}
```

Notice `market_cap` came back as `TEXT`, not `NUMBER`. That's deliberate — public quote pages render market cap as `2.95T` or `1,234.56B`, which is not a clean number. We keep it as text and normalize it ourselves (see Step 2 and the FAQ).

## Step 2 — Extract a fundamentals table

Now turn that schema into an `extract` call. The schema format is Thunderbit-flavored (see §5 of the [API reference](docs/thunderbit-api-reference.md)): an object whose `properties` use UPPERCASE types (`TEXT`, `NUMBER`, `URL`, `EMAIL`, `DATE`) plus an `instruction`. Because a single quote page describes **one** company, a flat object schema is the right shape; for a screener page that lists many rows, wrap it in `{ "type": "array", "items": { ... } }` instead.

Two things matter most here:

- **`renderMode`.** Many quote pages render their numbers through a JavaScript widget that isn't in the raw HTML. If `renderMode: "none"` returns empty fields, switch to `"full"` (headless browser) and add `waitFor` so the widget settles.
- **`instruction` strings for messy numbers.** Quote pages are full of `$1,234.56`, `2.3B`, `1.85%`, `N/A`. Write the instruction the way you'd brief a human: tell the model exactly what to strip and what units you want.

### Via HTTP API

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example-finance.com/quote/AAPL",
    "renderMode": "full",
    "waitFor": 2000,
    "timeout": 60000,
    "schema": {
      "type": "object",
      "properties": {
        "ticker":         { "type": "TEXT",   "instruction": "The stock ticker symbol, uppercase, e.g. AAPL" },
        "company":        { "type": "TEXT",   "instruction": "Full company name" },
        "price":          { "type": "NUMBER", "instruction": "Last price in USD. Strip the $ and any thousands separators; digits and decimal only, e.g. 1234.56" },
        "market_cap":     { "type": "TEXT",   "instruction": "Market cap exactly as displayed including the magnitude suffix, e.g. 2.95T or 812.4B" },
        "pe_ratio":       { "type": "NUMBER", "instruction": "P/E (TTM) as a decimal number. If shown as N/A, leave empty" },
        "eps":            { "type": "NUMBER", "instruction": "EPS (TTM) in USD, decimal. Negative if a loss" },
        "dividend_yield": { "type": "NUMBER", "instruction": "Forward dividend yield as a percentage value only, e.g. 0.55 for 0.55%. If no dividend, leave empty" },
        "as_of":          { "type": "DATE",   "instruction": "Date the quote was last updated, ISO format" }
      }
    }
  }'
```

Response (one record at `data`):

```json
{
  "data": {
    "ticker": "AAPL",
    "company": "Apple Inc.",
    "price": 1234.56,
    "market_cap": "2.95T",
    "pe_ratio": 31.2,
    "eps": 6.42,
    "dividend_yield": 0.55,
    "as_of": "2026-05-28"
  }
}
```

### Via Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

FUNDAMENTALS_SCHEMA = {
    "type": "object",
    "properties": {
        "ticker":         {"type": "TEXT",   "instruction": "Ticker symbol, uppercase"},
        "company":        {"type": "TEXT",   "instruction": "Full company name"},
        "price":          {"type": "NUMBER", "instruction": "Last price USD; strip $ and commas; decimal only"},
        "market_cap":     {"type": "TEXT",   "instruction": "Market cap as displayed, keep suffix e.g. 2.95T"},
        "pe_ratio":       {"type": "NUMBER", "instruction": "P/E (TTM) decimal; empty if N/A"},
        "eps":            {"type": "NUMBER", "instruction": "EPS (TTM) USD decimal; negative for loss"},
        "dividend_yield": {"type": "NUMBER", "instruction": "Forward yield as percent value, e.g. 0.55; empty if none"},
        "as_of":          {"type": "DATE",   "instruction": "Quote update date, ISO"},
    },
}

def fundamentals(url, render_mode="full"):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": FUNDAMENTALS_SCHEMA,
        "renderMode": render_mode, "waitFor": 2000, "timeout": 60000,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

print(fundamentals("https://example-finance.com/quote/AAPL"))
```

### Via CLI

Save the schema once, then reuse it across tickers:

```bash
# Interactively design + save the schema
thunderbit extract https://example-finance.com/quote/AAPL -i --save-schema fundamentals.schema.json

# Reuse it, render JS widgets, print a table
thunderbit extract https://example-finance.com/quote/MSFT \
  --schema fundamentals.schema.json --render-mode full -f table
```

`extract` costs 20 credits per page. That's fine for a watchlist of fundamentals, but it's the wrong tool for ingesting long filings — for that, use `distill`, which is where the real leverage is.

## Step 3 — Build a filings RAG corpus with distill

This is the highest-ROI use of Thunderbit in this vertical. A 10-K is mostly prose: business overview, risk factors, MD&A. You don't want to "extract" 80 pages into fields — you want clean Markdown you can chunk, embed, and retrieve against. `distill` does exactly that for **1 credit per page** — 20× cheaper than `extract`.

**SEC EDGAR is the canonical public source.** Filings on EDGAR are public, designed for programmatic access, and free to use. A real filing index URL looks like:

`https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK=0000320193&type=10-K`

and an individual filing document like:

`https://www.sec.gov/Archives/edgar/data/320193/000032019323000106/aapl-20230930.htm`

### Distill a single filing

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.sec.gov/Archives/edgar/data/320193/000032019323000106/aapl-20230930.htm",
    "renderMode": "none",
    "country_code": "US",
    "excludeTags": ["nav", "footer", "aside"],
    "timeout": 60000
  }'
```

Response — the Markdown lives at `data.markdown`:

```json
{
  "data": {
    "markdown": "# Apple Inc. — Form 10-K\n\n## Item 1A. Risk Factors\n\nThe Company's business...",
    "url": "https://www.sec.gov/Archives/edgar/data/320193/..."
  }
}
```

EDGAR filing documents are static HTML, so `renderMode: "none"` is the fastest and cheapest choice. Use `"full"` only for JS-rendered IR pages or transcript players that hydrate client-side.

### Batch-distill many filings into a corpus

When you have dozens of filing/transcript/IR URLs, use batch distill — up to 100 URLs per job, 1 credit each:

```python
import os, time, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

def batch_distill(urls, poll_every=5):
    job = requests.post(f"{BASE}/openapi/v1/batch/distill", headers=HEADERS,
                        json={"urls": urls, "timeout": 60000}).json()["data"]
    job_id = job["id"]
    while True:
        status = requests.get(f"{BASE}/openapi/v1/batch/distill/{job_id}",
                              headers=HEADERS, params={"page": 0, "pageSize": 100}
                             ).json()["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            return status
        time.sleep(poll_every)

filing_urls = [
    "https://www.sec.gov/Archives/edgar/data/320193/000032019323000106/aapl-20230930.htm",
    "https://www.sec.gov/Archives/edgar/data/789019/000095017023035122/msft-20230630.htm",
    # ... up to 100 filing / transcript / IR URLs
]

job = batch_distill(filing_urls)
for r in job["results"]:
    if r["status"] == "completed":
        # data.markdown per URL — write to the corpus
        save_doc(r["url"], r["data"]["markdown"])
```

Or from the CLI with a file of URLs (one per line, `#` lines ignored):

```bash
thunderbit batch distill --file filings.txt --timeout 60000 -f json > corpus.json
```

### Chunk and embed for retrieval

Once you have clean Markdown, the RAG side is standard. Split on headings, embed, store. A minimal chunker:

```python
def chunk_markdown(md, max_chars=2000):
    chunks, buf = [], ""
    for line in md.splitlines(keepends=True):
        if line.startswith("#") and len(buf) > max_chars:
            chunks.append(buf); buf = ""
        buf += line
    if buf.strip():
        chunks.append(buf)
    return chunks

# for chunk in chunk_markdown(markdown):
#     vector = your_embedder.embed(chunk)
#     vector_store.upsert(id=..., vector=vector,
#                         metadata={"ticker": "AAPL", "doc": "10-K 2023", "url": url})
```

Now an analyst (or an agent) can ask "what new risk factors did AAPL add this year?" and retrieve the exact paragraphs with source URLs. The cost of building this corpus for, say, 50 filings is 50 credits — versus 1,000 credits if you'd misused `extract`.

## News & sentiment monitoring

For each ticker, monitor public news and press pages: distill the articles to clean text, then run your own LLM over them for sentiment and event tagging. Use a public news search URL such as Google News, and honor each source's robots.txt and terms.

```python
def monitor_news(ticker, article_urls):
    job = batch_distill(article_urls)               # 1 credit per article
    items = []
    for r in job["results"]:
        if r["status"] != "completed":
            continue
        text = r["data"]["markdown"]
        verdict = llm_classify(ticker, text)        # your own LLM call
        items.append({"ticker": ticker, "url": r["url"], **verdict})
    return items
```

The LLM prompt does the analytical work — Thunderbit's job is to hand it clean text instead of HTML soup. A workable classification prompt:

```text
You are a financial news classifier. For the article below about {ticker},
return strict JSON:
  sentiment: one of "positive" | "neutral" | "negative"
  confidence: 0.0-1.0
  events: array from ["earnings","guidance","mna","litigation",
          "leadership","product","regulatory","buyback","dividend","other"]
  one_line: a neutral one-sentence summary
Article:
{text}
```

Distill is the right call here, not extract: you want the prose, the sentiment lives in your LLM, and 1 credit/article keeps a broad news sweep cheap.

## End-to-end pipeline

Here is one cohesive, runnable script that ties everything together: a watchlist of tickers → fundamentals per ticker → batch-distill the latest filings into a versioned `/corpus` folder → snapshot the fundamentals → diff against the last run and alert on meaningful moves. It includes the 401/402/429 backoff the [API reference](docs/thunderbit-api-reference.md) prescribes.

```python
import os, json, time, random, datetime, pathlib, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

CORPUS = pathlib.Path("corpus")
SNAPSHOTS = pathlib.Path("snapshots")
CORPUS.mkdir(exist_ok=True); SNAPSHOTS.mkdir(exist_ok=True)

# ticker -> (quote page, list of filing/IR/transcript URLs)
WATCHLIST = {
    "AAPL": {
        "quote": "https://example-finance.com/quote/AAPL",
        "filings": [
            "https://www.sec.gov/Archives/edgar/data/320193/000032019323000106/aapl-20230930.htm",
        ],
    },
    "MSFT": {
        "quote": "https://example-finance.com/quote/MSFT",
        "filings": [
            "https://www.sec.gov/Archives/edgar/data/789019/000095017023035122/msft-20230630.htm",
        ],
    },
}

FUNDAMENTALS_SCHEMA = {
    "type": "object",
    "properties": {
        "ticker":         {"type": "TEXT",   "instruction": "Ticker symbol, uppercase"},
        "company":        {"type": "TEXT",   "instruction": "Full company name"},
        "price":          {"type": "NUMBER", "instruction": "Last price USD; strip $ and commas; decimal only"},
        "market_cap":     {"type": "TEXT",   "instruction": "Market cap as displayed, keep suffix e.g. 2.95T"},
        "pe_ratio":       {"type": "NUMBER", "instruction": "P/E (TTM) decimal; empty if N/A"},
        "eps":            {"type": "NUMBER", "instruction": "EPS (TTM) USD decimal; negative for loss"},
        "dividend_yield": {"type": "NUMBER", "instruction": "Forward yield as percent value; empty if none"},
        "as_of":          {"type": "DATE",   "instruction": "Quote update date, ISO"},
    },
}


def call(path, body, method="POST", params=None, max_retries=6):
    """POST/GET with 429/5xx backoff, hard-stop on 402, no retry on 401."""
    for attempt in range(max_retries):
        if method == "POST":
            resp = requests.post(f"{BASE}{path}", headers=HEADERS, json=body, timeout=130)
        else:
            resp = requests.get(f"{BASE}{path}", headers=HEADERS, params=params, timeout=130)
        if resp.status_code == 200:
            return resp.json()["data"]
        if resp.status_code == 401:
            raise SystemExit("401 invalid API key — check THUNDERBIT_API_KEY, do not retry.")
        if resp.status_code == 402:
            raise SystemExit("402 out of credits — top up at https://thunderbit.com/billing.")
        if resp.status_code == 429 or resp.status_code >= 500:
            wait = min(2 ** attempt + random.random(), 30)
            print(f"  {resp.status_code}; backing off {wait:.1f}s")
            time.sleep(wait); continue
        resp.raise_for_status()
    raise RuntimeError(f"{path}: exhausted retries")


def get_fundamentals(url):
    return call("/openapi/v1/extract", {
        "url": url, "schema": FUNDAMENTALS_SCHEMA,
        "renderMode": "full", "waitFor": 2000, "timeout": 60000,
    })


def refresh_corpus(ticker, filing_urls):
    if not filing_urls:
        return
    job = call("/openapi/v1/batch/distill", {"urls": filing_urls, "timeout": 60000})
    job_id = job["id"]
    while True:
        status = call(f"/openapi/v1/batch/distill/{job_id}", None,
                      method="GET", params={"page": 0, "pageSize": 100})
        if status["status"] in ("completed", "failed", "cancelled"):
            break
        time.sleep(5)
    day = datetime.date.today().isoformat()
    out = CORPUS / ticker / day
    out.mkdir(parents=True, exist_ok=True)
    for i, r in enumerate(status.get("results", [])):
        if r.get("status") == "completed":
            (out / f"doc_{i}.md").write_text(r["data"]["markdown"])
        else:
            print(f"  {ticker} filing failed: {r.get('url')} ({r.get('error')})")


def load_last_snapshot():
    snaps = sorted(SNAPSHOTS.glob("*.json"))
    return json.loads(snaps[-1].read_text()) if snaps else {}


def diff_and_alert(prev, curr, pct=0.05):
    for ticker, now in curr.items():
        before = prev.get(ticker)
        if not before:
            print(f"ALERT {ticker}: newly tracked")
            continue
        p0, p1 = before.get("price"), now.get("price")
        if p0 and p1 and abs(p1 - p0) / p0 >= pct:
            print(f"ALERT {ticker}: price moved {(p1 - p0) / p0:+.1%} ({p0} -> {p1})")
        for metric in ("pe_ratio", "eps", "dividend_yield"):
            a, b = before.get(metric), now.get(metric)
            if a is not None and b is not None and a != b:
                print(f"NOTE  {ticker}: {metric} {a} -> {b}")


def run():
    prev = load_last_snapshot()
    curr = {}
    for ticker, cfg in WATCHLIST.items():
        print(f"== {ticker} ==")
        curr[ticker] = get_fundamentals(cfg["quote"])
        refresh_corpus(ticker, cfg["filings"])
    stamp = datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")
    (SNAPSHOTS / f"{stamp}.json").write_text(json.dumps(curr, indent=2))
    diff_and_alert(prev, curr)
    print("done")


if __name__ == "__main__":
    run()
```

Run it daily and you get a versioned fundamentals history, a dated filings corpus per ticker, and alerts when a metric crosses your threshold.

## No-code version: Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper Chrome extension**. An analyst with no code can run the identical workflow:

1. Install the **Thunderbit** extension from the Chrome Web Store and sign in. A free tier is available — new accounts get a monthly page allowance. (Sign-in is required; there's no "no-sign-up" path.)
2. Open a public quote or screener page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns.
3. Edit the columns in plain language (rename `market_cap`, tweak an instruction for messy numbers), then click **Scrape**. For a screener that lists many tickers, use **Scrape Subpages** to follow each row into its quote/detail page — the no-code version of a list→detail pipeline.
4. Export to **Excel, Google Sheets, Airtable, or Notion**, or **schedule** a recurring run (e.g. a pre-market refresh).

The field names and instructions transfer directly to the API/CLI — design and validate a schema in the extension, then port it into the pipeline above for automation at scale.

## Automate it

- **Cron — pre-market refresh.** Schedule the pipeline before the open so fresh fundamentals and a current corpus are ready when you start:

  ```cron
  # 07:30 America/New_York, Mon–Fri
  30 7 * * 1-5  cd /opt/research && /usr/bin/python3 pipeline.py >> logs/run.log 2>&1
  ```

- **Batch webhooks.** For large corpus rebuilds, skip polling — pass a `webhook` so Thunderbit notifies you when the job finishes, and verify the `secret`:

  ```json
  {
    "urls": ["https://www.sec.gov/Archives/edgar/data/..."],
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "your-signing-secret"
    }
  }
  ```

- **Corpus versioning.** The pipeline writes `corpus/<TICKER>/<date>/`. Keep the dated folders (or commit them to git) so you can re-embed and diff "this quarter's risk factors vs. last quarter's."
- **Alerting on thresholds.** `diff_and_alert` already fires on a ±5% price move or any change in P/E, EPS, or yield; wire its output into Slack/email/PagerDuty.

## Cost & scale

The decision that dominates your bill is **distill-for-RAG vs. extract-for-metrics**:

| Job | Endpoint | Cost | When |
|-----|----------|------|------|
| Fundamentals table (numbers you'll diff) | `extract` | 20 credits/page | One quote/screener page per ticker |
| Filings / transcripts / IR text (for RAG) | `distill` | 1 credit/page | Long prose you want to chunk + embed |
| Schema discovery | `suggest_fields` | 1 credit* | Once per page layout |
| Status polling | — | free | Batch/async jobs |
*\* Some Thunderbit docs list field suggestion as free — confirm against your plan.*

Worked example: a 50-ticker watchlist refreshing fundamentals daily costs 50 × 20 = 1,000 credits/day. Building a one-time RAG corpus from 50 annual filings costs just 50 × 1 = 50 credits. If you'd reflexively run `extract` over those filings, it would be 50 × 20 = 1,000 credits — 20× more for data you don't need as fields.

**Cache unchanged filings.** A 10-K doesn't change after it's filed. Key your corpus on the filing's accession number / URL and never re-distill a document you've already stored — only distill new filings. The same dedupe logic on quote pages (skip tickers whose `as_of` hasn't advanced) saves the expensive `extract` credits.

## Responsible use

- **Public sources only.** EDGAR filings, public IR pages, public quote pages, and public news. Do **not** target login-walled or paywalled data, premium terminals, or authenticated areas.
- **Respect `robots.txt`, Terms of Service, and market-data redistribution terms.** Many market-data providers restrict redistribution of quotes; use sources you're entitled to use, and prefer EDGAR for filings — it's built for public programmatic access.
- **Throttle.** Use batch with reasonable concurrency and don't hammer a single origin (EDGAR in particular asks clients to be considerate). Cache and dedupe.
- **Not investment advice; mind data licensing.** This pipeline produces *data*, not recommendations. Nothing here is investment advice, and you are responsible for complying with the licensing and terms of every source you ingest.
- **Minimize personal data.** Favor company-level and instrument-level public data; don't collect personal data you have no lawful basis to process.

## Troubleshooting & FAQ

**Numbers come back wrong / locale formatting.** Quote pages mix `$1,234.56`, `2.3B`, `1.85%`, `1.234,56` (EU locale), and `N/A`. The fix is the `instruction` string, not post-processing: be explicit — "strip the $ and thousands separators, decimal point only," "keep the magnitude suffix as text," "leave empty if N/A." For magnitude-suffixed values like market cap, keep them as `TEXT` and expand `T/B/M` yourself downstream.

**The fundamentals are empty (JS-rendered table).** The numbers load via a client-side widget. Set `renderMode: "full"` to use a headless browser and add `waitFor` (e.g. `2000` ms) so the widget settles. If a single page is consistently slow, use the async endpoint (`/openapi/v1/async/extract` → poll the job) so you're not holding a connection open.

**Big filings time out.** 10-Ks are large. Raise `timeout` (distill allows up to 60000 ms), and for batches that include heavy pages prefer `/batch/distill` with a webhook over synchronous calls. If one URL in a batch fails, `results[].status` tells you which — retry only those.

**Pagination on screeners.** A screener may paginate (`?page=2`) or lazy-load. Enumerate the page URLs and feed them to `/batch/extract` (up to 100 per job), or scroll-load with `renderMode: "full"` + `waitFor`. Batch poll results are themselves paginated — `page` is 0-based, `pageSize` defaults to 20 (max 100).

**Geo-specific listings.** Some exchanges/portals serve different content by region. Route the request with `country_code` (HTTP body) / `countryCode` (MCP tool) — e.g. `GB`, `DE`, `JP`.

**Which endpoint for what?** Need *fields you'll diff/alert on*? `extract`. Need *clean text for an LLM*? `distill` (20× cheaper). Discovering a new page layout? `suggest_fields` first. See the [API reference](docs/thunderbit-api-reference.md) for the full endpoint map, schema format, and error semantics.
