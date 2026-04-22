# Day 6 Review: The `trend/` package

**Scope:** 14 files, ~5,800 lines — the user-facing CLI layer and its support modules
**Date:** 2026-04-22
**Reviewer:** Claude

---

## Files Reviewed

- `trend/__init__.py` — Package marker that exposes `__version__` dynamically via `importlib.metadata` rather than a hardcoded string. Nothing else lives here, which is the right call.
- `trend/compat_entrypoints.py` — The graveyard of old entry points. Six functions (`trend_analysis`, `trend_multi_analysis`, `trend_model`, `trend_app`, `trend_run`, `trend_quick_report`) print a deprecation warning and then re-route to the modern `trend.cli.main`. Each also does light argument translation — stripping `--detailed`, mapping `--input` to `--returns`, `--artefacts` to `--out`, etc. Well-organised.
- `trend/diagnostics.py` — Defines the canonical result/diagnostic types used across the whole stack: `DiagnosticPayload` (a structured reason + message), `DiagnosticResult[T]` (value + optional diagnostic), `RunPayload` (Protocol), and `RunPayloadResult`. This is the right place for these and they're imported widely.
- `trend/validation.py` — Validates price data in long (tidy) format: symbol/date/close schema enforcement, per-symbol monotonicity, duplicates, and `assert_execution_lag`. Works on a different shape to `data/contracts.py` (which validates the wide columnar format), so these two modules aren't redundant.
- `trend/config_schema.py` — A deliberate lightweight alternative to Pydantic config validation. The module docstring is refreshingly honest about why it exists: full Pydantic startup is expensive and overkill for the CLI's early-exit checks. Validates `data.csv_path`, `data.frequency`, and cost model fields using only stdlib types. Includes `ConfigCoverageTracker` instrumentation hooks.
- `trend/input_validation.py` — CSV upload validation shared between the Streamlit app and the CLI. Handles case-insensitive column lookup, auto-correction of invalid day-of-month dates (e.g. 11/31 → 11/30), auto-sorting by date, and duplicate timestamp detection with row-level context in error messages. A practical user-facing module.
- `trend/mc/__init__.py` — Re-exports from the `mc` subpackage. Clean.
- `trend/mc/charts.py` — One constant: `NAV_PATH_REQUIRED_CHARTS`. See issue below.
- `trend/mc/io.py` — Loads `nav_paths.parquet` from a Monte Carlo bundle directory. Has an enum-like string constant for missing-file behavior (`"raise"` vs `"return-none"`) with validation on entry. Looks for accidental CSV/JSON files and raises a clear error if someone accidentally placed the wrong format. Clean.
- `trend/mc/viz.py` — Monte Carlo visualization: fan chart, path distribution, risk/return scatter. Handles chart selection validation, bundle file discovery, and chart export to both PNG (via Plotly/Kaleido) and HTML embedding. See issue below.
- `trend/reporting/__init__.py` — Re-exports `build_run_report` and `generate_unified_report`. Clean.
- `trend/reporting/quick_summary.py` — Builds a self-contained HTML report from existing run artefacts (metrics CSV, details JSON). Generates equity/drawdown and turnover charts, a parameter-grid heatmap, and assembles them into an inline-image HTML page. Also has a `main()` so it can be called directly as `trend quick-report`. See issue below.
- `trend/reporting/unified.py` — The full report generator. Produces an HTML report (and optionally PDF via fpdf2) including stats tables, regime analysis, narrative text (if the LLM layer is enabled), and all the charts. At 1,033 lines it's large, but the complexity is mostly inherent — there are a lot of moving parts in a complete run report. No structural issues.
- `trend/cli.py` — The main `trend` command (2,326 lines). Covers: `trend run` (pipeline execution), `trend report` / `trend quick-report` (HTML generation), `trend mc viz` (Monte Carlo charts), `trend nl` (natural language config patching and replay), `trend check` (environment check), and `trend app` (Streamlit launcher). Also contains a legacy CLI shim layer for backwards compatibility with test patches. See issues below.

---

## Issues Found

### 1. `NAV_PATH_REQUIRED_CHARTS` defined twice

`trend/mc/charts.py` line 6 and `trend/mc/viz.py` line 16 both define the same constant with the same value:

```python
NAV_PATH_REQUIRED_CHARTS: frozenset[str] = frozenset({"path_dist"})
```

`charts.py` exists precisely to hold this constant and re-export it. `mc/__init__.py` imports it from `charts.py`. But `viz.py` ignores `charts.py` and defines its own copy locally, which it then uses throughout. So there are two live definitions.

If anyone adds a chart to one but not the other, `mc.NAV_PATH_REQUIRED_CHARTS` and the internal `viz.py` logic will quietly disagree.

