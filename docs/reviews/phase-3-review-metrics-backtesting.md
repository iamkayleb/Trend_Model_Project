# Metrics & Backtesting
**Scope:** `metrics/*`, `backtesting/*`

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `backtesting/harness.py` | 652 | `run_backtest`, `CostModel`, `BacktestResult`, execution-lag & membership handling |
| `metrics/__init__.py` | 439 | Metric registry, core metrics, legacy aliases, submodule re-exports |
| `metrics/attribution.py` | 134 | Brinson-style attribution **+ plotting** |
| `metrics/backtesting/bootstrap.py` | 131 | Block bootstrap |
| `metrics/deflated_sharpe.py` | 129 | PSR / deflated Sharpe |
| `metrics/factor_attribution.py` | 81 | OLS factor exposures |
| `metrics/turnover.py` | 72 | Realised turnover + linear cost |
| `metrics/rolling.py` | 67 | Rolling metrics |
| `metrics/summary.py` | 64 | Summary table |

Largest functions:

| Lines | Function | Location |
|------:|----------|----------|
| **229** | `run_backtest` | `backtesting/harness.py:144` |
| 72 | `bootstrap_equity` | `backtesting/bootstrap.py:60` |
| 70 | `factor_exposures` | `metrics/factor_attribution.py:9` |
| 60 | `sortino_ratio` | `metrics/__init__.py:250` |
| 55 | `_apply_membership_mask` | `backtesting/harness.py:455` |

---

## Positive / Clean

### No duplicated helpers within the subsystem
I checked for repeated helper definitions across all nine files and found **none**. That's a notable contrast with Monte Carlo, where 15 helper names are defined more than once. Whoever built this kept the shared pieces in one place.

### `metrics/__init__.py` removed a `builtins` anti-pattern
The module previously injected names into `builtins` (`setattr(_bi, "annualize_return", …)`) and registered a synthetic `sys.modules["tests.legacy_metrics"]` entry. Both are gone, replaced by an explicit `__all__`. That's real cleanup of interpreter-global pollution — and worth flagging because the *same* pattern still lives in `config/models.py` (`builtins._TREND_CONFIG_CLASS`). The fix is evidently understood; it just hasn't been applied everywhere.

### The new small modules are well-shaped
`deflated_sharpe.py` (129) and `factor_attribution.py` (81) are both focused, fully annotated, and declare `__all__`. `deflated_sharpe.py` splits validation into `_validate_moments` (`:14`) rather than inlining checks in each public function — small file, but properly factored.

### `harness.py` uses real types at its boundaries
`CostModel` (`:37`) and `BacktestResult` (`:90`) are dataclasses with methods (`apply`, `as_dict`, `summary`, `to_json`) rather than dicts passed around. The private helpers below are short and single-purpose.

### `turnover.py` states its contract
The docstring explicitly commits to being "vectorised and free of plotting dependencies so it can be used from the metrics package without requiring the `viz` helpers." Clear, deliberate, and testable as a claim — see finding ①.

---

## Findings

### ① Importing `metrics` drags in `matplotlib` — Medium

`metrics/__init__.py:415` eagerly imports the attribution submodule at package-import time:

```python
attribution = import_module(".attribution", __name__)
```

and `metrics/attribution.py:23` imports plotting at module level:

```python
import matplotlib.pyplot as plt
```

So `import trend_analysis.metrics` unconditionally imports `matplotlib.pyplot`, whether or not the caller ever plots anything.

This isn't theoretical — I hit it in this session. A script that only needed `compute_constrained_weights` failed like this:

```
File ".../trend_analysis/metrics/__init__.py", line 415, in <module>
    attribution = import_module(".attribution", __name__)
File ".../trend_analysis/metrics/attribution.py", line 23, in <module>
    import matplotlib.pyplot as plt
ModuleNotFoundError: No module named 'matplotlib'
```

The chain was `risk.py → engine → walkforward → metrics → attribution → matplotlib`. A numeric computation path couldn't run without a plotting library installed.

