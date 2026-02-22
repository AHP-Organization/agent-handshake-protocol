# Agent Handshake Protocol: A New Contract Between AI Agents and the Web

**Nick Allain**
*agenthandshake.dev · github.com/AHP-Organization*

*Draft 0.1 — February 2026*

---

## Abstract

Current approaches to AI agent interaction with websites are fundamentally misaligned: agents receive entire documents when they need specific answers, parse HTML designed for human browsers, and have no standardised way to discover what a site can do on their behalf. We present the **Agent Handshake Protocol (AHP)**, an open specification defining structured discovery, conversational interaction, and agentic delegation between visiting AI agents and website-side concierge systems. Through a reference implementation tested across two independent sites, we demonstrate that AHP MODE2 reduces token consumption by **75–79%** compared to full-document parsing approaches, while delivering more precise answers. We further demonstrate MODE3, which enables capabilities — real-time data access, tool use, and human escalation — that are architecturally impossible with static content serving. AHP is designed for progressive adoption: a site can become MODE1-compliant in under five minutes, and each subsequent mode is backwards compatible.

---

## 1. Introduction

The web was designed for humans navigating with browsers. AI agents — autonomous systems that traverse the web as part of larger tasks — are increasingly being asked to use it too. The results are predictable: agents scrape HTML, attempt to parse layouts designed for visual rendering, and consume thousands of tokens on boilerplate before reaching the content they need.

Several partial solutions have emerged. Cloudflare's "Markdown for Agents" [CLOUDFLARE] converts pages to clean markdown on request. The `llms.txt` convention [LLMSTXT] provides a site-level content document in plain text. These approaches share a fundamental limitation: they are monologues. A document is thrown over a fence, and the agent must parse it entirely regardless of its actual query. There is no negotiation, no capability declaration, no stateful interaction.

This is the equivalent of responding to every question with the full contents of a library. It works, but poorly.

We ask a different question: **what if a website could talk back?**

AHP defines a protocol for exactly this. A site publishes a machine-readable manifest declaring its capabilities. Visiting agents discover the manifest, understand what the site can offer, and submit targeted queries — receiving precise, sourced answers rather than full documents. For sites with richer requirements, AHP's MODE3 enables the site-side concierge to use tools, call APIs, and escalate to humans on behalf of the visiting agent.

The implications extend beyond efficiency. As AI agents become the primary interface through which many users interact with the web, a site's AHP compliance determines whether those agents can find it, understand it, and act on its behalf. AHP is, in this sense, the infrastructure layer for agent-native web presence — the successor to SEO in a world where the agent is the user.

### 1.1 Contributions

This paper presents:

1. **The AHP specification** — a three-mode protocol covering discovery, conversational interaction, and agentic delegation
2. **A reference implementation** — a complete open-source MODE1/MODE2/MODE3 server deployable on any Node.js host
3. **Empirical evaluation** — conformance testing and token efficiency benchmarks across two independent sites
4. **A test suite** — an open-source visiting agent harness that any AHP implementer can run against their deployment

---

## 2. The AHP Protocol

AHP defines three modes of interaction, each building on the previous.

### 2.1 MODE1: Static Serve

In MODE1, the site publishes a machine-readable manifest at `/.well-known/agent.json` and provides static agent-readable content (a `llms.txt` file, markdown pages, or a JSON knowledge document). Interaction is read-only and stateless. A visiting agent fetches the manifest, discovers the content URL, and retrieves it.

MODE1 is directly compatible with existing `llms.txt` deployments. A site with an existing content document becomes MODE1-compliant by adding the manifest — no changes to the content document are required.

### 2.2 MODE2: Interactive Knowledge

MODE2 adds a conversational endpoint at `POST /agent/converse`. The visiting agent submits a query specifying a capability; the site-side concierge searches its knowledge base and returns a precise, sourced answer. Multi-turn sessions enable clarification when queries are ambiguous.

