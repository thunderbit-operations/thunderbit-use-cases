# Social Media Operations: Influencer Discovery, Public Content Tracking & Brand-Mention Monitoring with Thunderbit

Build a creator shortlist, a public content-performance tracker, and an open-web brand-mention monitor — entirely from publicly accessible, no-login pages.

---

## The data problem in social media ops

Running social and creator marketing is mostly a data-collection problem wearing a creative hat. The decisions are clear — which creators to brief, which content formats are winning, how the brand is being talked about this week — but the inputs are scattered across dozens of public surfaces that share no schema and never export cleanly.

Three pains dominate:

- **Fragmented public data.** Creator stats live on one page, niche on another, booking links somewhere else. Public video performance sits inside channel pages that paginate and lazy-load. There's no single feed.
- **Manual influencer vetting.** Teams open a directory, copy a handle into a spreadsheet, visit the profile, eyeball the follower count, paste a contact URL, and repeat for hundreds of creators. By the time the shortlist is built it's already stale.
- **Missing brand mentions.** Conversations about your brand and competitors happen in the open — Google News, blogs, Reddit threads, Product Hunt, Trustpilot — but nobody is watching all of it, so you find out about a sentiment spike days late.

**This guide deliberately uses PUBLIC, no-login sources only.** That means public YouTube and public TikTok pages, public creator directories and marketplaces, and open-web mention sources (Google News, public blogs, public Reddit threads, Product Hunt, Trustpilot, public hashtag/aggregator pages). It intentionally **does not** touch login-walled platforms — no Facebook, Instagram, LinkedIn, or private/authenticated X.

That stance is partly compliance (those platforms forbid scraping in their terms and their data is frequently personal data) and partly durability: public pages are stable, defensible, and don't break the moment a platform tightens its auth wall. Where a public page exposes personal data about a creator, we flag a lawful-basis caution.

Everything below maps to the canonical [API reference](docs/thunderbit-api-reference.md) — every endpoint, schema, and response shape comes from there. If a sample here ever disagrees with that file, the reference wins.

## What you'll build

- **An influencer shortlist** scraped from a public creator directory: handle, platform, follower count, niche, public booking/contact URL, and profile URL.
- **A public content-performance tracker** that pulls video lists (title, views, publish date, video URL) from public YouTube channel pages for competitive content analysis, across many channels via batch.
- **A brand & competitor mention monitor** that distills Google News / public mention pages to clean Markdown, then feeds an LLM for sentiment and theme tagging — at a fraction of the cost of structured extraction.
- **One runnable end-to-end Python pipeline** that ties discovery, enrichment, scoring, and a daily mention monitor together, with proper 401/402/429 handling.

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

New accounts get a one-time free allotment (~600 units) — enough to prototype distill and extract before you top up. See the [API reference](docs/thunderbit-api-reference.md) for auth, pricing, and error semantics.

## Three ways to call Thunderbit

The same web-data engine is exposed through three surfaces that all hit the identical HTTP API, so you can mix them freely.

| Surface | Entry point | Best for in social media ops |
|---------|-------------|------------------------------|
| **MCP server** | `@thunderbit/mcp-server` (npx) | An AI assistant (Claude, Cursor, Cline) doing interactive research: "shortlist 50 creators from this directory and tag their niche." No code. |
| **HTTP API** | `https://openapi.thunderbit.com` | Servers, cron jobs, dashboards, webhooks. Any language. The production workhorse for daily mention monitoring. |
| **CLI + SDK** | `@thunderbit/thunderbit-cli` (npm) | Local scripting, shell pipelines, and saving/reusing a schema for repeated channel pulls. |

Rule of thumb: prototype and validate the schema in MCP or the CLI, then automate at scale with the HTTP API.

## Step 1 — Discover the schema on a public creator directory

Don't guess field names. Point Thunderbit's `suggest_fields` at a public creator-directory listing page and let the AI propose a schema. We'll use a generic public marketplace listing as the example: `https://example-creators.com/directory/youtube/tech`.

### Via MCP (AI assistant)

In Claude Desktop / Cursor / Cline with the Thunderbit MCP server configured, just ask:

> "Use `thunderbit_suggest_fields` on `https://example-creators.com/directory/youtube/tech` with the prompt *'Each creator card: handle, platform, follower count, niche, public booking or contact link, and profile link'*."

The assistant calls the `thunderbit_suggest_fields` tool (`url`, `prompt`, `countryCode?`) and returns the proposed fields. Costs 1 credit.

