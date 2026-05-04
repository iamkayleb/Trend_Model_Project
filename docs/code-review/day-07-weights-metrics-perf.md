# Day 7 Review — `weights/`, `metrics/`, `perf/`

**Date:** 2026-05-04
**Branch:** `claude/trusting-newton-LNGXU`
**Scope:** 14 files, 2,062 lines (matches the Week 2 plan exactly)

Quick note up front: this is the first day of Week 2, so I'm easing back into the
review rhythm after the Week 1 wrap-up. These three packages are tightly related
— the weight engines consume covariance matrices, the metrics module computes
the numbers we report on those weights, and `perf/` is the caching layer that
keeps the rolling work from blowing up our runtime. Reading them as a group made
more sense than treating them as three independent days.

---

## Files Reviewed

### `src/trend_analysis/weights/` (5 files, ~759 lines)

| File | Lines | What it does |
|------|-------|--------------|
| `__init__.py` | 14 | Re-exports the five engines. |
| `risk_parity.py` | 50 | Plain inverse-volatility weights. |
| `equal_risk_contribution.py` | 102 | ERC via iterative scaling, with regularisation guards. |
| `hierarchical_risk_parity.py` | 122 | Lopez de Prado HRP with single-linkage clustering. |
| `robust_weighting.py` | 390 | `RobustMeanVariance` + `RobustRiskParity`, plus shrinkage helpers. |
| `robust_config.py` | 81 | Translates the `robustness:` config block into engine kwargs. |

### `src/trend_analysis/metrics/` (5 files, ~760 lines)

| File | Lines | What it does |
|------|-------|--------------|
| `__init__.py` | 439 | Core return/vol/Sharpe/Sortino/MDD/IR + a registry decorator. |
| `attribution.py` | 134 | Signal/rebalancing PnL decomposition + matplotlib chart. |
| `rolling.py` | 67 | Rolling IR (the only rolling metric here) wired to the disk cache. |
| `summary.py` | 64 | The CAGR/vol/MDD/IR/Sharpe/turnover summary table. |
| `turnover.py` | 56 | Turnover and linear cost helpers. |

### `src/trend_analysis/perf/` (3 files, ~543 lines)

| File | Lines | What it does |
|------|-------|--------------|
| `cache.py` | 267 | In-memory `CovCache` + `compute_cov_payload` + incremental update. |
| `rolling_cache.py` | 195 | Disk-backed `RollingCache` keyed by SHA-256 of the input frames. |
| `timing.py` | 81 | `log_timing` + `timed_stage` context manager. |

---

## Issues Found

I'll group these by severity rather than by file, because it makes the action
list a lot cleaner.

### High — these I'd want us to fix before the next release

**1. `metrics/__init__.py` is monkey-patching builtins and `sys.modules`.**
Lines 412–432 do two things I had to read twice to believe:

- It synthesises a fake `tests.legacy_metrics` module and registers it in
  `sys.modules` *at import time of the production package*. Production code
  should not be writing to `sys.modules['tests.*']`. It also shadows the real
  `tests/legacy_metrics.py` file that already exists on disk (we have one — I
  grepped for it).
- It calls `setattr(_bi, "annualize_return", annualize_return)` and the same
  for `annualize_volatility`. That puts those names on the `builtins` module,
  meaning they're globally available without any import in any process that
  has ever imported `trend_analysis.metrics`. That's the kind of thing that
  bites you a year later when somebody innocently reuses one of those names
  as a local variable.

We should rip both blocks out and migrate the few tests that depend on the
side effect. The legacy aliases (`annualize_return = annual_return`, etc.)
on lines 406–410 can stay until the test suite catches up; it's only the
`builtins` and `sys.modules` writes that are dangerous.

