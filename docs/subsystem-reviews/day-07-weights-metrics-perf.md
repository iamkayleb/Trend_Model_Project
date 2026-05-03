# Day 7 Subsystem Review: weights/, metrics/, perf/

**Reviewer:** Kayleb
**Date:** 2026-05-03
**Scope:** Week 2 / Day 7 — weighting strategies, metrics calculation, and the
performance caching layer.
**Total LOC reviewed:** ~2,062 across 14 files.

---

## Summary

Spent the day on the three packages that make up the "math core" of the
analytics layer: `src/trend_analysis/weights/`, `src/trend_analysis/metrics/`,
and `src/trend_analysis/perf/`. Overall I came away pretty positive — the
robustness story (shrinkage, condition-number checks, safe-mode fallback) is
well thought out and the caching layer is properly instrumented with hit/miss
logging. There are a few real issues worth flagging, mostly around code
duplication, a couple of legacy compatibility hacks I'd like to retire, and
one or two places where the math is approximated in ways the docstring doesn't
quite admit to. Nothing is on fire. Details below.

---

## Files Reviewed

### `src/trend_analysis/weights/` (5 files, ~660 lines)

| File | Lines | Purpose |
|------|------:|---------|
| `__init__.py` | 14 | Re-exports the four engines. |
| `risk_parity.py` | 50 | Plain inverse-volatility weighting (registered as `risk_parity` and `vol_inverse`). |
| `equal_risk_contribution.py` | 102 | ERC via iterative scaling, with condition-number guard and equal-weight fallback. |
| `hierarchical_risk_parity.py` | 122 | Lopez de Prado HRP using `scipy.cluster.hierarchy`. |
| `robust_weighting.py` | 390 | `RobustMeanVariance` and `RobustRiskParity`, plus shared shrinkage helpers (`ledoit_wolf_shrinkage`, `oas_shrinkage`, `diagonal_loading`). |
| `robust_config.py` | 81 | Translates the `robustness:` config block into engine kwargs. |

### `src/trend_analysis/metrics/` (5 files, ~760 lines)

| File | Lines | Purpose |
|------|------:|---------|
| `__init__.py` | 439 | Metric registry plus the core six functions (`annual_return`, `volatility`, `sharpe_ratio`, `sortino_ratio`, `max_drawdown`, `information_ratio`) and a legacy compat layer. |
| `attribution.py` | 134 | Signal/rebalancing attribution and a small plotting helper. |
| `rolling.py` | 67 | `rolling_information_ratio` with cache integration. |
| `summary.py` | 64 | `summary_table` — packs the headline stats (CAGR, vol, Sharpe, IR, MDD, turnover, hit-rate) into one DataFrame. |
| `turnover.py` | 56 | `realized_turnover` and `turnover_cost` from a weights history. |

### `src/trend_analysis/perf/` (3 files, ~540 lines)

| File | Lines | Purpose |
|------|------:|---------|
| `cache.py` | 267 | In-memory LRU `CovCache` plus `compute_cov_payload` and an `incremental_cov_update` for sliding windows. |
| `rolling_cache.py` | 195 | Filesystem-backed `RollingCache` (joblib pickles under `~/.cache/trend_model/rolling`). |
| `timing.py` | 81 | `log_timing` and `timed_stage` — single-line perf log emitter used by both caches. |

---

## Issues Found

### 1. `RiskParity` and `RobustRiskParity` are essentially the same engine

`weights/risk_parity.py` and the `RobustRiskParity` class in
`weights/robust_weighting.py` (line 311 onwards) both compute inverse-vol
weights with safety nets. The "robust" version adds a condition-number gate
and diagonal loading, but the core math is identical. We have two registry
entries (`risk_parity` and `robust_risk_parity`) doing the same thing with
slightly different guard rails. Either:

- collapse them and let the base `RiskParity` accept a `condition_threshold`
  kwarg (default `inf` gives the current behaviour), or
- have `RobustRiskParity` delegate to `RiskParity` after the loading step.

I'd lean toward the first — it removes ~50 lines and one registry alias.

### 2. ERC engine is missing the robustness knobs the others have

`EqualRiskContribution.__init__` only takes `max_iter` and `tol`
(`equal_risk_contribution.py:19`). It still does internal condition checks
and diagonal loading, but those values are hard-coded (`1e12` threshold,
`1e-6` loading factor). For consistency with `RobustMeanVariance` /
`RobustRiskParity` these should be plumbed through the constructor — and
ideally fed by `weight_engine_params_from_robustness` so the YAML config
controls all engines uniformly.

### 3. `ledoit_wolf_shrinkage` and `oas_shrinkage` are not the textbook formulas

