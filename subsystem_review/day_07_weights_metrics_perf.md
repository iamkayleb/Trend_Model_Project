# Day 7 Subsystem Review — Weights, Metrics, Perf

**Date:** 2026-05-05
**Scope:** `src/trend_analysis/weights/`, `src/trend_analysis/metrics/`, `src/trend_analysis/perf/`
**Total:** 14 files, 2,062 lines

This is the second day of the Week 2 audit. Goal for the day was to walk the
weighting strategies, the metrics calculators, and the performance/caching
helpers that the rest of the engine leans on. I went file by file, then ran
`grep` across the source tree to confirm what is actually called and what is
just sitting there.

---

## Files Reviewed

### `weights/` — 5 files, 663 lines

| File | Lines | Purpose |
|---|---|---|
| `__init__.py` | 14 | Re-exports the five weight engines |
| `risk_parity.py` | 50 | Inverse-volatility weighting (`risk_parity`, `vol_inverse`) |
| `equal_risk_contribution.py` | 102 | Iterative ERC solver (`erc`) |
| `hierarchical_risk_parity.py` | 122 | HRP via scipy linkage (`hrp`) |
| `robust_weighting.py` | 390 | `RobustMeanVariance` + `RobustRiskParity` with shrinkage and safe-mode fallback |
| `robust_config.py` | 81 | Translates the robustness config block into engine kwargs |

### `metrics/` — 5 files, 760 lines

| File | Lines | Purpose |
|---|---|---|
| `__init__.py` | 439 | Core metric library + registry + legacy compatibility shims |
| `attribution.py` | 134 | Per-signal contribution table, CSV export, and a plot helper |
| `rolling.py` | 67 | `rolling_information_ratio` with cached evaluation |
| `summary.py` | 64 | One-shot `summary_table` (CAGR, vol, MDD, IR, Sharpe, turnover, hit rate) |
| `turnover.py` | 56 | `realized_turnover` and `turnover_cost` from a weight history |

### `perf/` — 3 files, 543 lines

| File | Lines | Purpose |
|---|---|---|
| `cache.py` | 267 | `CovCache` with optional LRU + incremental rolling-window covariance update |
| `rolling_cache.py` | 195 | Filesystem-backed cache (`joblib`) for rolling computations |
| `timing.py` | 81 | `log_timing` + `timed_stage` context manager |

---

## Issues Found

### 1. `RiskParity` and `RobustRiskParity` overlap heavily

`weights/risk_parity.py` and the body of `RobustRiskParity.weight` in
`weights/robust_weighting.py:330-390` are running the same algorithm — pull the
diagonal, take inverse vol, normalise. The "robust" version adds diagonal
loading and condition-number checks. The plain version has its own ad-hoc
fallback that does almost the same thing (lines 29-42 of `risk_parity.py`).

It's not a duplicate in the strict sense (each is registered under a different
name in the plugin registry, and `RobustMeanVariance` even calls back into the
plain `RiskParity` as one of its `safe_mode` options). But there is one
inverse-vol routine reimplemented three times across the package. Worth
considering a shared `_inverse_vol_weights(cov, *, loading_factor=...)` helper
that both classes call.

### 2. The `1e-8` minimum-variance floor is hard-coded in three places

- `weights/risk_parity.py:32` — `min_var = np.max(variances) * 1e-8`
- `weights/hierarchical_risk_parity.py:22` — `std = np.maximum(std, np.max(std) * 1e-8)`
- `weights/robust_weighting.py:371` — `min_std = max_std * 1e-8 ...`

This is the kind of constant that should sit in `trend_analysis/constants.py`
next to `NUMERICAL_TOLERANCE_HIGH` (which `equal_risk_contribution.py:81`
already imports from there). Right now if anyone tunes the floor they have to
remember to touch all three files.

### 3. `metrics/__init__.py` mutates `builtins` and `sys.modules` at import time

Lines 412-432:

