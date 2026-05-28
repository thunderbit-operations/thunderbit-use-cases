# Give Your AI Agent Live Web Data with Thunderbit

An LLM agent is only as good as the context it can reach. Plug Thunderbit into your agent and it stops guessing from stale training data — it can discover sources, read live pages as clean Markdown, and pull structured JSON it can act on, all through a handful of tools the model calls itself.

> Every code sample in this tutorial conforms to the canonical [API reference](docs/thunderbit-api-reference.md). If anything here ever disagrees with that file, the reference wins. Base URL is `https://openapi.thunderbit.com`, the prefix is `/openapi/v1`, and auth is `Authorization: Bearer tb_...`.

This is a developer-integration guide. If you're shipping a user-facing product on top of these primitives, see the companion [App development](15-app-development.md) tutorial; here we focus on the agent itself — MCP wiring, function-calling tool schemas, the agent loop, and cost control in autonomous runs.

---

## Two ways to give an agent web data

There are exactly two integration paths, and which you pick depends on how much of the loop you control.

| Path | What it is | Use it when |
|------|------------|-------------|
| **A — MCP server** | Run `@thunderbit/mcp-server` and register it with an MCP-aware host (Claude Desktop/Code, Cursor, Cline). The model sees seven `thunderbit_*` tools and calls them directly. **Zero glue code.** | You're inside an MCP-aware assistant or framework and want the model to reach the web with no plumbing. Fastest to set up; least control over the loop. |
| **B — HTTP API as function-calling tools** | You define tool/function JSON schemas, run your own agent loop, and execute each proposed tool call against the HTTP API yourself. | You're building a custom agent (your own loop, your own model, your own budget/guardrail code). Full control over every call, retry, and credit spent. |

Both paths drive the **same engine** underneath, so a schema or field instruction you validate in MCP transfers verbatim into your function-calling agent. A common progression: prototype the workflow conversationally in MCP, then re-implement the proven flow as function-calling tools in your production agent where you can enforce budgets and guardrails in code.

The three core capabilities your agent gets, either way:

1. **Distill** — turn any public page into clean, LLM-ready Markdown (1 credit).
2. **Extract** — pull structured JSON from a page using a field schema (20 credits).
3. **Suggest fields** — let the AI inspect a page and propose an extraction schema (1 credit).

Plus batch variants of distill and extract (up to 100 URLs per job) and free status polling.

---

## Path A — wire up the MCP server

The MCP server exposes seven tools over stdio. You don't write any code — you add one block to your MCP client config, and the model calls the tools itself.

### Client config (Claude Desktop, Cursor, Cline)

Claude Desktop reads `~/Library/Application Support/Claude/claude_desktop_config.json` on macOS (`%APPDATA%\Claude\claude_desktop_config.json` on Windows). Cursor and Cline use the same shape:

```json
{
  "mcpServers": {
    "thunderbit": {
      "command": "npx",
      "args": ["-y", "@thunderbit/mcp-server"],
      "env": { "THUNDERBIT_API_KEY": "tb_your_api_key_here" }
    }
  }
}
```

`npx` pulls the server on demand, so there's nothing to install ahead of time. Restart the host after editing the config so it picks up the new server.

### Claude Code plugin

The Claude Code plugin bundles the MCP server plus four skills:

```bash
claude plugin marketplace add thunderbit-open/thunderbit-mcp-server
claude plugin add thunderbit
export THUNDERBIT_API_KEY=tb_your_api_key_here
```

Get a key at <https://app.thunderbit.com/console/api-keys> — keys are prefixed `tb_`. Both the MCP server and the CLI read `THUNDERBIT_API_KEY` automatically; resolution order is explicit `apiKey` argument → `THUNDERBIT_API_KEY` env var → error.

### The seven tools

| Tool | Cost | Key arguments |
|------|------|---------------|
| `thunderbit_suggest_fields` | 1 credit | `url`, `prompt?`, `countryCode?`, `apiKey?` |
| `thunderbit_distill` | 1 credit | `url`, `renderMode?`, `timeout?`, `waitFor?`, `includeTags?`, `excludeTags?`, `countryCode?`, `apiKey?` |
| `thunderbit_extract` | 20 credits | `url`, `schema`, `renderMode?`, `timeout?`, `waitFor?`, `apiKey?` |
| `thunderbit_batch_distill_create` | 1 credit/URL | `urls`, `timeout?`, `apiKey?` |
| `thunderbit_batch_distill_status` | free | `jobId`, `page?`, `pageSize?`, `apiKey?` |
| `thunderbit_batch_extract_create` | 20 credits/URL | `urls`, `schema`, `timeout?`, `apiKey?` |
| `thunderbit_batch_extract_status` | free | `jobId`, `page?`, `pageSize?`, `apiKey?` |

