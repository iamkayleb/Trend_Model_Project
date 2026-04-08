# Day 3 Review: src/trend_analysis/ root standalone files (Part 1)

**Scope:** First 20 files by line count (1–169 lines each), total ~1,250 lines
**Date:** 2026-03-27
**Reviewer:** Claude

---

## Files Reviewed

- `automation_multifailure.py` — CI fixture with a single `aggregate_numbers()` function; exists so the autofix pipeline has a stable, importable target to run against. Imported directly by workflow tests.
- `cash_policy.py` — Defines the `CashPolicy` dataclass used to configure how implicit cash is handled in rebalancing outputs. Canonical definition; re-exported via the `rebalancing.py` shim.
- `_autofix_probe.py` — Deliberate autofix probe; originally had a missing import so the CI pipeline had something deterministic to repair. Now clean but kept as a callable fixture.
- `_typing.py` — Shorthand NumPy array type aliases (`FloatArray`, `VectorF`, `MatrixF`, `AnyArray`) used across numerical computation modules to avoid verbose `np.ndarray[...]` annotations.
- `constants.py` — Central store for package-wide magic values: default output directory, default export formats, and three numerical tolerance levels. Used in 11 places across src.
- `regime_utils.py` — Two small helpers for normalising regime key strings (`normalize_regime_key`) and resolving their aliases between calm/stress and riskon/riskoff naming conventions.
- `typing.py` — Defines a `MultiPeriodPeriodResult` TypedDict and two supporting aliases (`CovarianceDiagonal`, `StatsMapping`) describing the intended shape of multi-period engine results. See issue below — unused in production.
- `_ci_probe_faults.py` — Style-only CI probe; exercises the yaml and math import paths with three trivial functions. Currently clean; validates that the autofix pipeline produces a passing run.
- `_autofix_trigger_sample.py` — Intentionally badly formatted code (spacing, long lines, indentation) so black and ruff have a deterministic target to reformat during CI.
- `_autofix_violation_case2.py` — Longer set of deliberate violations targeting docformatter, black, and isort specifically; includes an excessively long string and verbose docstrings.
- `_autofix_violation_case3.py` — Deliberate violations targeting ruff's F841 (unused variable) rule and old-style `List[int]` annotations; comments document which violations are fixable vs. not.
- `rebalancing.py` — Backward-compatibility shim that re-exports everything from `rebalancing/strategies.py` so legacy imports of `trend_analysis.rebalancing` continue to work without touching the canonical implementations.
- `script_logging.py` — Thin wrapper around `logging_setup.py` that initialises the perf logger for standalone scripts. Used by 30+ scripts via `setup_script_logging()` and `run_with_script_logging()`.
- `selector.py` — Registers `RankSelector` and `ZScoreSelector` into the plugin system. `RankSelector` picks top-N funds by a metric column; `ZScoreSelector` filters by z-score threshold. Used lazily by `multi_period/engine.py`.
- `run_multi_analysis.py` — Standalone argparse CLI entry point for multi-period analysis. Lazily registered in `__init__.py` but never imported or called by any production code. See issue below.
- `logging_setup.py` — Configures the root Python logger with a timestamped file handler and optional console handler. Returns the log file path. Used by both CLIs and several submodules.
- `timefreq.py` — Centralises pandas frequency aliases (`ME`, `M`, `QE`, `Q`) and provides `monthly_date_range` / `monthly_period_range` wrappers to enforce the project's datetime frequency policy and avoid FutureWarnings.
- `run_analysis.py` — Standalone argparse CLI entry point for single-period analysis. Not wired as an entry point in `pyproject.toml`, not imported anywhere. See issue below.
- `signal_presets.py` — Defines three named `TrendSpec` presets (Conservative, Balanced, Aggressive) and lookup helpers used by both CLIs and the Streamlit UI to populate dropdowns and form defaults.
- `universe_catalog.py` — Loads named universe definitions from YAML files in `config/universe/`, resolving relative paths for data and membership CSVs, and builds a filtered membership mask via `build_membership_mask`.

---

## Issues Found

### 1. `run_analysis.py` — Orphaned CLI module (DELETE candidate)

`run_analysis.py` contains a `main()` function with its own `argparse` parser (`prog="trend-analysis"`). However:

- It is **not registered as an entry point** in `pyproject.toml`. The `trend-analysis` entry point goes to `trend.compat_entrypoints:trend_analysis`, which delegates to `trend.cli.main`.
- It is **not imported anywhere** in `src/` (grep confirms zero import hits).
- It is **not in `__init__.py`'s lazy loader**.
- It duplicates logic from `trend_analysis/cli.py` (same `argparse` pattern, same flags).

**Recommendation:** Delete. If a standalone non-Click entry point is ever needed, it should be documented and wired in `pyproject.toml`.

---

