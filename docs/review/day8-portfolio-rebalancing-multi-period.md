# Day 8 Review: `portfolio/`, `rebalancing/`, and `multi_period/` subpackages

**Scope:** 9 files, ~5,090 lines — weight-policy cleanup, rebalancing strategies, and the multi-period back-tester
**Date:** 2026-04-28
**Reviewer:** Claude

---

## Files Reviewed

- `portfolio/__init__.py` — Re-exports `apply_weight_policy`. Nothing else lives here.
- `portfolio/weight_policy.py` — `apply_weight_policy` with three modes: `drop` (remove invalid assets and renormalise), `carry` (fill from previous weights then renormalise), and `cash` (zero-out and leave unnormalised so the residual is an implicit cash buffer). Used both by `stages/portfolio.py` (single-period) and the multi-period engine on every rebalance.
- `multi_period/__init__.py` — Re-exports `Portfolio`, `run`, `run_from_config`, `run_schedule`. Clean.
- `rebalancing/__init__.py` — Re-exports the five strategy classes plus the registry helpers from `strategies.py`. Clean.
- `multi_period/scheduler.py` — `generate_periods` builds a list of `PeriodTuple` (in-sample start/end, out-of-sample start/end) for a given frequency and date range, respecting partial final windows. Drives the period loop in `engine.run()`.
- `multi_period/loaders.py` — `load_prices`, `load_membership`, `load_benchmarks`, plus a heuristic `detect_index_columns`. Handles path resolution, validation, and UTC coercion; called from `run_from_config`.
- `multi_period/replacer.py` — `Rebalancer` class: per-period z-score-based fund entry/exit logic with soft and hard thresholds, cooldown tracking via `_entry_strikes` / `_strikes`, and a random-selection mode. See issue below about the docstring.
- `rebalancing/strategies.py` — Five `Rebalancer` subclasses registered by name: `TurnoverCapStrategy`, `PeriodicRebalanceStrategy`, `DriftBandStrategy`, `VolTargetRebalanceStrategy`, `DrawdownGuardStrategy`. They all share a `_apply_cash_policy` helper and a `apply_rebalancing_strategies` chaining function.
- `multi_period/engine.py` — The multi-period back-tester (3,898 lines). Exposes `run`, `run_from_config`, `run_schedule`, and the `Portfolio` dataclass. This is where most of the issues live.

---

## Issues Found

### 1. `run()` in `engine.py` is two algorithms wedged into one 3,150-line function

`run()` spans lines 736–3,885 and branches at line 1038 on `cfg.portfolio.get("policy") == "threshold_hold"`. The two branches have no interaction after the split:

- **Standard path** (lines 1039–1223, ~185 lines): one call to `_call_pipeline_with_diag` per period, no persistent selection state.
- **Threshold-hold path** (lines 1225–3,885, ~2,660 lines): full selection loop with z-score entry/exit, sticky-add/sticky-drop rules, cooldown tracking, min/max fund enforcement, manager-change logs, tenure tracking, regime-aware turnover caps, and weight-bounds enforcement.

The threshold-hold branch defines 19 nested helpers (the count from `^    def ` in that range), several of which close over mutable loop state — `cooldown_book`, `holdings_tenure`, `low_weight_strikes`, `add_streaks`, `drop_streaks`. Because they're closures rather than methods, the shared state is invisible from the function signatures and the mutation is implicit at every call site.

**Recommendation:** Extract the threshold-hold path into its own internal function (e.g. `_run_threshold_hold(df, cfg, ...)`) so `run()` becomes a short router that picks one of two paths. Once that's done, the nested closures become natural candidates to move to module-level helpers, or to methods on a small selection-state class. This is the highest-value structural change in these packages.

---

### 2. `_min_tenure_protected` and `_min_tenure_guard` are the same function

Both are nested closures inside the threshold-hold path, defined nine lines apart at lines 1714 and 1724. The bodies are character-for-character identical: iterate over `holdings`, check `holdings_tenure`, build a `protected` set, return. The only difference is that `_min_tenure_protected` takes a second `score_frame` parameter that it never uses.

