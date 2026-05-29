# Healthcare & Pharma Intelligence with Thunderbit

Build a structured clinical-trial dataset, monitor new trials, drug labels, and device recalls for changes, and distill long protocols and labels into a searchable RAG corpus — all from **public regulatory sources**, using one AI web-data engine reachable from MCP, HTTP, or CLI.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

> ⚠️ **Read this before you build anything in this domain.**
> - **Public, aggregate, non-personal regulatory data ONLY.** This tutorial targets public registry and regulatory pages (trial records, drug labels, recall notices). Never collect patient-level data, identifiable individuals, or any Protected Health Information (PHI).
> - **This is NOT medical advice** and the output is **not** for clinical decision-making, diagnosis, or treatment. It is competitive- and regulatory-intelligence tooling.
> - **Prefer official APIs.** ClinicalTrials.gov and openFDA both publish free, well-documented, structured APIs. Use them first. Treat scraping as a *fallback* for public pages that have no API.
> - **HIPAA / GDPR awareness.** Even on public sources, do not assemble datasets that re-identify individuals. Honor `robots.txt` and each site's Terms of Service.

---

## The data problem in life-sciences intel

Tracking the regulatory and clinical landscape is a data-plumbing problem disguised as a research problem. The pains are concrete:

- **Scattered sources.** Trial records live on ClinicalTrials.gov; drug labels live on DailyMed and the FDA label databases; device recalls live in FDA enforcement databases. Each has its own structure, vocabulary, and update cadence. There is no single clean feed for "every Phase 2 oncology trial that changed status this week, plus the matching label revisions."
- **Manual monitoring misses changes.** A trial flips from *Recruiting* to *Active, not recruiting*; a completion date slips; a label adds a boxed warning; a device recall is upgraded to Class I. If a human is eyeballing pages weekly, these slip through. The signal is in the *diff*, not the snapshot.
- **Unstructured long documents.** Protocols, statistical analysis plans, and full prescribing information run to dozens of pages of dense prose. To answer "what are the exclusion criteria?" or "what's the boxed warning?" you need that text chunked, indexed, and answerable with citations — not a PDF you skim by hand.
- **Volume.** A single condition can map to thousands of trials. Re-reading everything on every run is wasteful and slow.

The fix is a small, repeatable pipeline: discover the schema once, extract a public search-results page into a list of records, follow each record's detail link in a batch, distill the long documents into clean Markdown for RAG, store the result, and diff it against the previous run to surface what's *new* or *changed*. Thunderbit gives you the extraction and distill engine; this tutorial gives you the pipeline — and tells you where to use the official APIs instead.

---

## What you'll build

- A **clinical-trial dataset** with `nct_id`, `title`, `phase`, `status`, `condition`, `intervention`, `sponsor`, `start_date`, `completion_date`, `enrollment`, and `detail_url`.
- A **new-trial / recall monitor** — a scheduled diff that flags *new* trials and recalls and *status changes* (e.g. a trial moving to *Terminated*, a recall upgraded to *Class I*) since the previous run.
- A **protocol / label RAG corpus** — long trial protocols and drug labels distilled to clean Markdown, chunked and indexed for Q&A *with citations* (pairs with the [RAG tutorial](07-rag-knowledge-base.md)).
- A multi-stage pipeline: **(1)** a public search-results page → `suggest_fields` → extract the list of records with detail URLs; **(2)** **batch-extract** the detail pages for structured fields; **(3)** **batch-distill** the long protocol/label text into Markdown for RAG; **(4)** store + diff against the previous run.
- The same workflow, no-code, via the Thunderbit Chrome extension for non-developer teammates.

---

## Prerequisites

1. **Prefer the official APIs first.** Before scraping, check whether the data is already available as structured JSON:
   - **ClinicalTrials.gov** offers a free, modern REST API (the v2 data API). For trial *metadata*, use it.
   - **openFDA** offers free APIs for drug labels (`/drug/label`), device recalls/enforcement (`/device/enforcement`), and more.
   - **DailyMed** offers a public web-services API for label content (SPL documents).

   Use Thunderbit when a public page has **no API**, when an API field is missing from the structured feed but visible on the human page, or when you want the rendered page exactly as a human sees it. Scraping is the fallback, not the default.

2. **An API key.** Create one at <https://app.thunderbit.com/console/api-keys>. Keys are prefixed `tb_`.
3. **Export it** so the CLI, MCP server, and your scripts all pick it up automatically:

   ```bash
   export THUNDERBIT_API_KEY=tb_your_api_key_here
   ```

4. **Install a surface** (pick what fits your workflow):

   ```bash
   # CLI + SDK (local scripting / shell)
   npm i -g @thunderbit/thunderbit-cli

   # MCP server (AI assistants) — add to your MCP client config; npx pulls it on demand
   npx -y @thunderbit/mcp-server
   ```

