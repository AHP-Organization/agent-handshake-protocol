# Agent Handshake Protocol: A New Contract Between AI Agents and the Web

**Nick Allain**
*agenthandshake.dev · github.com/AHP-Organization*

*Draft 0.1 — February 2026 (revised)*

---

## Abstract

Current approaches to AI agent interaction with websites are fundamentally misaligned: agents receive entire documents when they need specific answers, parse HTML designed for human browsers, and have no standardised way to discover what a site can do on their behalf. We present the **Agent Handshake Protocol (AHP)**, an open specification defining structured discovery, conversational interaction, and agentic delegation between visiting AI agents and website-side concierge systems.

Through a reference implementation tested against the AHP Specification Site, we demonstrate that AHP MODE2 reduces token consumption by **77–80% compared to a naive visiting agent that fetches and passes the full content document without retrieval** (9,700-token corpus, tiktoken estimate; 3-run mean ± stddev). We also report a RAG-baseline comparison — a visiting agent implementing client-side chunking and retrieval over the same `llms.txt` content — which is the fairer competitive benchmark: on this compact corpus, a keyword-retrieval RAG baseline uses 266–910 tokens per query versus AHP MODE2's 1,946–2,429. **For simple retrieval queries on small corpora, a well-implemented RAG visiting agent is more token-efficient than AHP MODE2.** AHP's advantages are protocol-level: structured discovery, capability declaration, session management, content signals, and MODE3 capabilities — none of which are available to a document-retrieval agent. We further demonstrate MODE3, which enables real-time data access, computation, and natural language delegation that static content and retrieval approaches cannot provide. AHP is designed for progressive adoption: a site can become MODE1-compliant in under five minutes, and each subsequent mode is backwards compatible.

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
4. **A test suite** — an open-source visiting agent harness (v2: includes RAG-baseline comparison, multi-run averaging, and extended conformance tests T01–T17)

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

#### 2.4.1 In-Page Notice and Headless Browser Agents

The in-page notice spec recommends the element be visually hidden (`display:none`) while remaining in the DOM. This ensures the notice does not disrupt human UX. However, headless browsers (Puppeteer, Playwright, Selenium) execute JavaScript and render the full DOM; agents using these frameworks may not encounter `display:none` content through normal text extraction. The notice is most reliably discovered by agents that read raw HTML source (e.g. via `curl` or `requests`). A future revision will provide clearer guidance distinguishing raw-HTML agents from fully-rendered headless browser agents.

### 2.5 Content Signals

AHP includes a standardised content signals sub-protocol, allowing sites to declare machine-readable preferences for AI usage of their content. Signals include `ai_train`, `ai_input`, `search`, and `attribution_required`. These signals are declared in the manifest and echoed in every response, creating a persistent, per-response record of usage intent.

The current implementation relies on the visiting agent honouring these signals. Signals of type `MUST respect` in the specification reflect the intent and aspirational compliance expectation for the ecosystem; technical enforcement is not part of v0.1. A future revision will explore cryptographic attestation. The `ai_train: false` and similar signals are best understood as SHOULD-level behavioural guidance for visiting agents until enforcement mechanisms are specified.

---

## 3. Implementation

### 3.1 Reference Server

Our reference implementation is a Node.js/Express server (~700 lines across 7 source files) implementing all three AHP modes. Key components:

- **Knowledge base**: markdown content files, chunked by H2 section and scored by keyword overlap for retrieval (v0; vector embeddings planned for v1). **Evaluation framing note**: the v0 keyword retrieval is adequate for the compact corpora tested (~10–50KB) but is expected to degrade on larger, more ambiguous content. This limitation is considered in the evaluation framing in Section 4.2.
- **MODE2 concierge**: Claude Haiku via the Anthropic API; retrieves top-5 relevant chunks, synthesises a sourced answer. Token costs appear on the server side — the visiting agent pays fewer tokens, but the site pays for each Haiku call. At low-to-moderate query volumes this is economically favourable; at high traffic volumes the server-side cost model requires analysis (see §5.1).
- **MODE3 tool use**: Claude's native tool_use API with an agentic loop; tools include inventory lookup, quote calculation, order retrieval, and knowledge search
- **Async queue**: human escalation tickets with configurable simulated response delay; fires callbacks or resolves via polling
- **Session management**: in-memory sessions with 10-minute TTL and 10-turn limit
- **Caching**: normalised query key, 5-minute TTL; cached responses return in <30ms
- **Rate limiting**: 30 req/min unauthenticated, with AHP-standard `X-RateLimit-*` headers