`robust_weighting.py:30-71` and `:74-111`. Both functions admit in their
comments that they use a "matrix-based approximation" because they don't have
access to raw observations. That's fine pragmatically, but the function names
are textbook names and a reader will reasonably expect the actual Ledoit-Wolf
optimal-intensity calculation. We should either:

- pass the raw return matrix in and compute the real intensity, or
- rename to `ledoit_wolf_like_shrinkage` (or similar) and put the caveat at
  the top of the docstring instead of buried in a comment.

I lean toward fixing the implementation since `compute_cov_payload`
(`perf/cache.py:150`) already has the raw frame in hand at the call site.

### 4. `metrics/__init__.py` is doing too much and pollutes `builtins`

Lines 431–432:

```python
setattr(_bi, "annualize_return", annualize_return)
setattr(_bi, "annualize_volatility", annualize_volatility)
```

That installs two functions onto Python's `builtins` module on import. The
same trick lives in `config/models.py:182` for the config class, so this is a
pattern in the repo, but it's still bad — it's there to satisfy old tests
that reference these names without importing them. Worth retiring as part of
a small cleanup PR; I checked usage and only `tests/test_run_analysis.py` and
`tests/test_metrics.py` still rely on it.

### 5. `tests.legacy_metrics` is registered twice — once as a real file, once as a synthetic module

