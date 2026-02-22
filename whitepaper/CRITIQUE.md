# AHP Whitepaper Critique

**Critiqued version:** AHP-WHITEPAPER-0.1.md (Draft 0.1, February 2026)
**Critique date:** 2026-02-22
**Test results file:** `/tmp/ahp-report-latest.json` — NOT FOUND (no live test data available; critique based on whitepaper claims and test suite code analysis)

---

## Critical Problems (must fix)

### 1. The token efficiency benchmark is a straw-man comparison

The headline claim — "75–79% token reduction" — is produced by comparing AHP against a baseline that no competent agent implementation would use: fetching the entire `llms.txt` document and passing it raw to the LLM with no retrieval.

The test runner (`run_token_comparison`) makes this explicit:
```python
markdown_base_tokens = len(full_content) // 4  # whole document, no retrieval
md_tokens = markdown_base_tokens + len(query) // 4 + 50  # query overhead added
```

A visiting agent with a 20-line RAG implementation over the same `llms.txt` content would retrieve 3–5 relevant chunks and pay ~2,000–3,000 tokens per query — nearly identical to what AHP MODE2 charges. The whitepaper acknowledges this in Section 5.4 ("A more sophisticated markdown agent might implement its own chunking and retrieval, narrowing the gap") but buries it in a limitations paragraph after making the 75–79% claim the paper's centrepiece. The limitation invalidates the headline result; it should be in the abstract.

The paper is comparing a sophisticated server-side RAG system (MODE2 with Claude Haiku doing retrieval + synthesis) against a naive "no retrieval" agent. That is not a fair comparison of protocols — it is a comparison of retrieval strategies. AHP the protocol deserves a fair evaluation.

### 2. The two "independent" test sites share a corpus problem

Every single comparison query on the Nate Jones site (`natebjones.com`, described as "AI practitioner site — guides, projects, writing") is about the AHP protocol itself:

- "Explain what MODE1 is"
- "How does AHP discovery work for headless browser agents?"
- "What are AHP content signals and what do they declare?"
- "How do I build a MODE2 interactive knowledge endpoint?"
- "What rate limits should a MODE2 AHP server enforce?"

A personal practitioner blog would not normally contain detailed, queryable answers to AHP-specific technical questions. The site was almost certainly seeded with AHP documentation during the "custom extraction script" scraping process described in Section 3.2. The claim of "two independent sites with different content" is undermined by both sites answering the same AHP-specific queries with high confidence. This should be verified and disclosed.

More fundamentally, using only AHP-self-referential queries on both sites tests nothing about generalization to real-world web content. The entire evaluation is AHP querying about AHP.

### 3. Token counts are character-approximations reported as precise measurements

The whitepaper presents numbers like "10,091 tokens" and "9,898 tokens" with false precision. The actual methodology in the test runner is:

```python
markdown_base_tokens = len(full_content) // 4  # ← character count / 4
md_tokens = markdown_base_tokens + len(query) // 4 + 50
```

