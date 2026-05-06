# Week 2, Day 7 Review: `weights/`, `metrics/`, `perf/`

Picking up where I left off in Week 1. Today's block covered the three smaller subsystems
that sit between the engine and the reporting layer — the weighting strategies, the metric
calculations, and the performance/caching helpers. About 2,060 lines across 14 files, which
is a manageable chunk and let me read everything end to end rather than just skimming.

Overall, these three packages are in noticeably better shape than the standalone modules I
reviewed in Week 1. The boundaries are clean, the test coverage looks reasonable, and there
is no obvious dead code. There are some duplication and design issues worth flagging, but
nothing that blocks anything else we are doing.

## Files Reviewed

**`src/trend_analysis/weights/` (5 files, 709 lines)**

- `__init__.py` — re-exports the five weight engines.
- `risk_parity.py` — `RiskParity`, registered as both `risk_parity` and `vol_inverse`.
- `equal_risk_contribution.py` — `EqualRiskContribution` (iterative ERC).
- `hierarchical_risk_parity.py` — `HierarchicalRiskParity` using SciPy clustering.
- `robust_weighting.py` — `RobustMeanVariance` and `RobustRiskParity` plus shrinkage helpers.
- `robust_config.py` — translator that maps the user-facing robustness config dict into the
  constructor kwargs the robust engines expect.

**`src/trend_analysis/metrics/` (5 files, 760 lines)**

- `__init__.py` — the heavy one (439 lines). All the core ratios live here: `annual_return`,
  `volatility`, `sharpe_ratio`, `sortino_ratio`, `max_drawdown`, `information_ratio`, plus a
  metric registry and the legacy alias plumbing.
- `attribution.py` — signal-vs-rebalancing PnL decomposition, with a matplotlib helper.
- `rolling.py` — `rolling_information_ratio`, with optional caching via `perf.rolling_cache`.
- `summary.py` — `summary_table` convenience wrapper that pulls everything together.
- `turnover.py` — `realized_turnover` and `turnover_cost`, no plotting deps.

**`src/trend_analysis/perf/` (3 files, 543 lines)**

- `cache.py` — the `CovCache` LRU plus `compute_cov_payload` and `incremental_cov_update`
  (the latter is the actual workhorse for multi-period rolling windows).
- `rolling_cache.py` — filesystem-backed cache for rolling computations, sandboxed to the
  user's home directory with a tempdir fallback.
- `timing.py` — `log_timing` / `timed_stage` helpers used across the perf module.

## Issues Found

### 1. `RiskParity` and `RobustRiskParity` are doing essentially the same thing

`weights/risk_parity.py` (50 lines) and the `RobustRiskParity` class in `weights/robust_weighting.py`
(lines 310–390, ~80 lines) are both inverse-volatility weighting. The differences are:

- `RobustRiskParity` adds optional diagonal loading when the condition number is high.
- `RobustRiskParity` exposes a `diagnostics` dict.
- `RiskParity` falls back to equal weights when inversions go non-finite; `RobustRiskParity`
  falls back to equal weights when the volatility sum is non-finite.

Otherwise the math is identical: `1/std`, normalize. We have two registry entries
(`risk_parity` / `vol_inverse`) for one of them and a separate `robust_risk_parity` for the
other. We should pick one — my preference is to fold the robustness path into `RiskParity`
behind a flag and delete the duplicate. Right now the only thing keeping them apart is the
`condition_threshold` parameter, which we could just default to `inf` for the non-robust case.

### 2. The legacy compatibility plumbing in `metrics/__init__.py` is loud

Lines 406–432 of `metrics/__init__.py` do three different things to keep old test code working:

- Old short names (`annualize_return`, `info_ratio`, etc.) are aliased.
- A synthetic `tests.legacy_metrics` module is built on the fly and stuffed into `sys.modules`.
- `setattr(_bi, "annualize_return", ...)` literally writes into Python `builtins` so anything,
  anywhere, can reference it without an import.

Polluting `builtins` is the part that bothers me most. It means any other test file in the
process gets these names for free, which is a recipe for hidden coupling. If there are still
callers relying on this, we should hunt them down and import properly; if nothing relies on
it (I suspect the `builtins` patch is older than the `tests.legacy_metrics` shim) we should
just delete the patch. Worth a focused sweep before we touch it.

### 3. Sortino's "single downside observation" branch is suspicious

`metrics/__init__.py` lines 276–281 (and again 296–299 for the DataFrame path):

```python
if len(downside) == 1:
    # When only one downside observation, use 2 * abs(value) as downside volatility
    # This matches the golden test file expectations
    down_vol = 2.0 * abs(downside.iloc[0])
```

This is a special case to make a golden test pass. There is no statistical reason
`2 * |x|` is the right answer for a single observation — it is just whatever number happened
to land in the snapshot. This is the kind of thing that quietly bites us when someone
re-runs the goldens. We should either revisit the golden, or change this to return `NaN`
when there is only one downside observation (which is what `std(ddof=1)` would do anyway).

### 4. Duplicate `_timed_stage` between `signals.py` and `perf/timing.py`

`src/trend_analysis/signals.py` defines its own `_timed_stage` context manager at line 97.
`perf/timing.py` provides a public `timed_stage` that does the same thing, with more
features (status updates, extra metadata). The `signals.py` version even uses the same
`perf_counter` call. Easy cleanup — delete the local copy and import from `perf.timing`.

