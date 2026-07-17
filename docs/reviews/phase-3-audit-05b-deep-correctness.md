# Phase-3 Deep Correctness Pass 05b: `rank_select_funds` & `_compute_weights_and_stats`

**Date:** 2026-07-16
**Reviewer:** Project review (deep correctness pass)
**Branch:** `phase-3`
**Scope:** The two large functions deferred from audit-05:
- `core/rank_selection.py::rank_select_funds` (`:446-703`)
- `stages/portfolio.py::_compute_weights_and_stats` (`:184-695`)

---

## Summary of Findings

| # | Function | Finding | Severity |
|---|----------|---------|----------|
| 1 | `_compute_weights_and_stats` | Vol-targeting `scale_factors` appear applied **twice** to the user portfolio (weights already carry the tilt, returns are scaled again) | High **if confirmed** |
| 2 | `_compute_weights_and_stats` | `_compute_stats` ignores `window.periods_per_year`; annualisation hardcoded to 12 via metric defaults | Medium |
| 3 | `_compute_weights_and_stats` | `monthly_cost` subtracted from every asset every period (holding-cost, not turnover-based) | Low (confirm intent) |
| 4 | `rank_select_funds` | `_dedupe_by_firm` backfill is O(n²) (`name in chosen` on a list) | Low |
| — | `rank_select_funds` | Sort order / `bottom_k` / threshold semantics | ✅ Correct |

---

## `rank_select_funds` — Correct

Reviewed the full selection path. No correctness defects found:

- **Sort order** (`:588-591`): `transform=="rank"` → ascending (1=best); otherwise `ascending = metric_name in ASCENDING_METRICS` (smaller-is-better metrics ascending, else descending). Best-first ordering is correct for all branches.
- **`bottom_k`** (`:603-617`): coerced to a non-negative int, guarded against `>= len`, and `scores.iloc[:-bottom_k]` correctly drops the worst k after best-first sorting.
- **Inclusion approaches** (`:669-699`): `top_n` requires `n`; `top_pct` validates `0 < pct <= 1` and computes `k = max(1, round(len*pct))`; `threshold` keeps `<=` when ascending else `>=` — all correct.
- **Cache-aliasing awareness**: the cached-metric path calls `.copy()` before `_apply_transform` (`:572`) so the transform can't corrupt the shared `WindowMetricBundle`. Good.
- **`risk_free` override** correctly disables bundle/cov caching (`:515-519`) since an override invalidates cached metrics.

**Minor (④):** `_dedupe_by_firm` backfill (`:659-665`) does `if name in chosen` against a list each iteration — O(n²). Negligible for realistic universes; use a `set` if universes ever get large.

---

## `_compute_weights_and_stats` — Findings

### ① Vol-targeting scale appears double-applied to the user portfolio — High (if confirmed)

**Mechanism.** `compute_constrained_weights` (in `risk.py`) returns weights that **already embed** the per-asset vol scale:
```python
scaled = _normalise(constrained_base.mul(scale_factors))   # weights ∝ base · scale, summing to 1
return constrained, diagnostics(scale_factors=scale_factors)
```
Back in `_compute_weights_and_stats`, the same `scale_factors` are then applied **again** to the returns before computing the portfolio:
```python
scale_factors = risk_diagnostics.scale_factors.reindex(fund_cols).fillna(0.0)   # portfolio.py:569
in_scaled  = window.in_df[fund_cols].mul(scale_factors, axis=1) - monthly_cost  # :571
...
user_w = weights_series.to_numpy(...)                                           # :643  (∝ base·scale)
in_user = calc_portfolio_returns(user_w, in_scaled, cash_weight=..., cash_returns=rf_in)  # :646
```
`calc_portfolio_returns` does **not** renormalise (`:150` is a plain `returns.mul(weights).sum(axis=1)`). So the effective coefficient on asset *i*'s raw return is:

```
user_w_i · scale_i   where   user_w_i ∝ base_i · scale_i
                     ⟹   effective exposure ∝ base_i · scale_i²
```

