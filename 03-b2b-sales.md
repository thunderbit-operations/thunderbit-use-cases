# B2B Sales: Building Lead Lists & Account Enrichment from Public Sources with Thunderbit

Turn fragmented public directories, company websites, and news feeds into a clean, enriched, CRM-ready account list — without ever touching a login-walled network.

---

## The data problem in B2B prospecting

Prospecting is mostly a data-plumbing problem disguised as a sales problem. The signals you need — who exists in your target market, what they do, where they're located, how to reach the right team, and whether *now* is the right moment — are spread across dozens of public surfaces: industry directories, marketplace listings, review sites, company websites, and news. None of them share a schema, none of them export cleanly, and all of them go stale.

So reps fall back on manual research: open a directory, copy a company name into a spreadsheet, visit the website, hunt for an email, paste it back, repeat 400 times. The result is a CRM full of half-filled rows that were already out of date the day they were entered.

This guide deliberately builds on **public sources only** — public directories, public company websites, and public news — and intentionally **does not** use login-walled networks like LinkedIn, Facebook, or Instagram. That's partly compliance (those platforms forbid scraping in their terms, and their data is frequently personal data), and partly durability: public, company-level data is more stable, more defensible, and doesn't break the moment a platform changes its auth wall. We favor **company-level firmographics** over personal data throughout, and we flag a lawful-basis caution wherever personal contact data appears.

Everything below maps to the canonical [API reference](docs/thunderbit-api-reference.md) — every endpoint, schema, and response shape comes from there. If a sample here ever disagrees with that file, the reference wins.

## What you'll build

- **A target-account list** scraped from a public business directory: company name, website, category, location, and size where public.
- **An enrichment layer** that visits each company's public website and pulls industry/tech signals, a public contact email (with a GDPR/CCPA caution), phone, and whether they have careers/pricing pages.
- **A trigger monitor** that watches public news for funding/expansion signals so you can reach out when timing is good.
- **One runnable end-to-end Python pipeline** that ties it together and writes a CSV ready for CRM import, with proper 401/402/429 handling.

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

New accounts get a one-time free allotment (~600 units) — enough to prototype before you top up. See the [API reference](docs/thunderbit-api-reference.md) for auth, pricing, and error semantics.

## Three ways to call Thunderbit

The same web-data engine is exposed through three surfaces that all hit the identical HTTP API, so you can mix them freely.

| Surface | Entry point | Best for in B2B sales |
|---------|-------------|------------------------|
| **MCP server** | `@thunderbit/mcp-server` (npx) | An AI assistant (Claude, Cursor, Cline) doing interactive research: "find me 50 accounts in this directory and enrich them." No code. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, CRM sync, webhooks. Any language. The production workhorse. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` (npm) | Local scripting and one-off list pulls from your shell; saving/reusing a schema. |

Rule of thumb: prototype the schema in MCP or the CLI, then automate at scale with the HTTP API.

## Step 1 — Discover the schema on a public directory

Don't guess field names. Point Thunderbit's `suggest_fields` at a directory listing page and let the AI propose a schema. We'll use a generic public directory listing as the example: `https://example-directory.com/agencies/marketing`.

### Via MCP (AI assistant)

In Claude Desktop / Cursor / Cline with the Thunderbit MCP server configured, just ask:

> "Use `thunderbit_suggest_fields` on `https://example-directory.com/agencies/marketing` with the prompt *'Each company listing card: company name, website link, category, city/region, company size'*."

The assistant calls the `thunderbit_suggest_fields` tool (`url`, `prompt`, `countryCode?`) and returns the proposed fields. (Costs 1 credit.)

### Via curl

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/suggest_fields \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example-directory.com/agencies/marketing",
    "prompt": "Each company listing card: company name, website link, category, city/region, and company size if shown",
    "country_code": "US"
  }'
