# Day 9 Review: `backtesting/`, `io/`, and `util/` subpackages

**Scope:** 17 files, ~3,416 lines — backtest harness, market data validation, and small utilities
**Date:** 2026-05-08
**Reviewer:** Claude

---

## Files Reviewed

- `backtesting/__init__.py` — Re-exports `BacktestResult`, `CostModel`, `run_backtest`, and `bootstrap_equity`. Clean.
- `backtesting/bootstrap.py` — `bootstrap_equity`: circular block bootstrap producing 5/50/95th-percentile equity bands aligned to the original calendar.
- `backtesting/harness.py` — `run_backtest`: walk-forward backtester with membership masking, execution-lag enforcement, turnover capping, and per-rebalance cost modelling. Houses `BacktestResult`, `CostModel`, and the private compute helpers. See issues below — there's a real numerical bug in `_compute_metrics`.
- `io/__init__.py` — Re-exports from `market_data.py` and `validators.py`. Acts as the public surface for IO.
- `io/date_correction.py` — `analyze_date_column` / `apply_date_corrections` / `format_corrections_for_display`: UI-oriented date-error analysis that classifies failures as correctable, trailing-empty, or unfixable before the user confirms a fix.
- `io/market_data.py` — `validate_market_data` and helpers: date parsing, numeric coercion, missing-data policy application, frequency inference, mode (price vs returns) detection. The longest file in the group at 1,002 lines.
- `io/ui_ingest.py` — `load_ui_dataset` / `inspect_ui_date_issues`: thin orchestration that runs `date_correction` analysis, applies fixes if asked, and then delegates to `validate_market_data`.
- `io/utils.py` — `export_bundle` / `cleanup_bundle_file`: assembles a ZIP archive of analysis outputs into a temporary file and registers it for cleanup on exit.
- `io/validators.py` — `validate_returns_schema`, `detect_frequency`, `load_and_validate_upload`, `create_sample_template`: backwards-compatibility wrappers and legacy entry points.
- `util/__init__.py` — Module docstring only; callers import submodules directly.
- `util/frequency.py` — `detect_frequency`: returns a `FrequencySummary` describing daily/weekly/monthly/quarterly/annual cadence.
- `util/hash.py` — `sha256_bytes`/`sha256_text`/`sha256_file`/`sha256_config` and `normalise_for_json`: deterministic hashing utilities.
- `util/joblib_shim.py` — `dump`/`load`: thin pickle wrappers used where the full joblib dependency is unnecessary. Filename is deliberately not `joblib.py` so `import joblib` still resolves to the real package.
- `util/missing.py` — `apply_missing_policy`: column-by-column missing-data enforcement returning a typed `MissingPolicyResult`. The version imported by `pipeline.py`, `stages/preprocessing.py`, and `multi_period/engine.py`.
- `util/risk_free.py` — `resolve_risk_free_settings`: extracts `risk_free_column` and `allow_risk_free_fallback` from a data-settings mapping. Eight lines of logic.
- `util/rolling.py` — `rolling_shifted`: rolling aggregations after a one-period lag to prevent same-day look-ahead.
- `util/weights.py` — `normalize_weights`: converts a weight mapping or Series to fractions, auto-detecting percent-scale inputs.

---

## Issues Found

### 1. Sortino in `harness._compute_metrics` is missing a `√periods_per_year` factor

`_compute_metrics` (lines 633–641) computes annualized volatility, Sharpe, and Sortino. The Sharpe formula is annualized correctly:

```python
vol = active_returns.std(ddof=0) * np.sqrt(periods_per_year)   # annualized
downside_std = downside.std(ddof=0) * np.sqrt(periods_per_year)   # annualized
sharpe = active_returns.mean() / active_returns.std(ddof=0) * np.sqrt(periods_per_year)
sortino = active_returns.mean() / downside_std if downside_std else float("nan")
```

For Sharpe, `active_returns.std(ddof=0)` (per-period) is divided into the per-period mean, then multiplied by `√ppy` — standard annualization.

For Sortino, the per-period mean is divided by `downside_std`, which is already annualized (multiplied by `√ppy` on line 635). The result is `mean_per_period / (downside_std_per_period × √ppy)`, which is the per-period Sortino divided by `√ppy` — i.e., **deannualized**. To match Sharpe's annualization and the canonical implementation in `metrics/__init__.py::sortino_ratio` (which uses `annual_return / annualized_downside_vol`), the formula should be:

```python
sortino = active_returns.mean() / downside.std(ddof=0) * np.sqrt(periods_per_year)
```

For monthly data (`ppy=12`), the current formula yields a Sortino roughly `√12 ≈ 3.46×` smaller than the canonical one. Tests don't catch this because `tests/backtesting/test_harness.py` only checks that the `"sortino"` key exists in the metrics dict — never the value.

**Recommendation:** Multiply by `np.sqrt(periods_per_year)` and use the per-period downside std, mirroring the Sharpe formula. Add a value assertion to the existing harness tests so the regression doesn't recur.

