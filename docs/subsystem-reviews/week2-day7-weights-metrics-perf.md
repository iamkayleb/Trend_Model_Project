# Week 2, Day 7 — Review of `weights/`, `metrics/`, and `perf/`

**Date:** 2026-04-30
**Reviewer:** Kayleb
**Scope:** 14 files, 2,062 lines across the three packages that handle portfolio
weighting, performance metrics, and the in-memory / on-disk caching layer that
sits underneath them.

This is the second day of Week 2. Yesterday I finished the analysis-engine core
(`core/`, `stages/`, `engine/`); today's review is the layer immediately around
it — how we turn covariance matrices into weights, how we score the resulting
return streams, and how we avoid recomputing those things on every overlapping
window.

---

## Files Reviewed

### `src/trend_analysis/weights/` (5 files, 759 lines)

| File | Lines | What it does |
|---|---|---|
| `__init__.py` | 14 | Re-exports the five weight engines. Nothing else. |
| `risk_parity.py` | 50 | `RiskParity` — inverse-volatility weighting with non-positive-variance and non-finite-inverse guards. |
| `equal_risk_contribution.py` | 102 | `EqualRiskContribution` — iterative ERC with its own condition-number check, eigenvalue check, and ad-hoc regularization. |
| `hierarchical_risk_parity.py` | 122 | `HierarchicalRiskParity` — scipy-backed clustering, recursive bisection, fallback to equal weights on any failure. |
| `robust_weighting.py` | 390 | `RobustMeanVariance` and `RobustRiskParity` plus the Ledoit–Wolf and OAS shrinkage helpers and a shared `diagonal_loading` utility. |
| `robust_config.py` | 81 | Pure config-translation layer — turns the robustness section of the YAML into kwargs for the engines above. |

### `src/trend_analysis/metrics/` (5 files, 760 lines)

| File | Lines | What it does |
|---|---|---|
| `__init__.py` | 439 | The metric registry plus the canonical implementations of `annual_return`, `volatility`, `sharpe_ratio`, `sortino_ratio`, `max_drawdown`, `information_ratio`. Also imports the four submodules and does some legacy-name plumbing (more on that below). |
| `attribution.py` | 134 | `compute_contributions`, `export_contributions`, `plot_contributions`. The only module in `metrics/` that pulls in matplotlib. |
| `rolling.py` | 67 | `rolling_information_ratio` — wraps the perf rolling cache. |
| `turnover.py` | 56 | `realized_turnover` and `turnover_cost`. Tidy and self-contained. |
| `summary.py` | 64 | `summary_table` — composes the metric primitives plus turnover into the standard one-column report. |

### `src/trend_analysis/perf/` (3 files, 543 lines)

| File | Lines | What it does |
|---|---|---|
| `cache.py` | 267 | `CovCache` (in-memory, optional LRU) plus `compute_cov_payload` and `incremental_cov_update` for sliding-window covariance reuse. |
| `rolling_cache.py` | 195 | `RollingCache` — filesystem-backed joblib cache for arbitrary rolling computations. Used by `metrics/rolling.py` today. |
| `timing.py` | 81 | `log_timing` and `timed_stage` — the lightweight stage-timing logger that the cache modules emit through. |

---

## Issues Found

I've grouped these by severity. Nothing here is on fire, but a few of these
should make the cleanup list before we start the Week 4 write-up.

### High — worth fixing soon

1. **`metrics/__init__.py` monkey-patches `builtins`.** Lines 431–432 do
   `setattr(_bi, "annualize_return", ...)` and the same for
   `annualize_volatility`. That makes those two names visible globally in
   every Python module in the process — without an import. The only thing
   relying on this is `scripts/run_multi_demo.py`, which explicitly checks
   `hasattr(builtins, "annualize_return")`. This was almost certainly added
   to keep an old demo script working; we should either delete the demo
   check or import the names properly there. Polluting builtins from a
   library import is the kind of thing that bites you six months later when
   another module accidentally shadows it.