### 3.2 Test Sites

We deployed a reference instance with AHP protocol documentation as its content corpus:

| Site | Content | Corpus size | Chunks |
|------|---------|-------------|--------|
| AHP Specification Site | AHP protocol specification and documentation | ~50KB, ~12,400 tokens | 56 |

A second instance (Nate Jones — `nate.agenthandshake.dev`) hosts an AI practitioner blog corpus. Comparison queries on this site use AI-practitioner topics (RAG, prompt engineering, MCP, agentic systems) rather than AHP-specific queries, to test generalization to a different content domain.

### 3.3 Test Harness

Our test suite (`AHP-Organization/test-suite`, v2) acts as a visiting agent, running 17 conformance tests (T01–T17) covering all four discovery mechanisms, all three modes, error handling, sessions, content signals, MODE3 authentication enforcement, rate limiting, and caching.

The suite also runs whitepaper benchmarks:

- **Naive baseline**: tiktoken (cl100k_base) token estimate for a visiting agent that fetches the full `llms.txt` and passes it to an LLM without retrieval, plus a measured document fetch latency.
- **RAG-baseline**: a visiting agent that fetches `llms.txt`, chunks by paragraph (~500 chars), selects top-3 chunks by keyword overlap, and calls Claude Haiku with those chunks. This is the more competitive baseline representing a reasonably-implemented visiting agent.
- **AHP MODE2**: the visiting agent POSTs the query to `/agent/converse`. Token count = actual tokens reported by the API.

Each comparison query is run **3 times** and results are reported as **mean ± stddev**.

---

## 4. Evaluation

### 4.1 Protocol Conformance

The reference deployment passed all 17 conformance tests (T01–T16 from the original suite; T17 is a new MODE3 authentication enforcement test added in v2).

**Scope note**: conformance was demonstrated only for the reference implementation, which was co-developed with the spec and test suite. No independent third-party implementations have been tested. The 17/17 result reflects internal consistency of the reference implementation, not ecosystem-wide interoperability. Independent implementation testing is planned for v0.2.

| Test | Category | AHP Site |
|------|----------|----------|
| T01: Well-known discovery | Discovery | ✓ |
| T02: Accept header discovery | Discovery | ✓ |
| T03: In-page agent notice | Discovery | ✓ |
| T04: Manifest schema | Protocol | ✓ |
| T05: MODE2 content query | Functionality | ✓ |
| T06: Unknown capability error | Error handling | ✓ |
| T07: Missing field error | Error handling | ✓ |
| T08: Multi-turn session + memory | Sessions | ✓ |
| T09: Response schema | Schema | ✓ |
| T10: Content signals in response | Signals | ✓ |
| T11: MODE3 inventory (tool use) | MODE3 | ✓ |
| T12: MODE3 quote calculation | MODE3 | ✓ |
| T13: MODE3 order lookup | MODE3 | ✓ |
| T14: MODE3 async human escalation | MODE3 async | ✓ |
| T15: Response caching | Infrastructure | ✓ |
| T16: Rate limiting (429) | Infrastructure | ✓ |
| T17: MODE3 action auth enforcement | Security | ✓ |

**17/17 (100%)** on the reference deployment.

### 4.2 Token Efficiency

We compared token consumption for three approaches on five representative queries against the AHP Specification Site.

