# AHP Whitepaper Critique

**Critiqued version:** AHP-WHITEPAPER-0.1.md (Draft 0.1, revised 2026-02-22)
**Critique date:** 2026-02-22 (07:30 UTC)
**Live test data:** `/tmp/ahp-report-latest.json` — present, run at 07:01 UTC against `https://ref.agenthandshake.dev`
**Previous critique:** 2026-02-22 06:30 UTC — this revision responds to several issues raised there

---

## Preface: What the Revision Fixed

The revised whitepaper addresses several critical problems from the prior version and deserves acknowledgement before the remaining issues are catalogued:

- ✓ The naive full-document baseline is now explicitly labelled as "the upper bound case, not a competitive comparison"
- ✓ A RAG-baseline comparison is added and honestly shows AHP uses 2–9× more tokens than a well-implemented client-side RAG agent on small corpora
- ✓ The token counting methodology references tiktoken instead of character division
- ✓ The conformance scope note correctly scopes results to the reference implementation
- ✓ GPT Actions, Schema.org, ai.txt, RFC 8615, OpenAPI, and DID are now cited
- ✓ The simulated human escalation latency is clearly flagged as simulated
- ✓ The in-page notice / headless browser ambiguity is acknowledged (§2.4.1)
- ✓ Content signal MUST/SHOULD tension is softened appropriately
- ✓ The p95-equals-max problem with 5 samples is acknowledged
- ✓ T17 (MODE3 auth enforcement) is added

What follows are the remaining critical and secondary issues — some new, some surviving the revision.

---

## Critical Problems (must fix)

### 1. T12 (MODE3 quote calculation) is a false pass — live data confirms it

The live test results show T12 passed with the following answer (from `test_results[11].notes`):

> *"I attempted to calculate a quote with the product IDs I inferred, but the system couldn't find those products. To provide you with an accurate quote…"*

The MODE3 quote operation **failed at the tool level**. The LLM couldn't resolve the product IDs from the query. Yet T12 reports `passed: true`. Why? The test assertion:

```python
price_mentioned = any(term in answer.lower() for term in [
    "$", "usd", "price", "cost", "quote", "total", "license", "support", "discount"
])
assert price_mentioned, ...
```

The word "quote" appears in the failure message ("accurate quote"), so the assertion fires green. This is a semantic false positive: the test is designed to verify that a quote was *produced*, but it passes when a quote was *requested but not delivered*.

The whitepaper's Section 4.4 states: *"a visiting agent requesting pricing for a multi-item order receives a structured quote with volume discounts applied."* The live run shows the opposite occurred. The whitepaper claim is empirically false for this test run, and the test suite would not catch this.

**Fix required**: T12 must assert specific structural content of the quote (numeric prices, line items, a total) — not just the presence of quote-adjacent vocabulary in an error message.

### 2. AHP MODE2 stddev of ±0 across all queries is suspicious and suggests cache contamination

The comparison table reports "3-run mean ± stddev" for AHP MODE2 with ±0 on every single query. LLM outputs are non-deterministic; even if the Haiku model generates similar answers, token counts vary by tens to hundreds of tokens across runs. Stddev of ±0 means all three runs returned identical token counts.

The most likely explanation: runs 2 and 3 are cache hits. The reference server has a 5-minute cache TTL on normalised query keys. If three consecutive comparison queries are run against the same endpoint without cache-busting, the second and third runs return cached responses that are byte-for-byte identical to run 1 — hence ±0. In this scenario, the "3-run mean" is just the first run repeated three times, and the reported stddev is measuring cache consistency, not answer variance.

The paper presents ±0 as evidence of stability. It may instead be evidence that only 1 real LLM call was made per query. This needs to be verified and disclosed. If cache-busting is not used between runs, the multi-run methodology provides no additional information over a single run.

**Fix required**: verify whether comparison runs use cache-busting (unique nonces, timestamps, or deliberate cache invalidation). If not, add cache-busting and re-run. Report the real stddev.

### 3. The "Nate Jones" second site has been quietly dropped but the paper claims generalisation it no longer demonstrates

The previous version claimed "two independent sites with different content." The revised paper removes the Nate Jones benchmark data entirely — the comparison table now shows only the AHP Specification Site. The second site (`nate.agenthandshake.dev`) is mentioned in Section 3.2 but produces no data in Section 4.2.