For full endpoint, schema, pricing, and error details, keep the [API reference](docs/thunderbit-api-reference.md) open in a tab. For the chunk/embed/citation half of the RAG corpus, follow the [RAG tutorial](07-rag-knowledge-base.md) — this tutorial produces the clean Markdown it consumes.

---

## Three ways to call Thunderbit

All three surfaces hit the identical HTTP API underneath, so schemas and field instructions transfer verbatim between them.

| Surface | Entry point | When to use it |
|---------|-------------|----------------|
| **MCP** | `@thunderbit/mcp-server` | Inside an AI assistant (Claude Desktop/Code, Cursor, Cline). You describe the task; the model calls `thunderbit_*` tools. Best for exploration and ad-hoc pulls. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CI, webhooks. Any language. Best for the automated weekly monitor. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` | Shell pipelines, quick scripts, saving/reusing schemas. Best for prototyping and glue scripts. |

A good real-world split: **explore** in MCP, **freeze a schema** with the CLI, **run it on a schedule** over the HTTP API.

---

## Step 1 — discover the schema

Don't guess field names. Let the AI inspect the page and propose a schema. Throughout this tutorial, treat `https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer` as **a public registry search-results page**. ClinicalTrials.gov is designed to be a public registry — but read the "Responsible use" section first, prefer the official API for metadata, and check the site's `robots.txt` and Terms of Service before pointing a scraper at it.

`suggest_fields` costs **1 credit** and returns an array of field descriptors you can drop straight into an extract schema.

### (a) MCP — let the assistant call the tool

Prompt your assistant:

> Use Thunderbit to suggest fields for `https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer`. Focus on each trial in the results list: NCT ID, title, phase, recruitment status, condition, intervention, sponsor, start date, completion date, enrollment, and the link to the trial's detail page.

The model issues a `thunderbit_suggest_fields` call with arguments like:

```json
{
  "url": "https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer",
  "prompt": "Extract each trial result: NCT ID, title, phase, status, condition, intervention, sponsor, start date, completion date, enrollment, detail-page link",
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
    "url": "https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer",
    "prompt": "Extract each trial result: NCT ID, title, phase, status, condition, intervention, sponsor, start date, completion date, enrollment, detail-page link",
    "country_code": "US"
  }'
```

### (c) CLI

```bash
thunderbit suggest-fields "https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer" \
  --prompt "Extract each trial result: NCT ID, title, phase, status, condition, intervention, sponsor, start date, completion date, enrollment, detail-page link" \
  --country-code US
```

### The suggested-fields response

The fields come back at `data` (one array of descriptors):

```json
{
  "data": [
    { "name": "nct_id",          "type": "TEXT",   "instruction": "The NCT registry identifier, e.g. NCT01234567" },
    { "name": "title",           "type": "TEXT",   "instruction": "The brief / public title of the trial" },
    { "name": "phase",           "type": "TEXT",   "instruction": "Trial phase, e.g. Phase 1, Phase 2, Phase 3, or Not Applicable" },
    { "name": "status",          "type": "TEXT",   "instruction": "Recruitment status, e.g. Recruiting, Active not recruiting, Completed, Terminated" },
    { "name": "condition",       "type": "TEXT",   "instruction": "The condition(s) studied" },
    { "name": "intervention",    "type": "TEXT",   "instruction": "The intervention(s) / treatment(s) studied" },
    { "name": "sponsor",         "type": "TEXT",   "instruction": "The lead sponsor organization" },
    { "name": "start_date",      "type": "DATE",   "instruction": "Study start date" },
    { "name": "completion_date", "type": "DATE",   "instruction": "Primary or overall completion date" },
    { "name": "enrollment",      "type": "NUMBER", "instruction": "Target or actual enrollment count, digits only" },
    { "name": "detail_url",      "type": "URL",    "instruction": "Absolute link to the trial's detail page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both:
> `const fields = res.data?.fields ?? res.data;` (JS) or `fields = res["data"].get("fields", res["data"])` (Python).

Treat the suggestion as a starting point. You'll usually tighten the `instruction` strings — the single biggest lever on extraction quality.

---

## Step 2 — extract the trial list

Now turn the public search-results page into structured rows. The schema is **not** raw JSON Schema; it's the Thunderbit-flavored shape: an array of objects, where each property has an UPPERCASE `type` (`TEXT | NUMBER | URL | EMAIL | DATE`) and an `instruction`. Note the **`DATE`** types for `start_date` / `completion_date` and the **`NUMBER`** type for `enrollment`. Extract costs **20 credits** per page.

### (a) HTTP — `curl`

Registry result lists are rendered client-side, so set `renderMode:"full"` (headless browser) and give the lazy content a moment with `waitFor`.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer",
    "renderMode": "full",
    "waitFor": 3000,
    "timeout": 90000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "nct_id":          { "type": "TEXT",   "instruction": "The NCT registry identifier, e.g. NCT01234567" },
          "title":           { "type": "TEXT",   "instruction": "The brief / public title of the trial" },
          "phase":           { "type": "TEXT",   "instruction": "Trial phase exactly as shown, e.g. Phase 2; use Not Applicable if none" },
          "status":          { "type": "TEXT",   "instruction": "Recruitment status exactly as shown, e.g. Recruiting, Completed, Terminated" },
          "condition":       { "type": "TEXT",   "instruction": "The condition(s) studied, comma-separated if multiple" },
          "intervention":    { "type": "TEXT",   "instruction": "The intervention(s)/treatment(s), comma-separated if multiple" },
          "sponsor":         { "type": "TEXT",   "instruction": "The lead sponsor organization name" },
          "start_date":      { "type": "DATE",   "instruction": "Study start date; return ISO YYYY-MM-DD if a full date is shown" },
          "completion_date": { "type": "DATE",   "instruction": "Primary or overall completion date; ISO YYYY-MM-DD if shown" },
          "enrollment":      { "type": "NUMBER", "instruction": "Enrollment count, digits only, no commas" },
          "detail_url":      { "type": "URL",    "instruction": "Absolute link to this trial's detail page" }
        }
      }
    }
  }'
