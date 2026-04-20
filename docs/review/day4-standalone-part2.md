# Day 4 Review: src/trend_analysis/ root standalone files (Part 2)

**Scope:** Remaining 19 files by line count (172–773 lines each), total ~6,800 lines
**Date:** 2026-04-20 (revised)
**Reviewer:** Claude

> **Note:** This document replaces a previous draft that contained several factual errors (misidentified test patch targets, incorrect claim of a Click CLI, overstated severity of a redundant assignment). All claims below have been verified against the source.

---

## Files Reviewed

- `weighting.py` — Defines `BaseWeighting` ABC and four concrete strategies: `EqualWeight`, `ScorePropSimple`, `ScorePropBayesian`, `AdaptiveBayesWeighting`. The adaptive strategy is stateful with exponential decay and `get_state`/`set_state` for cross-run serialization.
- `diagnostics.py` — Re-exports `DiagnosticPayload`, `DiagnosticResult`, `RunPayload`, `RunPayloadResult` from `trend.diagnostics` and adds `PipelineResult` (dict-like dataclass), `PipelineReasonCode` enum (9 codes), and `pipeline_success` / `pipeline_failure` / `coerce_pipeline_result` helpers. Used by `pipeline_runner.py` to wrap execution results.
- `schedules.py` — Provides `get_rebalance_dates`, `normalize_positions`, and `apply_rebalance_schedule`. Handles frequency string aliases (`"monthly"`, `"weekly"`), timezone alignment, and custom rebalance calendars. Distinct from the `rebalancing.py` compat shim.
- `signals.py` — Core signal computation: `TrendSpec` frozen dataclass and `compute_trend_signals()`. Vectorized, strictly causal rolling signal with memoization via a `_FrameHandle` sentinel. Debug timing via the `_timed_stage` context manager.
- `time_utils.py` — `align_calendar()` aligns a DataFrame's Date column to a business-day/weekly/monthly calendar with US holiday generation (`_simple_us_holidays`), timezone handling, and frequency inference. Exports only `align_calendar`.
- `logging.py` — JSONL structured run logger (separate from `logging_setup.py`): `JsonlHandler`, `init_run_logger`, `log_step`, `iter_jsonl`, `latest_errors`, `logfile_to_frame`, `error_summary`. Includes a simple single-backup rotation strategy.
- `pipeline_runner.py` — Contains `_run_analysis_with_diagnostics()` (the real implementation) and `_run_analysis()` (its bare-payload wrapper, used as a test monkeypatch target — see issue below).
- `pipeline_entrypoints.py` — `ConfigBindings` dataclass plus `run_from_config()` and `run_full_from_config()`. Bridges Pydantic config objects to `pipeline.run()` / `pipeline.run_full()`. See issues below.
- `__init__.py` — Package bootstrap: dataclass module-guard patch, optional `MPLCONFIGDIR` configuration, `_SpecProxy` for spec resilience, eager + lazy submodule registration, version discovery via `importlib.metadata`. Also holds the authoritative `__all__`.
- `universe.py` — `MembershipWindow`, `MembershipTable`, `load_universe_membership`, `apply_membership_windows`, `build_membership_mask`, `gate_universe`. `_expand_active_pairs` uses numpy broadcasting for efficient date × membership activity computation.
- `risk.py` — Stateless `compute_constrained_weights`, `realised_volatility`. Handles vol targeting, turnover caps, EWMA vs simple rolling vol, group caps, max active positions. Delegates constraint optimisation to `engine.optimizer`.
- `tool_layer.py` — `ToolLayer` dataclass with `apply_patch`, `validate_config`, `preview_diff`, `run_analysis`. Includes rate limiting, path sandboxing, and JSONL call logging. Only imported by `src/trend_analysis/api_server/__init__.py`.
- `walk_forward.py` — Standalone YAML-config-driven parameter sweep that writes CSV/JSONL/PNG output. Imported by `scripts/walk_forward.py` (only). See issue below.
- `presets.py` — YAML-driven preset registry (`lru_cache`-backed) that loads `TrendPreset` objects from `config/presets/*.yml`. Provides `apply_trend_preset`, `get_trend_preset`, `list_preset_slugs`, plus metric pipeline configuration and UI aliases.
- `pipeline.py` — Integration hub: imports from stages, `pipeline_runner`, `pipeline_helpers`, wires them together. Exports `run()`, `run_full()`, `run_analysis()`, `compute_signal()`, `position_from_signal()`. Contains the `_sync_stage_dependencies()` monkeypatch coordinator — see note below.
- `regimes.py` — `RegimeSettings`, `compute_regimes`, `aggregate_performance_by_regime`, `build_regime_payload`. Two detection methods: `rolling_return` and `volatility`. Integrates with the perf cache via `_compute_regime_series`.
- `data.py` — `load_csv`, `load_parquet`, `validate_dataframe`, `identify_risk_free_fund`, `ensure_datetime`, `compute_inception_dates`. Central data ingestion layer. See issue below.
- `api.py` — Main public API: `run_simulation(config, returns) -> RunResult`. `RunResult` wraps metrics, weights, exposures, turnover, costs, portfolio series, and multi-period period results. Detects `config.multi_period` and delegates to `_run_multi_period_simulation()`.
- `pipeline_helpers.py` — Config-resolution helpers (`_cfg_value`, `_cfg_section`, `_section_get`, `_resolve_sample_split`, `_derive_split_from_periods`, `_policy_from_config`, `_build_trend_spec`, `_attach_calendar_settings`), regime override application, turnover cap parsing, plus `compute_signal()` and `position_from_signal()`.

