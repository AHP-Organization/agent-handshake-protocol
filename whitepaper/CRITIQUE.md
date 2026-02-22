# AHP Whitepaper Critique

**Critiqued version:** AHP-WHITEPAPER-0.1.md (Draft 0.1, revised 8 — 16:00 UTC)
**Critique date:** 2026-02-22 16:01 UTC (4:01 PM session)
**Live test data:** `/tmp/ahp-report-latest.json` — present, run at 15:52 UTC (v5.3 suite)
**Sequence:** Code committed 15:46 → test ran 15:52 → whitepaper updated 16:00. Correct.
**Prior critiques:** 06:30, 07:30, 08:30, 09:30, 10:30, 11:33, 13:39/14:42 (consolidated)

---

## Overall Assessment: Pre-Submission Quality

After eight revision cycles, the AHP whitepaper has converged to a state where no critical methodological problems remain. The current v5.3 run is the cleanest data across all cycles:

- **22/23 conformance** confirmed in live JSON — matches paper's claim exactly
- **Zero outliers detected** by the new 2× median threshold across all 10 comparison queries
- **T21b passes** with all four mock §6.3 format cases validated correctly
- **All benchmark numbers reproducible and consistent** across runs
- **RAG baseline is genuine API-measured data** with real variance (not ±0 cache artifacts)
- **Latency profile has genuine cold/hot split** with cold p50 = 3,806ms, cached p50 = 9.7ms
- **T17 CRITICAL DEFECT correctly labelled** and the only test failure

The paper's core analytical claims are well-supported:
- AHP uses 3–8× more tokens than a RAG-capable visiting agent on the evaluated corpora (honest and prominently stated)
- AHP provides protocol-level benefits (discovery, caching, MODE3) that are architecturally distinct from token efficiency
- Cached responses deliver sub-30ms latency; cold-cache MODE2 operates at ~3.5–4.5s (infrastructure-dependent)
- Reference deployment passes 22/23 conformance checks, with the single failure (T17) being a known, deliberately disclosed production deployment requirement

What follows is a tight set of remaining issues — none of which are blocking for submission, but all of which are worth addressing.

---

## Critical Problems (must fix)

### 1. `hot_max_ms: 3,551ms` is structurally misleading — it is always a cold sample, not a cache-hit maximum

The latency profile's hot cohort design sends the same "latency-hot-fixed" query 10 times. Run 11 (the first hot sample) is **always** uncached because the server has never seen this query before the profile begins. The live run confirms: `run 11: type=hot, cached=false, latency=3551.58ms`. Runs 12–20 are genuine cache hits (7.7ms–15.3ms).

The JSON field `hot_max_ms: 3,551.6ms` reports run 11's latency as the maximum of the hot cohort. This is incorrect framing: the "maximum hot latency" suggests 3.5 seconds is the worst-case cache hit, when the actual worst-case cache hit is 15.3ms (run 12). The 3.5-second run 11 is structurally a cold call that happens to be labeled "hot."

The `cached_p50_ms: 9.7ms` correctly excludes run 11 (computed from the 9 confirmed-cached samples). But `hot_max_ms` does not exclude it, creating an asymmetry.

**Fix required**: Either:
- Rename `hot_max_ms` to `first_hot_latency_ms` with a note that it is always cold ("hot cohort run 1 — first call on fresh query is always cold")
- Or add a `cached_true_max_ms` field computed only from samples with `cached=true` (currently 15.3ms in this run)
- And update the latency profile table in §4.3 accordingly: the cached_max is 15.3ms, not 3,551ms

The paper's §4.3 already includes the note "Run 1 only — first call on fresh query is always cold" — this is good. The JSON field naming needs to match.

---

## Secondary Issues

### 2. "RAG" Nate site query AHP latency shows 24% CV not flagged by the 2× outlier detector

The "What is RAG and how does it work?" Nate site query shows AHP latency of: 2,333ms / **3,513ms** / 2,370ms (mean 2,738ms ±671ms, CV=24%). Run 2 at 3,513ms is 1.48× the median of 2,370ms — below the 2× threshold and therefore not flagged.

This is the highest AHP latency CV for any Nate site query in this run (all others are 6–14%). The pattern (low–high–low) with run 2 substantially above the others is a mild flag for API variability on that specific request. The outlier detection threshold of 2× correctly avoids false positives from normal LLM variance, but at 3 samples, a 1.48× spike represents a meaningful outlier that goes unreported.

**Note required**: This is not a critical data integrity issue — the mean (2,738ms) remains reasonable and the effect on the table is modest. A footnote would suffice: *"AHP latency for the RAG explanation query shows slightly higher variance (24% CV); run 2 at 3,513ms may reflect API load at query time."* This is consistent with the same footnote approach requested for the rate-limits RAG outlier in the prior critique.