> The MCP tools use `countryCode` (camelCase); the raw HTTP body uses `country_code` (snake_case) — the MCP server translates for you.

### Driving a `suggest_fields → extract` flow with plain language

In an MCP host you describe the task and the model chooses and sequences the tools. A prompt that drives the recommended discovery-then-extract flow:

> Use Thunderbit to suggest fields for `https://example.com/products` — focus on each product card: name, price, rating, and the detail-page link. Show me the proposed fields, then once I confirm, extract every product on the page as a table.

The model issues a `thunderbit_suggest_fields` call:

```json
{
  "url": "https://example.com/products",
  "prompt": "Extract each product card: name, price, rating, detail-page link",
  "countryCode": "US"
}
```

reads back the proposed fields (cheap — 1 credit), and after you confirm, issues a `thunderbit_extract` call with a schema built from those fields. Asking the model to *show the proposed fields and wait for confirmation* before extracting is the human-in-the-loop pattern that stops it from spending 20 credits on a guess.

---

## The recommended agent workflow

Whichever path you take, the engine's intended sequence is the same:

```
1. thunderbit_suggest_fields   → discover what data the page has   (1 credit)
2. thunderbit_extract          → pull it as structured JSON        (20 credits)
   OR thunderbit_distill       → get the page as Markdown for RAG  (1 credit)
3. thunderbit_batch_*_create   → submit up to 100 URLs
4. thunderbit_batch_*_status   → poll until done                   (free)
```

The critical habit to bake into your agent — in the system prompt for Path A, in code for Path B — is **suggest fields before you extract.** `suggest_fields` costs 1 credit and tells the model exactly what the page contains and what a good schema looks like. `extract` costs 20 credits. An agent that jumps straight to `extract` with a guessed schema risks paying 20 credits for fields that aren't on the page, then paying another 20 to retry. Cheap discovery first, then one well-formed extract — that's a 20× difference per misfire.

The same logic scales to batch: validate the schema on **one** representative page with `suggest_fields` + `extract`, then run `batch_extract_create` over the remaining URLs only once you trust it.

---

## Path B — expose Thunderbit as LLM tools (function calling)

In a custom agent you own the loop: the model proposes a tool call, *your code* executes it against the HTTP API, and you feed the result back. Start by describing the tools to the model. These JSON schemas map one-to-one onto the `distill` and `extract` endpoints — most providers (OpenAI, Anthropic, Gemini, plus open-source runners) accept this shape with minor wrapper differences.

```python
TOOL_SCHEMAS = [
    {
        "name": "thunderbit_distill",
        "description": (
            "Fetch a public web page and return it as clean Markdown for reading, "
            "reasoning, or RAG. Cheap (1 credit). Use this when you need to READ a "
            "page's text, not when you need structured fields."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "url": {"type": "string", "description": "Public page URL to fetch."},
                "renderMode": {
                    "type": "string",
                    "enum": ["none", "basic", "full"],
                    "description": (
                        "none = raw HTML (fastest); basic = light JS; "
                        "full = headless browser for JS-heavy SPAs. Default none."
                    ),
                },
                "waitFor": {
                    "type": "integer",
                    "description": "Extra wait in ms (0-10000) for lazy content.",
                },
            },
            "required": ["url"],
        },
    },
    {
        "name": "thunderbit_extract",
        "description": (
            "Extract STRUCTURED JSON from a public page using a field schema. "
            "Expensive (20 credits). Use only when you need deterministic fields "
            "(prices, dates, emails) for downstream actions — not just to read text. "
            "Call suggest_fields-style reasoning first to pick the right fields."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "url": {"type": "string", "description": "Public page URL to extract from."},
                "schema": {
                    "type": "object",
                    "description": (
                        "Thunderbit schema: "
                        "{type:'array', items:{type:'object', properties:{ "
                        "<field>:{type:'TEXT|NUMBER|URL|EMAIL|DATE', instruction:'...'} }}}"
                    ),
                },
                "renderMode": {
                    "type": "string",
                    "enum": ["none", "basic", "full"],
                },
            },
            "required": ["url", "schema"],
        },
    },
]
```

