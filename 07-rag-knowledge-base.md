# Building a RAG Knowledge Base with Thunderbit

Turn any public documentation site, blog, or open knowledge base into a clean, retrieval-augmented-generation corpus — discover URLs, distill them to LLM-ready Markdown for 1 credit each, chunk, embed, store, and answer questions with citations.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

---

## Why distill is the right tool for RAG

Retrieval-augmented generation is only as good as the text you feed it. The classic failure mode is ingesting raw HTML: your chunks fill up with navigation menus, cookie banners, sidebars, footers, ad slots, and `<script>` noise. Embeddings of that junk pull in irrelevant matches, your token budget evaporates on boilerplate, and the model cites a nav link instead of a paragraph.

`distill` solves this at the source. It takes a page and returns **clean, LLM-ready Markdown** — the article body, headings, lists, code blocks, and tables — with the chrome stripped out. That is exactly the shape a chunker and an embedding model want:

- **Markdown chunks cleanly.** Headings (`#`, `##`, `###`) give you natural semantic boundaries. You can split on them, then sub-split long sections, and every chunk stays coherent.
- **No boilerplate to embed.** `distill` drops nav/footer/aside by default, and you can name extra tags to drop with `excludeTags`. Less noise in, better retrieval out.
- **It is 20× cheaper than `extract`.** Per the [API reference](docs/thunderbit-api-reference.md), distill costs **1 credit** per page and extract costs **20 credits** per page. For RAG you almost never want structured fields — you want the *text*. That is a 20× cost difference on every page you ingest.

The rule from the reference is worth internalizing: *always ask whether you need structured fields (extract) or just clean text (distill).* For a knowledge base, the answer is clean text. Reach for `extract` only for the handful of metadata fields you want to filter on later (covered in "When to add extract" below).

---

## What you'll build

A complete, runnable RAG ingestion-and-query pipeline:

```
1. Discover URLs      → parse sitemap.xml (or a curated list), honoring robots.txt
2. Batch-distill      → POST /openapi/v1/batch/distill (up to 100 URLs/job) → Markdown
3. Chunk + embed      → split Markdown on headings/token windows, call embed()
4. Store              → vectors + metadata (source_url, title, chunk_id) in a vector store
5. Retrieve + answer  → embed the query, cosine top-k, prompt an LLM, return citations
6. Incremental refresh→ hash each page, re-distill only what changed (saves credits)
```

The embedding model, the LLM, and the vector store are all written as **pluggable stubs** with clear comments. Swap in OpenAI, Cohere, a local sentence-transformer, Chroma, FAISS, pgvector — the pipeline doesn't care. Thunderbit's job is the one part that's hard to do well: turning messy public web pages into clean text.

---

## Prerequisites

1. **A Thunderbit API key.** Create one at <https://app.thunderbit.com/console/api-keys>. Keys are prefixed `tb_`.
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

4. **An embeddings model and a vector store.** These are *yours to choose* and stay pluggable throughout. Any embeddings provider works (hosted API or a local model); any vector store works (SQLite with a cosine scan for small corpora, or Chroma / FAISS / pgvector for larger ones). The Python examples below use plain `requests` plus the standard library so they run anywhere, with comments showing where to drop in your provider.

For full endpoint, schema, pricing, and error details, keep the [API reference](docs/thunderbit-api-reference.md) open in a tab.

---

## Three ways to call Thunderbit

All three surfaces hit the identical HTTP API underneath, so arguments transfer verbatim between them.

| Surface | Entry point | When to use it |
|---------|-------------|----------------|
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and adding a few pages ad hoc. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. **For bulk ingestion, `batch/distill` over HTTP is the workhorse.** |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, dumping a URL list to JSON. Best for prototyping and glue scripts. |

A good split: **explore** a few pages in MCP to confirm distill output looks right, **prototype** the URL list with the CLI, then **run the ingestion** over the HTTP `batch/distill` endpoint.

---

## Step 1 — enumerate your sources

You need a list of URLs to ingest. The two most reliable sources are a site's `sitemap.xml` and a curated, hand-picked list.

Throughout this tutorial, treat `https://docs.example.com` as **a public documentation site you are authorized to crawl**. Before you point anything at a real site, read the "Responsible use" section and check the site's `robots.txt` and Terms of Service.

### From `sitemap.xml`

