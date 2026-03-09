# Evaluation Scoring Rubric

Source: https://github.com/rshu/Agent_Eval/blob/main/agent_eval/evaluate/prompt_template.py

## Criteria (Score each 0-5)

For each criterion: 0=unacceptable, 1=very poor, 2=poor, 3=acceptable, 4=good, 5=excellent.
Use integer scores only.

### A. Functional Correctness (0-5)

Evaluate whether the patch correctly fixes the bug or implements the requested feature at the semantic level.
Focus on: Does the logic/approach correctly address the root cause of the issue?

- 5 (Excellent): Clearly addresses the root cause; changes align perfectly with expected semantics; no logical flaws or edge case failures apparent.
- 4 (Good): Addresses the root cause correctly; minor edge cases may be unhandled, but core functionality is sound and reliable.
- 3 (Acceptable): Plausibly fixes the issue; approach is reasonable but may have minor logical gaps or miss some edge cases; still functionally correct for the main use case.
- 2 (Poor): Touches the right area but fix is incomplete or has significant logical issues; may work for some cases but fails for others; approach is partially correct but unreliable.
- 1 (Very Poor): Superficial or speculative fix; likely misses the actual root cause; changes may be related but don't meaningfully address the issue.
- 0 (Unacceptable): Does not address the issue at all, or changes are clearly wrong/contradict the requirement, or introduces new bugs.

### B. Completeness & Coverage (0-5)

Evaluate whether the patch handles all required updates across the codebase, including related files, tests, documentation, and edge cases.

- 5 (Excellent): Covers all essential code-paths, related files, and includes/updates appropriate tests; matches Ground Truth scope or provides an equally complete alternative.
- 4 (Good): Covers all essential changes but may miss one minor related update; still comprehensive and usable.
- 3 (Acceptable): Fix exists but misses some secondary updates; still materially improves behavior and covers the primary changes.
- 2 (Poor): Partial fix with notable gaps; incomplete but not broken.
- 1 (Very Poor): Major gaps in coverage; patch is significantly incomplete.
- 0 (Unacceptable): Incomplete to the point of being non-functional.

Note on tests: Tests are required when (1) the issue statement explicitly requests them, (2) the Ground Truth includes tests, or (3) the fix is non-trivial.

### C. Behavioral Equivalence to Ground Truth (0-5)

Evaluate how semantically similar the Generated Patch's behavior is to Ground Truth, focusing on functional equivalence rather than textual similarity.

- 5 (Excellent): Semantically equivalent to Ground Truth; differences are purely cosmetic.
- 4 (Good): Nearly equivalent with only minor semantic differences.
- 3 (Acceptable): Achieves the same high-level goal but differs in approach or implementation details.
- 2 (Poor): Overlaps significantly but has notable semantic differences.
- 1 (Very Poor): Overlaps partially but misses key semantic changes.
- 0 (Unacceptable): Diverges from Ground Truth in a way that likely fails the intended behavior.

## Scoring Notes

Each criterion is scored independently (0-5). No overall/total score is computed — report each dimension separately with justification. Downstream consumers can apply their own weighting if needed.

## Verdict Rules (apply in order, first match wins)

1. **FAIL** if any of:
   - A <= 1
   - overall_score <= 30
   - Patch introduces breaking changes
   - Patch clearly contradicts issue intent

2. **PASS** if all of:
   - A >= 4 AND B >= 4 AND C >= 3
   - overall_score >= 70
   - No major semantic conflicts with Ground Truth

3. **PARTIAL** if:
   - A >= 2
   - overall_score between 31-69
   - Either B < 4 OR C < 3

4. Default to PARTIAL for edge cases.