```

A realistic response (fields live at `data`):

```json
{
  "data": [
    { "name": "company_name", "type": "TEXT", "instruction": "The business name shown on the listing card" },
    { "name": "website",      "type": "URL",  "instruction": "The company's external website link" },
    { "name": "category",     "type": "TEXT", "instruction": "Service category or industry label" },
    { "name": "location",     "type": "TEXT", "instruction": "City and region/state for the company" },
    { "name": "company_size", "type": "TEXT", "instruction": "Employee count or size range if shown, else empty" }
  ]
}
```

> Some deployments wrap the array as `data.fields`. Handle both: `fields = res["data"].get("fields", res["data"])` in Python, or `res.data?.fields ?? res.data` in JS.

### Via CLI

```bash
thunderbit suggest-fields "https://example-directory.com/agencies/marketing" \
  --prompt "Each company listing card: company name, website link, category, location, size" \
  --country-code US
```

## Step 2 — Extract the account list

Now turn the listing page into structured rows. The directory is a list of companies, so use an **array** schema. Use good `instruction` strings — they're your biggest quality lever.

### Via curl

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example-directory.com/agencies/marketing",
    "renderMode": "none",
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "company_name": { "type": "TEXT",  "instruction": "The business name on the listing card" },
          "website":      { "type": "URL",   "instruction": "The company external website link, absolute URL" },
          "category":     { "type": "TEXT",  "instruction": "Service category or industry label" },
          "location":     { "type": "TEXT",  "instruction": "City and region/state, e.g. Austin, TX" },
          "public_email": { "type": "EMAIL", "instruction": "Any publicly listed contact email; leave empty if none shown" },
          "phone":        { "type": "TEXT",  "instruction": "Public phone number, digits and separators as shown" }
        }
      }
    }
  }'
```

Response (rows at `data`):

```json
{
  "data": [
    { "company_name": "Northwind Marketing", "website": "https://northwind.example.com", "category": "B2B Marketing", "location": "Austin, TX", "public_email": "", "phone": "(512) 555-0142" },
    { "company_name": "Bluewave Digital",    "website": "https://bluewave.example.com",  "category": "Performance Marketing", "location": "Denver, CO", "public_email": "hello@bluewave.example.com", "phone": "" }
  ]
}
```

> **JS-heavy directories:** if the listing only renders after JavaScript runs (infinite scroll, client-side filters), set `"renderMode": "full"` to use a headless browser, and optionally `"waitFor": 2000` for lazy content. For static HTML, `"none"` is fastest and cheapest.

### Via Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

ACCOUNT_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "company_name": {"type": "TEXT",  "instruction": "The business name on the listing card"},
            "website":      {"type": "URL",   "instruction": "The company external website link, absolute URL"},
            "category":     {"type": "TEXT",  "instruction": "Service category or industry label"},
            "location":     {"type": "TEXT",  "instruction": "City and region/state"},
            "public_email": {"type": "EMAIL", "instruction": "Publicly listed contact email; empty if none shown"},
            "phone":        {"type": "TEXT",  "instruction": "Public phone number as shown"},
        },
    },
}

def extract(url, schema, render_mode="none", timeout=60000):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema, "renderMode": render_mode, "timeout": timeout,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

accounts = extract("https://example-directory.com/agencies/marketing", ACCOUNT_SCHEMA)
print(f"{len(accounts)} accounts")
```

### Via CLI (save the schema for reuse)

```bash
# Interactive: auto-suggest, edit fields, then save the schema
thunderbit extract "https://example-directory.com/agencies/marketing" -i \
  --save-schema accounts.schema.json

# Re-run later against any directory page with the saved schema, as a table
thunderbit extract "https://example-directory.com/agencies/saas" \
  --schema accounts.schema.json -f table
```

## Step 3 — Enrich each account

You now have company names and **website URLs**. The website is where the real firmographics live. There are two approaches; pick per field-set and budget.

### Approach A — Distill, then LLM-summarize (cheap & flexible)

`distill` returns the page as clean Markdown for **1 credit**. Feed that Markdown to your own LLM to pull whatever signals you want, with no fixed schema. Best when you want flexible, narrative firmographics or you're already running an LLM.

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://bluewave.example.com/about",
    "renderMode": "basic",
    "excludeTags": ["nav", "footer", "aside"],
    "timeout": 30000
  }'
```

The Markdown lands at `data.markdown`. Pass it to your LLM with a prompt like *"From this company About page, return JSON: industry, what they sell, target customers, HQ location, and any team-size hints."*

### Approach B — Extract with a fixed enrichment schema (structured)

