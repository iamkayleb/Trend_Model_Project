# Day 10 Review: `reporting/`, `export/`, and `viz/` subpackages

**Scope:** 19 files, ~5,308 lines — the output and presentation layer: narrative generation, Excel/CSV/JSON export, bundle packaging, and Plotly/Matplotlib chart helpers
**Date:** 2026-05-12
**Reviewer:** Claude

---

## Files Reviewed

- `reporting/__init__.py` — Backwards-compat shim that re-exports `ReportArtifacts` and `generate_unified_report` from `trend.reporting`, with stub fallbacks when that package isn't installed.
- `reporting/run_artifacts.py` — `write_run_artifacts`: assembles a per-run directory with manifest JSON, an HTML receipt, and copied exported files. Generates SHA256 hashes of every artifact and records the git commit hash.
- `reporting/portfolio_series.py` — `select_primary_portfolio_series`: tries a priority list of result-dict keys to find the best available portfolio return series. Falls back to constructing an equal- or user-weighted combination from raw per-fund data.
- `reporting/narrative.py` — Template-driven narrative generation. Defines `NarrativeTemplateSection`, `DEFAULT_NARRATIVE_TEMPLATES`, `extract_narrative_metrics`, `generate_narrative_sections`, and `validate_narrative_quality`. Also handles forward-looking language detection and disclaimer enforcement.
- `export/__init__.py` — The largest file in the group at 2,051 lines. Drives Excel export (with openpyxl and xlsxwriter compatibility), CSV, JSON, and TXT. Contains sheet formatters, the summary table builder, multi-period aggregation (`combined_summary_result`, `phase1_workbook_data`), and the narrative integration layer.
- `export/bundle.py` — `export_bundle`: assembles a self-contained ZIP artefact from a run object, including portfolio CSV, optional benchmark/weights/bootstrap CSVs, equity and drawdown PNGs, a summary XLSX, and a reproducibility manifest.
- `viz/__init__.py` — Lazy-import shim that proxies chart names through `__getattr__` to avoid pulling in Matplotlib at import time.
- `viz/theme.py` — `base_layout`, `trend_template`, `apply_theme`: shared Plotly theme config (font, colors, grid, hover mode).
- `viz/utils.py` — `QuantileBand`, `coerce_series`, `coerce_frame`, `validate_quantiles`, `quantiles_over_columns`, `quantile_bands`, `hex_to_rgba`: low-level chart utilities shared across the viz subpackage.
- `viz/artifacts.py` — `extract_bundle_zip`: extracts chart bundle ZIPs with path-traversal protection and collision-safe renaming.
- `viz/adapters.py` — Monte Carlo adapter layer. `make_summary`, `make_paths`, `terminal_returns`, `rolling_stats`, `path_correlations`: normalize heterogeneous Monte Carlo outputs to the canonical `("date", "path")` MultiIndex schema that chart modules expect.
- `viz/fan.py` — `make`: Plotly fan chart from NAV paths using configurable quantile bands.
- `viz/path_dist.py` — `make`, `terminal_distribution`: histogram of terminal path values with quantile markers.
- `viz/risk_return.py` — `make`, `risk_return_summary`: annualized risk/return scatter plot.
- `viz/sharpe_ladder.py` — `build_figure`, `prepare_sharpe_ladder`: horizontal bar chart of Sharpe ratios sorted ascending.
- `viz/charts/__init__.py` — Matplotlib chart helpers: `equity_curve`, `drawdown_curve`, `rolling_information_ratio`, `turnover_series`, `weights_heatmap`, `weights_heatmap_data`.
- `viz/charts/rolling_panel.py` — `build_figure`: Plotly rolling diagnostics panel (rolling Sharpe, rolling vol, drawdown) from canonical `make_paths` output.
- `viz/charts/seasonality_heatmap.py` — `build_figure`: monthly seasonality heatmap using `px.imshow`.
- `viz/charts/corr_heatmap.py` — `build_figure`: cross-path correlation heatmap.

