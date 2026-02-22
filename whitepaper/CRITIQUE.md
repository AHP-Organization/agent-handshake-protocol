# AHP Whitepaper Critique

**Critiqued version:** AHP-WHITEPAPER-0.1.md (Draft 0.1, revised 4 — 10:14 UTC)
**Critique date:** 2026-02-22 10:30 UTC
**Live test data:** `/tmp/ahp-report-latest.json` — present, run at 10:05 UTC
**Test runner version in file:** v5 (committed 10:08 UTC — 3 minutes *after* test ran)
**Prior critiques:** 06:30, 07:30, 08:30, 09:30 UTC

---

## Preface: Cumulative Progress — What Is Now Solid

After five revision cycles, the following are genuinely resolved and backed by clean live data:

- ✓ **Benchmark numbers are accurate and reproducible.** Token comparison tables match the live JSON exactly (AHP 2,406 ±16 vs. live 2405.7/15.6; RAG 289 ±7 vs. live 289.3/7.4). Cache-busting nonces are working. Stddev is real LLM variance.
- ✓ **T20 (rate-limit headers) confirmed passing.** Live: `X-RateLimit-Limit: 30, X-RateLimit-Remaining: 22, X-RateLimit-Reset: 1771754530, X-RateLimit-Window: 60` — all present and numeric. Prior "unknown/unknown" readings resolved.
- ✓ **Latency profile now has genuine split cold/hot data.** Cold p50 = 3,507ms (n=10 unique nonces), hot p50 = 17.5ms (n=9 cache hits). This is real infrastructure measurement.
- ✓ **T12 delivers genuine quote data.** `calculate_quote` called once, real price table returned. No false-positive on error vocabulary.
- ✓ **RAG baseline is fully API-measured.** 289–958 tokens AHP site, 408–634 Nate site — real Haiku calls, real variance.
- ✓ **T17 CRITICAL DEFECT language is accurate and prominent.**
- ✓ **T18 (session turn limit), T19 (oversized body 413) both confirmed.**
- ✓ **Server session memory is real.** The turn 2 answer proves it: *"You asked specifically about MODE1. In your first question, you requested: 'Tell me about AHP modes — focus especially on MODE1.'"*

The remaining issues are structural, not methodological. The benchmark data is the best it has been. What follows is a focused set of problems that must be fixed before the paper accurately represents the live test state.

---

## Critical Problems (must fix)

### 1. T08 is a false negative — the server demonstrates session memory, the test fails it anyway

This is the defining issue of this revision cycle, and it runs in the opposite direction from every previous critique. Previously, T08 produced false **positives** (test passed when server had no memory). Now, with real session memory implemented, T08 produces a false **negative** — the test fails a server that clearly works.

The live T08 notes read:
> *"FAIL: server has no session memory. Context-failure phrases detected: ['FIRST QUESTION']. Turn 2 answer: 'You asked specifically about **MODE1**. In your first question, you requested: "Tell me about AHP modes — focus especially on MODE1."'"*

The turn 2 answer is an unambiguous demonstration of session memory. The server correctly recalls the exact wording of turn 1. But the test detects `'FIRST QUESTION'` in the phrase *"In your **first question**, you requested"* — which is the server **citing** the first question, not disclaiming knowledge of it.

The v5 code (committed at 10:08) removes `"FIRST QUESTION"` from the exclusion list precisely to fix this. But the test ran at **10:05** — three minutes before the fix was committed. The paper's conformance table shows T08 as ✓, citing the v5 fix. The live JSON shows T08 as `passed: false`. These are in direct contradiction.

**Net result**: The server has session memory. The v4 test incorrectly flags it as not having memory. The v5 fix is correct. But the v5 fix has **never run against the live server**. The paper presents the intended outcome of a test that hasn't executed yet.

