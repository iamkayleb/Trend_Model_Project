# Day 4 Review: src/trend_analysis/ root standalone files (Part 2)

**Scope:** Remaining 19 files by line count (172–773 lines each), total ~6,500 lines
**Date:** 2026-04-20
**Reviewer:** Claude

---

## Files Reviewed

- `weighting.py` — Defines `BaseWeighting` ABC and four concrete strategies: `EqualWeight`, `ScorePropSimple`, `ScorePropBayesian`, `AdaptiveBayesWeighting`. The adaptive strategy is stateful with exponential decay and `get_state`/`set_state` for serialization across pipeline runs.
- `diagnostics.py` — Imports `DiagnosticPayload`, `DiagnosticResult`, `RunPayload`, `RunPayloadResult` from `trend.diagnostics` and adds `PipelineResult` (dict-like dataclass), `PipelineReasonCode` enum (9 codes), and three coercion helpers. Used by `pipeline_runner.py` to wrap execution results.
- `schedules.py` — Provides `get_rebalance_dates`, `normalize_positions`, and `apply_rebalance_schedule`. Handles frequency string aliases (`"monthly"`, `"weekly"`), timezone alignment, and custom rebalance calendars. Distinct from the `rebalancing.py` compat shim.
- `signals.py` — Core signal computation: `TrendSpec` frozen dataclass (kind, window, min_periods, lag, vol_adjust, vol_target, zscore) plus `compute_trend_signals()`. Vectorized, strictly causal rolling signal with memoization via a `_FrameHandle` sentinel. Single-responsibility, no issues.
- `time_utils.py` — `align_calendar()` aligns a DataFrame's Date column to a business-day/weekly/monthly calendar with US holiday generation and timezone handling. Exports only `align_calendar`. Used by the pipeline to normalise dates before signal computation.
- `pipeline_runner.py` — Contains `_run_analysis_with_diagnostics()` (the real implementation returning a `PipelineResult`) and `_run_analysis()` (a one-liner backward-compat wrapper that calls `_run_analysis_with_diagnostics` and unwraps the result). See issue below.
- `pipeline_entrypoints.py` — `ConfigBindings` dataclass plus `run_from_config()` and `run_full_from_config()`. These two functions bridge Pydantic config objects to `pipeline.run()` and `pipeline.run_full()`. See issues below.
- `universe.py` — `MembershipWindow`, `MembershipTable`, `load_universe_membership`, `apply_membership_windows`, `build_membership_mask`, `gate_universe`. `_expand_active_pairs` uses numpy broadcasting for efficient date × membership activity computation. Used by `universe_catalog.py`.
- `risk.py` — Stateless functions: `compute_constrained_weights`, `realised_volatility`. Handles vol targeting, turnover caps, EWMA vs simple rolling vol, group caps, max active positions. Delegates constraint optimisation to `engine.optimizer`.
- `tool_layer.py` — `ToolLayer` dataclass with `apply_patch`, `validate_config`, `preview_diff`, `run_analysis` methods. Includes rate limiting, path sandboxing, and JSONL logging of all tool calls. Only imported by `src/trend_analysis/api_server/__init__.py`.
- `walk_forward.py` — Self-contained parameter sweep engine: YAML config → grid search → fold evaluation → CSV/JSONL/PNG output. Uses `annual_return`, `sharpe_ratio`, `max_drawdown` from `trend_analysis.metrics`. See issue below regarding a second walk-forward implementation in `engine/`.
- `presets.py` — YAML-driven preset registry (`lru_cache`-backed) that loads `TrendPreset` objects from `config/presets/*.yml`. Provides `form_defaults()`, `signals_mapping()`, `vol_adjust_defaults()`, `metrics_pipeline()`, plus `UI_METRIC_ALIASES` and `PIPELINE_METRIC_ALIASES` mappings. See issue below regarding the parallel `signal_presets.py`.
- `pipeline.py` — Integration hub: imports from stages, `pipeline_runner`, `pipeline_helpers`, and wires them together. Exports `run()`, `run_full()`, `run_analysis()`, `compute_signal()`, `position_from_signal()`. Contains the `_sync_stage_dependencies()` monkeypatching pattern. See issue below.
- `regimes.py` — `RegimeSettings`, `compute_regimes`, `aggregate_performance_by_regime`, `build_regime_payload`. Two detection methods: `rolling_return` and `volatility`. Integrates with the perf cache via `_compute_regime_series`. Well-structured single responsibility.
- `data.py` — `load_csv`, `load_parquet`, `validate_dataframe`, `identify_risk_free_fund`, `ensure_datetime`, `compute_inception_dates`. Central data ingestion layer. See issue below regarding duplication between `load_csv` and `load_parquet`.
- `api.py` — Main public API: `run_simulation(config, returns) -> RunResult`. `RunResult` dataclass wraps metrics, weights, exposures, turnover, costs, portfolio series, and multi-period period results. Detects `config.multi_period` and delegates to `_run_multi_period_simulation()`. Complex but coherent; no structural issues.
- `pipeline_helpers.py` — Hub of config-resolution helpers (`_cfg_value`, `_cfg_section`, `_section_get`, etc.), trend spec construction (`_build_trend_spec`), regime override application, and turnover cap parsing. Also contains `compute_signal()` and `position_from_signal()` — placement slightly odd (signal utilities inside a helpers file) but not harmful.
- `universe.py` — (see entry above; same file as listed at position 8)