---

## Issues Found

### 1. Direct `[]` key access on result dicts in two export functions

`_build_summary_formatter` (line 611) and `format_summary_text` (line 923) both access result-dict keys without `.get()`:

```python
# _build_summary_formatter, line 611
for label, ins, outs in [
    ("Equal Weight", res["in_ew_stats"], res["out_ew_stats"]),
    ("User Weight", res["in_user_stats"], res["out_user_stats"]),
]:
```

```python
# format_summary_text, line 923
for fund, stat_in in res["in_sample_stats"].items():
    stat_out = res["out_sample_stats"][fund]
    weight = res["fund_weights"][fund] * 100
```

Both functions are called from real pipeline paths and not exclusively from test harnesses. The aggregate stats (`in_ew_stats`, `out_ew_stats`, etc.) are guarded with `.get()` earlier in the same `format_summary_text` function (lines 892–898), making the bare `[]` accesses at line 923 inconsistent. A result dict missing `in_sample_stats` or `fund_weights` — which can happen when a period produces no valid output — raises a `KeyError` rather than being skipped or emitting a fallback row.

`summary_frame_from_result` (line 1423) handles this correctly with `.get()` calls:

```python
in_stats_map = cast(..., res.get("in_sample_stats", {}))
out_stats_map = cast(..., res.get("out_sample_stats", {}))
```

**Recommendation:** Replace the bare `[]` accesses in both functions with `.get(key, {})`, matching the pattern already used in `summary_frame_from_result`.

---

### 2. Two openpyxl proxy pairs doing the same job in `export/__init__.py`

Lines 101–177 define `_OpenpyxlWorksheetProxy` / `_OpenpyxlWorkbookProxy`.
Lines 180–261 define `_OpenpyxlWorksheetAdapter` / `_OpenpyxlWorkbookAdapter`.

Both pairs wrap an openpyxl workbook to expose a minimal xlsxwriter-style API (`add_worksheet`, `add_format`, `write`, `write_row`, `set_column`, `freeze_panes`, `autofilter`). The differences are minor: `_OpenpyxlWorkbookProxy.add_format` returns the raw spec dict, while `_OpenpyxlWorkbookAdapter.add_format` wraps it in `dict(spec or {})`; `_OpenpyxlWorksheetProxy` imports `get_column_letter` from the module top-level (with a `None` guard), while `_OpenpyxlWorksheetAdapter` imports it lazily inside each method. Both pairs are live in `export_to_excel`: the `proxy` path is taken when xlsxwriter's openpyxl fallback is detected, and the `workbook_adapter` path is taken when `add_worksheet` / `add_format` attributes are absent.

The dead statement at line 1110 is a visible symptom of the confusion:

```python
if proxy is not None:
    pass
```

Nothing has been done with `proxy` being set here — it's only used later in the per-sheet loop (lines 1116–1154).

**Recommendation:** Consolidate into a single adapter pair. Pick the `_OpenpyxlWorksheetAdapter` / `_OpenpyxlWorkbookAdapter` approach (lazy imports, no module-level None guard), remove the `Proxy` variants, and delete the dead `if proxy is not None: pass` statement.

---

### 3. `portfolio_series` defined three times in `export/__init__.py`

An identical weight-normalization-and-dot-product helper appears at lines 593, 873, and 1367 under three different local names, all inside separate outer functions:

```python
def portfolio_series(
    df: pd.DataFrame | None, weights: Mapping[str, float] | None
) -> pd.Series | None:
    if df is None or df.empty:
        return None
    if weights:
        w = pd.Series({c: float(weights.get(c, 0.0)) for c in df.columns})
        s = float(w.sum())
        if s > 0:
            w = w / s
        else:
            w = pd.Series(1.0 / float(len(df.columns)), index=df.columns)
    else:
        w = pd.Series(1.0 / float(len(df.columns)), index=df.columns)
    return df.mul(w, axis=1).sum(axis=1)
```