**Baseline definitions:**
- **Naive visiting agent (no retrieval)**: fetches the full `llms.txt` document (~40,900 chars / ~10,000 tiktoken tokens) and passes it with the query to an LLM. Token count estimated via tiktoken cl100k_base; not API-measured. *This is the baseline no competent agent implementation would use in production — it represents the upper bound on token waste and serves as a reference point, not a competitive comparison.*
- **RAG-baseline visiting agent**: fetches `llms.txt`, chunks by paragraph (~500 chars), selects top-3 chunks by keyword overlap, calls Claude Haiku. This is a **fair competitive baseline** representing a reasonably-implemented visiting agent. Token counts are API-measured. Run 3 times; reported as mean ± stddev.
- **AHP MODE2**: visiting agent POSTs query to `/agent/converse`. Token counts are API-measured. Run 3 times; reported as mean ± stddev.

#### AHP Specification Site (corpus: ~40,900 chars, ~9,727 tokens tiktoken)

Results from 2026-02-22 benchmark run. Naive baseline is a tiktoken (cl100k_base) estimate. RAG and AHP token counts are API-measured; reported as mean ± stddev across 3 runs.

| Query | Naive tokens* | RAG-baseline (3-run) | AHP MODE2 (3-run) | AHP vs. naive | AHP vs. RAG |
|-------|--------------|---------------------|-------------------|--------------|-------------|
| Explain what MODE1 is | ~9,727† | 266 ±19 | 2,429 ±0‡ | **75.1%** | −813% |
| How does AHP discovery work? | ~9,727† | 478 ±23 | 1,946 ±0 | **80.0%** | −307% |
| What are AHP content signals? | ~9,727† | 412 ±12 | 2,168 ±0 | **77.7%** | −426% |
| How do I build a MODE2 endpoint? | ~9,726† | 910 ±32 | 2,270 ±0 | **76.7%** | −150% |
| What rate limits should AHP enforce? | ~9,727† | 555 ±0 | 2,040 ±0 | **79.0%** | −268% |
| **Average** | | | | **77.7%** | |

† Naive token counts are tiktoken cl100k_base estimates (full document + query overhead); not API-measured. Error expected ±5–10%.

‡ Stddev of 0 indicates all 3 runs returned the same value — either identical API response or cache-hit behaviour on repeated queries.

**The naive baseline finding** (76–80% reduction) reflects a structural property: a visiting agent that fetches a full 9,700-token corpus to answer a 15-token query wastes most of what it fetches. This is the upper-bound case.

**The RAG-baseline finding** requires honest presentation: on this compact, well-structured corpus, a simple RAG visiting agent (3-chunk keyword retrieval, calling Claude Haiku directly) uses **significantly fewer tokens per query** (266–910) than AHP MODE2 (1,946–2,429). The RAG baseline narrows the gap to zero and inverts it. **AHP MODE2 uses approximately 2–9× more tokens than a well-implemented visiting-agent RAG pipeline on small corpora.**

This result is expected, not a failure. The trade is explicit:

- **RAG-baseline advantages**: lower per-query token cost for simple lookup queries on compact corpora; no server infrastructure required
- **AHP MODE2 advantages**: structured discovery and capability declaration (the visiting agent learns *what* the site can do); session management, content signals, error handling as protocol primitives; no client-side retrieval infrastructure to build or maintain per site; consistent answer quality across all corpora without client-side tuning
- **At scale**: on larger corpora (100K+ tokens), RAG retrieval quality degrades without embedding infrastructure. AHP offloads this to the server side, where it can use embeddings or hybrid retrieval without the visiting agent changing its implementation.
- **MODE3**: RAG is irrelevant here — no document retrieval approach can provide real-time data, computation, or human delegation.

**Evaluation framing**: the v0 keyword retrieval in the reference implementation is adequate for compact, well-structured corpora. On larger or more ambiguous corpora, retrieval quality will degrade, affecting AHP response quality. Embedding-based retrieval is planned for v1, which may reduce the AHP token cost by selecting better chunks and requiring fewer synthesis tokens.

### 4.3 Latency Profile

We profiled latency across response types. Results from 2026-02-22 run (5 samples; 20-sample runs included in v2 suite):

| Response type | Observed latency |
|---------------|-----------------|
| Cached MODE2 | 9–31ms |
| Uncached MODE2 (Haiku) | 2,300–4,500ms |
| MODE3 sync (tool use, 1–2 tool calls) | 2,000–3,500ms |
| MODE3 async (human escalation) | ~8,100ms [SIMULATED — see note] |