When you want the same columns for every account, `extract` (20 credits) against a fixed schema is cleaner. Use a **single-object** schema since each company site is one record:

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://bluewave.example.com",
    "renderMode": "basic",
    "schema": {
      "type": "object",
      "properties": {
        "industry":      { "type": "TEXT",  "instruction": "Primary industry/vertical the company operates in" },
        "what_they_do":  { "type": "TEXT",  "instruction": "One-sentence description of the product or service" },
        "public_email":  { "type": "EMAIL", "instruction": "Public contact email if shown on the site; empty otherwise" },
        "phone":         { "type": "TEXT",  "instruction": "Public phone number if shown" },
        "has_careers":   { "type": "TEXT",  "instruction": "yes if a careers/jobs page or link exists, else no" },
        "has_pricing":   { "type": "TEXT",  "instruction": "yes if a pricing page or link exists, else no" }
      }
    }
  }'
```

### Batch versions (up to 100 URLs)

For real lists, batch the website enrichment. Distill batch is **1 credit/URL**; extract batch is **20 credits/URL**.

```bash
# Batch distill 100 company sites → Markdown
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "urls": ["https://northwind.example.com", "https://bluewave.example.com"], "timeout": 30000 }'
# → { "data": { "id": "batch_...", "status": "pending", "total": 2 } }

# Poll (0-based page; free)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/distill/batch_...?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

Swap `/batch/distill` for `/batch/extract` (add the `schema`) if you want structured rows instead.

## End-to-end pipeline

One cohesive script: scrape the directory → batch-distill each company site → LLM-free heuristic enrichment + a simple lead score → write a CRM-ready CSV. It handles `401`/`402`/`429` per the [API reference](docs/thunderbit-api-reference.md).

```python
import os, csv, time, random, requests
from urllib.parse import urlparse

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

DIRECTORY_URL = "https://example-directory.com/agencies/marketing"

ACCOUNT_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "company_name": {"type": "TEXT",  "instruction": "The business name on the listing card"},
        "website":      {"type": "URL",   "instruction": "The company external website link, absolute URL"},
        "category":     {"type": "TEXT",  "instruction": "Service category or industry label"},
        "location":     {"type": "TEXT",  "instruction": "City and region/state"},
        "public_email": {"type": "EMAIL", "instruction": "Publicly listed contact email; empty if none shown"},
        "phone":        {"type": "TEXT",  "instruction": "Public phone number as shown"},
    }},
}

class OutOfCredits(Exception): pass

def post(path, body, max_retries=5):
    """POST with exponential backoff. 402 is a hard stop; 401 never retries."""
    for attempt in range(max_retries):
        r = requests.post(f"{BASE}{path}", headers=HEADERS, json=body, timeout=130)
        if r.status_code == 401:
            raise SystemExit("401 Unauthorized — check THUNDERBIT_API_KEY.")
        if r.status_code == 402:
            raise OutOfCredits("402 — out of credits. Top up at thunderbit.com/billing.")
        if r.status_code == 429 or r.status_code >= 500:
            time.sleep((2 ** attempt) + random.random())
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"{path}: exhausted retries")

def get(path, params=None, max_retries=5):
    for attempt in range(max_retries):
        r = requests.get(f"{BASE}{path}", headers=HEADERS, params=params, timeout=60)
        if r.status_code == 401:
            raise SystemExit("401 Unauthorized — check THUNDERBIT_API_KEY.")
        if r.status_code == 429 or r.status_code >= 500:
            time.sleep((2 ** attempt) + random.random())
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"{path}: exhausted retries")

def extract(url, schema, render_mode="none"):
    return post("/openapi/v1/extract", {"url": url, "schema": schema, "renderMode": render_mode})["data"]

def batch_distill(urls, poll_every=5):
    job = post("/openapi/v1/batch/distill", {"urls": urls, "timeout": 30000})["data"]
    job_id = job["id"]
    while True:
        status = get(f"/openapi/v1/batch/distill/{job_id}", {"page": 0, "pageSize": 100})["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            return status
        time.sleep(poll_every)

def domain(url):
    try:
        return urlparse(url).netloc.lower().lstrip("www.")
    except Exception:
        return ""

def enrich_from_markdown(md):
    """Cheap heuristic enrichment off distilled Markdown — swap in your LLM for richer signals."""
    low = md.lower()
    return {
        "has_careers": "yes" if any(k in low for k in ("careers", "we're hiring", "join our team", "open roles")) else "no",
        "has_pricing": "yes" if "pricing" in low or "/pricing" in low else "no",
        "tech_signals": ", ".join(t for t in ("shopify", "hubspot", "salesforce", "stripe", "react") if t in low),
    }

def score(acct):
    s = 0
    if acct.get("public_email"): s += 30          # reachable
    if acct.get("phone"):        s += 10
    if acct.get("has_pricing") == "yes": s += 20   # self-serve / mature
    if acct.get("has_careers") == "yes": s += 20   # hiring = growth
    if acct.get("tech_signals"):         s += 20   # fit signal
    return s

# 1. Target-account list from the directory (renderMode="full" for JS directories)
accounts = extract(DIRECTORY_URL, ACCOUNT_SCHEMA, render_mode="none")

# 2. Dedupe by domain, keep ones with a website
seen, deduped = set(), []
for a in accounts:
    d = domain(a.get("website", ""))
    if not d or d in seen:
        continue
    seen.add(d)
    a["domain"] = d
    deduped.append(a)

# 3. Batch-enrich the company sites (chunks of 100)
md_by_url = {}
urls = [a["website"] for a in deduped]
for i in range(0, len(urls), 100):
    chunk = urls[i:i + 100]
    result = batch_distill(chunk)
    for row in result.get("results", []):
        if row.get("status") == "completed":
            md_by_url[row["url"]] = row.get("data", {}).get("markdown", "")

# 4. Merge enrichment + score
for a in deduped:
    a.update(enrich_from_markdown(md_by_url.get(a["website"], "")))
    a["lead_score"] = score(a)

deduped.sort(key=lambda a: a["lead_score"], reverse=True)

# 5. Write CRM-ready CSV
cols = ["company_name", "domain", "website", "category", "location",
        "public_email", "phone", "has_careers", "has_pricing", "tech_signals", "lead_score"]
with open("accounts.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=cols, extrasaction="ignore")
    w.writeheader()
    w.writerows(deduped)

print(f"Wrote {len(deduped)} accounts to accounts.csv")
```

