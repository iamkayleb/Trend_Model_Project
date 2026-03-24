# Day 2 Function Reference: Configuration System

**Date:** 2026-03-24
**Scope:** All classes, functions, and helpers across the 12 files reviewed in the Day 2 config-system audit.

---

## 1. `src/trend/config_schema.py` (Tier 1 — stdlib-only)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 54 | `CoreConfigError` | class | `ValueError` subclass raised by Tier-1 validation |
| 59 | `DataSettings` | dataclass | Core data fields: `csv_path`, `managers_glob`, `date_column`, `frequency`, `universe_membership_path` |
| 70 | `CostSettings` | dataclass | `bps_per_trade`, `slippage_bps` |
| 81 | `CoreConfig` | dataclass | Top-level container: `data: DataSettings`, `cost: CostSettings` |
| 120 | `_as_mapping()` | helper | Coerce a value to `Mapping[str, Any]` or raise |
| 126 | `_normalise_path()` | helper | Resolve and validate a file path (existence, traversal, glob rejection) |
| 159 | `_normalise_glob()` | helper | Validate a glob pattern matches ≥1 CSV file |
| 191 | `_normalise_string()` | helper | Coerce to non-empty string with default |
| 202 | `_normalise_frequency()` | helper | Map frequency aliases → `D`, `W`, `M`, `ME` |
| 216 | `_coerce_float()` | helper | Parse a float with field-name context in errors |
| 226 | `validate_core_config()` | **public** | Entry point: dict → `CoreConfig`. Called by `bridge.py` and `cli.py --check` |
| 323 | `load_core_config()` | **public** | YAML file → `CoreConfig` (resolves path, reads, calls `validate_core_config`) |

---

## 2. `src/trend_analysis/config/model.py` (Tier 2 — Pydantic startup)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 38 | `_resolve_path()` | helper | Resolve relative/absolute path against multiple candidate roots; enforces traversal guard |
| 116 | `_candidate_roots()` | helper | Yield search roots for relative path resolution |
| 135 | `_expand_pattern()` | helper | Expand a glob pattern against candidate roots |
| 154 | `_ensure_glob_matches()` | helper | Assert glob resolves to ≥1 CSV file |
| 186 | `DataSettings` | Pydantic model | 9 fields (superset of Tier-1): adds `missing_policy`, `missing_limit`, `risk_free_column`, `allow_risk_free_fallback`. Field validators for every field. Model validator `_ensure_source` requires `csv_path` or `managers_glob`. |
| 318 | `CostModelSettings` | Pydantic model | `bps_per_trade`, `slippage_bps`, `per_trade_bps`, `half_spread_bps` with non-negative validation |
| 347 | `PortfolioSettings` | Pydantic model | `rebalance_calendar`, `max_turnover` (0–1), `transaction_cost_bps` (≥0), `lambda_tc` (0–1), `min_tenure_n` (≥0), `ci_level` (0–1), `cooldown_periods`/`cooldown_months` (≥0), `cost_model`, `turnover_cap`, `weight_policy` |
| 488 | `RiskSettings` | Pydantic model | `target_vol` (>0), `floor_vol` (≥0), `warmup_periods` (≥0) |
| 531 | `TrendConfig` | Pydantic model | Top-level: `data: DataSettings`, `portfolio: PortfolioSettings`, `vol_adjust: RiskSettings` |
| 546 | `_resolve_config_path()` | helper | Locate config YAML by name, env var, or `config/` directory fallback |
| 571 | `validate_trend_config()` | **public** | dict → `TrendConfig` with user-friendly error formatting |
| 590 | `load_trend_config()` | **public** | YAML file → `(TrendConfig, Path)` |

---

## 3. `src/trend_analysis/config/legacy.py` (Tier 3 — raw Config)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 37 | `Config` | Pydantic model | Flat container with every top-level section (`data`, `portfolio`, `vol_adjust`, `signals`, `benchmarks`, `regime`, `robustness`, `metrics`, `export`, `run`, `seed`, `multi_period`, `sample_split`, `preprocessing`). Migrates `output` → `export`. |
| 80 | `load()` | **public** | YAML file → `Config` (resolves path, reads, constructs) |

---

## 4. `src/trend_analysis/config/models.py` (Full Config — Pydantic-or-fallback)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 48 | `_ValidateConfigFn` | Protocol | Callable signature for `validate_trend_config` |
| 53 | `ConfigProtocol` | Protocol | Duck-type interface for Config objects (used by pipeline) |
| 124 | `SimpleBaseModel` | class | Fallback base when Pydantic is unavailable; provides `get()`, `setdefault()`, dict-like access |
| 147 | `_find_config_directory()` | helper | Locate `config/` directory from repo root |
| 164 | `_validate_version_value()` | helper | Coerce version to string `"1"` |
| 567 | `PresetConfig` | model | Preset configuration (name, description, trend params) |
| 608 | `ColumnMapping` | model | Column rename/alias mapping |
| 664 | `ConfigurationState` | model | Tracks active config path + dirty flag |
| 687 | `load_preset()` | **public** | Load a named preset from `config/presets/` |
| 705 | `list_available_presets()` | **public** | List available preset YAML files |
| 733 | `load_config()` | **public** | Load config from dict, string, or Path → `ConfigProtocol`. Calls `validate_trend_config` internally. |
| 768 | `load()` | **public** | Convenience wrapper around `load_config()` |

