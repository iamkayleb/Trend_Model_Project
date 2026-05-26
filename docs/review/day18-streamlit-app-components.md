# Day 18 Code Review: streamlit_app/ — App entry, state, and components

**Scope:** The Streamlit application's entry point, session-state layer, and the
entire `components/` directory (the reusable infrastructure behind the pages).
23 files, roughly 6,400 lines. The large page modules — `pages/2_Model.py`
(4,565 lines), `pages/3_Results.py` (2,366), `pages/1_Data.py`, `pages/8_Validation.py`,
`pages/monte_carlo.py`, `pages/4_Help.py` — are deferred to Day 19.

---

## Files Reviewed

### App shell

- `streamlit_app/app.py` (236 lines) — The landing page. Puts the repo root on `sys.path`, renders the demo section (preset selector + parameter overrides) and a custom-analysis entry point, and dispatches to `run_demo_with_overrides` before switching to the Results page.
- `streamlit_app/state.py` (454 lines) — Session-state management: initialization, upload data storage, save/load/rename/delete/export/import of named model states and config wrappers, plus a recursive `diff_model_states` with tolerance-aware numeric comparison and a copy-friendly text formatter.
- `streamlit_app/config_bridge.py` (16 lines) — Deprecation shim re-exporting `build_config_payload` / `validate_payload` from `trend_analysis.config.bridge`; emits a `DeprecationWarning` on import.

### Data ingest & guardrails

- `streamlit_app/components/data_schema.py` (370 lines) — Core ingest: reads raw bytes (CSV/Excel), extracts original headers without pandas duplicate-mangling, sanitizes formula-prefixed headers, runs `validate_market_data`, and builds the `SchemaMeta` payload with frequency/missing-policy diagnostics.
- `streamlit_app/components/csv_validation.py` (285 lines) — Upload-flow CSV validation: structural checks (duplicate headers, empty data, row caps, missing columns), date parsing with an auto-correction path (`DateCorrectionNeeded`), and uniqueness checks. Strips CSV-injection formula prefixes.
- `streamlit_app/components/date_correction.py` (28 lines) — Deprecation shim re-exporting the date-correction helpers from `trend_analysis.io.date_correction`; emits a `DeprecationWarning`.
- `streamlit_app/components/guardrails.py` (270 lines) — Streamlit-independent guardrails: frequency inference, resource-usage estimation (memory/runtime heuristics + warnings), a minimal startup-payload validator, and a `prepare_dry_run_plan` that pulls a sample window from the *start* of the dataset to avoid look-ahead.
- `streamlit_app/components/upload_guard.py` (217 lines) — File-upload guardrails: extension allowlist, 10 MiB size cap (env-overridable), SHA-256 content hashing, and atomic persistence via `tempfile.mkstemp` + `os.replace` with a content-hash-derived filename.
- `streamlit_app/components/data_cache.py` (101 lines) — `@st.cache_data`-decorated dataset loaders (from path / from bytes), sample-dataset discovery, and a `cache_key_for_frame` helper.

### Analysis execution

- `streamlit_app/components/analysis_runner.py` (593 lines) — Translates UI model-state into a backend `Config`: builds the sample split (explicit vs. relative date modes), signals config, and runs the simulation; also drives A/B variant patches via `ConfigPatch`.
- `streamlit_app/components/demo_runner.py` (547 lines) — Demo pipeline: loads built-in demo returns, reads preset YAML, derives the analysis window, normalizes metric weights, and assembles both a policy config and a pipeline `Config`.
- `streamlit_app/components/policy_engine.py` (218 lines) — The selection policy: `PolicyConfig`, a `CooldownBook`, z-score ranking, and `decide_hires_fires` with sticky-rank/threshold gates, diversification caps, min-tenure guards, and a turnover budget.

### Comparison & explanation

- `streamlit_app/components/comparison.py` (258 lines) — A/B comparison primitives: deterministic run keys, metric winner detection, and ZIP packaging of comparison artifacts.
- `streamlit_app/components/comparison_export.py` (602 lines) — Builds comprehensive Excel/CSV comparison workbooks (summary, selection differences, side-by-side periods, raw data table).
- `streamlit_app/components/comparison_llm.py` (370 lines) — LLM-backed comparison narrative: extracts/compacts metric catalogs from both runs, builds a comparison prompt, invokes the configured LLM, and caches the result in session state.
- `streamlit_app/components/explain_results.py` (429 lines) — Single-run LLM explanation: metric extraction, prompt building, claim-issue post-processing, and session caching.
- `streamlit_app/components/nl_operation_viewer.py` (412 lines) — Renders natural-language operation logs (`NLOperationLog`) with `ConfigPatch` replay, including URL sanitization for displayed references.
- `streamlit_app/components/llm_settings.py` (221 lines) — API-key resolution with sanitization (placeholder rejection), Streamlit-secrets reads, Anthropic key precedence (`CLAUDE_API_STRANSKE` → `ANTHROPIC_API_KEY`), and `LLMProviderConfig` construction.