**2. The "Ledoit-Wolf" and "OAS" shrinkage in `robust_weighting.py` are not
actually Ledoit-Wolf or OAS.** Lines 30–111 implement a *matrix-only*
approximation that estimates `n_samples` with the heuristic `max(p+1, 2*p)`.
Real Ledoit-Wolf needs the raw return panel, not just the sample covariance
— that's the whole point of the estimator. The function name and the
`shrinkage_method: Literal["ledoit_wolf", ...]` config promise something
we're not delivering, and the docstrings don't flag it.

There are two ways out: (a) push the raw returns through to these helpers and
implement the real estimators (sklearn already has both), or (b) rename them
to something honest like `diagonal_target_shrinkage` and add a paragraph in
the docstring explaining the heuristic. Either way, the current naming is
misleading to anyone reading the config.

**3. `RiskParity` and `RobustRiskParity` are doing the same math twice.**
Compare `weights/risk_parity.py:19-50` with `weights/robust_weighting.py:330-390`:
both extract diagonal variances, guard against non-positive values, take
inverse vol, and renormalise. The "robust" version adds diagonal loading and
condition-number tracking, but the core computation is duplicated.
`RobustRiskParity` should call `RiskParity.weight()` after pre-conditioning
the matrix, or both should share a private helper.

### Medium — worth fixing but not urgent

**4. The empty/square-shape guard at the top of every `weight()` method.**
The same five lines appear in `RiskParity`, `HRP`, `ERC`, `RobustMeanVariance`,
and `RobustRiskParity`:

```python
if cov.empty:
    return pd.Series(dtype=float)
if not cov.index.equals(cov.columns):
    raise ValueError("Covariance matrix must be square with matching labels")
```

Same for the "fall back to equal weights" pattern (`return pd.Series(np.ones(n)/n, index=cov.index)`)
which appears in four of the five files. These belong on the `WeightEngine`
base class as a `_validate_cov(cov)` method and a `_equal_weights(cov)` helper.
Not a bug, just sandpaper.

**5. `metrics/summary.py` recomputes turnover twice.** Line 38 calls
`realized_turnover(weights)`, then line 39 calls `turnover_cost(weights, ...)`
which *also* calls `realized_turnover(weights)` internally. For long weight
histories this doubles a `.diff().abs().sum()` over a wide DataFrame. Either
have `turnover_cost` accept a precomputed turnover series, or compute the
diff once locally and pass it in.

**6. The Sortino "single downside observation" hack.** `metrics/__init__.py:275-279`
and the equivalent block at 296-300 do this:

```python
if len(downside) == 1:
    # When only one downside observation, use 2 * abs(value) as downside volatility
    # This matches the golden test file expectations
    down_vol = 2.0 * abs(downside.iloc[0])
```

That comment is a smell. Production behaviour shouldn't be bent to match a
golden file — usually the right fix is to update the golden file. I want to
look at the golden test before recommending we change it, but flagging it
now so we don't forget.

**7. `_DEFAULT_ROLLING_CACHE = RollingCache()` runs on import.**
`perf/rolling_cache.py:175` instantiates the cache at module import time,
which calls `_ensure_cache_dir`, which calls `mkdir(parents=True, exist_ok=True)`.
That means *importing* `trend_analysis.metrics.rolling` (which imports this
module) creates a directory under `~/.cache/trend_model/rolling/` even when
the caller never intends to use the cache. It's harmless but it's the kind
of side effect that surprises people running our code in sandboxes or
read-only environments. Lazy initialise it inside `get_cache()` instead.

**8. `attribution.py` imports `matplotlib.pyplot` at module top.**
Line 23. That makes matplotlib a hard import dependency for anyone who calls
`compute_contributions` (which is the only function most consumers want). The
plotting function should do `import matplotlib.pyplot as plt` lazily inside
`plot_contributions`. We've done this elsewhere — `attribution.py` is the
holdout.

