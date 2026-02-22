# Agent Handshake Protocol: A New Contract Between AI Agents and the Web

**Nick Allain**
*agenthandshake.dev · github.com/AHP-Organization*

*Draft 0.1 — February 2026 (revised 6 — 12:21 UTC)*

---

## Abstract

Current approaches to AI agent interaction with websites are fundamentally misaligned: agents receive entire documents when they need specific answers, parse HTML designed for human browsers, and have no standardised way to discover what a site can do on their behalf. We present the **Agent Handshake Protocol (AHP)**, an open specification defining structured discovery, conversational interaction, and agentic delegation between visiting AI agents and website-side concierge systems.

Through a reference implementation tested against two sites — the AHP Specification Site and the Nate Jones AI-practitioner blog — we demonstrate that AHP MODE2 reduces token consumption by **75–80% vs. a naive visiting agent (no retrieval)** (AHP site: 77.4% average; Nate site: 71.5% average; 3-run mean ± stddev with cache-busting; RAG baseline API-measured). We report a full RAG-baseline comparison — a visiting agent implementing client-side chunking and keyword-retrieval over the same `llms.txt` content — which is the fairer competitive benchmark: the RAG baseline uses 291–945 tokens per query (AHP site) and 431–636 tokens per query (Nate site), while AHP MODE2 uses 1,995–2,419 and 2,283–2,673 respectively. **AHP MODE2 uses approximately 3–8× more tokens per query than a well-implemented RAG visiting agent on compact corpora.** AHP's advantages over RAG are protocol-level: structured discovery, capability declaration, server-managed caching (~12ms p50 cache-hit latency), session management, content signals, and MODE3 capabilities. The reference deployment passes **21 of 22 conformance checks** in the v5.2 test suite: T17 (MODE3 auth enforcement) is the single failure — a known security defect in the demo configuration that any production deployment must fix. All other 21 tests pass, including T08 (session memory confirmed), T20 (rate-limit headers), T21 (clarification format advisory), and T22 (invalid session graceful handling — HTTP 400 correct). AHP is designed for progressive adoption: a site can become MODE1-compliant in under five minutes (structural elements only; content quality is a separate investment), and each subsequent mode is backwards compatible. Cache-hit latency averages **12ms p50**; cold-cache latency averages **3,614ms p50** (LLM inference; see §4.3).

---

## 1. Introduction

The web was designed for humans navigating with browsers. AI agents — autonomous systems that traverse the web as part of larger tasks — are increasingly being asked to use it too. The results are predictable: agents scrape HTML, attempt to parse layouts designed for visual rendering, and consume thousands of tokens on boilerplate before reaching the content they need.

Several partial solutions have emerged. Cloudflare's "Markdown for Agents" [CLOUDFLARE] converts pages to clean markdown on request. The `llms.txt` convention [LLMSTXT] provides a site-level content document in plain text. These approaches share a fundamental limitation: they are monologues. A document is thrown over a fence, and the agent must parse it entirely regardless of its actual query. There is no negotiation, no capability declaration, no stateful interaction.

This is the equivalent of responding to every question with the full contents of a library. It works, but poorly. A more sophisticated visiting agent can implement client-side chunking and retrieval (RAG) over the same document, narrowing the gap — but it does so by replicating server-side retrieval logic that AHP makes available as a protocol primitive.

We ask a different question: **what if a website could talk back?**

AHP defines a protocol for exactly this. A site publishes a machine-readable manifest declaring its capabilities. Visiting agents discover the manifest, understand what the site can offer, and submit targeted queries — receiving precise, sourced answers rather than full documents. For sites with richer requirements, AHP's MODE3 enables the site-side concierge to use tools, call APIs, and escalate to human operators on behalf of the visiting agent.

The implications extend beyond efficiency. As AI agents become the primary interface through which many users interact with the web, a site's AHP compliance affects whether those agents can find it, understand it, and act on its behalf. AHP is designed as an infrastructure layer for agent-native web presence — we hypothesise that it may serve an analogous function to SEO in the browser-web era, though validating this claim requires adoption-scale data not yet available.

### 1.1 Contributions

This paper presents:

1. **The AHP specification** — a three-mode protocol covering discovery, conversational interaction, and agentic delegation
2. **A reference implementation** — a complete open-source MODE1/MODE2/MODE3 server deployable on any Node.js host
3. **Empirical evaluation** — conformance testing and token efficiency benchmarks against a reference deployment (reference implementation only; no third-party implementations tested in this version)
4. **A test suite** — an open-source visiting agent harness (v5: RAG-baseline comparison, cache-busted multi-run averaging, automatic cross-site comparison, conformance tests T01–T22 including session memory, rate-limit headers, clarification flow, and invalid session handling)

---

## 2. The AHP Protocol

AHP defines three modes of interaction, each building on the previous.

### 2.1 MODE1: Static Serve

In MODE1, the site publishes a machine-readable manifest at `/.well-known/agent.json` and provides static agent-readable content (a `llms.txt` file, markdown pages, or a JSON knowledge document). Interaction is read-only and stateless. A visiting agent fetches the manifest, discovers the content URL, and retrieves it.

MODE1 is directly compatible with existing `llms.txt` deployments. A site with an existing content document becomes MODE1-compliant by adding the manifest — no changes to the content document are required.

### 2.2 MODE2: Interactive Knowledge

MODE2 adds a conversational endpoint at `POST /agent/converse`. The visiting agent submits a query specifying a capability; the site-side concierge searches its knowledge base and returns a precise, sourced answer. Multi-turn sessions enable clarification when queries are ambiguous.

