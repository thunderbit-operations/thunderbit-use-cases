# Recruiting & HR: Job-Market Intelligence, Competitor Hiring Monitors & Compliant Sourcing with Thunderbit

Turn thousands of scattered public job postings into a clean, queryable dataset — track demand, salaries, and skills, watch a competitor's hiring, and source talent from public portfolios, all without ever touching a login-walled network.

---

## The data problem in recruiting

Recruiting and talent-market work is, underneath, a data problem. The signals you need — which roles are in demand, what they pay, which skills employers ask for, where competitors are expanding, and who's available — are smeared across dozens of public surfaces that share no schema and export nothing cleanly.

A single role might appear on a company's own `/careers` page, a Greenhouse- or Lever-hosted board, Indeed's public listings, and Wellfound, each with a different layout. Salary data is opaque: some postings publish a range, some bury it in the description, many omit it entirely. Skills live in unstructured prose. So recruiters and people-analytics teams fall back on manual market mapping — open a board, copy a title into a spreadsheet, eyeball the salary, paste it back, repeat hundreds of times. The result is stale before it's finished.

This guide deliberately builds on **public, no-login sources only**: public job boards and company career pages (Indeed public listings, public company `/careers` pages, public Greenhouse/Lever-hosted boards, public Wellfound listings), public talent-market data (public salary pages, public posting aggregators), public employer-reputation pages (Glassdoor public pages, Trustpilot), and public professional portfolios that people publish specifically for discovery (public GitHub profiles, personal portfolio sites).

It intentionally **does not** use LinkedIn, Facebook, or Instagram, and does **not** scrape candidate profiles behind logins. That's partly compliance — those platforms forbid scraping in their terms and their data is overwhelmingly personal data — and partly durability: public, **job-market and company-level** data is more stable, more defensible, and doesn't break the moment a platform changes its auth wall. We favor company- and market-level data throughout, and wherever any candidate personal data appears we **minimize it** and flag a GDPR/CCPA lawful-basis caution.

Everything below maps to the canonical [API reference](docs/thunderbit-api-reference.md) — every endpoint, schema, and response shape comes from there. If a sample here ever disagrees with that file, the reference wins.

## What you'll build

- **A job-market dataset** scraped across public boards: `job_title`, `company`, `location`, `salary_range`, `posted_date`, `employment_type`, `skills`, and `job_url`, enriched with detail-page fields, ready for demand/salary/skills trend analysis.
- **A competitor hiring monitor** that snapshots a competitor's public `/careers` page daily and diffs it to detect new and closed roles — an early signal of expansion, pivots, and team-building.
- **A public-portfolio sourcing list** (e.g. public GitHub profiles matching a skill) — wrapped in a strong lawful-basis caution and built to collect the *minimum* personal data.
- **One runnable end-to-end Python pipeline** that ties it together, writes to SQLite + CSV, aggregates salary and skills trends, and handles 401/402/429 correctly.

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

4. For the Python pipeline: `pip install requests` (SQLite ships with Python's stdlib).

New accounts get a one-time free allotment (~600 units) — enough to prototype before you top up. See the [API reference](docs/thunderbit-api-reference.md) for auth, pricing, and error semantics.

## Three ways to call Thunderbit

The same web-data engine is exposed through three surfaces that all hit the identical HTTP API, so you can mix them freely.

| Surface | Entry point | Best for in recruiting/HR |
|---------|-------------|----------------------------|
| **MCP server** | `@thunderbit/mcp-server` (npx) | An AI assistant (Claude, Cursor, Cline) doing interactive market research: "pull every open role from this board and give me a salary table." No code. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, daily competitor monitors, ATS/Sheets sync, webhooks. Any language. The production workhorse. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` (npm) | Local scripting and one-off board pulls from your shell; saving and reusing a schema. |

Rule of thumb: prototype the schema in MCP or the CLI, then automate at scale with the HTTP API.

## Step 1 — Discover the schema on a public job board

Don't hand-write field names. Point Thunderbit's `suggest_fields` at a public board search-results page and let the AI propose a schema. We'll use a generic public board search page as the example: `https://example-jobs.com/search?q=data+engineer&l=remote`.

### Via MCP (AI assistant)

In Claude Desktop / Cursor / Cline with the Thunderbit MCP server configured, just ask:

> "Use `thunderbit_suggest_fields` on `https://example-jobs.com/search?q=data+engineer&l=remote` with the prompt *'Each job posting card on the results page: job title, company, location, salary range if shown, posted date, employment type, and the link to the posting'*."

The assistant calls `thunderbit_suggest_fields` (`url`, `prompt`, `countryCode?`) and returns the proposed fields. Costs 1 credit.

### Via curl

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/suggest_fields \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example-jobs.com/search?q=data+engineer&l=remote",
    "prompt": "Each job posting card on the results page: job title, company, location, salary range if shown, posted date, employment type, and the link to the posting",
    "country_code": "US"
  }'
