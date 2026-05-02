# Week 2, Day 7 Review — `weights/`, `metrics/`, `perf/`

Picking up where Week 1 left off. Day 7 covers the three supporting packages
that the analysis engine leans on for portfolio construction, scoring, and
caching: `src/trend_analysis/weights/`, `src/trend_analysis/metrics/`, and
`src/trend_analysis/perf/`. Roughly 2,060 lines across 14 files. Smaller than
yesterday's CLI work, but denser — most of this is numerical code with a fair
amount of defensive logic, so I read it carefully rather than skimming.

Overall impression: the code is in reasonable shape. Each package has a clear
job, the public surface is small, and the consumers (api, pipeline, multi_period,
core/rank_selection) reach in via narrow imports. There are a handful of things
worth cleaning up, and one part of `metrics/__init__.py` that I think we should
quietly fix sooner rather than later. Details below.

## Files Reviewed

**`src/trend_analysis/weights/` (5 files, 759 lines)**

- `__init__.py` — re-exports the five engine classes.
- `risk_parity.py` — inverse-volatility weighting, registered as both
  `risk_parity` and `vol_inverse`.
- `equal_risk_contribution.py` — iterative ERC with regularisation guards.
- `hierarchical_risk_parity.py` — HRP via single-linkage clustering.
- `robust_weighting.py` — `RobustMeanVariance` (with Ledoit-Wolf / OAS
  shrinkage and a configurable safe-mode fallback) plus `RobustRiskParity`.
- `robust_config.py` — translator that turns a free-form robustness config
  block into constructor kwargs for the robust engines.

**`src/trend_analysis/metrics/` (5 files, 760 lines)**

- `__init__.py` — the metric registry plus the core scalar/vector
  implementations (`annual_return`, `volatility`, `sharpe_ratio`,
  `sortino_ratio`, `max_drawdown`, `information_ratio`).
- `attribution.py` — signal-and-rebalancing PnL decomposition with a
  matplotlib helper.
- `rolling.py` — `rolling_information_ratio` wired through the rolling cache.
- `summary.py` — convenience `summary_table` builder.
- `turnover.py` — `realized_turnover` and `turnover_cost`.

**`src/trend_analysis/perf/` (3 files, 543 lines, no `__init__.py`)**

- `cache.py` — `CovCache` (in-memory LRU) plus `CovPayload`,
  `compute_cov_payload`, and `incremental_cov_update` for sliding-window
  recomputation.
- `rolling_cache.py` — filesystem-backed cache for rolling computations,
  defaulting under `~/.cache/trend_model/rolling`.
- `timing.py` — `log_timing` and a `timed_stage` context manager.

## Issues Found

These are sorted roughly by how much I think they matter. None of them are
production-blocking, but the first two I'd actually like to schedule as
follow-up tickets.

**1. `metrics/__init__.py` is doing things in `__init__` that don't belong there.**
   The bottom of the file (lines 406–439) does three surprising things:
   it builds a fake `tests.legacy_metrics` module and pokes it into
   `sys.modules`; it monkey-patches Python `builtins` so that
   `annualize_return` and `annualize_volatility` are globally available; and
   it imports four submodules purely for side-effect-style attribute exposure.
   The `tests.legacy_metrics` shim in particular means importing
   `trend_analysis.metrics` mutates global interpreter state — that's the kind
   of thing that bites you when something is run in isolation. I'd like to
   move the legacy compatibility shim into the test layer where it belongs
   and drop the `builtins` patch entirely.

**2. The `sortino_ratio` single-downside special case is a test-fitting kludge.**
   Lines 275–281 (and again 295–301 for the DataFrame path) hard-code
   `down_vol = 2.0 * abs(downside.iloc[0])` when there is only one negative
   observation, with a comment explicitly saying it exists to match a golden
   test file. That's the wrong direction of causality — the implementation
   is bending to the fixture. We should either regenerate the golden file
   with the standard `ddof=1` definition (and accept NaN for n=1), or write
   down somewhere why the 2× rule is the right answer for this dataset.

**3. Duplicated inverse-volatility logic between `RiskParity` and `RobustRiskParity`.**
   `risk_parity.py` and the `RobustRiskParity` class in `robust_weighting.py`
   compute the same thing — invert the diagonal, normalise, fall back to
   equal weights on degeneracy. The robust variant adds diagonal loading
   when the condition number is bad, but the core math is the same.
   `RobustRiskParity` could be a thin wrapper that pre-conditions the matrix
   and then delegates to `RiskParity.weight()`. It would shave ~40 lines
   and remove a place where the two implementations could drift.

**4. `_timed_stage` is reimplemented in `signals.py`.**
   `perf/timing.py` already exposes a `timed_stage` context manager.
   `src/trend_analysis/signals.py:97` defines a near-identical helper that
   logs to a different logger at DEBUG instead of INFO. Not a bug, but it's
   the kind of small fork that means future timing improvements have to be
   made in two places. Worth a one-line cleanup when someone is in
   `signals.py` next.

**5. `perf/` has no `__init__.py`.**
   It works as a namespace package (every consumer imports submodules
   directly, e.g. `from ..perf.cache import CovCache`), so nothing breaks.
   But it's inconsistent with the rest of `trend_analysis/`, where every
   subpackage has one. A trivial empty `__init__.py` would make package
   introspection (and tooling like `pkgutil.walk_packages`) behave the same
   as elsewhere.