The irony is that `turnover.py` was deliberately written to avoid exactly this ("free of plotting dependencies so it can be used from the metrics package"), and the package `__init__` undoes that care for every consumer.

**Recommendation:** move the plotting function (`plot_contributions`, `attribution.py:91`) into `viz/`, or import matplotlib lazily inside it. If the eager submodule exposure is needed for back-compat, use a module `__getattr__` (PEP 562) so `metrics.attribution` resolves on first access instead of at import.

### ② `run_backtest` is 229 lines — Medium

`harness.py:144-372`. The rest of the file is well factored — 15-to-55-line private helpers, each doing one thing (`_enforce_execution_lag_calendar`, `_apply_membership_mask`, `_normalise_weights`, `_compute_metrics`). `run_backtest` is the outlier by a factor of four, and it holds the main loop plus setup and assembly inline.

Given how cleanly the helpers around it are cut, this looks like the one function that didn't get the same treatment rather than something inherently resistant to splitting.

**Recommendation:** extract setup, the per-rebalance loop body, and result assembly, following the helper style already established in the file.

### ③ Fourth copy of `_json_default` — Medium

`harness.py:640`, used by `BacktestResult.to_json` (`:141`). Repo-wide there are four:

```
trend/cli.py
trend_analysis/walk_forward.py
trend_analysis/backtesting/harness.py
trend_analysis/core/rank_selection.py
```

I flagged this on day one; it's unchanged. Four separate definitions of "how this codebase serialises numpy/pandas types to JSON" means four places that can disagree about the output format.

**Recommendation:** one `util/json_compat.py`, imported by all four.

### ④ Imports placed mid-file to satisfy the linter — Low

`metrics/__init__.py:405-411` puts imports 400 lines into the module, each carrying a suppression:

```python
from .deflated_sharpe import (  # noqa: E402
    deflated_sharpe_ratio, estimate_sharpe_moments, probabilistic_sharpe_ratio,
)
from .factor_attribution import factor_exposures  # noqa: E402
```

and `:413-419` uses `import_module` with the comment *"exposed via attribute assignment for compatibility while keeping Ruff satisfied about unused imports."*

Both are the code being shaped to appease the linter rather than the linter being configured for the code. The `import_module` calls in particular are just imports written in a way that the unused-import rule can't see — which also means static analysis and IDEs can't see them either.

**Recommendation:** move the imports to the top; if they're genuinely re-exports, list them in `__all__` (already done) and add a targeted per-file Ruff ignore for the re-export rule rather than five `noqa`s and an `import_module` workaround.

### ⑤ Legacy aliases with no deprecation path — Low

`:400-404` defines five back-compat aliases:

```python
annualize_return = annual_return
annualize_volatility = volatility
annualize_sharpe_ratio = sharpe_ratio
annualize_sortino_ratio = sortino_ratio
info_ratio = information_ratio  # ← old short name
```

The module docstring says these are "kept for back-compat with the test-suite" (`:5`). There are zero `DeprecationWarning`s in the file, and all five are exported in `__all__` — so they read as equally-supported public API rather than legacy.

If they exist only for the test suite, that's a reason to update the tests, not to carry a permanent second name for every metric.

**Recommendation:** either mark them deprecated (module `__getattr__` emitting `DeprecationWarning`) and drop them from `__all__`, or update the callers and delete them.

---

## Verdict

**The best-organised subsystem I've reviewed so far.** No internal helper duplication, real dataclasses at the boundaries, small focused modules, and evidence of active cleanup — the `builtins` pollution that still exists in `config/models.py` has already been removed here.

The findings are narrow. ① is the one worth acting on: a metrics package that can't be imported without a plotting library is a genuine coupling problem, it's already breaking a real import path, and it directly contradicts a contract another module in the same package wrote down. ② and ③ are ordinary refactors. ④ and ⑤ are tidiness — but ④ is worth doing because the `import_module` workaround actively hides structure from tooling.

Order: ① first (small change, removes a hard dependency from a numeric path), then ③ since it spans four files and gets harder as they drift, then ② and the tidy-ups.