```

Rows come back at `data`:

```json
{
  "data": [
    { "nct_id": "NCT05012345", "title": "Study of Drug X in Advanced NSCLC", "phase": "Phase 2", "status": "Recruiting", "condition": "Non-Small Cell Lung Cancer", "intervention": "Drug X", "sponsor": "Acme Oncology", "start_date": "2024-03-01", "completion_date": "2026-09-30", "enrollment": 240, "detail_url": "https://clinicaltrials.gov/study/NCT05012345" },
    { "nct_id": "NCT04987654", "title": "Combination Therapy in Metastatic NSCLC", "phase": "Phase 3", "status": "Active, not recruiting", "condition": "Non-Small Cell Lung Cancer", "intervention": "Drug Y + Chemotherapy", "sponsor": "Globex Therapeutics", "start_date": "2022-06-15", "completion_date": "2025-12-31", "enrollment": 600, "detail_url": "https://clinicaltrials.gov/study/NCT04987654" }
  ]
}
```

### (b) Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

TRIAL_LIST_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "nct_id":          {"type": "TEXT",   "instruction": "The NCT registry identifier, e.g. NCT01234567"},
        "title":           {"type": "TEXT",   "instruction": "The brief / public title of the trial"},
        "phase":           {"type": "TEXT",   "instruction": "Trial phase exactly as shown, e.g. Phase 2; Not Applicable if none"},
        "status":          {"type": "TEXT",   "instruction": "Recruitment status exactly as shown, e.g. Recruiting, Completed, Terminated"},
        "condition":       {"type": "TEXT",   "instruction": "Condition(s) studied, comma-separated if multiple"},
        "intervention":    {"type": "TEXT",   "instruction": "Intervention(s)/treatment(s), comma-separated if multiple"},
        "sponsor":         {"type": "TEXT",   "instruction": "Lead sponsor organization name"},
        "start_date":      {"type": "DATE",   "instruction": "Study start date; ISO YYYY-MM-DD if a full date is shown"},
        "completion_date": {"type": "DATE",   "instruction": "Primary/overall completion date; ISO YYYY-MM-DD if shown"},
        "enrollment":      {"type": "NUMBER", "instruction": "Enrollment count, digits only, no commas"},
        "detail_url":      {"type": "URL",    "instruction": "Absolute link to this trial's detail page"},
    }},
}

def extract(url, schema, render_mode="full", timeout=90000, wait_for=3000):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema,
        "renderMode": render_mode, "timeout": timeout, "waitFor": wait_for,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

trials = extract("https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer", TRIAL_LIST_SCHEMA)
print(len(trials), "trials")
```

### (c) CLI — save the schema, then reuse it

Save the schema to a file once, then re-run it any time:

```bash
# trials.schema.json holds the array-of-objects schema above, then:
thunderbit extract "https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer" \
  --schema trials.schema.json \
  --render-mode full \
  --timeout 90000 \
  -f json > trials.json
```

> **`renderMode` cheat sheet:** `none` = raw HTML fetch (fastest/cheapest render; fine for server-rendered pages); `basic` = light JS; `full` = headless browser for JS-heavy SPAs like the registry UI. If a page comes back empty, the first thing to try is `renderMode:"full"` plus a larger `waitFor`. The valid `waitFor` range is `0–10000` ms; `extract` `timeout` is `5000–120000` ms (default `60000`).

