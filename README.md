# thunderbit-use-cases

**Six end-to-end tutorials for building real workflows on Thunderbit's web-data platform — via the HTTP API, the MCP server, and the CLI/SDK.**

Thunderbit turns any public web page into clean Markdown or structured JSON. The same
engine is reachable three ways:

- **HTTP API** (`https://openapi.thunderbit.com`) — any language, servers, cron, CI, webhooks
- **MCP server** (`@thunderbit/mcp-server`) — AI assistants: Claude Desktop/Code, Cursor, Cline, ChatGPT
- **CLI + SDK** (`@thunderbit/thunderbit-cli`) — shell pipelines and programmatic scripts

Each tutorial below is a complete, runnable guide: the data problem, the schema, code for
all three surfaces, an end-to-end pipeline, the no-code Chrome-extension flow, automation,
cost math, and responsible-use guardrails. Sixteen guides in total — fourteen industry
playbooks plus two developer-integration guides (building apps and AI agents on the API).

---

## Start here

**[docs/thunderbit-api-reference.md](docs/thunderbit-api-reference.md)** — the canonical
reference: every endpoint, request/response shape, the extract schema format, MCP tools,
CLI commands, pricing, and error handling. Every code sample in this repo conforms to it.

```bash
# 1. Get an API key at https://app.thunderbit.com/console/api-keys  (keys are prefixed tb_)
export THUNDERBIT_API_KEY=tb_your_api_key_here

# 2. Pick a surface
npm i -g @thunderbit/thunderbit-cli        # CLI + SDK
npx -y @thunderbit/mcp-server              # MCP server (add to your AI client config)

# 3. Smoke-test it
thunderbit distill https://example.com -f markdown
```

---

## Industry playbooks

| # | Industry | What you'll build |
|---|----------|-------------------|
| 1 | [Real Estate](01-real-estate.md) | Self-updating comps database, daily new-listing & price-drop diff, neighborhood roll-ups |
| 2 | [E-commerce Operations](02-ecommerce-operations.md) | Daily price & competitor monitoring, assortment-gap analysis, review mining, stockout alerts |
| 3 | [B2B Sales](03-b2b-sales.md) | Target-account lists & firmographic enrichment from public directories and company sites + funding-trigger monitoring |
| 4 | [Social Media Operations](04-social-media-operations.md) | Influencer discovery, public content-performance tracking, open-web brand-mention monitoring |
| 5 | [Financial Research](05-financial-research.md) | Fundamentals datasets, a filings/transcripts RAG corpus, news-sentiment monitoring, watchlist tracking |
| 6 | [Recruiting / HR](06-recruiting-hr.md) | Job-market intelligence, competitor hiring monitor, compliant public-portfolio sourcing |
| 7 | [RAG Knowledge Base](07-rag-knowledge-base.md) | Turn any public docs site into a citable RAG corpus via `distill` — enumerate, batch-distill, chunk, embed, retrieve, incremental refresh |
| 8 | [Travel & Hospitality](08-travel-hospitality.md) | Rate & availability monitoring across public OTA pages, sellout alerts, review-sentiment digests |
| 9 | [Academic & Research](09-academic-research.md) | Literature datasets, a paper RAG corpus, new-paper & citation tracking (arXiv/PubMed/Semantic Scholar) |
| 10 | [News & Media Monitoring](10-news-media-monitoring.md) | Article aggregation, LLM sentiment/entity tagging, coverage-trend & spike alerting |
| 11 | [Local Business](11-local-business.md) | Local-business datasets, NAP-consistency auditing across sources, review monitoring |
| 12 | [Government & Public Data](12-government-public-data.md) | Tender/RFP & grant monitoring, solicitation summarization, new-opportunity alerts with deadlines |
| 13 | [Healthcare & Pharma](13-healthcare-pharma.md) | Clinical-trial datasets, new-trial/recall/label monitoring, protocol RAG — public aggregate data only, not medical advice |
| 14 | [Manufacturing & Supply Chain](14-manufacturing-supply-chain.md) | Supplier discovery, spec-sheet extraction, price & lead-time monitoring |

## Developer-integration guides

| # | Guide | What you'll learn |
|---|-------|-------------------|
| 15 | [App Development](15-app-development.md) | Build a production feature on the API: server-side key handling, SSRF-safe URL validation, caching, async+webhooks, retries, queues, cost controls (Express/Next.js/FastAPI) |
| 16 | [AI Agent Development](16-ai-agent-development.md) | Give agents web data via the MCP server or HTTP-API function-calling tools; agent loops, distill-vs-extract rules, budget guards, guardrails |

Every guide covers the same three primitives applied to its domain:

- **`suggest_fields`** (1 credit) — let the AI propose an extraction schema.
- **`distill`** (1 credit) — page → clean, LLM-ready Markdown (great for RAG).
- **`extract`** (20 credits) — page → structured JSON via your schema.

…plus **batch** (up to 100 URLs/job) and **async** variants. See the reference for the full map.

---

## How the tutorials are organized

Each article follows the same arc so you can jump between industries without relearning the shape:

1. The data problem in this industry
2. What you'll build
3. Prerequisites
4. Three ways to call Thunderbit (MCP / HTTP / CLI+SDK)
5. Discover the schema → extract the list → batch-extract details
6. A single cohesive, runnable end-to-end pipeline
7. The no-code Chrome-extension flow (the same workflow for non-developers)
8. Automation (cron, webhooks), cost & scale math, responsible use, troubleshooting

---

## Responsible use (applies to every tutorial)

- **Public data only.** These guides target publicly accessible pages. They do **not** build
  workflows against login-walled platforms (Facebook, Instagram, LinkedIn data, private/
  authenticated areas). Public directories, marketplaces, review sites, news, and company
  sites are fair game.
- **Respect `robots.txt` and Terms of Service** for every site you target.
- **Throttle** and use batch with reasonable concurrency — don't hammer a single origin.
- **Minimize personal data.** Prefer company-level and public, non-personal data; where
  personal data is unavoidable, ensure a lawful basis (GDPR/CCPA) and collect the minimum.
- **Cache and dedupe** to avoid re-scraping unchanged pages — it saves credits and load.

---

## License

MIT
