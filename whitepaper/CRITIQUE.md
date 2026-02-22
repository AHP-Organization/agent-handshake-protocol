# AHP Whitepaper Critique

**Critiqued version:** AHP-WHITEPAPER-0.1.md (Draft 0.1, revised 7 — 14:33 UTC)
**Critique date:** 2026-02-22 (covers queued 1:39 PM and 2:42 PM sessions — consolidated)
**Live test data:** `/tmp/ahp-report-latest.json` — present, run at 14:38 UTC (v5.2 suite, same code as 12:26 run; test runner last modified 12:20)
**Sequence:** Code committed 12:20 → test ran 14:38 → whitepaper updated 14:42. Correct post-commit order maintained.
**Prior critiques:** 06:30, 07:30, 08:30, 09:30, 10:30, 11:33 UTC

---

## Cumulative Status: The Paper Is Nearly Ready

After seven revisions and nine critique cycles, the fundamental structural, methodological, and data integrity issues identified early have all been resolved. Specifically:

- ✓ Straw-man naive baseline honestly labelled as upper-bound reference, not competitive comparison
- ✓ RAG-baseline is API-measured, cache-busted, 3-run mean ± stddev — the correct competitive comparison
- ✓ AHP uses 3–8× more tokens than RAG on compact corpora: prominently stated, not buried
- ✓ Cache-busting nonces eliminating ±0 stddev artifacts — variance is real LLM variance
- ✓ Cold/hot latency split (10+10): genuine bimodal profile with cold p50 = 3,711ms and cached p50 = 29ms
- ✓ T08 session memory confirmed working (server correctly cites prior turn content)
- ✓ T20 rate-limit header conformance confirmed
- ✓ T12 quote assertion validated with real price table
- ✓ T17 CRITICAL DEFECT prominently and correctly labelled
- ✓ Three consistent conformance figures (21/22) throughout abstract, body, and conclusion
- ✓ Test ran post-commit in all recent cycles
- ✓ Related work comprehensive; conformance scope note accurate; limitations properly disclosed

What follows documents the remaining genuine issues — fewer and smaller than in prior cycles, but worth addressing before the paper is submitted anywhere.

---

## Critical Problems (must fix)

### 1. The "rate limits" query RAG latency has 68% CV driven by a single 3× outlier — and the table hides it

The "What rate limits should a MODE2 AHP server enforce?" query shows RAG latency of:
- Run 1: **3,234ms**
- Run 2: 1,066ms
- Run 3: 1,140ms
- Mean: 1,813ms ±1,231ms, **CV = 68%**

One run (run 1) took 3.23 seconds — nearly 3× the other two runs, which cluster at ~1.1 seconds. The reported mean (1,813ms) is 64% higher than the actual typical RAG latency for this query (~1.1s). The table shows "1,813ms" with no indication of the outlier. A reader interpreting RAG latency for this query as ~1.8s is materially misled; the genuine expectation is ~1.1s.

This is the worst data quality point in the entire benchmark. For comparison, AHP latency on the same query shows only ±107ms CV (3.9%) — AHP is dramatically more consistent than the RAG baseline on this query, but that difference is an infrastructure artefact (Anthropic API variability on a RAG call can spike), not a protocol property.

The run 1 outlier is almost certainly an Anthropic API cold-start or transient rate-limit event during the benchmark execution. This happens and is expected; the problem is not that it occurred, but that the table presents the distorted mean without disclosure.

**Fix required**: Add a footnote to the rate-limits row: *"†RAG run 1 recorded 3,234ms vs ~1,100ms for runs 2–3; mean (1,813ms) is inflated by a likely transient API event. Excluding the outlier, RAG latency is ~1,100ms for this query."* Alternatively, if 4-run capability is added to the suite, this is exactly the kind of query that benefits from an additional run to detect outliers. Without the note, the RAG latency comparison for this query overstates the gap.

The same issue at smaller scale affects "AHP discovery" query RAG latency (run 3 = 2,880ms vs ~1,860ms for runs 1–2; CV=27%) and "content signals" (runs: 995/1,365/783ms; CV=28%). These are less extreme but the same class of problem. Given the discovery query appears in the table with RAG latency of 2,199ms — when typical is ~1,850ms — the same footnote approach would improve it.