---

## Step 3 — batch detail + distill protocols/labels

The results list rarely has everything. The detail page carries the full eligibility criteria, study arms, outcome measures, locations, and — for drug records — the full prescribing information. There are **two** jobs here, and they cost very differently:

- **Structured fields** you want to *query and diff* (status, dates, enrollment, sponsor) → use **`/batch/extract`** at **20 credits/URL**.
- **Long free-text documents** you want to *read, summarize, or feed a RAG index* (protocols, full labels) → use **`/batch/distill`** at **1 credit/URL** — 20× cheaper. Distill returns clean Markdown, exactly what the [RAG tutorial](07-rag-knowledge-base.md) chunks and embeds.

Both handle **up to 100 URLs per job**.

### 3a — batch-extract structured detail fields

A richer detail schema for the fields you'll diff:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "nct_id":          { "type": "TEXT",   "instruction": "The NCT registry identifier on this page, e.g. NCT01234567" },
      "title":           { "type": "TEXT",   "instruction": "Official title of the study" },
      "phase":           { "type": "TEXT",   "instruction": "Trial phase exactly as shown" },
      "status":          { "type": "TEXT",   "instruction": "Current overall recruitment status exactly as shown" },
      "condition":       { "type": "TEXT",   "instruction": "Condition(s) studied, comma-separated" },
      "intervention":    { "type": "TEXT",   "instruction": "Intervention(s)/treatment(s), comma-separated" },
      "sponsor":         { "type": "TEXT",   "instruction": "Lead sponsor organization" },
      "start_date":      { "type": "DATE",   "instruction": "Study start date; ISO YYYY-MM-DD if a full date is shown" },
      "completion_date": { "type": "DATE",   "instruction": "Primary completion date; ISO YYYY-MM-DD if shown" },
      "enrollment":      { "type": "NUMBER", "instruction": "Enrollment count, digits only" },
      "last_update_date":{ "type": "DATE",   "instruction": "The 'Last Update Posted' date if shown; ISO YYYY-MM-DD" }
    }
  }
}
```

Create the job, then poll:

```bash
# 1. Create the batch job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://clinicaltrials.gov/study/NCT05012345",
      "https://clinicaltrials.gov/study/NCT04987654"
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
      { "url": "https://clinicaltrials.gov/study/NCT05012345", "status": "completed", "data": [ { "nct_id": "NCT05012345", "status": "Recruiting", "enrollment": 240 } ] },
      { "url": "https://clinicaltrials.gov/study/NCT04987654", "status": "failed",    "error": "timeout" }
    ]
  }
}
```

Job-level `status` is `pending | processing | completed | failed | cancelled`. Per-URL `results[].status` lets you re-submit only those URLs that came back `failed` — don't re-run the whole batch and pay again for rows that already succeeded.

### 3b — batch-distill long protocols & labels for RAG

For the long-form documents (full protocol pages, full drug labels on DailyMed, recall enforcement reports), you want clean Markdown, not 20-credit structured extraction. Use **`/batch/distill`**.

```bash
# 1. Create the distill job (1 credit / URL)
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://dailymed.nlm.nih.gov/dailymed/drugInfo.cfm?setid=example-setid-1",
      "https://dailymed.nlm.nih.gov/dailymed/drugInfo.cfm?setid=example-setid-2"
    ],
    "timeout": 60000
  }'
# → { "data": { "id": "batch_dist01", "status": "pending", "total": 2 } }

# 2. Poll (paginated)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/distill/batch_dist01?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

Each completed result carries the page Markdown at `results[].data.markdown`. Pipe that into your chunk → embed → store flow from the [RAG tutorial](07-rag-knowledge-base.md), keeping the source URL and (for trials) the `nct_id` as citation metadata so every answer can point back to the authoritative record.

> **Why two jobs?** Extract is for the handful of fields you query and diff; distill is for the prose you search. Distilling 100 protocols costs **100 credits**; extracting them costs **2,000**. Use the cheap tool for text.

### CLI / MCP equivalents

```bash
# CLI: batch distill from a file of URLs (one per line; blank lines and # comments ignored)
thunderbit batch distill --file label_urls.txt --timeout 60000 -f json > labels.json
```

In MCP, ask: *"Use Thunderbit to batch-distill these label URLs, then poll until done."* The model calls `thunderbit_batch_distill_create` (args `urls`, `timeout?`) then `thunderbit_batch_distill_status` (args `jobId`, `page?`, `pageSize?`) until `completed`.

---

## Monitoring new trials & recalls