### Via curl

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/suggest_fields \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example-creators.com/directory/youtube/tech",
    "prompt": "Each creator card: handle, platform, follower count, niche, public booking or contact link, profile link",
    "country_code": "US"
  }'
```

A realistic response (fields live at `data`):

```json
{
  "data": [
    { "name": "creator_handle", "type": "TEXT",   "instruction": "The creator's @handle or display name on the card" },
    { "name": "platform",       "type": "TEXT",   "instruction": "Platform label, e.g. YouTube or TikTok" },
    { "name": "followers",      "type": "NUMBER", "instruction": "Follower/subscriber count as a number" },
    { "name": "niche",          "type": "TEXT",   "instruction": "Content category or niche tag" },
    { "name": "booking_url",    "type": "URL",    "instruction": "Public booking, contact, or business inquiry link" },
    { "name": "profile_url",    "type": "URL",    "instruction": "Link to the creator's profile page" }
  ]
}
```

> Some deployments wrap the array as `data.fields`. Handle both: `fields = res["data"].get("fields", res["data"])` in Python, or `res.data?.fields ?? res.data` in JS.

### Via CLI

```bash
thunderbit suggest-fields "https://example-creators.com/directory/youtube/tech" \
  --prompt "Each creator card: handle, platform, followers, niche, booking link, profile link" \
  --country-code US
```

## Step 2 — Extract the creator list

Now turn the listing page into structured rows. The directory is a list of creators, so use an **array** schema. The `instruction` strings are your biggest quality lever — write them the way you'd brief a human assistant.

### Via curl

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example-creators.com/directory/youtube/tech",
    "renderMode": "full",
    "waitFor": 2500,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "creator_handle": { "type": "TEXT",   "instruction": "The creator @handle or display name" },
          "platform":       { "type": "TEXT",   "instruction": "Platform name, e.g. YouTube or TikTok" },
          "followers":      { "type": "NUMBER", "instruction": "Follower/subscriber count as a plain integer; convert 1.2M to 1200000 and 34K to 34000" },
          "niche":          { "type": "TEXT",   "instruction": "Content niche or category label" },
          "booking_url":    { "type": "URL",    "instruction": "Public booking/contact link, empty if none shown" },
          "profile_url":    { "type": "URL",    "instruction": "Absolute URL to the creator profile page" }
        }
      }
    }
  }'
```

> **JS-heavy directories:** many creator marketplaces render cards client-side. Set `renderMode: "full"` to run a headless browser, and add `waitFor` (ms, 0–10000) so lazy content settles before extraction. If a page is still slow, raise `timeout` (extract allows 5000–120000 ms) or use the async endpoint.

Response — rows at `data`:

```json
{
  "data": [
    { "creator_handle": "@techwithmaya", "platform": "YouTube", "followers": 1200000, "niche": "Consumer tech reviews", "booking_url": "https://example-creators.com/book/techwithmaya", "profile_url": "https://example-creators.com/c/techwithmaya" },
    { "creator_handle": "@gadgetgrove",  "platform": "YouTube", "followers": 340000,  "niche": "Gadgets & unboxing",   "booking_url": "",                                            "profile_url": "https://example-creators.com/c/gadgetgrove" }
  ]
}
```

### Via Python

```python
import os, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

CREATOR_SCHEMA = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "creator_handle": {"type": "TEXT",   "instruction": "The creator @handle or display name"},
            "platform":       {"type": "TEXT",   "instruction": "Platform name, e.g. YouTube or TikTok"},
            "followers":      {"type": "NUMBER", "instruction": "Follower count as a plain integer; convert 1.2M to 1200000, 34K to 34000"},
            "niche":          {"type": "TEXT",   "instruction": "Content niche or category label"},
            "booking_url":    {"type": "URL",    "instruction": "Public booking/contact link, empty if none"},
            "profile_url":    {"type": "URL",    "instruction": "Absolute URL to the creator profile page"},
        },
    },
}

def extract(url, schema, render_mode="full", wait_for=2500, timeout=90000):
    r = requests.post(f"{BASE}/openapi/v1/extract", headers=HEADERS, json={
        "url": url, "schema": schema, "renderMode": render_mode,
        "waitFor": wait_for, "timeout": timeout,
    }, timeout=130)
    r.raise_for_status()
    return r.json()["data"]

creators = extract("https://example-creators.com/directory/youtube/tech", CREATOR_SCHEMA)
print(f"{len(creators)} creators")
```

### Via CLI