---

## 5. `src/trend_analysis/config/bridge.py` (Streamlit ↔ CLI sync)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 13 | `build_config_payload()` | **public** | Assemble a config dict from Streamlit session state for CLI-compatible validation |
| 62 | `validate_payload()` | **public** | Validate a payload dict via `validate_core_config` (Tier 1 only). Returns validated dict or raises `CoreConfigError`. |

---

## 6. `src/trend_analysis/config/ui_mapping.py` (Streamlit UI → Config)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 22 | `coerce_positive_int()` | helper | Silent coercion to int with default and minimum |
| 30 | `coerce_positive_float()` | helper | Silent coercion to float with default |
| 38 | `month_end()` | helper | Snap a timestamp to month-end |
| 44 | `build_sample_split()` | helper | Derive `sample_split` dict from DatetimeIndex + model state |
| 136 | `build_signals_config()` | helper | Assemble `signals` config section from trend spec dict |
| 183 | `normalise_metric_weights()` | helper | Normalize metric weight dict to sum to 1.0 |
| 206 | `build_portfolio_config()` | helper | Assemble `portfolio` config section from model state + weights |
| 303 | `build_config_from_ui_state()` | **public** | Main entry: Streamlit widget state → complete `Config` object (~270 lines). No Pydantic validation — relies on inline coercion. |

---

## 7. `src/trend_analysis/config/patch.py` (NL-driven config mutation)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 36 | `RiskFlag` | enum | `REMOVES_CONSTRAINT`, `REMOVES_VALIDATION`, `INCREASES_LEVERAGE`, `BROAD_SCOPE` |
| 43 | `PatchOperation` | Pydantic model | Single mutation: `op` (set/delete/merge), `path` (JSON pointer), `value` |
| 82 | `ConfigPatch` | Pydantic model | Container: `operations: list[PatchOperation]`, `description`, `risk_flags` |
| 123 | `risky_patch_flags()` | **public** | Detect risk flags on a patch |
| 132 | `apply_patch()` | **public** | Apply a `ConfigPatch` to a config dict (in-place) |
| 154 | `apply_config_patch()` | **public** | Alias for `apply_patch` |
| 160 | `diff_configs()` | **public** | Human-readable diff between two config dicts |
| 179 | `apply_and_diff()` | **public** | Load YAML → apply patch → return (new_config, diff_string) |
| 189 | `apply_and_validate()` | **public** | Apply patch then run `validate_config` on result |
| 199 | `parse_config_patch()` | **public** | Parse LLM response text → `ConfigPatch` |
| 205 | `parse_config_patch_with_retries()` | **public** | Retry wrapper around `parse_config_patch` |
| 238 | `_to_dotpath()` | helper | Convert JSON pointer `/a/b` to dot path `a.b` |
| 243 | `_parse_path_segments()` | helper | Split path string into string/int segments |
| 252 | `_strip_code_fence()` | helper | Remove markdown code fences from LLM output |
| 261 | `_parse_dotpath()` | helper | Parse dot-path `a.b.0.c` into segments |
| 275 | `_format_dotpath()` | helper | Segments → dot-path string |
| 288 | `_resolve_parent()` | helper | Walk config dict to find parent container for a path |
| 335 | `_apply_operation()` | helper | Execute a single set/delete/merge on a resolved parent |
| 409 | `_deep_merge()` | helper | Recursive dict merge |
| 417 | `_has_invalid_json_pointer_escape()` | helper | Detect malformed `~` escapes |
| 430 | `_path_depth()` | helper | Count segments in a path |
| 434 | `_detect_risk_flags()` | helper | Scan operations for risk conditions |
| 449 | `_removes_constraint()` | helper | Check if op deletes a constraint field |
| 461 | `_removes_validation()` | helper | Check if op disables validation |
| 475 | `_increases_leverage()` | helper | Check if `vol_adjust.target_vol > 0.15` |
| 486 | `_is_broad_scope()` | helper | Check if op replaces an entire top-level section |
| 492 | `_collect_paths()` | helper | Recursively collect all dot-paths in a nested dict |
| 507 | `_path_error()` | helper | Format a path-resolution error |
| 515 | `format_retry_error()` | **public** | Format error message for retry prompt |

