# Phase-3 Verification 01: CLI & Entry Points

**Date:** 2026-07-16
**Reviewer:** Project review (second pass)
**Branch:** `phase-3`
**Purpose:** Verify whether the Day 1 (CLI & orchestration) action items were correctly applied on the `phase-3` branch.
**Scope:** `src/trend/cli.py`, `src/trend/cli_helpers.py`, `src/trend/compat_entrypoints.py`, `src/trend_analysis/cli.py`, `src/trend_model/`, `src/cli.py`

---

## Subsystem Summary

| File | Lines | Purpose | Change vs Day 1 |
|------|-------|---------|-----------------|
| `trend/cli.py` | 2364 | Main CLI — canonical entry point | Grew (was 2044); 3 helpers now imported |
| `trend/cli_helpers.py` | 126 | **New** — shared CLI helpers | New file (extraction target) |
| `trend/compat_entrypoints.py` | 160 | Backward-compat shims | Unchanged in substance |
| `trend_analysis/cli.py` | 2158 | Legacy CLI | **Grew from 1151** |
| `trend_model/cli.py` | 197 | Legacy shim CLI | Still present |
| `trend_model/app.py` | 45 | Legacy shim app | Still present |
| `trend_model/spec.py` | 333 | Legacy spec | Still present |
| `src/cli.py` | 132 | Top-level dispatcher | Present |

---

## Verification Results

| # | Original Suggestion (Day 1) | Priority | Verdict |
|---|------------------------------|----------|---------|
| 1 | Deduplicate the 8 shared functions between `trend_analysis/cli.py` and `trend/cli.py` | HIGH | ⚠️ Partial (3 of 8) |
| 2 | Extract `_json_default` to a shared utility (4 copies → 1) | MEDIUM | ❌ Not applied |
| 3 | Extract `_load_configuration` to a shared utility (3 copies → 1) | MEDIUM | ❌ Not applied (diverged) |
| 4 | Evaluate whether `trend_model/` can be retired | LOW | ❌ Not applied |
| 5 | Document deprecation timeline for compat entry points | LOW | ❌ Not applied |

---

## Detail

### ① Deduplicate 8 shared CLI functions — ⚠️ Partial

A new module `src/trend/cli_helpers.py` was created and now holds 3 of the 8 previously duplicated functions:

- `_apply_trend_spec_preset` (`cli_helpers.py:9`)
- `_apply_universe_mask` (`cli_helpers.py:38`)
- `_attach_universe_paths` (`cli_helpers.py:68`)

Both call sites now import them instead of redefining:
- `src/trend/cli.py:22`
- `src/trend_analysis/cli.py:19`

The extraction also hardened error handling: broad `except ValueError` was replaced with `(AttributeError, TypeError, ValueError)` plus debug logging in `_attach_universe_paths`.

**However, the remaining 5 functions are still copy-pasted in both CLIs:**

| Function | `trend/cli.py` | `trend_analysis/cli.py` |
|----------|----------------|--------------------------|
| `_resolve_returns_path` | 608 | 2106 |
| `_ensure_dataframe` | 653 | 2114 |
| `_run_pipeline` | 699 | 2122 |
| `_print_summary` | 950 | 2145 |
| `_write_report_files` | 971 | 2153 |

Furthermore, `trend_analysis/cli.py` **grew from 1151 → 2158 lines**. The Day 1 goal of eliminating ~800 lines of redundancy was not met; the file expanded.

### ② Extract `_json_default` (4 copies → 1) — ❌ Not applied

Still 4 copies, unchanged from Day 1:
- `src/trend/cli.py`
- `src/trend_analysis/walk_forward.py`
- `src/trend_analysis/backtesting/harness.py`
- `src/trend_analysis/core/rank_selection.py`

No shared JSON utility was created (`src/trend_analysis/util/` contains no json helper).

### ③ Extract `_load_configuration` (3 copies → 1) — ❌ Not applied, and diverged

Still 3 copies, and the signatures have now drifted apart:

- `src/trend/cli.py:1197` → `def _load_configuration(path: str) -> Any`
- `src/trend_analysis/cli.py:2097` → `def _load_configuration(path: str) -> tuple[Path, Any]`
- `src/trend_model/cli.py:71` → `def _load_configuration(path: str) -> tuple[Path, Any]`

Two of the three now return a tuple while the canonical one returns `Any`. Consolidation is harder than it was at Day 1.

### ④ Evaluate retiring `trend_model/` — ❌ Not applied

`src/trend_model/` still contains `app.py` (45), `cli.py` (197), `spec.py` (333). No retirement or documented decision. (This item was "evaluate," so it is not strictly a regression — but no action or rationale is recorded.)

### ⑤ Document deprecation timeline — ❌ Not applied

`src/trend/compat_entrypoints.py:11` still only prints:

```python
print(f"Warning: '{old}' is deprecated; use '{new}' instead.", file=sys.stderr)
```

No removal date or target version is documented for any of the deprecated commands.

---

## Verdict for Subsystem A

**1 partial, 4 not applied.**

The `cli_helpers.py` extraction is a genuine (if incomplete) improvement and the error-handling hardening is a bonus. But the highest-leverage work was left undone:

- The 5 remaining duplicated pipeline functions still exist in both CLIs.
- `trend_analysis/cli.py` grew rather than shrank.
- `_json_default` and `_load_configuration` were never consolidated, and `_load_configuration` has actively diverged.

---

## Files Reviewed

- [x] `src/trend/cli.py`
- [x] `src/trend/cli_helpers.py`
- [x] `src/trend/compat_entrypoints.py`
- [x] `src/trend_analysis/cli.py`
- [x] `src/trend_model/cli.py`
- [x] `src/trend_model/app.py`
- [x] `src/trend_model/spec.py`
- [x] `src/cli.py`