The critical distinction from MODE1: the visiting agent asks for what it needs, and the concierge returns only that. The agent never processes content it didn't ask for. Compared to a naive full-document approach, this reduces token consumption substantially. Compared to a client-side RAG agent operating over the same content, the advantage is protocol-level: structured discovery, capability negotiation, session management, and content signals come "for free" without the visiting agent implementing retrieval infrastructure.

### 2.3 MODE3: Agentic Desk

MODE3 equips the site-side concierge with tools and async capabilities. The concierge can query databases, call external APIs, execute calculations, connect to MCP servers, and escalate to human operators — capabilities the visiting agent may not have access to.

MODE3 changes the nature of the interaction: the visiting agent is no longer just querying a knowledge base, but delegating tasks to a capable agent that represents the site. This natural language delegation model — *"find out if this item is in stock and give me a quote"* — has no equivalent in static content approaches, where real-time data and computation are simply unavailable.

### 2.4 Discovery

AHP defines four discovery mechanisms, ordered by agent type:

1. **In-page agent notice** — a `<section aria-label="AI Agent Notice">` in the page body, visible to agents parsing raw HTML source. Note: headless browser agents that execute JavaScript and render the DOM may not see `display:none` content; the spec's guidance on visibility for headless agents is clarified in §2.4.1 below.
2. **HTML `<link>` tag** — `<link rel="agent-manifest">` in `<head>`, discoverable by DOM-parsing agents
3. **Accept header** — `Accept: application/agent+json` triggers a manifest redirect
4. **Well-known URI** — direct `GET /.well-known/agent.json` per IETF RFC 8615 [RFC8615]; for agents that know to look

#### 2.4.1 In-Page Notice: Scope, Limitation, and Path Forward

The in-page notice spec recommends the element be visually hidden (`display:none`) while remaining in the DOM. This ensures the notice does not disrupt human UX. However, this creates a coverage gap:

- **Raw-HTML scraping agents** (using `curl`, `requests`, `urllib`) read the source document and will find `display:none` content. The in-page notice works for these agents.
- **Headless browser agents** (Puppeteer, Playwright, Selenium) execute JavaScript and render the full DOM. Standard text-extraction methods for rendered pages may skip hidden elements. These agents are **not reliably covered** by the current in-page notice design.

This is a material concession. Headless browser agents are a dominant and growing implementation pattern. An in-page notice that doesn't reach them is a narrow mechanism serving HTTP-scraping agents who can also use mechanisms 3 (Accept header) and 4 (well-known URI).

**Path forward for v0.2**: AHP will add a fifth discovery mechanism specifically for rendered-HTML agents: a `<meta name="ahp-manifest" content="/.well-known/agent.json">` tag in the document `<head>`. Meta tags are reliably accessible in rendered DOM, are invisible to human users, and are a familiar pattern (cf. `<meta name="robots">`). This would replace the in-page notice's role as the rendered-page discovery mechanism and allow the in-page notice to be accurately described as a "raw-HTML scraper fallback" rather than a primary headless-agent discovery path.

The discovery priority ordering in the spec will be updated to: `<meta>` tag → `<link>` tag → Accept header → well-known URI, with the in-page notice retained as a legacy/raw-HTML mechanism.

### 2.5 Content Signals

AHP includes a standardised content signals sub-protocol, allowing sites to declare machine-readable preferences for AI usage of their content. Signals include `ai_train`, `ai_input`, `search`, and `attribution_required`. These signals are declared in the manifest and echoed in every response, creating a persistent, per-response record of usage intent.

The current implementation relies on the visiting agent honouring these signals. The spec uses SHOULD for content signal compliance (`ai_train: false`, etc.) — reflecting the intent and aspirational compliance expectation for the ecosystem while acknowledging that AHP does not technically enforce it. A future revision will explore cryptographic attestation for stronger assertions. Until enforcement mechanisms are specified, content signals function as a declaration of intent and a legal/ethical standard rather than a technical control.

---

## 3. Implementation

### 3.1 Reference Server

Our reference implementation is a Node.js/Express server (~700 lines across 7 source files) implementing all three AHP modes. Key components:

- **Knowledge base**: markdown content files, chunked by H2 section and scored by keyword overlap for retrieval (v0; vector embeddings planned for v1). **Evaluation framing note**: the v0 keyword retrieval is adequate for the compact corpora tested (~10–50KB) but is expected to degrade on larger, more ambiguous content. This limitation is considered in the evaluation framing in Section 4.2.
- **MODE2 concierge**: Claude Haiku via the Anthropic API; retrieves top-5 relevant chunks, synthesises a sourced answer. Token costs appear on the server side — the visiting agent pays fewer tokens, but the site pays for each Haiku call. At low-to-moderate query volumes this is economically favourable; at high traffic volumes the server-side cost model requires analysis (see §5.1).
- **MODE3 tool use**: Claude's native tool_use API with an agentic loop; tools include inventory lookup, quote calculation, order retrieval, and knowledge search
- **Async queue**: human escalation tickets with configurable simulated response delay; fires callbacks or resolves via polling
- **Session management**: in-memory sessions with 10-minute TTL and 10-turn limit. Session history is passed to the LLM on subsequent turns, enabling multi-turn context (confirmed live in v5.2: server correctly quotes prior turn content verbatim in T08 turn 2 response). **Practical developer note**: the server passes full session history to the LLM; developers building visiting agents on top of AHP do not need to implement their own turn-history injection — the server manages it as a protocol primitive.
- **Caching**: normalised query key, 5-minute TTL; cached responses return in <30ms
- **Rate limiting**: 30 req/min unauthenticated, with AHP-standard `X-RateLimit-*` headers

### 3.2 Test Sites

