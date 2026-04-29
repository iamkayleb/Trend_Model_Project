# Day 7 Subsystem Review — `weights/`, `metrics/`, `perf/`

**Reviewer:** Kayleb
**Date:** 2026-04-29
**Scope:** Week 2, Day 7 of the rolling subsystem audit. Covers the three packages that sit between the analysis engine and the reporting layer: weighting strategies, performance metrics, and the lightweight caching/timing helpers that back them.
**Total footprint:** 14 files, ~2,062 lines.

---

## Files Reviewed

### `src/trend_analysis/weights/` (5 files, 759 lines)

| File | Lines | What it does |
|------|------:|--------------|
| `__init__.py` | 14 | Re-exports the four engine classes. |
| `risk_parity.py` | 50 | Plain inverse-volatility weighting. |
| `equal_risk_contribution.py` | 102 | ERC via iterative scaling with regularisation. |
| `hierarchical_risk_parity.py` | 122 | HRP using SciPy's single-linkage clustering. |
| `robust_config.py` | 81 | Translates the YAML "robustness" block into engine kwargs. |
| `robust_weighting.py` | 390 | `RobustMeanVariance` + `RobustRiskParity`, plus Ledoit-Wolf, OAS and diagonal-loading helpers. |

### `src/trend_analysis/metrics/` (5 files, 760 lines)

| File | Lines | What it does |
|------|------:|--------------|
| `__init__.py` | 439 | The big one — registry, scalar/vector metrics (CAGR, vol, Sharpe, Sortino, MDD, IR), legacy aliases. |
| `attribution.py` | 134 | PnL contribution table + CSV export + matplotlib plot. |
| `rolling.py` | 67 | Rolling information ratio (cache-aware). |
| `summary.py` | 64 | One-shot summary table that wires returns + weights + costs into a tidy DataFrame. |
| `turnover.py` | 56 | Realised turnover and linear cost helpers. |

### `src/trend_analysis/perf/` (3 files, 543 lines)

| File | Lines | What it does |
|------|------:|--------------|
| `cache.py` | 267 | In-memory `CovCache` (LRU-capable) plus incremental covariance update via S1/S2 aggregates. |
| `rolling_cache.py` | 195 | Filesystem-backed cache for rolling outputs, joblib-serialised, home-dir-scoped. |
| `timing.py` | 81 | `log_timing()` and the `timed_stage` context manager — the timing primitives the caches use. |

---

## Issues Found

I'll keep this in priority order so we know what's worth fixing now versus what's just worth knowing.

### 1. Duplicate `_timed_stage` implementation in `signals.py`
**Severity:** Medium. **Action:** Consolidate.

`src/trend_analysis/perf/timing.py` already provides a `timed_stage` context manager that emits structured stage logs. But `src/trend_analysis/signals.py:97` defines its own private `_timed_stage` that does almost exactly the same thing — DEBUG-gated stopwatch around a `yield`. The signals version logs to a different logger (`trend_analysis.signals` vs `trend_analysis.performance`), so the duplication is partly intentional, but it still means we now have two slightly different patterns for "time this block." I'd switch `signals.py` to call the shared helper and pass a logger name through, so we have one timing path everywhere.

### 2. `metrics/__init__.py` is monkey-patching `builtins`
**Severity:** Medium-High. **Action:** Remove or quarantine.

Lines 431–432:

```python
setattr(_bi, "annualize_return", annualize_return)
setattr(_bi, "annualize_volatility", annualize_volatility)
```

Importing the metrics module silently puts two names on Python's `builtins`. That's the kind of thing that bites you six months later when something else shadows them or a test runs in a different import order. If the goal is back-compat for old notebooks, we should expose them through an explicit compatibility module instead of patching builtins. This is the single biggest "this would surprise a reviewer" thing I found today.

### 3. Synthetic `tests.legacy_metrics` module collides with the real one
**Severity:** Medium. **Action:** Pick one.

`metrics/__init__.py` (lines 412–426) builds a fake module and registers it with `sys.modules.setdefault("tests.legacy_metrics", _legacy)`. But there's a real `tests/legacy_metrics.py` on disk too. Whichever import happens first wins, so the resolution depends on test collection order. Two of the tests (`test_metric_vectorise.py`, `test_metric_vectorise_param.py`) actually `import tests.legacy_metrics as L` and compare against it. We should delete the synthetic registration and keep the real file, or vice versa — but not both.

### 4. `metrics/__init__.py` is doing too many things
**Severity:** Low (cosmetic, but the file is 439 lines).

In one file we have: a registry decorator, six scalar/vector metric implementations, several private helpers, the legacy `annualize_*` aliases, the `tests.legacy_metrics` shim, the builtins patch, and `import_module(...)` calls that re-attach the four submodules. Splitting the registry + helpers from the metric implementations would make this much easier to read and would let us drop the side-effect block at the bottom altogether.

### 5. Ledoit-Wolf / OAS shrinkage are explicitly "simplified" approximations
**Severity:** Low (documented), but worth flagging to the boss.

`robust_weighting.py:30-71` and `:74-111` carry comments such as *"Simplified Ledoit-Wolf approach … When we don't have access to raw data, use matrix-based approximation"*. The intensity uses an `n_samples` heuristic of `max(p+1, 2p)` when the caller doesn't pass a real sample size. That's fine as a defensive default, but it means our shrinkage intensities will not match `sklearn.covariance.LedoitWolf` for the same data. We should either (a) plumb the actual `n_samples` through from the cov-payload (we already track `n` in `CovPayload`) or (b) document the deviation in the engine docstring so users don't think they're getting textbook LW.

