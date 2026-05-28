# Literature Intelligence & Research RAG with Thunderbit

Build a self-updating literature dataset from public scholarly sources, turn paper abstracts and full pages into an LLM-ready RAG corpus, and track new papers and citation trends on any topic over time — all from one AI web-data engine reachable from MCP, HTTP, or CLI.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## The data problem in research

Staying on top of a literature is a data-plumbing problem disguised as a reading problem. The pains are concrete:

- **Scattered across databases.** The same topic surfaces on arXiv, PubMed, Semantic Scholar, OpenReview, conference proceedings, and lab pages — each with its own HTML, its own metadata fields, and its own quirks. There is no single clean feed for "every new paper on retrieval-augmented generation this month."
- **Mixed formats: HTML and PDF.** The structured metadata lives in an HTML listing or abstract page; the actual content lives in a PDF. You need both — the metadata to organize and the text to reason over.
- **Manual lit-review doesn't scale.** Reading abstracts, copying titles, authors, DOIs, and citation counts into a spreadsheet, then re-reading PDFs to answer a question — it's slow, error-prone, and impossible to schedule. By the time a survey table is "done," new papers have appeared.
- **It decays fast.** A literature map built by hand on Monday is incomplete by Friday. New preprints land daily; citation counts climb; a once-obscure paper becomes the must-cite reference.

The fix is a small, repeatable pipeline: discover the schema once, extract a search-results page into a list of papers with their metadata, distill the abstract/paper pages into clean Markdown for a RAG corpus, store the result, and diff it against last week's run. Thunderbit gives you the extraction and distillation engine; this tutorial gives you the pipeline.