### Monte Carlo & charts

- `streamlit_app/components/mc_plots.py` (685 lines) — Monte Carlo plotting helpers (fan charts, distributions).
- `streamlit_app/components/mc_tables.py` (140 lines) — Monte Carlo summary tables via `aggregate_monte_carlo_results`, with Sharpe alias resolution and configurable quantiles.
- `streamlit_app/components/charts.py` (119 lines) — Shared Altair chart helpers with a fixed 5-colour palette.

### Small UI helpers

- `streamlit_app/components/explain_results.py` and `comparison_llm.py` cache via session state.
- `streamlit_app/components/progress_eta.py` (52 lines) — EWMA-style ETA estimator with min/max clamps for progress bars.
- `streamlit_app/components/disclaimer.py` (44 lines) — License/security disclaimer modal; reads URLs from env with hardcoded defaults.

---

## Issues Found

### 1. Deprecation shims are still imported by the app's own live modules

`config_bridge.py` and `date_correction.py` both warn on import and tell callers to switch to the `trend_analysis.*` source modules:

```python
# config_bridge.py:9-14
warnings.warn(
    "streamlit_app.config_bridge is deprecated; import from "
    "trend_analysis.config.bridge instead.",
    DeprecationWarning,
    stacklevel=2,
)
```

But sibling modules never made the switch:

```python
# guardrails.py:22  — imports the deprecated shim
from streamlit_app.config_bridge import build_config_payload, validate_payload

# csv_validation.py:18  — imports the deprecated shim
from streamlit_app.components.date_correction import (
    DateCorrection,
    analyze_date_column,
)
```

`pages/1_Data.py` also imports the deprecated `date_correction` shim. Because `guardrails` and `csv_validation` are imported on virtually every app run, the app emits its own `DeprecationWarning`s at startup, against its own advice. Meanwhile `analysis_runner.py:14` already uses the correct path (`from trend_analysis.config.bridge import ...`), so the inconsistency is purely a missed find-and-replace. Either update the three importers to the canonical modules (and delete the shims), or remove the warnings if the shims are meant to stay.

---

### 2. `data_cache.py` `_hash_dataframe` returns the whole serialized frame, not a hash

```python
# data_cache.py:52-56
def _hash_dataframe(df: pd.DataFrame) -> str:
    """Return a stable hash for a dataframe used as a cache key."""
    payload = df.to_json(date_format="iso", date_unit="ns", orient="split")
    return json.dumps({"data": payload})
```

Despite the name and docstring ("a stable hash … used as a cache key"), this returns the full JSON serialization of the entire DataFrame wrapped in another JSON envelope. For a real upload that "cache key" is the whole dataset re-encoded as a string — potentially several megabytes held as a dict key. `cache_key_for_frame` (used by `analysis_runner.py:21`) inherits the bloat. Every other hash in this codebase uses a digest (`upload_guard.hash_bytes`, `streamlit_param_sweep._hash_df` → `hashlib.sha256(...).hexdigest()[:12]`). The fix is the same here:

```python
payload = df.to_json(date_format="iso", date_unit="ns", orient="split")
return hashlib.sha256(payload.encode("utf-8")).hexdigest()
```

---

### 3. `csv_validation.py` line 284: `logger.exception` on every routine validation failure

```python
# csv_validation.py:283-285
    except CSVValidationError as err:
        logger.exception("CSV upload failed validation: %s", err)
        raise
```

`CSVValidationError` is raised for expected, user-facing problems — missing columns, non-unique dates, an empty file, a file that's too large. `logger.exception` logs at ERROR level *with a full traceback*. So a user who uploads a CSV with a typo'd header floods the logs with a stack trace for what is a normal, handled outcome. This makes genuine errors harder to find and inflates log volume. These should be `logger.info`/`logger.warning` without the traceback; reserve `logger.exception` for the unexpected `pandas` read failure at line 130.

---

### 4. `state.py` `_values_equal`: cross-type numeric values always compared unequal

```python
# state.py:316-320
def _values_equal(left: Any, right: Any, float_tol: float) -> bool:
    if type(left) is not type(right):
        return False
    if isinstance(left, numbers.Number) and isinstance(right, numbers.Number):
        return math.isclose(left, right, rel_tol=float_tol, abs_tol=float_tol)
```

The `type(left) is not type(right)` short-circuit runs before the numeric branch, so `_values_equal(10, 10.0, tol)` returns `False` — `int` and `float` are different types. In `diff_model_states`, that surfaces as a spurious `type_changed` diff. Model states routinely round-trip through JSON (`export_model_state` / `import_model_state`), where `10` may deserialize as `int` in one snapshot and `10.0` as `float` in another. Comparing a freshly-built state against an imported one will therefore report phantom "10 → 10.0 [type changed]" differences. The numeric `math.isclose` branch should be checked *before* the strict type comparison so equal numeric values compare equal regardless of int/float representation.

---

### 5. `state.py` `diff_model_states`: `zip_longest` import and sentinel defined after the function that uses them