Now the real part — the code that executes whatever the model proposes against the HTTP API. This is fully real and correct against the reference; only the LLM call is a pluggable stub.

```python
import os, time, json, random
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


class OutOfCredits(Exception):
    """402 — surface to the model so it can stop or ask a human."""


def _post(path, body, attempts=5):
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


# --- Tool implementations: these run when the model proposes a call ---------

def tool_thunderbit_distill(url, renderMode="none", waitFor=0):
    body = {"url": url, "renderMode": renderMode}
    if waitFor:
        body["waitFor"] = waitFor
    res = _post("/openapi/v1/distill", body)
    return res["data"]["markdown"]


def tool_thunderbit_extract(url, schema, renderMode="none"):
    res = _post("/openapi/v1/extract", {
        "url": url, "schema": schema, "renderMode": renderMode,
    })
    return res["data"]


TOOL_IMPLS = {
    "thunderbit_distill": tool_thunderbit_distill,
    "thunderbit_extract": tool_thunderbit_extract,
}


def execute_tool(name, args):
    """Run a proposed tool call, returning a string the model can read.
    Errors are returned as concise strings, not raised — the model reacts to them."""
    fn = TOOL_IMPLS.get(name)
    if not fn:
        return f"ERROR: unknown tool {name!r}"
    try:
        result = fn(**args)
        return json.dumps(result) if not isinstance(result, str) else result
    except OutOfCredits as e:
        return f"TOOL_ERROR(out_of_credits): {e}. Stop and ask a human to top up."
    except Exception as e:  # noqa: BLE001 — surface everything to the model as text
        return f"TOOL_ERROR: {e}"
```

The agent loop ties it together. The LLM call is a stub you swap for your provider's SDK; everything around it is real.

```python
def call_llm(messages, tools):
    """STUB — replace with your provider's chat/completions call.

    Must return either:
      {"type": "tool_call", "name": <str>, "args": <dict>}   to invoke a tool, or
      {"type": "final",     "content": <str>}                to finish.

    e.g. OpenAI: client.chat.completions.create(model=..., messages=..., tools=...)
         Anthropic: client.messages.create(model=..., messages=..., tools=...)
    Parse the provider's tool-call response into the shape above.
    """
    raise NotImplementedError("Wire this to your LLM provider")


def run_agent(goal, max_tool_calls=8):
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},  # defined in the guardrails section
        {"role": "user", "content": goal},
    ]
    for _ in range(max_tool_calls):
        step = call_llm(messages, TOOL_SCHEMAS)
        if step["type"] == "final":
            return step["content"]
        name, args = step["name"], step["args"]
        print(f"→ {name}({json.dumps(args)[:120]})")
        observation = execute_tool(name, args)
        # Append the model's tool call and the observation back into the transcript.
        messages.append({"role": "assistant", "tool_call": {"name": name, "args": args}})
        messages.append({"role": "tool", "name": name, "content": observation})
    return "Stopped: hit the tool-call budget before finishing."
```

The `max_tool_calls` cap is your first line of defense against a runaway loop — more on that below.

---

## distill vs extract for agents

This is the single most important cost decision your agent makes, and it's worth encoding as an explicit rule.

| | `distill` | `extract` |
|---|-----------|-----------|
| **Cost** | 1 credit | 20 credits |
| **Returns** | Clean Markdown (`data.markdown`) | Structured rows (`data`) |
| **Best for** | Reading, reasoning, summarizing, RAG indexing | Deterministic downstream actions: writing to a DB, filtering on price, deduping by email |

**Rule of thumb for the agent (put it in the system prompt):** *If you're going to read the page and reason about its text, use `distill`. Only use `extract` when you need specific fields as machine-readable values that something downstream will act on.* Need to summarize twenty articles? Distill them (20 credits total) and let the model read them. Need a price-and-availability table you'll sort and store? Extract the one page (20 credits) — but distill the other nineteen if you only need their prose. Getting this wrong is a 20× cost error repeated on every page the agent touches.

---

## A worked multi-step agent