A standing caveat for this domain: **most of these sources also publish official APIs** (arXiv API, NCBI E-utilities for PubMed, the Semantic Scholar Graph API, OpenReview's API). Where an official API exists and covers your need, **prefer it** — it's the polite, stable, rate-limit-aware path. Use scraping for the gaps the APIs don't fill (rendered pages, fields the API omits, ad-hoc one-offs), and **throttle hard**: scholarly servers are sensitive to load, and a heavy crawl hurts everyone.

---

## What you'll build

- A **literature dataset** with `title` (TEXT), `authors` (TEXT), `abstract` (TEXT), `published` (DATE), `venue` (TEXT), `doi` (TEXT), `pdf_url` (URL), `paper_url` (URL), and `citation_count` (NUMBER) per paper.
- A **paper RAG corpus** — abstracts and full paper pages distilled to Markdown, chunked, embedded, and stored with citation metadata so an assistant can answer literature questions *with citations*.
- A **new-paper / citation tracker** — re-run extraction on a schedule, diff against the previous run to surface newly published papers, and aggregate citation counts to spot rising work.
- The same workflow, no-code, via the Thunderbit Chrome extension for non-developer teammates (and a CSV that imports cleanly into a reference manager like Zotero).

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

4. **An embeddings model + vector store** for the RAG portion. These are **pluggable** — any embeddings provider (OpenAI, Cohere, a local `sentence-transformers` model) and any vector store (pgvector, Chroma, Qdrant, FAISS, LanceDB) will do. This tutorial keeps the RAG mechanics deliberately light and cross-links the **[RAG tutorial](07-rag-knowledge-base.md)** for the full chunking/embedding/retrieval detail so we don't duplicate it here.

For full endpoint, schema, pricing, and error details, keep the [API reference](docs/thunderbit-api-reference.md) open in a tab.

---

## Three ways to call Thunderbit

All three surfaces hit the identical HTTP API underneath, so schemas and field instructions transfer verbatim between them.

| Surface | Entry point | When to use it |
|---------|-------------|----------------|
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and ad-hoc lit-review. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. Best for the scheduled weekly literature sweep. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, saving/reusing schemas. Best for prototyping and glue scripts. |

A good real-world split: **explore** a search page in MCP, **freeze a schema** with the CLI, **run it weekly** over the HTTP API.

---

## Step 1 — discover the schema

Don't guess field names. Let the AI inspect the page and propose a schema. Throughout this tutorial, the canonical public example is an **arXiv listing/search page** — for instance `https://arxiv.org/list/cs.IR/recent` (recent papers in Information Retrieval) or an arXiv search results URL like `https://arxiv.org/search/?searchtype=all&query=retrieval+augmented+generation`. arXiv is a public, scraping-friendly scholarly source; the same pattern works on PubMed, Semantic Scholar, and OpenReview public pages. **Before you point a scraper at any of them, read the "Responsible use" section, prefer the site's official API where it covers your need, and check the site's `robots.txt` and Terms of Service.**

`suggest_fields` costs **1 credit** and returns an array of field descriptors you can drop straight into an extract schema.

### (a) MCP — let the assistant call the tool

Prompt your assistant:

> Use Thunderbit to suggest fields for `https://arxiv.org/list/cs.IR/recent`. Focus on each paper entry: title, authors, abstract, submission date, the arXiv abstract-page link, and the PDF link.

The model issues a `thunderbit_suggest_fields` call with arguments like:

```json
{
  "url": "https://arxiv.org/list/cs.IR/recent",
  "prompt": "Extract each paper entry: title, authors, abstract, submission date, abstract-page link, PDF link",
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
    "url": "https://arxiv.org/list/cs.IR/recent",
    "prompt": "Extract each paper entry: title, authors, abstract, submission date, abstract-page link, PDF link",
    "country_code": "US"
  }'
```

### (c) CLI

```bash
thunderbit suggest-fields "https://arxiv.org/list/cs.IR/recent" \
  --prompt "Extract each paper entry: title, authors, abstract, submission date, abstract-page link, PDF link" \
  --country-code US
```

### The suggested-fields response

The fields come back at `data` (one array of descriptors):

```json
{
  "data": [
    { "name": "title",       "type": "TEXT", "instruction": "The paper title, plain text, no 'Title:' prefix" },
    { "name": "authors",     "type": "TEXT", "instruction": "All author names as a single comma-separated string in listed order" },
    { "name": "abstract",    "type": "TEXT", "instruction": "The full abstract text, if shown on the listing" },
    { "name": "published",   "type": "DATE", "instruction": "The submission / announcement date" },
    { "name": "paper_url",   "type": "URL",  "instruction": "Absolute link to the paper's abstract page" },
    { "name": "pdf_url",     "type": "URL",  "instruction": "Absolute link to the paper PDF" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;` (JS) or `fields = res["data"].get("fields", res["data"])` (Python).

Treat the suggestion as a starting point. You'll usually tighten the `instruction` strings — the single biggest lever on extraction quality — and add the fields the page doesn't surface in its first guess (`venue`, `doi`, `citation_count`).

---

## Step 2 — extract the paper list

Now turn the search-results page into structured rows. The schema is **not** raw JSON Schema; it's the Thunderbit-flavored shape: an array of objects, where each property has an UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction`. Extract costs **20 credits** per page.

Here's the full literature schema. Note the deliberate type choices: `published` is `DATE`, `citation_count` is `NUMBER`, and everything else (including the multi-author string and the DOI identifier) is `TEXT`.

### (a) HTTP — `curl`

arXiv listing pages are largely server-rendered, so `renderMode:"none"` often works and is the cheapest to render. For JS-heavy scholarly pages (Semantic Scholar, some OpenReview views) switch to `renderMode:"full"` and add a little `waitFor`.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://arxiv.org/list/cs.IR/recent",
    "renderMode": "none",
    "timeout": 60000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "title":          { "type": "TEXT",   "instruction": "The paper title as plain text; strip any 'Title:' label" },
          "authors":        { "type": "TEXT",   "instruction": "All author names as a single comma-separated string in listed order, e.g. 'Jane Doe, John Smith'" },
          "abstract":       { "type": "TEXT",   "instruction": "The abstract text if present on the listing; empty string if not shown" },
          "published":      { "type": "DATE",   "instruction": "The submission or announcement date for this paper" },
          "venue":          { "type": "TEXT",   "instruction": "Journal, conference, or 'arXiv preprint' if no formal venue is shown" },
          "doi":            { "type": "TEXT",   "instruction": "The DOI string only (e.g. 10.1145/1234567); empty string if none shown" },
          "pdf_url":        { "type": "URL",    "instruction": "Absolute link to the paper PDF" },
          "paper_url":      { "type": "URL",    "instruction": "Absolute link to the paper abstract / landing page" },
          "citation_count": { "type": "NUMBER", "instruction": "Citation count as digits only if the page shows one; null if not present" }
        }
      }
    }
  }'