---

## Issues Found

### 1. `pipeline_entrypoints.py` — Redundant duplicate assignment of `risk_free_column`

In `run_full_from_config`, lines 232 and 235 assign `risk_free_column` with **identical** right-hand sides:

```python
# line 232
risk_free_column = bindings.section_get(data_settings, "risk_free_column")
# line 235 (identical)
risk_free_column = bindings.section_get(data_settings, "risk_free_column")
```

This is not a behavioural bug (both RHS evaluate to the same value), just redundant code — one line is a dead statement.

**Recommendation:** Delete line 232. (Low-risk, no behavioural change.)

---

### 2. `pipeline_entrypoints.py` — Inconsistent risk-free resolution between the two entry points

`run_from_config` (line 102) resolves the risk-free column via the dedicated helper:

```python
risk_free_column, allow_risk_free_fallback = resolve_risk_free_settings(
    data_settings if isinstance(data_settings, Mapping) else None
)
```

`run_full_from_config` (lines 235–236) reads the same fields directly via `section_get`:

```python
risk_free_column = bindings.section_get(data_settings, "risk_free_column")
allow_risk_free_fallback = bindings.section_get(data_settings, "allow_risk_free_fallback")
```

The `resolve_risk_free_settings` helper handles non-Mapping inputs and may apply defaults the direct reads miss. Two entry points resolving the same setting differently is a latent divergence source.

**Recommendation:** Make `run_full_from_config` use `resolve_risk_free_settings` like its sibling.

---

### 3. `pipeline_entrypoints.py` — ~50 lines duplicated between `run_from_config` and `run_full_from_config`

Lines 46–100 of `run_from_config` and lines 178–231 of `run_full_from_config` perform the same sequence: unwrap cfg, read sections, load CSV, attach calendar settings, resolve sample split, build stats cfg, resolve policy, build trend spec, read portfolio/run/vol_adjust sections, compute weight engine params. The two functions only diverge after `invoke_analysis_with_diag` is called.

**Recommendation:** Extract the shared setup into a private helper returning a resolved-params dataclass. See prior conversation for a concrete sketch.

---

### 4. `data.py` — `load_csv` and `load_parquet` share ~50 lines of near-identical structure

Both functions (lines 383–460 and 463–530) handle: legacy kwargs coercion, `Path` construction + existence/directory/permission checks, `_validate_payload` call, reset-index, and a matching `except`/logging block. The only real differences:

- `pd.read_csv` vs `pd.read_parquet`
- `load_csv` catches `pd.errors.ParserError` (parquet has no equivalent)
- `load_csv` logs permission errors in non-raise mode; `load_parquet` always raises

**Recommendation:** Extract a private `_load_file(read_fn, path, extra_except=(), ...)` helper that both delegate to. Keeps the small behaviour differences explicit via arguments.

---

### 5. `walk_forward.py` vs `engine/walkforward.py` — Two walk-forward implementations with different scopes

Both modules exist and are actively used:

- `trend_analysis.walk_forward.run_from_config` — YAML-driven orchestration (reads config, grid search, writes CSV/JSONL/PNG). Imported by `scripts/walk_forward.py`.
- `trend_analysis.engine.walkforward.walk_forward` — Lower-level `walk_forward(returns, cfg, folds) -> WalkForwardResult` function used by `scripts/walkforward_cli.py` and `tests/test_walkforward_engine.py`.

This is **not duplication** per se — the scopes differ (orchestration vs. computation). But the naming (`walk_forward.py` vs `walkforward.py`) and the lack of a delegation boundary (the orchestration module re-implements fold iteration rather than delegating to the engine) invite confusion.

