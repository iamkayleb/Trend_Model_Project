# Reporting / Export / Viz
**Scope:** `trend/reporting/*`, `trend_analysis/reporting/*`, `trend_analysis/export/*`, `trend_analysis/viz/*`

*Mixed subsystem: `unified.py` and `quick_summary.py` were covered on Day 1; everything else is first-time.*

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `trend_analysis/export/__init__.py` | 1974 | Excel/CSV/JSON export, summary formatting |
| `trend/reporting/unified.py` | 1027 | Unified HTML/PDF report |
| `trend_analysis/reporting/narrative.py` | 532 | Narrative section templates |
| `trend_analysis/viz/adapters.py` | 512 | Viz data adapters |
| `trend_analysis/reporting/run_artifacts.py` | 462 | Run artifact writing |
| `trend/reporting/quick_summary.py` | 429 | Compact HTML quick-report |
| `trend_analysis/export/bundle.py` | 393 | Bundle export |
| `trend_analysis/export/run_envelope.py` | 248 | Run envelope |
| `trend_analysis/reporting/portfolio_series.py` | 170 | Portfolio series selection |
| `trend_analysis/viz/*` (12 more) | 1316 | Charts, theme, utils, artifacts |
| `trend/reporting/_matplotlib.py` | 24 | **New** — shared matplotlib setup |

Largest functions:

| Lines | Function | Location |
|------:|----------|----------|
| **366** | `export_bundle` | `export/bundle.py:28` |
| 232 | `format_summary_text` | `export/__init__.py:730` |
| 228 | `_build_summary_formatter` | `export/__init__.py:437` |
| 158 | `summary_frame_from_result` | `export/__init__.py:1212` |
| 129 | `combined_summary_result` | `export/__init__.py:1372` |
| 120 | `_render_pdf` | `trend/reporting/unified.py:826` |

---

## Change since Day 1 (not previously recommended)

### `_matplotlib.py` — a new shared setup module, and it's the right shape

At Day 1, `unified.py` and `quick_summary.py` each defined their own `_init_matplotlib`. Phase-3 adds `trend/reporting/_matplotlib.py` (24 lines) and both files now import from it (`unified.py:18`, `quick_summary.py:28`). The duplication is genuinely gone.

I didn't recommend this — it's the developers' own change — and it's well done. Two things it gets right that matter beyond dedup:

```python
def init_matplotlib() -> Any:
    import matplotlib            # ← imported inside the function
    matplotlib.use("Agg")        # ← headless backend set before pyplot
    from matplotlib import pyplot as plt
    ...
```

The import is **inside** the function, and `matplotlib.use("Agg")` is called **before** `pyplot` is imported — which is the order that actually works for headless rendering. Centralising the `savefig` rcParams also means the two report surfaces can't drift on output DPI or background colour.

This is precisely the pattern `metrics/attribution.py` is missing (module-level `import matplotlib.pyplot`, no backend selection) — see the metrics review, finding ①. The fix already exists in this repo; it just hasn't been applied there.

**Residual (Low):** both consumers still call it at module scope — `plt = init_matplotlib()` (`unified.py:26`, `quick_summary.py:31`) — so importing either module still initialises matplotlib eagerly. And `export/bundle.py:11` still does a bare module-level `import matplotlib`. Better than before, not yet lazy.

---

## Positive / Clean

### `viz/charts/` uses a consistent module protocol
Each chart module exposes `build_figure(data, ...) -> go.Figure` — `corr_heatmap.py`, `rolling_panel.py`, `seasonality_heatmap.py`, `sharpe_ladder.py`. I checked whether this was duplication and it isn't: same name, same return type, genuinely different implementations. It's a convention that makes chart modules interchangeable.

Worth noting there's no registry enforcing it (`viz/charts/__init__.py` doesn't reference `build_figure`), so the protocol is by convention only — fine at four modules, worth formalising if it grows.

### The `viz/` package is well-sized
13 files, largest 512 lines, most under 180. Charts, theme, utils, and artifacts are separated rather than piled into one module. Good contrast with `export/`.

---

## Findings

### ① `_coerce_series` — three copies, and two disagree on error handling — Medium

Defined in `trend/reporting/unified.py`, `trend/reporting/quick_summary.py`, and `trend_analysis/reporting/portfolio_series.py`. The two in the same package diverge in the branch that matters:

```python
# unified.py — surfaces bad input
raise TypeError("Unable to convert value to pandas Series")

# quick_summary.py — swallows it
return pd.Series(dtype=float)
```

Everything above that line is the same logic (Series passthrough, Mapping, Sequence). The only real difference is what happens on unrecognised input: one raises, the other silently produces an empty Series.