**Recommendation:** Make `viz.py` import from `charts.py` instead of re-defining the constant. That's what `charts.py` is for.

---

### 2. `_init_matplotlib()` copied between the two reporting modules

`reporting/quick_summary.py:31–45` and `reporting/unified.py:27–41` each define an identical `_init_matplotlib()` function (sets `matplotlib.use("Agg")`, configures `savefig.*` rcParams, returns `plt`) and both call `plt = _init_matplotlib()` at module import time.

**Recommendation:** Extract to `reporting/_matplotlib.py` and import from there. Six lines of boilerplate aren't worth maintaining twice.

---

### 3. `trend/cli.py` at 2,326 lines carries too many distinct concerns

This file has grown to cover at least five separate jobs:

- **Pipeline orchestration** — `_run_pipeline`, `_handle_exports`, `_write_bundle`, etc.
- **LLM/NL operations** — `_apply_nl_instruction`, `_build_nl_chain`, `_log_nl_operation`, NL replay
- **Monte Carlo visualization** — `_run_mc_viz_command`, `_build_mc_fan_chart`, `_load_mc_bundle_frames`, etc.
- **Legacy CLI shim** — `_refresh_legacy_cli_module`, `_legacy_callable`, `_LEGACY_BASELINES`, `_ORIGINAL_FALLBACKS`
- **Argument parsing** — `build_parser` and ~10 sub-parsers

None of these are wrong individually, but the combination makes the file genuinely hard to navigate. The LLM/NL block alone spans several hundred lines and is logically independent of the Monte Carlo block.

**Recommendation:** No immediate action required, but the natural split would be to move the NL/LLM handlers into `trend/nl.py` and the MC viz handlers into `trend/mc/commands.py`, leaving `trend/cli.py` as a thin dispatcher that imports and calls those. This is medium-effort refactoring that pays off the next time anyone needs to extend either area.

---

### 4. The legacy CLI shim in `trend/cli.py` is non-obvious

Lines 105–380 implement a delegation layer (`_legacy_callable`, `_refresh_legacy_cli_module`, `_LEGACY_BASELINES`, `_ORIGINAL_FALLBACKS`) that exists so `trend.cli` functions can detect when `trend_analysis.cli` has been monkeypatched in tests and forward to the patched version. This is how the codebase supports patches to the old CLI module affecting the new one during test runs.

The mechanism is correct and intentional, but it's genuinely surprising to read and easy to break if someone doesn't know why it's there.

**Recommendation:** No code change needed, but this deserves a block comment at the top of that section explaining the problem it solves and why the indirection is necessary. Right now there's a docstring on `_legacy_callable` but nothing explaining the broader system.

---

## Notes (not actionable)

### `compat_entrypoints.py` argument translation is worth keeping an eye on

The `_translate_trend_run_args` function maps old `trend-run` flags to their new `trend report` equivalents (`--artefacts` → `--out`, `-o` → `--output`). If the `trend report` parser ever renames those flags again, this translation will silently pass the old flag through and the user will get a confusing parse error rather than a deprecation message. Low probability, but worth a note in the function's docstring.

### `config_schema.py` tracker instrumentation is a bit scattered

`validate_core_config` calls `tracker.track_validated(key)` after every field resolution — seven separate `if tracker is not None` blocks interleaved with validation logic. It works, but if a new field is added it's easy to forget to add the tracker call. Not a bug, just a code smell that a decorator or context manager might clean up eventually.

---

## No Issues (clean files)

- `trend/__init__.py` — minimal and correct
- `trend/compat_entrypoints.py` — clean translation layer, well-structured
- `trend/diagnostics.py` — clean, the right home for these types
- `trend/validation.py` — clean tidy-format validator; distinct from `data/contracts.py`
- `trend/input_validation.py` — well-done; auto-date-correction is a practical UX choice
- `trend/mc/io.py` — clean, good error messages for the wrong-format case
- `trend/mc/viz.py` — complex but single-concern
- `trend/reporting/__init__.py` — clean re-export
- `trend/reporting/unified.py` — large but the complexity is inherent

---

## Action Items Summary

| Priority | Action | File(s) |
|----------|--------|---------|
| Low | Make `viz.py` import `NAV_PATH_REQUIRED_CHARTS` from `charts.py` instead of re-defining it | `trend/mc/viz.py`, `trend/mc/charts.py` |
| Low | Extract `_init_matplotlib()` to a shared module | `trend/reporting/quick_summary.py`, `trend/reporting/unified.py` |
| Low | Add a block comment explaining the legacy CLI shim mechanism | `trend/cli.py:105–380` |
| Info | Consider splitting LLM/NL and MC viz handlers into their own modules | `trend/cli.py` |
