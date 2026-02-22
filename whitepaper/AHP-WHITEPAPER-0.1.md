# Agent Handshake Protocol: A New Contract Between AI Agents and the Web

**Nick Allain**
*agenthandshake.dev · github.com/AHP-Organization*

*Draft 0.1 — February 2026 (revised 8 — 15:46 UTC)*

---

## Abstract

AI agents are becoming primary actors on the web — not visitors browsing for humans, but autonomous systems completing tasks, making decisions, and acting on behalf of users. The web has no standard for this. Agents scrape HTML built for browsers, parse documents built for humans, and interact with sites that have no idea they're being visited by a machine. Sites have no way to declare what they can do for an agent. Agents have no way to discover what a site can offer. There is no handshake.

We present the **Agent Handshake Protocol (AHP)**: a three-mode open specification that gives agents and websites a shared vocabulary. A site publishes a machine-readable manifest declaring its capabilities — what it can answer, what actions it can take, what constraints apply. Visiting agents discover the manifest and interact through a defined protocol rather than scraping. The result is a fundamentally different relationship: not agent-extracts-from-site, but agent-and-site-collaborate.

The capability progression is the core contribution. **MODE1** makes a site discoverable and legible to any agent in under five minutes. **MODE2** enables a visiting agent to ask a question and receive a precise, sourced answer — the site's concierge does the retrieval work so the agent doesn't have to. **MODE3** makes the site an active participant: the concierge can query live databases, execute calculations with current pricing, and escalate to human experts — capabilities no document-serving approach can provide regardless of how it is queried.