```python
# state.py:386-388  — inside the nested _walk()
for idx, (item_left, item_right) in enumerate(
    zip_longest(left, right, fillvalue=_missing_sentinel)
):
...
# state.py:425-428  — at the end of diff_model_states, AFTER _walk is defined
_missing_sentinel = object()
from itertools import zip_longest
_walk(dict(config_a), dict(config_b), "")
```

This only works because `_walk` is a closure: `zip_longest` and `_missing_sentinel` are free variables resolved at call time (line 428), by which point both exist. It is correct today, but fragile — any future refactor that calls `_walk` earlier, or hoists it out of the function, raises `NameError`. The `from itertools import zip_longest` buried mid-function (after a nested `def`) is also surprising. Move the import to module top and define `_missing_sentinel` before `_walk`.

---

## Notes

- **`disclaimer.py` lines 9-15: hardcoded `stranske/Trend_Model_Project` URLs.** `LICENSE_URL` and `SECURITY_URL` default to the upstream `stranske` repo. In the `iamkayleb` fork these links point at the wrong repository unless the env vars are set. Same stale-namespace issue noted for `test_docker.sh` in Day 15.

- **`policy_engine.py` `allow_add`: sticky-rank gate blocks all hires when the caller doesn't maintain `rule_state`.**
  ```python
  if r == "sticky_rank_window" and int(policy.sticky_add_x) > 1:
      if int(add_streak.get(name, 0)) < int(policy.sticky_add_x):
          return False
  ```
  `add_streak` comes from `(rule_state or {}).get("add_streak", {})` and is never updated inside `decide_hires_fires`. If a caller sets `sticky_add_x > 1` but doesn't thread `rule_state` across periods, `add_streak` stays empty, every candidate's streak is 0 < `sticky_add_x`, and *no hires ever happen*. The cross-period streak bookkeeping is an implicit contract that isn't documented at the function boundary.

- **`llm_settings.py` `resolve_api_key_input`: a user typing a known key *name* triggers env/secret lookup.** If the user pastes the literal string `OPENAI_API_KEY` (rather than a key value) into the API-key field, the function interprets it as a reference and resolves the actual key from secrets/env. Clever, but surprising — and a real key that happened to equal one of the six magic names would be misresolved. Edge case, worth a docstring note.

- **`demo_runner.py` `_normalise_metric_weights` fallback doesn't sum to 1.0.** When all weights are non-positive, it returns `{"sharpe": 1/3, "return_ann": 1/3, "drawdown": 1/3}` — three floats that sum to 0.9999…, not 1.0. The non-fallback path normalizes by total, so only the fallback is slightly off. Cosmetic unless downstream code asserts the weights sum to exactly 1.

- **`guardrails.py` `estimate_resource_usage`: magic-number heuristics.** "75,000 cell operations per second", "8 bytes × 1.5 safety multiplier", "512 MB", "150 columns" are all undocumented constants. They drive user-facing warnings, so when they drift from reality the warnings either over- or under-fire. Worth naming the constants and citing where the numbers came from.

- **`charts.py`: fixed 5-colour `PALETTE`.** Any chart with more than five series will reuse colours (Altair cycles the scale), which can make two distinct funds indistinguishable. Fine for the demo's small fund counts, but a dataset with >5 selected funds in one chart loses visual separation.

- **`upload_guard.py` is a model of careful I/O.** Temp file written then `os.replace`'d atomically, content-hash-derived filenames to dedupe, extension allowlist, env-overridable size cap, cleanup-on-failure. No issues — calling it out as the strongest file in the directory.

- **CSV-injection defense is consistent and correct.** Both `data_schema.py` (`DANGEROUS_HEADER_PREFIXES`) and `csv_validation.py` (`_strip_formula_prefix`) strip leading `= + - @` from header cells before they can be re-exported into Excel. Good security hygiene; just note the two implementations are independent and could drift — a shared helper would keep them in lockstep.

- **`comparison_llm.py` / `explain_results.py` timestamps are timezone-aware.** Both use `datetime.now(timezone.utc).isoformat()`, avoiding the naive-datetime issue flagged in `generate_settings_evidence.py` (Day 16). Consistent and correct.

- **`data_cache.py` `list_sample_datasets` / `dataset_choices` are `@st.cache_data`-decorated and depend on disk state.** `_available_demo_files` is cached, so if demo files are generated *after* first call (e.g., a test that runs `generate_demo.py` mid-session), the cached empty list persists until `clear_cache()` (which only clears the two loaders, not the discovery functions). Minor staleness risk in test scenarios.

- **`app.py` runs entirely at module scope.** This is the standard Streamlit page model (no `main()` guard), so it's idiomatic rather than a defect — but it means `st.set_page_config` must remain the first Streamlit call, a constraint that's currently satisfied and worth preserving when editing.

- **`nl_operation_viewer.py` sanitizes URLs via `urlsplit`/`urlunsplit`.** Good practice for displaying user/LLM-provided references, preventing `javascript:` and similar scheme injection in rendered links.