---

## 8. `src/trend_analysis/config/validation.py` (Full structural validation)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 23 | `ValidationError` | Pydantic model | Issue DTO: `path`, `message`, `expected`, `actual`, `suggestion` (name collides with `pydantic.ValidationError`) |
| 31 | `ValidationResult` | Pydantic model | Container: `valid: bool`, `errors: list[ValidationError]` |
| 66 | `validate_config()` | **public** | Orchestrator: runs schema → required → sample-split → portfolio passes. Returns `ValidationResult`. |
| 116 | `_run_schema_validation()` | pass | JSON Schema (Draft 2020-12) validation |
| 124 | `_run_required_validation()` | pass | Hand-coded required-field checks (partially redundant with schema) |
| 135 | `_run_sample_split_validation()` | pass | Business-logic date ordering |
| 143 | `_run_portfolio_validation()` | pass | Portfolio selection mode cross-field checks |
| 156 | `format_validation_messages()` | **public** | Format `ValidationResult` into user-facing strings |
| 169 | `_collect_schema_errors()` | helper | Gather JSON Schema errors into `ValidationError` list |
| 182 | `_schema_error_to_issues()` | helper | Convert single jsonschema error → list of issues |
| 219 | `_suggest_additional_property()` | helper | Suggest closest valid key for unknown properties |
| 235 | `_expected_for_error()` | helper | Extract expected-value hint from schema error |
| 262 | `_suggestion_for_error()` | helper | Build actionable suggestion string |
| 286 | `_missing_required_field()` | helper | Extract field name from "required" error message |
| 291 | `_unexpected_property()` | helper | Extract field name from "additional properties" error |
| 296 | `_check_required_sections()` | checker | Verify top-level sections (`data`, `portfolio`, `vol_adjust`) exist |
| 319 | `_check_required_fields()` | checker | Verify required fields within each section |
| 333 | `_check_version_field()` | checker | Verify `version` field is present and valid |
| 366 | `_require_field()` | helper | Generic required-field check with path/message assembly |
| 397 | `_is_present()` | helper | Check if a value is non-null, non-empty |
| 405 | `_check_data_required_fields()` | checker | Required fields for `data` section |
| 453 | `_check_portfolio_required_fields()` | checker | Required fields for `portfolio` section |
| 490 | `_check_vol_adjust_required_fields()` | checker | Required fields for `vol_adjust` section |
| 503 | `_collect_trend_model_errors()` | helper | Collect `validate_trend_config` errors as `ValidationError` items |
| 514 | `_check_date_ranges()` | checker | Validate date ordering in `sample_split` |
| 575 | `_check_sample_split_requirements()` | checker | Validate sample-split window lengths and date consistency |
| 628 | `_check_rank_fund_count()` | checker | Validate `selection_count` ≤ available funds |
| 673 | `_check_portfolio_selection_requirements()` | checker | Cross-field validation for portfolio selection mode |
| 704 | `_check_manual_selection_requirements()` | checker | Validate manual-selection fund lists |
| 755 | `_check_rank_inclusion_requirements()` | checker | Validate rank-inclusion mode constraints |
| 812 | `_check_rank_value_ranges()` | checker | Validate rank-mode numeric ranges |
| 853 | `_count_available_funds()` | helper | Count funds from CSV/glob for capacity checks |
| 885 | `_resolve_path()` | helper | Local path resolver for validation context |
| 892 | `_error_from_exception()` | helper | Convert exception → `ValidationError` DTO |
| 911 | `_actual_from_path()` | helper | Extract nested value from config by dot-path |
| 956 | `_format_path()` | helper | Format path parts into dot-path string |
| 969 | `_join_path()` | helper | Join two path segments |
| 975 | `_format_issue()` | helper | Format a single `ValidationError` for display |
| 982 | `_format_actual()` | helper | Truncate/format actual values for error messages |
| 992 | `_append_issue()` | helper | Deduplicate and append issue to list |

---

## 9. `src/trend_analysis/config/schema_validation.py` (JSON Schema runner)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 17 | `load_schema()` | **public** | Load `config.schema.json` from disk |
| 24 | `load_config()` | **public** | Load a YAML config file into a dict |
| 35 | `_format_error()` | helper | Format a jsonschema error for display |
| 41 | `validate_config_data()` | **public** | Validate dict against schema, return list of error strings |
| 49 | `validate_config_file()` | **public** | End-to-end: file path → list of error strings |

---

