# Day 9 Review: `backtesting/`, `io/`, and `util/` subpackages

**Scope:** 17 files, ~3,416 lines — backtest harness, market data validation, and small utilities
**Date:** 2026-05-08
**Reviewer:** Claude

---

## Files Reviewed

- `backtesting/__init__.py` — Re-exports `BacktestResult`, `CostModel`, `run_backtest`, and `bootstrap_equity`. Clean.
- `backtesting/bootstrap.py` — `bootstrap_equity`: circular block bootstrap for equity curve uncertainty bands. Returns a DataFrame with `p05`, `median`, and `p95` columns aligned to the backtest calendar.
- `backtesting/harness.py` — `run_backtest`: walk-forward backtest engine with membership masking, execution lag enforcement, turnover capping, and per-rebalance cost modelling. Also houses `BacktestResult`, `CostModel`, and the private helpers that support them.
- `io/__init__.py` — Re-exports from `market_data.py` and `validators.py`. Acts as the public surface for IO.
- `io/date_correction.py` — `analyze_date_column` / `apply_date_corrections` / `format_corrections_for_display`: UI-oriented date-error analysis that classifies failures as correctable, trailing-empty, or unfixable before the user confirms a fix.
- `io/market_data.py` — `validate_market_data` and its helpers: date parsing, numeric coercion, missing-data policy application, frequency inference, and mode (price vs returns) detection. The longest and most dense file in the group at 1,002 lines.
- `io/ui_ingest.py` — `load_ui_dataset` / `inspect_ui_date_issues`: thin orchestration layer that runs `date_correction` analysis, applies fixes if the caller asked for auto-correction, and then delegates to `validate_market_data`.
- `io/utils.py` — `export_bundle` / `cleanup_bundle_file`: assembles a ZIP archive of analysis outputs into a temporary file and registers it for cleanup on process exit.
- `io/validators.py` — `validate_returns_schema`, `detect_frequency`, `load_and_validate_upload`, `create_sample_template`: backwards-compatibility wrappers and legacy entry points that delegate to `market_data.py`.
- `util/__init__.py` — Module docstring only; no re-exports, leaving callers to import submodules directly.
- `util/frequency.py` — `detect_frequency`: detects date cadence (daily / weekly / monthly / quarterly / annual) from a date index by combining `pd.infer_freq` with a bucket-majority fallback. Returns a `FrequencySummary` dataclass.
- `util/hash.py` — `sha256_bytes`, `sha256_text`, `sha256_file`, `sha256_config`, `normalise_for_json`: deterministic hashing and JSON normalisation utilities.
- `util/joblib_shim.py` — `dump` / `load`: thin pickle wrappers used in place of joblib for contexts where the full joblib dependency is unnecessary.
- `util/missing.py` — `apply_missing_policy`: column-by-column missing-data enforcement returning a typed `MissingPolicyResult`. This is the version imported by `pipeline.py`, `stages/preprocessing.py`, and `multi_period/engine.py`.
- `util/risk_free.py` — `resolve_risk_free_settings`: extracts `risk_free_column` and `allow_risk_free_fallback` from a data-settings mapping. Eight lines of logic.
- `util/rolling.py` — `rolling_shifted`: applies rolling aggregations after a one-period lag to prevent same-day look-ahead.
- `util/weights.py` — `normalize_weights`: converts a weight mapping or Series to fractions, auto-detecting percent-scale inputs.

---

## Issues Found

### 1. `apply_missing_policy` exists in two modules with different contracts

`util/missing.py::apply_missing_policy` and `io/market_data.py::apply_missing_policy` share the same name but are not the same function:

| | `util/missing.py` | `io/market_data.py` |
|---|---|---|
| Return type | `tuple[pd.DataFrame, MissingPolicyResult]` | `tuple[pd.DataFrame, Dict[str, Any]]` |
| Extra params | `columns`, `enforce_completeness` | — |
| Leading NaN handling | `ffill` only (leading NaNs may survive) | `ffill` then `bfill` (leading NaNs filled) |
| Valid policies | `drop`, `ffill`, `zero` | `drop`, `ffill`, `zero` |

The core of the project uses `util/missing.py` — `pipeline.py`, `stages/preprocessing.py`, and `multi_period/engine.py` all import from there. The IO validators use `io/market_data.py` internally, which is only exposed through `io/__init__.py`. The two functions are not interchangeable: tests that monkeypatch `mp_engine.apply_missing_policy` are patching the `util/missing` version, so any attempt to consolidate must preserve that module-level binding.

