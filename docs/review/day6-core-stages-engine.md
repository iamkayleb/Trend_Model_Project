# Day 6 Review: `core/`, `stages/`, and `engine/` subpackages

**Scope:** 10 files, ~3,794 lines — the computational heart of the pipeline
**Date:** 2026-04-24
**Reviewer:** Claude

---

## Files Reviewed

- `core/__init__.py` — Package docstring and nothing else. Callers have to import from submodules directly (`from ..core.rank_selection import ...`), which is different from how `stages/` and `engine/` handle it but works fine.
- `core/metric_cache.py` — The small, focused memoisation layer for scalar metric series. `MetricCache` keeps an ordered dict with hit/miss counters; `make_metric_key` SHA-1s the window bounds + universe + config hash into a deterministic key; `get_or_compute_metric_series` is the one-liner callers use. Opt-in via an `enable` flag so the cache can be bypassed entirely.
- `core/rank_selection.py` — The rank-selection workhorse. Mixes four things: hashing/canonicalisation helpers, a window-metric caching layer (`WindowMetricBundle`, `WindowMetricCache`), the metric registry with lambda adapters for `AnnualReturn`/`Volatility`/`Sharpe`/`Sortino`/`MaxDrawdown`/`InformationRatio`/`AvgCorr`, and the actual `rank_select_funds` + `blended_score` selection logic. See issues below.
- `engine/__init__.py` — Clean re-export of `walk_forward`, `WalkForwardResult`, `Split`.
- `engine/optimizer.py` — Portfolio constraint projection. `ConstraintSet` dataclass plus `apply_constraints`, which walks a fixed pipeline: long-only clipping → normalise → optional CASH carve-out → per-asset cap with iterative redistribution → group caps with proportional shrink-and-redistribute → re-cap pass → final sum-to-one. Raises `ConstraintViolation` on infeasibility. Well-bounded file.
- `engine/walkforward.py` — The *computational* walk-forward (distinct from `trend_analysis/walk_forward.py`, which is the YAML-driven orchestration layer — flagged back on Day 4). Builds fixed rolling train/test splits, aggregates per window, slices out-of-sample union by regime, and infers `periods_per_year` from the median spacing of the index.
- `stages/__init__.py` — Re-exports the three stage modules' `_`-prefixed helpers. See issue below.
- `stages/preprocessing.py` — `_prepare_preprocess_stage` (date-column checks, calendar alignment, frequency detection, missing-policy application, monthly resampling, column-count validation) plus `_build_sample_windows` which slices the prepared frame into in-sample and out-of-sample windows with month-end continuity reapplied if the missing policy is non-`drop`. Returns `PipelineResult` failures instead of raising for the expected empty-window/no-columns cases.
- `stages/selection.py` — The fund-selection stage. `_resolve_risk_free_column` picks the RF series with a coverage check (60% non-null in the window) and an opt-in fallback to the lowest-vol column; `single_period_run` builds the score frame for a single window; `_select_universe` glues together manual, random, and rank selection modes and applies the missing-data filter.
- `stages/portfolio.py` — The computation stage. `_compute_weights_and_stats` is the big one (~500 lines): runs the weight engine with safe-mode detection and fallback, extracts constraints from the config mapping, computes trend signals, calls `compute_constrained_weights` with a defensive fallback if it raises, applies the weight policy, computes cash weight, scales returns and deducts costs, zeroes out the warm-up, and builds three parallel `_Stats` bundles (user, equal-weight, raw-out-sample) for both in-sample and out-of-sample. `_assemble_analysis_output` then builds benchmarks, regime summaries, and the final payload.

---

## Issues Found

### 1. `DEFAULT_METRIC` is defined twice with different values

`core/rank_selection.py` assigns `DEFAULT_METRIC = "annual_return"` at line 39 and then reassigns it to `"AnnualReturn"` at line 879. These are 840 lines apart, so it's very easy to miss.

The kicker is that `rank_select_funds` is defined at line 462 with `score_by: str = DEFAULT_METRIC` in its signature. Default arguments are evaluated when the `def` runs, not each time the function is called — so that default captures the string `"annual_return"` (the value at line 39). The later reassignment at line 879 updates the module attribute, but the function's bound default stays as the lowercase form.