```

A realistic response (fields live at `data`):

```json
{
  "data": [
    { "name": "job_title",       "type": "TEXT", "instruction": "The job title shown on the posting card" },
    { "name": "company",         "type": "TEXT", "instruction": "The hiring company name" },
    { "name": "location",        "type": "TEXT", "instruction": "City/region or 'Remote' as shown" },
    { "name": "salary_range",    "type": "TEXT", "instruction": "Salary range text if shown, else empty" },
    { "name": "posted_date",     "type": "DATE", "instruction": "When the posting was published" },
    { "name": "employment_type", "type": "TEXT", "instruction": "Full-time, part-time, contract, etc." },
    { "name": "job_url",         "type": "URL",  "instruction": "Link to the full job posting detail page" }
  ]
}
```

> Some deployments wrap the array as `data.fields` instead of `data`. Always handle both: `const fields = res.data?.fields ?? res.data;`

### Via CLI

```bash
thunderbit suggest-fields "https://example-jobs.com/search?q=data+engineer&l=remote" \
  --prompt "Each job posting card: title, company, location, salary range, posted date, employment type, link" \
  --country-code US
```

Use whichever surface fits your workflow — they return the same field descriptors. Treat the suggestion as a starting point and tighten the `instruction` strings (especially `salary_range`) before you extract.

## Step 2 — Extract job postings

Now turn the schema into a full extraction over the board's results page. The schema is the Thunderbit-flavored array-of-objects shape (see §5 of the [API reference](docs/thunderbit-api-reference.md)). Note the UPPERCASE field types and the deliberately specific `instruction` strings.

### Via curl

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example-jobs.com/search?q=data+engineer&l=remote",
    "renderMode": "full",
    "waitFor": 2000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "job_title":       { "type": "TEXT", "instruction": "The job title exactly as posted" },
          "company":         { "type": "TEXT", "instruction": "The hiring company name" },
          "location":        { "type": "TEXT", "instruction": "City/region, or Remote if remote" },
          "salary_range":    { "type": "TEXT", "instruction": "Salary range text as shown, e.g. \"$120k-$150k\"; empty string if none" },
          "posted_date":     { "type": "DATE", "instruction": "Date the posting was published" },
          "employment_type": { "type": "TEXT", "instruction": "Employment type: Full-time, Part-time, Contract, Internship" },
          "job_url":         { "type": "URL",  "instruction": "Absolute URL to the full posting detail page" }
        }
      }
    }
  }'
```

Response — rows at `data`, one object per posting:

```json
{
  "data": [
    {
      "job_title": "Senior Data Engineer",
      "company": "Acme Analytics",
      "location": "Remote (US)",
      "salary_range": "$150k-$185k",
      "posted_date": "2026-05-24",
      "employment_type": "Full-time",
      "job_url": "https://example-jobs.com/job/acme-senior-data-engineer-88213"
    },
    {
      "job_title": "Data Engineer II",
      "company": "Northwind Labs",
      "location": "Austin, TX",
      "salary_range": "",
      "posted_date": "2026-05-21",
      "employment_type": "Full-time",
      "job_url": "https://example-jobs.com/job/northwind-data-engineer-ii-77104"
    }
  ]
}
```

> **JS-heavy boards.** Many modern boards (Greenhouse/Lever embeds, infinite-scroll aggregators) render postings client-side. Set `renderMode: "full"` to run a headless browser, and add `waitFor` (0–10000 ms) so lazily-loaded cards appear before extraction. For a static server-rendered page, `renderMode: "none"` is faster and cheaper to render. Extract costs 20 credits regardless of render mode.

### Via Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