```

Rows come back at `data`:

```json
{
  "data": [
    {
      "title": "Retrieval-Augmented Generation for Knowledge-Intensive Tasks",
      "authors": "Jane Doe, John Smith, Wei Zhang",
      "abstract": "We study retrieval-augmented generation across...",
      "published": "2024-09-18",
      "venue": "arXiv preprint",
      "doi": "",
      "pdf_url": "https://arxiv.org/pdf/2409.12345",
      "paper_url": "https://arxiv.org/abs/2409.12345",
      "citation_count": null
    },
    {
      "title": "Dense Retrieval with Hard Negatives",
      "authors": "Maria Garcia, Liang Chen",
      "abstract": "We propose a training scheme...",
      "published": "2024-09-17",
      "venue": "arXiv preprint",
      "doi": "",
      "pdf_url": "https://arxiv.org/pdf/2409.11111",
      "paper_url": "https://arxiv.org/abs/2409.11111",
      "citation_count": null
    }
  ]
}
```

> arXiv listings rarely show DOIs or citation counts — that's expected; those fields fill in when you extract a richer source (Semantic Scholar shows `citation_count`; the published version page shows the `doi`). The instructions above tell the engine to leave them empty/null rather than hallucinate.

### (b) Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

PAPER_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "title":          {"type": "TEXT",   "instruction": "The paper title as plain text; strip any 'Title:' label"},
            "authors":        {"type": "TEXT",   "instruction": "All author names as a single comma-separated string in listed order"},
            "abstract":       {"type": "TEXT",   "instruction": "The abstract text if present on the listing; empty string if not shown"},
            "published":      {"type": "DATE",   "instruction": "The submission or announcement date for this paper"},
            "venue":          {"type": "TEXT",   "instruction": "Journal, conference, or 'arXiv preprint' if no formal venue is shown"},
            "doi":            {"type": "TEXT",   "instruction": "The DOI string only; empty string if none shown"},
            "pdf_url":        {"type": "URL",    "instruction": "Absolute link to the paper PDF"},
            "paper_url":      {"type": "URL",    "instruction": "Absolute link to the paper abstract / landing page"},
            "citation_count": {"type": "NUMBER", "instruction": "Citation count as digits only if the page shows one; null if not present"},
        },
    },
}

def extract(url, schema, render_mode="none", timeout=60000, wait_for=0):
    body = {"url": url, "schema": schema, "renderMode": render_mode, "timeout": timeout}
    if wait_for:
        body["waitFor"] = wait_for
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json=body, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

papers = extract("https://arxiv.org/list/cs.IR/recent", PAPER_SCHEMA)
print(len(papers), "papers")
```

### (c) CLI — save the schema, then reuse it

Save the schema to a file once, then re-run it any time:

```bash
# papers.schema.json holds the array-of-objects schema above, then:
thunderbit extract "https://arxiv.org/list/cs.IR/recent" \
  --schema papers.schema.json \
  --render-mode none \
  --timeout 60000 \
  -f json > papers.json
```