`reporting/portfolio_series.py` already exports `select_primary_portfolio_series`, and `_weighted_portfolio` there does the same weighted sum. The three local copies are distinct only by context, not by logic.

**Recommendation:** Extract to a module-level private helper `_weighted_sum` and call it from all three sites. Or import `_weighted_portfolio` from `reporting.portfolio_series` for the two sites that don't need the fallback-key lookup.

---

### 4. `_git_hash()` duplicated in `export/bundle.py` and `reporting/run_artifacts.py`

`export/bundle.py:23` and `reporting/run_artifacts.py:27` both define the same `_git_hash()` helper — same subprocess call, same fallback string. The two implementations diverge only in their exception handling:

- `bundle.py` catches `(subprocess.CalledProcessError, FileNotFoundError)` — specific and correct.
- `run_artifacts.py` catches bare `Exception` — silently swallows any failure including permission errors and unexpected crashes.

**Recommendation:** Move `_git_hash` to `util/hash.py` (or a small `util/vcs.py`) and import it in both files. That also tightens `run_artifacts.py` to catch only the expected exceptions.

---

### 5. `_to_nav_wide` duplicated in `charts/rolling_panel.py` and `charts/seasonality_heatmap.py`

Both files contain an identical 7-line helper (lines 34–41 in each):

```python
def _to_nav_wide(paths: pd.DataFrame) -> pd.DataFrame:
    if paths.empty:
        return pd.DataFrame()
    nav = pd.to_numeric(paths["nav"], errors="coerce")
    wide = nav.unstack("path")
    wide.index = pd.to_datetime(wide.index, errors="coerce")
    wide = wide[wide.index.notna()]
    return wide.sort_index()
```

`viz/adapters.py` already has a more thorough `_paths_to_wide_nav` that validates the index and raises on bad input. The two duplicated helpers silently return empty DataFrames instead.

**Recommendation:** Replace both local `_to_nav_wide` calls with an import from `viz/adapters.py`. If the silent-on-empty behaviour is intentional, wrap `_paths_to_wide_nav` in a helper that catches its `ValueError` and returns empty.

---

## Notes

**`_cache_data` identity fallback repeated four times.** `viz/adapters.py`, `viz/sharpe_ladder.py`, `viz/charts/rolling_panel.py`, and `viz/charts/corr_heatmap.py` all define the same pattern: wrap `st.cache_data` when streamlit is present, return `_identity` otherwise. This could live in a single `viz/_cache.py` and be imported in each module.

**`fan.py` bypasses `apply_theme`.** `path_dist.py` and `risk_return.py` call `apply_theme(fig)` at the end of `make()`, applying the standard Trend Analysis Plotly template. `fan.py` sets `template="plotly_white"` directly in `fig.update_layout(...)` instead. The `theme.py` module exists precisely for this, so `fan.py` should call `apply_theme(fig)` like its siblings.

**`_normalise_color` drops 3-digit hex silently.** `export/__init__.py:43` maps color strings to ARGB hex. A value like `"#f00"` passes `startswith("#")` but fails the `len(stripped) in {6, 8}` check after the `#` is stripped (length is 3), so the function returns `None` and the color is silently dropped. `viz/utils.py::hex_to_rgba` correctly expands 3-digit hex via character doubling. In practice only `"red"` is passed by the built-in formatters, so this doesn't trigger today, but it's a latent inconsistency.

**Dead guard in `validate_narrative_quality`.** Line 256 (`if not sections: return issues`) can never be True — the function already returned early at line 225–230 if sections was falsy. Dead code, low priority.

**`export_to_excel` formatter type detection is fragile.** Lines 1064–1070 inspect the number of parameters in the `formatter` callable to decide whether it's a DataFrame transformer or a sheet formatter. Passing a callable with one parameter treats it as a DataFrame transformer; two or more parameters treats it as a sheet formatter with `df_formatter = None`. This is fragile enough to break silently if a formatter gains a default parameter. The two roles should be separate arguments rather than inferred from a signature.