A single-site benchmark does not demonstrate generalisability. The Abstract's "reference implementation tested against the AHP Specification Site" is accurate, but the paper's broader efficiency claims are now supported by exactly one site — the site created specifically to host AHP documentation, queried exclusively with AHP-specific questions. This is self-referential evaluation.

The Conclusion states: *"we demonstrated a 76–80% token reduction across the AHP Specification Site"* — which is at least now accurate — but Section 4.2's framing still implies general validity. **Any evaluation section claiming efficiency advantages needs at least one non-AHP content domain with domain-appropriate queries.** The Nate Jones data should either be properly presented (with non-AHP queries) or the generalisability claim should be explicitly limited to AHP-domain content.

### 4. The live test rate-limit hit at 8 requests, not 30 — creating a non-deterministic T16

From `test_results[15].notes`: *"429 hit after 8 requests"*

The reference server advertises 30 req/min unauthenticated (Section 3.1, rate_limit header), but the test triggered a 429 after only 8 requests. This discrepancy likely reflects carry-over from prior test requests consuming the per-IP window, since T16 runs last and the suite makes 20–30+ requests before reaching it. The test still passes (a 429 was received), but:

1. The whitepaper claims the rate limit is 30/min; the test shows it fires at 8 in practice
2. T16 is non-deterministic — its outcome depends on how many requests the suite consumed earlier in the run
3. The test never verifies the *actual limit value* (`X-RateLimit-Limit` header) against the manifest's declared `rate_limit`

A suite that runs T16 last will always have used most of the window already. The rate-limit test needs to run on a fresh IP or after a window reset, and it should verify the declared limit matches what the headers report.

### 5. The in-page notice "fix" (§2.4.1) effectively concedes the feature doesn't work for modern agents

The revised §2.4.1 states: *"The notice is most reliably discovered by agents that read raw HTML source (e.g. via `curl` or `requests`). A future revision will provide clearer guidance."*

This is a significant concession: the in-page notice — Discovery Mechanism 1, described as "the most novel" of the four in the original paper — is explicitly acknowledged to not work for headless browser agents (Puppeteer, Playwright, Selenium), which is the dominant implementation pattern for agents that "interact with websites via headless browsers" as described in the intro. The target audience (headless browser agents) cannot reliably see a `display:none` element.

The solution proposed — use `curl`/`requests` agents — is not "agents interacting via headless browsers." It's just standard HTTP. These agents are already handled by mechanisms 3 and 4 (Accept header, well-known URI).

The in-page notice as currently specced reduces to: a discovery mechanism for HTTP-scraping agents that don't know to look for `.well-known/agent.json` (mechanism 4). That's a narrow use case that doesn't justify its billing as the primary headless-agent discovery path. Either:
- The spec should change the notice to use visible text (which disrupts human UX) or a non-hidden ARIA pattern that works in the rendered DOM
- Or the in-page notice should be reframed as a "fallback for raw-HTML scrapers" rather than primary headless agent discovery

The current T03 test remains trivially satisfied by passing for any API server — the exemption logic hasn't changed in the test code.

### 6. The latency comparison is still structurally unfair to AHP

The revised paper adds a RAG-baseline for token comparison but still provides no latency comparison against that baseline. The live data shows AHP MODE2 cold latency of 3,000–4,700ms. A RAG visiting agent calling Claude Haiku client-side would have approximately equal latency (fetch static file ~50ms + Haiku call ~2,500ms = ~2,600ms) and use 3–8× fewer tokens.

The Section 4.3 latency table presents only AHP latency numbers. A reader comparing AHP to RAG on both dimensions (tokens and latency) would need to compute the RAG-baseline latency themselves. Given that AHP uses more tokens AND adds latency overhead (server-side retrieval + synthesis + network round-trip to the concierge), presenting latency in isolation is selective. The paper should include a side-by-side latency comparison in the same table as the token comparison.

---

## Secondary Issues

### 7. T11 called 4 tools for a simple inventory query — MODE3 orchestration quality not evaluated

From live `test_results[10].notes`:
> `tools_used: ['check_inventory', 'calculate_quote', 'check_inventory', 'calculate_quote']`