POSTING_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "job_title":       {"type": "TEXT", "instruction": "The job title exactly as posted"},
            "company":         {"type": "TEXT", "instruction": "The hiring company name"},
            "location":        {"type": "TEXT", "instruction": "City/region, or Remote if remote"},
            "salary_range":    {"type": "TEXT", "instruction": 'Salary range text as shown, e.g. "$120k-$150k"; empty if none'},
            "posted_date":     {"type": "DATE", "instruction": "Date the posting was published"},
            "employment_type": {"type": "TEXT", "instruction": "Full-time, Part-time, Contract, or Internship"},
            "job_url":         {"type": "URL",  "instruction": "Absolute URL to the full posting detail page"},
        },
    },
}

def extract(url, schema, render_mode="full", wait_for=2000, timeout=90000):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema, "renderMode": render_mode,
        "waitFor": wait_for, "timeout": timeout,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

rows = extract("https://example-jobs.com/search?q=data+engineer&l=remote", POSTING_SCHEMA)
print(f"{len(rows)} postings")
```

### Via CLI

```bash
# Save the schema once (interactive editor), then reuse it
thunderbit extract "https://example-jobs.com/search?q=data+engineer&l=remote" \
  -i --save-schema postings.schema.json --render-mode full

# Re-run on another board/search with the saved schema, print a table
thunderbit extract "https://example-jobs.com/search?q=ml+engineer&l=remote" \
  --schema postings.schema.json --render-mode full -f table
```

## Step 3 — Batch-extract posting detail pages

The results page gives you titles, companies, and `job_url`s — but the *skills*, *seniority*, and *full description* live on each posting's detail page. Collect the `job_url` values from Step 2 and fan them out with `/batch/extract` (up to 100 URLs per job).

Define a detail schema with a flat object shape (one record per detail page):

```python
DETAIL_SCHEMA = {
    "type": "object",
    "properties": {
        "full_description": {"type": "TEXT",   "instruction": "The complete job description text"},
        "seniority":        {"type": "TEXT",   "instruction": "Seniority level: Junior, Mid, Senior, Staff, Lead, etc."},
        "skills":           {"type": "TEXT",   "instruction": "Comma-separated required/preferred skills and technologies"},
        "salary_range":     {"type": "TEXT",   "instruction": 'Salary range if stated in the body, e.g. "$120k-$150k"; empty if none'},
        "remote_policy":    {"type": "TEXT",   "instruction": "Remote, Hybrid, or On-site"},
        "apply_url":        {"type": "URL",    "instruction": "The direct apply link if present"},
    },
}
```

Submit the batch and poll, retrying only the per-URL failures:

```python
import time

def batch_extract(urls, schema, render_mode="full", poll_every=5):
    job = requests.post(f"{BASE}/openapi/v1/batch/extract", headers=HEADERS, json={
        "urls": urls, "schema": schema, "renderMode": render_mode, "timeout": 90000,
    }).json()["data"]
    job_id = job["id"]
    while True:
        status = requests.get(
            f"{BASE}/openapi/v1/batch/extract/{job_id}",
            headers=HEADERS, params={"page": 0, "pageSize": 100},
        ).json()["data"]
        if status["status"] in ("completed", "failed", "cancelled"):
            return status
        time.sleep(poll_every)

job_urls = [r["job_url"] for r in rows if r.get("job_url")][:100]
result = batch_extract(job_urls, DETAIL_SCHEMA)

# Retry only the failures (e.g. transient timeouts)
failed = [r["url"] for r in result["results"] if r["status"] != "completed"]
if failed:
    retry = batch_extract(failed, DETAIL_SCHEMA)
```

Each entry in `results` carries its own `status` and either `data` (the extracted object) or `error`. Per the [API reference](docs/thunderbit-api-reference.md), `page` is **0-based** and `pageSize` maxes at 100. Batch extract costs 20 credits per URL — see *Cost & scale* below for when to substitute the 20×-cheaper distill instead.

## Competitor hiring monitor

A competitor's public `/careers` page is one of the loudest open signals in the market. New roles reveal where they're investing; a cluster of sales roles signals a go-to-market push, a wave of infra roles signals a scaling effort, and roles that quietly disappear can signal a freeze or a pivot. You don't need anyone's profile — just the public listings.

The pattern is: extract the careers page once a day, store a snapshot keyed by `job_url`, and diff today's set against yesterday's.

```python
import json, sqlite3, datetime

CAREERS_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "job_title": {"type": "TEXT", "instruction": "The open role title"},
            "team":      {"type": "TEXT", "instruction": "Department/team if shown (Engineering, Sales, etc.)"},
            "location":  {"type": "TEXT", "instruction": "Location or Remote"},
            "job_url":   {"type": "URL",  "instruction": "Absolute link to the role detail page"},
        },
    },
}