This uses **distill (1 credit/URL)** for enrichment — the cheap path. Swap `batch_distill` for `batch_extract` with an enrichment schema only when you need strictly structured fields for every site.

## Trigger monitoring

A clean list is half the job; *timing* is the other half. Funding rounds, expansions, and new-office announcements are the moments outreach lands. Watch public news with `distill`.

For each priority account, distill a public news search result page (e.g. Google News for `"<company> funding"`), then hand the Markdown to an LLM to detect a fresh trigger.

```python
def funding_signal(company):
    q = requests.utils.quote(f'"{company}" funding OR raises OR Series')
    news_url = f"https://news.google.com/search?q={q}&hl=en-US&gl=US"
    md = post("/openapi/v1/distill", {
        "url": news_url, "renderMode": "full", "waitFor": 2000, "timeout": 30000,
    })["data"]["markdown"]
    # Pass `md` to your LLM: "Is there a funding/expansion announcement in the
    # last 90 days? Return {trigger: bool, headline, date, amount}."
    return md

for a in deduped[:20]:          # top-scored accounts only — saves credits
    a["news_markdown"] = funding_signal(a["company_name"])
```

The pattern is always **distill → LLM**: distill turns a noisy news page into clean text for 1 credit, and your LLM does the timely-trigger judgment. Run it on a schedule (below) so triggers reach reps while they're fresh.

## No-code version: Thunderbit Chrome extension

Your non-technical teammates can run this exact workflow in the browser — the field names and instructions transfer directly to the API later.

1. Install the **Thunderbit** AI Web Scraper from the Chrome Web Store and **sign in** (a free tier is available — new accounts get a monthly page allowance; this is not a no-sign-up tool).
2. Open the public directory page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. It proposes columns like company name, website, category, location.
3. Edit columns in plain language (add `public_email`, `phone`), then click **Scrape** to capture the listing.
4. Use **Scrape Subpages** to follow each row's website link and pull enrichment fields from the company site — the no-code version of the list → detail pipeline above.
5. Export to **Excel, Google Sheets, Airtable, or Notion**. To land in HubSpot, export to Google Sheets and use HubSpot's Sheets import.

