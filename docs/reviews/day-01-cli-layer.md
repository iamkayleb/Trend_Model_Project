# Day 1 Review: CLI & Orchestration Layer (`src/trend/`)

**Date:** 2026-03-18
**Reviewer:** Project review
**Scope:** `src/trend/` (10 files, 4618 lines) — CLI entry points, config schema, validation, diagnostics, reporting

---

## Subsystem Summary

| File | Lines | Purpose | Status |
|------|-------|---------|--------|
| `cli.py` | 2044 | Main CLI — 8 subcommands (check, run, report, stress, app, explain, nl, quick-report) | Active, primary entry point |
| `compat_entrypoints.py` | 161 | Backward-compat shims for 6 deprecated commands | Active, deprecation path |
| `config_schema.py` | 332 | Lightweight config validation (stdlib-only, no Pydantic) | Active, clean |
| `input_validation.py` | 385 | CSV/DataFrame ingestion validation with auto-fix dates | Active, clean |
| `validation.py` | 147 | Price frame schema enforcement and lag checks | Active, clean |
| `diagnostics.py` | 68 | `DiagnosticResult[T]` generic container for early-exit signaling | Active, clean |
| `__init__.py` | 26 | Dynamic `__version__` from package metadata | Active, minimal |
| `reporting/__init__.py` | 13 | Package init | Active, minimal |
| `reporting/unified.py` | 1033 | Unified report generation (HTML/PDF/JSON) | Active |
| `reporting/quick_summary.py` | 443 | Compact HTML quick-report builder | Active |

---

## Duplicate Analysis

### Critical: `trend_analysis/cli.py` is a legacy copy (1151 lines)

8 functions in `src/trend/cli.py` have exact-name duplicates in `src/trend_analysis/cli.py`:

| Function | `trend/cli.py` line | `trend_analysis/cli.py` line |
|----------|---------------------|------------------------------|
| `_apply_trend_spec_preset` | 137 | 172 |
| `_apply_universe_mask` | 166 | 255 |
| `_attach_universe_paths` | 196 | 285 |
| `_ensure_dataframe` | 688 | 1107 |
| `_resolve_returns_path` | 643 | 1099 |
| `_run_pipeline` | 734 | 1115 |
| `_print_summary` | 879 | 1138 |
| `_write_report_files` | 900 | 1146 |

**Root cause:** `src/trend_analysis/cli.py` is the legacy CLI (1151 lines) that predates the modern `src/trend/cli.py` (2044 lines). The modern CLI is the canonical version — all `pyproject.toml` entry points route through `trend.cli:main`.

**Recommendation:** Refactor `trend_analysis/cli.py` to delegate to `trend/cli.py` or remove duplicated functions and import from the modern CLI. This eliminates ~800 lines of redundant code.

### `_load_configuration` — 3 copies

| Location | Line |
|----------|------|
| `src/trend/cli.py` | 1126 |
| `src/trend_analysis/cli.py` | 1090 |
| `src/trend_model/cli.py` | 71 |

**Recommendation:** Extract to a shared utility (e.g., `trend.config_schema.load_configuration`) and import from all three locations.

### `_json_default` — 4 copies

| Location | Line |
|----------|------|
| `src/trend/cli.py` | 940 |
| `src/trend_analysis/walk_forward.py` | 317 |
| `src/trend_analysis/backtesting/harness.py` | 678 |
| `src/trend_analysis/core/rank_selection.py` | 69 |

**Recommendation:** Extract to `trend_analysis/util/json_compat.py` and import everywhere.

### `main` — expected duplication (not a problem)

11 `main()` functions exist across the codebase. Each is a distinct entry point for a different CLI tool (trend, trend-llm-proxy, health-summarize, etc.). This is standard Python packaging — no action needed.

### `build_parser` — 2 copies, different signatures

- `src/trend/cli.py:380` — full parser with all subcommands
- `src/trend_model/cli.py:86` — minimal legacy parser

**Recommendation:** `trend_model/cli.py` is a legacy shim (574 lines total in package). Consider whether `trend_model` package is still needed or can be collapsed into `compat_entrypoints.py`.

---

## Other Findings

### `src/trend/config_schema.py` vs `src/trend_analysis/config/`

These are intentionally separate. `config_schema.py` (332 lines) is a lightweight stdlib-only validation layer for fast CLI startup. `trend_analysis/config/` (4740 lines) is the full Pydantic-powered validation. The docstring explicitly documents this design decision. **No action needed.**

### `src/trend/input_validation.py` vs `src/trend/validation.py`

- `input_validation.py` (385 lines): CSV upload validation (column presence, date parsing, auto-fix)
- `validation.py` (147 lines): Price frame schema enforcement (dtype checks, lag validation)

These serve different stages of the pipeline (ingestion vs. post-processing). **No overlap, no action needed.**

### `src/trend/compat_entrypoints.py` — deprecation timeline undefined

6 deprecated commands (`trend-analysis`, `trend-multi-analysis`, `trend-model`, `trend-app`, `trend-run`, `trend-quick-report`) all print warnings and delegate to `trend.cli.main()`. No removal date is documented.

**Recommendation:** Add deprecation timeline comments. Consider whether any can be removed now.

---

## Action Items (prioritized)

1. **HIGH** — Deduplicate `trend_analysis/cli.py` against `trend/cli.py` (eliminates ~800 lines)
2. **MEDIUM** — Extract `_json_default` to shared utility (4 copies → 1)
3. **MEDIUM** — Extract `_load_configuration` to shared utility (3 copies → 1)
4. **LOW** — Evaluate whether `trend_model/` package can be retired
5. **LOW** — Document deprecation timeline for compat entry points

---

## Files Reviewed

- [x] `src/trend/__init__.py`
- [x] `src/trend/cli.py`
- [x] `src/trend/compat_entrypoints.py`
- [x] `src/trend/config_schema.py`
- [x] `src/trend/input_validation.py`
- [x] `src/trend/validation.py`
- [x] `src/trend/diagnostics.py`
- [x] `src/trend/reporting/__init__.py`
- [x] `src/trend/reporting/unified.py` (read, 1033 lines)
- [x] `src/trend/reporting/quick_summary.py` (read, 443 lines)
