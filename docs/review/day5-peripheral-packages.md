# Day 5 Review: Peripheral packages

**Scope:** `backtest/`, `data/`, `health_summarize/`, `trend_model/`, `trend_portfolio_app/`, `utils/`, plus `src/cli.py` and `src/__init__.py` (14 files, ~1,550 lines total)
**Date:** 2026-04-21
**Reviewer:** Claude

---

## Files Reviewed

- `src/__init__.py` — Trivial namespace marker (`"Lightweight namespace for convenience CLI wrappers."`, no imports). Makes `src/` importable as a package so tests can do `import src.cli`.
- `src/cli.py` — Argparse CLI with two subcommands: `cv` (walk-forward cross-validation via the root-level `analysis/` package) and `report` (tearsheet generation). Not wired as an entry point in `pyproject.toml`. See issue below.
- `backtest/__init__.py` — Single re-export: `shift_by_execution_lag` from `backtest.utils`.
- `backtest/utils.py` — One function: `shift_by_execution_lag(obj, lag)`. Shifts a Series or DataFrame forward by `lag` periods to honour the compute-at-close / trade-next-bar convention. Preserves `attrs` and stamps `execution_lag` into them.
- `data/__init__.py` — Re-exports `coerce_to_utc` and `validate_prices` from `data.contracts`.
- `data/contracts.py` — Price ingest validation: UTC timezone coercion, monotonic/duplicate-timestamp checks, frequency inference with equivalence groups (`ME ≡ BME`, etc.), and non-negative price enforcement. The non-negative check is gated on a `"price"` mode flag read from `df.attrs`.
- `utils/__init__.py` — Re-exports `proj_path` and `repo_root` from `utils.paths`.
- `utils/paths.py` — Two helpers: `repo_root()` (honours `TREND_REPO_ROOT` env override) and `proj_path(*parts)` (joins parts onto the repo root). Used for reproducible path resolution independent of CWD.
- `health_summarize/__init__.py` — In-tree fallback for the external `health_summarize` package. Summarises two CI guardrails: the CI signature hash check and the branch-protection snapshot check. Produces JSON or Markdown output. See issue below.
- `trend_portfolio_app/__init__.py` — Empty compat stub (`__all__ = []`). The package it once backed has been retired. Only referenced in `retired/tests/`. See issue below.
- `trend_model/__init__.py` — Empty bootstrap (`__all__ = []`).
- `trend_model/app.py` — Streamlit launcher: runs `streamlit run streamlit_app/app.py` via `subprocess.run`. Returns exit code 127 when `streamlit` is not on PATH.
- `trend_model/cli.py` — Argparse CLI (`trend-run`) for headless pipeline runs. Wraps helpers from `trend.cli` and writes an HTML (optionally PDF) report. See issue below.
- `trend_model/spec.py` — TOML config loader. Defines `SampleWindow`, `BacktestSpec`, `TrendRunSpec` frozen dataclasses and builds them from config objects. Provides `ensure_run_spec` to attach the resolved spec to a config object at runtime.

---

## Issues Found

### 1. `trend_model/cli.py` — Imports 8 private symbols from `trend.cli`

Lines 16–25 import private implementation details by name:

```python
from trend.cli import (
    _determine_seed,
    _ensure_dataframe,
    _prepare_export_config,
    _print_summary,
    _resolve_report_output_path,
    _resolve_returns_path,
    _run_pipeline,
    _write_report_files,
)
```

These are private by convention (leading `_`). Any rename or signature change inside `trend.cli` will silently break `trend_model/cli.py` with an `ImportError` at runtime rather than a type error at development time.

**Recommendation:** Promote the 8 functions to the public API of `trend.cli` (rename, remove leading `_`, add to `__all__`) if they are genuinely shared. If they are internal implementation details, the right fix is to expose a single `trend.cli.run_headless(cfg, ...)` function that `trend_model/cli.py` calls instead.

---

### 2. `src/cli.py` — Dev-only utility not wired as an entry point

`src/cli.py` provides `cv` and `report` subcommands that wrap the root-level `analysis/` package. It is:

- Not listed in `[project.scripts]` in `pyproject.toml`
- Only imported by `tests/test_cv.py` (as `import src.cli`)
- Works only because `src/__init__.py` makes `src/` importable as a package — an unusual setup for a source directory