def snapshot_careers(db, company, url):
    roles = extract(url, CAREERS_SCHEMA, render_mode="full", wait_for=2500)
    today = datetime.date.today().isoformat()
    con = sqlite3.connect(db)
    con.execute("""CREATE TABLE IF NOT EXISTS careers_snapshot
                   (company TEXT, snapshot_date TEXT, job_url TEXT,
                    job_title TEXT, team TEXT, location TEXT,
                    PRIMARY KEY (company, snapshot_date, job_url))""")
    for r in roles:
        if not r.get("job_url"):
            continue
        con.execute("INSERT OR REPLACE INTO careers_snapshot VALUES (?,?,?,?,?,?)",
                    (company, today, r["job_url"], r.get("job_title"),
                     r.get("team"), r.get("location")))
    con.commit(); con.close()
    return today

def diff_careers(db, company, day_a, day_b):
    """Roles new in day_b and roles closed since day_a, keyed by job_url."""
    con = sqlite3.connect(db); con.row_factory = sqlite3.Row
    def urls(day):
        rows = con.execute(
            "SELECT job_url, job_title FROM careers_snapshot WHERE company=? AND snapshot_date=?",
            (company, day)).fetchall()
        return {r["job_url"]: r["job_title"] for r in rows}
    a, b = urls(day_a), urls(day_b)
    con.close()
    opened = {u: t for u, t in b.items() if u not in a}
    closed = {u: t for u, t in a.items() if u not in b}
    return {"opened": opened, "closed": closed}
```

Run `snapshot_careers` daily on a cron and call `diff_careers(db, company, yesterday, today)`. Diffing **by `job_url`** (a stable identifier) avoids false positives from re-ordered listings or minor title-text changes. Pipe `opened`/`closed` to Slack or email for a daily hiring-signal digest.

## Talent sourcing from public portfolios (carefully)

> **Read this before you build it.** This section involves *personal data*, which raises real legal and ethical obligations. Most professional-network sourcing (LinkedIn et al.) is **off-limits** — it's login-walled, the terms forbid scraping, and the data is personal. The narrow, defensible alternative is data people **publish themselves for discovery**: public GitHub profiles, public personal portfolio sites, public conference-speaker pages. Even then, processing personal data needs a **lawful basis** under GDPR (legitimate interest, properly assessed and documented) or compliance with CCPA, plus data-minimization and the ability to honor deletion requests. **Collect the minimum, store the minimum, and keep a record of why.** When in doubt, talk to counsel — not to a scraper.

With that framing, you can extract a *public directory* of profiles that match a skill — for example, a public "GitHub topics" or community-curated index page — to build a top-of-funnel signal list. Keep the schema deliberately thin: a public handle, the public profile URL, a short public bio, and public skill/repo signals. Do **not** harvest emails, phone numbers, or anything not voluntarily published for professional discovery.

```python
PORTFOLIO_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "handle":      {"type": "TEXT", "instruction": "The person's public username/handle"},
            "profile_url": {"type": "URL",  "instruction": "Link to their public profile page"},
            "public_bio":  {"type": "TEXT", "instruction": "The short public bio/headline as written; empty if none"},
            "skills":      {"type": "TEXT", "instruction": "Public listed languages/skills/topics, comma-separated"},
            "location":    {"type": "TEXT", "instruction": "Public location field if the person published one; empty if absent"},
        },
    },
}

sourcing = extract(
    "https://example-devcommunity.com/topics/rust?sort=stars",
    PORTFOLIO_SCHEMA, render_mode="full",
)
# Minimization: drop any row where the person did not publish a bio/skills signal,
# and never persist fields you don't have a documented lawful basis to keep.
sourcing = [r for r in sourcing if r.get("public_bio") or r.get("skills")]
```

Treat the output as a *research signal*, not a contact list. Reach out through the person's own published channels, disclose how you found them, and delete on request. See *Responsible use* below for the full guardrails — including a fairness note on automated sourcing.

## End-to-end pipeline

One cohesive script: search a board → extract postings → batch-extract details → write to SQLite + CSV → aggregate salary and skills trends → run the competitor careers diff. It implements the 401/402/429 handling from §4 of the [API reference](docs/thunderbit-api-reference.md).

```python
#!/usr/bin/env python3
"""Job-market intelligence + competitor hiring monitor on PUBLIC sources only."""
import os, re, csv, time, json, sqlite3, random, datetime, collections, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
DB = "jobmarket.db"