The critical distinction from MODE1: the visiting agent asks for what it needs, and the concierge returns only that. The agent never processes content it didn't ask for.

### 2.3 MODE3: Agentic Desk

MODE3 equips the site-side concierge with tools and async capabilities. The concierge can query databases, call external APIs, execute calculations, connect to MCP servers, and escalate to human operators — capabilities the visiting agent may not have access to.

MODE3 changes the nature of the interaction: the visiting agent is no longer just querying a knowledge base, but delegating tasks to a capable agent that represents the site.

### 2.4 Discovery

AHP defines four discovery mechanisms, ordered by agent type:

1. **In-page agent notice** — a `<section aria-label="AI Agent Notice">` in the page body, visible to headless browser agents reading rendered HTML
2. **HTML `<link>` tag** — `<link rel="agent-manifest">` in `<head>`, discoverable by DOM-parsing agents
3. **Accept header** — `Accept: application/agent+json` triggers a manifest redirect
4. **Well-known URI** — direct `GET /.well-known/agent.json` for agents that know to look

The in-page notice is the most novel of these: it places agent-targeted instructions in the rendered page content, ensuring that agents using headless browsers encounter discovery information without any prior knowledge of AHP.

### 2.5 Content Signals

AHP includes a standardised content signals sub-protocol, allowing sites to declare machine-readable preferences for AI usage of their content. Signals include `ai_train`, `ai_input`, `search`, and `attribution_required`. These signals are declared in the manifest and echoed in every response, creating a persistent, auditable record of usage intent.

---

## 3. Implementation

### 3.1 Reference Server

Our reference implementation is a Node.js/Express server (~700 lines across 7 source files) implementing all three AHP modes. Key components:

- **Knowledge base**: markdown content files, chunked by H2 section and scored by keyword overlap for retrieval (v0; vector embeddings planned for v1)
- **MODE2 concierge**: Claude Haiku via the Anthropic API; retrieves top-5 relevant chunks, synthesises a sourced answer
- **MODE3 tool use**: Claude's native tool_use API with an agentic loop; tools include inventory lookup, quote calculation, order retrieval, and knowledge search
- **Async queue**: human escalation tickets with configurable simulated response delay; fires callbacks or resolves via polling
- **Session management**: in-memory sessions with 10-minute TTL and 10-turn limit
- **Caching**: normalised query key, 5-minute TTL; cached responses return in <30ms
- **Rate limiting**: 30 req/min unauthenticated, with AHP-standard `X-RateLimit-*` headers

### 3.2 Test Sites

We deployed two independent instances with distinct content corpora:

| Site | Content | Corpus size | Chunks |
|------|---------|-------------|--------|
| AHP Specification Site | AHP protocol specification and documentation | ~50KB, ~12,400 tokens | 56 |
| Nate Jones (natebjones.com) | AI practitioner site — guides, projects, writing | ~39KB, ~9,900 tokens | ~50 |

The Nate Jones corpus was scraped from the live site using a custom extraction script, demonstrating that real-world web content requires minimal preparation to load into an AHP knowledge base.

### 3.3 Test Harness

Our test suite (`AHP-Organization/test-suite`) acts as a visiting agent, running 16 conformance tests covering all four discovery mechanisms, all three modes, error handling, sessions, content signals, rate limiting, and caching. It also runs whitepaper benchmarks: a token efficiency comparison against the `llms.txt` baseline and a latency profile.

---

## 4. Evaluation

### 4.1 Protocol Conformance

Both deployments passed all 16 conformance tests.