Here's a cohesive research agent: given a goal, it works over a set of seeded source URLs, **batch-distills** them cheaply, then asks the model to synthesize an answer with citations. The Thunderbit parts are fully real (including 401/402/429 handling); the LLM reasoning is a clearly marked stub.

```python
#!/usr/bin/env python3
"""Research agent: seed sources -> batch distill -> synthesize with citations.
Thunderbit calls are real; the LLM synthesis is stubbed."""
import os, time, json, random
import requests

BASE = "https://openapi.thunderbit.com"
KEY = os.environ["THUNDERBIT_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


class OutOfCredits(Exception):
    """402 — hard stop, escalate to a human."""


def _post(path, body, attempts=5):
    for i in range(attempts):
        r = requests.post(f"{BASE}{path}", headers=HEADERS, json=body, timeout=140)
        if r.status_code == 401:
            raise RuntimeError("401 invalid/missing API key — never retry")
        if r.status_code == 402:
            raise OutOfCredits("402 out of credits — top up at https://thunderbit.com/billing")
        if r.status_code == 429 or r.status_code >= 500:
            time.sleep(min(60, 2 ** i) + random.uniform(0, 1))
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"Gave up on {path}")


def _get(path, params=None):
    r = requests.get(f"{BASE}{path}", headers=HEADERS, params=params or {}, timeout=60)
    r.raise_for_status()
    return r.json()


def batch_distill(urls):
    """Distill up to 100 URLs/job and return {url: markdown}. 1 credit/URL."""
    out = {}
    for start in range(0, len(urls), 100):
        chunk = urls[start:start + 100]
        job = _post("/openapi/v1/batch/distill", {"urls": chunk, "timeout": 60000})["data"]
        job_id = job["id"]
        while True:  # polling is free
            status = _get(f"/openapi/v1/batch/distill/{job_id}",
                          {"page": 0, "pageSize": 100})["data"]
            if status["status"] in ("completed", "failed", "cancelled"):
                break
            time.sleep(3)
        for result in status.get("results", []):
            if result.get("status") != "completed":
                print(f"  ! distill failed: {result['url']} ({result.get('error')})")
                continue
            payload = result.get("data") or {}
            out[result["url"]] = payload.get("markdown", "")
    return out


# --- LLM stubs: swap for your provider --------------------------------------

def llm_seed_sources(goal):
    """STUB — ask the model for PUBLIC source URLs relevant to `goal`.
    Replace with a real call; here we hard-code an illustrative seed list.
    In production, validate each URL with is_allowed_url() below before fetching."""
    return [
        "https://example.com/report-2024",
        "https://example.org/industry-overview",
    ]


def llm_synthesize(goal, sources):
    """STUB — feed {url: markdown} to the model and ask for an answer that
    cites each claim with its source URL. Replace with a real call."""
    cited = "\n".join(f"- [{u}] ({len(md)} chars of context)" for u, md in sources.items())
    return f"[stubbed answer to: {goal}]\nSources used:\n{cited}"


def research(goal):
    urls = [u for u in llm_seed_sources(goal) if is_allowed_url(u)]
    if not urls:
        return "No allowed public sources found for this goal."
    print(f"Distilling {len(urls)} sources...")
    sources = batch_distill(urls)            # cheap: Markdown, not structured extract
    return llm_synthesize(goal, sources)


if __name__ == "__main__":
    try:
        print(research("What are the headline findings of the 2024 report?"))
    except OutOfCredits as e:
        print(f"HARD STOP: {e}")  # wire to your alerting
        raise
```

The lead-agent variant follows the list→detail pattern instead: `extract` a public directory page into a list of companies (with detail URLs), then loop — for each company, `distill` its public site for context the model enriches, or `extract` a tight schema (name, public email, location) when you need structured fields. Use batch for the detail pass so 80 companies are one job, not 80 calls.

Notice the research agent leans on **distill, not extract** — reading and synthesizing is exactly the distill use case, and it's 20× cheaper across many sources.

---

## Cost control in autonomous loops

An autonomous agent can burn credits fast: a single confused loop that calls `extract` on every link it sees can spend thousands of credits before you notice. Defend in depth.

