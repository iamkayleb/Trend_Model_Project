# Week 2, Day 7 Review: `weights/`, `metrics/`, `perf/`

**Reviewer:** Kayleb
**Date:** 2026-05-01
**Scope:** `src/trend_analysis/weights/` + `src/trend_analysis/metrics/` + `src/trend_analysis/perf/`
**Total:** 14 files, ~2,062 lines

---

## Files Reviewed

### `src/trend_analysis/weights/` (6 files, 759 lines)

| File | Lines | What it does |
|------|------:|--------------|
| `__init__.py` | 14 | Re-exports the four weight engines. Nothing surprising. |
| `risk_parity.py` | 50 | Plain inverse-volatility weighting. Registered under both `risk_parity` and `vol_inverse`. |
| `equal_risk_contribution.py` | 102 | ERC via iterative scaling, with a regularisation pass when the cov matrix is ill-conditioned. |
| `hierarchical_risk_parity.py` | 122 | HRP using `scipy.cluster.hierarchy`. Falls back to equal weights on any numerical hiccup. |
| `robust_config.py` | 81 | Translates a `robustness:` config block into kwargs for the robust engines. Pure mapping logic, no math. |
| `robust_weighting.py` | 390 | The biggest module here. Hosts `RobustMeanVariance` and `RobustRiskParity`, plus the Ledoit-Wolf / OAS shrinkage and diagonal-loading helpers. |

### `src/trend_analysis/metrics/` (5 files, 760 lines)

| File | Lines | What it does |
|------|------:|--------------|
| `__init__.py` | 439 | The metric registry plus the core stats: `annual_return`, `volatility`, `sharpe_ratio`, `sortino_ratio`, `max_drawdown`, `information_ratio`. Also wires up the legacy `annualize_*` aliases. |
| `attribution.py` | 134 | Per-signal contribution table + a matplotlib plotter. |
| `rolling.py` | 67 | Rolling information ratio with optional caching via `perf.rolling_cache`. |
| `summary.py` | 64 | One-shot performance summary: CAGR, vol, max DD, IR, Sharpe, turnover, costs, hit rate. |
| `turnover.py` | 56 | `realized_turnover` and `turnover_cost`. Vectorised, no plotting deps. |

### `src/trend_analysis/perf/` (3 files, 543 lines)

| File | Lines | What it does |
|------|------:|--------------|
| `cache.py` | 267 | In-memory `CovCache` with optional LRU plus `compute_cov_payload` / `incremental_cov_update` for sliding-window updates. |
| `rolling_cache.py` | 195 | Filesystem-backed cache for rolling metrics, keyed on a SHA-256 dataset hash. Falls back to `tempfile` if `~/.cache` isn't writable. |
| `timing.py` | 81 | `log_timing` and the `timed_stage` context manager. Used by both caches above. |

---

## Issues Found

The good news first: this is the cleanest stretch of the codebase I've reviewed so far. Everything actually does what its docstring says, the registries are wired up correctly, and the math is defensible. That said, there are a handful of things I want to flag.

### 1. The `tests.legacy_metrics` shim and the `builtins` injection

`metrics/__init__.py` lines 404–432 do something I really don't love:

```python
_legacy = types.ModuleType("tests.legacy_metrics")
...
sys.modules.setdefault("tests.legacy_metrics", _legacy)
setattr(_bi, "annualize_return", annualize_return)
setattr(_bi, "annualize_volatility", annualize_volatility)
```

We're synthesising a fake `tests.legacy_metrics` module at import time and stuffing two functions onto Python's `builtins` so they're available as bare names anywhere. The only places that actually rely on this are `scripts/run_multi_demo.py` and `tests/`. This is a back-compat hack from a past rename that never got cleaned up. It's not actively broken, but it's the kind of thing that will bite us during the next big refactor — anything that imports `trend_analysis.metrics` is silently mutating global namespaces.

**Recommendation:** add a TODO to remove it once `run_multi_demo.py` is updated. Low-risk, but worth scheduling.

### 2. Two parallel registry patterns living side-by-side