```python
# line 1714
def _min_tenure_protected(holdings, score_frame) -> set[str]:
    if min_tenure_n <= 0:
        return set()
    protected: set[str] = set()
    for mgr in holdings:
        ...

# line 1724
def _min_tenure_guard(holdings) -> set[str]:
    if min_tenure_n <= 0:
        return set()
    protected: set[str] = set()
    for mgr in holdings:
        ...   # same body
```

Call sites: `_min_tenure_protected` is used at lines 2580 and 3264 (always with `sf` as the second argument); `_min_tenure_guard` is used at line 3373.

**Recommendation:** Delete `_min_tenure_protected` and update both call sites to drop the unused `sf` argument and call `_min_tenure_guard` instead.

---

### 3. Turnover computation is implemented twice in the same module

`_compute_turnover_state` (module level, lines 282–324) computes one-sided turnover between two weight vectors via pandas alignment. `run_schedule` then defines a nested `_fast_turnover` closure (lines 576–630) that does the same computation via manually-built Python dicts, with a docstring describing it as "vectorised using NumPy".

Inside the same `run_schedule` loop, `_fast_turnover` is used in the if-branch (lines 682–684) and `_compute_turnover_state` is used in the else-branch (line 691). Both produce the same result by construction. Both have direct test coverage: `tests/test_multi_period_engine.py`, `test_turnover_vectorization.py`, and others exercise `_compute_turnover_state` directly; `test_multi_period_engine_turnover_regression.py` and friends exercise `_fast_turnover` end-to-end through `run_schedule`.

The two implementations are not algorithmically equivalent in performance terms — `_fast_turnover` builds Python dicts and iterates in Python; `_compute_turnover_state` does pandas reindex-into-numpy. For realistic universe sizes the pandas path is likely the faster of the two, despite the closure's docstring framing it as the speedup.

**Recommendation:** Benchmark both at typical universe sizes (50–500 columns). Whichever wins, delete the loser and use the survivor in both branches. The duplication means any correctness fix has to be applied twice and gets caught by different test suites.

---

### 4. `apply_missing_policy` is called twice per run, with the first call's result discarded

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

The return value is dropped on the floor. The comment above it says "Legacy behavior: call apply_missing_policy in this module so tests can monkeypatch it for observability." The actual data-cleaning call that produces the `filled` frame used downstream is ~60 lines later (lines 930–944), where the result is captured in `filled, _missing_result = ...`.

This pattern means `apply_missing_policy` runs twice on every non-skip path: once as a no-op tracer, once for real. The tests in `test_multi_period_engine_branch_new.py`, `test_multi_period_engine_keepalive.py`, and `test_multi_period_engine_missing_policy.py` all monkeypatch the module-level binding to capture arguments. Because both calls hit the patched function and the second-call captures overwrite the first, the first call is genuinely vestigial — removing it would not break any of the existing assertions about `policy` / `limit` capture.

**Recommendation:** Remove the phantom first call. The real call at line 930 already invokes the module-level `apply_missing_policy`, which is what tests monkeypatch. Add a one-line comment at the import (line 59) or at the real call site explaining that the module-level binding exists for monkeypatching.

---

### 5. The threshold-hold branch calls `cfg.model_dump()` four times in a row

Lines 1226, 1235, 1530, and 1671 each invoke `cfg.model_dump()` to get a dict snapshot of the pydantic config:

```python
periods = generate_periods(cfg.model_dump())                              # 1226
mp_cfg = cast(dict[str, Any], cfg.model_dump().get("multi_period", {}))   # 1235
...
mp_cfg = cast(dict[str, Any], cfg.model_dump().get("multi_period", {}))   # 1530 — same as 1235
...
rebalancer = Rebalancer(cfg.model_dump())                                  # 1671
```

`model_dump()` walks the entire pydantic model and produces a fresh dict each time. The cost is small in absolute terms but the duplication is unnecessary — and the line-1530 reassignment to `mp_cfg` overwrites the line-1235 value with what should be the same content.