- **Cap tool calls per run.** The `max_tool_calls` parameter in `run_agent` is a hard ceiling — the loop stops regardless of what the model wants.
- **Prefer cheap tools first.** Steer the model (in the system prompt) toward `suggest_fields` and `distill` before `extract`. Discovery and reading are 1 credit; structured extraction is 20.
- **Human-in-the-loop before big batches.** A `batch_extract_create` over 100 URLs is 2,000 credits. Require explicit confirmation before any batch extract above a threshold.
- **Track spend and set a per-run budget.** Charge each call against a running budget and refuse the call when it would blow the cap.

A simple budget guard you wrap around the executor:

```python
COSTS = {  # credits per call, from the reference
    "thunderbit_suggest_fields": 1,
    "thunderbit_distill": 1,
    "thunderbit_extract": 20,
}


class BudgetExceeded(Exception):
    pass


class Budget:
    def __init__(self, limit, confirm_batch_over=200):
        self.limit = limit
        self.spent = 0
        self.confirm_batch_over = confirm_batch_over

    def charge(self, name, args):
        # Batch cost scales by URL count; per-URL price matches the single-call price.
        n_urls = len(args.get("urls", [])) or 1
        per_call = COSTS.get(name.replace("batch_", "").replace("_create", ""), 20)
        cost = per_call * n_urls
        if name.endswith("_create") and cost >= self.confirm_batch_over:
            if not human_confirms(f"{name} will cost ~{cost} credits. Proceed?"):
                raise BudgetExceeded(f"human declined {name} (~{cost} credits)")
        if self.spent + cost > self.limit:
            raise BudgetExceeded(f"{name} (~{cost}) would exceed budget {self.limit}")
        self.spent += cost
        return cost


def human_confirms(prompt):  # replace with your real confirmation channel
    return input(f"{prompt} [y/N] ").strip().lower() == "y"
```

Call `budget.charge(name, args)` inside `execute_tool` before running the tool; on `BudgetExceeded`, return the message as a tool-error string so the model knows to stop or ask for a higher budget rather than crash blindly.

---

## Error handling inside the agent

The reference defines three terminal HTTP errors, and each maps to a different agent behavior. The principle: **handle transport errors in code (outside the model); surface decision errors to the model as readable text.**

- **`402` out of credits — surface to the model.** This is a decision the agent should make consciously: stop, report that it's out of credits, and ask a human to top up at <https://thunderbit.com/billing>. Return it as a concise tool-error string (e.g. `TOOL_ERROR(out_of_credits): ...`) so the model can react gracefully instead of the process crashing.
- **`429` rate-limited — retry in code, never the model.** Back off with exponential backoff + jitter *inside* `_post`, transparent to the model. Don't burn a model turn (and tokens) deciding to retry a rate limit.
- **`401` invalid key — never retry, fail loud.** A bad key won't fix itself. Raise immediately; this is an operator problem, not something the agent can route around.

Keep tool-error strings short and literal — `TOOL_ERROR: timeout, try renderMode:"full" and a larger waitFor` is something the model can actually act on; a stack trace is not. The `execute_tool` function above already returns errors as strings rather than raising, which is exactly what lets the model read and respond to them.

---

## Guardrails the agent must respect

An autonomous agent will follow links you didn't anticipate, so guardrails must live in **both** the system prompt (so the model self-restricts) and in **code** (so a confused model can't override them).

System-prompt rules (used as `SYSTEM_PROMPT` in the loop above):

```python
SYSTEM_PROMPT = """You are a web-research agent with Thunderbit tools.

RULES — never violate these:
- PUBLIC DATA ONLY. Only fetch publicly accessible pages.
- NEVER target login-walled platforms (Facebook, Instagram, LinkedIn
  profiles/company data) or any URL behind authentication, /login, /account,
  /admin, or a paywall.
- Respect robots.txt and the site's Terms of Service.
- Prefer DISTILL (1 credit) for reading; use EXTRACT (20 credits) only when you
  need structured fields for a downstream action.
- Call suggest_fields-style reasoning to pick fields BEFORE extracting.
- Do not collect personal data without a lawful basis; prefer company-level,
  non-personal, public data.
"""
```

A code-level check that refuses obviously private or login-walled URLs — belt and suspenders, because the model will sometimes ignore the prompt:

