# Add eval-reliability toolkit (paired delta, multiplicity, power, variance)

> Builds on #4161 (`ci()`). Recommended to merge after / stack on that PR; the
> diff is cleanest viewed against the `feature/ci-metric` branch.

## Problem

Inspect reports point estimates and (with `ci()`, #4161) a confidence interval
for a single eval's mean — but the decisions people make are *comparisons*, and
those are usually made without statistical rigor:

- model A vs model B on the same eval — "is a 2-point gap real?"
- a model across a suite of N tasks — "how many of these wins are just multiple
  comparisons?"
- "how many samples before a delta this size is trustworthy?"

The most common error is the first one: a comparison on the **same** samples is
*paired*, but it's almost always analyzed as if the two score vectors were
independent, which throws away information and inflates the error bar.

## Approach

Four small, dependency-free helpers in the scorer metrics layer, in the same
style as `stderr()`/`ci()` (stdlib `statistics.NormalDist` + the already-present
numpy; large-sample normal approximation via the CLT, consistent with `ci()`):

1. **`paired_delta(scores_a, scores_b)`** → `{delta, stderr, lower, upper,
   p_value, n}`, plus `align_paired_scores()` to align two `SampleScore` lists
   by `sample_id`.
2. **`holm_bonferroni(pvalues)` / `benjamini_hochberg(pvalues)`** → adjusted
   p-values + which survive (family-wise error rate / false discovery rate).
3. **`min_samples_for_delta(delta, sd_diff)` / `power_for_samples(n, ...)`** →
   power / minimum-sample guidance.
4. **`variance_surface(...)`** → a dict-valued metric (like `ci()`) exposing a
   score's variance components: sample variance, CLT and clustered standard
   error, and the clustering **design effect** — the eval's noise floor.

It reuses the existing `_clt_stderr` / `_clustered_stderr` helpers rather than
duplicating standard-error logic, so the numbers line up across `stderr()`,
`ci()`, and `variance_surface()`.

## The correctness point (paired, not independent)

When A and B are run on the **same** samples, the right standard error of the
mean delta is the SD of the per-sample *differences* `d_i = a_i − b_i`, divided
by √n. That absorbs the per-sample difficulty common to both models. Treating
the vectors as independent (`√(SE_A² + SE_B²)`) discards that cancellation and
reports a much wider interval for positively-correlated scores — the usual case.
`paired_delta` does the differencing; a test asserts the paired SE is far below
the independent-samples SE on correlated inputs (and collapses to ~0 when the
difference is constant).

## Tests

`tests/scorer/test_metric.py` (alongside the `ci()` tests), all CPU-only:

- **paired delta** vs a hand computation (`a−b=[1,1,1,0]` → δ=0.75, SE=0.25,
  95% CI [0.260, 1.240], two-sided p=2·Φ(−3)≈0.0027), one-sided alternatives,
  the paired-vs-independent SE contrast, alignment by `sample_id`, and the
  small-sample collapse.
- **Holm / BH** on known p-sets: on p=[.01,.02,.03,.04,.05] Holm rejects 1 with
  adjusted [.05,.08,.09,.09,.09], BH rejects all 5 (q=.05); a step-up gap case;
  BH ≥ Holm discoveries; input-order preservation; validation/empty.
- **power**: the textbook d/σ=0.5 → n=32 at 80%/5%, the 0.25 → 126 scaling, and
  `power_for_samples` inverting `min_samples_for_delta`.
- **variance surface**: components vs `var()`/`stderr()`, and design effect > 1
  under within-cluster correlation.

Run locally (Python 3.12 via uv): `pytest tests/scorer/test_metric.py` →
**56 passed** (40 existing + 16 new). `ruff check`, `ruff format --check`, and
`mypy src/inspect_ai/scorer/_metrics/reliability.py` are clean.

I did **not** run the full GPU/model test suite (no model access in this
environment); the change is pure-Python scorer math with no model/runtime
dependency.

## Notes

- Like `ci()`, this uses the normal approximation; the *paired* structure (not
  `t` vs `z`) is the statistical point. A Student-`t` variant could be added
  later if desired.
- AI assistance (Claude) was used; I've reviewed the statistics (each value
  checked by hand) and the code.