# ---- HTTP with backoff (401 hard stop, 402 hard stop, 429/5xx retry) ----
class CreditsExhausted(Exception): ...
class AuthError(Exception): ...

def call(method, path, **kw):
    for attempt in range(6):
        r = requests.request(method, f"{BASE}{path}", headers=HEADERS, timeout=140, **kw)
        if r.status_code == 401:
            raise AuthError("401: check THUNDERBIT_API_KEY")          # never retry
        if r.status_code == 402:
            raise CreditsExhausted("402: top up at thunderbit.com/billing")  # hard stop
        if r.status_code == 429 or r.status_code >= 500:
            wait = min(2 ** attempt + random.random(), 30)            # backoff + jitter
            time.sleep(wait); continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"giving up after retries: {method} {path}")

def extract(url, schema, render_mode="full", wait_for=2000, timeout=90000):
    return call("POST", "/openapi/v1/extract", json={
        "url": url, "schema": schema, "renderMode": render_mode,
        "waitFor": wait_for, "timeout": timeout})["data"]

def batch_extract(urls, schema, render_mode="full", poll_every=5):
    job = call("POST", "/openapi/v1/batch/extract",
               json={"urls": urls, "schema": schema, "renderMode": render_mode,
                     "timeout": 90000})["data"]
    while True:
        st = call("GET", f"/openapi/v1/batch/extract/{job['id']}",
                  params={"page": 0, "pageSize": 100})["data"]
        if st["status"] in ("completed", "failed", "cancelled"):
            return st
        time.sleep(poll_every)

# ---- Schemas ----
POSTING_SCHEMA = {"type": "array", "items": {"type": "object", "properties": {
    "job_title":       {"type": "TEXT", "instruction": "Job title exactly as posted"},
    "company":         {"type": "TEXT", "instruction": "Hiring company name"},
    "location":        {"type": "TEXT", "instruction": "City/region, or Remote"},
    "salary_range":    {"type": "TEXT", "instruction": 'Salary range as shown, e.g. "$120k-$150k"; empty if none'},
    "posted_date":     {"type": "DATE", "instruction": "Date the posting was published"},
    "employment_type": {"type": "TEXT", "instruction": "Full-time, Part-time, Contract, Internship"},
    "job_url":         {"type": "URL",  "instruction": "Absolute URL to the posting detail page"},
}}}
DETAIL_SCHEMA = {"type": "object", "properties": {
    "skills":     {"type": "TEXT", "instruction": "Comma-separated required/preferred skills and technologies"},
    "seniority":  {"type": "TEXT", "instruction": "Seniority: Junior, Mid, Senior, Staff, Lead"},
    "salary_min": {"type": "NUMBER", "instruction": "Lower bound of annual salary in USD, digits only; empty if none"},
    "salary_max": {"type": "NUMBER", "instruction": "Upper bound of annual salary in USD, digits only; empty if none"},
}}

# ---- Storage ----
def init_db():
    con = sqlite3.connect(DB)
    con.execute("""CREATE TABLE IF NOT EXISTS postings (
        job_url TEXT PRIMARY KEY, job_title TEXT, company TEXT, location TEXT,
        salary_range TEXT, posted_date TEXT, employment_type TEXT,
        skills TEXT, seniority TEXT, salary_min REAL, salary_max REAL,
        fetched_at TEXT)""")
    con.commit(); con.close()

def upsert(rows):
    con = sqlite3.connect(DB)
    now = datetime.datetime.utcnow().isoformat()
    for r in rows:                                   # dedupe by job_url (PK)
        con.execute("""INSERT INTO postings VALUES
            (:job_url,:job_title,:company,:location,:salary_range,:posted_date,
             :employment_type,:skills,:seniority,:salary_min,:salary_max,:fetched_at)
            ON CONFLICT(job_url) DO UPDATE SET
              salary_range=excluded.salary_range, skills=excluded.skills,
              seniority=excluded.seniority, salary_min=excluded.salary_min,
              salary_max=excluded.salary_max, fetched_at=excluded.fetched_at""",
            {**{k: r.get(k) for k in
                ("job_url","job_title","company","location","salary_range",
                 "posted_date","employment_type","skills","seniority",
                 "salary_min","salary_max")}, "fetched_at": now})
    con.commit(); con.close()