Design and validate the schema here interactively, then port it to the API/CLI for scale.

## Automate it

- **Cron.** Wrap the pipeline in a daily/weekly cron job. Re-run the directory extract to catch new listings; re-distill news for top accounts.

```bash
# Every Monday 07:00 — refresh the account list and triggers
0 7 * * 1 cd /opt/prospecting && THUNDERBIT_API_KEY=tb_... python pipeline.py >> run.log 2>&1
```

- **Batch webhooks.** For large lists, skip polling — supply a callback and let Thunderbit notify you when the batch finishes:

```json
{
  "urls": ["https://northwind.example.com", "..."],
  "webhook": {
    "url": "https://your-server.example.com/thunderbit/callback",
    "secret": "your-signing-secret"
  }
}
```

  Verify the `secret` on the incoming POST before trusting it, then upsert into your CRM.

- **CRM sync + dedupe by domain.** Always dedupe on normalized **domain** (not company name — names collide and vary). Upsert so re-runs update existing accounts instead of creating duplicates.

## Cost & scale

Credit math drives every design decision here. For **1,000 accounts**:

| Stage | Endpoint | Credits |
|-------|----------|---------|
| Directory list extract (a few listing pages) | `extract` ×~10 pages | ~200 |
| Enrich via **distill** (1/URL) | `batch/distill` ×1,000 | **1,000** |
| Enrich via **extract** (20/URL) | `batch/extract` ×1,000 | **20,000** |
| News triggers (top 100, distill) | `distill` ×100 | 100 |
| Status polling | — | free |

Enriching via distill is **20× cheaper** than extract. So: use **extract** for the directory list (you need clean structured rows once) and the cheap **distill → LLM** path for per-site enrichment, reserving **extract** for the handful of fields you truly need as strict columns across the whole list. Distill beats extract whenever you'll post-process the text with your own LLM anyway, or when site layouts vary too much for one fixed schema.

## Responsible use

- **Public sources only.** This guide uses public directories, public company websites, and public news. Thunderbit targets publicly accessible pages.
- **No walled gardens.** Do **not** scrape LinkedIn (profiles or company pages), Facebook, Instagram, or any login-walled / authenticated area. Their terms forbid it and the data is often personal. This is a hard line, not a preference.
- **Prefer company-level data.** Firmographics (industry, location, size, tech signals) over individuals. When personal contact data appears — e.g. a public "contact us" email or a named person's address — you need a **lawful basis** to process it under **GDPR** (legitimate interest, documented and balanced) and to honor **CCPA** opt-outs/disclosure. Don't collect personal data you can't justify.
- **Respect `robots.txt` and Terms of Service** for every site you target.
- **Throttle.** Use batch with reasonable concurrency; don't hammer a single origin. Cache and dedupe so you never re-scrape unchanged pages.

## Troubleshooting & FAQ

**Many sites don't show an email — why are `public_email` fields empty?**
That's expected and correct. Lots of companies deliberately hide email behind contact forms. Leave the field empty; don't try to defeat obfuscation. An empty `public_email` is a respectful, honest result — lean on phone or the contact page instead.

**The directory shows nothing / partial rows.**
It's likely a JS-rendered SPA. Set `"renderMode": "full"` (headless browser) and add `"waitFor": 2000` for lazy content. Static HTML works fine with `"none"`.

**How do I handle pagination?**
Most directories paginate via `?page=2` or `/page/2`. Loop over the page URLs and call `extract` per page (or feed them all into `batch/extract`), then concatenate and dedupe. Stop when a page returns zero rows.

**I'm getting duplicate accounts.**
Dedupe on normalized **domain** before enrichment (strip `www.`, lowercase). Company names are unreliable keys — domains are unique. The pipeline above does this in step 2.

**A few sites failed in a batch.**
Batch results carry per-URL `results[].status`. Filter for `"failed"` and retry just those URLs (often with a higher `timeout` or `renderMode:"full"`), instead of re-running the whole batch.

**401 / 402 / 429?**
`401` = bad/missing key — fix `THUNDERBIT_API_KEY`, never retry. `402` = out of credits — hard stop, top up. `429` = rate-limited — exponential backoff with jitter. All handled in the pipeline above and documented in the [API reference](docs/thunderbit-api-reference.md).