---

## Issues Found

### 1. `pipeline_entrypoints.py` — `risk_free_column` assigned twice

In `run_full_from_config`, `risk_free_column` is assigned on line 232 and again on line 235 with a different expression. The second assignment silently overwrites the first. This is a copy-paste error — one of the two assignments is wrong.

**Recommendation:** Read both assignments and determine which is correct; delete the other. The bug is silent in production because the second assignment always wins.

---

### 2. `pipeline_entrypoints.py` — Config extraction duplicated between `run_from_config` and `run_full_from_config`

Both functions open with ~80 lines of identical config extraction boilerplate (resolving universe, split, signals, risk settings, regime overrides, etc.). This makes future changes to the config schema require updates in two places.

**Recommendation:** Extract the shared block into a private `_extract_bindings(cfg) -> ConfigBindings` helper and call it from both functions.

---

### 3. `pipeline_runner.py` — `_run_analysis()` is a trivial compat wrapper

`_run_analysis(cfg, returns)` calls `_run_analysis_with_diagnostics(cfg, returns).unwrap()` and nothing else. It exists only so legacy callers that expect a bare result rather than a `PipelineResult` keep working.

**Recommendation:** Document it explicitly as a compat wrapper (one-line comment). If no external callers remain outside `pipeline.py`, consider removing it and updating the single internal call site.

---

### 4. `pipeline.py` — `_sync_stage_dependencies()` uses runtime `setattr` monkeypatching

`_sync_stage_dependencies()` patches functions from `pipeline_helpers.py` into stage modules at call time using `setattr(module, name, fn)`. This enables test code to swap implementations but is non-obvious and breaks static analysis (mypy, IDEs lose cross-reference).

**No immediate action required** — the pattern is intentional and documented in comments. Worth noting for future refactoring toward dependency injection.

---

### 5. `data.py` — `load_csv` and `load_parquet` share ~80 lines of near-identical boilerplate

Both functions handle the same sequence: resolve `missing_policy`, apply `limit`, validate, `ensure_datetime`, and apply the same optional column filtering. The only difference is the read call (`pd.read_csv` vs `pd.read_parquet`).

**Recommendation:** Extract a `_load_file(read_fn, path, **kwargs) -> pd.DataFrame` helper that both delegate to. Reduces the duplication to a single maintenance point.

---

### 6. `walk_forward.py` vs `engine/walkforward.py` — Two walk-forward implementations

`walk_forward.py` (416 L) is a standalone YAML-config-driven parameter sweep that writes CSV/JSONL/PNG output. `engine/walkforward.py` exposes a lower-level `walk_forward(returns, cfg, folds) -> WalkForwardResult` function with `Split` and `WalkForwardResult` types used by the multi-period engine.

Both are actively used by different callers. This is not dead code, but the two codepaths serve overlapping use cases (parameter sweep over folds) with incompatible APIs.

**Recommendation:** No immediate action, but document the boundary. If the standalone module is ever extended, consider whether it should delegate to the engine-level function rather than re-implementing fold logic.

---

### 7. `presets.py` vs `signal_presets.py` — Two parallel preset systems

`signal_presets.py` (144 L) defines three hardcoded `TrendSpecPreset` objects (Conservative, Balanced, Aggressive) with lookup helpers used by both the Click CLI and the Streamlit UI.

`presets.py` (426 L) provides a richer YAML-driven `TrendPreset` registry with metric pipeline configuration, UI aliases, and form defaults — also used by both CLI and UI.

The two systems coexist without cross-referencing each other. Neither wraps the other.

**Recommendation:** Determine which should be the canonical source for TrendSpec presets at the CLI/UI layer. If `presets.py` is the long-term target, migrate the hardcoded three from `signal_presets.py` into YAML config files and deprecate `signal_presets.py`. If both need to coexist, add a comment in each explaining the boundary.

---

## No Issues (clean files)

- `weighting.py` — clean, stateful strategy hierarchy well-designed
- `diagnostics.py` — clean, clear separation from `trend.diagnostics`
- `schedules.py` — clean, distinct responsibility from `rebalancing.py`
- `signals.py` — clean, excellent single responsibility
- `time_utils.py` — clean, single export
- `universe.py` — clean, efficient numpy-backed membership computation
- `risk.py` — clean, stateless, well-composed
- `regimes.py` — clean, two detection methods well-encapsulated
- `tool_layer.py` — isolated to `api_server/`, coherent
- `api.py` — complex but coherent; complexity is inherent to the integration surface
- `pipeline_helpers.py` — large but coherent; signal utilities placement slightly odd but acceptable

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| High | Fix double assignment of `risk_free_column` | `pipeline_entrypoints.py:232–235` |
| Medium | Extract shared config extraction into `_extract_bindings()` | `pipeline_entrypoints.py` |
| Medium | Extract `_load_file()` helper to remove `load_csv`/`load_parquet` duplication | `data.py` |
| Low | Determine canonical preset system; deprecate or document the boundary | `presets.py`, `signal_presets.py` |
| Low | Document the boundary between the two walk-forward implementations | `walk_forward.py`, `engine/walkforward.py` |
| Low | Mark `_run_analysis()` explicitly as a compat wrapper | `pipeline_runner.py` |
| Info | Document `_sync_stage_dependencies()` monkeypatching rationale | `pipeline.py` |