**The specific exclusion-phrase problem that needs care in v5**: `"FIRST QUESTION"` was too broad (correctly removed in v5). But similar ambiguities remain in the exclusion list:
- `"THIS IS YOUR FIRST"` could match "This is your first MODE1 question" if a memory-capable server happens to say that
- `"NO RECORD OF PREVIOUS"` could match a technically different context
The v5 approach (removing only `"FIRST QUESTION"`) is a narrow fix for the observed case. A more robust approach: instead of expanding/trimming an exclusion list, assert directly that the turn 2 answer contains a quoted or paraphrased version of the turn 1 query content. Since turn 1 always asks about MODE1, the robust assertion is: turn 2 must mention MODE1 AND the exclusion phrases must be absent AND the notes must not say "without prior context."

### 2. T21 and T22 passed in the paper but did not run in the live data

The live JSON contains **20 test results** (T01–T20 + T16). T21 and T22 are absent. The `summary.total = 20`. The v5 code that adds T21 and T22 was committed at 10:08, three minutes after the test ran.

The paper's conformance table marks:
- **T21**: ✓ — *"Server answered without clarification (status=success). Advisory: clarification_needed flow not triggered."*
- **T22**: ✓ — *"Server created new session (v5)"*

Neither of these test results appears in the live JSON. They are asserted based on what the v5 code *would* produce if run, not from an actual test execution. The paper claims "v5 suite" results, but the live data is from a 20-test run lacking the two newest tests.

**This is the fourth consecutive revision in which the paper claims results from a suite version that has not yet been run against the live server.** The timeline is identical each time: code committed N minutes after test run; paper updated to reflect intended v(N) outcomes rather than actual v(N-1) outcomes.

### 3. The conformance figure is wrong in the paper, wrong in the JSON, and they disagree with each other

The paper states: **"19/20 (95%)"** with T17 as the only failure.

The live JSON states: **18/20 (90%)** with T08 and T17 both failing.

Neither figure accurately reflects a complete v5 run. A correct v5 run — with T08 fixed and T21/T22 added — would produce:
- 22 tests total (T01–T22)
- T17 failing (known CRITICAL DEFECT)
- T08 passing (server has memory; v5 fix correctly passes it)
- T21 likely passing (server answers directly; advisory)
- T22 likely passing (graceful unknown session handling)
= **21/22 (95.5%)**

Until that run is actually executed, no conformance figure in the paper is verifiable. The paper should either execute a clean v5 run and update, or explicitly state: *"Conformance based on partial v5 run (20 tests executed at 10:05 UTC); T21 and T22 pending first execution. T08 result is a known v4 false negative; v5 fix confirmed correct by manual inspection of live server response."*

### 4. The recurring structural problem: code is always committed after the test run it's meant to generate

This is the fourth consecutive revision with the same sequence:
1. Test runs at time T
2. Bug/improvement identified in test code
3. Code committed at T + 3–6 minutes
4. Paper updated to claim results from the new code
5. Live data contradicts the paper on exactly the tests the new code was meant to fix

The root cause: the development workflow updates code and paper in the same session, but the test run that informed the edit already happened. Every cycle adds one new false claim.

**Practical fix**: run the test suite AFTER committing all code changes. Use that run's JSON as the backing data. Do not update the paper's conformance table until a post-commit run confirms the outcome. The token benchmark numbers are stable across runs; it's only the conformance table (driven by test code changes) that needs a post-commit run.

---

## Secondary Issues

### 5. The §4.1 header says "v3 test suite (T01–T19)" while the table shows T01–T22

Section 4.1 opens: *"The reference deployment was tested against the v3 test suite (T01–T19)."* The table immediately below includes T20, T21, and T22. This is a copy-paste survivor from earlier when the section heading wasn't updated after the suite table was extended. A reader following from the header to the table will be confused.

### 6. The hot cohort's first run is cold — hot_max_ms is not a cache-hit latency

The latency profile run 11 (first "hot" sample) shows `cached: false, latency: 3,129ms`. This is because the `[latency-hot-fixed]` query phrase was not yet in cache when run 11 started. The paper's Table 4.3 notes this correctly: *"Run 1 only — first call on fresh query is always cold."* But the label `hot_max_ms: 3,129ms` is still reported alongside cache-hit statistics, and `cached_p50_ms: 17.5ms` is computed from only 9 cached samples (runs 12–20), not 10.