> **`renderMode` cheat sheet:** `none` = raw HTML fetch (fastest/cheapest render; fine for arXiv/PubMed server-rendered pages); `basic` = light JS; `full` = headless browser for JS-heavy scholarly SPAs (Semantic Scholar, some OpenReview views). If a page comes back empty, the first thing to try is `renderMode:"full"` plus a larger `waitFor`. The valid `waitFor` range is `0–10000` ms; `extract` `timeout` is `5000–120000` ms (default `60000`).

---

## Step 3 — build the RAG corpus with distill

This is the **highest-ROI step** in the whole pipeline. Step 2 gives you structured metadata (great for filtering, sorting, and citations). But to answer a question like *"what methods reduce hallucination in RAG?"* you need the **text**, not just the metadata — and you almost never need that text as structured fields. So you **distill** the abstract/paper pages into clean Markdown.

The economics are decisive: **distill is 1 credit per page; extract is 20.** For text you'll chunk and embed, distill is a flat **20× cheaper**. Distilling 1,000 abstract pages costs 1,000 credits; extracting them would cost 20,000.

`batch/distill` handles **up to 100 URLs per job** at **1 credit/URL**. Feed it the `paper_url` values from Step 2.

### (a) HTTP — create the batch-distill job, then poll

```bash
# 1. Create the batch-distill job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://arxiv.org/abs/2409.12345",
      "https://arxiv.org/abs/2409.11111"
    ],
    "timeout": 30000
  }'
# → { "data": { "id": "batch_dist_001", "status": "pending", "total": 2 } }

# 2. Poll for results (page is 0-based; pageSize default 20, max 100)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/distill/batch_dist_001?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

Each completed result carries the page as Markdown at `data.markdown` (per the [API reference](docs/thunderbit-api-reference.md) §3.3). For abstract pages the Markdown is exactly what a RAG index wants: the title, authors, the full abstract, and the metadata block, with navigation and boilerplate stripped.

### (b) Python — distill, chunk, embed, store

The chunking and embedding below are **pluggable placeholders** — swap in your own splitter, embeddings provider, and vector store. See the **[RAG tutorial](07-rag-knowledge-base.md)** for the production-grade version (overlap strategy, metadata filtering, hybrid search, reranking). The only Thunderbit-specific part is `batch_distill`.

```python
import os, time, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

def batch_distill(urls, poll_every=3, timeout_ms=30000):
    """Distill up to 100 URLs per job; returns [{'url':..., 'markdown':...}]."""
    out = []
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = requests.post(f"{BASE}/openapi/v1/batch/distill", headers=HEADERS,
                            json={"urls": chunk, "timeout": timeout_ms}).json()["data"]
        job_id = job["id"]
        while True:
            status = requests.get(f"{BASE}/openapi/v1/batch/distill/{job_id}",
                                  headers=HEADERS, params={"page": 0, "pageSize": 100}
                                 ).json()["data"]
            if status["status"] in ("completed", "failed", "cancelled"):
                break
            time.sleep(poll_every)  # polling is free
        for result in status.get("results", []):
            if result.get("status") != "completed":
                print(f"  ! distill failed: {result['url']} ({result.get('error')})")
                continue
            md = (result.get("data") or {}).get("markdown", "")
            out.append({"url": result["url"], "markdown": md})
    return out

# --- pluggable RAG layer (see 07-rag-knowledge-base.md for the real thing) ---
def chunk(text, size=1200, overlap=150):
    chunks, i = [], 0
    while i < len(text):
        chunks.append(text[i:i + size])
        i += size - overlap
    return [c for c in chunks if c.strip()]

def embed(texts):
    # Replace with your embeddings provider (OpenAI, Cohere, sentence-transformers, ...)
    raise NotImplementedError("Plug in your embeddings model here")

def upsert(vector_store, records):
    # Replace with pgvector / Chroma / Qdrant / FAISS upsert
    raise NotImplementedError("Plug in your vector store here")