### 2. The latency stddev example values in the §4.2 footnote are stale from an intermediate run

The §4.2 table footnote states: *"Latency stddev: both RAG and AHP latency stddev values reflect genuine LLM inference variance (e.g. AHP discovery ±341ms; RAG endpoint ±353ms)."*

The 14:38 live run shows:
- AHP discovery latency stddev: **±125ms** (not ±341ms)
- RAG MODE2 endpoint latency stddev: **±414ms** (not ±353ms)

These example values don't match either the 12:26 run (AHP discovery ±386ms, RAG endpoint ±93ms) or the 14:38 run. They appear to come from an intermediate test run — likely the 1:39 PM run that would have run between these two — whose JSON is no longer the latest file. The whitepaper updated at 14:42 uses the 14:38 data for the table but retained the footnote examples from a prior run.

This is a data consistency issue: the paper claims to be reporting the "14:33 UTC — v5.2 suite" run, but the stddev examples in the footnote are from a different run. A reader who attempts to verify the example values against the live JSON would find a discrepancy.

**Fix required**: Update the example stddev values in the footnote to match the 14:38 run: *"(e.g. AHP discovery ±125ms; RAG discovery ±590ms — reflecting a 2,880ms spike on run 3 vs ~1,860ms typical)."* This simultaneously fixes the stale examples and surfaces the most notable variance in the table.

---

## Secondary Issues

### 3. The RAG latency column in the comparison table shows only mean, not stddev — asymmetric disclosure

The table header shows "RAG-baseline ± σ / (lat.)" where ± σ refers to token stddev and (lat.) is the mean latency only. AHP latency stddev is not shown explicitly in the table either, but the paper's footnote discusses AHP latency variance. The RAG latency variance is not discussed anywhere except the proposed footnote fix in issue #1 above.