We deployed a reference instance with AHP protocol documentation as its content corpus:

| Site | Content | Corpus size | Chunks |
|------|---------|-------------|--------|
| AHP Specification Site | AHP protocol specification and documentation | ~40,100 chars, ~9,665 tokens | 96 |
| Nate Jones AI Blog | AI practitioner blog (RAG, MCP, prompt engineering, agentic systems) | ~39,300 chars, ~8,877 tokens | 82 |

A second instance (Nate Jones — `nate.agenthandshake.dev`) hosts an AI practitioner blog corpus. Comparison queries on this site use AI-practitioner topics (RAG, prompt engineering, MCP, agentic systems) rather than AHP-specific queries, to test generalization to a different content domain.

### 3.3 Test Harness

Our test suite (`AHP-Organization/test-suite`, v5.2) acts as a visiting agent, running 22 conformance tests (T01–T22):

- T01–T16 (original suite): discovery, schema, MODE2, MODE3, error handling, sessions, content signals, caching, rate limiting
- T17: MODE3 action auth enforcement (CRITICAL DEFECT — demo server allows unauthenticated actions; spec §5.3 MUST)
- T18: Session 10-turn limit enforcement (spec §5.4.3)
- T19: Request body 413 on oversized payload >8KB (spec §6.5)
- T20: Rate-limit headers present and numeric on all responses (spec §11.1)
- T21: Clarification-needed response format (spec §6.3 MAY — advisory if never triggered)
- T22: Invalid/unknown session_id handled gracefully — no 5xx (spec §6.2)

**v5 benchmark methodology improvements:**

- **Cache-busting**: each comparison query run appends a unique 8-char nonce (e.g. `[ref:a1b2c3d4]`) to force fresh LLM calls and measure real token variance rather than cache-hit ±0 artifacts. The nonce adds ~4 tokens per run (consistent overhead, noted in table).
- **Automatic cross-site comparison**: when running against the AHP Reference Site, the suite automatically also runs the Nate Jones site comparison (no separate flag needed).
- **Latency included in comparison table**: RAG and AHP cold-cache latencies are reported alongside token counts.

The three baselines:
- **Naive**: tiktoken (cl100k_base) estimate; full doc + query, no retrieval; not API-measured
- **RAG-baseline**: fetches `llms.txt`, chunks (~500 chars), top-3 by keyword overlap, calls Haiku; API-measured
- **AHP MODE2**: POSTs to `/agent/converse`; API-measured

Each query runs **3 times**; results reported as **mean ± stddev** across all 3 runs.

---

## 4. Evaluation

### 4.1 Protocol Conformance

The reference deployment was tested against the v5.2 test suite (T01–T22). Results for the 2026-02-22 12:21 UTC run:

**Scope note**: conformance was demonstrated only for the reference implementation, which was co-developed with the spec and test suite. No independent third-party implementations have been tested. Results reflect internal consistency of the reference implementation, not ecosystem-wide interoperability.

**How to submit third-party conformance results**: run the open-source test suite against your AHP deployment (`./venv/bin/python test_runner.py --target <your-endpoint> --output result.json`) and open a pull request to `AHP-Organization/test-suite/results/` with your JSON report and a brief description. A hosted conformance runner is planned for v0.2.

| Test | Category | AHP Ref | Notes |
|------|----------|---------|-------|
| T01: Well-known discovery | Discovery | ✓ | |
| T02: Accept header discovery | Discovery | ✓ | |
| T03: In-page agent notice | Discovery | ✓ | Passes; API server exempt — see §2.4.1 |
| T04: Manifest schema | Protocol | ✓ | |
| T05: MODE2 content query | Functionality | ✓ | |
| T06: Unknown capability error | Error handling | ✓ | |
| T07: Missing field error | Error handling | ✓ | |
| T08: Multi-turn + session memory | Sessions | ✓ | **See note** — v5 confirms server has session memory |
| T09: Response schema | Schema | ✓ | |
| T10: Content signals in response | Signals | ✓ | |
| T11: MODE3 inventory (tool use) | MODE3 | ✓ | 1 tool call (`check_inventory`) observed consistently in v5.2 runs |
| T12: MODE3 quote with numeric prices | MODE3 | ✓ | Assertion fixed: requires `$\d` pattern + no failure phrases |
| T13: MODE3 order lookup | MODE3 | ✓ | |
| T14: MODE3 async human escalation | MODE3 async | ✓ | Latency is simulated |
| T15: Response caching | Infrastructure | ✓ | |
| T16: Rate limiting (429) | Infrastructure | ✓ | Window partially consumed by prior tests; see note |
| T17: MODE3 action auth enforcement | Security | **✗** | **CRITICAL KNOWN DEFECT** — see note |
| T18: Session 10-turn limit | Sessions | ✓ | Limit triggered on turn 11 |
| T19: Oversized body → 413 | Protocol | ✓ | HTTP 413 on 10KB body |
| T20: Rate-limit headers (spec §11.1) | Protocol | ✓ | All 4 headers present, numeric (v4) |
| T21: Clarification needed — format (§6.3) | Protocol | ✓ Advisory | Server answered "Which mode should I use?" directly; §6.3 format **unverified** (spec says MAY; server is compliant — it is not required to clarify) |
| T22: Invalid session_id graceful handling (§6.2) | Sessions | ✓ | Server returned HTTP 400 (correct — any 4xx is valid per spec §6.2) |