This is a rough approximation (1 token ≈ 4 chars). Real tokenizer counts (tiktoken for Claude's cl100k encoding) differ materially from this estimate. The "suspiciously round" consistency of the markdown token counts across all five queries (differing by only 1–2 tokens per query for the same site) is direct evidence they are all `(same_constant) + (slightly different query length)`, not actual measured token counts. Do not present character-divided-by-four as measured token consumption.

The AHP token counts are actual API-reported values (which is fine), but the comparison is between a measured quantity and an approximation. The tables in Section 4.2 should not be presented as equivalent measurements.

### 4. The latency comparison fabricates the baseline figure

The test runner hardcodes:
```python
md_latency = 200.0  # flat estimate for fetching a static doc
```

This is not measured. The "markdown agent" latency in any comparison result in the generated report is a constant the developer invented, not an observed value. The whitepaper does not surface this — Section 4.3 presents a "Latency Profile" comparing observed MODE2 latencies without disclosing that the markdown baseline was never measured. A hostile reviewer who reads the test code (which is published) will immediately flag this.

### 5. The 100% conformance result tests only the reference implementation against its own spec

The "32/32 (100%) across both sites" result is testing a server that was co-developed with the spec against a test harness that was co-developed with the same spec. This is guaranteed to produce 100%. No third-party implementation has been tested. No independent interoperability has been demonstrated. This is unit testing dressed as an interoperability result.

A credible conformance claim for a protocol paper requires at minimum two independent implementations passing the test suite. Even if no third-party implementation exists yet, the paper should clearly state that conformance was demonstrated only for the reference implementation, not for independent implementations of the spec.

### 6. The test suite contains a spec-violating omission in MODE3 security tests

The AHP specification (Section 5.3) states: *"Capabilities of type 'action' or 'async' MUST require authentication (authentication MUST NOT be 'none')"*

The test suite calls `test_inventory_check`, `test_get_quote`, and `test_order_lookup` (all MODE3 capabilities) with no authentication headers. These tests expect `200 success` responses. If the reference server passes them (as claimed), it is violating its own spec's mandatory authentication requirement for MODE3 action capabilities. There is no test anywhere in the suite that verifies MODE3 action capabilities *reject* unauthenticated requests. This is a critical gap: the spec makes a security requirement and the test suite never exercises it.

### 7. The human escalation claim is simulation theatre

The MODE3 async claim — a full round trip completing in "~8.1 seconds" — is from a simulated human with a fixed delay. Section 5.4 acknowledges this but doesn't quantify what "real human availability" means. In production, human response time is measured in hours to days. The ~8 second figure gives the impression of a practical, near-real-time capability. Showing the simulated latency in Table 4.3 alongside real measured latencies (without distinguishing "simulated fixed delay" from "measured") is misleading.

### 8. Test IDs are swapped between T15 and T16

In the test runner code, `test_rate_limiting` returns `TestResult("T15", ...)` but is invoked and labeled as T16 in `run_all_tests`. Conversely, `test_caching` returns `TestResult("T16", ...)` but is invoked as T15. Any test reports generated will show pass/fail against incorrect test IDs. The conformance table in Table 4.1 maps `T15: Response caching` and `T16: Rate limiting`, but the code has these backwards. This is a code bug that affects every reported result.

---

## Secondary Issues

### 9. T02 (Accept header discovery) passes even on failure

```python
def test_accept_header(client):
    try:
        manifest = client.discover_via_accept_header()
        ...
        return TestResult("T02", ..., True, ...)
    except Exception as e:
        return TestResult("T02", ..., False, error=str(e),
            notes="Site may not implement Accept header redirect (optional)")
```

The except branch returns `passed=False`, which is correct. However, the runner's `run_test` helper catches `AssertionError` and `Exception` and returns `passed=False` — but the function itself catches exceptions and returns `passed=False` via the dataclass. This means T02 failures are reported in the result but `notes` says "optional," which is misleading: the spec says SHOULD (not MAY), implying strong recommendation. A site that completely ignores the Accept header should not be described as passing with a note about being optional — it's a conformance issue.

### 10. T03 (in-page agent notice) is trivially bypassable

```python
is_html_site = "text/html" in content_type and r.status_code < 400
if not is_html_site:
    return TestResult("T03", ..., True, notes="Not applicable — API server exempt")
```

A site that returns any non-HTML response from `/` is exempt from this test. This means a site could strip its HTML responses or serve JSON from `/` and permanently pass T03 without implementing in-page notices. The test should check for the notice on a known HTML page (e.g., `/index.html`, or a configured test URL), not just assume that any API-shaped server is exempt.

### 11. T08 (multi-turn session) doesn't test session memory

The test verifies that `session_id` echoes back on turn 2 — not that the server actually uses the session context. A server that stores sessions but ignores them (or even a server that just echoes back any session_id it receives) would pass this test. The test should verify that turn 2 references or builds on turn 1 content — e.g., ask "Tell me about AHP modes" in turn 1, then ask "Which one did you just describe?" in turn 2 and verify the answer reflects turn 1 context.

### 12. The latency profile p95 calculation is statistically invalid

```python
"p95_ms": round(sorted(latencies)[int(len(latencies) * 0.95)], 1) if len(latencies) >= 2 else latencies[-1]
```

With 5 samples: `int(5 * 0.95) = int(4.75) = 4`. This returns `sorted(latencies)[4]`, which is the maximum value. p95 equals p100 with 5 samples. The table in Section 4.3 is presenting maximums labeled as 95th percentiles.

### 13. The "progressive adoption" argument conflates protocol and implementation

Section 5.3 claims "a site can become MODE1-compliant in under five minutes." This is true only if (a) the site already has `llms.txt` content and (b) the developer knows what they're doing. The claim refers to adding the manifest file and two HTML elements — but it ignores content quality. A MODE1 site with a badly structured `llms.txt` provides minimal value. The adoption story overstates ease while understating the content preparation work.

### 14. The spec has a section numbering bug

Section 14 is titled "Examples" but its subsections are numbered `13.1` and `13.2`. This is a copy-paste error in the specification document that affects its professional credibility.

### 15. Missing related work

The paper cites only four references. A systems/ML venue reviewer would expect engagement with:
- **OpenAI GPT Actions / Plugin spec**: The most widely deployed agent-web protocol before the AHP submission. AHP should compare against it explicitly.
- **Google Structured Data / Schema.org**: The existing machine-readable web layer. AHP's discovery and content signals should be positioned against this.
- **OpenAPI 3.1 / AsyncAPI 2.x**: The standard for structured API capability declaration. AHP capability schemas resemble these but don't cite them.
- **W3C DID / Verifiable Credentials**: The trust/identity future work (Section 8.3) should reference existing identity standards it may build on.
- **IETF RFC 8615** (Well-Known URIs): The `.well-known/` convention AHP relies on has a formal RFC that should be cited.
- **Agentic frameworks (AutoGPT, LangGraph, CrewAI)**: How do these existing agent orchestration systems interact with web content? What does adoption look like?
- **ai.txt**: A competing convention for declaring AI usage preferences.

### 16. Content signal enforcement is a MUST with no teeth

The spec states: *"Visiting agents and downstream systems MUST respect ai_train: false"* and immediately notes *"AHP does not enforce this technically."* A MUST-level requirement that the protocol itself acknowledges cannot be enforced is meaningless from a compliance standpoint. This should either be downgraded to SHOULD (which accurately reflects that it's a best-effort signal), or the future work on cryptographic attestation should be moved to a concrete proposal rather than a one-line aspiration.

### 17. Token budget / cost analysis is absent at any realistic scale

Section 5.1 extrapolates from 10–12K token corpora to "millions of tokens in a large documentation site." This claim is made without any data. At 1M token corpora, the MODE2 concierge's keyword-overlap retrieval will degrade significantly (acknowledged in Section 5.4), meaning the token efficiency gap may not hold at scale. There is no analysis of concierge-side LLM API costs — the visiting agent pays fewer tokens, but the site now pays Claude Haiku for every query. For high-traffic sites, the MODE2 cost model may be worse than serving static files.

### 18. The in-page agent notice visibility guidance is contradictory

The spec says the notice SHOULD be visually hidden (`display:none`) but MUST remain in the DOM. Then: *"The text MUST NOT be removed by JavaScript hydration in a way that eliminates it from the rendered text visible to headless browsers."* But headless browsers (Puppeteer, Playwright, Selenium) execute JavaScript and render the full DOM, including `display:none` elements. The notice will not be "visible to headless browsers" if it's hidden — unless the LLM is reading raw HTML source. The spec conflates raw HTML parsing with headless browser rendering. This needs a clear technical distinction.

---

## What the Tests Need

### Missing test coverage

1. **Authentication enforcement for MODE3 actions**: A test that calls `inventory_check`, `get_quote`, or `order_lookup` *without* auth and asserts a `401` response. This is the most critical missing test — the spec mandates it and the current suite never checks it.

2. **`clarification_needed` response flow**: The spec defines a full clarification protocol (Section 6.3). No test exercises this. A valid MODE2 implementation that never returns `clarification_needed` would pass all tests but not be fully compliant.

3. **Session expiry (10-minute TTL)**: No test verifies that sessions expire. A server could keep sessions indefinitely and pass all tests.

4. **10-turn session limit**: No test sends 11 turns to verify the limit is enforced.

5. **Content type negotiation**: Section 6.6 defines a full negotiation protocol. Zero tests exercise `accept_types`, `response_types`, `accept_fallback`, or `unsupported_type` errors.

6. **Version mismatch**: No test sends `"ahp": "99.0"` and verifies graceful fallback per Section 12.

7. **Request body cap (8KB)**: No test sends an oversized body to verify `413` handling.

8. **SSRF protection**: The spec (Section 13) requires callback URL validation. No test sends a malicious callback URL.

9. **Token budget exhaustion**: Section 11.4 defines `rate_limited` with `scope: session_tokens`. No test exercises this.

10. **The `sources` field in responses**: T09 only checks that `answer` or `payload` exists. No test verifies that sourced answers actually provide meaningful `sources` with valid URLs.

### Methodology fixes required

11. **Replace hardcoded markdown latency with measured values**: The comparison benchmark must actually fetch the document and time it, not use `md_latency = 200.0`.

12. **Replace character-count token estimation with actual tokenizer**: Use `tiktoken` or the Anthropic token-counting API endpoint. The current approximation (`len // 4`) is not appropriate for published results.

13. **Run token comparisons multiple times and report mean ± stddev**: LLM responses vary in length. Single-run token counts don't reflect the variance of the approach.

14. **Test third-party implementations**: The suite should be run against at least one independent implementation to claim interoperability rather than just self-compliance.

15. **Fix the T15/T16 ID swap** in the test runner code.

16. **Make T02 and T03 report accurate pass/fail**: T02 should distinguish "optional feature absent" from "pass" — perhaps with a `warning` status. T03 should accept a configurable HTML page URL to check rather than only the root.

17. **Fix the p95 calculation** — either use 20+ samples for latency profiling, or label the reported value correctly as "max of N samples" not "p95."

---

## What the Whitepaper Needs

### Methodology

1. **Replace the straw-man baseline**: Add a "RAG-capable markdown agent" baseline that fetches `llms.txt`, chunks it, and runs embedding similarity retrieval — the same retrieval approach AHP uses on the server side. This is the fair comparison. The claim that AHP provides a protocol-level benefit (not just a retrieval-location benefit) requires demonstrating an advantage over a client-side equivalent.

2. **Either clarify or retract the "independent sites" claim**: If the Nate Jones site has been seeded with AHP documentation, this must be disclosed. If not, explain how a personal practitioner blog answers AHP-specific technical questions with confidence.

3. **Report actual tokenizer counts**: Replace all token counts derived from character division.

4. **Report distributions, not single runs**: For token efficiency and latency, show mean and standard deviation across at least 5 runs per query.

5. **Remove or clearly label simulated results**: The ~8.1 second MODE3 async latency from a simulated human operator should not appear in the same table as real measured latencies without clear labeling.

### Framing and claims

6. **Move the RAG-baseline acknowledgment to the abstract**: The paper's headline claim is invalidated by the limitation disclosed in Section 5.4. Either the abstract should be rewritten to frame the claim correctly ("compared to a no-retrieval baseline") or the paper needs the fair baseline comparison.

7. **Tone down Section 5.2 ("AHP as Infrastructure Layer")**: This section is marketing copy. "AHP is the answer to that question" is not analysis. Structural analogies to SEO require evidence. Either back this up with adoption data or move it to a blog post.

8. **Clarify conformance scope**: Every claim of "conformance" or "100% pass rate" should be explicitly scoped to "reference implementation only — no third-party implementations tested."

9. **Address the site-side cost model**: Who pays for the Claude Haiku calls on the concierge side? At what query volume does MODE2 become more expensive than static serving? This is a real adoption barrier that should be addressed.

10. **Fix the spec section numbering error** (13.1/13.2 under section 14) before submission.

11. **Cite IETF RFC 8615** for the `.well-known/` URI convention — this is a required reference for any protocol using that mechanism.

12. **Expand related work** to include GPT Actions, Schema.org, ai.txt, and at minimum acknowledge the trust/identity prior art (DIDs, signed JWTs).

13. **Clarify the in-page notice / headless browser contradiction**: The current guidance is technically imprecise about what "headless browser agents" can and cannot see with `display:none` content.

14. **Downgrade content signal MUST to SHOULD**, or commit to a concrete enforcement mechanism with a timeline.

---

## Summary of Most Critical Issues

- **The 75–79% token reduction claim is against a baseline that no real agent would use.** A client-side RAG agent over the same `llms.txt` would achieve near-identical efficiency. The paper's headline result measures retrieval strategy location, not protocol value.

- **The token counts in the comparison tables are not measured values** — they are character-count approximations (`len(content) // 4`) presented with false precision, and the markdown latency comparison uses a hardcoded constant of 200ms, not a measured figure.

- **The "two independent sites" claim is undermined** by both sites answering only AHP-specific queries, including detailed questions that a personal blog corpus would not naturally contain.

- **The test suite has a swapped T15/T16 ID bug**, never tests that MODE3 actions require authentication (the spec's primary security requirement), and uses statistically invalid p95 calculations on 5-sample data.

- **100% conformance is tested only against the reference implementation** that was co-developed with the spec. No independent interoperability has been demonstrated.