Most documentation sites and blogs publish a sitemap at `/sitemap.xml` (sometimes a sitemap index that points to child sitemaps). Parse it with the standard library:

```python
import urllib.robotparser
import xml.etree.ElementTree as ET
import requests

SITEMAP = "https://docs.example.com/sitemap.xml"
USER_AGENT = "my-rag-ingester/1.0"

def fetch_sitemap_urls(sitemap_url):
    """Return all <loc> URLs from a sitemap or sitemap index."""
    r = requests.get(sitemap_url, headers={"User-Agent": USER_AGENT}, timeout=30)
    r.raise_for_status()
    root = ET.fromstring(r.content)
    ns = {"sm": "http://www.sitemaps.org/schemas/sitemap/0.9"}
    urls = []
    # A sitemap index points to child sitemaps; recurse into them.
    for sm in root.findall(".//sm:sitemap/sm:loc", ns):
        urls.extend(fetch_sitemap_urls(sm.text.strip()))
    # A urlset lists the actual pages.
    for loc in root.findall(".//sm:url/sm:loc", ns):
        urls.append(loc.text.strip())
    return urls

def allowed_by_robots(urls, base="https://docs.example.com"):
    """Honor robots.txt — drop any URL we're not allowed to fetch."""
    rp = urllib.robotparser.RobotFileParser()
    rp.set_url(base.rstrip("/") + "/robots.txt")
    rp.read()
    return [u for u in urls if rp.can_fetch(USER_AGENT, u)]

all_urls = fetch_sitemap_urls(SITEMAP)
urls = allowed_by_robots(all_urls)
print(f"{len(urls)} crawlable URLs of {len(all_urls)} in sitemap")
```

### From a curated list

For a focused knowledge base you often want *only* the pages that matter — the guides, not the changelog or the marketing pages. Keep a plain text file, one URL per line, and filter the sitemap down to it (or skip the sitemap entirely):

```python
# urls.txt — one URL per line, blank lines and # comments ignored
def load_url_file(path):
    with open(path) as f:
        return [ln.strip() for ln in f
                if ln.strip() and not ln.startswith("#")]
```

Either way, you end up with a clean Python list of URLs to feed Step 2.

---

## Step 2 — batch-distill to Markdown

This is the heart of ingestion. `POST /openapi/v1/batch/distill` accepts **up to 100 URLs per job** at **1 credit per URL**, runs them server-side, and lets you poll the results. Each completed URL gives you Markdown at `results[].data.markdown`.

### (a) HTTP — create the job, then poll

```bash
# 1. Create the batch distill job (≤ 100 URLs)
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://docs.example.com/guide/getting-started",
      "https://docs.example.com/guide/configuration"
    ],
    "timeout": 60000
  }'
# → { "data": { "id": "batch_doc123", "status": "pending", "total": 2 } }

# 2. Poll for results (page is 0-based; pageSize default 20, max 100)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/distill/batch_doc123?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

The batch status response carries a job-level `status` plus a per-URL `results` array. For distill, each successful result holds the Markdown:

```json
{
  "data": {
    "id": "batch_doc123",
    "status": "completed",
    "total": 2,
    "completed": 2,
    "results": [
      { "url": "https://docs.example.com/guide/getting-started", "status": "completed", "data": { "markdown": "# Getting started\n\nClean body text...", "url": "https://docs.example.com/guide/getting-started" } },
      { "url": "https://docs.example.com/guide/configuration",    "status": "failed",    "error": "timeout" }
    ]
  }
}
```

Job-level `status` is `pending | processing | completed | failed | cancelled`. Per-URL `results[].status` lets you **retry only the failures** — re-submit just the `failed` URLs in a fresh job rather than paying for the whole batch again.

> **Reading the Markdown.** A single-page `distill` response puts the text at `data.markdown`. Inside a batch, each per-URL entry mirrors that shape at `results[].data.markdown`. Always guard for the per-URL `status` before reading `data`.

### (b) Python — a reusable batch-distill helper

```python
import time, requests

BASE = "https://openapi.thunderbit.com"
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}  # KEY from env