Design and save the schema interactively, then reuse it:

```bash
# -i opens an interactive field editor; omit --schema to auto-suggest first
thunderbit extract "https://example-creators.com/directory/youtube/tech" \
  -i --render-mode full --save-schema creators.schema.json

# Re-run with the saved schema, print a table
thunderbit extract "https://example-creators.com/directory/youtube/tech" \
  --schema creators.schema.json --render-mode full -f table
```

## Step 3 — Public content performance

Now track what's actually working. Point `extract` at a **public YouTube channel's Videos tab** — public metadata only, no login — and pull the recent video list for competitive content analysis.

Video lists are an array, one object per video:

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://www.youtube.com/@exampletechchannel/videos",
    "renderMode": "full",
    "waitFor": 3000,
    "schema": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "title":     { "type": "TEXT",   "instruction": "The video title text" },
          "views":     { "type": "NUMBER", "instruction": "View count as integer; convert 1.2M views to 1200000, 45K to 45000" },
          "published": { "type": "DATE",   "instruction": "When the video was published; normalize relative dates like 2 weeks ago" },
          "video_url": { "type": "URL",    "instruction": "Absolute URL to the watch page for this video" }
        }
      }
    }
  }'
```

YouTube channel pages are JS-rendered and lazy-load on scroll, so `renderMode: "full"` plus a `waitFor` of ~3000 ms is important here. See Troubleshooting for infinite-scroll notes.

### Batch several channels at once

To compare competitors, batch up to 100 channel URLs in one job with `/batch/extract` (20 credits per URL):

```bash
# 1. Create the batch job
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/extract \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://www.youtube.com/@exampletechchannel/videos",
      "https://www.youtube.com/@anothertechchannel/videos",
      "https://www.youtube.com/@thirdtechchannel/videos"
    ],
    "schema": { "type": "array", "items": { "type": "object", "properties": {
      "title":     { "type": "TEXT",   "instruction": "The video title text" },
      "views":     { "type": "NUMBER", "instruction": "View count as integer; convert 1.2M to 1200000" },
      "published": { "type": "DATE",   "instruction": "Publish date; normalize relative dates" },
      "video_url": { "type": "URL",    "instruction": "Absolute watch-page URL" }
    } } },
    "timeout": 90000
  }'
# → { "data": { "id": "batch_yt001", "status": "pending", "total": 3 } }

# 2. Poll until done (page is 0-based; pageSize max 100)
curl -sS "https://openapi.thunderbit.com/openapi/v1/batch/extract/batch_yt001?page=0&pageSize=100" \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY"
```

The batch response carries per-URL `results[].status`, so you can retry only channels that hit `failed` (e.g. `"error": "timeout"`) instead of re-running the whole job.

> **Deeper analysis angle:** Thunderbit also offers a public YouTube transcript capability. For content strategy you can take the `video_url`s from above and `distill` each public watch page to clean Markdown (1 credit each), then feed the transcript/description text to an LLM to mine hooks, topics, and structure — far cheaper than structured extraction when you only need text.

## Brand-mention monitoring with distill

For monitoring you usually don't need rigid columns — you need clean text you can hand to an LLM. That's exactly what **distill** is for: any public page → LLM-ready Markdown for **1 credit** (vs 20 for extract). For mention monitoring across Google News, blogs, and forum threads, distill is the right tool and 20× cheaper.

### Distill a Google News results page

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://news.google.com/search?q=%22YourBrand%22",
    "renderMode": "full",
    "country_code": "US",
    "excludeTags": ["nav", "footer", "aside"],
    "timeout": 30000
  }'
```

Response — Markdown at `data.markdown`:

```json
{
  "data": {
    "markdown": "# YourBrand - Google News\n\n- **YourBrand launches new feature** — TechBlog, 2h ago...\n- **Review: YourBrand vs CompetitorX** — ReviewSite, 1d ago...",
    "url": "https://news.google.com/search?q=%22YourBrand%22"
  }
}
```

### Feed the Markdown to an LLM for sentiment + themes

Distill gives you the text; your LLM does the tagging. Pass the Markdown with a structured prompt:

```python
import os, json, requests
from openai import OpenAI   # any LLM works; this is illustrative

llm = OpenAI()

def tag_mentions(markdown, brand):
    prompt = f"""You are a brand-monitoring analyst. From the Markdown below (a list of
public mentions of "{brand}"), return a JSON array. For each mention output:
  headline, source, sentiment (positive|neutral|negative), themes (array of short tags).
Only use what's in the text. Markdown:

{markdown}"""
    resp = llm.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"},
    )
    return json.loads(resp.choices[0].message.content)
```

