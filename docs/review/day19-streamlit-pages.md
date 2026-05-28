# Day 19 Code Review — `streamlit_app/pages/`

## Files Reviewed

- `pages/1_Data.py` — Data upload, date-correction approval flow, fund column selection table, and sample dataset autoload
- `pages/2_Model.py` — Main model configuration page: LLM-powered config chat panel, parameter widgets, what-if variant runner, saved-state management
- `pages/3_Results.py` — Results display: trailing stats, period-by-period breakdown, manager changes, charts, A/B comparison tab, export bundle builder
- `pages/4_Help.py` — Static reference page explaining every configuration parameter and available weighting schemes
- `pages/8_Validation.py` — Developer tool for verifying each UI setting is wired to the analysis pipeline; runs baseline vs. variant pairs
- `pages/monte_carlo.py` — Monte Carlo simulation page: scenario registry, run overrides, fan charts, diagnostic charts, download bundles

---

## Issues Found

### 1. `8_Validation.py:893` — `if __name__ == "__main__"` guard means the page renders blank in Streamlit

The module's only top-level call to `render_validation_page()` is wrapped in the standard Python entry-point guard:

```python
if __name__ == "__main__":
    render_validation_page()
```

Streamlit runs page modules by importing them, not by executing them directly, so `__name__` is never `"__main__"` inside the Streamlit runner. The function is never called and the page renders completely blank. Every other page in the app uses either an unconditional top-level call (`1_Data.py:837`, `4_Help.py:246`) or the guarded `_should_auto_render()` pattern that checks for an active Streamlit session context (`2_Model.py:4564`, `3_Results.py:2365`, `monte_carlo.py:645`):

```python
# Correct pattern used everywhere else:
if _should_auto_render():
    render_model_page()
```

In addition, inside `render_validation_page()` itself, `st.set_page_config()` is called at line 529, well after the function body has begun executing. Streamlit requires `set_page_config` to be the first Streamlit command in the script. If the entry-point guard is ever fixed, this will immediately raise `StreamlitAPIException`. The call should be moved to module top level, outside the function.

---

### 2. `8_Validation.py:543` — reads session key `"app_data"` that is never written anywhere

Even once the entry-point guard is corrected, the page will immediately bail out because it reads returns data from the wrong session state key:

```python
app_data = st.session_state.get("app_data")
if app_data is None or app_data.get("returns") is None:
    st.warning("⚠️ Please load data on the Data page first.")
    st.stop()

returns = app_data["returns"]
```

A search of the entire `streamlit_app/` directory confirms `"app_data"` is never written anywhere — it is read only here. The rest of the app stores the returns dataframe under `"returns_df"` via `app_state.store_validated_data()`, and retrieves it via `app_state.get_uploaded_data()`. The page should be rewritten to use:

```python
returns, _ = app_state.get_uploaded_data()
if returns is None:
    st.warning("⚠️ Please load data on the Data page first.")
    st.stop()
```

---

### 3. `8_Validation.py:498` — calls private `_execute_analysis` directly

`run_test_analysis` bypasses the public API and calls the underscore-prefixed internal function:

```python
result = analysis_runner._execute_analysis(payload)
```

The public interface is `analysis_runner.run_analysis(returns, model_state, benchmark, data_hash=...)`, which adds caching, error normalization, and any future retry logic. Calling the private function means validation tests skip those layers and will silently break if the internal signature changes. The function should be rewritten to call `run_analysis` and construct its arguments the same way other callers do.

---

### 4. `3_Results.py:1370` — `DataFrame.applymap` is deprecated since pandas 2.1

Inside `_render_period_breakdown`, the weights pivot table is formatted with `applymap`:

```python
pivot_fmt = pivot.applymap(lambda x: _fmt_pct(float(x), 1))
```

`applymap` was renamed to `map` in pandas 2.1.0 and emits `FutureWarning` on every page render. This warning gets printed to the terminal and, on some Streamlit setups, surfaced in the app logs. The fix is a one-word change:

```python
pivot_fmt = pivot.map(lambda x: _fmt_pct(float(x), 1))
```

---

### 5. `1_Data.py:735-750` — internal performance telemetry always visible to users

The fund selection section displays raw timing measurements via an unconditional `st.caption`:

```python
# Always-visible measurements (so you don't need to open Debug).
st.caption(
    " | ".join([
        f"Perf: total {(_t_render_done - _t_fund_start) * 1000:.0f}ms",
        f"render {(_t_render_done - _t_render_start) * 1000:.0f}ms",
        f"seed {(_t_seed_done - _t_seed_start) * 1000:.0f}ms",
        f"funds {len(available_funds)}",
        f"selected {n_selected}",
        ...
    ])
)
```

The comment itself says this is intentionally always-on ("so you don't need to open Debug"), but the output — internal variable names like `init_applied`, `defaults_seeded`, `default_selected`, raw millisecond timings — is developer-facing noise that confuses end users. The duplicate debug expander below (lines 793–834) already captures all of this information in a collapsed view. The always-visible caption should be removed or moved inside that expander.

---

## Notes

- `4_Help.py:242` — the documentation link still points to `stranske/Trend_Model_Project` instead of `iamkayleb/Trend_Model_Project`, the same stale URL noted in the Day 18 review of `disclaimer.py`.

- `3_Results.py:609-618` — `_format_change_reason` has a dead conditional branch. Both the `added` and `dropped` arms inside the z-score block return the same string `f"{base_reason} (z={z_val:.2f})"`, making the `if action == "dropped"` check pointless.

- `2_Model.py:1627-1650` and `3_Results.py:151-171` — `_current_run_key` is defined nearly identically in both files. It should live in a shared utility module (e.g., `components/run_key.py`) so changes to the key format stay in sync.

- `monte_carlo.py:59-67` — the `_cache_data` shim wraps `st.cache_data` in a conditional that returns an identity decorator when the attribute is absent. Modern Streamlit (1.x+) always has `cache_data`, so this guard is dead code, but it's harmless.

- `3_Results.py:48-54` and `2_Model.py:4555-4561` — the `_should_auto_render` function is also duplicated between both files. It too belongs in a shared module or in `streamlit_app/__init__.py`.