---

### 2. `util/missing.py::_coerce_policy` silently aliases `bfill`/`backfill` to `ffill`

In `util/missing.py` at lines 73–81:

```python
def _coerce_policy(policy: str | None) -> str:
    value = (policy or "drop").strip().lower()
    if value in {"both", "bfill", "backfill"}:
        value = "ffill"
    if value in {"zeros", "zero_fill", "fillzero"}:
        value = "zero"
    if value not in {"drop", "ffill", "zero"}:
        raise ValueError(f"Unsupported missing-data policy: {policy!r}")
    return value
```

A user who configures `missing_policy: "bfill"` expecting backward-fill silently gets forward-fill instead, with no warning. The canonical `apply_missing_policy` implementation has no actual `bfill` code path — only `drop`, `ffill`, and `zero`.

For comparison, `io/market_data.py::_normalise_policy_value` is strict and raises on any value outside `{"drop", "ffill", "zero"}`. So the same configuration string is silently rewritten in one module and rejected in the other. Tests that pass `"bfill"` (e.g. `tests/test_multi_period_engine_missing_policy.py:93`) monkeypatch `apply_missing_policy` and never reach the real `_coerce_policy`, so the aliasing is invisible to the test suite.

**Recommendation:** Remove the silent aliases. Either implement actual `bfill` (a `series.bfill(limit=...)` call) or have `_coerce_policy` raise on `bfill`/`backfill` like the io version does. The `both` and `zero_fill` aliases are similarly suspect — drop them unless they're documented in the user-facing config schema.

---

### 3. Two `apply_missing_policy` functions with different contracts

`util/missing.py::apply_missing_policy` and `io/market_data.py::apply_missing_policy` share the same name but are not interchangeable:

| | `util/missing.py` | `io/market_data.py` |
|---|---|---|
| Return type | `tuple[pd.DataFrame, MissingPolicyResult]` | `tuple[pd.DataFrame, Dict[str, Any]]` |
| Extra params | `columns`, `enforce_completeness` | — |
| Wildcard key | `"default"` | `"*"` |
| Leading NaN handling | `ffill` only | `ffill` then `bfill` |
| Accepts `bfill` | Yes (silently → `ffill`) | No (raises) |

The pipeline, stages, and multi-period engine import from `util/missing.py`. The IO validators (`validate_market_data`) use `io/market_data.py`. The two are not interchangeable — switching between them changes both the return shape and the behavior on series with leading NaNs (column kept vs dropped). A configuration dict written for one cannot be reused for the other because the wildcard key differs.

**Recommendation:** Rename `io/market_data.py::apply_missing_policy` to something private and IO-specific (e.g. `_apply_io_missing_policy`) and remove it from `io/__init__.py.__all__`. Pick one wildcard convention (`"default"` matches the test fixtures) and standardise on it. Document the leading-NaN handling difference if it's deliberate.

---

### 4. `_infer_periods_per_year` is duplicated between `backtesting/harness.py` and `engine/walkforward.py`

Both files define `_infer_periods_per_year(index: pd.DatetimeIndex) -> int` with near-identical bodies: compute median nanosecond diff, convert to days, classify into 252/52/12/4/1. Neither uses the canonical `risk.py::periods_per_year_from_code` (which takes a code string, not a date index).

The two copies have small drifts:
- `engine/walkforward.py:117` adds an extra `if median_days <= 0: return 1` guard before the `approx` calculation.
- `harness.py:574` adds a `max(1, approx)` floor at the end.
- `engine/walkforward.py` is missing the `max(1, approx)` floor.

**Recommendation:** Move the function to `util/frequency.py` (which already owns frequency-related logic) and import it from both call sites. Reconcile the small differences (the `max(1, approx)` floor is the safer behaviour).

---

### 5. Date-fix logic is duplicated between `io/market_data.py` and `io/date_correction.py`

`io/market_data.py::_fix_invalid_day` (lines 67–112) and `io/date_correction.py::_try_correct_date` (lines 97–129) both fix the same problem — a day number that exceeds the month's last day — but with different scope:

- `_fix_invalid_day` handles `M/D/YYYY` and `YYYY-MM-DD` (2 formats); clamps to `max_day` with no overflow tolerance.
- `_try_correct_date` handles 5 formats including European variants; only corrects when the day is within 3 of `last_day` (so e.g. November 35 is left unfixed).

Both run on every UI ingest. `load_ui_dataset` calls `analyze_date_column` first (which uses `_try_correct_date`), then `validate_market_data` (which calls `_auto_fix_invalid_dates` → `_fix_invalid_day`). When the first pass already fixed the date, the second pass is a no-op — but a fix added to one function will silently fail to apply when the data is routed through the other.

**Recommendation:** `date_correction.py` is the more comprehensive implementation. Have `validate_market_data` either call into it directly or skip its own correction pass when called from `ui_ingest.py` (passing `auto_fix_dates=False`, which is already the convention there).

---