**Recommendation:** Compute `cfg_dump = cfg.model_dump()` once at the top of the threshold-hold branch and reuse it for `generate_periods`, the two `mp_cfg` extractions, and the `Rebalancer` constructor. This also makes it obvious that the branch reads from a single config snapshot.

---

### 6. `multi_period/replacer.py` has a stale module docstring

The first lines of the module say:

> *Phase-2 placeholder: echoes the incoming weights so the rest of the pipeline keeps running. Real strike / replacement logic comes later.*

The class now has a ~200-line implementation covering soft/hard z-score exits, soft entry with `entry_soft_strikes` accumulation, capacity-limited additions in priority order (hard > auto > eligible), random-selection mode, and Bayesian score-proportional weighting. "Echoes the incoming weights" hasn't been true for a while.

**Recommendation:** Rewrite the docstring to describe what `Rebalancer` actually does, particularly the entry/exit thresholds, the strike-counting state, and the random-selection branch.

---

## Notes (not actionable)

### `scheduler.py::FREQ_MAP` has 17 entries for three underlying aliases

`FREQ_MAP` maps `"M"`, `"ME"`, `"monthly"`, `"MONTHLY"`, `"Monthly"` (and similar variants for quarterly and annual) all to the same canonical targets. This works and is exhaustive, but a case-normalised lookup against three canonical strings would be smaller and easier to extend if anyone ever adds new user-friendly names.

### `rebalancing/strategies.py::RebalancingStrategy` is a backwards-compatibility alias

Line 21 has `RebalancingStrategy = Rebalancer`. It's in `__all__`, re-exported from `rebalancing/__init__.py`, and listed in that package's `__all__` too. Anyone reading the strategies module sees two names for one thing. Low risk, worth knowing about.

### `multi_period/engine.py` re-exposes a `_run_analysis` shim for test monkeypatching

Lines 87–88 define a module-level `_run_analysis` that delegates to `_invoke_analysis_with_diag`. This mirrors the pattern in `stages/portfolio.py::avg_corr_handler` (Day 6): a named module-level attribute that tests can replace via monkeypatching. The comment at line 84 calls this out clearly. Not a bug, just the same known pattern.

### The covariance-diagnostic block inside `run()` accumulates complexity over time

Lines 1127–1221 attach an experimental `cov_diag` key to each period result when `enable_cache` is true. This includes incremental covariance update logic (shift-detection by trailing-block compare, sequential row swaps) that is gated by `performance.incremental_cov`. The feature appears sound but the condition nesting is deep and the imports of `compute_cov_payload` / `incremental_cov_update` are duplicated inside both branches of the inner `if`. If this stops being experimental, it's a good candidate for extraction into its own helper.

---

## No Issues (clean files)

- `portfolio/__init__.py` — clean
- `portfolio/weight_policy.py` — clean, well-scoped, the three modes are documented and exercised
- `multi_period/__init__.py` — clean
- `rebalancing/__init__.py` — clean
- `multi_period/scheduler.py` — clean; the `FREQ_MAP` verbosity is a style preference, not a bug
- `multi_period/loaders.py` — clean; error messages are clear and the benchmark fallback warns rather than raises
- `rebalancing/strategies.py` — clean, well-structured; the five strategy classes follow a consistent interface

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| Medium | Extract the threshold-hold branch of `run()` into its own function; promote nested closures to module-level helpers or a state class | `multi_period/engine.py` |
| Low | Delete `_min_tenure_protected`; replace its two call sites with `_min_tenure_guard` | `multi_period/engine.py` |
| Low | Pick one turnover implementation, delete the other | `multi_period/engine.py` |
| Low | Remove the phantom `apply_missing_policy` call and document the monkeypatching pattern at the real call site | `multi_period/engine.py` |
| Low | Compute `cfg.model_dump()` once at the top of the threshold-hold branch and reuse it | `multi_period/engine.py` |
| Low | Rewrite the stale module docstring in `replacer.py` | `multi_period/replacer.py` |