**Why distill, not extract, here:** the news/forum surface changes layout constantly and you care about *meaning*, not fixed columns. Distill (1 credit) + LLM tagging is cheaper and more robust than extract (20 credits) for unstructured sentiment work. Reach for extract only when you need stable, structured rows (like the creator list in Step 2).

### Batch distill many mention URLs

When you have a list of mention pages (blog posts, Reddit threads, Product Hunt comments, Trustpilot reviews), batch distill them — 1 credit per URL:

```bash
curl -sS https://openapi.thunderbit.com/openapi/v1/batch/distill \
  -H "Authorization: Bearer $THUNDERBIT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [
      "https://www.reddit.com/r/technology/comments/example_thread/",
      "https://www.producthunt.com/posts/yourbrand",
      "https://www.trustpilot.com/review/yourbrand.com",
      "https://someblog.example.com/yourbrand-review"
    ],
    "timeout": 45000
  }'
# → { "data": { "id": "batch_mentions01", "status": "pending", "total": 4 } }
```

Poll the same way as batch extract: `GET /openapi/v1/batch/distill/batch_mentions01?page=0&pageSize=100`. Polling is free.

## End-to-end pipeline

Here is one cohesive, runnable script that ties the workflow together: discover creators from a directory, batch-enrich their profile pages, score and rank them — plus a daily brand-mention monitor that batch-distills mention URLs, runs LLM sentiment, and stores results. It handles `401` / `402` / `429` exactly as the [API reference](docs/thunderbit-api-reference.md) prescribes.