**9. `RobustMeanVariance._safe_mode_weights` does local imports of HRP and
RP.** `weights/robust_weighting.py:198-207`. The comment in the codebase
suggests this is to avoid a circular import, but I checked and there isn't
one — both modules only depend on `..plugins`. We can hoist these imports
to the top of the file.

### Low — notes, not action items

**10.** `weights/__init__.py` exports five names but the registry decorator
already registers them under string keys (`"erc"`, `"hrp"`, `"risk_parity"`,
etc.). Most of the codebase looks them up by string through the registry, so
the explicit class re-exports are mostly used by the test suite.

**11.** `robust_config.py` has a fairly ornate fallback chain (lines 31–50)
for back-compat with old config shapes. If we're past the deprecation window
on the old keys, we can simplify this. Worth checking the changelog.

**12.** `cache.py:202-203` raises `ValueError("Cannot reconstruct aggregates
with n < 1")` but the only caller is `incremental_cov_update` which already
guards `n < 2` two lines earlier. The `n < 1` branch is unreachable.

---

## Notes from the review

A few things I want to keep in my head as I go into Day 8.

**The two caches don't talk to each other.** `perf/cache.py` is in-memory,
keyed by `(start, end, universe_hash, freq)`. `perf/rolling_cache.py` is
disk-backed, keyed by `(sha256_of_frames, window, freq, method)`. They both
hash the inputs, both log through `timing.py`, and both implement
`get_or_compute`. Different design choices for different problems (covariance
across overlapping windows vs. expensive rolling reductions on a fixed
dataset), so I don't think we should *merge* them, but a shared
`CacheBackend` protocol with a common `get_or_compute` signature would let
callers swap implementations and would make the `log_timing` calls
consistent. Putting it on the Week 2 write-up list, not acting today.

**The robustness story is incomplete.** Between `robust_config.py`,
`RobustMeanVariance`, and `RobustRiskParity` we have three layers translating
config flags into engine behaviour, with a `safe_mode` fallback that can
land you back in HRP or RP. That's a reasonable design but I want to confirm
the diagnostics dict on `RobustMeanVariance` actually surfaces in our reports
— there's no point computing `condition_source` and `fallback_used` if
nothing logs them downstream. Adding to my Day 11 (reporting) checklist.

**The metrics registry is barely used.** `_METRIC_REGISTRY` /
`available_metrics()` exists in `metrics/__init__.py` but I grepped and
nothing in `src/` reads from it — callers all import the metric functions
directly. Either we wire it up to the report builder so users can configure
their own metrics list, or we delete the registry. I lean toward wiring it
up but want to see how reporting is structured before deciding.

**`HRP` has a bare `except Exception` at the bottom (line 119).** Catches
literally anything and returns equal weights. I understand the intent — HRP
has a lot of moving parts and we don't want one bad correlation matrix
crashing the run — but it makes debugging genuinely hard because real bugs
get swallowed. I'd narrow this down to the specific scipy/numpy errors we've
seen in the wild, or at minimum log the traceback at error level so we can
spot it in the logs.

**Test coverage feels uneven.** I didn't run the full suite, but the legacy
`annualize_*` aliases have multiple dedicated tests
(`test_metrics.py`, `test_metrics_extra.py`, `test_metric_vectorise.py`)
while I don't see anything specifically exercising `RobustMeanVariance`'s
`safe_mode` fallback path. That's the kind of gap the Day 16 coverage audit
should catch but I'm noting it now so I don't forget.

---

## Summary

Three small packages, mostly in good shape, but with a handful of things I'd
like to clean up. The `metrics/__init__.py` builtins/`sys.modules` patches
and the misleading shrinkage names are the two I'd want to fix soonest. Most
of the rest is duplication and mild over-engineering that I can roll into a
single "weights cleanup" PR after Week 2 is done.

Tomorrow (Day 8) I'm into `portfolio/`, `rebalancing/`, and `multi_period/` —
roughly 5K lines, with `multi_period/` being the lion's share. I'll budget
most of the day for that one.
