# Day 3 Review: src/trend_analysis/ root standalone files (Part 1)

**Scope:** First 20 files by line count (1–169 lines each), total ~1,250 lines
**Date:** 2026-03-27
**Reviewer:** Claude

---

## Files Reviewed

| File | Lines | Verdict | Notes |
|------|-------|---------|-------|
| `automation_multifailure.py` | 16 | CI fixture | Not dead — imported by 3 workflow tests |
| `cash_policy.py` | 17 | Keep | `CashPolicy` dataclass; canonical definition |
| `_autofix_probe.py` | 18 | CI fixture | Intentional probe for autofix pipeline |
| `_typing.py` | 19 | Keep | NumPy array type aliases; used in 4 src files |
| `constants.py` | 22 | Keep | Widely used (11 import sites) |
| `regime_utils.py` | 25 | Keep | Used by `pipeline_helpers.py` and `monte_carlo/runner.py` |
| `typing.py` | 34 | Issue (see below) | Exports `MultiPeriodPeriodResult`; unused in production |
| `_ci_probe_faults.py` | 36 | CI fixture | Style probe; has dedicated test |
| `_autofix_trigger_sample.py` | 39 | CI fixture | Intentional style violations for autofix |
| `_autofix_violation_case2.py` | 57 | CI fixture | Intentional violations for autofix |
| `_autofix_violation_case3.py` | 46 | CI fixture | Intentional violations for autofix |
| `rebalancing.py` | 58 | Keep (shim) | Backward-compat re-export from `rebalancing/strategies.py` |
| `script_logging.py` | 60 | Keep | Wraps `logging_setup.py`; used by 30+ scripts |
| `selector.py` | 69 | Keep | Plugin-registered selectors; used lazily by `multi_period/engine.py` |
| `run_multi_analysis.py` | 78 | Likely dead | See below |
| `logging_setup.py` | 99 | Keep | Root logger + file handler setup; 7 src imports |
| `timefreq.py` | 106 | Keep | Pandas freq aliases + helpers; used by 5 src files |
| `run_analysis.py` | 137 | Dead | See below |
| `signal_presets.py` | 144 | Keep | Preset TrendSpecs; used by both CLIs + tests |
| `universe_catalog.py` | 169 | Keep | Named universe YAML loader; used by both CLIs |

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