`metrics/__init__.py:412-426` builds a `types.ModuleType("tests.legacy_metrics")`
and calls `sys.modules.setdefault(...)`. There is also a real file at
`tests/legacy_metrics.py`. Whichever loads first wins, and `setdefault` makes
this nondeterministic depending on import order. The real file is the source
of truth (it's used by `test_metric_vectorise.py` and friends). The synthetic
shim is dead code in any normal test run — recommend deleting lines 403–426.

### 6. Sortino's "single observation" branch is a documented hack

`metrics/__init__.py:276-281` and again at 296-301:

```python
if len(downside) == 1:
    # When only one downside observation, use 2 * abs(value) as downside volatility
    # This matches the golden test file expectations
    down_vol = 2.0 * abs(downside.iloc[0])
```

The comment literally says "matches golden test expectations". That's a smell —
the golden tests are leaking into the math. Either there's a real reason for
the `2.0 * abs(...)` (in which case I want a citation in the docstring) or the
golden file should be regenerated against `np.nan` (which is what every other
ratio does for n=1 stdev).

### 7. `_compute_ratio_with_zero_handling` is only used once

Defined at `metrics/__init__.py:124`, used only by `sharpe_ratio` at line 242.
The Sortino and IR functions have their own ad-hoc div-by-zero handling. This
is the classic "premature helper" — either make all three ratio functions
share it (preferred) or inline it.

### 8. Sortino's DataFrame branch loops in Python while the Series branch is vectorised

`metrics/__init__.py:288-312`. The DataFrame path iterates columns one at a
time. For wide universes this is meaningfully slower than necessary. The
Series logic could be vectorised across columns with a couple of masks
(downside count per column, then an apply for the n=1 special case). Not
urgent, but easy win.

### 9. `RobustMeanVariance` passes the *shrunk* matrix to the safe-mode fallback

`robust_weighting.py:303` — when condition number exceeds the threshold, it
calls `self._safe_mode_weights(shrunk_cov)`. So an HRP fallback gets the
already-shrunk covariance, not the raw one. That might be intentional (HRP
benefits from a better-conditioned input) but the docstring doesn't say so.
Worth a one-line note in the diagnostics or the class docstring so future
debugging is less mysterious.

### 10. `robust_config.py` has six chained "fallback key" lookups

`weights/robust_config.py:32-50` accepts `condition_check`, `condition_threshold`,
`condition_check_enabled`, `safe_mode`, and `diagonal_loading_factor` at
either the top level of `robustness_cfg` or inside a nested `condition_check`
mapping. This is honest backwards compat for old configs, but it's a lot of
surface area. Once we've confirmed that no live config file uses the flat
keys (a `grep` on the YAML configs would do it), we can rip out half of this
file.

### 11. `RollingCache` has no payload version stamp

`rolling_cache.py:113` — file names are
`{hash}_{method}_{freq}_{window}.joblib`. If pandas changes its pickle layout
between versions (it has happened), or if we change a calculation under the
same `method_tag`, cached results will deserialise into something subtly
wrong. Two cheap mitigations:

- prepend a 2-character schema version to the filename (`v1_…`),
- write a sidecar `.meta` with `pandas.__version__` and check on load.

I'd take the schema version. The sidecar is overkill for now.

### 12. Two cache implementations with no shared protocol

`CovCache` and `RollingCache` have the same `get_or_compute(key, compute_fn)`
shape but no shared base class or `Protocol`. If we ever want to swap one in
for the other (e.g. "use the rolling disk cache as L2 behind the in-memory
L1") we'll need to refactor anyway. A small `Cache[K, V]` protocol in
`perf/timing.py` (or a new `perf/_base.py`) would future-proof this without
much cost.

### 13. `compute_cov_payload` and `incremental_cov_update` have a subtle precondition mismatch

`perf/cache.py:175-179` — `compute_cov_payload` cheerfully returns a payload
with `n=1` and a zero covariance matrix. `perf/cache.py:239` —
`incremental_cov_update` raises `ValueError` if `n < 2`. So a caller who
chains the two together on a degenerate window gets a confusing error a few
calls later, not at the source. Either tighten the input check upstream or
make the incremental update tolerate `n < 2` gracefully.

### 14. `log_timing` always logs at INFO

`perf/timing.py:34` — guarded by `isEnabledFor(logging.INFO)` but the level
isn't configurable. Probably fine, but note that turning on INFO globally
will spam the perf logger. A `level: int = logging.INFO` kwarg with a
`trend_analysis.performance` logger configured to its own handler would be
the right shape.

---

## Notes from the review

### What's good

- **Plugin registration is consistent.** Every weight engine registers itself
  with `weight_engine_registry`, and the metrics registry is symmetric
  (`@register_metric("name")`). Easy to find, easy to extend.
- **Robustness story is real.** The combination of shrinkage + condition-number
  gate + named safe-mode fallback in `RobustMeanVariance` is genuinely
  thoughtful. Diagnostics dictionary on the engine is a nice touch — anyone
  doing a postmortem on a weird run can pull it straight off.
- **Caches are instrumented.** Hits, misses, and incremental updates are all
  counted. `log_timing` emits a single key=value line per stage which is
  trivially `grep`-able and would feed cleanly into a Splunk/ELK pipeline.
- **Defensive math.** Every weight engine has at least one fall-through path
  to equal weights when the covariance matrix collapses. ERC also catches
  `np.linalg.LinAlgError` per iteration. This is the right level of paranoia
  for a portfolio engine that's going to see real-world data.
- **`turnover.py` is exactly the right size.** 56 lines, two functions, no
  plotting deps, vectorised. Model citizen.
- **Test coverage looks healthy.** I see `test_perf_timing`, `test_perf_cache_extended`,
  `test_rolling_cache`, `test_metrics_attribution`, `test_metrics_summary`,
  `test_metrics_turnover_extra`, `test_weighting_engines_extended`,
  `test_weight_engines_pathological`, `test_weighting_robustness`,
  and `test_robust_weighting`. The pathological/edge-case tests in particular
  give me confidence.

### Things I want to follow up on next week

- `RobustMeanVariance` is the only place we do a "minimum variance"
  computation. That's not a true robust mean-variance — there's no `mu`
  vector and no risk-aversion parameter. Either rename it or extend it. I'll
  pick this up when I get to the engine reviews on Day 8.
- `core/rank_selection.py` imports `CovCache` and `CovPayload` lazily
  (`perf/cache.py` types under `TYPE_CHECKING`). Fine, but I want to confirm
  on Day 8 that we're not paying for repeated cache construction in the
  multi-period engine.
- Day 8 covers `multi_period/`, which is the heaviest user of both cache
  classes. That's where any performance pathologies will show up first. I
  expect to have a stronger opinion on whether we need the shared `Cache`
  protocol after that.

### Quick wins I'd batch into one cleanup PR

1. Delete the synthetic `tests.legacy_metrics` shim (Issue 5).
2. Drop the `setattr(_bi, ...)` lines from `metrics/__init__.py` after
   updating the two test files that depend on them (Issue 4).
3. Replace the "single observation" Sortino hack with `nan` and regenerate
   the golden file (Issue 6).
4. Either inline `_compute_ratio_with_zero_handling` or use it from all three
   ratio functions (Issue 7).

That's a one-day PR, ~150 lines net deletion, no behaviour change for any
non-legacy caller. Happy to queue it up after the Week 2 review wraps.

### Bigger items for the backlog

- Consolidate `RiskParity` and `RobustRiskParity` (Issue 1).
- Plumb robustness knobs into ERC (Issue 2).
- Either fix the LW/OAS shrinkage formulas or rename them honestly (Issue 3).
- Add a schema version to the filesystem cache filenames (Issue 11).
- Add a `Cache` protocol in `perf/` (Issue 12).