Monitoring is a **diff against a stored set**, keyed by a stable identifier — `nct_id` for trials, `recall_id` (or the FDA recall number) for device recalls. On each run you:

1. Extract the current search/results page into rows.
2. Compare each row's identifier against the most recent stored snapshot.
3. Emit two alert lists: **new** records (identifier never seen before) and **changed** records (a tracked field — most importantly `status` — differs from the prior snapshot).

For device recalls, a public FDA enforcement results page yields a similar schema:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "recall_id":        { "type": "TEXT", "instruction": "The recall / enforcement report number exactly as shown" },
      "product":          { "type": "TEXT", "instruction": "Recalled product / device name" },
      "firm":             { "type": "TEXT", "instruction": "Recalling firm / manufacturer name" },
      "classification":   { "type": "TEXT", "instruction": "Recall classification, e.g. Class I, Class II, Class III" },
      "reason":           { "type": "TEXT", "instruction": "Stated reason for the recall" },
      "status":           { "type": "TEXT", "instruction": "Recall status, e.g. Ongoing, Completed, Terminated" },
      "recall_date":      { "type": "DATE", "instruction": "Recall initiation or report date; ISO YYYY-MM-DD if shown" },
      "detail_url":       { "type": "URL",  "instruction": "Absolute link to this recall's detail page" }
    }
  }
}
```

The diff for recalls is the same shape as for trials — new `recall_id`s are *new recalls*, and a `classification` that changes from `Class II` to `Class I` is a high-priority alert. Keep the rule generic: store identifier + a few tracked fields, compare, alert.

> **Reminder:** for trial metadata the ClinicalTrials.gov API and for recalls the openFDA `/device/enforcement` endpoint return this data as structured JSON directly. If the official API covers your fields, diff *that* — it's free, faster, and kinder to the source. Reach for Thunderbit when a field you need is only on the rendered page, or the page has no API.

---

## End-to-end pipeline

Here's one cohesive, runnable Python script that ties it all together: extract the trial search page, batch-extract structured details (in chunks of 100), batch-distill the protocol text for the RAG corpus, write a SQLite snapshot, then diff against the previous run to surface **new trials** and **status changes**. Error handling follows the reference: retry `429`/`5xx` with exponential backoff + jitter, hard-stop on `402` (out of credits), never retry `401`.

```python
#!/usr/bin/env python3
"""Weekly clinical-trial monitor + protocol RAG feed built on the Thunderbit Open API.

PUBLIC, AGGREGATE REGULATORY DATA ONLY. Not medical advice. Prefer the official
ClinicalTrials.gov / openFDA APIs where they cover your fields; this is the scrape fallback.
"""
import os, time, json, random, sqlite3, datetime
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
SEARCH_URL = "https://clinicaltrials.gov/search?cond=non-small+cell+lung+cancer"  # public registry
DB = "trials.sqlite"

TRIAL_LIST_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "nct_id":          {"type": "TEXT",   "instruction": "NCT registry identifier, e.g. NCT01234567"},
        "title":           {"type": "TEXT",   "instruction": "Brief / public title of the trial"},
        "phase":           {"type": "TEXT",   "instruction": "Trial phase exactly as shown; Not Applicable if none"},
        "status":          {"type": "TEXT",   "instruction": "Recruitment status exactly as shown"},
        "condition":       {"type": "TEXT",   "instruction": "Condition(s) studied, comma-separated"},
        "intervention":    {"type": "TEXT",   "instruction": "Intervention(s), comma-separated"},
        "sponsor":         {"type": "TEXT",   "instruction": "Lead sponsor organization name"},
        "start_date":      {"type": "DATE",   "instruction": "Study start date; ISO YYYY-MM-DD if a full date is shown"},
        "completion_date": {"type": "DATE",   "instruction": "Primary/overall completion date; ISO YYYY-MM-DD if shown"},
        "enrollment":      {"type": "NUMBER", "instruction": "Enrollment count, digits only, no commas"},
        "detail_url":      {"type": "URL",    "instruction": "Absolute link to this trial's detail page"},
    }},
}

DETAIL_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "nct_id":           {"type": "TEXT",   "instruction": "NCT identifier on this page"},
        "status":           {"type": "TEXT",   "instruction": "Current overall recruitment status exactly as shown"},
        "enrollment":       {"type": "NUMBER", "instruction": "Enrollment count, digits only"},
        "completion_date":  {"type": "DATE",   "instruction": "Primary completion date; ISO YYYY-MM-DD if shown"},
        "last_update_date": {"type": "DATE",   "instruction": "'Last Update Posted' date; ISO YYYY-MM-DD"},
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