def build_corpus(papers, vector_store):
    """papers: rows from Step 2 (must carry paper_url + metadata)."""
    urls = [p["paper_url"] for p in papers if p.get("paper_url")]
    meta_by_url = {p["paper_url"]: p for p in papers if p.get("paper_url")}
    distilled = batch_distill(urls)

    records = []
    for d in distilled:
        meta = meta_by_url.get(d["url"], {})
        for j, piece in enumerate(chunk(d["markdown"])):
            records.append({
                "id": f"{d['url']}#chunk-{j}",
                "text": piece,
                # citation metadata travels WITH the vector so answers can cite
                "metadata": {
                    "title": meta.get("title"),
                    "authors": meta.get("authors"),
                    "published": meta.get("published"),
                    "venue": meta.get("venue"),
                    "doi": meta.get("doi"),
                    "paper_url": d["url"],
                    "pdf_url": meta.get("pdf_url"),
                },
            })
    vectors = embed([r["text"] for r in records])
    for r, v in zip(records, vectors):
        r["embedding"] = v
    upsert(vector_store, records)
    return len(records)
```

The crucial detail is the **metadata block**: store `doi`, `paper_url`, `title`, and `authors` *alongside* each chunk's vector. When the assistant retrieves a chunk to answer a question, it can cite the source paper precisely — which is the whole point of a research RAG. Everything else about retrieval, reranking, and answer synthesis is covered in the **[RAG tutorial](07-rag-knowledge-base.md)**; we won't duplicate it here.

### (c) CLI / MCP

```bash
# CLI: batch-distill abstract pages from a file (one URL per line)
thunderbit batch distill --file paper_urls.txt --timeout 30000 -f json > distilled.json
```

In an assistant: *"Use Thunderbit to batch-distill these arXiv abstract URLs to Markdown, then poll until done."* It calls `thunderbit_batch_distill_create` (args `urls`, `timeout?`) then `thunderbit_batch_distill_status` (args `jobId`, `page?`, `pageSize?`).

---

## Tracking new papers & trends

A literature is a moving target. Re-run Step 2 on a schedule and **diff against the previous run** to surface what's new, then aggregate citation counts for a trend signal.

- **New papers.** Key each run's rows by a stable identifier — prefer `doi`, fall back to `paper_url` (the arXiv abstract URL is stable). Any key not in the previous snapshot is a new paper.
- **Citation trend.** When your source exposes `citation_count` (Semantic Scholar does; arXiv doesn't), store it per run and compute the delta. A paper whose citations jumped this week is "rising."

```python
def stable_key(row):
    """Dedupe by DOI when present, else by the (stable) abstract-page URL."""
    return (row.get("doi") or "").strip().lower() or (row.get("paper_url") or "").strip().lower()

def diff_new_papers(prev_keys, rows):
    new = [r for r in rows if stable_key(r) and stable_key(r) not in prev_keys]
    return new

def citation_deltas(prev_counts, rows):
    """prev_counts: {key: citation_count} from the last run."""
    rising = []
    for r in rows:
        k, c = stable_key(r), r.get("citation_count")
        if k and c is not None and k in prev_counts and prev_counts[k] is not None:
            delta = c - prev_counts[k]
            if delta > 0:
                rising.append({"key": k, "title": r.get("title"),
                               "old": prev_counts[k], "new": c, "delta": delta})
    rising.sort(key=lambda x: x["delta"], reverse=True)
    return rising