| Test | Category | AHP Site | Nate Site |
|------|----------|----------|-----------|
| T01: Well-known discovery | Discovery | ✓ | ✓ |
| T02: Accept header discovery | Discovery | ✓ | ✓ |
| T03: In-page agent notice | Discovery | ✓ | ✓ |
| T04: Manifest schema | Protocol | ✓ | ✓ |
| T05: MODE2 content query | Functionality | ✓ | ✓ |
| T06: Unknown capability error | Error handling | ✓ | ✓ |
| T07: Missing field error | Error handling | ✓ | ✓ |
| T08: Multi-turn session | Sessions | ✓ | ✓ |
| T09: Response schema | Schema | ✓ | ✓ |
| T10: Content signals in response | Signals | ✓ | ✓ |
| T11: MODE3 inventory (tool use) | MODE3 | ✓ | ✓ |
| T12: MODE3 quote calculation | MODE3 | ✓ | ✓ |
| T13: MODE3 order lookup | MODE3 | ✓ | ✓ |
| T14: MODE3 async human escalation | MODE3 async | ✓ | ✓ |
| T15: Response caching | Infrastructure | ✓ | ✓ |
| T16: Rate limiting (429) | Infrastructure | ✓ | ✓ |

**32/32 (100%)** across both sites.

### 4.2 Token Efficiency

For each of five representative queries, we measured token consumption for two approaches:

- **Markdown baseline**: the visiting agent fetches the full `llms.txt` document and processes it with the query. Token count = document tokens + query overhead.
- **AHP MODE2**: the visiting agent POSTs the query to `/agent/converse`. Token count = actual tokens consumed by the concierge (input + output, as reported by the API).

#### AHP Specification Site (corpus: ~12,400 tokens)

| Query | Markdown tokens | AHP tokens | Reduction |
|-------|----------------|------------|-----------|
| Explain what MODE1 is | 10,091 | 2,396 | **76.3%** |
| How does AHP discovery work for headless browser agents? | 10,092 | 1,967 | **80.5%** |
| What are AHP content signals and what do they declare? | 10,091 | 2,188 | **78.3%** |
| How do I build a MODE2 interactive knowledge endpoint? | 10,091 | 2,259 | **77.6%** |
| What rate limits should a MODE2 AHP server enforce? | 10,090 | 2,038 | **79.8%** |
| **Average** | | | **78.5%** |

#### Nate Jones Site (corpus: ~9,900 tokens)

| Query | Markdown tokens | AHP tokens | Reduction |
|-------|----------------|------------|-----------|
| Explain what MODE1 is | 9,898 | 2,079 | **79.0%** |
| How does AHP discovery work for headless browser agents? | 9,899 | 2,441 | **75.3%** |
| What are AHP content signals and what do they declare? | 9,898 | 2,702 | **72.7%** |
| How do I build a MODE2 interactive knowledge endpoint? | 9,898 | 2,249 | **77.3%** |
| What rate limits should a MODE2 AHP server enforce? | 9,897 | 2,634 | **73.4%** |
| **Average** | | | **75.5%** |

**AHP MODE2 reduces token consumption by 75–79% across two independent sites with different content.**

The reduction is consistent because it reflects a structural property of the approach: the visiting agent never processes content it didn't ask for. Regardless of how large the site's knowledge base grows, the AHP response targets the query. The markdown approach scales linearly with document size; the AHP approach is bounded by the answer complexity.

### 4.3 Latency Profile

We profiled latency across response types. Results show a clear two-tier structure:

| Response type | Observed latency |
|---------------|-----------------|
| Cached MODE2 | 9–31ms |
| Uncached MODE2 (Haiku) | 2,300–3,600ms |
| MODE3 sync (tool use, 1-2 tool calls) | 2,500–4,100ms |
| MODE3 async (human escalation) | ~8,100ms (simulated) |

**Caching is the primary latency lever for MODE2.** Common queries return in under 30ms — comparable to a static file serve. The cold-cache latency (2–4 seconds) is the LLM inference time; this can be reduced with a faster model (e.g. claude-haiku at its fastest) or pre-warmed caches for high-frequency queries.

MODE3 sync latency (2.5–4.1 seconds) reflects one to two tool call round trips plus synthesis. This is appropriate for queries requiring real-time data. MODE3 async latency is unbounded by design — it scales to however long a human takes to respond.

The Anthropic API accounts for most of the latency; a production deployment using a locally-hosted model would significantly reduce cold-cache times.

