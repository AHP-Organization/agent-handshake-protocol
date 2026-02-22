# AHP Whitepaper Critique

**Critiqued version:** AHP-WHITEPAPER-0.1.md (Draft 0.1, revised 5 — 11:07 UTC)
**Critique date:** 2026-02-22 11:33 UTC
**Live test data:** `/tmp/ahp-report-latest.json` — present, run at 11:13 UTC (v5.1 suite)
**Sequence:** Code committed 11:07 → test ran 11:13 — **first post-commit run across all critique cycles**
**Prior critiques:** 06:30, 07:30, 08:30, 09:30, 10:30 UTC

---

## Status: The Structural Problem Is Resolved

Every prior critique cycle identified the same root issue: test code was committed *after* the test ran, and the paper claimed results from a version of the suite that had never executed. That pattern ends here.

The v5.1 test suite code was committed at 11:07. The test ran at 11:13 — six minutes later. For the first time, the live JSON is a genuine v5.1 run, not a v4 run with editorial corrections. Consequently:

- T08 **passes** in the live data: `"You asked specifically about **MODE1**. In your first message, you explicitly requested: 'Tell me about AHP modes — focus especially on MODE1.'"` — the server has session memory, and the v5 fix correctly recognises it.
- T22 **passes** in the live data: server returned HTTP 400 on unknown session_id; the v5.1 fix accepting any 4xx works correctly.
- T21 **runs** in the live data: the test executes and returns advisory pass as designed.
- All 22 tests appear in the JSON. The summary: **21/22 passed (95.5%), T17 ✗**.
- All benchmark token counts match the paper's tables exactly (AHP 2,403 ±17 vs. live 2402.7 ±16.8 etc.).

The paper's core claims are now backed by clean, genuine test data. What follows are the remaining issues — considerably smaller in scope than in prior iterations, but still worth addressing before submission.

---

## Critical Problems (must fix)

### 1. The MODE1 query AHP latency has a 31% coefficient of variation driven by a 7.5-second outlier

The "Explain what MODE1 is" query shows AHP latency of 5,278ms / 4,062ms / 7,470ms across three runs — a ±1,727ms standard deviation and 31% coefficient of variation. Run 3 at 7,470ms is more than double the cold-cache p50 of 3,436ms (from the latency profile) and nearly double run 2.

This is the single worst-variance data point in the entire benchmark. Every other AHP site query has CV ≤ 17%; every Nate site query has CV ≤ 17%. The MODE1 query at 31% CV stands apart. The mean of 5,603ms is substantially inflated by the outlier — the median of the three runs is 5,278ms, still significantly above the profile p50.

The paper reports `5,603ms ±1,727ms` in the latency column of Table 4.2, which is internally consistent, but two things are missing:

1. **No acknowledgement of the outlier.** A 7.5-second response for a straightforward content query is anomalous. It may reflect an API cold start, transient rate-limit, or network event on run 3. The paper should note this: *"Run 3 of the MODE1 query recorded 7,470ms vs. 4,062–5,278ms for the other runs, likely reflecting a transient API event; this inflates the mean and stddev for this row."*

2. **The abstract and conclusion use "cold-cache latency averages 3,436ms p50"** (from the latency profile) as the representative cold-cache figure. This is correct for the profile queries. But the MODE1 comparison-table latency of 5,603ms is substantially higher than this figure and uses different queries. A reader comparing the abstract's "3,436ms p50" to the comparison table's "5,603ms" for MODE1 will see an apparent contradiction. The paper should clarify that the profile p50 (3,436ms) comes from the latency profile queries (short, uniform-format queries) and the comparison table shows per-query latency for evaluation queries, which vary based on answer complexity.

The most conservative fix: re-run the MODE1 query once (4 total runs, drop the outlier) or add a footnote explaining the 7,470ms run. A 31% CV in a 3-run benchmark is high enough that a reviewer will notice.

---

## Secondary Issues

### 2. T21 is advisory-only and the ✓ in the conformance table overstates coverage of spec §6.3

T21's live result: *"Server answered without clarification (status=success, mode=MODE2). Advisory: clarification_needed flow (spec §6.3) not triggered by test queries. Server is not REQUIRED to clarify — spec says MAY."*

The test passes in two distinct situations: (a) the server returns `clarification_needed` with a correct format (genuine §6.3 coverage), or (b) the server never returns `clarification_needed` at all (advisory — spec compliant per MAY). Every run will take path (b) because the reference server appears not to return clarification for any test query, including the maximally ambiguous "What is it?".

A ✓ in the conformance table implies that §6.3 was verified. It was not. The server is spec-compliant *because the spec says MAY*, not because the test verified anything. A practitioner reading the conformance table would expect that T21 confirms the clarification response format works. It confirms only that the server chose not to use it.

**Two options:**
- Change T21 to "N/A" or "Advisory (spec §6.3 MAY — not triggered)" in the conformance table, removing the ✓ that implies positive verification
- Add a second test that injects a mock `clarification_needed` response to test the format parser in isolation, which would provide genuine §6.3 coverage

