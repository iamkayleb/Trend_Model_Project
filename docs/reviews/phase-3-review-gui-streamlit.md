# GUI / Streamlit
**Scope:** `streamlit_app/**` (32 files), `trend_analysis/gui/*`, `trend_analysis/ui/*`

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `streamlit_app/pages/2_Model.py` | 4617 | Model configuration page |
| `streamlit_app/pages/3_Results.py` | 2528 | Results page |
| `streamlit_app/pages/8_Validation.py` | 915 | Validation page |
| `trend_analysis/gui/app.py` | 913 | **Second GUI stack** (ParamStore/plugins based) |
| `streamlit_app/pages/1_Data.py` | 869 | Data upload page |
| `streamlit_app/monte_carlo_page.py` | 813 | Monte Carlo page |
| `streamlit_app/components/mc_plots.py` | 685 | MC plotting |
| `streamlit_app/components/analysis_runner.py` | 615 | Pipeline execution from UI |
| `streamlit_app/components/demo_runner.py` | 608 | Demo execution |
| `streamlit_app/components/comparison_export.py` | 602 | Comparison export |
| `streamlit_app/state.py` | 458 | Session state management |
| `trend_analysis/ui/rank_widgets.py` | 406 | Rank widget builder |
| *(21 more components/pages)* | ~4000 | Various |

**`streamlit_app/` total: 17,096 lines** — the largest subsystem in the repo.

Largest functions:

| Lines | Function | Location |
|------:|----------|----------|
| **1841** | `render_model_page` | `pages/2_Model.py:2764` |
| 484 | `render_results_page` | `pages/3_Results.py:2041` |
| 404 | `render_data_page` | `pages/1_Data.py:463` |
| 378 | `build_ui` | `ui/rank_widgets.py:29` |
| 358 | `render_validation_page` | `pages/8_Validation.py:545` |

---

## Positive / Clean

### `state.py` is a real abstraction, well designed
458 lines exposing a proper session-state API — `initialize_session_state`, `store_validated_data`, `get_uploaded_data`, `has_valid_upload`, `save_model_state`, `load_saved_model_state`, `rename_saved_model_state`. Named operations with types, not raw dictionary poking. This is the right way to manage Streamlit session state, and someone clearly thought about it.

The problem is that the pages largely ignore it — see finding ②.

### The component layer is reasonably decomposed
21 modules under `components/`, most between 250 and 700 lines, split by concern (`csv_validation`, `guardrails`, `llm_settings`, `comparison`, `data_schema`). The page files are the problem, not the components.

### `2_Model.py` does keep most logic in functions
Despite its size, 83% of its lines sit inside functions and there are only 35 top-level non-import statements. It isn't a script with logic sprayed at module scope — the structure exists, it's just that one function is enormous.

---

## Findings

### ① `render_model_page` is 1,841 lines — High

`pages/2_Model.py:2764-4605`. The second-largest function in the repo after the multi-period engine's threshold-hold loop, and 40% of an already-oversized 4,617-line file.

The file has **89 top-level functions**, so the author was clearly willing to extract helpers — this one function just never got the same treatment. At 1,841 lines it can't be read in one sitting, can't be tested, and any change to the Model page carries risk proportional to the whole page rather than the section being edited.

**Recommendation:** split by UI section. Streamlit pages decompose naturally along visual boundaries (each `st.expander` / `st.tabs` block is a candidate `render_*` function), and the file's existing 89 helpers show the pattern is already understood here.

### ② `state.py` exists but the two biggest pages bypass it — High

Direct `st.session_state` accesses:

| File | Accesses |
|------|---------:|
| `pages/2_Model.py` | **141** |
| `pages/1_Data.py` | **113** |
| `state.py` | 29 |
| `pages/3_Results.py` | 27 |
| `app.py` | 19 |

`2_Model.py` imports from `state` exactly **once**, then reaches into `st.session_state` directly 141 times.

This is worse than having no abstraction. A dedicated state module means a reader reasonably assumes state access is centralised and that `state.py` is the place to understand what keys exist and what shape they have. In practice the two pages that own the most state don't route through it, so:

- session-state keys are string literals scattered across ~250 call sites with no single declaration
- a typo'd key silently creates a new entry rather than failing
- `state.py`'s invariants (e.g. what `store_validated_data` guarantees) can be bypassed by any page writing the underlying key directly