### 4.4 MODE3: Capabilities Beyond Static Content

MODE3 enables a qualitatively different class of interactions. We demonstrate three capabilities architecturally impossible with static content approaches:

**Real-time data access**: a visiting agent querying `inventory_check` receives current stock levels, pricing, and availability — data that would be stale in any static document. The concierge uses Claude's tool_use to call a structured inventory database, synthesise the result, and respond in natural language. Token cost: ~2,848–5,091 (2-4 tool call rounds).

**Quote calculation**: a visiting agent requesting pricing for a multi-item order receives a structured quote with volume discounts applied, valid for 48 hours, and a line-item breakdown. This involves a calculation that a static document cannot perform — the concierge calls a pricing engine, gets structured JSON, and explains the result. This class of interaction — *compute on behalf of the agent* — has no precedent in static web protocols.

**Human escalation**: a visiting agent can submit a question requiring human judgment, receive an `accepted` response with a polling URL and ETA, and retrieve the human's answer when ready. The async model (poll or webhook callback) handles both fast and slow responses gracefully. In our tests, the full round trip (submit → simulated human response → poll → resolved) completed in ~8.1 seconds. In production, the ETA reflects real human availability.

These three patterns represent a shift in the relationship between visiting agents and websites. The site is no longer a passive document repository — it is an active participant capable of doing work on the agent's behalf.

### 4.5 Content Signal Propagation

In all 32 test runs, content signals (`ai_train: false`, `ai_input: true`, `search: true`, `attribution_required: true`) were present in every response's `meta` object. This provides a machine-readable, per-response record of the site's AI usage policy — more precise than a site-level declaration in `robots.txt` and more persistent than a page-level meta tag.

The current implementation relies on the visiting agent honoring these signals. A future revision will explore cryptographic attestation of signal declarations and standardised logging requirements for agents that consume AHP endpoints.

---

## 5. Discussion

### 5.1 The Token Economy Argument

The 75–79% token reduction is a structural result, not an optimisation. When a visiting agent fetches a full document, it pays for every token in that document regardless of relevance. When it queries an AHP endpoint, it pays only for the tokens required to answer its question. As site corpora grow — from the 10–12K tokens in our test sites to the millions of tokens in a large documentation site or e-commerce catalogue — the gap widens further.

At scale, this has significant implications. An agent conducting research across 50 sites in a session could consume tens of millions of tokens fetching full documents. The same research via AHP MODE2 consumes a fraction of that, reducing both cost and latency.

### 5.2 AHP as the Infrastructure Layer for Agent-Native SEO

Search Engine Optimisation emerged because websites needed to be found by crawlers that served humans through a results page. As AI agents increasingly become the primary interface through which people interact with the web, the question shifts: how does a site make itself accessible, useful, and preferentially chosen by those agents?

AHP is the answer to that question. A site that implements AHP:
- Is discoverable by agents through four independent mechanisms
- Can declare exactly what it can do, in machine-readable form
- Returns answers an agent can act on, rather than documents an agent must parse
- Can offer capabilities competitors without AHP cannot provide at all (MODE3)

This is the agent-native equivalent of having a well-structured, fast-loading, properly-indexed site. The analogy is not metaphorical — it is structural. AHP compliance, for an agentic web, performs the same function as SEO compliance for the browser web.

### 5.3 Progressive Adoption

We designed AHP explicitly for progressive adoption. A MODE1 site adds one file (`/.well-known/agent.json`) and two HTML elements (a `<link>` tag and an in-page notice) to become compliant. Our reference implementation takes a developer from zero to MODE2 in an afternoon. MODE3 requires additional infrastructure (tool integrations, an async queue) but no changes to the lower modes.

This progression matters for ecosystem adoption. A protocol that requires full implementation to deliver any value will see slow uptake. A protocol where partial compliance is immediately useful — and each subsequent mode adds demonstrable capability — can spread organically.