The current framing is technically accurate per the spec's MAY, but it's the weakest test in the suite and the ✓ inflates the effective coverage of the protocol.

### 3. The paper says "1–2 tool call rounds" for MODE3 but live data shows exactly 1 consistently

Section 4.4 states: *"Token cost: ~2,400–3,500 (1–2 tool call rounds in v5 test runs)."* The live data shows:
- T11 (inventory): `tools_used: ['check_inventory']` — 1 tool call
- T12 (quote): `tools_used: ['calculate_quote']` — 1 tool call
- T13 (order): `tools_used: ['lookup_order']` — 1 tool call

Three consecutive tests, each 1 tool call. The "1–2 rounds" language is hedging from a prior test run where T11 showed 4 tool calls (now fixed). The current reference server consistently completes single-domain MODE3 queries in 1 tool call. The paper should update to "1 tool call per query in v5.1 test runs" and remove the "–2" hedge.

This matters because the 1 vs. 2 tool call distinction directly affects the MODE3 token cost range (2,400–3,500). With 1 tool call consistently, a tighter cost estimate is possible.

### 4. The latency comparison in §4.3 needs clearer cross-reference to the comparison table

Section 4.3 states: *"AHP's cold-cache latency (~3.4s) is 1.2–3× higher than the RAG baseline (~1.1–2.8s)."* This comparison uses the latency profile's 3,436ms p50. But the latency comparison table shows AHP latency ranging from 2,746ms to **5,603ms** for the five evaluation queries. A reader will notice that one evaluation query (MODE1, 5,603ms) is 1.6× the stated "~3.4s."

The paper should note: *"The 3,436ms cold p50 in §4.3 is from the latency profile queries (short nonce queries with uniform structure). Evaluation query latency (§4.2 table) varies by answer complexity — from 2,746ms to 5,603ms — reflecting that more complex queries require longer LLM output."* Without this context, the two figures appear contradictory.

### 5. Nate site "Explain prompt engineering" RAG latency has 25% CV — worth acknowledging

The Nate site RAG latency for "Explain prompt engineering" is 2,258ms / 1,414ms / 1,639ms (mean 1,770ms ±437ms, 25% CV). One run at 2,258ms is 1.6× the other two. The paper reports this without comment. This is a minor issue but one a careful reviewer might flag as evidence of API-side variance in the RAG baseline — consistent with the same API variance that affected the MODE1 AHP query.

This variance (and the similar ±420ms for "AI agents in production") suggests the RAG baseline API latency on the Nate site is subject to meaningful infrastructure variance — the same infrastructure as the AHP calls. A sentence noting *"Both RAG and AHP latency reflect Anthropic API variance; the CV range of 5–31% across queries is consistent with single-provider LLM inference variability"* would contextualise all the stddev figures.

### 6. T21 format: the test sends "What is it?" as its ambiguity probe but the reference server answers it

The notes confirm: *"Server answered without clarification (status=success, mode=MODE2)."* The test premise is that "What is it?" is maximally ambiguous and might trigger clarification. The server answered it — presumably with something about AHP, the only topic it knows. For a server with a well-defined domain, even an ambiguous question has a clear answer: "it" refers to AHP. A genuinely domain-agnostic test for clarification triggering would need a query that is ambiguous *within the site's domain* — e.g., "Which mode should I use?" when MODE1/2/3 are all valid answers. This would more reliably test whether the server implements the clarification pathway.

This is a minor improvement suggestion, not a blocking issue. The test is honestly marked advisory.

---

## What the Tests Need

### Remaining coverage gaps (post-v5.1)

1. **T21 needs genuine §6.3 format verification.** Options: (a) use a domain-internally-ambiguous query ("Which mode should I use?") that might realistically trigger clarification, or (b) add a companion test `T21b` that directly parses a mocked `clarification_needed` response to verify the format without requiring the live server to trigger it. Until one of these exists, §6.3 format compliance is completely unverified.

2. **Content type negotiation (spec §6.6) is untested across all suite versions.** `accept_types`, `response_types`, `accept_fallback`, and `unsupported_type` (400) are spec primitives. No test exercises any of them. A server that omits the entire content type negotiation system passes all 22 tests.

3. **Session time-based expiry (10-minute TTL) is untested.** T18 verifies the 10-turn limit. The time limit is unverified. A server with an infinite session TTL passes all 22 tests.

4. **T16's window consumption is non-deterministic.** The burst test "429 after 10 burst requests" reflects that only 9 requests were available when T16 started (earlier tests consumed 21). The declared limit of 30/min is confirmed by T20 header verification, but the T16 "burst count to 429" varies by how many earlier tests ran. T16 should either (a) note the effective window at test time and compute expected 429 trigger point from that, or (b) be moved to run immediately after T20 (before most test requests) so the full 30-request window is available for more reproducible burst testing.

### Methodology improvements