### 6. `_read_uploaded_file` in `validators.py` has copy-pasted exception handling

`_read_uploaded_file` (lines 175–245) has two near-identical 14-line blocks of exception handling: one wrapping the `.read()` path (lines 211–224) and another wrapping the fallthrough CSV path (lines 230–243). Each catches `FileNotFoundError`, `PermissionError`, `IsADirectoryError`, `pd.errors.EmptyDataError`, `pd.errors.ParserError`, and a generic `Exception`, converting all of them to `ValueError`.

The fallthrough path itself is also subtly opaque: it's only reached when `file_like` is neither a `str`/`Path` nor has a `.read()` attribute, which is an unusual case, and yet calls `pd.read_csv(file_like)` — an operation pandas would only accept on a path-like or file-like object.

**Recommendation:** Extract the five-exception mapping into a helper (e.g. `_wrap_read_errors(exc, source)`) and call it from both branches. Add a comment explaining what type of `file_like` the fallthrough path handles.

---

## Notes (not actionable)

### `_normalise_delta_days` is an immediate wrapper around `_normalize_delta_days`

In `io/market_data.py`, line 54 defines `_normalise_delta_days` whose body is a single line: `return _normalize_delta_days(delta_days)`. The actual implementation is at line 494 (American spelling). The wrapper is called once, by `classify_frequency` at line 529. Either the wrapper is redundant or there's a planned migration that never landed; either way it can be deleted.

### `util/frequency.py::_classify_from_diffs` has a 4.0–4.5 day gap between buckets

The classifier uses `daily = (diffs_days > 0) & (diffs_days <= 4.0)` and `weekly = (diffs_days >= 4.5) & (diffs_days <= 9.0)`. Spacings between 4.0 and 4.5 days fall into no bucket. If many spacings land in this gap (uncommon in practice but conceivable for irregular weekend-crossing weekly data) the classifier produces all-zero counts and raises "Unable to determine series frequency". Easy to fix by extending one of the bounds (e.g. `daily <= 4.5`).

### `util/weights.py::normalize_weights` has a no-op `elif` branch

Lines 35–38:

```python
if total_abs and np.isclose(total_abs, 100.0, ...):
    series = series / 100.0
elif total_abs and np.isclose(total_abs, 1.0, ...):
    series = series   # no-op
```

The `elif` branch reassigns `series` to itself. The intent appears to be "if it already sums to 1.0, leave it alone" — which is what would happen if the entire branch were deleted. The branch isn't harmful but is a code smell.

### `io/utils.py::export_bundle` wraps only two of four file writes

`portfolio_returns.csv` and `event_log.csv` are wrapped in `try/except: write empty file`. `summary.json` and `config.json` are written without protection. `event_log_df()` is also called outside the try block (line 59), so a failing `event_log_df()` propagates while a failing `to_csv` is silenced. The protection is half-applied; either wrap all four (with a `logger.warning` so silent failures aren't invisible) or remove the partial wrapping.

### `harness._first_index_position` and `_last_index_position` have asymmetric ndarray handling

`_last_index_position` (lines 378–393) explicitly checks `loc.dtype == bool` before deciding how to extract the position. `_first_index_position` (lines 367–375) just calls `np.argmax(loc)` regardless of dtype. Both branches are reachable only from non-monotonic indices with duplicate labels, which the project doesn't normally produce, so this is dead code in practice — but the asymmetry would matter if a non-boolean ndarray ever surfaced.

---

## No Issues (clean files)

- `backtesting/__init__.py` — clean
- `backtesting/bootstrap.py` — clean; circular block bootstrap is implemented correctly
- `io/__init__.py` — clean
- `io/ui_ingest.py` — clean; formula-header sanitization and BOM stripping are good defensive habits
- `util/__init__.py` — clean
- `util/hash.py` — clean; `normalise_for_json` correctly handles pydantic models, mappings, and sequences
- `util/joblib_shim.py` — clean; correctly named so `import joblib` resolves to the real package
- `util/risk_free.py` — clean
- `util/rolling.py` — clean

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| High | Fix Sortino formula in `_compute_metrics` to multiply by `√periods_per_year`; add a value assertion in tests | `backtesting/harness.py` |
| Medium | Remove silent `bfill`/`backfill` → `ffill` aliasing in `_coerce_policy`; either implement bfill or raise like the io version | `util/missing.py` |
| Medium | Make `io/market_data.py::apply_missing_policy` private; standardize wildcard key on `"default"`; document leading-NaN handling | `io/market_data.py`, `io/__init__.py` |
| Low | Move `_infer_periods_per_year` to `util/frequency.py`; import from both call sites; reconcile the floor/guard differences | `backtesting/harness.py`, `engine/walkforward.py`, `util/frequency.py` |
| Low | Consolidate the two date-fix implementations into `date_correction.py`; have `market_data.py` defer to it | `io/market_data.py`, `io/date_correction.py` |
| Low | Extract the shared exception-to-`ValueError` mapping in `_read_uploaded_file` | `io/validators.py` |
