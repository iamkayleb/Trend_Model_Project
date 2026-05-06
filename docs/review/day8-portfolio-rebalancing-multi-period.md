# Day 8 Review: `portfolio/`, `rebalancing/`, and `multi_period/` subpackages

**Scope:** 9 files, ~5,090 lines — portfolio weight policy, rebalancing strategies, and the multi-period back-tester
**Date:** 2026-04-28
**Reviewer:** Claude

---

## Files Reviewed

- `portfolio/__init__.py` — Re-exports `apply_weight_policy`. Nothing else.
- `portfolio/weight_policy.py` — `apply_weight_policy` handles three modes for cleaning weights when signals are invalid or absent: `drop` (remove and renormalise), `carry` (fill from previous weights then renormalise), `cash` (zero out and leave unnormalised so the residual is an implicit cash buffer). Used by `stages/portfolio.py` and the multi-period engine on every rebalance.
- `multi_period/__init__.py` — Re-exports `Portfolio`, `run`, `run_from_config`, `run_schedule`. Clean.
- `rebalancing/__init__.py` — Re-exports all five strategy classes plus registry utilities from `strategies.py`. Clean.
- `multi_period/scheduler.py` — `generate_periods` builds a list of `PeriodTuple` (in-sample start/end, out-of-sample start/end) for a given frequency and date range, respecting partial final windows. Used by `multi_period/engine.py::run()`.
- `multi_period/loaders.py` — `load_prices`, `load_membership`, `load_benchmarks`, `detect_index_columns`. Handles path resolution, validation, and UTC coercion; used by `run_from_config`.
- `multi_period/replacer.py` — `Rebalancer` class: per-period z-score based fund entry/exit logic with soft and hard thresholds, cooldown tracking via `_entry_strikes` / `_strikes`, and a random-selection mode. See issue below.
- `rebalancing/strategies.py` — Five `Rebalancer` subclasses registered by name: `TurnoverCapStrategy` (prioritised trade execution within a turnover budget), `PeriodicRebalanceStrategy` (every-N-periods rebalance), `DriftBandStrategy` (rebalance only assets that have drifted beyond a band), `VolTargetRebalanceStrategy` (scale to hit a target volatility via leverage), `DrawdownGuardStrategy` (scale back exposure when a drawdown threshold is breached). All share the same `_apply_cash_policy` helper and `apply_rebalancing_strategies` chaining function.
- `multi_period/engine.py` — The multi-period back-tester (3,898 lines). Exposes `run`, `run_from_config`, `run_schedule`, and `Portfolio`. See issues below.

---

## Issues Found

### 1. `run()` in `engine.py` is two separate algorithms in one 3,800-line function

The `run()` function (lines 736–3,885) branches at line 1038 on `cfg.portfolio.get("policy") == "threshold_hold"`. The two branches share only data loading and config parsing; after the split they have no interaction:

- **Standard path** (~185 lines, 1039–1223): calls `_call_pipeline_with_diag` once per period, no persistent selection state.
- **Threshold-hold path** (~2,660 lines, 1225–3,885): full selection loop with z-score entry/exit, sticky rules, cooldown tracking, min/max fund enforcement, intra-period rebalancing, manager change logs, tenure tracking, regime-based turnover caps, and weight-bounds enforcement.

The threshold-hold path also defines roughly 20 nested functions that close over mutable loop state (`cooldown_book`, `holdings_tenure`, `low_weight_strikes`, `add_streaks`, `drop_streaks`). Because they're closures rather than methods, the shared state is invisible from the function signatures and the mutation side-effects are implicit.

**Recommendation:** Extract the threshold-hold path into a separate internal function (e.g. `_run_threshold_hold(df, cfg, ...)`) so that `run()` becomes a short router that calls either path. The nested closures would then naturally become module-level helpers or methods of a small selection-state class. This is the highest-priority structural issue in these packages.

---

### 2. `_min_tenure_protected` and `_min_tenure_guard` are identical

Both closures are defined inside the threshold-hold path (lines 1714 and 1724). They have different names and slightly different signatures — `_min_tenure_protected` takes a `score_frame` parameter that it never uses — but the body of each is character-for-character the same: iterate over `holdings`, check `holdings_tenure`, add to a `protected` set, return.

```python
# _min_tenure_protected (line 1714) — score_frame unused
def _min_tenure_protected(holdings, score_frame) -> set[str]:
    ...  # identical body

# _min_tenure_guard (line 1724) — no score_frame
def _min_tenure_guard(holdings) -> set[str]:
    ...  # identical body
```

**Recommendation:** Delete `_min_tenure_protected`. All call sites should use `_min_tenure_guard`.

---

### 3. Turnover computation is implemented twice in `run_schedule`

`engine.py` has a module-level `_compute_turnover_state` (lines 282–324) that computes one-sided turnover between two weight vectors via pandas alignment. `run_schedule` (lines 524–733) defines a nested `_fast_turnover` closure (lines 576–630) that does the same computation via manually-built Python dicts, described in its docstring as a faster alternative.

