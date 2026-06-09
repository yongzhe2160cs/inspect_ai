# Proposal: an eval-reliability toolkit for scorer metrics (paired deltas, multiplicity, power)

## Summary

Inspect reports point estimates and (with the recently proposed `ci()`, #4161)
a confidence interval for a single eval's mean. But the questions people
actually act on are *comparisons*:

- "Model A scored 2 points higher than model B — is that real or noise?"
- "Across these 12 tasks, A beats B on 7 — how many of those are just multiple
  comparisons?"
- "How many samples do I need before a 1-point delta is trustworthy?"

These are routinely answered without statistical rigor, and the most common
mistake is treating a comparison on the **same** samples as if the two score
vectors were independent. They are *paired*, and the paired analysis is both
more correct and more powerful.

I'd like to contribute a small, dependency-free toolkit of scorer helpers that
make these comparisons rigorous, in the same style as `stderr()`/`ci()`.

## Proposed API (all in `inspect_ai.scorer`)

- `paired_delta(scores_a, scores_b, level=0.95, alternative="two-sided")` →
  `{delta, stderr, lower, upper, p_value, n}`. CI + significance for a
  per-sample score delta, computed from the per-sample **differences** (paired).
  `align_paired_scores(a, b)` aligns two `SampleScore` lists by `sample_id`.
- `holm_bonferroni(pvalues, alpha=0.05)` and
  `benjamini_hochberg(pvalues, alpha=0.05)` → adjusted p-values + which
  hypotheses survive (family-wise error rate / false discovery rate) for a suite
  of N tasks.
- `min_samples_for_delta(delta, sd_diff, power=0.8, alpha=0.05)` and
  `power_for_samples(n, delta, sd_diff, ...)` → power / minimum-sample guidance.
- `variance_surface(...)` — a dict-valued metric (like `ci()`) exposing a
  score's variance components (sample variance, CLT and clustered standard
  error, clustering design effect): the eval's noise floor.

## Design / constraints

- **No new dependencies**: stdlib `statistics.NormalDist` + the already-present
  numpy, exactly like `ci()`. Intervals/p-values use the large-sample normal
  approximation (CLT), consistent with `ci()`.
- **Reuses existing helpers** (`_clt_stderr`, `_clustered_stderr`) rather than
  duplicating standard-error logic, so numbers are consistent across metrics.
- Pure, importable, unit-testable functions — no model/GPU needed.
- Builds naturally on `ci()` (#4161): same paired-vs-independent insight, same
  CLT machinery. Happy to stack the PR on that branch or rebase onto main once
  it merges.

## Why it belongs in core

Inspect already ships `stderr(cluster=)` and lookahead/recursive-style rigor;
significance-aware *comparison* is the missing piece that every model-vs-model
or suite-level claim needs. Keeping it in the scorer metrics layer (rather than
leaving each user to hand-roll it, usually with the independent-samples error
by mistake) standardizes the correct analysis.

Would the maintainers be interested in this? I have a tested, lint/mypy-clean
branch ready (paired delta validated against the hand computation and against
the naive independent-samples SE; Holm/BH against known p-sets; the textbook
n≈32 power value). Happy to adjust scope, naming, or the metric/helper split to
match your preferences before opening the PR.