def extract_trial_list():
    res = _post("/openapi/v1/extract", {
        "url": SEARCH_URL, "schema": TRIAL_LIST_SCHEMA,
        "renderMode": "full", "waitFor": 3000, "timeout": 90000,
    })
    return res["data"]

def _poll_batch(kind, job_id):
    """kind is 'extract' or 'distill'. Returns the per-URL results list."""
    while True:
        status = _get(f"/openapi/v1/batch/{kind}/{job_id}",
                      {"page": 0, "pageSize": 100})["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            return status.get("results", [])
        time.sleep(3)  # polling is free

def batch_extract_details(urls):
    """Structured detail fields (20 credits/URL). Chunks of 100."""
    rows = []
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/extract",
                    {"urls": chunk, "schema": DETAIL_SCHEMA, "timeout": 90000})["data"]
        for result in _poll_batch("extract", job["id"]):
            if result.get("status") != "completed":
                print(f"  ! detail failed: {result['url']} ({result.get('error')})")
                continue
            payload = result.get("data") or []
            record = payload[0] if isinstance(payload, list) and payload else (payload or {})
            record["detail_url"] = result["url"]
            rows.append(record)
    return rows

def batch_distill_protocols(urls):
    """Long protocol/label text as Markdown (1 credit/URL) for the RAG corpus."""
    docs = []
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/distill",
                    {"urls": chunk, "timeout": 60000})["data"]
        for result in _poll_batch("distill", job["id"]):
            if result.get("status") != "completed":
                print(f"  ! distill failed: {result['url']} ({result.get('error')})")
                continue
            data = result.get("data") or {}
            md = data.get("markdown") if isinstance(data, dict) else None
            if md:
                docs.append({"url": result["url"], "markdown": md})
    return docs