End state:
- `trend_analysis.core.rank_selection.DEFAULT_METRIC` → `"AnnualReturn"`
- `inspect.signature(rank_select_funds).parameters["score_by"].default` → `"annual_return"`

It works in practice only because `_METRIC_ALIASES` maps `"annual_return"` → `"AnnualReturn"` inside the function body. If someone drops that alias or relies on introspecting the default, the two will disagree.

**Recommendation:** Delete the line 39 assignment and move the canonical one above `rank_select_funds` so the default captures the real value.

---

### 2. The AvgCorr-from-covariance computation is written out three times in one file

`core/rank_selection.py` has three near-identical blocks that compute AvgCorr (or `__COV_VAR__`) from a covariance matrix:

- `_cov_metric_from_payload` at lines 300–313
- `_metric_from_cov_payload` at lines 964–980
- Inlined inside `compute_metric_series_with_cache` at lines 1030–1041

All three do the same thing — clip the covariance diagonal to zero, square-root, build the correlation matrix via outer-product division, sum each row and subtract 1 for the self-correlation, divide by `n-1`. The only difference is where the column labels come from.

And `_metric_from_cov_payload` + `_ensure_cov_payload` (lines 948–981) aren't called by any production code — the only references are in `tests/test_rank_selection_uncovered.py`. They're orphaned helpers kept alive by coverage tests.

**Recommendation:** Extract one private helper (e.g. `_avg_corr_from_cov(payload, columns)`), route the three live call sites through it, and delete `_metric_from_cov_payload`, `_ensure_cov_payload`, and their coverage tests. Coverage numbers will drop by the amount of dead code that was previously "tested", which is the correct behaviour.

---

### 3. `selector_cache_stats()` overwrites fresh values with a potentially stale mirror

`core/rank_selection.py:404–410`:

```python
def selector_cache_stats() -> dict[str, int]:
    stats = _WINDOW_METRIC_CACHE.stats()
    stats["selector_cache_hits"] = _SELECTOR_CACHE_HITS
    stats["selector_cache_misses"] = _SELECTOR_CACHE_MISSES
    return stats
```

`_WINDOW_METRIC_CACHE.stats()` already returns `"selector_cache_hits"` and `"selector_cache_misses"` populated straight from the live `WindowMetricCache._hits` / `_misses` counters. The next two lines then overwrite those fresh values with the private module-level mirrors, which only stay in sync because `_sync_cache_counters()` is called manually from a handful of places.

If any code path records a hit or miss without calling `_sync_cache_counters()` afterwards, `selector_cache_stats()` will return stale numbers even though `WindowMetricCache` itself has the right ones. That's the wrong direction for a stats function to fail in.

**Recommendation:** Drop the two overwrite lines. `WindowMetricCache.stats()` is authoritative.

---

### 4. `core/rank_selection.py` at 1,141 lines carries four distinct concerns

The file holds:

- **Hashing / canonicalisation helpers** — ~60 lines
- **Window-metric caching infrastructure** — `WindowMetricBundle`, `WindowMetricCache`, the context-scoping, covariance-payload plumbing — ~350 lines
- **Metric registry + adapters** — `METRIC_REGISTRY`, `register_metric`, `AnnualReturn`/`Volatility`/etc. adapters, the frame-aware `AvgCorr` — ~120 lines
- **Rank selection** — `_apply_transform`, `rank_select_funds`, `blended_score`, `_compute_metric_series`, `_call_metric_series` — ~430 lines
- **UI hook** — 8 lines

The selector is the point of the file; the cache and registry are scaffolding. The file's length is part of why issue 1 (`DEFAULT_METRIC`) was easy to miss — 840 lines between the two assignments.

**Recommendation:** No immediate action. A natural split would be `core/selector_cache.py` for the bundle and window-cache machinery, `core/metric_registry.py` for the registry and metric adapters, keeping `rank_selection.py` focused on selection. Medium-effort refactor, pays off the next time anyone needs to extend any of those three areas.

---

### 5. `stages/__init__.py` re-exports `_`-prefixed helpers as public API

The `__all__` list contains 15 `_`-prefixed names and 2 public names. All 15 are imported by other modules (`pipeline.py` alone pulls in a dozen; `pipeline_runner.py` imports the stage modules as namespaces; `monte_carlo/runner.py` and `monte_carlo/config.py` reach in for `single_period_run` and `_resolve_risk_free_column`). The leading underscore is supposed to say "internal to this module"; here it means nothing.