The 2× threshold works correctly for extreme outliers (e.g., the 3.23s vs ~1.1s case in the previous run). For 3-sample benchmarks, the paper might consider noting that a 1.5× threshold would catch moderate outliers, with acknowledgement that this would increase the flag rate on runs with genuine infrastructure variance.

### 3. T21 remains advisory for a well-implemented server — §6.3 live behavior is permanently unverifiable

T21 sends "Which mode should I use?" and receives a direct answer explaining all three modes. T21b validates the `clarification_needed` format in isolation. Together these provide appropriate coverage, but the paper should be clear about the distinction.

The T21 advisory note ("Server is not REQUIRED to clarify — spec says MAY") is correct. However, the broader implication is that §6.3's live trigger condition — when should a server return `clarification_needed` vs. answer helpfully? — has no test coverage. The spec says MAY, which means a well-designed server should almost never trigger it. This is a protocol design characteristic, not a test suite gap, but it means the §6.3 *behavioural* spec (when to use it) is effectively untestable.

The conformance table should clarify: T21 = "§6.3 format ✓ (T21b mock); §6.3 trigger condition ✓ by MAY (server chooses not to trigger — spec-compliant)."

### 4. T14's simulated human response format is a visible template artifact

T14 notes: *"Answer: [Human response] Thank you for your query: 'I need a custom enterprise agreement for 500 sites...'"*

The `[Human response]` prefix is a string template literal, not a realistic human response format. The paper notes this is simulated — that disclosure is good. But if the reference server ships with this response, a developer exploring the reference implementation will see `[Human response]` as a prefix and might copy it into their own implementation. The simulation is honest, but the format could be more realistic to avoid cargo-culting the artifact into production deployments. Consider replacing the prefix with `**Human reviewer:** ` or removing the prefix entirely.

This does not affect test results or paper claims — it is a reference implementation quality note.

### 5. Two spec features remain in known_coverage_gaps — final status before submission

The `known_coverage_gaps` JSON field correctly documents:
- **§6.6 content type negotiation** (`accept_types`, `response_types`, `accept_fallback`, `unsupported_type`)
- **§5.4.3 session time-based expiry** (10-minute TTL; T18 covers turn limit only)

These have appeared in every critique iteration without resolution. They are correctly disclosed, and the paper's §5.5 Limitations section documents them. For the purpose of v0.1 scope, this is appropriate — the paper is evaluating the protocol as implemented, and these are implementation gaps, not spec ambiguities.

**Before submission**: The §5.5 disclosure should explicitly invite the community to implement these tests (T23 and T24 in the suite roadmap) rather than just listing them as gaps. This frames the limitations as future contributions rather than weaknesses.

---

## What the Tests Need

### The only remaining suite addition worth doing before v0.1 submission

1. **Fix `hot_max_ms` JSON field or add `cached_true_max_ms`**: The current `hot_max_ms` reports the uncached run 11 value, making it appear that "hot" queries can take 3.5 seconds. A `cached_true_max_ms` field computed from `s["cached"] == true` samples would correctly report 15.3ms as the true worst-case cache hit.

### Suite roadmap items (post-v0.1, not blocking)

2. **T23: Content type negotiation (§6.6)** — test `accept_types: ["application/data"]` request and verify the server returns `unsupported_type` (400) or a valid alternative. One test, three assertions.

3. **T24: Session time-based expiry (§5.4.3)** — start a session, wait 11 minutes (or mock the TTL), attempt a turn, verify the session is expired. Requires either a configurable TTL for testing or a separate `--short-ttl-mode` flag on the reference server.

4. **Consider 1.5× outlier threshold option** (`--outlier-ratio=1.5`) as a CLI flag for tighter benchmarking contexts where 3-sample benchmarks are used.

---

## What the Whitepaper Needs

### Required before submission

1. **Fix the `hot_max_ms` interpretation in §4.3**: Update the latency profile table footnote to clearly state that `hot_max_ms: 3,551ms` is run 11 (first hot call, always cold by design), and the true worst-case cache-hit latency is 15.3ms. Alternatively report both: "hot_max (incl. first uncached): 3,551ms | hot_max (cache hits only): 15.3ms."

2. **Add a footnote for the Nate "RAG" query AHP latency 24% CV**: One sentence, same pattern as the rate-limits footnote: *"AHP latency for the 'What is RAG' Nate query shows 24% CV (runs: 2,333/3,513/2,370ms); run 2 reflects likely API variance."*

3. **Clarify the T21/T21b split in the conformance table**: The table should make it clear that T21b (not T21) provides §6.3 format verification, and T21 provides spec compliance confirmation (server correctly chose not to clarify per the MAY specification).

### Improvements to strengthen the paper