`run_schedule` then uses `_fast_turnover` in the if-branch (lines 682–684) and `_compute_turnover_state` in the else-branch (lines 691), within the same loop.

The two implementations produce the same result by construction. The performance difference is real but both paths already exist in the same module, and the duplication means any correctness fix must be applied to both.

**Recommendation:** Benchmark whether the dict-based approach is materially faster for realistic universe sizes. If it is, replace `_compute_turnover_state` with the faster version and delete the older one. If it isn't, delete `_fast_turnover` and use `_compute_turnover_state` in both branches.

---

### 4. `apply_missing_policy` is called for observability only, with the result discarded

`run()` lines 871–878:

```python
try:
    apply_missing_policy(
        df.set_index("Date"),
        policy=policy_spec,
        limit=missing_limit_cfg,
    )
except Exception:  # pragma: no cover - best-effort only
    pass
```

The return value is silently discarded. The comment says "Legacy behavior: call apply_missing_policy in this module so tests can monkeypatch it for observability." The actual data-cleaning call that produces the `filled` frame used downstream is ~60 lines later (lines 930–944).

This pattern means `apply_missing_policy` is called twice per run in the non-skip path: once as a no-op tracer, once for real. It also means the first call's `TypeError` fallback (lines 937–944) exists because some test patches implement a simplified signature. The real call already has that fallback, so the test-observability call could be replaced with a single `# tests monkeypatch apply_missing_policy` comment and an unconditional call to the real path.

**Recommendation:** Remove the phantom first call and consolidate into the real data-cleaning call at line 930. Document in that call's comment why `apply_missing_policy` is imported at module scope (for test monkeypatching).

---

### 5. `multi_period/replacer.py` has a stale module docstring

The docstring on the module says:

> *Phase-2 placeholder: echoes the incoming weights so the rest of the pipeline keeps running. Real strike / replacement logic comes later.*

The class now has a ~200-line implementation covering hard/soft z-score exits, soft entry with `entry_soft_strikes`, capacity-limited additions, random-selection mode, and Bayesian weighting. "Placeholder" no longer applies.

**Recommendation:** Rewrite the module docstring to describe what the class actually does.

---

## Notes (not actionable)

### `scheduler.py::FREQ_MAP` has 17 entries for three underlying aliases

`FREQ_MAP` maps `"M"`, `"ME"`, `"monthly"`, `"MONTHLY"`, `"Monthly"` (and similar variants for quarterly and annual) all to the same targets. This works and is exhaustive, but a case-normalised lookup against three canonical strings would be easier to maintain if new user-friendly names are ever added.

### `rebalancing/strategies.py::RebalancingStrategy` is a backwards-compatibility alias

Line 21: `RebalancingStrategy = Rebalancer`. It's in `__all__`, re-exported from `rebalancing/__init__.py`, and appears in `__init__.py`'s `__all__` as well. Callers using `RebalancingStrategy` will get the same object. Low risk, worth knowing.

### `multi_period/engine.py` re-exposes a `_run_analysis` shim for test monkeypatching

Lines 87–88 define a module-level `_run_analysis` that delegates to `_invoke_analysis_with_diag`. This mirrors the same pattern used in `stages/portfolio.py::avg_corr_handler` (Day 6): a named module-level attribute that tests can replace via monkeypatching. The comment at line 84 explains it clearly. Not a bug, just the same known pattern.

### The covariance-diag block inside `run()` accumulates complexity over time

Lines 1127–1221 attach an experimental `cov_diag` key to each period result when `enable_cache` is true. This includes incremental covariance update logic (shift-detection, sequential row swaps) that is controlled by an `os.getenv("DEBUG_TURNOVER_VALIDATE")` style flag (`performance.incremental_cov`). The feature appears sound but the condition nesting is deep. If it ever becomes non-experimental, it would benefit from extraction into its own helper.

---

## No Issues (clean files)

- `portfolio/__init__.py` — clean
- `portfolio/weight_policy.py` — clean, well-scoped
- `multi_period/__init__.py` — clean
- `rebalancing/__init__.py` — clean
- `multi_period/scheduler.py` — clean; `FREQ_MAP` verbosity is a style preference, not a bug
- `multi_period/loaders.py` — clean; error messages are clear, fallback detection is explicit
- `rebalancing/strategies.py` — clean, well-structured; all five strategy classes follow the same interface

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| Medium | Extract the threshold-hold branch of `run()` into its own function; promote nested closures to module-level helpers | `multi_period/engine.py` |
| Low | Delete `_min_tenure_protected`; replace call sites with `_min_tenure_guard` | `multi_period/engine.py` |
| Low | Consolidate the two turnover-computation implementations into one | `multi_period/engine.py` |
| Low | Remove the phantom `apply_missing_policy` call and document the monkeypatching pattern at the real call site | `multi_period/engine.py` |
| Low | Update the stale module docstring in `replacer.py` | `multi_period/replacer.py` |