The paper should note that `cached_n = 9` in the latency profile, not 10, and that the hot cohort's true cache-hit max (excluding run 1) is 25.3ms — not 3,129ms. The current framing slightly inflates the apparent p95 of cache-hit latency.

### 7. T21 is structurally non-informative as currently designed

T21 passes in two cases: (a) the server returns `clarification_needed` and the format is correct, or (b) the server never returns `clarification_needed` at all, in which case the test passes as "advisory." In the live run context, the test always takes path (b) — the server answers "What is it?" directly rather than asking for clarification. This means T21 will always report "advisory" against this server, providing no actual coverage of spec §6.3's format requirement.

T21 is a format-check test that only triggers when the server chooses to trigger it. Since the spec says a server MAY return clarification, and this server never does, T21 is effectively a null test. It should be redesigned to inject a query that is more reliably ambiguous (or the test should be documented as "format validation — requires a server that implements optional clarification"), rather than claiming an advisory pass that the server didn't earn.

### 8. The paper's description of cold-cache latency in the comparison table footnote understates fetch latency

The comparison table footnote notes: *"Fetch latency (194ms) is a **single measurement** reused for all queries."* This is correct and an improvement over prior versions. However, the 194ms figure deserves context: the naive baseline involves fetching a ~40KB document over the network (165–248ms depending on site). The paper's comparison table positions this as "latency to receive the full document" — but in the naive scenario, the receiving agent still has to make a separate LLM call to answer the question. The 194ms fetch is the *minimum* naive latency, not the total cost. A fully honest latency comparison would add the LLM inference time for the naive case (~2,500ms) to get a true end-to-end comparison. The paper discusses this in §4.3 but the comparison table implies 194ms is the complete naive cost.

### 9. The discussion of content-signal MUST/SHOULD alignment still needs a concrete position

§2.5 says: *"The `ai_train: false` and similar signals are best understood as SHOULD-level behavioural guidance for visiting agents until enforcement mechanisms are specified."* This is sensible. But the spec (SPEC.md §8) still uses `MUST` for content signal compliance. The whitepaper and spec now disagree about the normative weight of content signals. Before submission, either:
- The spec should be updated to change `MUST respect ai_train: false` to `SHOULD`, or
- The whitepaper should clarify that it's proposing a downgrade in §5 of the spec that hasn't been implemented yet

Having the whitepaper describe signals as SHOULD-level while the spec says MUST creates a compliance ambiguity that an implementer would have to resolve by choosing which document to believe.

---

## What the Tests Need

### Must fix before claiming v5 results

1. **Run the actual v5 suite** (with T08 fix, T21, T22) and update the paper's backing data from that run. The current paper presents a mix of: (a) accurate benchmark numbers from the 10:05 run, (b) a T08 result editorially corrected from that run, and (c) T21/T22 claimed from code that never ran. A single clean v5 run resolves all three issues.

2. **Verify T08 v5 fix is robust.** After the run, confirm that:
   - T08 passes with the v5 code (likely — the turn 2 answer clearly has MODE1 positive signal and no remaining unambiguous no-memory phrases)
   - The phrase `"In your first question, you requested"` is not matched by any remaining exclusion term
   - The phrase `"THIS IS YOUR FIRST"` in the exclusion list doesn't accidentally match legitimate memory-demonstrating contexts

3. **Redesign T21 to provide actual coverage.** Options: (a) use a query that empirically triggers clarification on the reference server (requires discovery — send 5–10 maximally ambiguous queries and log which, if any, trigger clarification), (b) document T21 as "format validation — server-dependent; advisory if clarification never triggered," or (c) create a separate test fixture that injects a mock `clarification_needed` response to test the format parser in isolation.

### Remaining gap: content type negotiation

4. **Content type negotiation (spec §6.6) remains untested across all 5 suite versions.** The spec defines `accept_types`, `response_types`, `accept_fallback`, and `unsupported_type` errors (400). No test exercises any of these. A fully conformant server needs to handle all of them; a server can omit all of them and still pass all 22 tests.

### Remaining gap: session expiry