```python
_legacy = types.ModuleType("tests.legacy_metrics")
for _name in (...):
    _legacy.__dict__[_name] = globals()[...]
sys.modules.setdefault("tests.legacy_metrics", _legacy)

setattr(_bi, "annualize_return", annualize_return)
setattr(_bi, "annualize_volatility", annualize_volatility)
```

Two real problems here. First, importing `trend_analysis.metrics` injects
`annualize_return` and `annualize_volatility` straight into Python's `builtins`
module — they become globally visible to every other module in the process,
not just to anything that imports the metrics package. That is exactly the
kind of side effect that makes test isolation and import order fragile.

Second, the synthetic `tests.legacy_metrics` module is created so that
`tests/test_metric_vectorise.py` and `tests/test_metric_vectorise_param.py`
can do `import tests.legacy_metrics as L`. That works, but it ties production
code to test layout. The cleaner fix is a real shim file under `tests/` and
let production code stay clean. `scripts/run_multi_demo.py:638-650` even has
a self-test that asserts both side effects are still in place, which makes
this hard to remove without touching the smoke script too.

### 4. `metrics/sortino_ratio` carries a "match the golden test" workaround

Lines 275-280 and 296-301 of `metrics/__init__.py`:

```python
# Special handling for single downside observation to match golden test
# expectations
if len(downside) == 1:
    down_vol = 2.0 * abs(downside.iloc[0])
```

The comment is honest about it: the formula bends to fit a fixture rather
than matching a textbook downside-deviation definition. The golden file
should be regenerated on the correct definition and this branch removed.
Right now any caller with exactly one negative observation gets a value
that no other Sortino implementation will reproduce.

### 5. `metrics/__init__.py` is a 439-line catch-all

The package has split out attribution, rolling, summary and turnover into
sibling files, but the `__init__` itself still holds eight metrics, the
registry, the legacy aliasing, and the import-time monkey patching. Splitting
the actual metric implementations into `core.py` (or per-metric files) and
keeping `__init__.py` as the public surface would make it much easier to
navigate. The four `import_module(".attribution", __name__)` calls at the
bottom (lines 436-439) are also unusual — that pattern only exists to keep
Ruff quiet about unused imports, and a normal `from . import attribution as
_attribution` with `__all__` would be more idiomatic.

### 6. `metrics/attribution.py` pulls matplotlib at import time

Line 23 imports `matplotlib.pyplot` unconditionally. Because
`metrics/__init__.py` then does `import_module(".attribution", __name__)` on
every import, matplotlib is loaded any time anyone touches the metrics
package, even if they only want `sharpe_ratio`. matplotlib is not cheap to
import. Moving the `import matplotlib.pyplot as plt` inside `plot_contributions`
would fix this without any API change.

### 7. `metrics/summary.py` looks like a dead module

`summary_table` is imported in exactly one place outside its own file:
`examples/legacy_streamlit_app/pages/04_Results.py`. There are no tests for
it and no production callers. The string match `metrics_summary` in
`src/trend_analysis/export/__init__.py` is a different thing (it's a dict key
naming an export sheet, not a call to this function).

Before deleting I'd want to confirm with you whether the legacy streamlit
example is supposed to be retired with the rest of `examples/legacy_*`. If
yes, this whole file goes too.

### 8. `signals.py` reimplements `perf.timing.timed_stage`

`src/trend_analysis/signals.py:96-110` defines its own `_timed_stage` context
manager that does essentially the same thing as `perf/timing.py:timed_stage`.
Two separate implementations of the same idea. The signals one is slightly
narrower (DEBUG-only, no metadata), but it should be a thin wrapper around
the perf version, not a parallel implementation.

### 9. `perf/rolling_cache.py` does filesystem work at import time

Line 175: `_DEFAULT_ROLLING_CACHE = RollingCache()`. The `RollingCache.__init__`
calls `_ensure_cache_dir`, which calls `mkdir(parents=True, exist_ok=True)`
on `~/.cache/trend_model/rolling`. Importing the module touches disk. The
fallback to `tempfile.gettempdir()` softens it, but for a unit-test process,
or anything running with a read-only home, this can still fail or create
unexpected paths. A lazy `get_cache()` that constructs on first call would
remove the side effect.