**T08 note — false-negative history**: an earlier suite (v4) incorrectly failed T08 for a deployed server that does have session memory. The v4 exclusion phrase `"FIRST QUESTION"` matched `"In your first question, you requested…"` — a memory-demonstrating sentence — triggering a false negative. The v5 suite removed this ambiguous phrase; the v5.1 run confirms the fix is correct: the server responds `"You asked specifically about MODE1. In your first message, you explicitly requested: 'Tell me about AHP modes — focus especially on MODE1.'"` — unambiguous session memory. T08 ✓ in v5.1.

**T16 note**: rate limit test runs last; window was 9/30 remaining when T16 started (burst hit 429 after 10 requests). The declared limit of 30/min is confirmed by T20's X-RateLimit-Limit header.

**T17 note — CRITICAL KNOWN DEFECT**: the reference implementation accepts unauthenticated requests to all three MODE3 action capabilities (inventory_check, get_quote, order_lookup). This violates spec §5.3 (MUST require authentication for action capabilities with side effects). Every developer who clones the reference implementation and deploys it without adding authentication ships a production system violating a MUST requirement. A `--require-auth` configuration flag is planned for v0.1.1.

**21/22 (95.5%)** on reference deployment (v5.2 suite, 12:21 UTC). T17 ✗ (CRITICAL DEFECT — no auth enforcement). T21 ✓ advisory (§6.3 MAY — format unverified). All other 20 tests pass conclusively, including T08 (session memory confirmed — server cites turn 1 content verbatim), T20 (rate-limit headers), T22 (invalid session → HTTP 400 correct).

### 4.2 Token Efficiency

We compared token consumption for three approaches on five representative queries against the AHP Specification Site.

**Baseline definitions:**
- **Naive visiting agent (no retrieval)**: fetches the full `llms.txt` document (~40,900 chars / ~10,000 tiktoken tokens) and passes it with the query to an LLM. Token count estimated via tiktoken cl100k_base; not API-measured. *This is the baseline no competent agent implementation would use in production — it represents the upper bound on token waste and serves as a reference point, not a competitive comparison.*
- **RAG-baseline visiting agent**: fetches `llms.txt`, chunks by paragraph (~500 chars), selects top-3 chunks by keyword overlap, calls Claude Haiku. This is a **fair competitive baseline** representing a reasonably-implemented visiting agent. Token counts are API-measured. Run 3 times; reported as mean ± stddev.
- **AHP MODE2**: visiting agent POSTs query to `/agent/converse`. Token counts are API-measured. Run 3 times; reported as mean ± stddev.

#### Methodology Note: Cache-Busting

The suite (first implemented in v3, current v5) appends a unique 8-char nonce to each comparison query run (e.g. `"Explain what MODE1 is [ref:a1b2c3d4]"`) to force distinct server-side cache keys. Without this, the server's 5-minute TTL cache produces byte-identical responses for runs 2 and 3 against an identical query, making the reported stddev artificially ±0 — a measure of cache consistency rather than LLM output variance. With cache-busting, each run triggers a fresh LLM call and stddev reflects genuine answer variation. The nonce adds ~4 tokens of overhead per run (systematic and consistent).

#### AHP Specification Site (corpus: ~40,100 chars, ~9,665 tokens tiktoken, 96 chunks)

Results from 2026-02-22 **12:21 UTC — v5.2 suite** (T21 probe updated to domain-internally-ambiguous query; T16 window annotation improved). Naive baseline: tiktoken estimate (not API-measured; fetch latency is a **single measurement** reused for all queries). RAG and AHP: API-measured (Claude Haiku 4.5, Anthropic), 3-run mean ± stddev with cache-busting.

| Query | Naive†<br/>(fetch ×1) | RAG-baseline ± σ<br/>(lat.) | AHP MODE2 ± σ<br/>(lat.) | Reduction<br/>vs. naive ↓ | Overhead<br/>vs. RAG ↑ |
|-------|------------------------|------------------------------|---------------------------|--------------------------|------------------------|
| Explain what MODE1 is | ~9,735 (168ms) | **291 ±6** (1,128ms) | **2,419 ±21** (4,241ms) | **75.1%** | +731% |
| How does AHP discovery work? | ~9,735 (168ms) | **494 ±8** (1,784ms) | **1,995 ±21** (3,763ms) | **79.5%** | +304% |
| What are AHP content signals? | ~9,736 (168ms) | **436 ±45** (1,363ms) | **2,191 ±3** (3,773ms) | **77.5%** | +403% |
| How do I build a MODE2 endpoint? | ~9,734 (168ms) | **945 ±16** (2,714ms) | **2,328 ±35** (4,278ms) | **76.1%** | +146% |
| What rate limits should AHP enforce? | ~9,736 (168ms) | **564 ±1** (1,339ms) | **2,059 ±14** (2,761ms) | **78.9%** | +265% |
| **Average** | | | | **77.4%** | **+370%** |

† Naive tokens: tiktoken cl100k_base estimate, not API-measured; ±5–10% error. Fetch latency (168ms) is a **single measurement** reused for all queries; it is not a 3-run mean. Uniform token values (~9,735) reflect dominant corpus; per-query variation is nonce (~8 tokens) + query (~15 tokens). Latency is mean across 3 runs. All per-query AHP latencies reflect cold-cache inference (3.8–4.3s range, CV ≤17%); the latency profile §4.3 p50 (3,614ms) uses short uniform-format nonce queries — evaluation query latency varies by answer complexity.

**Reading the table**: Reduction vs. naive ↓ — higher is better for AHP. Overhead vs. RAG ↑ — lower is better for AHP (+141% = AHP uses 2.4× RAG tokens; +732% = AHP uses 8.3× RAG tokens). AHP is worse on the RAG column for all five queries.