```

This is the literature analogue of the price-drop diff in the [real estate tutorial](01-real-estate.md): same snapshot-and-compare pattern, different signal.

---

## End-to-end pipeline

Here's one cohesive, runnable Python script that ties it all together: extract the topic search page → store a snapshot in SQLite → diff for new papers → batch-distill the new papers' pages into a `corpus/` directory of Markdown files (ready to feed your RAG indexer). Error handling follows the reference: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Weekly literature-intelligence pipeline built on the Thunderbit Open API."""
import os, time, json, random, sqlite3, datetime, pathlib, hashlib
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
# A public, scraping-friendly scholarly listing you're authorized to access.
# Prefer the arXiv API where it covers your need; throttle this politely.
SEARCH_URL = "https://arxiv.org/list/cs.IR/recent"
DB = "literature.sqlite"
CORPUS = pathlib.Path("corpus")

PAPER_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "title":          {"type": "TEXT",   "instruction": "The paper title as plain text; strip any 'Title:' label"},
        "authors":        {"type": "TEXT",   "instruction": "All author names as a single comma-separated string in listed order"},
        "abstract":       {"type": "TEXT",   "instruction": "The abstract text if present; empty string if not shown"},
        "published":      {"type": "DATE",   "instruction": "The submission or announcement date for this paper"},
        "venue":          {"type": "TEXT",   "instruction": "Journal, conference, or 'arXiv preprint' if no formal venue is shown"},
        "doi":            {"type": "TEXT",   "instruction": "The DOI string only; empty string if none shown"},
        "pdf_url":        {"type": "URL",    "instruction": "Absolute link to the paper PDF"},
        "paper_url":      {"type": "URL",    "instruction": "Absolute link to the paper abstract / landing page"},
        "citation_count": {"type": "NUMBER", "instruction": "Citation count as digits only if shown; null if not present"},
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
            time.sleep(min(60, 2 ** i) + random.uniform(0, 1))  # backoff + jitter
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"Gave up on {path} after {attempts} attempts")

def _get(path, params=None):
    r = requests.get(f"{BASE}{path}", headers=HEADERS, params=params or {}, timeout=60)
    r.raise_for_status()
    return r.json()

def stable_key(row):
    return (row.get("doi") or "").strip().lower() or (row.get("paper_url") or "").strip().lower()

def extract_papers():
    res = _post("/openapi/v1/extract", {
        "url": SEARCH_URL, "schema": PAPER_SCHEMA,
        "renderMode": "none", "timeout": 60000,
    })
    return res["data"]

def batch_distill(urls):
    """Distill paper pages in chunks of 100 (the batch ceiling)."""
    out = []
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/distill", {"urls": chunk, "timeout": 30000})["data"]
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
            out.append({"url": result["url"], "markdown": md})
    return out

def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS papers (
        run_date TEXT, key TEXT, title TEXT, authors TEXT, abstract TEXT,
        published TEXT, venue TEXT, doi TEXT, pdf_url TEXT, paper_url TEXT,
        citation_count REAL,
        PRIMARY KEY (run_date, key))""")
    con.commit()
    return con

def previous_keys(con):
    cur = con.execute("SELECT key FROM papers WHERE run_date = "
                      "(SELECT MAX(run_date) FROM papers)")
    return {row[0] for row in cur.fetchall()}

def save_snapshot(con, run_date, rows):
    for r in rows:
        k = stable_key(r)
        if not k:
            continue
        con.execute("INSERT OR REPLACE INTO papers VALUES (?,?,?,?,?,?,?,?,?,?,?)", (
            run_date, k, r.get("title"), r.get("authors"), r.get("abstract"),
            r.get("published"), r.get("venue"), r.get("doi"),
            r.get("pdf_url"), r.get("paper_url"), r.get("citation_count")))
    con.commit()

def write_corpus(distilled, meta_by_url):
    CORPUS.mkdir(exist_ok=True)
    for d in distilled:
        meta = meta_by_url.get(d["url"], {})
        slug = hashlib.sha1(d["url"].encode()).hexdigest()[:12]
        front = (f"---\ntitle: {meta.get('title')!r}\nauthors: {meta.get('authors')!r}\n"
                 f"published: {meta.get('published')!r}\ndoi: {meta.get('doi')!r}\n"
                 f"paper_url: {d['url']!r}\n---\n\n")
        (CORPUS / f"{slug}.md").write_text(front + d["markdown"], encoding="utf-8")

def main():
    run_date = datetime.date.today().isoformat()
    con = init_db()
    prev = previous_keys(con)

    rows = extract_papers()
    print(f"Found {len(rows)} papers on the search page")
    save_snapshot(con, run_date, rows)

    new_papers = [r for r in rows if stable_key(r) and stable_key(r) not in prev]
    print(f"{len(new_papers)} NEW papers since last run")

    # Only distill the NEW papers — cheap (1 credit each) and dedupes by design.
    new_urls = [r["paper_url"] for r in new_papers if r.get("paper_url")]
    meta_by_url = {r["paper_url"]: r for r in new_papers if r.get("paper_url")}
    distilled = batch_distill(new_urls)
    write_corpus(distilled, meta_by_url)
    print(f"Distilled {len(distilled)} new paper pages into {CORPUS}/")

    for r in new_papers[:10]:
        print(f"  + {r.get('published')}  {r.get('title')}")

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire this to your alerting (Slack/email)
        raise
```