### 5. Ledoit-Wolf and OAS shrinkage are using sample-size heuristics, not real sample sizes

`weights/robust_weighting.py` lines 50–67 and 92–106:

```python
if n_samples is None:
    n_samples = max(p + 1, int(p * 2))  # Conservative estimate
```

Both shrinkage functions accept an optional `n_samples` but the call sites in
`RobustMeanVariance._apply_shrinkage` never pass it, so we always fall through to this
heuristic. The whole point of Ledoit-Wolf is that the shrinkage intensity depends on how
much data you actually have. With a synthetic `n = max(p+1, 2p)` we are not really doing
Ledoit-Wolf — we are doing a hand-tuned blend of sample covariance and identity-scaled
target. We should plumb the actual sample size through from the call site, or be honest
in the docstring that this is an approximation.

### 6. `CovCache.incremental_cov_update` reconstructs `S2` lossy

`perf/cache.py` `_ensure_aggregates` (lines 193–210) reconstructs `S2` from `cov` when it's
missing using:

```python
s2 = cov * (n - 1) + n * outer_mm
```

That identity is correct, but it depends on `cov` itself being exact. After a chain of
incremental updates, floating-point error in `cov` propagates back into `S2` and amplifies
on the next update. We should either always materialise `S1`/`S2` upfront when we know we
will roll the window (the `materialise_aggregates=True` path in `compute_cov_payload`), or
add a periodic full-recompute every N steps to cap error growth. The `multi_period/engine.py`
caller does ask for materialised aggregates on the first window, so this is mostly a
defensive concern, but worth a comment in the code.

### 7. `summary_table` and the attribution helpers are not used by the main pipeline

`metrics/summary.py` and `metrics/attribution.py` both have unit tests but nothing in
`pipeline.py` or the engine actually calls them. They look like utilities someone built
for a notebook or one-off analysis. Not dead code — the tests exercise them — but not
load-bearing either. Worth deciding whether they're public API (in which case document
them) or internal helpers (in which case maybe move them under `util/`).

### 8. Minor: `metrics/__init__.py` imports `matplotlib.pyplot` transitively

`attribution.py` does a top-level `import matplotlib.pyplot as plt`, which means any code
that imports `trend_analysis.metrics` ends up pulling matplotlib through the
`import_module(".attribution", __name__)` line at the bottom of `__init__.py`. That's a
chunky transitive dep for a metrics module. Either lazy-import inside `plot_contributions`
or move the plot helper to `viz/`.

## Notes from the Review

- The weighting subsystem cleanly uses the `WeightEngine` plugin registry from
  `plugins/__init__.py`. That's the right pattern, and as far as I can tell every engine
  in the registry is actually referenced from `multi_period/engine.py` or
  `monte_carlo/strategy/variant.py`. No orphan engines.
- HRP is the most defensively coded of the bunch — every numerical step has a fallback,
  the outer `try/except Exception` at line 119 catches anything else and degrades to equal
  weights. This is good for production but it does mean genuine bugs would be silently
  masked. Worth at least logging at `ERROR` (which it does) rather than `WARNING` and
  considering whether we want a strict mode for testing.
- ERC's convergence check (`max_deviation < self.tol` at line 75) compares absolute risk
  contribution deviations against `tol = 1e-8`. For typical portfolio variances of ~1e-4,
  this is effectively requiring 4 decimal places of relative precision. Probably fine in
  practice but worth confirming with whoever picked the tolerance — the comment doesn't say.
- The `perf` package is the cleanest of the three. Pure functions, well-typed, no surprise
  dependencies, deterministic SHA-256 keys. The only thing I'd change is that
  `_DEFAULT_ROLLING_CACHE` is created at import time, which means every test that imports
  the module touches the filesystem (creates `~/.cache/trend_model/rolling/`). Lazy-init
  via a getter would be tidier.
- `rolling_cache.py` has nice path-traversal protection on lines 41–46 — `is_relative_to(home)`
  prevents `TREND_ROLLING_CACHE` from pointing outside the user's home dir. Glad to see that.
- The `robust_config` module is doing more aliasing than I'd like — there are at least
  three different keys it accepts for "condition threshold" (`threshold`,
  `condition_threshold` nested, `condition_threshold` flat). That kind of forgiveness is
  helpful while a config schema is in flux but it should get pinned down once we settle
  on the canonical shape.

### Suggested cleanup order

If we were to act on these, my priority list would be:

1. Pick one risk-parity implementation and delete the other (Issue 1). Easy win, real LOC reduction.
2. Audit and ideally remove the `builtins` patch in `metrics/__init__.py` (Issue 2).
3. Consolidate `_timed_stage` (Issue 4). Trivial.
4. Fix or document the Sortino single-downside branch (Issue 3).
5. The shrinkage sample-size question (Issue 5) is more of a numerical-correctness review
   and probably wants Phil's eyes before we change anything.
6. Everything else is cosmetic.

That's roughly half a day of cleanup if we batch them into a single PR — happy to do that
once the rest of Week 2 is reviewed and we know what other refactors are coming.

— Day 7 done. Tomorrow: `portfolio/`, `rebalancing/`, `multi_period/` (4,959 lines, the
multi-period engine alone is 4,352 of those, so I'll be budgeting most of the day for it).
