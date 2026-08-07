# Phase-3 First-Pass Audit 09: Metrics & Backtesting

**Date:** 2026-07-16
**Reviewer:** Project review (first pass)
**Branch:** `phase-3`
**Scope:** `metrics/*`, `backtesting/*`

> First-time review — this subsystem was not covered in the earlier reviews. Reviewed
> against the current phase-3 code.

---

## Subsystem Summary

| File | Lines | Role | Depth |
|------|-------|------|-------|
| `metrics/__init__.py` | ~440 | Metric registry + core metrics (`annual_return`, `sharpe_ratio`, …) | Read |
| `metrics/deflated_sharpe.py` | 129 | PSR / deflated Sharpe (Bailey–López de Prado) | **Deep** |
| `metrics/factor_attribution.py` | 81 | OLS factor exposures | **Deep** |
| `metrics/turnover.py` | 72 | Realised turnover + linear cost | Read |
| `metrics/attribution.py` | 134 | Brinson-style attribution | Scan |
| `metrics/rolling.py` | 67 | Rolling metrics | Scan |
| `metrics/summary.py` | 64 | Summary metrics | Scan |
| `backtesting/harness.py` | 653 | `run_backtest`, `CostModel`, execution-lag, membership | Read |
| `backtesting/bootstrap.py` | 131 | Block bootstrap | Scan |

---

## Summary of Findings

| # | Finding | Severity |
|---|---------|----------|
| 1 | `factor_exposures` uses listwise deletion across **all** managers | Medium |
| 2 | `deflated_sharpe_ratio` reuses single-SR estimation variance as the cross-trial variance | Low (doc) |
| 3 | `_json_default` copy in `harness.py:640` (Day 1 four-copy item, still unconsolidated) | Low (cross-ref) |
| — | `deflated_sharpe.py` PSR/DSR correctness (incl. non-excess kurtosis) | ✅ Correct |
| — | `metrics/__init__.py` removed `builtins`/`sys.modules` pollution | ✅ Positive |
| — | `turnover.py` vectorised turnover + cost | ✅ Clean |

---

## Positive / Correct

### `deflated_sharpe.py` — statistically correct
The probabilistic Sharpe ratio matches Bailey–López de Prado:
`PSR = Φ((SR − SR*)·√(n−1) / √(1 − γ₃·SR + (γ₄−1)/4·SR²))` (`:37,70`). Crucially, `estimate_sharpe_moments` returns **non-excess** kurtosis (`clean.kurtosis() + 3.0`, `:128`) — pandas reports *excess* kurtosis, and the `+3` correctly matches the formula's `γ₄` convention (normal = 3). This is a subtle detail that is frequently gotten wrong, and it is right here. The expected-max-Sharpe deflation (`:99-102`) matches the published `√V·[(1−γ)Φ⁻¹(1−1/N) + γΦ⁻¹(1−1/(Ne))]`. Per-period (non-annualised) Sharpe is used throughout, which is correct for PSR.

### `metrics/__init__.py` — removed a `builtins` anti-pattern
The updated module deletes the previous machinery that injected names into `builtins` (`setattr(_bi, "annualize_return", …)`) and registered a synthetic `sys.modules["tests.legacy_metrics"]`, replacing it with a clean `__all__` and normal imports. This is a genuine improvement and removes interpreter-global pollution.

> Cross-reference: this is the *same* class of anti-pattern I flagged in `config/models.py` (`builtins._TREND_CONFIG_CLASS`, audit-02 finding ④, still unresolved). It has been cleaned up here in metrics but not in the config model — worth applying the same treatment there.

---

## Findings

### ① `factor_exposures` — listwise deletion across all managers — Medium

`factor_attribution.py:34`:
```python
valid_rows = aligned_returns.notna().all(axis=1) & aligned_factors.notna().all(axis=1)
```
A row is kept only if **every** manager column *and* every factor is non-NaN. So a single manager with a gap or a late inception date drops that period for **all** managers, and every manager is regressed on the same shrunken sample. The docstring states this is intentional ("fitting every manager on the same sample"), but for fund panels — where staggered inception is normal and pre-inception is even encoded as zeros→NaN (see `data.compute_inception_dates`) — this can:

- drastically shrink the usable sample, and/or
- raise `insufficient observations` (`:42`) for the whole panel because one manager is short.

**Recommendation:** fit each manager on its **own** valid overlap with the factors (per-manager dropna), rather than a global listwise deletion. If a shared sample is genuinely wanted, make it an explicit option and document the trade-off.

### ② `deflated_sharpe_ratio` — variance reuse — Low (documentation)

The deflation term uses `sharpe_variance` (the single-strategy SR *estimation* variance) as `V` in the expected-max-Sharpe computation (`:99-102`), whereas the DSR definition's `V` is the **cross-sectional variance of the N trial Sharpe ratios**. Under the null that all trials share the same true SR this is a standard approximation, but there is no parameter to pass the actual trial dispersion when it is known.

**Recommendation:** document that `V` is approximated by the single-SR estimation variance, and optionally accept an explicit `trial_sharpe_variance` argument for callers that have the real dispersion.

### ③ `_json_default` copy in `harness.py:640` — Low (Day 1 cross-ref)

`BacktestResult.to_json` uses a local `_json_default` (`:640`), one of the four copies flagged on Day 1 (also `trend/cli.py`, `walk_forward.py`, `core/rank_selection.py`). Still unconsolidated on phase-3.

---

## Verdict for Subsystem I

Good quality overall. `deflated_sharpe.py` is a correct, careful implementation of a genuinely tricky statistic, and `metrics/__init__.py` shows the codebase actively removing a `builtins` anti-pattern (which strengthens the case for doing the same in `config/models.py`). The one substantive finding is `factor_exposures`' global listwise deletion (①), which is a real robustness problem for staggered-inception fund data. The rest are a Low documentation note and the recurring `_json_default` duplication.

---

## Files Reviewed

- [x] `metrics/deflated_sharpe.py` (full — deep)
- [x] `metrics/factor_attribution.py` (full — deep)
- [x] `metrics/turnover.py` (full)
- [x] `metrics/__init__.py` (structure + registry; core metric formulas cross-checked in audit-05b)
- [x] `backtesting/harness.py` (structure, `CostModel`, `_json_default`)
- [~] `metrics/attribution.py`, `rolling.py`, `summary.py`, `backtesting/bootstrap.py` (scan)