Run it weekly. On the second run it prints exactly which papers are new, and drops fresh Markdown into `corpus/` for your RAG indexer to pick up — with title/authors/DOI front-matter so every chunk stays citable.

---

## No-code version: the Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper** Chrome extension. Non-developer teammates — a librarian, a research assistant, a grad student — can run the identical workflow without writing code, and the schema they build transfers verbatim to the API.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **AI Suggest Fields.** Open the public scholarly search/listing page (e.g. an arXiv listing or a PubMed results page), click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns (title, authors, abstract, published, links).
3. **Edit columns** in plain language. Rename, drop, or add columns (`venue`, `doi`, `citation_count`); tune each column's instruction exactly like the `instruction` strings in the API schema.
4. **Scrape**, then **Scrape Subpages** to follow each row's `paper_url` and pull richer per-paper fields — the no-code equivalent of the list → distill/extract pipeline in Steps 2–3.
5. **Export** to Excel, Google Sheets, Airtable, or Notion. For reference managers, export **CSV** and import it into **Zotero** (File → Import) so your literature dataset lands straight in your citation library.

The recommended flow: design and validate the schema interactively in the extension, then port the field names and instructions to the CLI/API for automation at scale. They map one-to-one.

---

## Automate it

- **Cron — weekly literature sweep.** New preprints land daily, but a weekly cadence is usually enough for a survey and is kinder to the source. The diff logic already compares against the most recent prior snapshot in SQLite:

  ```cron
  # Every Monday at 07:30, sweep the topic and update the corpus
  30 7 * * 1  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/lit/pipeline.py >> /var/log/lit.log 2>&1
  ```

- **Batch webhooks instead of polling.** For server-side jobs, supply a callback so Thunderbit notifies you when the batch finishes — no open connection needed:

  ```json
  {
    "urls": ["https://arxiv.org/abs/2409.12345", "..."],
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "whatever-signing-secret-you-choose"
    }
  }
  ```

  Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call, then write the Markdown to your `corpus/` and trigger re-indexing.

- **Corpus versioning.** Keep one snapshot row per `(run_date, key)` and timestamp your `corpus/` outputs (or commit them to git). That history powers "papers added this week," citation-trend charts, and reproducible literature reviews — you can always rebuild the RAG index from a specific date.

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit**, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**Worked example.** A weekly sweep of a topic that surfaces **300 papers**, of which **~40 are new**:

```
1 search-page extract × 20                 =  20 credits
40 new abstract pages distilled × 1        =  40 credits
≈ 60 credits / week
```

**Extract vs distill — a 20× decision.** This is the core cost lever for research RAG. You need *structured fields* (extract, 20 credits) only for the metadata you'll filter and sort on — title, authors, published, DOI, citation_count — and you only pay it once per search page. You need *clean text* (distill, 1 credit) for everything that goes into the RAG index. Distilling 1,000 abstract pages costs **1,000 credits**; extracting them would cost **20,000**. Reach for distill by default; reserve extract for the metadata table.

**Prefer official APIs, throttle politely:**