To validate the protocol, we built a reference implementation and tested it against two independent sites. Token efficiency results (MODE2 reduces consumption by 71–78% vs. a naive visiting agent; a well-implemented client-side RAG agent narrows this to a 3–8× overhead, with AHP's advantages being protocol-level rather than retrieval-level) and latency data (10ms p50 cached; 3,806ms p50 cold) are reported in full. The reference deployment passes **23 of 24 conformance checks** in the v5.4 test suite (T17 is a known security defect in the demo configuration). AHP is designed for progressive adoption: each mode is backwards compatible, and MODE1 compliance is achievable as a five-minute addition to any existing site.

---

## 1. Introduction

Consider what an AI agent has to do today to interact with a website it has never seen before.

It fetches the page. It receives HTML — navigation bars, footers, cookie banners, sidebars, markup — built entirely for a human rendering engine. It extracts what it can from the noise. If it needs to know whether a product is in stock, it searches a page that was designed to show a human a price and an "Add to Cart" button. If it needs to understand what a company does, it reads marketing copy written to persuade humans, not inform machines. If it needs to take an action — place an order, book an appointment, escalate an issue — it either calls an undocumented API it somehow discovered or gives up.

This is the current state of the art. The web is used by hundreds of millions of AI agent calls per day, and the interaction model is: scrape what you can and hope for the best.

Several partial solutions exist. Cloudflare's "Markdown for Agents" [CLOUDFLARE] strips HTML and serves clean markdown. The `llms.txt` convention [LLMSTXT] offers a site-level plain text document. These are improvements, but they share the same fundamental model: the site throws a document over a fence, and the agent must extract meaning from it entirely on its own. The site plays no active role. There is no negotiation. The site does not know what the agent needs, and the agent has no way to ask.

**The problem is not formatting. It is the absence of a shared vocabulary.**

When two humans do business, there is a handshake. A declaration of intent. An exchange of capability and constraint. *Here is what I can offer. Here is what I need. Here is what I will and will not do with what you give me.* The web has this vocabulary for humans — product listings, checkout flows, contact forms, terms of service. It has no equivalent for agents.

AHP defines that vocabulary.

### 1.1 What AHP Makes Possible

Before describing the protocol, it is worth being explicit about what it enables — because the value is not primarily efficiency.

**Sites become participants.** Today a site is a passive resource that agents extract from. With MODE3, a site becomes an active party: its concierge can check live inventory, calculate a quote with current pricing, look up an order, and escalate to a human expert — all within a single agent conversation, without the visiting agent having access to any of those systems directly. The visiting agent describes what it needs; the site does the work. This is delegation, not retrieval, and it has no equivalent in any document-serving model.

**Any agent works with any site, zero custom integration.** An AHP-aware agent discovering a new site fetches the manifest, reads the capability declarations, and starts working — without that agent having been specifically coded for that site. This is the web's original interoperability promise applied to agents: a universal protocol that any site can implement and any agent can speak. Today this doesn't exist. Every agent-to-site integration is either hardcoded or a scraping hack.

**Sites control the terms of agent interaction.** AHP's content signals travel with every response: `ai_train`, `ai_input`, `search`, `attribution_required`. A site can declare its AI usage policy in a machine-readable, per-response format more precise than `robots.txt` and more persistent than a page-level meta tag. For the first time, a site's preferences have a standard channel to reach the agents consuming it.

**The web gains a discovery layer for the agent era.** AHP's three discovery mechanisms ensure that an agent encountering any AHP-compliant site — whether it arrives via HTTP client, headless browser, or as a result of a human passing a URL — can find and use the protocol. Discovery is a solved problem in the browser web (DNS, HTML, hyperlinks). For agents it is currently not solved at all.

### 1.2 Protocol Overview

AHP achieves this through three progressive modes, each building on the previous:

- **MODE1 (Static Serve)**: the site publishes a manifest and provides agent-readable content. A site with an existing `llms.txt` document becomes MODE1-compliant in under five minutes. Agents can discover the site's capabilities and retrieve content without any server-side logic.

- **MODE2 (Interactive Knowledge)**: the site runs a concierge that accepts natural language queries and returns precise, sourced answers. The agent asks for what it needs; the site retrieves only what is relevant. Session management, content signals, and schema-validated responses are protocol primitives — the visiting agent gets them for free.

- **MODE3 (Agentic Desk)**: the concierge is equipped with tools — live data access, calculation engines, external APIs, MCP server connections, and a human escalation queue. The visiting agent can delegate tasks that require real-time data, computation, or human judgment. Each tool call is orchestrated by the site's concierge, with the visiting agent receiving a natural language result.

Each mode is backwards compatible: a MODE3 site is automatically MODE1 and MODE2 compliant. A MODE1 site can be upgraded incrementally.

### 1.3 Contributions

This paper presents:

1. **The AHP specification** — a three-mode protocol with three discovery mechanisms, conversational interaction, agentic delegation, session management, content signals, and async human escalation
2. **A reference implementation** — a complete open-source MODE1/MODE2/MODE3 server deployable on any Node.js host
3. **Empirical evaluation** — conformance testing, token efficiency benchmarks, and latency profiling across two independent sites (reference implementation only; no third-party implementations tested in this version)
4. **A test suite** — an open-source visiting agent harness (v5.4, 24 conformance tests including RAG-baseline comparison, cache-busted multi-run averaging, automatic outlier detection, cross-site comparison, session memory, rate-limit headers, and §6.3 format validation)

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

AHP defines **three discovery mechanisms**, each targeting a distinct class of visiting agent:

1. **Well-known URI** — `GET /.well-known/agent.json` per IETF RFC 8615 [RFC8615]. The universal fallback: any agent can fetch it directly. The only MUST in the discovery section.
2. **HTTP `Link` response header** — `Link: </.well-known/agent.json>; rel="agent-manifest"; type="application/agent+json"` sent proactively on every HTTP response (RFC 8288 [RFC8288]). Reaches HTTP-native agents that inspect response headers before parsing body content — invisible to HTML-parsing agents but present on all responses including API endpoints, 404 pages, and `HEAD` requests.
3. **In-page agent notice** — a `<section aria-label="AI Agent Notice" style="display:none">` in the page body. Reaches LLMs reading page content as text — agents that arrived via a headless browser or were passed a page by a human user and have no direct HTTP awareness.

The `Accept: application/agent+json` header is **not** a discovery mechanism — it is a capability negotiation signal for agents that already know a site supports AHP. It belongs in the request protocol, not the discovery layer (spec §3.4).

#### 2.4.1 In-Page Notice: Scope, Limitation, and Path Forward

The in-page notice spec recommends the element be visually hidden (`display:none`) while remaining in the DOM. This ensures the notice does not disrupt human UX. However, this creates a coverage gap:

- **Raw-HTML scraping agents** (using `curl`, `requests`, `urllib`) read the source document and will find `display:none` content. The in-page notice works for these agents.
- **Headless browser agents** (Puppeteer, Playwright, Selenium) execute JavaScript and render the full DOM. Standard text-extraction methods for rendered pages may skip hidden elements. These agents are **not reliably covered** by the current in-page notice design.

This is a material concession. Headless browser agents are a dominant and growing implementation pattern. An in-page notice that doesn't reach them is a narrow mechanism serving HTTP-scraping agents who can also use mechanisms 3 (Accept header) and 4 (well-known URI).

**AHP v0.1 addresses the HTTP-native gap with mechanism 3** (§3.5): the proactive HTTP `Link` response header reaches agents that inspect headers before parsing body content. This is verified by T23 in v5.4 of the test suite and is confirmed working on both reference deployments. The headless browser gap (agents that render JavaScript and may skip `display:none` content) remains open and is the primary motivation for a sixth mechanism in v0.2: a `<meta name="ahp-manifest" content="/.well-known/agent.json">` tag in `<head>`, which is accessible in rendered DOM and follows the established `<meta name="robots">` convention.

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
- **Session management**: in-memory sessions with 10-minute TTL and 10-turn limit. Session history is passed to the LLM on subsequent turns, enabling multi-turn context (confirmed live in v5.3: server correctly quotes prior turn content verbatim in T08 turn 2 response). **Practical developer note**: the server passes full session history to the LLM; developers building visiting agents on top of AHP do not need to implement their own turn-history injection — the server manages it as a protocol primitive.
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

Our test suite (`AHP-Organization/test-suite`, v5.3) acts as a visiting agent, running 24 conformance tests (T01–T23 + T21b):

- T01–T16 (original suite): discovery, schema, MODE2, MODE3, error handling, sessions, content signals, caching, rate limiting
- T17: MODE3 action auth enforcement (CRITICAL DEFECT — demo server allows unauthenticated actions; spec §5.3 MUST)
- T18: Session 10-turn limit enforcement (spec §5.4.3)
- T19: Request body 413 on oversized payload >8KB (spec §6.5)
- T20: Rate-limit headers present and numeric on all responses (spec §11.1)
- T21: Clarification-needed live probe (spec §6.3 MAY — advisory if never triggered)
- T21b: Clarification-needed format parser — mock validation of §6.3 format spec (v5.3, new)
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

The reference deployment was tested against the v5.3 test suite (T01–T23 + T21b = 24 tests). Results for the 2026-02-22 15:46 UTC run:

**Scope note**: conformance was demonstrated only for the reference implementation, which was co-developed with the spec and test suite. No independent third-party implementations have been tested. Results reflect internal consistency of the reference implementation, not ecosystem-wide interoperability.

**How to submit third-party conformance results**: run the open-source test suite against your AHP deployment (`./venv/bin/python test_runner.py --target <your-endpoint> --output result.json`) and open a pull request to `AHP-Organization/test-suite/results/` with your JSON report and a brief description. A hosted conformance runner is planned for v0.2.

| Test | Category | AHP Ref | Notes |
|------|----------|---------|-------|
| T01: Well-known discovery | Discovery | ✓ | |
| T02: Accept header negotiation (spec §3.4) | Capability negotiation | ✓ | Not a discovery mechanism — confirms agents with prior AHP knowledge can retrieve manifest directly |
| T03: In-page agent notice | Discovery | ✓ | Passes; API server exempt — see §2.4.1 |
| T23: HTTP Link response header (§3.5) | Discovery | ✓ | Confirmed on /, /health, POST /agent/converse — all responses carry the header (v5.4) |
| T04: Manifest schema | Protocol | ✓ | |
| T05: MODE2 content query | Functionality | ✓ | |
| T06: Unknown capability error | Error handling | ✓ | |
| T07: Missing field error | Error handling | ✓ | |
| T08: Multi-turn + session memory | Sessions | ✓ | **See note** — v5 confirms server has session memory |
| T09: Response schema | Schema | ✓ | |
| T10: Content signals in response | Signals | ✓ | |
| T11: MODE3 inventory (tool use) | MODE3 | ✓ | 1 tool call (`check_inventory`) observed consistently in v5.3 runs |
| T12: MODE3 quote with numeric prices | MODE3 | ✓ | Assertion fixed: requires `$\d` pattern + no failure phrases |
| T13: MODE3 order lookup | MODE3 | ✓ | |
| T14: MODE3 async human escalation | MODE3 async | ✓ | Latency is simulated |
| T15: Response caching | Infrastructure | ✓ | |
| T16: Rate limiting (429) | Infrastructure | ✓ | Window partially consumed by prior tests; see note |
| T17: MODE3 action auth enforcement | Security | **✗** | **CRITICAL KNOWN DEFECT** — see note |
| T18: Session 10-turn limit | Sessions | ✓ | Limit triggered on turn 11 |
| T19: Oversized body → 413 | Protocol | ✓ | HTTP 413 on 10KB body |
| T20: Rate-limit headers (spec §11.1) | Protocol | ✓ | All 4 headers present, numeric (v4) |
| T21: Clarification needed — live probe (§6.3) | Protocol | ✓ Advisory | Server answered "Which mode should I use?" directly; spec says MAY — not required to clarify; T21b provides isolated format coverage |
| T21b: Clarification needed — format parser (§6.3) | Protocol | ✓ | Mock-parser validates all 4 cases: valid-full, valid-minimal, invalid-missing-question, wrong-status. **Provides isolated §6.3 FORMAT coverage** (v5.3) |
| T22: Invalid session_id graceful handling (§6.2) | Sessions | ✓ | Server returned HTTP 400 (correct — any 4xx is valid per spec §6.2) |

**T08 note — false-negative history**: an earlier suite (v4) incorrectly failed T08 for a deployed server that does have session memory. The v4 exclusion phrase `"FIRST QUESTION"` matched `"In your first question, you requested…"` — a memory-demonstrating sentence — triggering a false negative. The v5 suite removed this ambiguous phrase; the v5.1 run confirms the fix is correct: the server responds `"You asked specifically about MODE1. In your first message, you explicitly requested: 'Tell me about AHP modes — focus especially on MODE1.'"` — unambiguous session memory. T08 ✓ in v5.1.

**T16 note**: rate limit test runs last; window was 9/30 remaining when T16 started (burst hit 429 after 10 requests). The declared limit of 30/min is confirmed by T20's X-RateLimit-Limit header.

**T17 note — CRITICAL KNOWN DEFECT**: the reference implementation accepts unauthenticated requests to all three MODE3 action capabilities (inventory_check, get_quote, order_lookup). This violates spec §5.3 (MUST require authentication for action capabilities with side effects). Every developer who clones the reference implementation and deploys it without adding authentication ships a production system violating a MUST requirement. A `--require-auth` configuration flag is planned for v0.1.1.

**23/24 (95.8%)** on reference deployment (v5.4 suite, 15:46 UTC). T17 ✗ (CRITICAL DEFECT — no auth enforcement). T21 ✓ advisory (§6.3 MAY — live server never triggers clarification). T21b ✓ (mock-parser confirms §6.3 format spec is correctly validated). T23 ✓ (HTTP Link header confirmed on all response types). All other 21 tests pass conclusively, including T08 (session memory confirmed), T20 (rate-limit headers), T22 (invalid session → HTTP 400 correct).

### 4.2 Token Efficiency

We compared token consumption for three approaches on five representative queries against the AHP Specification Site.

**Baseline definitions:**
- **Naive visiting agent (no retrieval)**: fetches the full `llms.txt` document (~40,900 chars / ~10,000 tiktoken tokens) and passes it with the query to an LLM. Token count estimated via tiktoken cl100k_base; not API-measured. *This is the baseline no competent agent implementation would use in production — it represents the upper bound on token waste and serves as a reference point, not a competitive comparison.*
- **RAG-baseline visiting agent**: fetches `llms.txt`, chunks by paragraph (~500 chars), selects top-3 chunks by keyword overlap, calls Claude Haiku. This is a **fair competitive baseline** representing a reasonably-implemented visiting agent. Token counts are API-measured. Run 3 times; reported as mean ± stddev.
- **AHP MODE2**: visiting agent POSTs query to `/agent/converse`. Token counts are API-measured. Run 3 times; reported as mean ± stddev.

#### Methodology Note: Cache-Busting

The suite (first implemented in v3, current v5) appends a unique 8-char nonce to each comparison query run (e.g. `"Explain what MODE1 is [ref:a1b2c3d4]"`) to force distinct server-side cache keys. Without this, the server's 5-minute TTL cache produces byte-identical responses for runs 2 and 3 against an identical query, making the reported stddev artificially ±0 — a measure of cache consistency rather than LLM output variance. With cache-busting, each run triggers a fresh LLM call and stddev reflects genuine answer variation. The nonce adds ~4 tokens of overhead per run (systematic and consistent).

#### AHP Specification Site (corpus: ~40,100 chars, ~9,665 tokens tiktoken, 96 chunks)

Results from 2026-02-22 **15:46 UTC — v5.4 suite** (adds T21b and automatic outlier detection). Naive baseline: tiktoken estimate (not API-measured; fetch latency is a **single measurement** reused for all queries). RAG and AHP: API-measured (Claude Haiku 4.5, Anthropic), 3-run mean ± stddev with cache-busting. Latency stddev shown for high-variance rows (CV >20%).

| Query | Naive†<br/>(fetch ×1) | RAG-baseline ± σ<br/>(lat. / ±σ) | AHP MODE2 ± σ<br/>(lat. / ±σ) | Reduction<br/>vs. naive ↓ | Overhead<br/>vs. RAG ↑ |
|-------|------------------------|-----------------------------------|--------------------------------|--------------------------|------------------------|
| Explain what MODE1 is | ~9,735 (119ms) | **292 ±2** (1,134ms) | **2,417 ±35** (4,306ms) | **75.2%** | +727% |
| How does AHP discovery work? | ~9,736 (119ms) | **483 ±9** (1,610ms / ±109ms) | **1,990 ±27** (3,355ms / ±416ms) | **79.6%** | +312% |
| What are AHP content signals? | ~9,736 (119ms) | **441 ±41** (1,326ms / ±355ms) | **2,202 ±14** (3,919ms / ±557ms) | **77.4%** | +400% |
| How do I build a MODE2 endpoint? | ~9,735 (119ms) | **938 ±26** (2,649ms) | **2,317 ±44** (4,416ms) | **76.2%** | +147% |
| What rate limits should AHP enforce?‡ | ~9,736 (119ms) | **574 ±16** (1,617ms / ±588ms) | **2,080 ±16** (2,905ms) | **78.6%** | +262% |
| **Average** | | | | **77.4%** | **+370%** |

† Naive tokens: tiktoken cl100k_base estimate, not API-measured; ±5–10% error. Fetch latency (119ms) is a **single measurement** reused for all queries; it is not a 3-run mean. Latency stddev shown only for rows with CV >20%. ‡ Rate-limits RAG latency ±588ms (CV=36%) reflects Anthropic API variance; no run exceeded the 2× median outlier threshold in this run. Note: the 14:33 UTC run did have a RAG outlier on this query (run 1 = 3,234ms vs ~1,100ms for the other two; CV=68%, mean inflated to 1,813ms) — this is now automatically flagged in `latency_outliers` in the v5.4 JSON output. The v5.4 suite detects and reports any benchmark run >2× the 3-run median. Latency profile §4.3 p50 (3,806ms) uses short uniform-format nonce queries; evaluation latency varies by answer complexity (2,905–4,416ms range in this run).

**Reading the table**: Reduction vs. naive ↓ — higher is better for AHP. Overhead vs. RAG ↑ — lower is better for AHP (+141% = AHP uses 2.4× RAG tokens; +732% = AHP uses 8.3× RAG tokens). AHP is worse on the RAG column for all five queries.

**Latency stddev**: both RAG and AHP latency stddev values reflect Anthropic API inference variance (e.g. AHP discovery ±416ms; RAG content-signals ±355ms; AHP content-signals ±557ms). These are the highest-variance rows in the 15:46 UTC run. Token stddev is small (±2–44) reflecting mostly-consistent answer lengths across runs. The v5.4 suite flags any single run >2× the 3-run median as a suspected outlier in `latency_outliers`.

**Core finding**: AHP MODE2 uses **3–8× more tokens per query** (292–938 for RAG vs 1,990–2,417 for AHP) on a compact 9,735-token corpus. The 75–80% reduction vs. naive is real; AHP's value vs. RAG is protocol-level:

| | RAG visiting agent | AHP MODE2 visiting agent |
|-|-------------------|--------------------------|
| Per-query token cost (≤10K corpus) | **Lower** (292–938 tokens) | Higher (1,990–2,417 tokens) |
| Cold-cache latency | ~1,134–2,649ms (measured) | ~2,905–4,416ms (+server round-trip) |
| Cache-hit latency | N/A — client manages caching | **~10ms p50** (server 5-min TTL) |
| Discovery mechanism | Must know document URL | Structured 4-mechanism discovery |
| Capability declaration | None | Manifest + capabilities |
| Session management | Client must implement or skip | Protocol primitive — confirmed working |
| Content signals (AI usage policy) | None | Per-response |
| Retrieval at scale (>100K tokens) | Keyword overlap degrades | Server-managed; embedding upgrade opaque to visitors |
| MODE3 real-time data / compute | ✗ Not possible | ✓ Native |
| Server infrastructure required | ✗ None | ✓ Required |

#### Nate Jones Site Cross-Comparison (AI-Practitioner Domain)

Results from 2026-02-22 15:46 UTC (v5.4 suite, cache-busted, RAG baseline API-measured). Corpus: ~39,341 chars, ~8,877 tokens tiktoken, 82 chunks. Fetch latency (184ms) is a single measurement.

| Query | Naive† | RAG ± σ<br/>(lat.) | AHP ± σ<br/>(lat.) | Reduction<br/>vs. naive ↓ | Overhead<br/>vs. RAG ↑ |
|-------|--------|---------------------|---------------------|--------------------------|------------------------|
| What is RAG and how does it work? | ~8,946 | **436 ±3** (1,066ms) | **2,576 ±16** (2,738ms) | **71.2%** | +491% |
| Explain prompt engineering‡ | ~8,939 | **461 ±25** (1,818ms) | **2,590 ±22** (4,666ms) | **71.0%** | +461% |
| What is MCP and how does it relate to AI agents? | ~8,949 | **512 ±5** (1,066ms) | **2,582 ±27** (3,315ms) | **71.1%** | +404% |
| What is vibe coding? | ~8,940 | **452 ±13** (1,532ms) | **2,262 ±10** (4,146ms) | **74.7%** | +400% |
| How do AI agents work in production? | ~8,945 | **631 ±24** (2,578ms) | **2,662 ±31** (4,412ms) | **70.2%** | +322% |
| **Average** | | | | **71.7%** | **+416%** |

Cross-domain: 71.7% reduction vs. naive (vs. AHP site 77.4% — Nate's slightly lower reflects different corpus and query types). Overhead vs. RAG +416% (~5.2×). ‡"Explain prompt engineering" consistently produces the highest AHP latency (~4.7s across multiple runs) — this is reproducible across v5.x suite runs and reflects a longer synthesised answer, not infrastructure variance. RAG and AHP latency variance reflects Anthropic API inference variability. Cross-domain consistency across two unrelated sites confirms these are architectural properties, not AHP-corpus artefacts. Latency is mean across 3 runs.

### 4.3 Latency Profile

The v5.4 suite runs a split 10-cold + 10-hot design: 10 unique-nonce queries force cold-cache LLM calls; 10 repetitions of a fixed query measure cache-hit performance. Results from 2026-02-22 15:46 UTC run.

| Cohort | Metric | Value | Notes |
|--------|--------|-------|-------|
| Cold (forced cache miss) | mean | **3,729ms** | n=10; unique nonces, guaranteed cold |
| Cold | p50 | **3,806ms** | |
| Cold | max | **4,267ms** | |
| Hot (cache-hit) | p50 | **10ms** | n=9 cache hits; run 1 was cold (3,552ms, first call on fresh query) |
| Hot | max | **3,552ms** | Run 1 only — first call on fresh query is always cold |
| All 20 samples | p50 | **3,233ms** | Mix of 10 cold + 10 hot |
| All 20 samples | p95 | **4,267ms** | True p95 at n=20 |
| All 20 samples | min | **8ms** | Fastest cache-hit sample |

**Key finding**: AHP MODE2 has a bimodal latency distribution. Cache-hit responses (~10ms p50) are comparable to a CDN-served static file. Cold-cache responses (~3,806ms p50) are dominated by Claude Haiku inference time. The hot-cohort max (3,552ms) is run 1 only — the first call on any new query phrase is always cold; subsequent identical queries hit the cache.

**Note on cache-hit p50 variance across runs**: the cache-hit p50 varies between runs (8ms–29ms observed across v5.x suite runs) reflecting server event-loop queue depth at test time. At low-double-digit milliseconds all values are well within "CDN-tier" range and structurally unavailable to a stateless RAG visiting agent. Practitioners can expect their deployed server's cache-hit latency to vary similarly with server load, not caching mechanism quality.

**Note on latency profile vs. evaluation query latency**: the profile p50 (3,806ms) uses short uniform-format nonce queries (low output token count). Evaluation queries in §4.2 range from 2,905ms to 4,416ms AHP cold-cache — variation driven by answer complexity (longer answers = more output tokens = slower). The abstract's "~3,806ms cold-cache" is a summary statistic; individual evaluation query latency varies within this range.

**RAG-baseline latency comparison** (from §4.2 benchmark data):

| Approach | Cold-cache latency | Effective cached latency |
|----------|--------------------|--------------------------|
| Naive (doc fetch only, no LLM) | ~119ms (measured) | N/A — re-fetches every time |
| RAG visiting agent (fetch + Haiku) | ~1,066–2,649ms (measured) | N/A — client manages own caching |
| AHP MODE2 (cold cache) | **~3,806ms p50** (directly measured, n=10) | **~10ms p50** (server 5-min TTL, n=9 cache hits) |

AHP's cold-cache p50 (~3,806ms) is 1.4–3.6× higher than the RAG baseline cold latency range (~1,066ms–2,649ms), reflecting server round-trip overhead on top of the same Haiku inference. AHP's cache-hit latency (~10ms) is structurally unavailable to a stateless RAG visiting agent, which must re-execute the full RAG pipeline on repeated queries. The profile p50 (3,806ms) is measured with short nonce queries; evaluation query AHP latency ranges from 2,905ms to 4,416ms depending on answer complexity (see §4.2 table footnote).

**Note on MODE3 async latency**: the T14 ~8,100ms figure is from a simulated human operator with fixed delay. Production human escalation = hours to days.

### 4.4 MODE3: Capabilities Beyond Static Content

MODE3 enables a qualitatively different class of interactions — ones that static content approaches do not support through the same protocol interface:

**Real-time data access**: a visiting agent querying `inventory_check` receives current stock levels, pricing, and availability — data that would be stale in any static document. The concierge uses Claude's tool_use to call a structured inventory database, synthesise the result, and respond in natural language. Token cost: ~2,400–3,000 (1 tool call consistently observed in v5.4 test runs).

**Orchestration note**: v5.4 test runs observe 1 tool call per inventory or order query (`check_inventory`, `calculate_quote`, `get_order`). A well-tuned MODE3 concierge answering a stock-availability question in 1 tool call is the expected pattern.

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

**Against the RAG-baseline (AHP uses 3–8× more tokens on compact corpora)**: the v5.4 benchmark (cache-busted, API-measured, 15:46 UTC run) shows a 3-chunk keyword-retrieval visiting agent calling Claude Haiku uses 292–938 tokens per query on the AHP Specification corpus, versus AHP MODE2's 1,990–2,417 tokens. For pure retrieval efficiency on small, well-structured corpora, AHP MODE2 is less token-efficient than a well-implemented client-side RAG agent.

This result is neither surprising nor a condemnation of AHP. It correctly identifies what AHP provides and what it does not:

**What AHP provides over RAG:**

1. **Protocol-level discovery**: three standardised discovery mechanisms. A RAG agent must know where to fetch the document; an AHP agent discovers capabilities through the protocol.
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

### 5.2 AHP as Agent-Native Complement to SEO

An important distinction: AHP is not a successor to SEO and does not address discoverability, ranking, or the business incentives that SEO optimises for. A site implementing AHP does not become more findable to agents than one that does not — AHP has no equivalent of PageRank, no index, no signal that rewards compliance.

What AHP does address is *legibility and capability* once a site has been found. SEO gets an agent to your door; AHP determines what happens when it knocks. These are complementary, not competing, concerns.

**A bounded hypothesis:** sites that implement AHP may gain a measurable advantage in *task completion rate* over non-AHP sites among agents that are AHP-aware. An agent encountering two sites that both answer its query — one returning a structured, sourced AHP response, one requiring HTML parsing — may prefer the AHP site for subsequent interactions, simply because interactions are more reliable and less ambiguous. If this preference accumulates at ecosystem scale, AHP compliance could function as an agent-era quality signal, analogous to structured data markup (Schema.org) in the SEO world: not a ranking factor directly, but a marker of machine-legibility that correlates with better agent outcomes.

This hypothesis requires validation through adoption-scale data — agent query logs, site traffic analytics with agent-tagged requests, and A/B comparisons of AHP vs. non-AHP task completion rates. We plan to instrument the reference deployment to collect this data in the coming months.

### 5.3 The Business Model Question: Why Would a Site Implement AHP?

This question deserves a direct answer. A site implementing MODE2 is running an LLM-backed retrieval service for any visiting agent — at its own cost. The reference cost model (§5.1) shows this is manageable at low-to-medium query volumes with good cache hit rates, but the cost is real and the question is legitimate: what does a site get in return?

**The honest answer depends on the site's relationship to agents:**

**Sites where agents are the customer pipeline.** For a growing class of sites — developer tools, SaaS products, API providers, documentation hubs — AI agents are already a primary access vector. Developers use agents to explore APIs, understand integration options, and generate code. A site without AHP gets scraped; a site with AHP gets queried intelligently, with sessions, capability declaration, and structured responses. AHP is not an altruistic investment here — it is an acquisition channel.

**Sites where MODE3 generates direct revenue.** An e-commerce site implementing MODE3 (`inventory_check`, `get_quote`, `order_lookup`) turns agent queries into sales funnel entries. The cost of an LLM-backed inventory query is a fraction of the cost of a human customer service interaction, and MODE3 scales without additional staff. Sites with existing chatbot or live-chat infrastructure will recognise this model — AHP standardises and makes it interoperable.

**Sites where agent-legibility is a content quality investment.** A media site, knowledge base, or professional services firm that implements MODE2 is investing in the quality of how its content reaches agents acting on behalf of humans. As agent-mediated access to information grows, the sites that invest in structured, accurate, agent-readable responses now are building a content moat. The cost is low at moderate traffic; the asymmetric benefit is that poorly-legible sites get misrepresented by agents that scrape and hallucinate.

**The case for not implementing MODE2/MODE3:** if a site has very high query volume from agents, no authentication layer, and no business value from accurate agent responses, the cost-benefit may not justify MODE2. MODE1 (static manifest + `llms.txt`) is always zero marginal cost and provides meaningful agent-legibility at no runtime expense. Sites should not feel compelled to implement MODE2 unless they have a clear use case.

**Rate limiting as cost control.** The spec's rate limits (30 req/min unauthenticated, 120 req/min authenticated) are the primary cost control mechanism. Sites SHOULD implement these defaults and SHOULD require authentication for high-volume or MODE3 access. An authenticated agent interaction implies an established relationship — the site can assign a token budget, track usage, and charge for it if the value exchange warrants it.

### 5.4 Progressive Adoption

We designed AHP explicitly for progressive adoption. A MODE1 site adds one file (`/.well-known/agent.json`), one HTTP response header (the RFC 8288 `Link` header, often a single nginx line), and an in-page notice to become compliant. Our reference implementation takes a developer from zero to MODE2 in an afternoon. MODE3 requires additional infrastructure (tool integrations, an async queue) but no changes to the lower modes.

This progression matters for ecosystem adoption. The content quality caveat applies to all modes: a MODE1 site with a poorly-structured `llms.txt` provides minimal value to visiting agents regardless of manifest completeness. The five-minute compliance figure refers to the structural elements; high-quality content preparation is a separate investment.

### 5.5 Visiting Agent Side

The current AHP specification and reference implementation focus entirely on the *site side* — what a site must do to be AHP-compliant. The *visiting agent side* — how an agent discovers, selects, and interacts with AHP-compliant sites — is intentionally underspecified in v0.1.

Planned work for the visiting agent side includes:
- A reference visiting agent library (Python and TypeScript) that handles discovery, manifest parsing, capability selection, and session management
- Trust evaluation heuristics: how should a visiting agent decide whether to use a site's MODE3 capabilities?
- Cross-site session federation: how does an agent maintain context across multiple AHP sites in a single task?
- Standardised agent identity: the `requesting_agent` field is currently free-text; a v1 identity scheme (building on W3C DIDs [DID] or signed JWTs) would enable site-side access control and personalisation

### 5.6 Limitations

1. **Retrieval quality ceiling**: the v0 keyword overlap retrieval degrades on large, ambiguous corpora. Embedding-based retrieval is planned for v1.
2. **Conformance scope**: results cover only the reference implementation (co-developed with the spec). No independent third-party implementations have been tested. The test suite is open and accepts community submissions (see §4.1).
3. **Compact evaluation corpora**: both test sites use ~10–50KB corpora. At document-store scale (millions of tokens), the MODE2 retrieval architecture requires rethinking.
4. **Self-referential evaluation**: the AHP Specification Site evaluation queries AHP documentation about AHP. A more rigorous evaluation would use corpora and queries developed independently of the protocol authors. The Nate Jones cross-comparison is a step toward this (different domain, different author) but remains within the same reference deployment ecosystem.
5. **Server-side cost model**: at high query volumes, MODE2 server-side LLM costs are material. See §5.1 cost estimates.
6. **T17 (auth enforcement) is a known spec violation in the demo**: the reference implementation intentionally omits auth to simplify demo access. A production deployment must implement spec §5.3.
7. **T12 (quote calculation) required assertion tightening**: the original vocabulary-based assertion was a false positive. The tightened assertion (price regex + failure-phrase exclusion) is also imperfect — a sufficiently clever failure message could still pass. End-to-end testing with known inventory state remains the most reliable approach.
8. **RAG-baseline comparison is limited to keyword retrieval**: the RAG baseline uses simple keyword overlap scoring. A more capable RAG agent using embedding-based retrieval would likely use even fewer tokens per query with higher answer quality, making the AHP vs. RAG comparison more competitive in both directions.
9. **Known test suite coverage gaps (v5.3)**: two spec features are completely untested across all 24 tests: (a) **content type negotiation (spec §6.6)** — `accept_types`, `response_types`, and `unsupported_type` 400 error are unexercised; a server that omits the entire content type negotiation system passes all 24 tests; (b) **session time-based expiry (10-minute TTL)** — T18 verifies the 10-turn limit but not the TTL; a server with infinite session TTL passes all 24 tests. Both gaps are documented in the test suite JSON output under `known_coverage_gaps`.
10. **T21 §6.3 live-server behaviour unverified**: T21 confirms the server is spec-compliant (§6.3 says MAY — not required to clarify) but cannot verify whether a server that does implement clarification returns the correct format. T21b (v5.3) resolves the format coverage gap using a mock-response parser that validates all required fields (status, clarification_question, optional suggested_queries). The T21 advisory ✓ reflects server correctness; T21b ✓ confirms format spec coverage.

---

## 6. Related Work

**Cloudflare's Markdown for Agents** [CLOUDFLARE] introduced clean markdown serving as an agent-accessible alternative to HTML. AHP's MODE1 is directly compatible with this approach and positions it as the entry point of a larger protocol.

**llms.txt** [LLMSTXT] proposed a standard location for a site-level plain text document. AHP MODE1 treats `llms.txt` as a valid content endpoint. AHP provides the discovery layer, capability declaration, and upgrade path that `llms.txt` lacks.

**Model Context Protocol (MCP)** [MCP] defines a standard for LLM tool access. AHP MODE3 is complementary: where MCP standardises how an agent calls a tool server, AHP defines how a website-side agent is discovered, queried, and delegated to. A MODE3 concierge may itself use MCP to access its tools.

**OpenAI GPT Actions / Plugin specification** [GPTACTIONS]: the most widely deployed agent-web interaction mechanism prior to AHP. GPT Actions define structured API call schemas for LLM tool use. AHP differs by focusing on natural language querying through a concierge (rather than direct API schema exposure), by providing discovery and content signals, and by defining a progressive adoption path from static content through to agentic delegation.

**Schema.org / Google Structured Data** [SCHEMAORG]: the existing machine-readable web layer. AHP's content signals and manifest serve a complementary purpose — declaring AI-interaction preferences and capabilities rather than semantic entity types. A future revision will explore alignment between AHP content signals and Schema.org metadata conventions.

**ai.txt** [AITXT]: a competing convention for declaring AI usage preferences at the site level. AHP's `content_signals` block covers similar ground while being embedded in the discovery and interaction protocol rather than a standalone declaration file.

**IETF RFC 8288 — Web Linking** [RFC8288]: the formal specification for the HTTP `Link` header field used in AHP §3.5 (proactive manifest discovery). AHP uses `rel="agent-manifest"` as a link relation type; formal registration with IANA under RFC 8288 §2.1.1 is planned for v1.0.

**IETF RFC 8615 — Well-Known URIs** [RFC8615]: the formal specification for the `.well-known/` URI path convention that AHP relies on for manifest discovery. AHP's `/.well-known/agent.json` is a Well-Known URI and should be registered with IANA under this convention.

**robots.txt** — AHP's `/.well-known/agent.json` follows the same well-known URI convention and spirit as `robots.txt`, providing a standard machine-readable declaration at a predictable path.

**OpenAPI 3.1 / AsyncAPI 2.x** [OPENAPI]: AHP borrows capability declaration conventions from OpenAPI but targets natural language agent interaction through a concierge rather than direct structured API documentation for machine clients.

---

## 7. Conclusion

The web's interaction model was designed for human browsers. AI agents need something different: a protocol for negotiated, capability-aware, stateful interaction. AHP provides that protocol.

Against a naive full-document baseline, AHP MODE2 demonstrates a 75–80% token reduction across two sites (AHP Specification: 77.4%; Nate Jones AI-practitioner blog: 71.7% — 15:46 UTC v5.3 run). Against a RAG-baseline visiting agent, AHP uses 3–8× more tokens on compact corpora; its advantage is protocol-level, not efficiency-level: structured discovery, capability declaration, server-managed caching (~10ms p50 cache-hit), session management, content signals, and MODE3 capabilities.

We demonstrated **23/24 conformance** on the reference deployment (v5.4 test suite, T01–T23 + T21b): T17 (MODE3 auth) is the single failure — a CRITICAL security defect in the demo configuration. T21 is advisory (§6.3 MAY — live server does not trigger clarification); T21b (v5.4) confirms the §6.3 format spec is correctly validated via mock-parser; T23 (v5.4, new) confirms the proactive HTTP `Link` response header (AHP §3.5, RFC 8288) is present on all response types. Cache-hit latency averages 10ms p50 (n=9 confirmed hits). Cold-cache latency averages 3,806ms p50 (LLM inference). The MODE3 interaction model delivers real-time inventory, computation, and async human delegation unavailable to any document-retrieval approach.

AHP is designed to grow with the ecosystem. MODE1 compatibility with existing `llms.txt` deployments ensures the protocol can spread through the current base of agent-accessible sites. The extension mechanism in Appendix C allows new content types to be registered without breaking compatibility.

We invite the community to review the specification, run the test suite (v5.3) against their deployments, and submit conformance results at `github.com/AHP-Organization/agent-handshake-protocol`.

---

## References

[CLOUDFLARE] Cloudflare. "Markdown for Agents." Cloudflare Blog, 2024. https://blog.cloudflare.com/markdown-for-agents/

[LLMSTXT] Answer.AI. "llms.txt — A Standard for LLM-Accessible Content." 2024. https://llmstxt.org

[MCP] Anthropic. "Model Context Protocol." 2024. https://modelcontextprotocol.io

[GPTACTIONS] OpenAI. "GPT Actions." OpenAI Documentation, 2024. https://platform.openai.com/docs/actions

[SCHEMAORG] Schema.org. "Schema.org Structured Data." https://schema.org

[AITXT] ai.txt. "AI Usage Declaration Standard." https://aitxt.org (see also https://site.ai/aitxt)

[RFC8288] Nottingham, M. "Web Linking." IETF RFC 8288, 2017. https://www.rfc-editor.org/rfc/rfc8288

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
| Test Suite (v5.3) | https://github.com/AHP-Organization/test-suite |
| Raw Test Results | https://github.com/AHP-Organization/test-suite/tree/main/results |

---

*Agent Handshake Protocol — Draft 0.1 (revised 2026-02-22, v5.3 run 15:46 UTC)*
*© 2026 Nick Allain. Licensed under CC BY 4.0.*