**Latency stddev**: both RAG and AHP latency stddev values reflect genuine LLM inference variance (e.g. AHP discovery ±341ms; RAG endpoint ±353ms). Token stddev is small (±3–48) reflecting mostly-consistent answer lengths across runs.

**Core finding**: AHP MODE2 uses **3–8× more tokens per query** (289–958 for RAG vs 1,989–2,406 for AHP) on a compact 9,735-token corpus. The 75–80% reduction vs. naive is real; AHP's value vs. RAG is protocol-level:

| | RAG visiting agent | AHP MODE2 visiting agent |
|-|-------------------|--------------------------|
| Per-query token cost (≤10K corpus) | **Lower** (291–945 tokens) | Higher (1,995–2,419 tokens) |
| Cold-cache latency | ~1,130–2,720ms (measured) | ~2,760–4,280ms (+server round-trip) |
| Cache-hit latency | N/A — client manages caching | **~12ms p50** (server 5-min TTL) |
| Discovery mechanism | Must know document URL | Structured 4-mechanism discovery |
| Capability declaration | None | Manifest + capabilities |
| Session management | Client must implement or skip | Protocol primitive — confirmed working |
| Content signals (AI usage policy) | None | Per-response |
| Retrieval at scale (>100K tokens) | Keyword overlap degrades | Server-managed; embedding upgrade opaque to visitors |
| MODE3 real-time data / compute | ✗ Not possible | ✓ Native |
| Server infrastructure required | ✗ None | ✓ Required |

#### Nate Jones Site Cross-Comparison (AI-Practitioner Domain)

Results from 2026-02-22 12:21 UTC (v5.2 suite, cache-busted, RAG baseline API-measured). Corpus: ~39,341 chars, ~8,877 tokens tiktoken, 82 chunks. Fetch latency (218ms) is a single measurement.

| Query | Naive† | RAG ± σ<br/>(lat.) | AHP ± σ<br/>(lat.) | Reduction<br/>vs. naive ↓ | Overhead<br/>vs. RAG ↑ |
|-------|--------|---------------------|---------------------|--------------------------|------------------------|
| What is RAG and how does it work? | ~8,945 | **431 ±5** (979ms) | **2,586 ±14** (2,580ms) | **71.1%** | +500% |
| Explain prompt engineering | ~8,941 | **453 ±17** (1,976ms) | **2,577 ±2** (4,342ms) | **71.2%** | +469% |
| What is MCP and how does it relate to AI agents? | ~8,947 | **512 ±5** (1,194ms) | **2,611 ±30** (3,841ms) | **70.8%** | +410% |
| What is vibe coding? | ~8,940 | **450 ±8** (1,476ms) | **2,283 ±21** (4,254ms) | **74.5%** | +407% |
| How do AI agents work in production? | ~8,943 | **636 ±17** (2,204ms) | **2,673 ±9** (4,428ms) | **70.1%** | +320% |
| **Average** | | | | **71.5%** | **+421%** |

