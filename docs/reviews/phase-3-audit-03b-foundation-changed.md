# Phase-3 First-Pass Audit 03b: Changed Foundation Modules

**Date:** 2026-07-16
**Reviewer:** Project review (first pass — not a verification)
**Branch:** `phase-3`
**Scope:** The four foundation modules that carry new logic since the Day 3 snapshot:
`time_utils.py`, `schedules.py`, `regimes.py`, `data.py`

> This is a genuine first-pass review (these modules were never reviewed in their current form), not a verification of prior recommendations.

---

## Summary of Findings

| # | Module | Finding | Severity |
|---|--------|---------|----------|
| 1 | `regimes.py` | `settings.model` not validated against the registry in `normalise_settings` | Medium |
| 2 | `regimes.py` | Volatility threshold/neutral_band now annualised — behavioral change vs prior | Medium (confirm) |
| 3 | `data.py` | `_canonicalise_date_column` wired into `load_csv` only, not `load_parquet` | Medium |
| 4 | cross-module | Frequency-normalisation logic now duplicated across ≥4 modules | Low |
| 5 | `time_utils.py` | Month-end boundary normalised to midnight excludes intraday end-day rows | Low |
| — | `regimes.py` | Volatility non-positive-threshold guard added | ✅ Positive |
| — | `time_utils.py` | Shared `resolve_period_bound` for validation + slicing | ✅ Positive |

---

## Positive Changes (worth calling out)

### `time_utils.py` — shared boundary resolver
New `is_month_label` (`:15`) and `resolve_period_bound` (`:21`) centralise sample-split boundary resolution so `config/validation.py` (ordering checks) and `stages/preprocessing.py` (window slicing) resolve a label to the **same instant**. This closes a real class of bug (validation and slicing disagreeing on what `"2020-01"` means). Good design.

### `regimes.py` — volatility threshold guard
`normalise_settings` now raises when `method="volatility"` and `threshold <= 0` (`regimes.py:88-93`), with an explanation that a non-positive threshold collapses the split to all-Risk-Off (since `signal = threshold - volatility` and volatility ≥ 0). This fixes a previously silent misconfiguration.

---

## Findings

### ① `regimes.py` — `model` not validated in `normalise_settings` — Medium

`normalise_settings` validates `method` via a lookup table with a safe fallback (`regimes.py:74-83`), but `model` is accepted as any non-empty string (`:70-72`). The value is only checked later when `compute_regimes` calls `regime_registry.create(settings.model)` (`:323`), which raises `ValueError: Unknown plugin: ...` (`plugins/__init__.py:46-49`).

**Impact:** a typo like `model: binary_treshold` passes settings normalisation and fails deep in `compute_regimes`, inconsistent with how `method` degrades gracefully.

**Recommendation:** validate `model` against `regime_registry.available()` in `normalise_settings` (fallback to `binary_threshold` or raise a clear config error early), mirroring the `method` handling.

### ② `regimes.py` — volatility threshold/band annualisation is a behavioral change — Medium (confirm intent)

New code annualises the threshold and neutral band to match the annualised volatility signal (`regimes.py:228-231`):

```python
if settings.annualise_volatility and periods and periods > 0:
    scale = float(np.sqrt(periods))
    threshold = threshold * scale
    neutral_band = neutral_band * scale
signal = threshold - signal
```

Previously the annualised volatility signal was compared against the **raw** threshold. This is now a different comparison. If users previously specified `threshold` already in annualised terms, this double-scales it; if they specified it per-period, the new behavior is correct.

**Recommendation:** confirm the intended unit of `regime.threshold` for the volatility method and document it. Add a test pinning the expected split for a known input so the interpretation can't silently drift again.

Note: the cache `method_tag` (`regimes.py:258-265`) keys on the **raw** `settings.threshold`/`neutral_band` plus `periods`, so annualisation stays deterministic per freq — no cache-collision concern.

### ③ `data.py` — date-column canonicalisation is CSV-only — Medium

New `_canonicalise_date_column` (`data.py:364`) renames a configured date column to `"Date"` before validation, and `load_csv` now accepts `date_column="Date"` (`:400`) and applies it (`:435`).

However, `load_parquet` (`:476`) neither accepts `date_column` nor calls `_canonicalise_date_column`. A config using a non-`"Date"` date column will work through the CSV path but fail (or silently mis-validate) through the Parquet path.

**Recommendation:** thread `date_column` through `load_parquet` and apply the same canonicalisation, or factor the shared prologue so both loaders behave identically.

### ④ Cross-module — frequency-normalisation logic proliferation — Low

Frequency alias handling now exists in at least four places with overlapping responsibility:
- `timefreq.py` (`MONTHLY_DATE_FREQ`/`MONTHLY_PERIOD_FREQ` constants + helpers)
- `time_utils.py::_resolve_frequency_tag`
- `schedules.py::normalize_frequency` (**new**)
- `util/frequency.py` (separate module)

`schedules.normalize_frequency` (`:54`) is well-written and conservative (correctly handles `M→ME`, `Q→QE`, `A/Y→YE`, numeric prefixes like `3M`, and avoids mangling `SM`/`BM`), but it adds a fourth home for the same concept.

**Recommendation:** consolidate frequency normalisation into one module (likely `timefreq.py` or `util/frequency.py`) and have the others delegate. Not urgent, but the drift risk is real.

### ⑤ `time_utils.py` — month-end boundary at midnight — Low

`resolve_period_bound(..., bound="end")` returns `period.end_time` then `.normalize()` to midnight (`:42-45`). For inclusive `<=` slicing, an end boundary of `"2020-01"` becomes `2020-01-31 00:00:00`, so any intraday timestamp on 2020-01-31 (e.g. `12:00`) would be excluded.

**Impact:** none for monthly/daily data stamped at midnight (the actual domain). Only matters if intraday timestamps are ever sliced. The docstring already notes the inclusive-`<=` intent.

**Recommendation:** none required now; note the assumption in case intraday data is introduced.

---

## Verdict

The changes to these four modules are, on balance, **improvements** — two of them (the shared boundary resolver and the volatility-threshold guard) fix real latent bugs. The open items are a validation-consistency gap (`model`), a behavioral change that needs an intent confirmation (volatility annualisation), and a loader asymmetry (`load_parquet` missing date-column canonicalisation).

No blocking defects found. Recommend addressing ① and ③ (both Medium, both cheap) and confirming ②.

---

## Files Reviewed

- [x] `src/trend_analysis/time_utils.py` (full)
- [x] `src/trend_analysis/schedules.py` (full)
- [x] `src/trend_analysis/regimes.py` (full)
- [x] `src/trend_analysis/data.py` (diff-focused: `_canonicalise_date_column`, `load_csv`, `load_parquet`)