**6. `weights/__init__.py` doesn't re-export `weight_engine_params_from_robustness`.**
   Three consumers (`api.py`, `pipeline.py`, `multi_period/engine.py`)
   import it directly from `weights.robust_config`. That's fine, but adding
   it to the package `__init__` would make the public surface match how
   it's actually used.

**7. Minor: `RobustMeanVariance._check_condition_number` is a one-line wrapper.**
   Lines 171–173 just delegate to the module-level `_safe_condition_number`.
   The wrapper adds nothing — call the helper directly.

**8. Minor: empty-input edge case in `summary_table`.**
   `summary.py:47` reads `turn_df["turnover"].mean()` without guarding the
   empty case. If a caller hands in an empty weights mapping, this returns
   NaN rather than raising — which is probably fine, but I'd want to
   document the contract or add an explicit check.

**9. Minor: `compute_cov_payload` silently zero-fills NaNs.**
   `perf/cache.py:171` does `arr.ffill().fillna(0.0)`. Conservative, as the
   docstring notes, but it means a column that's entirely missing becomes
   a column of zeros — and that contaminates the covariance estimate without
   any warning. At minimum this should log a debug line listing the columns
   that were forward-filled into nothing.

**10. Minor: dynamic submodule imports inside functions in `RobustMeanVariance._safe_mode_weights`.**
   Lines 198–207 do `from .hierarchical_risk_parity import HierarchicalRiskParity`
   inside the method. Presumably to avoid an import cycle, but the cycle
   isn't obvious from a quick read — both classes live in the same package
   and only depend on the plugin registry. We could lift these to the top
   of the file unless a circular import shows up.

## Notes from the review

A few things that don't rise to the level of "issue" but I want on the record
while it's fresh.

- The two cache layers (`CovCache` in memory, `RollingCache` on disk) have
  almost no overlap in responsibility, which is good. `CovCache` is keyed
  by `(start, end, asset_universe_hash, freq)` and stores aggregates so
  rolling windows can update incrementally; `RollingCache` is keyed by
  `(dataset_hash, window, freq, method)` and stores fully materialised
  pandas Series on disk under `~/.cache/trend_model/rolling`. The split
  makes sense: the in-memory one is for per-run reuse during multi-period
  walk-forward, the disk one is for skipping repeated work across runs.

- I like that `RollingCache._ensure_cache_dir` falls back to `tempfile`
  if the home cache dir isn't writable, and disables itself if even that
  fails. That's the right posture for a non-essential cache.

- The robust weighting engines log a lot when things go sideways
  (condition numbers, shrinkage intensities, method switches) and stash
  the same information on `self.diagnostics`. Good for debugging,
  particularly when someone asks "why did the optimiser fall back to HRP
  on this period?". The three booleans (`log_condition_numbers`,
  `log_method_switches`, `log_shrinkage_intensity`) feel slightly
  over-parameterised — in practice you either want all of it or none of
  it — but it's easy to leave alone.

- The `weight_engine_params_from_robustness` function in `robust_config.py`
  is doing a fair amount of compatibility work: it accepts both
  `condition_check: bool` and `condition_check: {enabled: bool, ...}`,
  hoists `safe_mode` and `condition_threshold` from either the inner or
  outer block, and so on. That's effectively a config migration layer.
  Worth a comment at the top of the function naming it as such, so future
  readers don't think the duplication is accidental.

- `_METRIC_REGISTRY` in `metrics/__init__.py` is exposed under two names —
  the private `_METRIC_REGISTRY` and the public alias `METRIC_REGISTRY`.
  Pick one. Same dict, two names, no policy difference between them.

- Per the Week 1 plan, I greped each public name to look for duplicates
  outside the package. Nothing alarming: `RiskParity`, `HRP`, etc. only
  appear inside `weights/` and the plugin registry; `CovCache` /
  `compute_cov_payload` are only consumed by `core/rank_selection.py`
  and `multi_period/engine.py`; metrics are imported by `walk_forward`,
  `engine/walkforward`, and `viz/charts`. The package boundaries are
  holding.

- One thing I want to flag for the Week 2 Day 8 review: the
  `multi_period/engine.py` consumer of `CovCache` does its imports inline
  inside the function body (lines 1058, 1130, 1214) and even aliases the
  same function to `_ccp` on a third import. Worth a closer look when I
  get to that file — feels like the inline imports are papering over an
  ordering problem.

## Suggested follow-ups

If I were prioritising tickets out of this:

1. Move the `tests.legacy_metrics` shim and the `builtins` monkey-patch
   out of `metrics/__init__.py`. Targeted cleanup, low risk, immediate
   benefit to anyone reading the module fresh.
2. Fix the Sortino single-downside golden-file kludge — either justify
   it in code or regenerate the fixture.
3. Collapse `RobustRiskParity` onto `RiskParity` (preserve diagnostics).
4. Add `perf/__init__.py` and re-export `weight_engine_params_from_robustness`
   from `weights/__init__.py`.
5. Consolidate the `signals.py` `_timed_stage` onto `perf/timing.timed_stage`.

Items 4 and 5 are five-minute changes I can roll in alongside the Week 4
cleanup PRs. Items 1–3 deserve their own PRs with proper test runs.

---

Total: ~2,060 lines reviewed. Day 7 done. Moving on to portfolio /
rebalancing / multi_period for Day 8 — fair warning, that one will run
long because `multi_period/engine.py` alone is ~4,350 lines.