That divergence is invisible at the call site. Feed the same malformed value to the unified report and you get an exception; feed it to the quick report and you get an empty chart with no indication anything went wrong. Two functions with the same name in the same package should not have opposite failure semantics.

**Recommendation:** one implementation, with the strict-vs-lenient behaviour as an explicit parameter (`strict: bool = True`) if both are genuinely wanted.

### ② `export/__init__.py` is 1,974 lines with four large functions — Medium

The single largest module in the subsystem, containing `format_summary_text` (232), `_build_summary_formatter` (228), `summary_frame_from_result` (158), `combined_summary_result` (129), `export_to_excel` (106), and `export_multi_period_metrics` (105).

A package `__init__.py` carrying ~2k lines of implementation is a structural problem on its own: it executes in full on `import trend_analysis.export`, it can't be split without touching every import site, and it gives no hint from the outside about what lives where.

The names also suggest overlapping responsibilities — `format_summary_text`, `_build_summary_formatter`, `summary_frame_from_result`, and `combined_summary_result` are four separate summary-shaping paths in one file.

**Recommendation:** move implementation into siblings (`export/summary.py`, `export/excel.py`, `export/multi_period.py`) and reduce `__init__.py` to re-exports. Worth auditing the four summary functions for genuine overlap while doing it.

### ③ `export_bundle` is 366 lines — Medium

`export/bundle.py:28` — the largest function in the subsystem, and 93% of its own module (393 lines total). A single function that is effectively the whole file.

**Recommendation:** split by output artifact; the function almost certainly has natural sections per emitted file.

### ④ Four more helpers duplicated across reporting modules — Medium

Beyond `_coerce_series`:

| Helper | Locations |
|--------|-----------|
| `_render_html` | `unified.py`, `quick_summary.py`, `reporting/run_artifacts.py` |
| `_turnover_chart` | `unified.py`, `quick_summary.py` |
| `_format_number` | `unified.py`, `reporting/narrative.py` |
| `_normalise_weights` | `reporting/narrative.py`, `reporting/portfolio_series.py` |

`unified.py` and `quick_summary.py` share at least four private helpers between them (`_coerce_series`, `_render_html`, `_turnover_chart`, `_format_number`). They're two renderers of the same underlying data with no shared helper module — `_matplotlib.py` shows the team already has the pattern for fixing this.

**Recommendation:** extend `trend/reporting/_matplotlib.py` into a small shared internals module (or add `_shared.py`) and pull these across, the same way matplotlib setup was consolidated.

### ⑤ `_build_summary_formatter` is a 228-line function wrapping a 217-line closure — Low

`export/__init__.py:437` returns `fmt_summary` (`:446`), which is 217 of the parent's 228 lines. The outer function exists almost entirely to close over `res`, `in_start`, `in_end`, `out_start`, `out_end`.

Same pattern flagged in the multi-period engine, at much smaller scale: a closure that can't be tested or called without going through its factory, where a small class or a `functools.partial` over a module-level function would expose the same behaviour testably.

**Recommendation:** make `fmt_summary` a module-level function taking those five values explicitly, and have the factory `partial` them in.

### ⑥ Two parallel reporting packages — Low

`trend/reporting/` (HTML/PDF report surfaces) and `trend_analysis/reporting/` (narrative, artifacts, portfolio series) are separate packages with the same name at different levels, and helpers leak between them (`_render_html`, `_coerce_series`, `_format_number` all appear in both).

The split may be deliberate — `trend/` as the user-facing layer, `trend_analysis/` as the library — but nothing states that, and the shared helper names suggest the boundary isn't being observed.

**Recommendation:** document the intended split in both `__init__.py` docstrings; the duplication in ④ is the symptom of it being unclear.

---

## Verdict

The standout here is a change the developers made on their own: **`_matplotlib.py` is exactly the right fix**, done in the right way — lazy import, `Agg` set before `pyplot`, shared rcParams, both consumers migrated. It's a small module that resolves a real duplication and demonstrates the pattern the rest of the codebase needs (notably `metrics/attribution.py`).

The findings cluster on the same theme as the surrounding work: **the two report renderers share four private helpers by copy rather than by import**, and one of those copies (`_coerce_series`, ①) has quietly diverged into opposite error behaviour. That's the one to fix first — it's a silent-failure path, not just untidiness.

Structurally, `export/` is the weak spot: a 1,974-line `__init__.py` and a 366-line function that is its own module. `viz/` is the opposite — 13 well-sized files with a consistent chart protocol.

Order: ① first (silent failure), then ④ using `_matplotlib.py` as the template since the pattern is already established, then ② and ③ as the larger structural work.