### 5.4 Limitations and Future Work

The current evaluation uses a simulated human operator for MODE3 async escalation. Real human response times vary by orders of magnitude; the async model handles this gracefully by design, but the whitepaper latency figures for MODE3 async should not be taken as production performance estimates.

The knowledge retrieval in MODE2 v0 uses simple keyword overlap scoring. This produces adequate results for the corpora tested but will degrade on larger, more ambiguous corpora. A v1 implementation with embedding-based retrieval will address this and is expected to further improve token efficiency by enabling more precise chunk selection.

The token comparison methodology uses the visiting agent's full `llms.txt` as the markdown baseline. A more sophisticated markdown agent might implement its own chunking and retrieval, narrowing the gap. We plan to extend the comparison to include a RAG-capable visiting agent as a more competitive baseline in a future version of this paper.

---

## 6. Related Work

**Cloudflare's Markdown for Agents** [CLOUDFLARE] introduced clean markdown serving as an agent-accessible alternative to HTML. AHP's MODE1 is directly compatible with this approach and positions it as the entry point of a larger protocol.

**llms.txt** [LLMSTXT] proposed a standard location for a site-level plain text document. AHP MODE1 treats `llms.txt` as a valid content endpoint. AHP provides the discovery layer, capability declaration, and upgrade path that `llms.txt` lacks.

**Model Context Protocol (MCP)** [MCP] defines a standard for LLM tool access. AHP MODE3 is complementary: where MCP standardises how an agent calls a tool server, AHP defines how a website-side agent is discovered, queried, and delegated to. A MODE3 concierge may itself use MCP to access its tools.

**robots.txt** — AHP's `/.well-known/agent.json` follows the same well-known URI convention and spirit as `robots.txt`, providing a standard machine-readable declaration at a predictable path.

**OpenAPI / AsyncAPI** — AHP borrows capability declaration conventions from OpenAPI but targets natural language agent interaction rather than structured API documentation.

---

## 7. Conclusion

The web's interaction model was designed for human browsers. AI agents need something different: a protocol for negotiated, capability-aware, stateful interaction. AHP provides that protocol.

We demonstrated 75–79% token reduction versus full-document parsing across two independent sites, 100% conformance on a 16-test harness, sub-30ms cached latency, and a MODE3 interaction model that enables capabilities — real-time data, computation, human judgment — that static content approaches cannot provide.

AHP is designed to grow with the ecosystem. MODE1 compatibility with existing `llms.txt` deployments ensures the protocol can spread through the current base of agent-accessible sites. The extension mechanism in Appendix C allows new content types to be registered without breaking compatibility. The versioning policy ensures implementations can adopt the protocol incrementally.

We invite the community to review the specification, run the test suite against existing deployments, and contribute to the open standard at `github.com/AHP-Organization/agent-handshake-protocol`.

---

## References

[CLOUDFLARE] Cloudflare. "Markdown for Agents." Cloudflare Blog, 2024. https://blog.cloudflare.com/markdown-for-agents/

[LLMSTXT] Answer.AI. "llms.txt — A Standard for LLM-Accessible Content." 2024. https://llmstxt.org

[MCP] Anthropic. "Model Context Protocol." 2024. https://modelcontextprotocol.io

---

## Appendix: Artefacts

All artefacts from this paper are open source and publicly accessible.

| Artefact | URL |
|----------|-----|
| AHP Specification (Draft 0.1) | https://agenthandshake.dev/spec |
| JSON Schemas | https://agenthandshake.dev/schema/0.1/ |
| Reference Implementation | https://github.com/AHP-Organization/reference-implementation |
| Live Reference Endpoint | https://ref.agenthandshake.dev |
| Test Suite | https://github.com/AHP-Organization/test-suite |
| Raw Test Results | https://github.com/AHP-Organization/test-suite/tree/main/results |

---

*Agent Handshake Protocol — Draft 0.1*
*© 2026 Nick Allain. Licensed under CC BY 4.0.*