## 10. `src/trend_analysis/config/schema_generator.py` (Schema generation)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 231 | `_load_model_overrides()` | helper | Load constraint overrides from `config_map.yml` |
| 282 | `collect_config_sources()` | helper | Find all YAML files in `config/` directory |
| 290 | `load_yaml()` | helper | Safe YAML load with encoding |
| 301 | `merge_defaults()` | helper | Recursive merge of two config dicts |
| 317 | `gather_samples()` | helper | Merge multiple config files into combined sample map |
| 347 | `_find_inline_comment()` | helper | Extract YAML inline comment for descriptions |
| 362 | `extract_inline_comments()` | helper | Parse all inline comments from a YAML file |
| 388 | `_sanitize_description()` | helper | Clean description strings |
| 396 | `_schema_root_description()` | helper | Build root schema description from config map |
| 407 | `_infer_type()` | helper | Infer JSON Schema type from Python value |
| 423 | `_infer_array_items()` | helper | Infer array item schema |
| 439 | `_infer_schema_type()` | helper | Infer type with override support |
| 447 | `_description_for()` | helper | Build description for a schema property |
| 462 | `_constraints_for()` | helper | Build min/max/enum constraints for a property |
| 474 | `_nl_editable()` | helper | Check if a path is NL-patch editable |
| 481 | `build_schema()` | **public** | Build full JSON Schema from defaults + samples + overrides |
| 546 | `_apply_constraints()` | helper | Apply override constraints to a schema property |
| 565 | `generate_schema()` | **public** | End-to-end: config dir → JSON Schema dict |
| 601 | `_compact_schema()` | helper | Strip descriptions/examples for compact output |
| 615 | `write_schema_files()` | **public** | Write full + compact schema to disk |

---

## 11. `src/trend_analysis/config/coverage.py` (Runtime coverage tracking)

| Line | Symbol | Kind | Purpose |
|------|--------|------|---------|
| 21 | `ConfigCoverageReport` | dataclass | Report: `validated_keys`, `read_keys`, `unvalidated_reads` |
| 37 | `ConfigCoverageTracker` | class | Tracks which config keys are validated vs read at runtime |
| 78 | `activate_config_coverage()` | **public** | Set global singleton tracker |
| 83 | `deactivate_config_coverage()` | **public** | Clear global singleton |
| 88 | `get_config_coverage_tracker()` | **public** | Retrieve active tracker (or `None`) |
| 92 | `compute_schema_validity()` | **public** | Compute fraction of read keys that were validated |
| 100 | `_TrackedMapping` | class | `MutableMapping` wrapper that records key access |
| 176 | `wrap_config_for_coverage()` | **public** | Wrap a config object with `_TrackedMapping` for tracking |

---

## 12. `src/trend_analysis/config/__init__.py` (Re-exports)

| Line | Symbols re-exported | Source |
|------|---------------------|--------|
| 4 | `TrendConfig`, `load_trend_config`, `validate_trend_config` | `model.py` |
| 5–16 | `DEFAULTS`, `ColumnMapping`, `Config`, `ConfigType`, `ConfigurationState`, `PresetConfig`, `list_available_presets`, `load`, `load_config`, `load_preset` | `models.py` |
| 17–26 | `ConfigPatch`, `PatchOperation`, `RiskFlag`, `apply_and_diff`, `apply_and_validate`, `apply_config_patch`, `apply_patch`, `diff_configs` | `patch.py` |
| 27–32 | `ValidationError`, `ValidationResult`, `format_validation_messages`, `validate_config` | `validation.py` |

**Notable omissions from `__all__`:** `validate_core_config`, `CoreConfig`, `CoreConfigError` (Tier 1) — these are only importable directly from `trend.config_schema`.

---

## Cross-Reference: Validation Call Chain

```
validate_config()                          validation.py:66
  ├→ _run_schema_validation()              validation.py:116  (JSON Schema)
  ├→ _run_required_validation()            validation.py:124  (hand-coded, partially redundant)
  │    ├→ _check_required_sections()       validation.py:296
  │    ├→ _check_required_fields()         validation.py:319
  │    ├→ _check_version_field()           validation.py:333
  │    └→ _collect_trend_model_errors()    validation.py:503
  │         └→ validate_trend_config()     model.py:571      (Pydantic TrendConfig)
  ├→ _run_sample_split_validation()        validation.py:135
  │    ├→ _check_date_ranges()             validation.py:514
  │    └→ _check_sample_split_requirements() validation.py:575
  └→ _run_portfolio_validation()           validation.py:143
       ├→ _check_portfolio_selection_requirements() validation.py:673
       ├→ _check_manual_selection_requirements()    validation.py:704
       ├→ _check_rank_inclusion_requirements()      validation.py:755
       ├→ _check_rank_value_ranges()                validation.py:812
       └→ _check_rank_fund_count()                  validation.py:628

validate_core_config()                     config_schema.py:226  (Tier 1, independent)
validate_payload()                         bridge.py:62
  └→ validate_core_config()                config_schema.py:226

build_config_from_ui_state()               ui_mapping.py:303    (no Pydantic, inline coercion only)
```