2. **Same file fabricates a fake `tests.legacy_metrics` module at import
   time.** Lines 412–426 build a `types.ModuleType("tests.legacy_metrics")`,
   stuff it with the public metric functions, and register it in
   `sys.modules`. Two test files (`test_metric_vectorise.py`,
   `test_metric_vectorise_param.py`) and `scripts/run_multi_demo.py` then
   `import tests.legacy_metrics`. So production code is creating a fake
   module on behalf of the test suite. The right fix is a small real module
   in `tests/` that imports from `trend_analysis.metrics`. Removing this
   block would also let me delete a chunk of `_bi`/`sys`/`types` imports
   from a metrics package that has no business with any of them.

3. **`RiskParity` vs `RobustRiskParity` are 80% the same engine.** Both
   compute inverse-volatility weights from the diagonal of the covariance
   matrix; `RobustRiskParity` adds a condition-number check and diagonal
   loading. We register both under the plugin registry as separate names
   (`risk_parity` / `vol_inverse` vs `robust_risk_parity`), but a config
   author has no obvious reason to pick one over the other. Recommend
   folding the robust version into `RiskParity` behind constructor kwargs
   (`condition_threshold`, `diagonal_loading_factor`) defaulting to
   permissive values, and dropping the duplicate class. Same arguments
   apply, less surface area, fewer ways to be wrong.

### Medium — flag for cleanup

4. **The `sortino_ratio` "single downside observation" branch is a
   test-coupled kludge.** `metrics/__init__.py` lines 275–281 and 295–301:
   when there is exactly one negative excess return, we hard-code the
   downside vol to `2.0 * abs(value)` "to match the golden test file
   expectations." That comment is in the source, twice. Either the formula
   is principled and deserves a real justification, or the golden file is
   wrong and should be regenerated. Right now the production code is
   deferring to a fixture and we won't notice if the math drifts.

5. **`_compute_ratio_with_zero_handling` is only used by `sharpe_ratio`.**
   `sortino_ratio` and `information_ratio` each reimplement their own
   zero-denominator guards inline, with slightly different behaviour
   (sortino returns `_empty_like`, IR replaces `0` with `np.nan` in the
   Series path). Either route all three through the helper or delete the
   helper. The current state — one consumer, three styles — is the worst
   of both worlds.

6. **`equal_risk_contribution.py` rolls its own diagonal loading.**
   `robust_weighting.py` already exports a `diagonal_loading()` helper.
   ERC duplicates the same idea inline at lines 41 and 51–52 with
   different scaling (`1e-6` vs `1e-4` vs trace-based). Two engines, two
   conventions for the same regularization. They should share the helper
   and we should pick one default.

7. **`signals.py` has its own `_timed_stage` context manager.** It does
   essentially the same thing as `perf/timing.py:timed_stage` but logs to
   a different logger and uses a different format (`%.2f ms` vs
   `duration_ms=` key/value). If we ever want to grep timing data across
   the codebase, the inconsistent format will hurt. Switch `signals.py`
   over to the perf helper.

8. **`metrics/attribution.py` imports `matplotlib.pyplot` at the top.**
   Anyone touching the metrics package pays the matplotlib import cost
   even if they never call `plot_contributions`. Either lazy-import inside
   the plotting function, or move `plot_contributions` into `viz/` and
   leave `attribution.py` purely numeric.

### Low — worth noting, not blocking

9. **`RollingCache` has no eviction.** Default cache directory is
    `~/.cache/trend_model/rolling/` and it just keeps growing as window
    sizes / dataset hashes change. Probably fine for a developer's machine
    but on a long-running container it's a slow disk leak. Add a max-size
    or max-age sweep, or at least document it.

10. **`RollingCache._ensure_cache_dir` silently disables itself on
    `OSError`/`PermissionError`.** No log line, no warning. If the path
    becomes unwritable for some reason, you'll get correct-but-slow
    behaviour and no signal that the cache is dead. One `logger.warning`
    would solve this.

11. **`compute_cov_payload` silently does `ffill().fillna(0.0)` on the
    input frame.** Forward-filling I'd defend; the trailing zero-fill
    biases variance downward and the caller has no way to opt out. At
    minimum we should log when fills happen, or take a `na_policy`
    parameter.

12. **`CovCache` and `RollingCache` are unrelated APIs that both call
    themselves "cache".** Different keys, different payloads, different
    eviction stories. Not a defect — they genuinely solve different
    problems — but if we ever add a third cache (e.g. for selector
    outputs), it'd be worth introducing a tiny shared protocol so they
    look similar from the outside.