# ---- Trends ----
def trends():
    con = sqlite3.connect(DB); con.row_factory = sqlite3.Row
    rows = con.execute("SELECT * FROM postings").fetchall(); con.close()
    sal = [(r["salary_min"], r["salary_max"]) for r in rows
           if r["salary_min"] and r["salary_max"]]
    skills = collections.Counter()
    for r in rows:
        for s in (r["skills"] or "").split(","):
            s = s.strip().lower()
            if s: skills[s] += 1
    median_mid = None
    if sal:
        mids = sorted((lo + hi) / 2 for lo, hi in sal)
        median_mid = mids[len(mids) // 2]
    return {
        "total_postings": len(rows),
        "with_salary": len(sal),
        "median_midpoint_salary": median_mid,
        "top_skills": skills.most_common(15),
    }

def export_csv(path="postings.csv"):
    con = sqlite3.connect(DB); con.row_factory = sqlite3.Row
    rows = con.execute("SELECT * FROM postings").fetchall(); con.close()
    if not rows: return
    with open(path, "w", newline="") as f:
        w = csv.DictWriter(f, fieldnames=rows[0].keys()); w.writeheader()
        for r in rows: w.writerow(dict(r))

# ---- Run ----
if __name__ == "__main__":
    init_db()
    boards = [
        "https://example-jobs.com/search?q=data+engineer&l=remote",
        "https://example-jobs.com/search?q=ml+engineer&l=remote",
    ]
    postings = []
    for b in boards:
        try:
            postings += extract(b, POSTING_SCHEMA)
        except CreditsExhausted as e:
            print("STOP:", e); break

    # Dedupe by job_url before paying for detail extraction
    by_url = {p["job_url"]: p for p in postings if p.get("job_url")}
    job_urls = list(by_url)[:100]
    details = batch_extract(job_urls, DETAIL_SCHEMA)
    for res in details["results"]:
        if res["status"] == "completed" and res.get("data"):
            by_url[res["url"]].update(res["data"])

    upsert(list(by_url.values()))
    export_csv()
    print(json.dumps(trends(), indent=2, default=str))
```

The competitor monitor (`snapshot_careers` / `diff_careers` from the section above) runs as a separate daily job against the same DB. Together they give you a market dataset plus a live hiring-signal feed.

## No-code version: Thunderbit Chrome extension

The Open API is the programmatic surface of the same engine that powers the **Thunderbit AI Web Scraper Chrome extension**. Recruiters and people-analytics teammates can run these exact workflows without writing code:

1. Install the **Thunderbit** extension from the Chrome Web Store and **sign in** (a free tier is available — new accounts get a monthly page allowance).
2. Open a public job board or a company `/careers` page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns like `job_title`, `company`, `salary_range`, `job_url`.
3. Edit columns in plain language (tighten the salary instruction, drop fields you don't need), then click **Scrape**. Use **Scrape Subpages** to follow each row's `job_url` and pull skills/seniority/description from the detail page — the no-code version of the Step 2 → Step 3 list→detail pipeline.
4. Export to **Excel, Google Sheets, Airtable, or Notion**, or **schedule** recurring runs (e.g. a daily competitor-careers scrape).

Design and validate a schema interactively in the extension, then port it to the API/CLI for automation at scale — field names and instructions transfer directly.

## Automate it

- **Cron the pipeline.** Run the board sweep daily/weekly and the competitor monitor daily:

  ```cron
  # Job-market sweep — weekdays 06:00
  0 6 * * 1-5  THUNDERBIT_API_KEY=tb_... /usr/bin/python3 /opt/recruiting/pipeline.py
  # Competitor careers diff — daily 06:30
  30 6 * * *   THUNDERBIT_API_KEY=tb_... /usr/bin/python3 /opt/recruiting/monitor.py
  ```

- **Batch webhooks instead of polling.** For large detail-extract jobs, supply a `webhook` so Thunderbit POSTs the finished job to you — verify the `secret`:

  ```json
  {
    "urls": ["https://example-jobs.com/job/...", "..."],
    "schema": { },
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "your-signing-secret"
    }
  }
  ```

- **Sync to your ATS / Google Sheets.** After `upsert`, push new rows to a Sheet or your ATS via its API. Key everything on `job_url`.
- **Dedupe by `job_url`.** It's the stable primary key across boards and across days — make it your `PRIMARY KEY` (as in the pipeline) so re-running never duplicates a role.

## Cost & scale

Pricing (from §9 of the [API reference](docs/thunderbit-api-reference.md)): **distill 1 credit**, **suggest_fields 1 credit**, **extract 20 credits**; batch scales per URL; **polling is free**.

Worked example — a 1,000-posting detail sweep:

| Approach | What you get | Credits |
|----------|--------------|---------|
| **Extract** each detail page | Structured `skills`, `seniority`, `salary_min/max` | 1,000 × 20 = **20,000** |
| **Distill** each detail page | Clean Markdown of the description (text only) | 1,000 × 1 = **1,000** |

**Decision rule:** if you need *structured fields* you can aggregate (salary numbers, a skills column, employment type), use **extract**. If you only need the *description text* — for RAG, summarization, or to feed an LLM that will parse it later — **distill is 20× cheaper**. A common hybrid: distill the long descriptions, but extract structured fields only for postings that pass an initial filter.

To stay cheap and polite at scale: **cache and dedupe by `job_url`** so you never re-extract an unchanged posting; only fetch detail pages for *new* `job_url`s each run; batch in chunks of up to 100; and throttle concurrency so you don't hammer a single board's origin.

## Responsible use

- **Public, no-login sources only.** Everything here targets publicly accessible pages — public job boards, public company `/careers` pages, public Greenhouse/Lever boards, public Wellfound listings, public salary and reputation pages (Glassdoor public pages, Trustpilot), and self-published public portfolios.
- **No LinkedIn, Facebook, or Instagram. No candidate-profile scraping behind logins.** Those platforms are login-walled, forbid scraping in their terms, and their data is overwhelmingly personal. This guide deliberately excludes them — favor job-market and company-level data instead.
- **GDPR / CCPA lawful basis + data minimization.** Any candidate personal data (from the portfolio-sourcing section) requires a documented lawful basis (e.g. a properly assessed legitimate interest under GDPR, or CCPA compliance), strict minimization (collect only what's needed), retention limits, and the ability to honor access/deletion requests. Prefer job-market and company-level data, which is non-personal. When in doubt, consult counsel.
- **Respect `robots.txt` and Terms of Service** for every site you target.
- **Throttle.** Use batch with reasonable concurrency; cache and dedupe; don't hammer a single origin.
- **Bias & fairness in automated sourcing.** Automated filtering can encode and amplify bias. Don't filter candidates on protected characteristics or proxies for them; keep a human in the loop for any decision that affects a person; document your criteria; and treat scraped portfolio signals as research leads, not as scores.

## Troubleshooting & FAQ

- **Salaries are inconsistent or missing.** Salary text is the messiest field on any board. Push the work into the `instruction`: for normalized numbers use two fields — `salary_min`/`salary_max` as `NUMBER` with `instruction: "Lower/upper bound of annual salary in USD, digits only, no symbols; empty if not stated"`. Keep a raw `salary_range` `TEXT` field too, so you can audit parsing. Many postings simply omit pay — expect empties and report a "with_salary" coverage rate (the pipeline does).
- **The board renders blank or returns no rows.** It's a JS-heavy SPA. Set `renderMode: "full"` and add `waitFor` (e.g. 2000–3000 ms) so client-side cards load before extraction. If a single heavy page still times out, raise `timeout` or use the `/async/extract` submit-and-poll flow from §3.6 of the reference.
- **Pagination.** Board search results paginate. Either iterate the page-number URL parameter (`&page=2`, `&start=20`) and extract each results URL, or collect the per-page result URLs and fan them out with `/batch/extract`. Dedupe the merged rows by `job_url`.
- **Duplicate roles across boards.** The same role often appears on a company page *and* an aggregator. Dedupe by `job_url`; for cross-board dedupe where URLs differ, add a secondary key of `lower(company) + lower(job_title) + location`.
- **Geo-specific results.** Boards localize by visitor region. Set `country_code` (ISO-2, e.g. `US`, `DE`, `GB`) on the request to geo-route it and get the listings a local visitor would see. (The MCP tool exposes this as `countryCode`.)
- **401 / 402 / 429.** `401` = bad/missing key (fix `THUNDERBIT_API_KEY`, never retry). `402` = out of credits (hard stop, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off exponentially with jitter, as the pipeline does).