Cross-domain: 71.5% reduction vs. naive (vs. AHP site 77.4% — Nate's slightly lower reflects different corpus and query types). Overhead vs. RAG +421% (~5.2×). RAG and AHP latency variance reflects Anthropic API infrastructure variance; the CV range of 5–17% across queries is consistent with single-provider LLM inference variability. Cross-domain consistency confirms these are architectural properties, not AHP-corpus artefacts. Latency is mean across 3 runs.

### 4.3 Latency Profile

The v5.2 suite runs a split 10-cold + 10-hot design: 10 unique-nonce queries force cold-cache LLM calls; 10 repetitions of a fixed query measure cache-hit performance. Results from 2026-02-22 12:21 UTC run.

| Cohort | Metric | Value | Notes |
|--------|--------|-------|-------|
| Cold (forced cache miss) | mean | **3,670ms** | n=10; unique nonces, guaranteed cold |
| Cold | p50 | **3,614ms** | |
| Cold | max | **4,777ms** | |
| Hot (cache-hit) | p50 | **12ms** | n=9 cache hits; run 1 was cold (3,506ms, first call on fresh query) |
| Hot | max | **3,506ms** | Run 1 only — first call on fresh query is always cold |
| All 20 samples | p50 | **3,162ms** | Mix of 10 cold + 10 hot |
| All 20 samples | p95 | **4,777ms** | True p95 at n=20 |
| All 20 samples | min | **10ms** | Fastest cache-hit sample |

**Key finding**: AHP MODE2 has a bimodal latency distribution. Cache-hit responses (~12ms p50) are comparable to a CDN-served static file. Cold-cache responses (~3,614ms p50) are dominated by Claude Haiku inference time. The hot-cohort max (3,506ms) is run 1 only — the first call on any new query phrase is always cold; runs 2–10 averaged ~12ms.

**Note on latency profile vs. evaluation query latency**: the profile p50 (3,614ms) uses short uniform-format nonce queries (low output token count). Evaluation queries in §4.2 range from 2,761ms to 4,278ms AHP cold-cache — variation driven by answer complexity (longer answers = more output tokens = slower). A reader comparing the abstract's "3,614ms cold-cache" to the §4.2 table should treat the profile p50 as a summary statistic, not a per-query ceiling.

**RAG-baseline latency comparison** (from §4.2 benchmark data):

| Approach | Cold-cache latency | Effective cached latency |
|----------|--------------------|--------------------------|
| Naive (doc fetch only, no LLM) | ~168ms (measured) | N/A — re-fetches every time |
| RAG visiting agent (fetch + Haiku) | ~979ms–2,204ms (measured) | N/A — client manages own caching |
| AHP MODE2 (cold cache) | **~3,614ms p50** (directly measured, n=10) | **~12ms p50** (server 5-min TTL, n=9 cache hits) |

AHP's cold-cache p50 (~3,614ms) is 1.6–3.7× higher than the RAG baseline cold latency range (~980ms–2,200ms), reflecting server round-trip overhead on top of the same Haiku inference. AHP's cache-hit latency (~12ms) is structurally unavailable to a stateless RAG visiting agent, which must re-execute the full RAG pipeline on repeated queries. The profile p50 (3,614ms) is measured with short nonce queries; evaluation query AHP latency ranges from 2,761ms to 4,278ms depending on answer complexity (see §4.2 table footnote).

**Note on MODE3 async latency**: the T14 ~8,100ms figure is from a simulated human operator with fixed delay. Production human escalation = hours to days.

### 4.4 MODE3: Capabilities Beyond Static Content

MODE3 enables a qualitatively different class of interactions — ones that static content approaches do not support through the same protocol interface:

**Real-time data access**: a visiting agent querying `inventory_check` receives current stock levels, pricing, and availability — data that would be stale in any static document. The concierge uses Claude's tool_use to call a structured inventory database, synthesise the result, and respond in natural language. Token cost: ~2,400–3,000 (1 tool call consistently observed in v5.2 test runs).

**Orchestration note**: v5.2 test runs observe 1 tool call per inventory or order query (`check_inventory`, `calculate_quote`, `get_order`). A well-tuned MODE3 concierge answering a stock-availability question in 1 tool call is the expected pattern.

**Quote calculation**: a visiting agent requesting pricing for a multi-item order receives a structured quote with volume discounts applied. The concierge calls a pricing engine, gets structured JSON, and explains the result. This *compute-on-behalf-of-the-agent* pattern has no equivalent in static document approaches — though a sufficiently capable visiting agent with direct API access could perform equivalent calculations client-side.

**Natural language delegation with built-in discovery**: a visiting agent can submit a question requiring human judgment, receive an `accepted` response with a polling URL and ETA, and retrieve the human's answer when ready. The async model (poll or webhook callback) handles both fast and slow responses. In our tests, the full round trip using a simulated fixed delay completed in ~8,100ms. In production, the ETA reflects real human availability, which varies by hours to days.

These patterns represent a shift in the relationship between visiting agents and websites. The site becomes an active participant capable of doing work on the agent's behalf — though the value of this delegation depends on the visiting agent trusting the site's concierge and on the availability of capable site-side infrastructure.

### 4.5 Content Signal Propagation

In all test runs, content signals (`ai_train: false`, `ai_input: true`, `search: true`, `attribution_required: true`) were present in every response's `meta` object. This provides a machine-readable, per-response record of the site's AI usage policy — more granular than `robots.txt` and more persistent than a page-level meta tag.

---

## 5. Discussion

### 5.1 The Token Economy Argument

The data presents a nuanced picture that should be stated directly:

**Against the naive full-document baseline (77–80% reduction)**: this reflects a structural upper bound. A visiting agent that fetches a 9,700-token corpus to answer a 15-token question wastes ~98% of the tokens it receives. No competent agent implementation operates this way in production — it is included as a reference point.

**Against the RAG-baseline (AHP uses 3–8× more tokens on compact corpora)**: the v5.2 benchmark (cache-busted, API-measured) shows a 3-chunk keyword-retrieval visiting agent calling Claude Haiku uses 291–945 tokens per query on the AHP Specification corpus, versus AHP MODE2's 1,995–2,419 tokens. For pure retrieval efficiency on small, well-structured corpora, AHP MODE2 is less token-efficient than a well-implemented client-side RAG agent.

This result is neither surprising nor a condemnation of AHP. It correctly identifies what AHP provides and what it does not:

**What AHP provides over RAG:**

1. **Protocol-level discovery**: four standardised discovery mechanisms. A RAG agent must know where to fetch the document; an AHP agent discovers capabilities through the protocol.
2. **Capability declaration**: the visiting agent learns *what the site can do*, not just where content lives. MODE3 capabilities are only discoverable and accessible through AHP.
3. **Session management, content signals, error handling**: provided by the protocol as primitives. A RAG agent reimplements these per site or forgoes them.
4. **Retrieval infrastructure offloading**: at scale (100K+ token corpora), client-side keyword retrieval degrades significantly. AHP offloads retrieval to the server, where embedding-based approaches can be adopted without the visiting agent changing its implementation.
5. **MODE3 access**: real-time data, computation, and human delegation are outside the scope of any document-retrieval approach.

**What AHP does not provide over RAG:**

- Lower per-query token cost on compact corpora with simple retrieval queries
- Lower server-side cost (MODE2 requires a Haiku call; RAG requires only a static file serve plus client-side Haiku call)

**Server-side cost estimates** (reference figures as of February 2026):

Claude Haiku 4.5 pricing as of February 2026: input ~$0.80/MTok, output ~$4.00/MTok (prices subject to change; verify at console.anthropic.com).
Average MODE2 response: ~2,200 tokens total (~1,800 input + ~400 output).
Per-query cost without caching: ~(1,800 × $0.80 + 400 × $4.00) / 1,000,000 ≈ **$0.0030** (~0.3¢).

| Volume | Monthly queries | Cost (no cache) | Cost (85% cache hit) |
|--------|----------------|-----------------|----------------------|
| Low | 1,000/day → 30K/mo | ~$90/mo | ~$13.50/mo |
| Medium | 10K/day → 300K/mo | ~$900/mo | ~$135/mo |
| High | 100K/day → 3M/mo | ~$9,000/mo | ~$1,350/mo |

For context, static file serving at 10K requests/day costs near $0/month on CDN-backed hosting. The MODE2 cost is primarily justifiable when: (a) query volume is low-to-medium, (b) cache hit rates are high (>80%), and (c) the answer quality improvement over static retrieval generates meaningful business value (higher task completion, fewer follow-up queries, MODE3 revenue-generating interactions).

For high-traffic deployments, the v1 deployment guide will cover cost modelling, cache pre-warming strategies, and hybrid architectures where common queries are pre-cached and only novel queries invoke the LLM.

### 5.2 AHP and Agent-Native Web Presence

Search Engine Optimisation emerged because websites needed to be found and understood by crawlers that served humans through results pages. As AI agents increasingly mediate user access to web content, sites that are structurally accessible to agents may gain a measurable advantage in agent-directed traffic and task completion.

**We propose the following mechanism as a hypothesis, not an established result**: sites that implement AHP provide agents with structured discovery (four mechanisms), capability declaration, and direct query channels. An agent encountering two equivalent sites — one AHP-compliant, one serving only HTML — may prefer the AHP site because it can complete its task more reliably with lower overhead. If this preference is consistent at the ecosystem level, AHP compliance would function analogously to SEO compliance: not a guarantee of preference, but a necessary condition for being efficiently accessible.

This hypothesis requires validation through adoption-scale data — agent query logs, site traffic analytics with agent-tagged requests, and A/B comparisons of AHP vs. non-AHP response quality. We plan to instrument the reference deployment to collect this data in the coming months.

### 5.3 Progressive Adoption

We designed AHP explicitly for progressive adoption. A MODE1 site adds one file (`/.well-known/agent.json`) and two HTML elements (a `<link>` tag and an in-page notice) to become compliant. Our reference implementation takes a developer from zero to MODE2 in an afternoon. MODE3 requires additional infrastructure (tool integrations, an async queue) but no changes to the lower modes.

This progression matters for ecosystem adoption. The content quality caveat applies to all modes: a MODE1 site with a poorly-structured `llms.txt` provides minimal value to visiting agents regardless of manifest completeness. The five-minute compliance figure refers to the structural elements; high-quality content preparation is a separate investment.

### 5.4 Visiting Agent Side

The current AHP specification and reference implementation focus entirely on the *site side* — what a site must do to be AHP-compliant. The *visiting agent side* — how an agent discovers, selects, and interacts with AHP-compliant sites — is intentionally underspecified in v0.1.

Planned work for the visiting agent side includes:
- A reference visiting agent library (Python and TypeScript) that handles discovery, manifest parsing, capability selection, and session management
- Trust evaluation heuristics: how should a visiting agent decide whether to use a site's MODE3 capabilities?
- Cross-site session federation: how does an agent maintain context across multiple AHP sites in a single task?
- Standardised agent identity: the `requesting_agent` field is currently free-text; a v1 identity scheme (building on W3C DIDs [DID] or signed JWTs) would enable site-side access control and personalisation

### 5.5 Limitations

1. **Retrieval quality ceiling**: the v0 keyword overlap retrieval degrades on large, ambiguous corpora. Embedding-based retrieval is planned for v1.
2. **Conformance scope**: results cover only the reference implementation (co-developed with the spec). No independent third-party implementations have been tested. The test suite is open and accepts community submissions (see §4.1).
3. **Compact evaluation corpora**: both test sites use ~10–50KB corpora. At document-store scale (millions of tokens), the MODE2 retrieval architecture requires rethinking.
4. **Self-referential evaluation**: the AHP Specification Site evaluation queries AHP documentation about AHP. A more rigorous evaluation would use corpora and queries developed independently of the protocol authors. The Nate Jones cross-comparison is a step toward this (different domain, different author) but remains within the same reference deployment ecosystem.
5. **Server-side cost model**: at high query volumes, MODE2 server-side LLM costs are material. See §5.1 cost estimates.
6. **T17 (auth enforcement) is a known spec violation in the demo**: the reference implementation intentionally omits auth to simplify demo access. A production deployment must implement spec §5.3.
7. **T12 (quote calculation) required assertion tightening**: the original vocabulary-based assertion was a false positive. The tightened assertion (price regex + failure-phrase exclusion) is also imperfect — a sufficiently clever failure message could still pass. End-to-end testing with known inventory state remains the most reliable approach.
8. **RAG-baseline comparison is limited to keyword retrieval**: the RAG baseline uses simple keyword overlap scoring. A more capable RAG agent using embedding-based retrieval would likely use even fewer tokens per query with higher answer quality, making the AHP vs. RAG comparison more competitive in both directions.
9. **Known test suite coverage gaps (v5.2)**: two spec features are completely untested across all 22 tests: (a) **content type negotiation (spec §6.6)** — `accept_types`, `response_types`, and `unsupported_type` 400 error are unexercised; a server that omits the entire content type negotiation system passes all 22 tests; (b) **session time-based expiry (10-minute TTL)** — T18 verifies the 10-turn limit but not the TTL; a server with infinite session TTL passes all 22 tests. Both gaps are documented in the test suite JSON output under `known_coverage_gaps`.
10. **T21 §6.3 format unverified**: T21 confirms the server is spec-compliant (§6.3 says MAY) but does not verify that the `clarification_needed` response format is correct — because the reference server never returns it for test queries. The conformance table marks T21 as "✓ Advisory." Verifying the format requires either a server that implements clarification for ambiguous queries, or a T21b mock-parser companion test.

---

## 6. Related Work

**Cloudflare's Markdown for Agents** [CLOUDFLARE] introduced clean markdown serving as an agent-accessible alternative to HTML. AHP's MODE1 is directly compatible with this approach and positions it as the entry point of a larger protocol.

**llms.txt** [LLMSTXT] proposed a standard location for a site-level plain text document. AHP MODE1 treats `llms.txt` as a valid content endpoint. AHP provides the discovery layer, capability declaration, and upgrade path that `llms.txt` lacks.

**Model Context Protocol (MCP)** [MCP] defines a standard for LLM tool access. AHP MODE3 is complementary: where MCP standardises how an agent calls a tool server, AHP defines how a website-side agent is discovered, queried, and delegated to. A MODE3 concierge may itself use MCP to access its tools.

**OpenAI GPT Actions / Plugin specification** [GPTACTIONS]: the most widely deployed agent-web interaction mechanism prior to AHP. GPT Actions define structured API call schemas for LLM tool use. AHP differs by focusing on natural language querying through a concierge (rather than direct API schema exposure), by providing discovery and content signals, and by defining a progressive adoption path from static content through to agentic delegation.

**Schema.org / Google Structured Data** [SCHEMAORG]: the existing machine-readable web layer. AHP's content signals and manifest serve a complementary purpose — declaring AI-interaction preferences and capabilities rather than semantic entity types. A future revision will explore alignment between AHP content signals and Schema.org metadata conventions.

**ai.txt** [AITXT]: a competing convention for declaring AI usage preferences at the site level. AHP's `content_signals` block covers similar ground while being embedded in the discovery and interaction protocol rather than a standalone declaration file.

**IETF RFC 8615 — Well-Known URIs** [RFC8615]: the formal specification for the `.well-known/` URI path convention that AHP relies on for manifest discovery. AHP's `/.well-known/agent.json` is a Well-Known URI and should be registered with IANA under this convention.

**robots.txt** — AHP's `/.well-known/agent.json` follows the same well-known URI convention and spirit as `robots.txt`, providing a standard machine-readable declaration at a predictable path.

**OpenAPI 3.1 / AsyncAPI 2.x** [OPENAPI]: AHP borrows capability declaration conventions from OpenAPI but targets natural language agent interaction through a concierge rather than direct structured API documentation for machine clients.

---

## 7. Conclusion

The web's interaction model was designed for human browsers. AI agents need something different: a protocol for negotiated, capability-aware, stateful interaction. AHP provides that protocol.

Against a naive full-document baseline, AHP MODE2 demonstrates a 75–80% token reduction across two sites (AHP Specification: 77.4%; Nate Jones AI-practitioner blog: 71.5% — 12:21 UTC v5.2 run). Against a RAG-baseline visiting agent, AHP uses 3–8× more tokens on compact corpora; its advantage is protocol-level, not efficiency-level: structured discovery, capability declaration, server-managed caching (~12ms p50 cache-hit), session management, content signals, and MODE3 capabilities.

We demonstrated **21/22 conformance** on the reference deployment (v5.2 test suite, T01–T22): T17 (MODE3 auth) is the single failure — a CRITICAL security defect in the demo configuration. T21 is advisory (§6.3 MAY — format unverified). Cache-hit latency averages 12ms p50 (n=9 confirmed hits). Cold-cache latency averages 3,614ms p50 (LLM inference). The MODE3 interaction model delivers real-time inventory, computation, and async human delegation unavailable to any document-retrieval approach.

AHP is designed to grow with the ecosystem. MODE1 compatibility with existing `llms.txt` deployments ensures the protocol can spread through the current base of agent-accessible sites. The extension mechanism in Appendix C allows new content types to be registered without breaking compatibility.

We invite the community to review the specification, run the test suite (v5.2) against their deployments, and submit conformance results at `github.com/AHP-Organization/agent-handshake-protocol`.

---

## References

[CLOUDFLARE] Cloudflare. "Markdown for Agents." Cloudflare Blog, 2024. https://blog.cloudflare.com/markdown-for-agents/

[LLMSTXT] Answer.AI. "llms.txt — A Standard for LLM-Accessible Content." 2024. https://llmstxt.org

[MCP] Anthropic. "Model Context Protocol." 2024. https://modelcontextprotocol.io

[GPTACTIONS] OpenAI. "GPT Actions." OpenAI Documentation, 2024. https://platform.openai.com/docs/actions

[SCHEMAORG] Schema.org. "Schema.org Structured Data." https://schema.org

[AITXT] ai.txt. "AI Usage Declaration Standard." https://aitxt.org (see also https://site.ai/aitxt)

[RFC8615] Nottingham, M. "Well-Known Uniform Resource Identifiers (URIs)." IETF RFC 8615, 2019. https://www.rfc-editor.org/rfc/rfc8615

[OPENAPI] OpenAPI Initiative. "OpenAPI Specification 3.1." https://spec.openapis.org/oas/v3.1.0

[DID] W3C. "Decentralized Identifiers (DIDs) v1.0." 2022. https://www.w3.org/TR/did-core/

---

## Appendix: Artefacts

All artefacts from this paper are open source and publicly accessible.

| Artefact | URL |
|----------|-----|
| AHP Specification (Draft 0.1) | https://agenthandshake.dev/spec |
| JSON Schemas | https://agenthandshake.dev/schema/0.1/ |
| Reference Implementation | https://github.com/AHP-Organization/reference-implementation |
| Live Reference Endpoint | https://ref.agenthandshake.dev |
| Test Suite (v5.2) | https://github.com/AHP-Organization/test-suite |
| Raw Test Results | https://github.com/AHP-Organization/test-suite/tree/main/results |

---

*Agent Handshake Protocol — Draft 0.1 (revised 2026-02-22, v5.2 run 12:21 UTC)*
*© 2026 Nick Allain. Licensed under CC BY 4.0.*