**Recommendation:** No urgent action. Add a module docstring to each making the boundary explicit, and consider whether future additions to the orchestration module should delegate to `engine.walkforward.walk_forward`.

---

### 6. `presets.py` and `signal_presets.py` — Two parallel-but-complementary preset systems

Both are imported by both CLIs (`src/trend_analysis/cli.py:59,62` and `src/trend/cli.py:74,80`):

- `signal_presets.py::TrendSpecPreset` — Three **hardcoded** presets (Conservative, Balanced, Aggressive), pure TrendSpec container.
- `presets.py::TrendPreset` — YAML-driven, richer (includes metric pipeline, UI aliases, form defaults, vol-adjust defaults).

They do not collide — CLIs use them side-by-side for different purposes. But `TrendPreset` is a proper superset of `TrendSpecPreset` in capability; the hardcoded three could be migrated into YAML config files and `signal_presets.py` retired.

**Recommendation:** No urgent action. Longer-term, consolidate onto `presets.py` by promoting the three hardcoded presets to YAML and deprecating `signal_presets.py`.

---

## Notes (not actionable issues)

### `pipeline_runner._run_analysis` is a deliberate test patch target

`_run_analysis()` in `pipeline_runner.py` is a one-line bare-payload wrapper around `_run_analysis_with_diagnostics()`. It looks trivial, but it is imported into `pipeline.py` (line 51) and re-wrapped there as `pipeline._run_analysis`, which `_invoke_analysis_with_diag` (line 205) specifically checks for monkeypatching (`if _run_analysis is _DEFAULT_RUN_ANALYSIS:`). Tests and legacy callers patch `pipeline._run_analysis` to inject raw dict payloads; the wrapper is the documented contract for that. **Do not delete.**

### `pipeline.py::_sync_stage_dependencies()` uses `setattr` monkeypatching

Lines 103–134 apply `setattr` to `preprocessing_stage`, `selection_stage`, `portfolio_stage` on every `_call_with_sync` invocation. This is intentional — it ensures monkeypatches applied to `pipeline.*` symbols propagate into stage modules during tests. The docstring documents the rationale. This is unusual but functional; worth noting for future refactors toward explicit dependency injection.

---

## No Issues (clean files)

- `weighting.py` — clean strategy hierarchy, stateful adaptive case well-designed
- `diagnostics.py` — clean, clear delegation to `trend.diagnostics`
- `schedules.py` — clean, distinct from `rebalancing.py`
- `signals.py` — clean, single responsibility
- `time_utils.py` — clean, single export
- `logging.py` — clean JSONL logger, single responsibility
- `__init__.py` — complex but necessarily so (dataclass patch, lazy submodules, matplotlib config)
- `universe.py` — clean, efficient numpy-backed membership computation
- `risk.py` — clean, stateless, well-composed
- `tool_layer.py` — isolated to `api_server/`, coherent
- `regimes.py` — clean, two detection methods well-encapsulated
- `api.py` — complex but coherent; complexity inherent to the integration surface
- `pipeline_helpers.py` — large but coherent

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| Low | Delete redundant duplicate assignment (line 232) | `pipeline_entrypoints.py` |
| Medium | Use `resolve_risk_free_settings` in `run_full_from_config` to match `run_from_config` | `pipeline_entrypoints.py` |
| Medium | Extract shared config-resolution block into a private helper | `pipeline_entrypoints.py` |
| Medium | Extract `_load_file()` helper to collapse `load_csv`/`load_parquet` boilerplate | `data.py` |
| Low | Document the boundary between the two walk-forward modules | `walk_forward.py`, `engine/walkforward.py` |
| Low | Longer-term: consolidate onto `presets.py`, retire `signal_presets.py` | `presets.py`, `signal_presets.py` |

---

## Corrections to the Prior Draft

For reviewers comparing against the superseded draft:

1. **Withdrawn:** "`_run_analysis()` is a trivial compat wrapper — consider removing it." It is a documented test patch target; removing it would break the pipeline's monkeypatch contract.
2. **Corrected:** Previous text said "Click CLI." The CLIs in this repo use `argparse`, not Click. No Click import exists anywhere under `src/`.
3. **Downgraded:** The `risk_free_column` double-assignment was described as a "copy-paste bug." Both assignments have identical RHS, so behaviour is unchanged — it's a redundant/dead statement, not a silent-corruption bug. Severity lowered from High to Low. A *separate* real issue (divergence in risk-free resolution between the two entry points) has been added as item 2.
4. **Added:** `logging.py` and `__init__.py` to Files Reviewed (omitted from the prior draft — they belong in Day 4 by line-count ordering, and `logging.py` was incorrectly listed in Day 3's "clean files" section).
