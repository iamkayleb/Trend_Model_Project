# Phase-3 First-Pass Audit 04: Data Ingestion & I/O

**Date:** 2026-07-16
**Reviewer:** Project review (first pass — not a verification)
**Branch:** `phase-3`
**Scope:** `src/data/contracts.py`, `src/trend_analysis/io/*`, and (re-check) `src/trend/input_validation.py`, `src/trend/validation.py`

> Mostly new territory. `trend/input_validation.py` and `trend/validation.py` were reviewed on Day 1 and marked clean (no action items); they are re-checked here only where they interact with the `io/` package.

---

## Subsystem Summary

| File | Lines | Purpose |
|------|-------|---------|
| `io/market_data.py` | 995 | Core validation: mode/frequency detection, missing-policy, date auto-fix, metadata |
| `data/contracts.py` | 236 | Price ingest contract (index/UTC/monotonic/frequency/non-negative) |
| `io/validators.py` | 281 | Legacy validator shims; `ValidationResult`, upload loading |
| `io/date_correction.py` | 281 | Standalone date-analysis/correction (`analyze_date_column`, `apply_date_corrections`) |
| `io/ui_ingest.py` | 240 | UI-aligned ingestion (header sanitization, formula-injection guard) |
| `io/utils.py` | 104 | Bundle export/cleanup helpers |
| `trend/input_validation.py` | 384 | CSV ingestion validation (Day 1) |
| `trend/validation.py` | 147 | Price-frame schema enforcement (Day 1) |

---

## Summary of Findings

| # | Finding | Severity |
|---|---------|----------|
| 1 | `_fix_invalid_day` byte-identical in two modules | Medium |
| 2 | Three overlapping date-correction code paths | Medium |
| 3 | `ValidationResult` name collision (Pydantic vs plain class) | Medium |
| 4 | `_normalise_delta_days` / `_normalize_delta_days` spelling twins | Low |
| 5 | `_auto_fix_invalid_dates` positional-index handling is fragile | Low |
| — | `apply_missing_policy` correctness | ✅ Clean |
| — | `contracts.validate_prices` layered checks | ✅ Clean |

---

## Findings

### ① `_fix_invalid_day` duplicated byte-for-byte — Medium

The same function exists in two modules and is identical except for one comment:

- `src/trend_analysis/io/market_data.py:60`
- `src/trend/input_validation.py:117`

Both handle the same two formats (`M/D/YYYY`, `YYYY-MM-DD`) and clamp an out-of-range day to `calendar.monthrange(...)[1]`. This is the classic copy that drifts silently — a fix to one (e.g. supporting `D-M-YYYY`) won't reach the other.

**Recommendation:** hoist a single `_fix_invalid_day` into a shared module (e.g. `io/date_correction.py`, which is already the date-fixing home) and import it in both call sites.

### ② Three overlapping date-correction code paths — Medium

Invalid-date handling is implemented three times, with overlapping responsibility:

- `market_data.py`: `_fix_invalid_day` + `_auto_fix_invalid_dates` (`:60`, `:108`) — auto-fix during validation
- `trend/input_validation.py`: `_fix_invalid_day` + `correct_invalid_dates` (`:117`, `:163`)
- `io/date_correction.py`: `_try_correct_date` + `analyze_date_column` + `apply_date_corrections` (`:97`, `:166`, `:216`) — a richer, standalone implementation

Three implementations of "detect and correct malformed dates" is a maintenance and consistency hazard: the same malformed input can be corrected differently (or dropped vs. fixed) depending on which entry point is used.

**Recommendation:** designate `io/date_correction.py` as the single source of truth and have `market_data` and `input_validation` delegate to it. At minimum, share `_fix_invalid_day` (finding ①).

### ③ `ValidationResult` name collision — Medium

Two unrelated classes share the name:

- `src/trend_analysis/config/validation.py:32` — `class ValidationResult(BaseModel)` (Pydantic; `valid`, `errors`, `warnings`)
- `src/trend_analysis/io/validators.py:64` — `class ValidationResult` (plain class; `is_valid`, `issues`, `warnings`, `frequency`, `metadata`)

Different fields (`valid` vs `is_valid`; `errors` vs `issues`), different backends, same name. Any module importing both must alias, and a caller expecting one shape will `AttributeError` on the other. This echoes the Day 2 `ValidationError` collision that also remains unresolved.

**Recommendation:** rename the `io/validators.py` one to something intent-revealing (e.g. `SchemaValidationReport` or `UploadValidationResult`).

### ④ `_normalise_delta_days` / `_normalize_delta_days` spelling twins — Low

`market_data.py:47` defines `_normalise_delta_days` whose entire body is `return _normalize_delta_days(delta_days)` — a British-spelling alias of the real function at `:487`. Harmless but confusing; two near-identical names invite calling the wrong one.

**Recommendation:** collapse to one name, or make it an explicit alias assignment (`_normalise_delta_days = _normalize_delta_days`) with a comment.

### ⑤ `_auto_fix_invalid_dates` positional-index handling — Low

At `market_data.py:136-137`:
```python
for idx in working.index[invalid_mask]:
    pos = working.index.get_loc(idx) if not isinstance(idx, int) else idx
```
If the frame has a non-unique index, `get_loc` returns a slice/array and `pos + 1` (`:145`) breaks; `working.at[idx, date_col] = fixed` would also write to multiple rows. For the normal unique-index ingest path this is fine, but the guard `isinstance(idx, int)` conflates "positional int" with "integer label."

**Recommendation:** iterate positionally (`enumerate(working.index[invalid_mask])` over positions) or assert index uniqueness before entering the loop.

---

## Clean / Positive

### `apply_missing_policy` (`market_data.py:354`) — correct
The gap logic is sound: it computes `max_consecutive_nans` per column, drops the column when the max gap exceeds the limit, otherwise `ffill(limit)` + `bfill(limit)` and drops only if residual NaNs remain. `drop`/`ffill`/`zero` are all handled, with an explicit `raise` on an unknown policy. No issues found.

### `contracts.validate_prices` (`contracts.py:218`) — clean
Well-layered: `_require_datetime_index` → `_ensure_utc_index` → `_check_monotonic` → `_check_frequency` → `_check_non_negative_prices`, with `freq=None` documented to skip only the cadence check. Price-mode gating (`_is_price_mode`) defensively reads metadata from several attr shapes. Strictly-positive enforcement is intentional and clearly errored.

---

## Verdict for Subsystem D

**No blocking defects.** Core validation logic (`apply_missing_policy`, `validate_prices`) is correct and well-structured. The issues are structural: **date-correction logic is triplicated** (findings ①/②) and there is a **third `ValidationResult`-style naming collision** (③) consistent with the unresolved Day 2 naming issues.

Recommend consolidating the date-correction paths (①/② together — one shared module) as the highest-value cleanup, and folding the `ValidationResult` rename into the same batch as the Day 2 `ValidationError`/`DataSettings` renames.

---

## Files Reviewed

- [x] `src/trend_analysis/io/market_data.py` (inventory + `apply_missing_policy`, date-fix helpers, delta-days twins)
- [x] `src/data/contracts.py` (full)
- [x] `src/trend_analysis/io/validators.py` (`ValidationResult`)
- [x] `src/trend_analysis/io/date_correction.py` (inventory)
- [x] `src/trend_analysis/io/ui_ingest.py` (inventory)
- [x] `src/trend_analysis/io/utils.py` (inventory)
- [x] `src/trend/input_validation.py` (`_fix_invalid_day` cross-check)
- [ ] Deep review of `classify_frequency` / `_infer_mode` (deferred — large; no red flags on scan)