13. **`metrics/summary.py` casts `float(ir)` etc. unconditionally.** With
    a scalar benchmark and a Series of returns, `information_ratio`
    returns a scalar, so this works in practice. But the cast hides the
    type contract — if someone ever passes a DataFrame in, the cast will
    raise inside `summary_table` rather than at the boundary where the
    error is more obvious.

---

## Notes from the Review

A few impressions that don't fit cleanly under "issues" but are worth keeping
in the back of my head as we move into Day 8 (portfolio + multi_period).

**The weighting layer is in better shape than I expected.** The plugin
registration via `weight_engine_registry` is consistent, every engine handles
the empty-covariance and label-mismatch cases the same way, and HRP in
particular has a thoughtful try/except wrapping that always returns *some*
weight vector instead of blowing up the pipeline. The robustness story is
real — it's just spread across two files (`robust_weighting.py` and the
inline guards in ERC/HRP) and could be consolidated.

**`metrics/__init__.py` is doing too much.** At 439 lines it's the registry,
six metric implementations, three private helpers, the legacy-alias
plumbing, the fake-test-module trick, and the builtins poke. Most of those
were added at different times for different reasons and nothing has been
removed. I'd budget a small refactor: split the canonical metrics into
`metrics/core.py`, leave `__init__.py` as the registry-and-public-API
shim, and delete the legacy plumbing in the same PR. That alone would
shrink the file by a third and make the public surface obvious.

**The perf layer is the surprise of the day.** `cache.py` in particular —
the `compute_cov_payload` / `incremental_cov_update` pair with the S1/S2
aggregates is actually a nice piece of code. The sliding-window math is
correct, the `_ensure_aggregates` reconstruction trick from
`cov*(n-1) + n*mm^T` is clever, and it's already wired into
`multi_period/engine.py`. If we get a performance complaint about
multi-period runs, this is where I'd look first — and it's already
solved. Worth a callout in the Week 4 write-up.

**Test coupling in the source tree is the recurring theme.** Between the
`tests.legacy_metrics` fake module, the builtins poke, and the
"single-downside = 2 × abs" sortino branch, this package has three
separate places where production code bends to keep an old test green.
None of these are catastrophic individually, but together they're a
pattern. I'd like to address all three in one cleanup PR rather than
piecemeal — easier to review, and we can regenerate the affected
golden files in a single change.

**Risk-engine duplication is mild but real.** `RiskParity` /
`RobustRiskParity`, the two flavours of diagonal loading, and the two
condition-number-check sites all want to be one thing eventually.
None of this is urgent — the engines work — but every time I read
the weights package I have to remind myself which entrypoint does
what. Consolidating would help future-me and any new contributor.

**No dead code in this slice.** Every file is imported somewhere
non-trivial. `summary.py` is used by `examples/legacy_streamlit_app/`
and tests; `attribution.py` is referenced in tests and the v0
exporter; everything in `perf/` is hit by `multi_period/engine.py` or
`metrics/rolling.py`. If we hadn't already triaged dead code in
Week 1, I'd have flagged that as a positive surprise.

### Suggested cleanup priorities (for the Week 4 list)

1. Remove the builtins poke and the fake `tests.legacy_metrics` module
   from `metrics/__init__.py`. Replace with a real test helper.
2. Fix the sortino single-downside branch (regenerate the golden or
   document the formula).
3. Consolidate `RiskParity` and `RobustRiskParity` into one configurable
   engine.
4. Share the `diagonal_loading` helper between ERC and the robust engines.
5. Add a small log line when `RollingCache` falls back to disabled mode
   and document the unbounded-growth behaviour.

Estimated effort: 1–2 days of focused work for items 1–4, half a day
for item 5. None of this needs to block other Week 2 reviews; it's
material for the consolidated cleanup PR I'll open at the end of Week 4.

### What's next

Day 8 tomorrow: `portfolio/`, `rebalancing/`, and `multi_period/`. The
multi-period engine is the heaviest single chunk in the codebase
(4,352 lines), so I'm going to start there and budget most of the day
for it. Today's perf-cache findings are already useful context — the
covariance reuse path I read in `cache.py` is the same one the
multi-period engine consumes.