def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS snapshots (
        run_date TEXT, nct_id TEXT, title TEXT, phase TEXT, status TEXT,
        condition TEXT, intervention TEXT, sponsor TEXT, start_date TEXT,
        completion_date TEXT, enrollment REAL, detail_url TEXT,
        PRIMARY KEY (run_date, nct_id))""")
    con.commit()
    return con

def previous_state(con):
    """Most recent prior run keyed by nct_id → (status, completion_date)."""
    cur = con.execute("SELECT nct_id, status, completion_date FROM snapshots "
                      "WHERE run_date = (SELECT MAX(run_date) FROM snapshots)")
    return {nct: {"status": st, "completion_date": cd} for nct, st, cd in cur.fetchall()}

def save_snapshot(con, run_date, rows):
    for r in rows:
        con.execute("INSERT OR REPLACE INTO snapshots VALUES (?,?,?,?,?,?,?,?,?,?,?,?)", (
            run_date, r.get("nct_id"), r.get("title"), r.get("phase"), r.get("status"),
            r.get("condition"), r.get("intervention"), r.get("sponsor"), r.get("start_date"),
            r.get("completion_date"), r.get("enrollment"), r.get("detail_url")))
    con.commit()

def diff(prev, rows):
    """New trials (unseen nct_id) and status/completion changes."""
    new_trials, changes = [], []
    for r in rows:
        nct = r.get("nct_id")
        if not nct:
            continue
        if nct not in prev:
            new_trials.append(r)
            continue
        before = prev[nct]
        if r.get("status") and r["status"] != before["status"]:
            changes.append({"nct_id": nct, "field": "status",
                            "old": before["status"], "new": r["status"],
                            "detail_url": r.get("detail_url")})
        if r.get("completion_date") and r["completion_date"] != before["completion_date"]:
            changes.append({"nct_id": nct, "field": "completion_date",
                            "old": before["completion_date"], "new": r["completion_date"],
                            "detail_url": r.get("detail_url")})
    return new_trials, changes

def main():
    run_date = datetime.date.today().isoformat()
    con = init_db()
    prev = previous_state(con)

    trials = extract_trial_list()
    detail_urls = [t["detail_url"] for t in trials if t.get("detail_url")]
    print(f"Found {len(trials)} trials on the search page")

    # Merge fresh structured detail fields back onto the list rows by nct_id.
    detail_rows = batch_extract_details(detail_urls)
    by_nct = {d.get("nct_id"): d for d in detail_rows if d.get("nct_id")}
    for t in trials:
        t.update({k: v for k, v in by_nct.get(t.get("nct_id"), {}).items() if v is not None})

    save_snapshot(con, run_date, trials)

    # Cheap distill of the long protocol pages → feed the RAG corpus (see tutorial 07).
    docs = batch_distill_protocols(detail_urls)
    with open(f"protocols_{run_date}.jsonl", "w") as f:
        for d in docs:
            f.write(json.dumps(d) + "\n")
    print(f"Distilled {len(docs)} protocol/label docs for RAG")

    new_trials, changes = diff(prev, trials)
    print(f"\n{len(new_trials)} NEW trials, {len(changes)} CHANGES")
    for t in new_trials:
        print(f"  + NEW {t['nct_id']}: {t.get('title')} [{t.get('status')}]")
    for c in changes:
        print(f"  ~ {c['nct_id']} {c['field']}: {c['old']} → {c['new']} ({c['detail_url']})")

if __name__ == "__main__":
    try:
        main()
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire this to your alerting (PagerDuty/Slack/email)
        raise
```

Run it, and on the second run it will print exactly which trials are new and which changed status or shifted their completion date since the previous snapshot — and it will have refreshed your protocol RAG corpus along the way.

---

## No-code version: the Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper** Chrome extension. Non-developer teammates — regulatory affairs, competitive intelligence, medical affairs — can run the identical workflow without writing code, and the schema they build transfers verbatim to the API.

1. **Install & sign in.** Add **Thunderbit** from the Chrome Web Store and create an account. A **free tier is available** (new accounts get a monthly page allowance) — note that an account is required; this is not an anonymous tool.
2. **AI Suggest Fields.** Open the public trial-results page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns (NCT ID, title, phase, status, condition, sponsor, dates, enrollment, detail link).
3. **Edit columns** in plain language. Rename, drop, or add columns; tune each column's instruction exactly like the `instruction` strings in the API schema (e.g. "return the date as ISO YYYY-MM-DD").
4. **Scrape**, then **Scrape Subpages** to follow each row's `detail_url` and pull the full attributes — the no-code equivalent of the two-stage list → detail batch pipeline in Steps 2–3.
5. **Export** to Excel, Google Sheets, Airtable, or Notion, or **schedule** recurring runs to power a living watchlist.

The recommended flow: design and validate the schema interactively in the extension, then port the field names and instructions to the CLI/API for automation at scale. They map one-to-one.

---

## Automate it

- **Cron.** Run the monitor weekly (trials and labels don't change minute-to-minute; weekly is plenty and is gentler on the source). The diff logic already compares against the most recent prior snapshot stored in SQLite:

  ```cron
  # Every Monday at 06:30, run the trial/recall monitor and append a log
  30 6 * * 1  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/pharma/monitor.py >> /var/log/pharma.log 2>&1
  ```

- **Batch webhooks instead of polling.** For server-side jobs, supply a callback so Thunderbit notifies you when the batch finishes — no open connection needed:

  ```json
  {
    "urls": ["https://clinicaltrials.gov/study/NCT05012345", "..."],
    "schema": { "type": "array", "items": { "type": "object", "properties": { } } },
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "whatever-signing-secret-you-choose"
    }
  }
  ```

  Thunderbit POSTs the completed job to `webhook.url`; verify the `secret` to trust the call, then write the results to your store.

- **Store snapshots.** Keep one row per `(run_date, nct_id)` (as the script does). That history powers new-trial alerts, status-transition timelines, and "trials whose completion date keeps slipping" reports.
- **Alert on what matters.** Pipe `new_trials` and `changes` to Slack/email. Useful triggers: a competitor's trial flips to *Recruiting*, a Phase 3 moves to *Completed*, a trial is *Terminated*, a device recall is upgraded to *Class I*, or a drug label adds/changes a boxed warning (detect by distilling the label and diffing the relevant section text).

---

## Cost & scale

Prices from the reference: **distill 1 credit**, **suggest_fields 1 credit** *(some docs list it as free — confirm against your plan)*, **extract 20 credits**. Batch scales per URL (distill 1/URL, extract 20/URL). **Polling is free.** New accounts get a one-time free allotment (~600 units) — enough to prototype before topping up at <https://thunderbit.com/billing>.

**The 20× decision — extract vs distill.** This is the single biggest cost lever in this domain, because trial protocols and drug labels are *long*:

```
100 full protocols, structured extract  = 100 × 20 = 2,000 credits
100 full protocols, distill to Markdown = 100 ×  1 =   100 credits   (20× cheaper)
```

Reserve **extract** for the handful of fields you actually query and diff (`status`, `enrollment`, dates). Use **distill** for everything you only read or feed to RAG (protocols, labels, recall narratives).

**Worked example — a weekly run over 300 trials:**

```
1 search-page extract            =    20 credits
300 detail extracts (fields)     = 6,000 credits   (or skip these via the official API)
300 protocol distills (RAG)      =   300 credits
+ 1 suggest_fields (one-time)    =     1 credit
≈ 6,321 credits / week
```

**Cut cost and load:**

- **Prefer official APIs.** ClinicalTrials.gov and openFDA return most metadata as free JSON. Pull `status`, dates, and enrollment from the API and spend Thunderbit credits only on what the API lacks. This can take the 6,000-credit detail-extract line to **zero**.
- **Dedupe by identifier.** Skip detail re-extraction for trials whose `last_update_date` is unchanged from the last snapshot — your SQLite history already knows them. Only re-extract what moved.
- **Stage cheaply.** The search-page extract is one call that yields many `detail_url`s; only process the details you don't already have fresh.
- **Throttle.** Batch caps at 100 URLs/job; chunk larger sets, add jitter to scheduled runs, and don't fire many jobs at one origin simultaneously.

---

## Responsible use

This domain is sensitive. These guardrails are mandatory, not optional.

- **PUBLIC, aggregate, non-personal regulatory data ONLY.** Target public registry and regulatory pages — trial records, drug labels, recall notices. **Never** collect patient-level data, identifiable individuals, free-text fields that could contain personal information, or any Protected Health Information (PHI). If a field could re-identify a person, drop it.
- **NOT medical advice.** Output from these pipelines is regulatory/competitive intelligence. It must **not** be used for diagnosis, treatment, dosing, or any clinical decision. Label your datasets and dashboards accordingly.
- **Prefer official APIs.** ClinicalTrials.gov and openFDA (and DailyMed's web services) publish free, structured, authoritative APIs. Use them first; scrape only public pages that have no API, and only for the fields the API omits.
- **HIPAA / GDPR awareness.** Aggregating public records is generally fine; assembling a dataset that re-identifies individuals is not. Don't combine sources to defeat de-identification. If you process any personal data, ensure you have a lawful basis (GDPR/CCPA) and the appropriate safeguards.
- **Check `robots.txt` and Terms of Service first.** Confirm each site permits automated access to the pages you target. Honor disallowed paths and crawl-rate directives. Public registries are designed to be public, but their ToS still governs *automated* access.
- **Throttle.** Use batch with reasonable concurrency; add jitter; don't hammer a single origin. If a source offers an API, using it *is* the polite, low-load choice.
- **Cache and dedupe** to avoid re-scraping unchanged pages — it saves credits and reduces load on public infrastructure that everyone shares.

---

## Troubleshooting & FAQ

**Extraction returns an empty array.** The registry UI is client-rendered. Set `renderMode:"full"` (headless browser) and raise `waitFor` (up to its `10000` ms ceiling) so the results list finishes loading. Also raise `timeout` (extract allows up to `120000` ms). `renderMode:"none"` against an SPA is the usual cause of an empty result.

**Dates come back inconsistent (e.g. "March 2024", "2024-03-01", "Mar 1, 2024").** Registries display dates in mixed formats; some are month-only. Tighten the `DATE` `instruction`: *"return ISO YYYY-MM-DD if a full date is shown; if only a month/year is shown, return YYYY-MM."* The `instruction` is what normalizes the value — brief it like a human assistant, then normalize again in your code before diffing.

**Pagination on the results page.** Most registry searches paginate via a query parameter (`?page=2`) or expose page-size controls. Enumerate the page URLs and feed them as a list to **batch extract** with the trial-list schema, then merge by `nct_id`. URL-based pagination is more reliable than scrolling.

**Duplicate trials across runs / across search pages.** Dedupe by `nct_id` (for recalls, by `recall_id`). It's the stable key — titles and sponsors can be reworded, but the registry identifier is canonical. Use it as the primary key in your snapshot table, exactly as the script does.

**Big protocol or label documents time out.** Full protocols and prescribing information are long. For a single very large page, prefer **distill** (it returns text far more cheaply and reliably than a 20-credit extract), raise `timeout` toward the max, and add `waitFor`. For one slow SPA page outside a batch, use the async endpoints (`POST /openapi/v1/async/distill` then poll `GET /openapi/v1/async/distill/{jobId}` every ~3s) so you don't hold an HTTP connection open.

**Some batch URLs fail while others succeed.** Inspect per-URL `results[].status`; re-submit only the `failed` URLs in a fresh batch job. Don't re-run the whole batch — you'd pay credits again for the rows that already succeeded.

**Geo-specific content.** Set `country_code` (HTTP/CLI) or `countryCode` (MCP) if a source serves region-specific pages. The default is `US`, which is right for ClinicalTrials.gov and US FDA sources.

**"Should I really be scraping this?"** If ClinicalTrials.gov or openFDA already returns the field via their free API, **use the API** — it's faster, free, authoritative, and lighter on shared public infrastructure. Reserve Thunderbit for public pages with no API, or for fields visible on the rendered page but missing from the structured feed.

**`401` / `402` / `429`.** `401` = bad or missing key (check `THUNDERBIT_API_KEY`; never retry). `402` = out of credits (hard stop, alert a human, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter and retry). See the [API reference](docs/thunderbit-api-reference.md) §4.