`3_Results.py` (27 accesses) shows the discipline is achievable in this codebase.

**Recommendation:** treat `state.py` as the only writer. Migrate `2_Model.py` and `1_Data.py` incrementally — each raw `st.session_state["x"]` becomes a named accessor. Highest value is on writes; reads can follow.

### ③ `analysis_runner.py` reimplements seven helpers it could import — High

`components/analysis_runner.py:17` already imports from the shared config mapper:

```python
from trend_analysis.config.ui_mapping import METRIC_REGISTRY, build_config_from_ui_state
```

The dependency edge exists and is used. Yet the same file privately redefines seven functions that `ui_mapping` already provides publicly:

| `analysis_runner.py` | `config/ui_mapping.py` |
|----------------------|------------------------|
| `_coerce_positive_int` | `coerce_positive_int` |
| `_coerce_positive_float` | `coerce_positive_float` |
| `_month_end` | `month_end` |
| `_build_sample_split` | `build_sample_split` |
| `_build_signals_config` | `build_signals_config` |
| `_normalise_metric_weights` | `normalise_metric_weights` |
| `_build_portfolio_config` | `build_portfolio_config` |

And they have already drifted — `_build_portfolio_config` is 115 lines against `ui_mapping`'s 98.

This is the most consequential duplication I've found in the codebase, because these functions decide *what configuration the pipeline actually runs*. Two divergent copies of "translate UI state into portfolio config" means the Streamlit path and the shared path can construct different configs from the same user input, and nothing surfaces the difference.

**Recommendation:** delete the seven private copies and import the public equivalents on the line that already imports from that module. Before deleting, diff `_build_portfolio_config` against `build_portfolio_config` to see which behaviours the 17-line difference represents — some may be fixes that belong upstream in `ui_mapping`.

### ④ A second, near-orphaned GUI stack — Medium

`trend_analysis/gui/` (1,038 lines across `app.py`, `plugins.py`, `store.py`, `utils.py`) is a separate UI implementation built on `ParamStore` and a plugin-discovery mechanism, importing `pipeline`, `export`, and `weighting` directly.

Its only consumer outside itself is `scripts/run_multi_demo.py` (`:260`, `:296`, `:322`). No `pyproject.toml` entry point references it. `streamlit_app/` doesn't touch it.

So the repo carries two GUI codepaths: the live Streamlit app, and a 913-line legacy app kept alive by one demo script — with its own plugin system (`trend_analysis.gui_plugins` entry-point group) that nothing else uses.

**Recommendation:** decide explicitly. If the demo script is the only consumer, either move `gui/` next to it as demo-support code, or retire both. If it's a supported alternative UI, it needs an entry point and a docstring saying so — `app.py` currently opens with bare imports and no module docstring at all.

### ⑤ Page files are oversized across the board — Medium

`2_Model.py` (4,617) and `3_Results.py` (2,528) together are 7,145 lines — 42% of the Streamlit app in two files. `8_Validation.py` (915) and `1_Data.py` (869) follow.

The `components/` directory is the established home for extractable UI logic and is already well populated. The page files should mostly be composition — layout plus calls into components — rather than holding the bulk of the implementation.

**Recommendation:** as ① proceeds, push extracted sections into `components/` rather than leaving them as page-local helpers.

### ⑥ `ui/rank_widgets.py::build_ui` is 378 lines — Low

`src/trend_analysis/ui/rank_widgets.py:29` — the module is 406 lines and this function is 93% of it. Also note `ui/__init__.py` is empty (0 bytes), so the package exposes nothing deliberately.

**Recommendation:** split by widget group; populate `__init__.py` with the intended public surface.

---

## Verdict

The infrastructure here is better than the page code that uses it. `state.py` is a genuinely well-designed session-state API and `components/` is a sensible, well-populated layer — but the two largest pages route around both, holding 141 and 113 raw `st.session_state` accesses and, in one case, private reimplementations of seven shared config helpers.

That's the theme: **the right abstractions exist and are being bypassed.** It's a more tractable problem than missing structure, because the target is already built — the work is migration, not design.

Priority is unusual for this subsystem in that ③ outranks the size findings despite being smaller. `_build_portfolio_config` existing twice, already 17 lines apart, determines what config the pipeline runs — a divergence there produces wrong results silently, whereas a 1,841-line function is only expensive. Fix ③ first, then ② (start with writes), then ① and ⑤ together. ④ needs a decision more than it needs code.