```python
import os, time, json, random, csv, requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


# ---- robust POST/GET with backoff per the API reference -------------------
def call(method, path, **kwargs):
    url = BASE + path
    for attempt in range(6):
        r = requests.request(method, url, headers=HEADERS, timeout=130, **kwargs)
        if r.status_code == 401:
            raise SystemExit("401 invalid API key — check THUNDERBIT_API_KEY")
        if r.status_code == 402:
            raise SystemExit("402 out of credits — top up at thunderbit.com/billing")
        if r.status_code == 429 or r.status_code >= 500:
            wait = (2 ** attempt) + random.random()      # exp backoff + jitter
            print(f"  {r.status_code}; retrying in {wait:.1f}s")
            time.sleep(wait)
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"gave up after retries: {method} {path}")


# ---- schemas --------------------------------------------------------------
CREATOR_SCHEMA = {
    "type": "array",
    "items": {"type": "object", "properties": {
        "creator_handle": {"type": "TEXT",   "instruction": "The creator @handle or display name"},
        "platform":       {"type": "TEXT",   "instruction": "Platform name, e.g. YouTube or TikTok"},
        "followers":      {"type": "NUMBER", "instruction": "Followers as integer; 1.2M->1200000, 34K->34000"},
        "niche":          {"type": "TEXT",   "instruction": "Content niche or category label"},
        "booking_url":    {"type": "URL",    "instruction": "Public booking/contact link, empty if none"},
        "profile_url":    {"type": "URL",    "instruction": "Absolute URL to the creator profile page"},
    }},
}

PROFILE_SCHEMA = {        # one record per public profile page (flat object)
    "type": "object",
    "properties": {
        "avg_views":      {"type": "NUMBER", "instruction": "Average views per recent video as integer if shown, else empty"},
        "engagement_pct": {"type": "NUMBER", "instruction": "Engagement rate percent as a number if shown, else empty"},
        "contact_email":  {"type": "EMAIL",  "instruction": "Public business email if listed, else empty"},
    },
}


# ---- Stage 1: discover creators from a public directory -------------------
def discover_creators(directory_url):
    return call("POST", "/openapi/v1/extract", json={
        "url": directory_url, "schema": CREATOR_SCHEMA,
        "renderMode": "full", "waitFor": 2500, "timeout": 90000,
    })["data"]


# ---- Stage 2: batch-enrich each public profile page -----------------------
def enrich_profiles(creators):
    urls = [c["profile_url"] for c in creators if c.get("profile_url")]
    job = call("POST", "/openapi/v1/batch/extract", json={
        "urls": urls[:100], "schema": PROFILE_SCHEMA,
        "timeout": 90000,
    })["data"]
    job_id = job["id"]
    while True:
        st = call("GET", f"/openapi/v1/batch/extract/{job_id}",
                  params={"page": 0, "pageSize": 100})["data"]
        if st["status"] in ("completed", "failed", "cancelled"):
            break
        time.sleep(5)                          # polling is free
    # map enrichment back onto creators by URL
    by_url = {r["url"]: (r.get("data") or {}) for r in st.get("results", [])
              if r["status"] == "completed"}
    for c in creators:
        extra = by_url.get(c.get("profile_url"), {})
        # batch profile data may be a 1-item list or an object
        if isinstance(extra, list):
            extra = extra[0] if extra else {}
        c.update({k: extra.get(k) for k in ("avg_views", "engagement_pct", "contact_email")})
    return creators


# ---- Stage 3: score & rank ------------------------------------------------
def score(creators):
    for c in creators:
        followers = c.get("followers") or 0
        eng = c.get("engagement_pct") or 0
        # simple fit score: reward engagement, dampen pure follower vanity
        c["fit_score"] = round((eng * 1000) + (followers ** 0.5), 1)
    return sorted(creators, key=lambda c: c["fit_score"], reverse=True)


# ---- Daily brand-mention monitor ------------------------------------------
def monitor_mentions(mention_urls, brand):
    job = call("POST", "/openapi/v1/batch/distill", json={
        "urls": mention_urls[:100], "timeout": 45000,
    })["data"]
    job_id = job["id"]
    while True:
        st = call("GET", f"/openapi/v1/batch/distill/{job_id}",
                  params={"page": 0, "pageSize": 100})["data"]
        if st["status"] in ("completed", "failed", "cancelled"):
            break
        time.sleep(5)
    tagged = []
    for r in st.get("results", []):
        if r["status"] != "completed":
            continue
        md = (r.get("data") or {}).get("markdown", "")
        tagged.append({"url": r["url"], "tags": tag_mentions(md, brand)})  # LLM step from earlier
    return tagged


if __name__ == "__main__":
    creators = discover_creators("https://example-creators.com/directory/youtube/tech")
    creators = enrich_profiles(creators)
    ranked = score(creators)

    with open("influencer_shortlist.csv", "w", newline="") as f:
        cols = ["creator_handle", "platform", "followers", "niche",
                "engagement_pct", "contact_email", "booking_url", "profile_url", "fit_score"]
        w = csv.DictWriter(f, fieldnames=cols, extrasaction="ignore")
        w.writeheader()
        w.writerows(ranked)
    print(f"Wrote {len(ranked)} creators to influencer_shortlist.csv")

    mentions = monitor_mentions([
        "https://news.google.com/search?q=%22YourBrand%22",
        "https://www.reddit.com/r/technology/comments/example_thread/",
        "https://www.trustpilot.com/review/yourbrand.com",
    ], brand="YourBrand")
    with open("mentions_today.json", "w") as f:
        json.dump(mentions, f, indent=2)
    print(f"Stored {len(mentions)} mention pages with sentiment tags")
```

The `tag_mentions` function is the LLM step shown earlier in the distill section — drop it in alongside this script.

## No-code version: Thunderbit Chrome extension

Everyone on the team can run these exact workflows without code through the **Thunderbit AI Web Scraper Chrome extension** — it's the no-code surface of the same engine.

1. Install **Thunderbit** from the Chrome Web Store and **sign in / create an account**. A free tier is available (new accounts get a monthly page allowance) — note that you do need an account.
2. Open the public creator directory or public YouTube channel page, click the extension, and hit **AI Suggest Fields** — the UI equivalent of `suggest_fields`. Thunderbit reads the page and proposes columns.
3. Edit columns in plain language (rename, add, tweak the instruction), then click **Scrape**. Use **Scrape Subpages** to follow each creator's `profile_url` and pull profile detail — the no-code version of the list→detail pipeline above.
4. Export to **Excel, Google Sheets, Airtable, or Notion**, or **schedule** recurring runs so the shortlist and content tracker refresh themselves.

The schema you design in the extension (field names + instructions) ports directly to the API/CLI when you're ready to automate at scale.

## Automate it

- **Cron the monitor.** Run the brand-mention monitor on a schedule (e.g. hourly for news, daily for reviews):

  ```bash
  # crontab -e — run the daily monitor at 08:00
  0 8 * * *  THUNDERBIT_API_KEY=tb_xxx /usr/bin/python3 /opt/social/pipeline.py >> /var/log/social.log 2>&1
  ```