### 2. `run_multi_analysis.py` — Lazily registered but never consumed (DELETE candidate)

Similar situation to `run_analysis.py`:

- `__init__.py` registers it in `_LAZY_SUBMODULES` as `"run_multi_analysis": "trend_analysis.run_multi_analysis"`.
- The `trend-multi-analysis` entry point goes to `trend.compat_entrypoints:trend_multi_analysis`, which also delegates to `trend.cli.main`.
- No production code imports from it directly.

**Recommendation:** Delete `run_multi_analysis.py`. Remove `"run_multi_analysis"` from `_LAZY_SUBMODULES` in `__init__.py` and from `__all__`.

---

### 3. `typing.py` — TypedDict unused in production source

`typing.py` exports `MultiPeriodPeriodResult` (a TypedDict), `CovarianceDiagonal`, and `StatsMapping`. However:

- Only `tests/test_trend_analysis_typing_contract.py` imports from it.
- **`multi_period/engine.py` defines its own local alias at line 76:**
  ```python
  MultiPeriodPeriodResult = Dict[str, Any]
  ```
  This overrides/ignores the TypedDict contract from `typing.py`, making that module dead in production.

**Issue:** The TypedDict in `typing.py` documents the intended contract for multi-period results, but `engine.py` bypasses it with a loose `Dict[str, Any]`. These are inconsistent.

**Recommendation:** Either:
- (a) Make `engine.py` import and use `MultiPeriodPeriodResult` from `typing.py` (proper fix — enforces the contract), or
- (b) Delete `typing.py` and remove the contract test if the TypedDict is aspirational and not currently enforced.

---

### 4. CI Fixture Files Polluting the `trend_analysis` Namespace (6 files)

These six files live in `src/trend_analysis/` but are not production code:

```
_autofix_probe.py
_autofix_trigger_sample.py
_autofix_violation_case2.py
_autofix_violation_case3.py
_ci_probe_faults.py
automation_multifailure.py
```

They're intentional CI fixtures for the autofix pipeline — they contain deliberate style violations and are tested by `tests/workflows/`. The leading `_` on four of them signals "private", but `automation_multifailure.py` has no such marker.

**They are not dead and must not be deleted.** However, they pollute `trend_analysis`'s public namespace and clutter IDE autocomplete alongside real business logic.

**Recommendation:** Move to a dedicated namespace, e.g. `src/trend_analysis/_ci_fixtures/`, and update the 5 workflow test files that import them. This is low-risk refactoring — no production code imports these.

---

### 5. `_typing.py` vs `typing.py` — Confusing naming

Two files with almost identical names serve completely different purposes:
- `_typing.py` → NumPy array type aliases (`FloatArray`, `VectorF`, `MatrixF`, `AnyArray`)
- `typing.py` → Multi-period result TypedDict (`MultiPeriodPeriodResult`, etc.)

The underscore prefix convention usually signals "private implementation detail", yet `typing.py` (no underscore) is only used by tests. This is inverted from what you'd expect.

**Recommendation:** Rename `typing.py` to `multi_period_typing.py` (or fold it into `multi_period/typing.py`) once issue #3 above is resolved.

---

### 6. `CashPolicy` dual exposure

`CashPolicy` is defined in `cash_policy.py` and re-exported from `rebalancing.py` (the compat shim). This is intentional — `rebalancing.py` exists specifically to maintain backward-compatible imports from `trend_analysis.rebalancing`.

**No action needed**, but note: if the shim is ever removed, callers must import from `trend_analysis.cash_policy` or `trend_analysis.rebalancing.strategies` directly.

---

## No Issues (clean files)

These files are clean, well-scoped, and actively used with no observed problems:

- `constants.py` — good central location for magic values
- `regime_utils.py` — small, focused, well-used
- `_typing.py` — clean numpy alias module
- `script_logging.py` — thin wrapper, good abstraction
- `logging_setup.py` — clear single responsibility
- `logging.py` — JSONL structured logger; separate concern from `logging_setup.py` (not a duplicate)
- `timefreq.py` — excellent policy doc + helpers for pandas freq handling
- `signal_presets.py` — clean preset container
- `universe_catalog.py` — clear YAML loader with good path resolution
- `selector.py` — clean plugin registration pattern
- `rebalancing.py` — explicit compat shim with clear docstring

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| High | Delete orphaned CLI module | `run_analysis.py` |
| High | Delete lazily-registered dead CLI module + clean `__init__.py` | `run_multi_analysis.py` |
| Medium | Fix type contract inconsistency in `engine.py` or delete `typing.py` | `typing.py`, `multi_period/engine.py:76` |
| Low | Move CI fixtures to `_ci_fixtures/` subpackage | 6 `_autofix_*` / `automation_*` / `_ci_probe_*` files |
| Low | Rename `typing.py` after resolving issue #3 | `typing.py` |
