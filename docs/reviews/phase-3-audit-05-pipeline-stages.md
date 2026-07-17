# Phase-3 First-Pass Audit 05: Pipeline & Stages

**Date:** 2026-07-16
**Reviewer:** Project review (first pass — current-state audit, no change-provenance claims)
**Branch:** `phase-3`
**Scope:** `pipeline.py`, `pipeline_entrypoints.py`, `pipeline_runner.py`, `pipeline_helpers.py`, `stages/{preprocessing,selection,portfolio}.py`, `core/{rank_selection,metric_cache}.py`

> First-pass review of the execution core (~3,900 lines). Findings are about the code as it stands on phase-3, judged on merits.

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `pipeline.py` | 456 | Public API facade; late-binds stage impls via `ConfigBindings` |
| `pipeline_entrypoints.py` | 321 | `run_from_config` / `run_full_from_config` wiring |
| `pipeline_runner.py` | 247 | `_run_analysis(_with_diagnostics)` orchestration |
| `pipeline_helpers.py` | 773 | Config parsing, regime overrides, `compute_signal`, sample-split derivation |
| `stages/preprocessing.py` | 476 | Input prep + in/out-of-sample window construction |
| `stages/selection.py` | 519 | Universe selection, risk-free resolution, single-period run |
| `stages/portfolio.py` | 870 | Weighting, stats, analysis-output assembly |
| `core/rank_selection.py` | 1125 | Rank/blended selection + window metric caching |
| `core/metric_cache.py` | 117 | Per-metric-series cache |

---

## Summary of Findings

| # | Finding | Severity |
|---|---------|----------|
| 1 | `_build_trend_spec` implemented 3× with different signatures | Medium |
| 2 | `_json_default` copy here (`rank_selection.py:69`) — Day 1 item, still unresolved | Medium |
| 3 | Two metric caches (`MetricCache` + `WindowMetricCache`) | Low (confirm) |
| 4 | `pipeline.py` dynamic `*args/**kwargs` binding shims defeat static typing | Low |
| — | `pipeline.py` is a clean facade, not duplicated logic | ✅ Positive |
| — | Window slicing uses the shared `resolve_period_bound` | ✅ Positive |

---

## Positive / Clean

### `pipeline.py` is a facade, not a duplicate
Despite `compute_signal`/`position_from_signal`/`_run_analysis` etc. appearing both here and in `pipeline_helpers.py`/`pipeline_runner.py`, `pipeline.py` **delegates**: e.g. `compute_signal` (`pipeline.py:374`) just calls `_compute_signal_impl`, imported as `from .pipeline_helpers import compute_signal as _compute_signal_impl` (`pipeline.py:45`). This is a deliberate public-API layer over the real implementations. No logic duplication.

### In/out-of-sample slicing uses the shared boundary resolver
`_build_sample_windows` (`stages/preprocessing.py:238`) resolves each bound through `resolve_period_bound` (via `_resolve_bound`, `:333-346`) and applies inclusive `>=/<=` masks (`:356-361`). This confirms — end to end — the claim in audit-03b: the ordering validator (`config/validation.py`) and the window slicer resolve a label like `"2020-01"` to the **same instant**. Correct and consistent.

---

## Findings

### ① `_build_trend_spec` — three implementations — Medium

The same conceptual operation (build a `TrendSpec` from config) exists three times with divergent signatures and behaviour:

- `pipeline_helpers.py:391` — `_build_trend_spec(cfg, vol_adjust_cfg) -> TrendSpec | None` (returns `None` when no `signals` section, to preserve legacy window-derivation)
- `presets.py:113` — `_build_trend_spec(config) -> TrendSpec` (preset construction)
- `trend_model/spec.py:159` — `_build_trend_spec(cfg) -> TrendSpec` (legacy `trend_model`)