4. **Add a "Test Suite Roadmap" paragraph in §5.5** alongside the known gaps: "T23 (content type negotiation) and T24 (session TTL) are planned for the v6 suite. Community contributions to implement these tests against third-party AHP deployments are invited via the test suite repository." This reframes the gaps as community opportunities.

5. **Quantify the two-site evaluation scope explicitly in the abstract**: The abstract should state the evaluation corpora: "...evaluated on two corpora: the 40KB AHP Specification Site and the 39KB Nate Jones AI-practitioner site, totalling 10 evaluation queries across 5 topics each." Quantifying the evaluation scope makes the limitation visible without requiring a reader to find §5.5.

6. **T14 reference implementation: replace `[Human response]` prefix** with a more realistic format. This is a reference implementation quality note — the simulated response correctly discloses its nature in the test notes, but the prefix may propagate into real deployments as cargo-culted code.

---

## Resolved Issue Log — Complete

The following is the complete list of issues raised and resolved across all critique cycles. This serves as a record for any future revision.

| Issue | First Raised | Resolved Version |
|-------|-------------|-----------------|
| Naive straw-man as headline claim | v0 06:30 | v1 07:30 |
| Token counting = char/4 (not tiktoken) | v0 06:30 | v1 07:30 |
| Latency baseline hardcoded 200ms | v0 06:30 | v1 07:30 |
| Nate site querying only AHP topics | v1 07:30 | v2 08:30 |
| T12 false-pass (quote vocab, not prices) | v1 07:30 | v2 08:30 |
| T15/T16 test ID swap | v0 06:30 | v1 07:30 |
| RAG baseline absent (no API key) | v2 08:30 | v3 09:30 |
| ±0 stddev (cache contamination) | v2 08:30 | v3 09:30 |
| Three inconsistent conformance numbers | v3 09:30 | v4 10:30 |
| Code committed after test ran (×4) | v3 09:30 | v5 11:33 |
| T08 false-pass (no memory, "FIRST" kw) | v1 07:30 | v4 10:30 |
| T08 false-negative ("FIRST QUESTION" excl) | v4 10:30 | v5 11:33 |
| T20 rate-limit headers "unknown/unknown" | v3 09:30 | v4 10:30 |
| T21 and T22 claimed but not run | v4 10:30 | v5 11:33 |
| Latency profile all-cached (no cold data) | v2 08:30 | v5 11:33 |
| Appendix stale version number | v2 08:30 | v5 11:33 |
| Related work missing | v0 06:30 | v1 07:30 |
| Content signals MUST/SHOULD tension | v0 06:30 | v1 07:30 |
| Single-site evaluation | v1 07:30 | v2 08:30 |
| p95 = max (5 samples) | v0 06:30 | v3 09:30 |
| MODE1 query 31% CV (7.5s outlier) | v5 11:33 | v5.2 (clean run) |
| RAG latency outlier undisclosed (68% CV) | v5.2 14:42 | v5.3 (outlier detection added) |
| §4.2 footnote stddev examples stale | v5.2 14:42 | v5.3 (updated) |
| T21 ✓ overstates §6.3 coverage | v5.1 11:33 | v5.3 (T21b mock added) |
| Naive latency single measurement misleading | v3 09:30 | v4 (footnote added) |
| T17 severity insufficient | v2 08:30 | v4 (CRITICAL DEFECT language) |
| Session memory not implemented (T08 false+) | v2 08:30 | v5 (server fixed, T08 passes) |

---

## Summary of Most Critical Remaining Issues

**The paper is ready for submission.** There are no remaining methodological problems, no data integrity issues, and no false claims in the current version. The suite ran cleanly, the outlier detection added in v5.3 confirmed no anomalies in the current run, and all conformance figures are internally consistent and backed by live data.

- **One structural fix required**: `hot_max_ms: 3,551ms` in the latency profile JSON and table represents the first hot sample (always cold by design — run 11, `cached=false`), not the worst-case cache-hit latency. The true cache-hit maximum is 15.3ms. A field rename or additional field would prevent this from being misread as implying that a "cached" response can take 3.5 seconds.

- **Two minor footnotes needed**: (1) AHP latency for the Nate "RAG" query has 24% CV (runs: 2,333/3,513/2,370ms); run 2 at 3,513ms is a moderate infrastructure spike not flagged by the 2× detector. (2) The clarification in §4.3 that `hot_max_ms` is run 11 is accurate in the text but should also be reflected in the JSON field name for external consumers of the data.

- **Two spec features remain untested** (§6.6 content type negotiation, §5.4.3 session TTL): correctly disclosed in §5.5 and the JSON output. Framing these as T23/T24 community contributions in the paper would strengthen the call-to-action.

- **T17 is the only conformance failure** — a known, clearly disclosed CRITICAL DEFECT in the reference implementation's demo mode. The production warning is prominent and accurate. This is an acceptable state for a reference paper that explicitly documents why authentication must be implemented before any production deployment.