5. **Add a note or re-run for the MODE1 query's 31% CV.** Either flag the 7,470ms outlier explicitly in the table footnote or add a fourth run to test stability. A consistent ≤20% CV across all queries would give stronger confidence in the latency comparisons.

6. **Update the latency comparison formula in §4.3.** "1.2–3× higher than the RAG baseline (~1.1–2.8s)" uses the latency profile p50, not the per-query evaluation latencies. Add a note clarifying the comparison uses the profile p50 as a summary statistic, while individual query latencies vary.

---

## What the Whitepaper Needs

### Required updates

1. **Add an outlier note for the MODE1 query latency.** One sentence in Table 4.2 footnotes: *"†Run 3 of the MODE1 query recorded 7,470ms (vs. 4,062ms and 5,278ms for runs 1–2), likely a transient API event; this inflates the mean and ±1,727ms stddev for this row."* This prevents a reviewer from treating the high stddev as evidence of general instability.

2. **Update "1–2 tool call rounds" to "1 tool call" in §4.4.** The v5.1 run consistently shows 1 tool call per MODE3 query. The hedged range is stale.

3. **Add a cross-reference clarifying the latency profile p50 vs. evaluation query latency.** One sentence in §4.3 connecting the 3,436ms cold p50 to the fact that evaluation queries vary 2,746–5,603ms based on answer complexity.

4. **Reconsider the T21 ✓ in the conformance table.** Either change the table cell to "Advisory (§6.3 MAY — not triggered in test run)" or note it prominently as "format-unverified." The current ✓ overstates what was tested.

5. **Update the conformance table note for T11** (Section 4.1): the current note says "1 tool call (`check_inventory`) observed in v5.1 run" — accurate. But the §4.4 text still says "1–2 tool call rounds." Make §4.4 consistent with the table note.

### What no longer needs fixing

The following issues from prior critiques are fully resolved in this version:
- ✓ Version mismatch (v4 code / v3 data): resolved — v5.1 code ran before v5.1 test
- ✓ Conformance number inconsistency: all three instances now agree at 21/22 (95.5%)
- ✓ T08 false pass/fail cycle: T08 genuinely passes with server session memory confirmed
- ✓ T22 not running: T22 runs and passes (HTTP 400 on invalid session_id)
- ✓ T20 "unknown/unknown" headers: T20 passes, headers present and numeric
- ✓ RAG baseline absent: fully populated, cache-busted, API-measured
- ✓ Latency profile all-cached: genuine 10-cold + 10-hot split working
- ✓ Appendix stale version: corrected to v5.1
- ✓ Straw-man baseline framing: honestly positioned as upper-bound reference
- ✓ Single-site evaluation: Nate Jones cross-comparison genuine and populated
- ✓ T12 false pass: asserting price regex + failure phrase exclusion works
- ✓ T15/T16 ID swap: fixed
- ✓ p95 with 5 samples: genuine p95 from 20 samples
- ✓ Naive latency 200ms hardcode: replaced with real measured fetch time
- ✓ Character-count token estimation: replaced with tiktoken
- ✓ Cache-busting ±0 stddev: nonces working, genuine variance measured
- ✓ T17 severity: CRITICAL KNOWN DEFECT language appropriate and clear
- ✓ Related work: comprehensive (GPT Actions, Schema.org, ai.txt, RFC 8615, OpenAPI, DID)
- ✓ Conformance scope: reference-implementation-only scoping clear throughout

---

## Summary of Most Critical Issues

**The good news first**: this is the strongest version of the paper to date. The benchmark data is clean, reproducible, and matches the live run exactly. The RAG-baseline comparison is honest, prominent, and correctly positioned as the competitive reference. The latency profile has genuine cold/hot cohort data. The conformance suite ran cleanly with 21/22 passing. T08 and T22 both pass correctly after their respective fixes. The version-mismatch problem that plagued five prior cycles is resolved. The paper's framing — AHP's advantage over RAG is protocol-level, not efficiency-level — is accurate and well-supported by the data.

Remaining issues are incremental, not structural:

- **The MODE1 query AHP latency (5,603ms ±1,727ms, 31% CV) is an outlier driven by a 7,470ms run 3.** This is the highest-variance data point in the benchmark. The paper should acknowledge the likely transient nature of that run. Left unexplained, a reviewer will flag the high stddev as evidence of instability.

- **T21 provides zero verified coverage of spec §6.3's `clarification_needed` format**, but shows ✓ in the conformance table. The MAY nature of the spec means the ✓ is technically correct — but a practitioner would expect T21 to confirm the format works. It does not. The table cell should either say "Advisory" or §6.3 format testing should be added.

- **Three remaining cross-reference clarifications in the paper body**: (1) 1–2 tool calls → 1 tool call consistently in §4.4; (2) latency profile p50 vs. evaluation query latency context in §4.3; (3) the MODE1 outlier footnote in Table 4.2.

- **Two spec features remain completely untested across all 22 tests**: content type negotiation (§6.6) and session time-based expiry (10-min TTL). These are not blocking issues for the current evaluation scope but should be called out explicitly in the test suite's known limitations.