### 6. HRP's catch-all `except Exception`
**Severity:** Low. **Action:** Narrow it.

`hierarchical_risk_parity.py:119-122` falls back to equal weights on *any* exception. That's safe in production but it will silently swallow bugs. At minimum I'd log the traceback, not just `str(e)`, so we have something to debug from when it fires.

### 7. `RobustRiskParity` diagnostic mismatch
**Severity:** Cosmetic.

In `robust_weighting.py:330-390`, `diagonal_loading` can be applied twice (once for non-positive diagonals, once for high condition number), but `diagnostics["used_diagonal_loading"]` is set from a single boolean comparison at the end (`condition_num >= self.condition_threshold`). If we triggered on the diagonal-non-positive branch but the condition number was fine, the diagnostic will say "False." Easy fix: track a flag at the point of application.

### 8. ERC's regularisation thresholds are hard-coded
**Severity:** Low.

`equal_risk_contribution.py` uses `1e12` for the condition-number threshold and `1e-6 / 1e-4` for diagonal loading factors. The robust engines accept these as constructor params, but the plain `EqualRiskContribution` does not. Inconsistent surface area between engines.

### 9. `weights/__init__.py` underexports
**Severity:** Cosmetic.

The package exports the four engine classes but not `weight_engine_params_from_robustness`, `ledoit_wolf_shrinkage`, `oas_shrinkage`, or `diagonal_loading`. Three call sites (`api.py`, `pipeline.py`, `multi_period/engine.py`, `pipeline_entrypoints.py`) all reach into the submodule directly. Not broken — just means there's no canonical "this is the package's public surface" line.

### 10. Rolling cache silently disables itself
**Severity:** Low. **Action:** Log it.

`rolling_cache.py:91-102` falls back from the configured cache dir → `$TMPDIR/trend_model/rolling` → disabled, with no log line at any step. If a deployment ends up with caching off because of a permissions issue, we'll never know unless we measure. A single warning when the fallback fires would be cheap insurance.

---

## Notes from the Review

A few impressions, in roughly the order they hit me:

**The weights package is in good shape.** Each engine has a clear, narrow responsibility and the registry pattern (`@weight_engine_registry.register("erc")`) keeps the wiring simple. The robustness checks (condition number, regularisation, equal-weight fallback) are consistent across `RiskParity`, `ERC`, and `HRP` — same defensive shape, just adapted to each algorithm. That consistency is honestly nicer than I expected, and I'd hold onto it as the standard for any new engines we add.

**`robust_config.py` is the workhorse for translating user YAML into engine kwargs**, and it's doing real work — multiple precedence chains for `condition_threshold`, `safe_mode`, `diagonal_loading_factor`. It's not pretty, but it's defensive against the various ways our config has evolved. The good news: it's well isolated, so we can refactor it without touching the engines.

**The metrics package is the messiest of the three.** Not because the metric *implementations* are bad — the actual math (`annual_return`, `volatility`, `sharpe_ratio`, `sortino_ratio`, `max_drawdown`, `information_ratio`) is careful about Series vs DataFrame paths, NaN handling, and zero-division. The mess is concentrated in `__init__.py`'s side-effect block at the bottom. Issues 2, 3, and 4 above are all really one ticket: clean up that bottom section and the file goes from "scary" to "fine."

**`metrics/summary.py` is genuinely useful but barely used.** It composes returns, weights, costs, and benchmark into a tidy summary DataFrame and only one test (`test_metrics_summary.py`) calls it. That feels like a missed opportunity — this is the natural API for our reporting layer. Worth a follow-up to ask whether this is the function we should be standardising on for top-line stats.

**The `perf/` package is the strongest of the three.** `CovCache` with its `OrderedDict`-backed LRU and the incremental S1/S2 update path in `incremental_cov_update` is a clean implementation of a real performance optimisation — particularly the `_ensure_aggregates` helper that reconstructs S1/S2 from `(cov, mean, n)` when they weren't materialised originally. That's the kind of thing that's easy to get wrong and was clearly thought about. The rolling filesystem cache is also a sensible companion: home-scoped, with hash-based filenames so two different inputs can never collide.

**Caching is wired correctly downstream.** I checked and `compute_dataset_hash` / `get_cache` / `CovCache` are pulled in by `pipeline.py`, `pipeline_helpers.py`, `regimes.py`, `metrics/rolling.py`, and `multi_period/engine.py` — i.e. all the high-volume paths. So the perf work is actually paying off, not dead code.

**No dead code in this slice.** Every file in all three packages is imported from somewhere outside its own package. Nothing here qualifies for the deletion list at the end of Week 4.

**Recommended follow-ups (in order I'd take them):**

1. Delete the `builtins` patch in `metrics/__init__.py` and resolve the `tests.legacy_metrics` ambiguity. (Half-day, mostly mechanical.)
2. Replace `signals._timed_stage` with `perf.timing.timed_stage`. (Hour, with tests.)
3. Document the LW/OAS approximation in the `RobustMeanVariance` docstring, and consider plumbing `n_samples` from `CovPayload.n`. (Half-day.)
4. Split `metrics/__init__.py` into `metrics/registry.py`, `metrics/core.py`, and a thin `__init__.py` that just re-exports. (Half-day.)
5. Add a one-line warning when `RollingCache` falls back to tempdir or disables itself. (Fifteen minutes.)

Nothing here is on fire. The shape of these subsystems is healthy; the issues are mostly about hygiene, surprise-reduction, and giving `metrics/__init__.py` the same careful organisation that `perf/cache.py` already has.

— K.