- **Use the source's API where it exists.** arXiv, PubMed (NCBI E-utilities), Semantic Scholar, and OpenReview all publish official APIs. For bulk metadata, those are faster, free of rendering cost, and explicitly sanctioned. Reserve scraping for rendered pages or fields the APIs omit.
- **Distill only what's new.** The diff already tells you which papers are new this run — don't re-distill the unchanged corpus.
- **Reasonable concurrency.** Batch caps at 100 URLs/job; chunk larger sets and don't fire many jobs at one scholarly origin simultaneously. Scholarly servers are sensitive to load.

---

## Responsible use

These guardrails apply to every workflow in this repo — academic research included, where they matter especially.

- **Public scholarly data only.** Target publicly accessible listing, search, and abstract pages (arXiv, PubMed, Semantic Scholar public pages, OpenReview public pages). Never build workflows against paywalled full-text behind a publisher login or any authenticated area.
- **Prefer official APIs.** Where a sanctioned API covers your need (arXiv API, NCBI E-utilities, Semantic Scholar Graph API, OpenReview API), use it instead of scraping. It's stable, fast, and respectful of the source.
- **Honor `robots.txt`, Terms of Service, and rate limits.** Check each site's `robots.txt` and ToS before pointing a scraper at it, and respect its crawl-rate directives. Scholarly servers are run by libraries and nonprofits on tight budgets — throttle hard and add jitter to scheduled runs.
- **Respect copyright on full text.** Abstracts, titles, and bibliographic metadata are fine to collect and index for search. **Full-text PDFs are usually copyrighted.** Index the *metadata* and *link* to the PDF; do not redistribute or republish copyrighted full text. Treat your RAG corpus as a private research aid, not a public mirror.
- **Throttle and cache.** Dedupe by DOI/`paper_url` and only fetch what changed. It saves credits and load on the source.

---

## Troubleshooting & FAQ

**The abstract comes back truncated or empty.** Some scholarly pages lazy-load the abstract or render it client-side. Switch the distill/extract call to `renderMode:"full"` and add `waitFor` (up to its `10000` ms ceiling) so the dynamic content settles before capture. Raise `timeout` too (`distill` allows up to `60000` ms; `extract` up to `120000` ms). For arXiv abstract pages `renderMode:"none"` is usually enough; Semantic Scholar typically needs `full`.

**Author lists are mangled (truncated, "et al.", or split rows).** This is an `instruction` problem, not an engine problem. Brief the field precisely: *"All author names as a single comma-separated string in listed order; expand any 'et al.' to the full list shown on the page; do not abbreviate first names."* If the listing only shows the first few authors, point the extraction at the per-paper `paper_url` (abstract page), which lists every author.

**Pagination on the search page.** Most scholarly listings paginate via a query parameter — arXiv uses `?skip=50&show=50`; PubMed and others use `&page=2`. Enumerate the page URLs and feed them as a list to **batch extract** with the paper schema, then merge and dedupe by `stable_key`. URL-based pagination is more reliable than scrolling.

**Duplicate papers across runs or sources.** A paper appears on arXiv *and* Semantic Scholar *and* in a proceedings listing. Dedupe on a stable identifier: prefer `doi` (canonical across sources), fall back to `paper_url` (the arXiv abstract URL is stable). The `stable_key` helper in the pipeline does exactly this — normalize to lowercase and strip whitespace before comparing.

**Big PDFs time out.** Two things. First, **distill the page URL (`paper_url` / abstract page), not the raw PDF** — the HTML abstract page carries the title, authors, and abstract cleanly and renders far faster than a multi-megabyte PDF. Second, if you must process a slow page, use the **async** endpoints: `POST /openapi/v1/async/distill` then poll `GET /openapi/v1/async/distill/{jobId}` every ~3s, and/or raise `timeout` toward the maximum. Don't hold an HTTP connection open on a heavy render.

**`citation_count` is always null on arXiv.** Expected — arXiv listings don't show citation counts. Pull `citation_count` from a source that does (Semantic Scholar public pages), or from its official API, and join on `doi`/`paper_url`. The instruction tells the engine to return null rather than guess, which is what you want.

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.