def batch_distill(urls, timeout=60000, poll_every=3):
    """Distill up to 100 URLs. Returns {url: markdown} for the ones that succeeded."""
    job = requests.post(f"{BASE}/openapi/v1/batch/distill", headers=HEADERS,
                        json={"urls": urls, "timeout": timeout}, timeout=140
                        ).json()["data"]
    job_id = job["id"]
    while True:
        status = requests.get(f"{BASE}/openapi/v1/batch/distill/{job_id}",
                              headers=HEADERS, params={"page": 0, "pageSize": 100},
                              timeout=60).json()["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            break
        time.sleep(poll_every)  # polling is free
    out = {}
    for result in status.get("results", []):
        if result.get("status") != "completed":
            print(f"  ! distill failed: {result['url']} ({result.get('error')})")
            continue
        md = (result.get("data") or {}).get("markdown", "")
        if md.strip():
            out[result["url"]] = md
    return out
```

### (c) CLI — distill a whole file of URLs

The CLI reads one URL per line from `--file` (blank lines and `#` comments are ignored) and prints JSON you can pipe into your indexer:

```bash
thunderbit batch distill --file urls.txt --timeout 60000 -f json > corpus.json
```

### (d) A single page on the fly

When you just want to test what distill returns for one page, or add a single doc:

```bash
# HTTP
curl -sS https://openapi.thunderbit.com/openapi/v1/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "url": "https://docs.example.com/guide/getting-started", "renderMode": "full", "excludeTags": ["nav","footer","aside"] }'

# CLI — Markdown straight to stdout
thunderbit distill "https://docs.example.com/guide/getting-started" -f markdown
```

The synchronous `data.markdown` is your input to Step 3.

---

## Step 3 — chunk + embed + store

Now turn Markdown into searchable vectors. Three pluggable pieces: a **chunker**, an **`embed()`** function, and a **store**. The chunker below splits on Markdown headings, then sub-splits any section that's too long, with a small overlap so context isn't lost across boundaries.

```python
import re, sqlite3, json, hashlib

# ---- Chunking ----------------------------------------------------------------
def chunk_markdown(md, target_chars=1500, overlap_chars=200):
    """Split Markdown on headings, then window long sections with overlap.
    target_chars ≈ 350–400 tokens; tune for your embedding model's context."""
    # Split into sections at each heading line (keeps the heading with its body).
    parts = re.split(r"(?m)^(#{1,6}\s.*)$", md)
    sections, buf = [], ""
    for piece in parts:
        if re.match(r"^#{1,6}\s", piece or ""):
            if buf.strip():
                sections.append(buf.strip())
            buf = piece + "\n"
        else:
            buf += (piece or "")
    if buf.strip():
        sections.append(buf.strip())

    chunks = []
    for sec in sections:
        if len(sec) <= target_chars:
            chunks.append(sec)
            continue
        start = 0
        while start < len(sec):
            chunks.append(sec[start:start + target_chars])
            start += target_chars - overlap_chars
    return [c for c in chunks if c.strip()]

def title_of(md, fallback):
    """First H1/H2 becomes the document title; fall back to the URL."""
    m = re.search(r"(?m)^#{1,2}\s+(.*)$", md)
    return m.group(1).strip() if m else fallback

# ---- Embeddings (PLUGGABLE) --------------------------------------------------
def embed(texts):
    """Return a list of float vectors, one per input string.

    Drop in ANY provider here — a hosted embeddings API (OpenAI, Cohere,
    Voyage, Mistral) or a local model (sentence-transformers, Ollama).
    The rest of the pipeline only assumes: same dimensionality for every
    vector, and cosine similarity is meaningful. Example with a local model:

        from sentence_transformers import SentenceTransformer
        _m = SentenceTransformer("all-MiniLM-L6-v2")
        return _m.encode(texts, normalize_embeddings=True).tolist()
    """
    raise NotImplementedError("Wire up your embeddings provider here.")

# ---- Store (PLUGGABLE — SQLite shown; Chroma/FAISS/pgvector are drop-ins) ----
def init_store(db="rag.sqlite"):
    con = sqlite3.connect(db)
    con.execute("""CREATE TABLE IF NOT EXISTS chunks (
        chunk_id   TEXT PRIMARY KEY,
        source_url TEXT,
        title      TEXT,
        text       TEXT,
        vector     TEXT)""")            -- JSON-encoded float list
    con.execute("""CREATE TABLE IF NOT EXISTS pages (
        source_url TEXT PRIMARY KEY,
        content_hash TEXT)""")          -- for incremental refresh (Step 6)
    con.commit()
    return con

def index_page(con, source_url, markdown):
    title = title_of(markdown, source_url)
    pieces = chunk_markdown(markdown)
    vectors = embed(pieces)  # one batch call per page keeps it efficient
    # Remove any previous chunks for this URL so re-indexing is idempotent.
    con.execute("DELETE FROM chunks WHERE source_url = ?", (source_url,))
    for i, (text, vec) in enumerate(zip(pieces, vectors)):
        chunk_id = f"{hashlib.sha1(source_url.encode()).hexdigest()[:12]}-{i}"
        con.execute("INSERT OR REPLACE INTO chunks VALUES (?,?,?,?,?)",
                    (chunk_id, source_url, title, text, json.dumps(vec)))
    con.commit()
    return len(pieces)
```

Each stored chunk carries the metadata RAG needs to cite its source: `source_url`, `title`, and a stable `chunk_id`. For anything beyond a few thousand chunks, swap the SQLite table for a real vector store (Chroma, FAISS, or pgvector) — the `index_page` contract (text + metadata + vector) is identical; only the `INSERT` and the search call in Step 4 change.

---

## Step 4 — retrieve & answer with citations

Retrieval embeds the user's question with the *same* `embed()` function, finds the nearest chunks by cosine similarity, and builds a prompt that includes each chunk's `source_url` so the model can cite. The LLM call is another pluggable stub.

```python
import math, json

def _cosine(a, b):
    dot = sum(x * y for x, y in zip(a, b))
    na = math.sqrt(sum(x * x for x in a))
    nb = math.sqrt(sum(y * y for y in b))
    return dot / (na * nb) if na and nb else 0.0

def search(con, query, k=5):
    """Brute-force cosine top-k. For larger corpora let the vector store do this."""
    qvec = embed([query])[0]
    rows = con.execute("SELECT chunk_id, source_url, title, text, vector FROM chunks")
    scored = []
    for chunk_id, source_url, title, text, vec_json in rows:
        score = _cosine(qvec, json.loads(vec_json))
        scored.append((score, source_url, title, text))
    scored.sort(reverse=True, key=lambda t: t[0])
    return scored[:k]

# ---- LLM (PLUGGABLE) ---------------------------------------------------------
def llm(prompt):
    """Return the model's text answer for a prompt.

    Drop in ANY chat/completions provider (OpenAI, Anthropic, Mistral, a local
    model via Ollama). The pipeline only needs: prompt in, text out.
    """
    raise NotImplementedError("Wire up your LLM provider here.")

def answer(con, query, k=5):
    hits = search(con, query, k=k)
    context = "\n\n".join(
        f"[Source {i+1}] {title} — {url}\n{text}"
        for i, (_score, url, title, text) in enumerate(hits)
    )
    prompt = (
        "Answer the question using ONLY the sources below. "
        "Cite sources inline as [Source N]. If the answer isn't in the "
        "sources, say you don't know.\n\n"
        f"{context}\n\nQuestion: {query}\nAnswer:"
    )
    text = llm(prompt)
    citations = [{"n": i + 1, "title": t, "url": u}
                 for i, (_s, u, t, _txt) in enumerate(hits)]
    return {"answer": text, "citations": citations}
```

The returned object has a grounded answer plus a `citations` list mapping each `[Source N]` marker back to a real `source_url` — exactly what a trustworthy RAG app should surface to users.

---

## Incremental refresh

Documentation changes, but most pages don't change *every day*. Re-distilling the whole site on every run wastes credits. Store a **content hash per URL** and only re-distill pages whose content (or `Last-Modified` / sitemap `lastmod`) has changed.

```python
import hashlib

def page_hash(markdown):
    return hashlib.sha256(markdown.encode("utf-8")).hexdigest()

def needs_refresh(con, source_url, new_markdown):
    """True if this page is new or its content changed since last ingest."""
    row = con.execute("SELECT content_hash FROM pages WHERE source_url = ?",
                      (source_url,)).fetchone()
    return row is None or row[0] != page_hash(new_markdown)

def record_hash(con, source_url, markdown):
    con.execute("INSERT OR REPLACE INTO pages VALUES (?, ?)",
                (source_url, page_hash(markdown)))
    con.commit()
```

A common pattern: a sitemap's `<lastmod>` tells you *which* URLs probably changed, so you only spend distill credits on those. For sites without reliable `lastmod`, distill is cheap enough (1 credit) that you can re-distill and compare hashes — re-embedding is the expensive part you skip when the hash matches.

Schedule the refresh with cron:

```cron
# Every day at 04:30, refresh the knowledge base and log the run
30 4 * * *  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/rag/ingest.py >> /var/log/rag.log 2>&1
```

---

## End-to-end pipeline

One cohesive, runnable script that ties Steps 1–4 together: enumerate URLs, batch-distill in chunks of 100, skip unchanged pages, chunk + embed + store, and run a sample query. Error handling follows the [API reference](docs/thunderbit-api-reference.md) §4: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Ingest a public docs site into a RAG knowledge base via the Thunderbit Open API."""
import os, time, json, random, sqlite3, hashlib, re
import urllib.robotparser
import xml.etree.ElementTree as ET
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
SITEMAP = "https://docs.example.com/sitemap.xml"   # a public site you're authorized to crawl
SITE_BASE = "https://docs.example.com"
USER_AGENT = "my-rag-ingester/1.0"
DB = "rag.sqlite"
BATCH_CEILING = 100  # the batch endpoint caps at 100 URLs/job

class OutOfCredits(Exception):
    """402 — stop the run and alert a human."""

# ---- HTTP with backoff -------------------------------------------------------
def _post(path, body, attempts=6):
    """POST with backoff on 429/5xx; hard-stop on 402; never retry 401."""
    for i in range(attempts):
        r = requests.post(f"{BASE}{path}", headers=HEADERS, json=body, timeout=140)
        if r.status_code == 401:
            raise RuntimeError("401 invalid/missing API key — check THUNDERBIT_API_KEY")
        if r.status_code == 402:
            raise OutOfCredits("402 out of credits — top up at https://thunderbit.com/billing")
        if r.status_code == 429 or r.status_code >= 500:
            time.sleep(min(60, 2 ** i) + random.uniform(0, 1))  # exp backoff + jitter
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"Gave up on {path} after {attempts} attempts")

def _get(path, params=None):
    r = requests.get(f"{BASE}{path}", headers=HEADERS, params=params or {}, timeout=60)
    r.raise_for_status()
    return r.json()

# ---- Step 1: enumerate sources ----------------------------------------------
def fetch_sitemap_urls(sitemap_url):
    r = requests.get(sitemap_url, headers={"User-Agent": USER_AGENT}, timeout=30)
    r.raise_for_status()
    root = ET.fromstring(r.content)
    ns = {"sm": "http://www.sitemaps.org/schemas/sitemap/0.9"}
    urls = []
    for sm in root.findall(".//sm:sitemap/sm:loc", ns):
        urls.extend(fetch_sitemap_urls(sm.text.strip()))
    for loc in root.findall(".//sm:url/sm:loc", ns):
        urls.append(loc.text.strip())
    return urls

def allowed_by_robots(urls):
    rp = urllib.robotparser.RobotFileParser()
    rp.set_url(SITE_BASE.rstrip("/") + "/robots.txt")
    rp.read()
    return [u for u in urls if rp.can_fetch(USER_AGENT, u)]

# ---- Step 2: batch-distill ---------------------------------------------------
def batch_distill(urls, timeout=60000):
    job = _post("/openapi/v1/batch/distill", {"urls": urls, "timeout": timeout})["data"]
    job_id = job["id"]
    while True:
        status = _get(f"/openapi/v1/batch/distill/{job_id}",
                      {"page": 0, "pageSize": 100})["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            break
        time.sleep(3)  # polling is free
    out = {}
    for result in status.get("results", []):
        if result.get("status") != "completed":
            print(f"  ! distill failed: {result['url']} ({result.get('error')})")
            continue
        md = (result.get("data") or {}).get("markdown", "")
        if md.strip():
            out[result["url"]] = md
    return out

# ---- Step 3: chunk + embed + store -------------------------------------------
def chunk_markdown(md, target_chars=1500, overlap_chars=200):
    parts = re.split(r"(?m)^(#{1,6}\s.*)$", md)
    sections, buf = [], ""
    for piece in parts:
        if re.match(r"^#{1,6}\s", piece or ""):
            if buf.strip():
                sections.append(buf.strip())
            buf = piece + "\n"
        else:
            buf += (piece or "")
    if buf.strip():
        sections.append(buf.strip())
    chunks = []
    for sec in sections:
        if len(sec) <= target_chars:
            chunks.append(sec); continue
        start = 0
        while start < len(sec):
            chunks.append(sec[start:start + target_chars])
            start += target_chars - overlap_chars
    return [c for c in chunks if c.strip()]

def title_of(md, fallback):
    m = re.search(r"(?m)^#{1,2}\s+(.*)$", md)
    return m.group(1).strip() if m else fallback

def embed(texts):
    """PLUGGABLE — wire up any embeddings provider (hosted or local)."""
    raise NotImplementedError("Wire up your embeddings provider here.")

def init_store(db=DB):
    con = sqlite3.connect(db)
    con.execute("""CREATE TABLE IF NOT EXISTS chunks (
        chunk_id TEXT PRIMARY KEY, source_url TEXT, title TEXT, text TEXT, vector TEXT)""")
    con.execute("""CREATE TABLE IF NOT EXISTS pages (
        source_url TEXT PRIMARY KEY, content_hash TEXT)""")
    con.commit()
    return con

def page_hash(md):
    return hashlib.sha256(md.encode("utf-8")).hexdigest()

def needs_refresh(con, url, md):
    row = con.execute("SELECT content_hash FROM pages WHERE source_url = ?", (url,)).fetchone()
    return row is None or row[0] != page_hash(md)

def index_page(con, url, md):
    pieces = chunk_markdown(md)
    vectors = embed(pieces)
    title = title_of(md, url)
    con.execute("DELETE FROM chunks WHERE source_url = ?", (url,))
    for i, (text, vec) in enumerate(zip(pieces, vectors)):
        chunk_id = f"{hashlib.sha1(url.encode()).hexdigest()[:12]}-{i}"
        con.execute("INSERT OR REPLACE INTO chunks VALUES (?,?,?,?,?)",
                    (chunk_id, url, title, text, json.dumps(vec)))
    con.execute("INSERT OR REPLACE INTO pages VALUES (?, ?)", (url, page_hash(md)))
    con.commit()
    return len(pieces)

# ---- Orchestration -----------------------------------------------------------
def main():
    con = init_store()
    urls = allowed_by_robots(fetch_sitemap_urls(SITEMAP))
    print(f"{len(urls)} crawlable URLs")

    total_chunks = 0
    for start in range(0, len(urls), BATCH_CEILING):
        chunk_urls = urls[start:start + BATCH_CEILING]
        markdowns = batch_distill(chunk_urls)
        for url, md in markdowns.items():
            if not needs_refresh(con, url, md):
                continue  # unchanged since last run — skip embedding, save work
            n = index_page(con, url, md)
            total_chunks += n
            print(f"  indexed {url} ({n} chunks)")
    print(f"Done. {total_chunks} new/updated chunks.")

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire to your alerting (PagerDuty/Slack/email)
        raise
```

Fill in `embed()` (and the `llm()`/`search()`/`answer()` helpers from Step 4) and this script will crawl a sitemap, distill every allowed page to Markdown, skip anything unchanged, and build a queryable, citation-ready knowledge base.

---

## Pulling a single page on the fly

Not every addition justifies a batch job. When a teammate finds one great page that belongs in the corpus, the **Thunderbit AI Web Scraper Chrome extension** can distill it to Markdown right in the browser, no code required — the UI surface of the same engine that powers `distill`.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **Open the page** you want to add, click the extension, and use it to convert the page to clean Markdown.
3. **Drop the Markdown** into your ingestion folder (or paste it into a `urls.txt`-style queue) and let the next pipeline run chunk, embed, and store it.

For developers, the equivalent one-liner is the CLI: `thunderbit distill "<url>" -f markdown`. Either way you get the same clean text, so manual additions stay consistent with batch-ingested pages.

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit** *(some docs list it as free — confirm against your plan)*, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**The 20× decision, applied to a 5,000-page knowledge base:**

```
5,000 docs × 1 credit  (distill)  =   5,000 credits   ← what RAG needs
5,000 docs × 20 credits (extract)  = 100,000 credits   ← 20× more, for fields you won't query
```

If you reach for `extract` when all you needed was clean text, you pay 20× for nothing. RAG wants the *text*, so distill is the correct, cheap default.

**Keep the bill low at scale:**

- **Dedupe with a content hash.** The incremental-refresh step skips re-distilling and re-embedding unchanged pages — on a stable docs site, most runs touch only a handful of pages.
- **Mind the batch ceiling.** Each `batch/distill` job takes **at most 100 URLs**. Chunk a larger URL list into groups of 100 (the end-to-end script does this) rather than firing one oversized request.
- **Throttle.** Don't fire many batch jobs at the same origin simultaneously; add jitter to scheduled runs.

---

## When to add extract

Distill is the default for RAG, but a *small* dose of `extract` makes a corpus filterable. Use `extract` only for the few structured metadata fields you'll actually filter or sort on — for example a docs page's **product version**, **category**, or **last-updated date** — so you can scope retrieval ("only v3 docs") before the cosine search.

The pattern: distill the page for its text (1 credit) *and*, where filtering matters, run a tiny `extract` for just those metadata fields (20 credits), then store both. Keep the extract schema minimal — every field you don't filter on is wasted spend. Use `suggest_fields` (1 credit) to discover what metadata a page exposes, then trim the schema to the two or three fields that earn their place. The extract schema is the Thunderbit-flavored shape — an array (or single object) with UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction` per field — exactly as defined in the [API reference](docs/thunderbit-api-reference.md) §5.

For the overwhelming majority of pages, skip extract entirely. The text is what gets embedded and retrieved.

---

## Responsible use

These guardrails apply to every workflow in this repo — RAG ingestion included.

- **Public sources only.** Ingest publicly accessible documentation, blogs, and open knowledge bases. Never build a corpus from login-walled platforms, paywalled articles, or private/authenticated areas. If a page requires a sign-in or a subscription to read, it does not belong in this pipeline.
- **Check `robots.txt` and Terms of Service first.** Honor disallowed paths (the `allowed_by_robots` filter above is the minimum) and crawl-rate directives before pointing the ingester at any site.
- **Attribute sources.** A RAG answer without citations is a plagiarism machine. The Step 4 pipeline carries `source_url` through to every answer — keep it. Surface the citations to your users and link back to the original docs.
- **Throttle.** Batch with reasonable concurrency; don't hammer a single origin. Add jitter to scheduled runs.
- **Don't ingest personal data you have no lawful basis to process** (GDPR/CCPA). Public, non-personal documentation is the sweet spot for RAG.

---

## Troubleshooting & FAQ

**Distill returns empty or near-empty Markdown.** The page is almost certainly client-rendered. Set `renderMode:"full"` (headless browser) and add `waitFor` (up to its `10000` ms ceiling) so lazy/JS content loads before distill reads it. If you're on `renderMode:"none"` against a JS-heavy docs site, that's the cause.

**Boilerplate (nav/footer/sidebar) leaks into the Markdown.** Distill drops the common chrome by default; for stubborn templates, pass `excludeTags` (e.g. `["nav","footer","aside"]`) to drop specific tags, or `includeTags` to keep only the content tags. Cleaner Markdown means cleaner chunks and better retrieval.

**Huge pages or slow renders time out.** Raise `timeout` toward distill's `60000` ms max. For a single very large or slow page outside a batch, use the async endpoints — `POST /openapi/v1/async/distill` then poll `GET /openapi/v1/async/distill/{jobId}` every ~3s — so you don't hold an HTTP connection open while it renders.

**Sitemap edge cases.** Some sites publish a **sitemap index** (a sitemap of sitemaps) — the `fetch_sitemap_urls` helper recurses into child sitemaps for you. Others gzip the sitemap (`/sitemap.xml.gz`) or list it in `robots.txt` under `Sitemap:`; check there if `/sitemap.xml` 404s. If a site has no sitemap at all, fall back to a curated `urls.txt`.

**Chunk size tuning.** Too-large chunks dilute relevance and blow your context budget; too-small chunks lose surrounding context. Start at ~1,500 characters (~350–400 tokens) with ~200 characters of overlap, then tune to your embedding model's window and your retrieval quality. Splitting on Markdown headings first keeps each chunk topically coherent.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status`; re-submit only the `failed` URLs in a fresh batch job. Don't re-run the whole batch — you'd pay distill credits again for pages that already succeeded.

**The corpus is stale.** That's what incremental refresh is for. Schedule the ingest script via cron, store a content hash per URL, and re-embed only pages whose hash changed. On a stable docs site most scheduled runs are nearly free (mostly skipped pages plus free polling).

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.