The query was *"Is the AHP Server License Single Site in stock? How much does it cost?"* — a straightforward lookup. The concierge called `check_inventory` and `calculate_quote` each twice before producing an answer. This is inefficient orchestration: the same tools called twice suggests the LLM is looping (possibly because the first tool result wasn't sufficient and it retried). The whitepaper reports token cost for this as 5,071 tokens — the second-highest cost of any test.

The paper doesn't evaluate MODE3 orchestration quality — only that tool calls were made and an answer was produced. A well-orchestrated MODE3 concierge should answer this query with 1 tool call (or 2 at most if inventory and pricing are separate lookups). The current result suggests prompt engineering or tool design issues in the reference implementation that deserve mention.

### 8. The paper still uses "tiktoken estimate" for the naive baseline without providing the actual estimated figure in the table

The table footnote says "†Naive token counts are tiktoken cl100k_base estimates (full document + query overhead); not API-measured. Error expected ±5–10%." But the comparison table shows values like `~9,727` for all five naive-baseline rows — these differ by only 0–1 tokens between queries (reflecting only the query length difference). This still looks like a character-count approximation renamed as a "tiktoken estimate," since actual tiktoken counts for subtly different queries would vary more due to subword tokenization differences on the query text.

If tiktoken was actually used, showing the estimated document token count (without query overhead) once as a corpus-level constant, and showing actual per-query tiktoken totals in each row, would be more credible than five nearly-identical values.

### 9. The RAG-baseline stddev for "What rate limits should AHP enforce?" is ±0 — same suspicious pattern

The RAG-baseline column shows ±0 for "What rate limits should a MODE2 AHP server enforce?" This same query also shows ±0 for AHP MODE2. Having both baselines show zero variance on the same query either means: (a) both cached, (b) the query is deterministic enough that LLM outputs are token-for-token identical across 3 runs (possible but unlikely given LLM temperature), or (c) the 3-run methodology isn't actually running 3 independent LLM calls. This warrants explanation.

### 10. Section 4.2's "AHP vs. RAG" percentages (-150% to -813%) need axis labeling

The table shows "AHP vs. RAG" column with values like "-813%" and "-307%". These are negative numbers meaning "AHP uses 813% MORE tokens than RAG for this query" — which is the correct honest presentation. However, the table mixes "AHP vs. naive: 75–80%" (where higher is better for AHP) with "AHP vs. RAG: -150% to -813%" (where higher is worse for AHP) in adjacent columns without a header clarifying the direction. A casual reader may interpret both as "larger number = better."

The table should label the "AHP vs. naive" column as "Token reduction (vs. naive)" and the "AHP vs. RAG" column as "Token overhead vs. RAG" or use a consistent directionality. The sign convention should be consistent or clearly annotated.

### 11. "5-minute compliance" claim needs qualification in the adoption section

Section 5.3 states: *"The content quality caveat applies to all modes: a MODE1 site with a poorly-structured `llms.txt` provides minimal value to visiting agents regardless of manifest completeness. The five-minute compliance figure refers to the structural elements; content preparation is a separate investment."*

This qualification is good but buried. The abstract and introduction still lead with "a site can become MODE1-compliant in under five minutes." A reader who stops at the abstract takes away an ease-of-adoption message that the content caveat substantially undermines. The abstract should include the caveat.

### 12. The paper mentions "v2 suite" with 20-sample latency runs, but live data shows 5 samples

The paper (Section 3.3, Section 4.3) references the "v2 suite" running 20 latency samples and reporting cached/uncached stats separately. The live test data (`latency_profile`) contains exactly 5 samples, with 4 of them being cache hits. Either the live data is from the v1 suite (not v2), or the v2 improvements haven't been deployed. The paper should not cite v2 features as implemented fixes if the live deployment still uses v1.

### 13. The visiting agent side is underspecified in a way that affects the evaluation

Section 5.4 acknowledges the visiting agent side is "intentionally underspecified in v0.1." This is understandable for a protocol draft. However, it creates an evaluation gap: the test suite acts as the visiting agent, but it's a purpose-built harness that knows AHP intimately. A real visiting agent (GPT, Claude, Gemini acting as an orchestrator) would need to:
- Discover the manifest without prior AHP knowledge
- Parse the capability schema and decide what to query
- Handle session management and error recovery

None of this agent-side behaviour is tested. The 17/17 conformance result reflects server-side protocol compliance. It says nothing about whether the protocol is discoverable and usable by real-world visiting agents encountering it in the wild.

---

## What the Tests Need

### Remaining missing coverage

1. **Fix T12's assertion to test actual quote delivery, not vocabulary presence.** The current assertion fires on error messages containing the word "quote." At minimum, assert that the answer contains a numeric price (`\$\d` or similar) and doesn't contain "couldn't find" or "unable to calculate."

2. **Add cache-busting to the token comparison multi-run methodology.** Each run should use a unique session or nonce to ensure fresh LLM calls. Without this, the ±0 stddev is an artefact, not a measurement.

3. **T16 (rate limiting) must run on a fresh window.** Either run it first (before other requests consume the window), or add a mandatory wait for window reset, or use a dedicated IP/key that hasn't been used in the current window. The current "run last" placement makes the test non-deterministic and the 8-request trigger is not the 30-request limit being tested.

4. **Add a latency comparison for the RAG-baseline.** The token comparison table has a RAG-baseline column; the latency profile doesn't. Either add RAG-baseline latency to the comparison, or explicitly note why it's excluded.

5. **T08 (multi-turn) must verify session memory continuity.** The current test only checks that `session_id` echoes back. Add a turn 1 that asks about a specific named topic, then a turn 2 that asks "what did you just tell me about?" and assert the turn 2 answer references turn 1 content. A server that returns a stable `session_id` without actually using session context would pass the current test.

6. **T03 (in-page notice) needs a configurable HTML page URL.** The current exemption for non-HTML servers (returning `True` if root isn't HTML) allows any API-style server to bypass the test permanently. If the site serves HTML on any page (e.g. `/index.html` or a docs URL), that page should be checked. Add a `--html-page` parameter for test configuration.

7. **Missing: clarification_needed response flow.** Section 6.3 of the spec defines a full clarification protocol. No test exercises this path. An implementation that never returns `clarification_needed` would pass all 17 tests despite being spec-non-compliant.

8. **Missing: content type negotiation.** Section 6.6 defines a full negotiation protocol for non-text response types. Zero tests exercise `accept_types`, `response_types`, `accept_fallback`, or `unsupported_type` (400) errors. The entire content type system in the spec is untested.

9. **Missing: session limit enforcement.** No test sends 11 turns to verify the 10-turn limit fires. No test waits 11 minutes to verify the 10-minute TTL fires.

10. **Missing: T17 should also test that T11/T12/T13 *reject* unauthenticated requests.** Currently T11–T13 pass without auth (suggesting these capabilities don't require it), while T17 tests a separate auth-required capability. It's unclear whether `inventory_check`, `get_quote`, and `order_lookup` are properly classified as query (auth optional) vs. action (auth required) per the spec. The test suite should make the capability type explicit and test accordingly.

11. **Add request-size limit test (413 response for oversized bodies).** Section 6.5 specifies an 8KB cap. No test sends a payload >8KB.

### Methodology improvements still needed

12. **Add a latency percentile that is statistically valid.** The p95 issue is acknowledged but a note saying "v2 runs 20 samples" isn't a fix until v2 is actually deployed. Verify the deployed test suite matches the paper's v2 claims, or retract the v2 latency claim.

13. **Verify tiktoken usage with a test that shows corpus token count separately from query overhead.** The near-identical `~9,727` values across all queries with only ±1 token variation suggests character-based estimation is still in use. If tiktoken is properly implemented, show the output of a standalone tokenisation call on the corpus document as a sanity check in the paper.

14. **Run T12's MODE3 quote test with a correctly specified product ID** to distinguish a passing quote calculation from a passing error message. The test should use a product ID known to exist in the reference server's inventory.

---

## What the Whitepaper Needs

### Remaining required changes

1. **Acknowledge the T12 result honestly.** Either (a) note that the live run showed a failed quote operation that nonetheless passed the test (and fix the test), or (b) run the suite again after fixing T12 and report corrected results. The current Section 4.4 claim that *"a visiting agent requesting pricing for a multi-item order receives a structured quote with volume discounts applied"* is contradicted by the live test data.

2. **Clarify the stddev=0 situation.** Either explain why all 3 AHP runs produce identical token counts (cache hit disclosure), or re-run with cache-busting and report real variance. A methodology note should appear in the table or footnotes.

3. **Add a latency column to the token comparison table**, or add a separate table. AHP is 3–4.5 seconds cold for every query where a RAG visitor would be ~2.5 seconds. This material fact belongs in the efficiency analysis, not only in the latency profile section.

4. **The Nate Jones site needs to actually provide data or be removed.** The paper mentions it in Section 3.2 as a second test site with "AI-practitioner topics" but the benchmark tables show zero results from it. This is referenced infrastructure that contributes nothing to the evaluation. Either present the data or drop the site from the paper until it's evaluated properly.

5. **Fix Section 4.2's column direction labeling** for "AHP vs. naive" vs "AHP vs. RAG" to make clear that one percentage shows AHP improving and the other shows AHP degrading. A reader scanning the table should not need to infer sign convention from context.

6. **Move the content-quality caveat on five-minute adoption to the abstract.** The abstract creates a strong ease-of-adoption impression that Section 5.3 partially walks back. This is the kind of gap between abstract and body that reviewers penalise.

7. **Clarify T08 improvement in v2 suite.** The paper says T08 is now "multi-turn session + memory" but doesn't describe what memory verification was added. The live test data shows no memory assertion evidence. If v2 adds this, describe it explicitly.

8. **In-page notice (§2.4.1) needs a positive path forward, not just a concession.** "A future revision will provide clearer guidance" is insufficient when the feature is listed as a primary discovery mechanism. Options include:
   - Specify a non-hidden fallback element (e.g. footer landmark) that works in rendered DOM
   - Add a `<meta name="ahp-manifest">` tag as a fifth discovery mechanism specifically for rendered-HTML agents
   - Clearly demote the in-page notice to "raw-HTML-scraper" tier in the discovery priority ordering

9. **The server-side cost analysis promised in §5.1 should include concrete numbers.** The section says a v1 deployment guide will include cost modelling but gives no estimates. A rough calculation belongs in the paper: at X queries/day, MODE2 costs approximately $Y/month in Haiku API calls, which becomes economically comparable to static serving at Z query volume with W cache hit rate. Without numbers, the cost discussion is aspirational.

10. **Expand the conformance section to describe a path toward independent implementation testing.** The honest "reference implementation only" scoping is good. The paper should also describe what would be required to claim ecosystem-level interoperability: what is the process for a third-party implementer to submit their conformance results? Is there a hosted test runner? This makes the "we invite the community to run the test suite" call-to-action actionable.

---

## Summary of Most Critical Issues

- **T12 is a false pass confirmed by live data.** The quote operation failed ("couldn't find those products") but the test passed because the word "quote" appeared in the error message. Section 4.4's claim that MODE3 quote calculation works is empirically false for this test run. This is the most acute immediate issue.

- **The ±0 stddev across all AHP MODE2 runs almost certainly reflects cache hits, not measurement stability.** If 2 of 3 "runs" per query are cache-served responses, the multi-run methodology provides no additional information. The RAG-baseline ±0 on at least one query raises the same concern. The paper should explicitly confirm or deny cache-busting between runs.

- **A single-site evaluation (AHP Spec Site queried about AHP) does not demonstrate generalizability.** The Nate Jones data was removed but not replaced. The only efficiency numbers in the paper come from a corpus that is literally the subject of the queries. The evaluation needs at least one non-AHP domain with domain-appropriate queries before efficiency claims can be taken seriously.

- **The in-page notice concession in §2.4.1 effectively removes Discovery Mechanism 1 from relevance for headless browser agents** — which are the stated primary audience for that mechanism. The paper should either fix the mechanism or reclassify it honestly in the discovery priority ordering.

- **AHP is slower on latency AND uses more tokens than a RAG-baseline for the queries tested, but the paper presents only one half of this comparison.** Latency numbers for the RAG baseline are absent from the paper. A complete comparison would show AHP's token overhead versus RAG alongside an honest latency comparison that acknowledges MODE2 cold-cache adds 1–2 seconds over a client-side RAG call.