The cross-sectional volatility tilt is therefore **squared** relative to the base allocation.

**Corroborating tell — EW vs user asymmetry.** The equal-weight benchmark uses weights that do *not* carry the scale:
```python
in_ew = calc_portfolio_returns(ew_weights, in_scaled)   # exposure ∝ scale_i  (applied once)
```
So the EW benchmark applies the scale **once** while the user portfolio applies it **twice**. Even if per-asset vol-targeting is intended, EW and user are being scaled inconsistently, which makes their comparison (a core output of this function) apples-to-oranges.

**Why "if confirmed":** vol-targeting composition is subtle and I'm reasoning statically. It is possible the design *intends* "tilt capital toward low-vol assets (weights) **and** lever each asset to target vol (returns)." But the EW/user asymmetry and the squared tilt both look unintended.

**Recommendation:** add a numerical test: construct 2 assets with known vols, `target_vol` set, and assert the realised portfolio vol equals target and that user vs EW differ only by the intended tilt. If the squared tilt is unintended, feed **unscaled** returns to the user-portfolio calculation (the weights already carry the tilt), or use `base_series` with scaled returns — but not both.

### ② `_compute_stats` ignores `window.periods_per_year` — Medium

`_compute_stats` (`portfolio.py:159`) calls the metrics without an annualisation factor:
```python
cagr=float(annual_return(df[col])),          # periods_per_year defaults to 12
vol=float(volatility(df[col])),              # defaults to 12
sharpe=float(sharpe_ratio(df[col], rf)),     # defaults to 12
```
All metric functions default `periods_per_year: int = 12` (`metrics/__init__.py:150,208,224,…`). Meanwhile the *same* function threads `window.periods_per_year` into `realised_volatility`/`compute_constrained_weights` (`:473`, `:490`).

**Impact:** for the normal monthly-resampled pipeline this is correct (12 = monthly). But it is a latent bug: if the pipeline ever runs on non-monthly data, the risk engine would use the true factor while the reported CAGR/vol/Sharpe would be annualised at 12 — silently wrong and internally inconsistent.

**Recommendation:** thread `window.periods_per_year` into `_compute_stats` and pass it to each metric, rather than relying on the default.

### ③ Flat `monthly_cost` drag — Low (confirm intent)

`in_scaled = returns·scale − monthly_cost` (`:571-572`) subtracts `monthly_cost` from **every asset in every period**, i.e. a holding-cost drag independent of turnover. For single-window analysis this is a defensible simplification (turnover-based costs live in the multi-period engine), but it is applied to both EW and user portfolios uniformly.

**Recommendation:** confirm this flat-drag model is the intended single-period cost treatment and document it; otherwise route costs through turnover.

---

## Verdict

`rank_select_funds` is **correct** (one negligible efficiency nit).

`_compute_weights_and_stats` has one **material, must-confirm** issue: the vol-targeting scale appears to be applied twice to the user portfolio (exposure ∝ base·scale²) and inconsistently versus the equal-weight benchmark (scale once). If confirmed, it distorts every vol-targeted run's returns and the user-vs-EW comparison. Finding ② is a smaller latent inconsistency (hardcoded annualisation). Both are cheap to fix and both warrant regression tests.

**These are static-analysis findings.** Finding ① in particular should be verified with a numerical fixture before any change — I am flagging a strong smell and the exact mechanism, not asserting a proven defect.

---

## Files Reviewed

- [x] `src/trend_analysis/core/rank_selection.py::rank_select_funds` (full, `:446-703`)
- [x] `src/trend_analysis/stages/portfolio.py::_compute_weights_and_stats` (full, `:184-695`)
- [x] `src/trend_analysis/stages/portfolio.py::calc_portfolio_returns`, `_compute_stats` (supporting)
- [x] `src/trend_analysis/metrics/__init__.py` (annualisation defaults)
- [x] `src/trend_analysis/risk.py::compute_constrained_weights` (weight/scale composition — cross-checked)