**Caching is the primary latency lever for MODE2.** Common queries return in under 30ms — comparable to a static file serve. The cold-cache latency (2–4.5 seconds) is LLM inference time.

**Note on MODE3 async latency**: the ~8,100ms figure is from a simulated human operator with a fixed programmatic delay. It is not a production measurement. In production, human escalation response time is measured in hours to days. The simulated figure is included for protocol completeness (demonstrating the round-trip mechanics) and should not be interpreted as a practical latency estimate.

**Note on p95**: with 5 samples, a statistically valid p95 cannot be computed — the reported p95 of 4,523ms equals the maximum of 5 samples. The v2 suite runs 20 latency samples and reports cached/uncached stats separately.

### 4.4 MODE3: Capabilities Beyond Static Content

MODE3 enables a qualitatively different class of interactions — ones that static content approaches do not support through the same protocol interface:

**Real-time data access**: a visiting agent querying `inventory_check` receives current stock levels, pricing, and availability — data that would be stale in any static document. The concierge uses Claude's tool_use to call a structured inventory database, synthesise the result, and respond in natural language. Token cost: ~2,848–5,091 (2–4 tool call rounds).

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

**Against the RAG-baseline (AHP uses 2–9× more tokens on compact corpora)**: our v2 benchmark shows that a simple 3-chunk keyword-retrieval visiting agent calling Claude Haiku uses 266–910 tokens per query on the AHP Specification corpus, versus AHP MODE2's 1,946–2,429 tokens. For pure retrieval efficiency on small, well-structured corpora, AHP MODE2 is less token-efficient than a well-implemented client-side RAG agent.

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

The server-side cost model also requires consideration: the visiting agent pays fewer tokens than a naive baseline, but the site pays for each Claude Haiku call. For high-traffic deployments, the per-query server-side LLM cost requires analysis. A v1 deployment guide will include cost modelling and guidance on when caching and pre-warming make MODE2 economically comparable to RAG-plus-static-serving.

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
2. **Conformance scope**: the 17/17 conformance result covers only the reference implementation. No independent third-party implementations have been tested against the test suite.
3. **Evaluation corpus**: both test sites use relatively compact corpora (~10–50KB). At document-store scale (millions of tokens), the MODE2 retrieval architecture requires rethinking.
4. **The token comparison uses the AHP corpus for queries on both sites.** A more rigorous independent evaluation would use corpora and queries developed separately from the protocol authors.
5. **Server-side cost model**: the visiting agent pays fewer tokens, but the site pays for Haiku inference. At scale, this cost model requires analysis before recommending MODE2 for high-traffic sites.

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

Against a naive full-document baseline, we demonstrated a 76–80% token reduction across the AHP Specification Site, 17/17 conformance on the reference deployment, sub-30ms cached latency, and a MODE3 interaction model that enables real-time data access, computation, and human delegation that static content approaches do not offer through the same protocol interface.

Against a RAG-baseline visiting agent (client-side chunking and retrieval), the advantage shifts from token efficiency to protocol-level benefits: structured discovery, capability negotiation, session management, content signals, and MODE3 access are provided as protocol primitives rather than reimplemented per site.

AHP is designed to grow with the ecosystem. MODE1 compatibility with existing `llms.txt` deployments ensures the protocol can spread through the current base of agent-accessible sites. The extension mechanism in Appendix C allows new content types to be registered without breaking compatibility. The versioning policy ensures implementations can adopt the protocol incrementally.

We invite the community to review the specification, run the test suite (v2) against existing deployments, and contribute to the open standard at `github.com/AHP-Organization/agent-handshake-protocol`.

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
| Test Suite (v2) | https://github.com/AHP-Organization/test-suite |
| Raw Test Results | https://github.com/AHP-Organization/test-suite/tree/main/results |

---

*Agent Handshake Protocol — Draft 0.1 (revised 2026-02-22)*
*© 2026 Nick Allain. Licensed under CC BY 4.0.*