It's genuinely confusing on first read — you see `_SelectionStage` in `__all__` and assume it's accidentally exposed, then find it's deliberately imported by half the codebase.

**Recommendation:** Pick a side. Either drop the underscores because these names are effectively public, or keep them and stop re-exporting from `__init__.py` — callers would then write `from ..stages.portfolio import _ComputationStage`, which at least makes the "I am crossing a private boundary" moment visible.

---

### 6. `RiskStatsConfig` imported twice in `stages/selection.py::single_period_run`

`stages/selection.py` imports `RiskStatsConfig` at the top of the module (line 11), and then again inside `single_period_run` at line 263:

```python
from ..core.rank_selection import RiskStatsConfig, _compute_metric_series
```

Only `_compute_metric_series` actually needs the function-local import (it's a private helper imported where used). The re-import of `RiskStatsConfig` just shadows the top-level one with an identical reference.

**Recommendation:** `from ..core.rank_selection import _compute_metric_series`.

---

## Notes (not actionable)

### The selector-cache counter mirrors are historical

`rank_selection.py` tracks cache hits and misses in three places: `WindowMetricCache._hits` / `_misses` on the class, the public `selector_cache_hits` / `selector_cache_misses` module attributes, and the private `_SELECTOR_CACHE_HITS` / `_SELECTOR_CACHE_MISSES` mirrors. `_sync_cache_counters()` has to be called to keep them aligned. The public attributes exist because older tests imported them directly; the private mirrors exist so `selector_cache_stats()` can read "the latest sync'd value". That's a lot of moving parts for two integers. A single `@property` on the cache class plus a back-compat module-level shim would be cleaner, but it works and removing it means touching tests.

### `_compute_weights_and_stats` has a silent fallback branch

`stages/portfolio.py:484–521` catches any exception from `compute_constrained_weights` and falls back to "base weights + crude vol-scaling" with a warning log. That branch is genuinely important — it's the pipeline's degraded-but-producing-output mode — but in a 500-line function it's easy to miss. A one-line block comment above the `except` would save a future reader two trips through the code.

### `engine/walkforward.py::_infer_periods_per_year` duplicates intent from `risk.py`

Lines 110–137 infer periods-per-year from the median day-spacing of a `DatetimeIndex` using a hardcoded ladder (`≥300 → 252`, `45–60 → 52`, `10–14 → 12`, `3–5 → 4`). `trend_analysis/risk.py::periods_per_year_from_code` does the conceptually identical mapping from a frequency code string. The inputs differ, so they aren't interchangeable, but the "12 months / 252 trading days / 52 weeks" constants should come from one place — right now they're duplicated across modules.

---

## No Issues (clean files)

- `core/__init__.py` — minimal but intentional
- `core/metric_cache.py` — clean, well-scoped caching helper
- `engine/__init__.py` — clean re-exports
- `engine/optimizer.py` — methodical constraint projection; complexity is inherent to the problem
- `engine/walkforward.py` — complex but single-concern
- `stages/preprocessing.py` — large but single-concern; failure paths return `PipelineResult` cleanly
- `stages/selection.py` — clean aside from issue 6
- `stages/portfolio.py` — large but the complexity is inherent

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| Low | Remove the duplicate `DEFAULT_METRIC = "annual_return"` at line 39 and reorder so the canonical value is defined before `rank_select_funds` | `core/rank_selection.py` |
| Low | Extract a single AvgCorr-from-cov helper; delete dead `_ensure_cov_payload` / `_metric_from_cov_payload` and their coverage tests | `core/rank_selection.py`, `tests/test_rank_selection_uncovered.py` |
| Low | Drop the stale-mirror overwrite in `selector_cache_stats()` | `core/rank_selection.py` |
| Low | Remove the redundant `RiskStatsConfig` re-import in `single_period_run` | `stages/selection.py` |
| Info | Decide whether `stages/__init__.py` should expose `_`-prefixed helpers or keep the underscores and stop re-exporting | `stages/__init__.py` |
| Info | Consider splitting `core/rank_selection.py` into separate registry, cache, and selector modules | `core/rank_selection.py` |