```python
from urllib.parse import urlparse

BLOCKED_HOSTS = {"facebook.com", "instagram.com", "linkedin.com"}
BLOCKED_PATH_HINTS = ("/login", "/signin", "/account", "/admin", "/checkout", "/cart")


def is_allowed_url(url):
    """Refuse obviously private / login-walled / non-public URLs."""
    try:
        p = urlparse(url)
    except Exception:
        return False
    if p.scheme not in ("http", "https"):
        return False
    host = (p.netloc or "").lower().lstrip("www.")
    if any(host == b or host.endswith("." + b) for b in BLOCKED_HOSTS):
        return False
    if any(hint in p.path.lower() for hint in BLOCKED_PATH_HINTS):
        return False
    return True
```

Gate every URL through `is_allowed_url()` inside `execute_tool` before any `distill`/`extract` runs. The research agent above already filters its seed list this way.

---

## Responsible use

These guardrails apply to every workflow in this repo — autonomous agents most of all, because they act without a human reviewing each call.

- **Public sources only.** Thunderbit targets publicly accessible pages. Never point the agent at login-walled platforms (Facebook, Instagram, LinkedIn profile/company data) or private/authenticated areas. Public review sites, directories, news, marketplaces, public company sites, and public job boards are fair game.
- **Respect `robots.txt` and Terms of Service** for every origin the agent touches. Honor disallowed paths and crawl-rate directives.
- **Personal-data caution.** If a schema collects names, emails, or phone numbers, only do so when publicly posted and only with a lawful basis to process them (GDPR/CCPA). Prefer company-level, non-personal data; drop personal fields you don't need.
- **Cost caps and oversight.** An autonomous loop with no budget cap is a way to spend money you didn't intend to. Always set a per-run budget, cap tool calls, and require human confirmation before large batch extracts.
- **Throttle and cache.** Use batch with reasonable concurrency, don't hammer a single origin, and dedupe so the agent doesn't re-scrape unchanged pages — it saves credits and reduces load on the source.

The Open API is the programmatic surface of the same engine behind the Thunderbit AI Web Scraper Chrome extension; an account is required to obtain a key (a free tier is available — new accounts get a one-time allotment of roughly 600 units, enough to prototype distill and extract before topping up at <https://thunderbit.com/billing>). This is not an anonymous tool.

---

## Troubleshooting & FAQ

**MCP tools don't appear in my assistant.** The model can't see tools that aren't registered. Confirm the `mcpServers` block is in the right config file for your host, that the JSON is valid, and **restart the host** — most MCP clients only load servers at startup. In Claude Code, verify the plugin installed (`claude plugin add thunderbit`) and that the marketplace was added first.

**Key not picked up / 401 even though the tools appear.** The server reads `THUNDERBIT_API_KEY` from its environment. For MCP, the key goes in the `env` block of the server config (not your shell). For the Claude Code plugin and CLI, `export THUNDERBIT_API_KEY=tb_...` in the shell that launches them. Resolution order is explicit `apiKey` arg → env var → error, so a stray `apiKey: ""` argument can shadow a good env var.

**The model keeps calling `extract` when `distill` would do.** Two fixes, use both: tighten the system prompt's distill-vs-extract rule, and add the budget guard so an over-eager `extract` is refused (or flagged for confirmation) before it spends 20 credits. The tool *descriptions* in your function schema also steer the model — make `extract`'s description explicitly say "expensive; only for structured fields."

**Empty results from a tool.** The page is almost certainly client-rendered. Have the agent retry with `renderMode:"full"` (headless browser) and a larger `waitFor` (up to its `10000` ms ceiling) so lazy/SPA content loads. Returning the error string `TOOL_ERROR: empty result, retry with renderMode:"full"` lets the model self-correct on the next turn.

**Polling a batch job in a loop.** `thunderbit_batch_*_status` (and the `GET /batch/*` endpoints) are **free**, so poll every ~3 seconds without worrying about cost. Don't make the *model* poll in a loop — that wastes model turns and tokens; poll in your code (as `batch_distill` does) and hand the model only the finished result.

**A single page is too slow for a synchronous call.** For one heavy SPA page outside a batch, use the async endpoints: `POST /openapi/v1/async/extract` (or `async/distill`), then poll `GET /openapi/v1/async/extract/{jobId}` every ~3s until `status` is `completed`. Polling is free.

**`401` / `402` / `429` recap.** `401` = bad or missing key (never retry; fix the key). `402` = out of credits (hard stop, surface to the model, top up at <https://thunderbit.com/billing>). `429` = rate-limited (back off with exponential backoff + jitter in code). See the [API reference](docs/thunderbit-api-reference.md) §4.