- **Use batch webhooks** instead of polling so your server is notified when a job finishes. Supply a callback `url` + a signing `secret`; verify the secret to trust the call:

  ```json
  {
    "urls": ["https://news.google.com/search?q=%22YourBrand%22", "..."],
    "webhook": {
      "url": "https://your-server.example.com/thunderbit/callback",
      "secret": "your-signing-secret"
    }
  }
  ```

- **Store trends.** Append each run's results to a time-series table (date, mention_count, positive/neutral/negative split, video views per channel). Trends matter more than snapshots.
- **Alert on mention spikes.** Compare today's mention count or negative-sentiment share against a trailing 7-day average; if it crosses a threshold, post to Slack/email. A sudden negative spike is the signal worth waking someone for.

## Cost & scale

Credit math is the whole game when monitoring at volume:

| Operation | Cost | 1,000 URLs |
|-----------|------|------------|
| `suggest_fields` | 1 credit | 1,000 |
| `distill` (mentions, text) | 1 credit / URL | 1,000 |
| `extract` (structured rows) | 20 credits / URL | 20,000 |

**Distilling mentions is 20× cheaper than extracting them.** For sentiment and theme work you only need clean text, so distill (1/URL) + LLM tagging beats extract (20/URL) every time. Reserve extract for cases where you genuinely need stable, typed columns — the creator shortlist and the YouTube video lists.

Practical levers:

- **Distill vs extract:** ask "do I need fixed fields, or just meaning?" Meaning → distill.
- **Throttle** with batch and reasonable concurrency; don't hammer one origin.
- **Cache and dedupe.** Don't re-scrape unchanged mention URLs or channel pages. Hash the URL + last-seen content and skip if unchanged — it saves credits and load.
- **Status polling is free**, so prefer batch + poll/webhook over many synchronous calls.

## Responsible use

- **PUBLIC, no-login sources only.** Everything here targets publicly accessible pages: public YouTube and public TikTok pages, public creator directories/marketplaces, Google News, public blogs, public Reddit threads, Product Hunt, Trustpilot, and public hashtag/aggregator pages.
- **No walled gardens.** Do **not** build workflows against login-walled platforms — **no Facebook, Instagram, LinkedIn, or private/authenticated X scraping.** Those platforms forbid it in their terms, and the data is frequently personal data. Public pages are also the more durable, defensible choice.
- **GDPR / CCPA caution for creator personal data.** Handles, follower counts, and contact emails can be personal data. Only collect what you have a lawful basis to process, prefer business/booking contact channels over personal ones, minimize what you store, and honor deletion requests.
- **Respect `robots.txt` and each site's Terms of Service.**
- **Throttle.** Use batch with reasonable concurrency and back off on `429`. Don't overload any single origin.

## Troubleshooting & FAQ

**Extract returns 0 rows or far fewer than I see on screen.** The page is JS-rendered. Set `renderMode: "full"` to run a headless browser, and add `waitFor` (up to 10000 ms) so cards finish loading. Creator marketplaces and YouTube channel pages almost always need this.

**Follower counts come back as `"1.2M"` instead of a number.** Use `type: "NUMBER"` and spell the conversion out in the `instruction`: *"Follower count as a plain integer; convert 1.2M to 1200000 and 34K to 34000."* The instruction is the lever — be explicit about the format you want.

**Pagination / infinite scroll.** A single render captures only what loads in the viewport-plus-`waitFor` window. For paginated directories, extract each page URL (`?page=2`, `?page=3`...) and pass them all to `/batch/extract`. For infinite scroll where only a fixed number of items ever load without scrolling, treat the first render as a capped sample, or look for an underlying paginated URL the site uses.

**Relative publish dates ("2 weeks ago").** Keep `type: "DATE"` and instruct normalization: *"Publish date; normalize relative dates like '2 weeks ago' to an absolute date."* Then normalize again in your own code against the run timestamp if you need exact dates.

**Geo-specific results (e.g. Google News in a region).** Pass `country_code` (ISO-2, e.g. `DE`, `GB`) on distill and suggest_fields to geo-route the request. Default is `US`. Note the HTTP body uses snake_case `country_code`; the MCP tool exposes it as `countryCode`.

**A few URLs in a batch fail.** Inspect per-URL `results[].status` and retry only the `failed` entries (common cause: `timeout` — raise `timeout` or set `renderMode:"full"` for those URLs).

**401 / 402 / 429.** `401` = bad key (never retry — fix `THUNDERBIT_API_KEY`). `402` = out of credits (hard stop, top up). `429` = rate-limited (back off with exponential backoff + jitter). The pipeline above implements all three.