The weights subsystem uses `..plugins.weight_engine_registry` (a proper `PluginRegistry`).
The metrics subsystem uses a plain module-level `_METRIC_REGISTRY: dict[str, Callable]` with a local `register_metric` decorator.

They do roughly the same job but the metrics one doesn't get any of the plugin-system goodies (entry-point discovery, parameter passing, etc.). Not urgent — metrics functions don't really need the heavier machinery — but worth noting that we have two ways to register pluggable functions in adjacent modules.

### 3. Soft duplicate: `_timed_stage` in `signals.py` vs `perf.timing.timed_stage`

`src/trend_analysis/signals.py:96` defines its own `_timed_stage` context manager. It's nearly identical in shape to `perf.timing.timed_stage` — the only real difference is that `signals` logs at DEBUG via its own logger, whereas `perf.timing` logs at INFO via `trend_analysis.performance`. The two existed independently; nobody ever consolidated them.

**Recommendation:** when we touch `signals.py` next (it's on Day 4 of Week 1 in the original plan), refactor it to use `perf.timing.timed_stage` with a logger override. Saves ~15 lines and gives us one timing path.

### 4. The Ledoit-Wolf and OAS implementations are approximations

In `robust_weighting.py:30-111`, both shrinkage functions explicitly note they're "simplified" because the raw return data isn't passed in — only the sample covariance. The intensity calc uses a heuristic `n_samples` estimate (`max(p+1, p*2)`) when the caller doesn't pass it. That's fine in principle, but two things worry me:

- We never actually pass `n_samples` from the call sites. `_apply_shrinkage` calls `ledoit_wolf_shrinkage(cov)` with no `n_samples`, so we *always* hit the heuristic in production.
- The library `scikit-learn` has a battle-tested `LedoitWolf` and `OAS` estimator. We avoid sklearn (I checked — it's not imported anywhere in `src/`), which is a defensible choice for keeping deps light, but we should at least pass `n_samples` from the actual window length when we know it. The plumbing already exists in `multi_period/engine.py` where covariances are built from a known window.

**Recommendation:** thread `n_samples` through `_apply_shrinkage` so production calls don't silently use the heuristic. Two-line change, real correctness improvement.

### 5. `RobustMeanVariance._safe_mode_weights` instantiates engines on every call

Each call to `weight()` that lands in safe mode constructs a fresh `HierarchicalRiskParity()` or `RiskParity()` (lines 199–207). HRP isn't free — it does linkage clustering. In a multi-period backtest with rolling windows that frequently fall back to safe mode (and they will, on real-world data), this is wasted work. Both classes are stateless aside from `__init__` args, so caching the instance on `self` after the first call would be safe.

Minor performance issue, not a correctness one.

### 6. `metrics/__init__.py` is doing too much

439 lines for an `__init__` is a smell. It contains:

- The registry decorator
- Six core metric implementations
- Internal helpers (`_empty_like`, `_validate_input`, `_check_shapes`, `_compute_ratio_with_zero_handling`, `_is_zero_everywhere`)
- The legacy aliases and the `builtins` injection
- Late imports of submodules at the bottom

I'd split the core metric functions into `metrics/core.py` and keep `__init__.py` for re-exports and registry wiring only. Not a bug, just hygiene — but it's where I'd start if a new contributor asked "where do I add a new metric?".

### 7. Sortino's "single downside" branch is glued to a golden test

```python
# Special handling for single downside observation to match golden test expectations
if len(downside) == 1:
    down_vol = 2.0 * abs(downside.iloc[0])
```

Repeated twice (Series path lines 276-281, DataFrame path 296-301). The `2.0 * abs(...)` factor isn't a Sortino convention — it's a workaround so a snapshot test passes. We should either (a) document why this is the right answer mathematically, or (b) update the golden test to match `std(ddof=1)` semantics and delete this branch. Right now it looks like cargo cult code.

### 8. `attribution.py` has very thin coverage and almost no callers

The only place that imports `compute_contributions` outside the file itself is `tests/test_metrics_attribution.py`. The streamlit hits I found are for an unrelated `report_attribution` config flag, not this module. So we're shipping a public API (`__all__` export) that nothing in `src/` actually consumes.

Either it's planned future work — in which case we should leave a note — or it's dead-on-arrival. Worth asking around before we delete, but it's a candidate for the dead-code list at the end of Week 4.

### 9. `metrics/summary.py` `summary_table` is also barely used

Same story. Only direct caller is `tests/test_metrics_summary.py`. The export pipeline uses a different function (`metrics_from_result` in `export/__init__.py`) which builds its own summary. So we have two different "summary" code paths. Probably should consolidate them, or at least document which one is canonical.

---

## Notes from the Review

A few broader observations I want to flag for the write-up at the end of the month:

**The robustness story is well-thought-out.** The pattern across all four engines — check condition number, apply shrinkage / regularisation, fall back to equal weights on any numerical failure — is consistent and the logging is good. Whoever wrote `RobustMeanVariance` clearly thought about the failure modes (`used_safe_mode`, `fallback_reason`, the diagnostics dict). I'd hold this up as the reference for how the rest of the engine modules should handle bad inputs.

**The two-tier caching design is sensible.** `perf.cache.CovCache` is in-memory with LRU for the hot path inside a single backtest run. `perf.rolling_cache.RollingCache` is filesystem-backed for cross-run reuse of expensive rolling computations. They don't overlap in scope and that's deliberate. The `compute_dataset_hash` SHA-256 keying is a nice touch — survives column reorderings and dtype quirks.

**`incremental_cov_update` is the most interesting piece in here.** The identity it uses (`S2' = S2 - oo^T + nn^T`) lets a sliding window roll forward in O(k²) instead of O(n·k²). I checked the call site in `multi_period/engine.py:1130–1199` and it's actually wired up. This is a meaningful perf win in walk-forward backtests and it's quietly tucked away in a 60-line function. Worth highlighting.

**Mypy / type-hint quality is uneven.** `metrics/__init__.py` has lots of `cast(...)` and `Any` workarounds for pandas stub limitations (see lines 180–202). The weights modules are cleaner. Not a defect — just a heads-up that the metrics module is harder to reason about statically.

**No security or data-handling red flags.** The only filesystem write is `RollingCache` writing joblib files into `~/.cache/trend_model/rolling`, with a tempdir fallback. The path normalisation (`_safe_resolve`, `_normalise_component`) handles the obvious traversal cases. Hashing uses SHA-256 (not MD5). Nothing to escalate.

**Test coverage looks solid based on file names.** I spotted `test_weighting_robustness.py`, `test_robust_weighting.py`, `test_robust_weighting_integration.py`, `test_weight_engines_pathological.py`, `test_metrics_attribution.py`, `test_metrics_summary.py`, `test_metrics_rolling.py`, `test_metrics_rolling_cache_disabled.py`. I'm deferring the actual coverage audit to Day 16 per the plan.

---

## Suggested Action Items (low priority, defer to Week 4)

1. Remove the `tests.legacy_metrics` / `builtins` injection once `scripts/run_multi_demo.py` no longer relies on it.
2. Thread `n_samples` through to the Ledoit-Wolf / OAS calls in `RobustMeanVariance._apply_shrinkage`.
3. Cache the safe-mode engine instance on `self` in `RobustMeanVariance`.
4. Split `metrics/__init__.py` into `__init__.py` (re-exports) and `core.py` (implementations).
5. Resolve the Sortino "single downside" branch — either justify it in a comment or drop it.
6. Decide the fate of `attribution.py` and `summary.py` — confirm with the team whether they're planned-but-unwired or actually obsolete.
7. Consolidate `signals.py::_timed_stage` with `perf.timing.timed_stage`.

Nothing here is urgent. No bugs that affect production output. Mostly maintenance debt and one or two correctness nits worth eventually fixing.

---

*On to Day 8: `portfolio/`, `rebalancing/`, `multi_period/`. The `multi_period` engine is the big one (4,352 lines on its own), so I'm budgeting most of tomorrow for it.*