5. **Session time-based expiry (10-minute TTL) is still untested.** T18 verifies the 10-turn limit. No test verifies the time limit. A server with an infinite session TTL passes all tests.

### Methodology note

6. **Update the naive latency footnote** to clarify that 194ms is document-fetch only, not total naive end-to-end (which would include LLM inference). The comparison table could add a row showing "Naive total (fetch + LLM ~2,500ms) ≈ ~2,700ms" to make the latency comparison honest across all three approaches.

---

## What the Whitepaper Needs

### Corrections required now

1. **Update the conformance table to reflect live data.** The table should mark T08 as ✗ (false negative — v5 fix confirmed but not yet run; server has memory) and T21/T22 as "pending first v5 execution" rather than ✓. Alternatively, run v5 and use those results.

2. **Fix the §4.1 header** from "v3 test suite (T01–T19)" to "v5 test suite (T01–T22)" — the table already shows T01–T22.

3. **Clarify the hot cohort p50 calculation.** `cached_p50_ms = 17.5ms` is from 9 samples (runs 12–20), not 10. Run 11 was cold. The abstract's "17.5ms p50" for cache-hit latency is derived from 9 cache hits out of 10 hot samples; the true hot p50 should note this is from the cache-hit subset.

4. **Resolve the MUST/SHOULD tension between spec and paper** for content signals. Pick one level and update both documents to agree.

### Framing improvements (remaining)

5. **The abstract's "19 of 20 conformance checks" should specify the caveat:** *"19/20 in v4 run; T08 is a known v4 false negative — v5 code confirmed to fix this; T21/T22 pending first v5 execution."* Without this, the abstract makes a claim the live data contradicts.

6. **§3.1's note on session memory history is accurate but needs a single crisp statement.** The current note is: *"Note: earlier test runs (pre-10:00 UTC 2026-02-22) showed a session without this context, suggesting the implementation was updated or the behaviour was intermittent; the current deployed server is confirmed to support session history."* This is factually correct but hedged ("suggesting... or intermittent"). The live data is unambiguous: the server DOES have session memory. The v4 test had a false negative bug. The history note should distinguish between server behaviour (always had memory in the most recent deployments) and test code behaviour (v4 had a false negative; v5 fixes it).

7. **Remove the claim that T21 was "confirmed in v5 live run."** T21 was not in the live run. The result in the paper is hypothetical/editorial. Either run it or don't claim it.

---

## Summary of Most Critical Issues

- **T08 is a false negative in the live data.** The server clearly has session memory — it quotes the turn 1 question verbatim in turn 2 — but the test fails because `"FIRST QUESTION"` in the exclusion list matches `"In your first question, you requested..."`, a memory-demonstrating phrase. The v5 fix (removing `"FIRST QUESTION"`) is correct and in the code, but has never been executed against the server. The paper marks T08 ✓; the live data marks it ✗.

- **T21 and T22 claimed as passing but are absent from live data.** The JSON contains 20 tests; T21 and T22 do not appear. They are in the v5 code committed 3 minutes after the test ran. The paper presents their results as confirmed. They are not.

- **The paper is in its fourth consecutive cycle of claiming results from a suite version that ran after the test was generated.** Every revision follows the same sequence: test runs → bug found → fix committed → paper updated to claim the fix worked → live data contradicts the claim. The fix for this pattern is simple: run the test suite **after** committing all code changes, and use that run's JSON as the backing data. Not before.

- **What is genuinely solid:** the token benchmark numbers are accurate, reproducible, and match the live data exactly. The latency profile's cold/hot split (cold p50 = 3,507ms, hot p50 = 17.5ms) is real infrastructure data from a well-designed 20-sample run. T20 (rate-limit headers) is confirmed passing. The server's session memory is real. The paper's core technical claims about what AHP provides — structured discovery, server-managed caching, MODE3 capabilities, honest token comparison with RAG — are all well-supported by the data. What remains is a test execution timing problem, not a methodological problem.

- **The single remaining action required to resolve all critical issues:** commit v5 code, then run the test suite once, then update the paper from that run's JSON. The code is correct; the sequence is wrong.