It is essentially a local dev tool accessed through the `src.` import path rather than an installed command.

**Recommendation:** Either register it as a `[project.scripts]` entry point if it should be a first-class command, or move it to `scripts/` (alongside `scripts/walk_forward.py` etc.) to make its role explicit. In either case, remove the `src/__init__.py` package-marker if there are no other `src.*` imports outside tests.

---

### 3. `trend_portfolio_app/__init__.py` — Empty compat stub with no active callers

The only references to `trend_portfolio_app` are in `retired/tests/`. No production code imports it.

**Recommendation:** Delete `src/trend_portfolio_app/` entirely, or at minimum add a comment explaining it is a tombstone and when it can be removed. Leaving a silent empty stub invites confusion about whether the package is live.

---

### 4. `health_summarize/__init__.py` — Private helpers exported in `__all__`

The module's `__all__` (lines 393–410) lists 13 private-by-convention `_*` symbols (`_branch_row`, `_doc_url`, `_escape_table`, etc.) alongside the genuinely public ones (`main`, `build_signature_hash`, `Args`).

`__all__` controls what `from health_summarize import *` exports and signals what is part of the public API. Listing private helpers there is misleading — it suggests they are stable contracts when they are implementation details.

**Recommendation:** Remove the `_*` symbols from `__all__`. Keep only `main`, `build_signature_hash`, and `Args`. Tests can import the private helpers directly by name if needed.

---

### 5. `trend_model/spec.py` — Config-access helpers duplicated from `pipeline_helpers.py`

`spec.py` defines its own `_cfg_value`, `_cfg_section`, `_section_get` helpers (lines 101–113) that are functionally identical to the same-named helpers in `trend_analysis/pipeline_helpers.py`. Cross-package duplication of three one-liners is low-risk but means any future logic change (e.g. supporting a new config object protocol) must be made in two places.

**Recommendation:** No urgent action — the duplication is small and a direct import from `trend_analysis.pipeline_helpers` would create a cross-package dependency that may not be desirable. Worth noting if either copy ever grows beyond trivial accessors.

---

## Notes (not actionable issues)

### `ensure_run_spec` uses `object.__setattr__` as a fallback

Lines 316–324 of `spec.py` try `setattr(cfg, attr, value)` and fall back to `object.__setattr__(cfg, attr, value)` if the first raises. The comment explains the intent: to attach resolved specs to frozen dataclasses or configs with custom `__setattr__`. The `try/except/continue` swallows failures silently, which is intentional (the function is best-effort). Worth knowing if attribute attachment ever appears to silently fail.

### `data/contracts.py` price mode gate is deep

`_check_non_negative_prices` only runs when `_is_price_mode(df)` returns `True`, which reads a chain of nested `df.attrs` fields (`attrs["market_data"]["metadata"].mode` → `attrs["market_data"]["mode"]` → `attrs["market_data_mode"]`). This is correct but non-obvious; callers who don't know to set `df.attrs["market_data_mode"] = "price"` will silently skip the check. No fix required — just worth documenting at the call site.

---

## No Issues (clean files)

- `backtest/__init__.py` — clean single re-export
- `backtest/utils.py` — clean, well-documented, preserves `attrs`
- `data/__init__.py` — clean re-export
- `data/contracts.py` — comprehensive ingest validation, clean internal structure
- `utils/__init__.py` — clean re-export
- `utils/paths.py` — clean, `TREND_REPO_ROOT` override well-handled
- `trend_model/__init__.py` — minimal, appropriate
- `trend_model/app.py` — clean Streamlit subprocess launcher

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| Medium | Expose a public API in `trend.cli` so `trend_model/cli.py` stops importing private `_*` symbols | `trend_model/cli.py`, `trend/cli.py` |
| Low | Register `src/cli.py` as an entry point or move to `scripts/`; clean up `src/__init__.py` | `src/cli.py`, `src/__init__.py` |
| Low | Delete `trend_portfolio_app/` (or add a tombstone comment) | `trend_portfolio_app/__init__.py` |
| Low | Remove `_*` symbols from `health_summarize.__all__` | `health_summarize/__init__.py` |
| Info | Note `_cfg_value`/`_cfg_section`/`_section_get` duplication across packages | `trend_model/spec.py`, `trend_analysis/pipeline_helpers.py` |