The behavioral difference is real: `util/missing.py` does not backfill leading NaNs after ffill (so a fund with no data in the first few months could survive the drop check with NaN leading values), while `io/market_data.py` adds a `bfill` pass. If a caller switches between the two versions they get different results on series with leading NaNs.

**Recommendation:** Give the two functions distinguishable names — e.g. keep `util/missing.py` as `apply_missing_policy` (the engine-facing function) and rename `io/market_data.py`'s version to `_apply_io_missing_policy` (private; only used internally by `validate_market_data`). Remove it from `io/__init__.py`'s `__all__`. Callers wanting missing-data logic should go through `util/missing.py`.

---

### 2. `_infer_periods_per_year` is copied between `backtesting/harness.py` and `engine/walkforward.py`

Both files define a private `_infer_periods_per_year(index: pd.DatetimeIndex) -> int` that derives periods-per-year from median date spacing. The logic is identical: compute median nanosecond diff, convert to days, classify into 252/52/12/4/1. Neither file imports the other or calls the canonical `risk.py::periods_per_year_from_code` (which maps frequency codes to PPY but doesn't inspect actual date spacing).

The two copies have drifted slightly:
- `engine/walkforward.py:117` guards `if median_days <= 0: return 1` as a separate check; `harness.py:565` folds this into the conditional expression `approx = int(...) if median_days else 1`.
- `engine/walkforward.py` doesn't include a `max(1, approx)` floor at the end.

**Recommendation:** Move `_infer_periods_per_year` to `util/frequency.py` (which already owns frequency-related logic), export it from there, and replace both copies with the import. Or, if the distinction between "infer from actual dates" and "look up from code" is worth preserving in the API, add a thin wrapper in `util/frequency.py` that calls `detect_frequency` and maps the result to an integer PPY.

---

### 3. `_normalise_delta_days` is an immediate wrapper around `_normalize_delta_days`

In `io/market_data.py`:

```python
# line 54 — British spelling wrapper
def _normalise_delta_days(delta_days: pd.Series) -> pd.Series:
    return _normalize_delta_days(delta_days)    # delegates to line 494

# line 494 — American spelling (actual implementation)
def _normalize_delta_days(delta_days: pd.Series) -> pd.Series:
    cleaned = delta_days.replace([np.inf, -np.inf], np.nan).dropna()
    return cleaned.astype(float)
```

`_normalise_delta_days` is called once (by `classify_frequency` at line 529). `_normalize_delta_days` is never called directly from outside this file. The wrapper exists because the function was renamed at some point and the old name was kept as an alias rather than updating the call site.

**Recommendation:** Delete `_normalise_delta_days`, rename the surviving function to whichever spelling matches the project convention, and update the one call site.

---

### 4. Date-fix logic is duplicated between `io/market_data.py` and `io/date_correction.py`

`io/market_data.py::_fix_invalid_day` (lines 67–112) and `io/date_correction.py::_try_correct_date` (lines 97–129) both fix the same problem: a date whose day number exceeds the last day of that month (e.g. November 31 → November 30). The implementations handle different format sets:

- `_fix_invalid_day`: handles `M/D/YYYY` and `YYYY-MM-DD` (2 formats); clamps to `max_day` with no tolerance on how far over it is.
- `_try_correct_date`: handles 5 formats including European variants; only corrects if the day is within 3 of `last_day` (so e.g. November 35 would be left unfixed).

Both run on every UI ingest. `load_ui_dataset` calls `analyze_date_column` first (which uses `_try_correct_date`), then calls `validate_market_data` (which calls `_auto_fix_invalid_dates` → `_fix_invalid_day`). When the first pass already fixed the date, the second pass is a no-op — but if the first pass left something unfixed (outside the ±3 tolerance), the second pass can still catch it.

The duplication means the two functions can drift: someone who adds a new date format to `_try_correct_date` won't think to add it to `_fix_invalid_day`, and vice versa.

**Recommendation:** `date_correction.py` already has the fuller implementation. Either have `market_data.py::_auto_fix_invalid_dates` call into `date_correction.py::_try_correct_date` (making the latter the single source), or eliminate the two-pass approach entirely by having `validate_market_data` trust that UI ingest has already corrected dates when `auto_fix_dates=False` is passed.

---

### 5. `_read_uploaded_file` in `validators.py` has copy-pasted exception handling

`_read_uploaded_file` (lines 175–245) has two nearly-identical blocks of exception handling: one wrapping the `.read()` path (lines 211–224) and another wrapping the fallthrough CSV path (lines 230–243). Each catches `FileNotFoundError`, `PermissionError`, `IsADirectoryError`, `pd.errors.EmptyDataError`, `pd.errors.ParserError`, and generic `Exception`, converting them all to `ValueError`. The two blocks are character-for-character identical except for the surrounding structure.

The fallthrough path (lines 226–243) is also subtly confusing: it is only reached when `file_like` is neither a `str`/`Path` nor has a `.read()` attribute — an unusual case — yet it calls `pd.read_csv(file_like)`, which pandas would only accept on a path-like or file-like object, so this path is hard to exercise in practice.

**Recommendation:** Extract the five-exception mapping into a helper (e.g. `_wrap_read_errors(exc, source)`) and call it from both branches. Clarify with a comment what object type the fallthrough path is designed to handle.

---

## Notes (not actionable)

### `io/utils.py::export_bundle` silently swallows write errors

Lines 53–56 and 62–66 catch `Exception` silently when writing `portfolio_returns.csv` and `event_log.csv`, falling back to writing empty files without logging. For a UI export path, survivability is the right tradeoff, but the silent fallback means a corrupted `results` object produces a silently empty bundle that looks valid from the outside. A `logger.warning` call before the fallback would make these failures visible.

### `backtesting/harness.py::_rolling_sharpe` uses `ddof=0` for both `.mean()` and `.std()`

The rolling Sharpe computed during backtesting (line 609: `rolling_obj.std(ddof=0)`) uses population std. `_compute_metrics` also uses population std (line 633), so the two are consistent within the file. Worth knowing if comparing against the `stages/portfolio.py` Sharpe, which uses a different convention.

### `util/frequency.py` and `io/market_data.py` both classify date frequency

`util/frequency.py::detect_frequency` returns a `FrequencySummary`; `io/market_data.py::classify_frequency` returns a `Dict[str, Any]`. They use different interval thresholds (e.g. `util` treats weekly as 4.5–9 days, `io` treats it as ≤10 days) and target different caller needs. Given they're used in distinct contexts (the pipeline vs market data validation), the duplication may be intentional. Worth keeping in mind if either is ever extended.

### `util/missing.py::MissingPolicyResult` implements `Mapping` via `_mapping` dict

`MissingPolicyResult` is a frozen dataclass that also implements `Mapping[str, Any]` by maintaining an internal `_mapping` dict built in `__post_init__`. The mapping includes both `"policy"` (same as `.policy`) and `"policy_map"` (also same as `.policy`) to support legacy callers that expect the dict-style key names. The double representation increases the chance of the two views drifting if the class is extended.

---

## No Issues (clean files)

- `backtesting/__init__.py` — clean
- `backtesting/bootstrap.py` — clean, circular block bootstrap is straightforward
- `backtesting/harness.py` — clean; execution-lag enforcement and membership masking are carefully guarded
- `io/__init__.py` — clean
- `io/ui_ingest.py` — clean; formula-header sanitization and binary BOM stripping are good defensive habits
- `util/__init__.py` — clean
- `util/hash.py` — clean; `normalise_for_json` handles pydantic models, mappings, and sequences correctly
- `util/joblib_shim.py` — clean; correctly named to avoid shadowing the real `joblib`
- `util/missing.py` — clean; `MissingPolicyResult` is well-designed as a typed container
- `util/risk_free.py` — clean
- `util/rolling.py` — clean
- `util/weights.py` — clean; percent-scale detection is conservative (within 1e-2 of 100)

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| Medium | Rename `io/market_data.py::apply_missing_policy` to avoid collision with `util/missing.py`; remove it from `io/__init__.py.__all__` | `io/market_data.py`, `io/__init__.py` |
| Low | Extract `_infer_periods_per_year` to `util/frequency.py` and import it from `backtesting/harness.py` and `engine/walkforward.py` | `backtesting/harness.py`, `engine/walkforward.py`, `util/frequency.py` |
| Low | Delete `_normalise_delta_days` (the wrapper); keep `_normalize_delta_days` and update the one call site | `io/market_data.py` |
| Low | Consolidate the two date-fix implementations into a single function in `date_correction.py` | `io/market_data.py`, `io/date_correction.py` |
| Low | Extract the shared exception-to-ValueError mapping in `_read_uploaded_file` into a helper | `io/validators.py` |