They read overlapping keys (`window`, `lag`, `vol_adjust`, `vol_target`, `zscore`) but differ in defaults, alias handling, and `None`-semantics. Three subtly different parsers for one type invites inconsistency: the same YAML `signals` block can yield different `TrendSpec`s depending on entry point.

**Recommendation:** define one canonical `TrendSpec`-from-mapping builder (e.g. on `signals.TrendSpec` or in `signal_presets`) and have the three sites call it, layering only their distinct `None`/default policies on top.

### ② `_json_default` copy in `rank_selection.py:69` — Medium (Day 1 cross-ref)

Day 1 flagged `_json_default` as 4 copies; one lives here (`core/rank_selection.py:69`). Still unresolved on phase-3 (also in `walk_forward.py`, `backtesting/harness.py`, `trend/cli.py`). Consistent with verification-01.

**Recommendation:** extract once (e.g. `util/json_compat.py`) and import in all four.

### ③ Two metric caches — Low (confirm complementary)

`core/metric_cache.py::MetricCache` (per-metric-series) and `core/rank_selection.py::WindowMetricCache` (per-window metric bundles) coexist; `metric_cache.py` is imported only by `rank_selection.py`. They appear to operate at different granularities (single series vs. a window's full bundle), so this is likely a deliberate two-level cache rather than redundancy.

**Recommendation:** confirm they are complementary and add a one-line comment in each pointing at the other, so a future reader doesn't "consolidate" two caches that serve different layers.

### ④ Dynamic binding shims defeat typing — Low

`pipeline.py` exposes a block of stage entry points as untyped passthroughs, e.g.:
```python
def _prepare_input_data(*args: Any, **kwargs: Any) -> Any:
    ...
```
(`pipeline.py:163-205`). These support the `ConfigBindings` late-binding mechanism (`_sync_stage_dependencies`, `:103`), but they erase parameter/return types at the most-imported module in the package and make the call graph hard to follow statically.

**Recommendation:** where the bound target has a stable signature, give the shim that signature (or `typing.ParamSpec`) instead of `*args/**kwargs: Any`. Not urgent.

---

## Not Deeply Reviewed (flagged)

- `rank_select_funds` (`rank_selection.py:446-706`, ~260 lines) — scanned, no red flags, but the blended-score / transform / inclusion-approach branches warrant a dedicated correctness pass with fixtures.
- `stages/portfolio.py::_compute_weights_and_stats` (`:184-697`, ~500 lines) — the largest single function in the subsystem; not line-audited here.

---

## Verdict for Subsystem E

**No correctness defects found in the paths reviewed**, and two design points are genuinely good: `pipeline.py` is a clean facade, and the sample-window slicing correctly shares `resolve_period_bound` with validation. The issues are structural duplication (`_build_trend_spec` ×3, `_json_default` ×4) rather than bugs.

Highest-value cleanup: consolidate the three `TrendSpec` builders (①). The two large functions (`rank_select_funds`, `_compute_weights_and_stats`) are the main untested-here risk and deserve a follow-up correctness pass.

---

## Files Reviewed

- [x] `src/trend_analysis/pipeline.py` (facade/delegation confirmed)
- [x] `src/trend_analysis/pipeline_helpers.py` (`_build_trend_spec`, `compute_signal`)
- [x] `src/trend_analysis/pipeline_entrypoints.py` (inventory)
- [x] `src/trend_analysis/pipeline_runner.py` (inventory)
- [x] `src/trend_analysis/stages/preprocessing.py` (`_build_sample_windows` — full)
- [x] `src/trend_analysis/stages/selection.py` (inventory)
- [x] `src/trend_analysis/stages/portfolio.py` (inventory)
- [x] `src/trend_analysis/core/rank_selection.py` (`_json_default`, caches, inventory)
- [x] `src/trend_analysis/core/metric_cache.py` (usage check)
- [ ] Deep correctness pass on `rank_select_funds` and `_compute_weights_and_stats` — deferred