### 10. Hard-coded `1e12` condition-number threshold scattered

- `weights/equal_risk_contribution.py:36` (literal `1e12`)
- `weights/robust_weighting.py:141, 317` (default param `1e12`)
- `weights/robust_config.py:59` (literal `1.0e12` as the fallback)

These are all the same number representing the same idea ("matrix is too
ill-conditioned to invert safely"). Same fix as issue 2 — promote to a
constant.

### 11. `metrics/__init__.py` Sortino path duplicates itself for Series vs DataFrame

Lines 269-287 (Series branch) and 290-312 (DataFrame branch) carry a
near-identical body, including the comment about the single-observation
workaround copy-pasted twice. If issue 4 is addressed, this is a good time
to also collapse it into one helper that gets `apply`-ed for the DataFrame
case.

---

## Notes from the Review

Overall this layer is in better shape than the CLI code I went through last
week. The weight engines have real numerical-safety logic, the perf cache
is a meaningful optimisation that is actually wired into
`multi_period/engine.py` and `core/rank_selection.py`, and the timing helpers
are tidy.

The pattern I want to flag for the Week 2 write-up is that a lot of the
issues in this slice are small — a hard-coded constant here, a duplicated
helper there — but they cluster around the same theme: **production code
reaching into test territory** (the `tests.legacy_metrics` shim, the
`builtins` patching, the Sortino "match the golden file" branch). That is
worth a focused PR on its own once we get to the cleanup phase.

A few things that I deliberately did *not* flag as issues:

- The plugin-registry decorator pattern in the weights package is sound. Each
  engine registers itself under one or two names and the rest of the code
  asks the registry for them by string. I had no problem tracing usage.
- `perf/cache.py`'s incremental covariance update (`incremental_cov_update`)
  looks correct on the math (`S2' = S2 - oo^T + nn^T`, mean re-derived from
  `S1`). It's the kind of thing that's easy to get wrong and this one is
  written carefully, with the aggregates either materialised up front or
  reconstructed from `cov` via the standard identity. It's also exercised
  by `tests/test_multi_period_engine_incremental_extra.py` and friends, so
  there's coverage.
- `rolling.py` correctly hashes the inputs (returns + benchmark) and tags
  the cache key with both the frequency and the method name
  (`rolling_information_ratio_ddof1`). If the formula changes, bumping the
  method tag will invalidate old entries. That's the right shape.

### Suggested follow-ups (in priority order)

1. **Remove the `builtins` patching and `tests.legacy_metrics` synthetic
   module from `metrics/__init__.py`.** Replace with an actual file under
   `tests/` and update `scripts/run_multi_demo.py:625-650` accordingly.
2. **Fix the Sortino single-downside branch** by regenerating the golden
   fixture rather than carrying a workaround in production code.
3. **Promote the magic numbers** (`1e-8` variance floor, `1e12` condition
   threshold) to `trend_analysis/constants.py`.
4. **Extract a shared `_inverse_vol_weights` helper** so `RiskParity` and
   `RobustRiskParity` share an implementation.
5. **Make matplotlib import lazy** inside `plot_contributions`.
6. **Make the rolling cache lazy** — replace the module-level singleton with
   a `get_cache()` that constructs on first call.
7. **Decide on `metrics/summary.py`.** If the legacy streamlit example is
   being retired, this goes with it. If not, add a test and a non-legacy
   caller.
8. **Replace `signals._timed_stage` with `perf.timing.timed_stage`.**

None of these are urgent. Most are 30-minute changes. Bundling them into one
"metrics/weights cleanup" PR after the audit is probably the right move.

---

**Lines reviewed today:** 2,062
**Files reviewed today:** 14
**Running total for the audit (CLI + Config + standalones + peripherals + Day 6 + today):** approximately 23,500 of ~45,000 lines.