Given that RAG latency has CVs ranging from 7% to 68% across queries (vs. AHP's 2.6% to 15.9%), and given that the RAG latency is used in §4.3's competitive comparison table ("RAG visiting agent: ~1,048–2,878ms"), showing mean RAG latency without variance is materially less informative than the AHP presentation. The range "~1,048–2,878ms" for RAG latency spans from the lowest mean (MODE1 query: ~1,221ms) to the highest mean (MODE2 endpoint query: ~2,878ms) — but the actual per-query variance means some of those means are themselves questionable. Adding a single parenthetical `(lat. mean; see footnote)` for the most variable rows would be sufficient disclosure.

### 4. The Nate site "prompt engineering" query has consistently high AHP latency (~5s) with no explanation

Across two consecutive runs (12:26 and 14:38), "Explain prompt engineering" on the Nate site shows stable AHP latency of approximately 5 seconds:
- 12:26 run: 4,221ms / 4,428ms / 4,376ms (mean 4,342ms)
- 14:38 run: 4,999ms / 5,213ms / 4,808ms (mean 5,007ms)

The 14:38 run is ~15% higher than 12:26, but both are consistently the highest-latency query in the Nate benchmark. The paper reports 5,007ms in the Nate table without noting that this query is persistently the slowest.

"Prompt engineering" likely produces a longer, more detailed AHP response (the concierge has more to explain), which drives up output tokens and hence latency. This is the answer-complexity explanation mentioned for the §4.3 profile vs. §4.2 table discrepancy. But the specific query's consistent behaviour across runs makes it worth a brief note: *"'Prompt engineering' is consistently the highest-latency Nate query (~5s), reflecting a longer synthesised answer."* Without this, readers may view the 5,007ms row as an anomaly rather than a reproducible complexity-driven observation.

### 5. T21 is structurally incapable of verifying §6.3 format for a well-functioning server, and the ✓ in the table overstates coverage

T21 sends "Which mode should I use?" and reports advisory pass because the reference server answers the question directly (explaining all three modes). The spec says the server MAY return `clarification_needed`. A server that never clarifies is fully spec-compliant — so T21 is correct to pass.

But the conformance table marks T21 as "✓ Advisory" next to "Clarification needed — format valid if triggered (spec §6.3)." A practitioner reading the table gets the impression that §6.3 was evaluated. It was not. The `clarification_needed` response format — its required fields, structure, and suggested_queries format — was never exercised by any test in the 22-test suite.

This is a spec design consequence: a protocol feature that is MAY will never trigger in a server that correctly tries to answer questions helpfully. T21 will be "advisory" for any competent AHP server.

The honest table entry is "N/A (server answered directly; §6.3 format unverified)" rather than "✓ Advisory." The distinction matters for practitioners implementing the `clarification_needed` flow — they cannot rely on the conformance table to tell them if their format is correct.

**Longer-term fix**: Add a T21b that directly sends a mock `clarification_needed`-structured response to a format-parsing function and verifies the required fields. This tests the spec in isolation from server behaviour.

### 6. The cached_p50 varies notably across runs (10ms–29ms) — worth one explanatory sentence

The cache-hit p50 has ranged across v5.x runs:
- 11:13 run: 12.1ms
- 12:26 run: 11.5ms
- 14:38 run: 28.5ms

The abstract says "~29ms p50" (current run). The abstract from the 11:33 critique said "~12ms p50." These vary nearly 3× despite being from the same caching infrastructure. The paper acknowledges this in §4.3 ("cache-hit p50 varies between runs (10ms–29ms) reflecting server response overhead variability at sub-100ms levels") — this note is good. But "server response overhead variability" is vague. The most likely cause is server-side event loop queue depth at test time (different numbers of background processes, garbage collection, etc.). A short explanation would give practitioners realistic expectations: their deployed server's cache-hit latency at 15ms vs 30ms is normal and reflects server load at query time, not the caching mechanism quality.

---

## What the Tests Need

### The two documented coverage gaps remain unaddressed

Both are correctly listed in `known_coverage_gaps` in the JSON output and in §5.5 (limitation 9). They are the only substantive spec features completely absent from all 22 tests:

1. **Content type negotiation (spec §6.6)**: `accept_types`, `response_types`, `accept_fallback`, and `unsupported_type` (400) are unexercised. A server that omits the entire content type negotiation system passes all 22 tests.

2. **Session time-based expiry (10-minute TTL)**: T18 verifies the 10-turn limit (correctly). No test verifies the TTL. A server with an infinite session TTL passes all 22 tests.

These are not blocking for the current paper — they're honestly disclosed and the paper's scope is explicitly limited. But they should be in the v5.3 or v6 milestone.

### T21 improvement (from secondary issue #5)

3. **Add T21b** as a format-parser unit test that validates the `clarification_needed` response structure (required fields: `status`, `clarification_request`, `suggested_queries`) against a mock response, independent of whether the live server ever returns one.

### Latency methodology improvement

4. **Add outlier detection to the 3-run benchmark.** When any single run in a 3-run sequence is more than 2× the median of the three (as in the rate-limits RAG run 1 = 3,234ms vs median ~1,103ms), flag it automatically in the JSON output as a suspected outlier and note that the mean is likely inflated. This is a simple improvement that would have automatically surfaced the 68% CV issue without requiring manual analysis.

---

## What the Whitepaper Needs

### Required before submission

1. **Add a footnote for the rate-limits RAG latency outlier** (run 1 = 3,234ms, inflating mean from ~1.1s to 1.8s). One sentence in the table footnotes is sufficient. Without it, the §4.3 RAG latency comparison range includes a distorted mean.

2. **Update the §4.2 footnote example stddev values** to match the current run: replace "AHP discovery ±341ms; RAG endpoint ±353ms" with values from the 14:38 run. The current examples are from an intermediate run that is no longer the backing data.

3. **Reconsider the T21 conformance table entry** from "✓ Advisory" to something that clearly communicates "§6.3 format unverified." The current ✓ overstates what was tested for readers scanning the table.

### Improvements to strengthen the paper

4. **Add "prompt engineering" consistently high latency note to Nate table footnote**: one sentence explaining that this query's ~5s latency is reproducible and reflects a longer synthesised answer, not infrastructure variance.

5. **Expand the cache-hit latency variability note in §4.3**: the existing note mentions "10ms–29ms across v5.x runs" but doesn't explain why. A sentence on server event-loop queue depth at test time would make this more actionable for practitioners.

6. **The §4.3 RAG latency range "~1,048–2,878ms"** currently represents the range of per-query means. After the rate-limits outlier footnote is added, consider whether the range should quote the "typical" value for that query (~1.1s) rather than the inflated mean (1.8s). The range endpoint would then be ~1,048–2,878ms but the rate-limits entry would be noted as "typical ~1.1s."

---

## What's Been Definitively Resolved — Full Accounting

The following issues identified across nine critique sessions are now fully resolved and should not be revisited:

| Issue | First Raised | Resolved |
|-------|-------------|---------|
| Naive full-doc straw-man as headline claim | 06:30 | ✓ 07:30 |
| Token counts = character/4 approximation | 06:30 | ✓ 07:30 |
| Hardcoded 200ms markdown latency | 06:30 | ✓ 07:30 |
| Nate site querying only AHP-specific topics | 06:30 | ✓ 08:30 |
| T12 false-pass on error vocabulary | 07:30 | ✓ 08:30 |
| T15/T16 test ID swap | 06:30 | ✓ 07:30 |
| RAG baseline absent (API key missing) | 08:30 | ✓ 09:30 |
| ±0 stddev from cache contamination | 08:30 | ✓ 09:30 |
| Three inconsistent conformance numbers | 09:30 | ✓ 10:30 |
| Code committed after test ran (4 consecutive cycles) | 09:30 | ✓ 11:33 |
| T08 false-pass (no memory, "FIRST" keyword) | 07:30 | ✓ 10:30 |
| T08 false-negative ("FIRST QUESTION" exclusion) | 10:30 | ✓ 11:33 |
| T21 and T22 claimed but not run | 10:30 | ✓ 11:33 |
| Latency profile all-cached (no cold measurements) | 08:30 | ✓ 11:33 |
| T20 rate-limit headers "unknown" | 09:30 | ✓ 11:33 |
| MODE1 query AHP latency 31% CV (7.5s outlier) | 11:33 | ✓ 12:37 (clean run) |
| Appendix stale version number | 08:30 | ✓ 11:33 |
| Missing related work | 06:30 | ✓ 07:30 |
| Content signals MUST/SHOULD tension | 06:30 | ✓ 07:30 |
| Single-site evaluation only | 07:30 | ✓ 08:30 |
| p95 = max with 5 samples | 06:30 | ✓ 09:30 |

---

## Summary of Most Critical Remaining Issues

**The paper is substantively sound.** The core claims are well-supported, the evaluation methodology is honest and complete, the limitations are clearly disclosed, and the data is clean and reproducible. The remaining issues are documentation refinements, not methodological flaws.

- **The "rate limits" query RAG latency (1,813ms reported) is inflated by a 3.23-second outlier run** — the other two runs show ~1.1 seconds, meaning the mean overstates typical RAG latency by 65% for that row. A one-sentence footnote would resolve this. It is the most significant data accuracy issue remaining in the paper.

- **The §4.2 footnote cites stale stddev example values** from an intermediate run (±341ms, ±353ms) that match neither the 12:26 nor 14:38 run. These need to be updated to the current run's values before submission.

- **T21 marks §6.3 as ✓ Advisory, but the clarification_needed response format was never tested.** The ✓ overstates conformance coverage for a spec feature that only validates when triggered, and the server correctly never triggers it for any test query. The table entry should clearly indicate the format is unverified.

- **Two spec features remain untested in all 22 tests** (§6.6 content type negotiation and §5.4.3 session TTL) — correctly documented in `known_coverage_gaps` and §5.5. These are the only remaining unverified spec areas and should appear on the v5.3 milestone.

- **The Nate site "prompt engineering" query's consistent ~5s AHP latency** is reproducible across runs and reflects answer complexity, not variance — worth a brief explanatory note in the table footnotes.
