# Day 2 Function Reference: Configuration System

**Date:** 2026-03-24
**Scope:** All classes, functions, and helpers across the 12 files reviewed in the Day 2 config-system audit.

---

## `src/trend/config_schema.py` (Tier 1 — stdlib-only)

- `src/trend/config_schema.py:#L54-L55` - `CoreConfigError` - ValueError subclass raised by Tier-1 validation
- `src/trend/config_schema.py:#L59-L66` - `DataSettings` - Core data fields dataclass: csv_path, managers_glob, date_column, frequency, universe_membership_path
- `src/trend/config_schema.py:#L70-L77` - `CostSettings` - Cost fields dataclass: bps_per_trade, slippage_bps
- `src/trend/config_schema.py:#L81-L117` - `CoreConfig` - Top-level Tier-1 container: data (DataSettings) + cost (CostSettings), with to_payload() serializer
- `src/trend/config_schema.py:#L120-L123` - `_as_mapping()` - Coerce a value to Mapping[str, Any] or raise with field context
- `src/trend/config_schema.py:#L126-L156` - `_normalise_path()` - Resolve and validate a file path (existence, traversal guard, glob rejection)
- `src/trend/config_schema.py:#L159-L188` - `_normalise_glob()` - Validate a glob pattern matches at least one CSV file
- `src/trend/config_schema.py:#L191-L199` - `_normalise_string()` - Coerce value to non-empty string with field-specific default
- `src/trend/config_schema.py:#L202-L213` - `_normalise_frequency()` - Map frequency aliases to canonical D, W, M, ME values
- `src/trend/config_schema.py:#L216-L223` - `_coerce_float()` - Parse a float with field-name context in error messages
- `src/trend/config_schema.py:#L226-L320` - `validate_core_config()` - Entry point: dict → CoreConfig; called by bridge.py and cli.py --check
- `src/trend/config_schema.py:#L323-L331` - `load_core_config()` - YAML file → CoreConfig (resolves path, reads, calls validate_core_config)

---

## `src/trend_analysis/config/model.py` (Tier 2 — Pydantic startup)

- `src/trend_analysis/config/model.py:#L38-L108` - `_resolve_path()` - Resolve relative/absolute path against multiple candidate roots; enforces traversal guard
- `src/trend_analysis/config/model.py:#L116-L132` - `_candidate_roots()` - Yield search roots (base_dir, parent, repo root, cwd) for relative path resolution
- `src/trend_analysis/config/model.py:#L135-L151` - `_expand_pattern()` - Expand a glob pattern against candidate roots
- `src/trend_analysis/config/model.py:#L154-L178` - `_ensure_glob_matches()` - Assert glob resolves to at least one CSV file
- `src/trend_analysis/config/model.py:#L186-L315` - `DataSettings` - Pydantic model with 9 fields (superset of Tier-1); field validators for every field; model validator _ensure_source requires csv_path or managers_glob
- `src/trend_analysis/config/model.py:#L318-L344` - `CostModelSettings` - Pydantic model for bps_per_trade, slippage_bps, per_trade_bps, half_spread_bps with non-negative validation
- `src/trend_analysis/config/model.py:#L347-L485` - `PortfolioSettings` - Pydantic model: rebalance_calendar, max_turnover (0-1), transaction_cost_bps (>=0), lambda_tc (0-1), min_tenure_n (>=0), ci_level (0-1), cooldown fields, cost_model, turnover_cap, weight_policy
- `src/trend_analysis/config/model.py:#L488-L528` - `RiskSettings` - Pydantic model: target_vol (>0), floor_vol (>=0), warmup_periods (>=0)
- `src/trend_analysis/config/model.py:#L531-L538` - `TrendConfig` - Top-level Pydantic model: data (DataSettings), portfolio (PortfolioSettings), vol_adjust (RiskSettings)
- `src/trend_analysis/config/model.py:#L546-L568` - `_resolve_config_path()` - Locate config YAML by name, env var, or config/ directory fallback
- `src/trend_analysis/config/model.py:#L571-L587` - `validate_trend_config()` - dict → TrendConfig with user-friendly error formatting
- `src/trend_analysis/config/model.py:#L590-L600` - `load_trend_config()` - YAML file → (TrendConfig, Path)

---

## `src/trend_analysis/config/legacy.py` (Tier 3 — raw Config)

- `src/trend_analysis/config/legacy.py:#L37-L74` - `Config` - Pydantic model flat container with every top-level section; migrates output → export
- `src/trend_analysis/config/legacy.py:#L80-L108` - `load()` - YAML file → Config (resolves path, reads, constructs)

---

## `src/trend_analysis/config/models.py` (Full Config — Pydantic-or-fallback)

- `src/trend_analysis/config/models.py:#L48-L50` - `_ValidateConfigFn` - Protocol defining callable signature for validate_trend_config
- `src/trend_analysis/config/models.py:#L53-L79` - `ConfigProtocol` - Protocol duck-type interface for Config objects used by pipeline
- `src/trend_analysis/config/models.py:#L124-L144` - `SimpleBaseModel` - Fallback base when Pydantic unavailable; provides get(), setdefault(), dict-like access
- `src/trend_analysis/config/models.py:#L147-L161` - `_find_config_directory()` - Locate config/ directory from repo root
- `src/trend_analysis/config/models.py:#L164-L177` - `_validate_version_value()` - Coerce version field to string "1"
- `src/trend_analysis/config/models.py:#L204-L380` - `_PydanticConfigImpl` - Internal Pydantic-backed Config implementation with field validators and dict-field introspection
- `src/trend_analysis/config/models.py:#L403-L556` - `_FallbackConfig` - Internal fallback Config implementation when Pydantic is unavailable
- `src/trend_analysis/config/models.py:#L567-L605` - `PresetConfig` - Preset configuration model (name, description, trend params)
- `src/trend_analysis/config/models.py:#L608-L661` - `ColumnMapping` - Column rename/alias mapping model
- `src/trend_analysis/config/models.py:#L664-L684` - `ConfigurationState` - Tracks active config path + dirty flag
- `src/trend_analysis/config/models.py:#L687-L702` - `load_preset()` - Load a named preset from config/presets/
- `src/trend_analysis/config/models.py:#L705-L717` - `list_available_presets()` - List available preset YAML files
- `src/trend_analysis/config/models.py:#L733-L765` - `load_config()` - Load config from dict, string, or Path → ConfigProtocol; calls validate_trend_config internally
- `src/trend_analysis/config/models.py:#L768-L855` - `load()` - Convenience wrapper around load_config()

---

## `src/trend_analysis/config/bridge.py` (Streamlit ↔ CLI sync)

- `src/trend_analysis/config/bridge.py:#L13-L59` - `build_config_payload()` - Assemble a config dict from Streamlit session state for CLI-compatible validation
- `src/trend_analysis/config/bridge.py:#L62-L94` - `validate_payload()` - Validate a payload dict via validate_core_config (Tier 1 only); returns validated dict or raises CoreConfigError

---

## `src/trend_analysis/config/ui_mapping.py` (Streamlit UI → Config)

- `src/trend_analysis/config/ui_mapping.py:#L22-L27` - `coerce_positive_int()` - Silent coercion to int with default and minimum
- `src/trend_analysis/config/ui_mapping.py:#L30-L35` - `coerce_positive_float()` - Silent coercion to float with default
- `src/trend_analysis/config/ui_mapping.py:#L38-L41` - `month_end()` - Snap a timestamp to month-end
- `src/trend_analysis/config/ui_mapping.py:#L44-L133` - `build_sample_split()` - Derive sample_split dict from DatetimeIndex + model state
- `src/trend_analysis/config/ui_mapping.py:#L136-L180` - `build_signals_config()` - Assemble signals config section from trend spec dict
- `src/trend_analysis/config/ui_mapping.py:#L183-L203` - `normalise_metric_weights()` - Normalize metric weight dict to sum to 1.0
- `src/trend_analysis/config/ui_mapping.py:#L206-L300` - `build_portfolio_config()` - Assemble portfolio config section from model state + weights
- `src/trend_analysis/config/ui_mapping.py:#L303-L570` - `build_config_from_ui_state()` - Main entry: Streamlit widget state → complete Config object (~270 lines); no Pydantic validation, relies on inline coercion

---

## `src/trend_analysis/config/patch.py` (NL-driven config mutation)

- `src/trend_analysis/config/patch.py:#L36-L40` - `RiskFlag` - Enum: REMOVES_CONSTRAINT, REMOVES_VALIDATION, INCREASES_LEVERAGE, BROAD_SCOPE
- `src/trend_analysis/config/patch.py:#L43-L79` - `PatchOperation` - Pydantic model for a single mutation: op (set/delete/merge), path (JSON pointer), value
- `src/trend_analysis/config/patch.py:#L82-L120` - `ConfigPatch` - Pydantic model container: operations list, description, risk_flags
- `src/trend_analysis/config/patch.py:#L123-L129` - `risky_patch_flags()` - Detect risk flags on a patch
- `src/trend_analysis/config/patch.py:#L132-L151` - `apply_patch()` - Apply a ConfigPatch to a config dict (in-place)
- `src/trend_analysis/config/patch.py:#L154-L157` - `apply_config_patch()` - Alias for apply_patch
- `src/trend_analysis/config/patch.py:#L160-L176` - `diff_configs()` - Human-readable diff between two config dicts
- `src/trend_analysis/config/patch.py:#L179-L186` - `apply_and_diff()` - Load YAML → apply patch → return (new_config, diff_string)
- `src/trend_analysis/config/patch.py:#L189-L196` - `apply_and_validate()` - Apply patch then run validate_config on result
- `src/trend_analysis/config/patch.py:#L199-L202` - `parse_config_patch()` - Parse LLM response text → ConfigPatch
- `src/trend_analysis/config/patch.py:#L205-L235` - `parse_config_patch_with_retries()` - Retry wrapper around parse_config_patch
- `src/trend_analysis/config/patch.py:#L238-L240` - `_to_dotpath()` - Convert JSON pointer /a/b to dot path a.b
- `src/trend_analysis/config/patch.py:#L243-L249` - `_parse_path_segments()` - Split path string into string/int segments
- `src/trend_analysis/config/patch.py:#L252-L258` - `_strip_code_fence()` - Remove markdown code fences from LLM output
- `src/trend_analysis/config/patch.py:#L261-L272` - `_parse_dotpath()` - Parse dot-path a.b.0.c into segments
- `src/trend_analysis/config/patch.py:#L275-L285` - `_format_dotpath()` - Segments → dot-path string
- `src/trend_analysis/config/patch.py:#L288-L332` - `_resolve_parent()` - Walk config dict to find parent container for a path
- `src/trend_analysis/config/patch.py:#L335-L406` - `_apply_operation()` - Execute a single set/delete/merge on a resolved parent
- `src/trend_analysis/config/patch.py:#L409-L414` - `_deep_merge()` - Recursive dict merge
- `src/trend_analysis/config/patch.py:#L417-L427` - `_has_invalid_json_pointer_escape()` - Detect malformed ~ escapes in JSON pointers
- `src/trend_analysis/config/patch.py:#L430-L431` - `_path_depth()` - Count segments in a path
- `src/trend_analysis/config/patch.py:#L434-L446` - `_detect_risk_flags()` - Scan operations for risk conditions
- `src/trend_analysis/config/patch.py:#L449-L458` - `_removes_constraint()` - Check if op deletes a constraint field
- `src/trend_analysis/config/patch.py:#L461-L472` - `_removes_validation()` - Check if op disables validation
- `src/trend_analysis/config/patch.py:#L475-L483` - `_increases_leverage()` - Check if vol_adjust.target_vol > 0.15
- `src/trend_analysis/config/patch.py:#L486-L489` - `_is_broad_scope()` - Check if op replaces an entire top-level section
- `src/trend_analysis/config/patch.py:#L492-L504` - `_collect_paths()` - Recursively collect all dot-paths in a nested dict
- `src/trend_analysis/config/patch.py:#L507-L512` - `_path_error()` - Format a path-resolution error
- `src/trend_analysis/config/patch.py:#L515-L518` - `format_retry_error()` - Format error message for retry prompt

---

## `src/trend_analysis/config/validation.py` (Full structural validation)

- `src/trend_analysis/config/validation.py:#L23-L28` - `ValidationError` - Pydantic model issue DTO: path, message, expected, actual, suggestion (name collides with pydantic.ValidationError)
- `src/trend_analysis/config/validation.py:#L31-L63` - `ValidationResult` - Pydantic model container: valid (bool), errors (list of ValidationError)
- `src/trend_analysis/config/validation.py:#L66-L113` - `validate_config()` - Orchestrator: runs schema → required → sample-split → portfolio passes; returns ValidationResult
- `src/trend_analysis/config/validation.py:#L116-L121` - `_run_schema_validation()` - JSON Schema (Draft 2020-12) validation pass
- `src/trend_analysis/config/validation.py:#L124-L132` - `_run_required_validation()` - Hand-coded required-field checks (partially redundant with schema)
- `src/trend_analysis/config/validation.py:#L135-L140` - `_run_sample_split_validation()` - Business-logic date ordering pass
- `src/trend_analysis/config/validation.py:#L143-L153` - `_run_portfolio_validation()` - Portfolio selection mode cross-field checks pass
- `src/trend_analysis/config/validation.py:#L156-L166` - `format_validation_messages()` - Format ValidationResult into user-facing strings
- `src/trend_analysis/config/validation.py:#L169-L179` - `_collect_schema_errors()` - Gather JSON Schema errors into ValidationError list
- `src/trend_analysis/config/validation.py:#L182-L216` - `_schema_error_to_issues()` - Convert single jsonschema error → list of issues
- `src/trend_analysis/config/validation.py:#L219-L232` - `_suggest_additional_property()` - Suggest closest valid key for unknown properties
- `src/trend_analysis/config/validation.py:#L235-L259` - `_expected_for_error()` - Extract expected-value hint from schema error
- `src/trend_analysis/config/validation.py:#L262-L283` - `_suggestion_for_error()` - Build actionable suggestion string
- `src/trend_analysis/config/validation.py:#L286-L288` - `_missing_required_field()` - Extract field name from "required" error message
- `src/trend_analysis/config/validation.py:#L291-L293` - `_unexpected_property()` - Extract field name from "additional properties" error message
- `src/trend_analysis/config/validation.py:#L296-L316` - `_check_required_sections()` - Verify top-level sections (data, portfolio, vol_adjust) exist
- `src/trend_analysis/config/validation.py:#L319-L330` - `_check_required_fields()` - Verify required fields within each section
- `src/trend_analysis/config/validation.py:#L333-L363` - `_check_version_field()` - Verify version field is present and valid
- `src/trend_analysis/config/validation.py:#L366-L394` - `_require_field()` - Generic required-field check with path/message assembly
- `src/trend_analysis/config/validation.py:#L397-L402` - `_is_present()` - Check if a value is non-null, non-empty
- `src/trend_analysis/config/validation.py:#L405-L450` - `_check_data_required_fields()` - Required fields for data section
- `src/trend_analysis/config/validation.py:#L453-L487` - `_check_portfolio_required_fields()` - Required fields for portfolio section
- `src/trend_analysis/config/validation.py:#L490-L500` - `_check_vol_adjust_required_fields()` - Required fields for vol_adjust section
- `src/trend_analysis/config/validation.py:#L503-L511` - `_collect_trend_model_errors()` - Collect validate_trend_config errors as ValidationError items
- `src/trend_analysis/config/validation.py:#L514-L572` - `_check_date_ranges()` - Validate date ordering in sample_split
- `src/trend_analysis/config/validation.py:#L575-L625` - `_check_sample_split_requirements()` - Validate sample-split window lengths and date consistency
- `src/trend_analysis/config/validation.py:#L628-L670` - `_check_rank_fund_count()` - Validate selection_count <= available funds
- `src/trend_analysis/config/validation.py:#L673-L701` - `_check_portfolio_selection_requirements()` - Cross-field validation for portfolio selection mode
- `src/trend_analysis/config/validation.py:#L704-L752` - `_check_manual_selection_requirements()` - Validate manual-selection fund lists
- `src/trend_analysis/config/validation.py:#L755-L809` - `_check_rank_inclusion_requirements()` - Validate rank-inclusion mode constraints
- `src/trend_analysis/config/validation.py:#L812-L850` - `_check_rank_value_ranges()` - Validate rank-mode numeric ranges
- `src/trend_analysis/config/validation.py:#L853-L882` - `_count_available_funds()` - Count funds from CSV/glob for capacity checks
- `src/trend_analysis/config/validation.py:#L885-L889` - `_resolve_path()` - Local path resolver for validation context
- `src/trend_analysis/config/validation.py:#L892-L908` - `_error_from_exception()` - Convert exception → ValidationError DTO
- `src/trend_analysis/config/validation.py:#L911-L953` - `_actual_from_path()` - Extract nested value from config by dot-path
- `src/trend_analysis/config/validation.py:#L956-L966` - `_format_path()` - Format path parts into dot-path string
- `src/trend_analysis/config/validation.py:#L969-L972` - `_join_path()` - Join two path segments
- `src/trend_analysis/config/validation.py:#L975-L979` - `_format_issue()` - Format a single ValidationError for display
- `src/trend_analysis/config/validation.py:#L982-L989` - `_format_actual()` - Truncate/format actual values for error messages
- `src/trend_analysis/config/validation.py:#L992-L996` - `_append_issue()` - Deduplicate and append issue to list

---

## `src/trend_analysis/config/schema_validation.py` (JSON Schema runner)

- `src/trend_analysis/config/schema_validation.py:#L17-L21` - `load_schema()` - Load config.schema.json from disk
- `src/trend_analysis/config/schema_validation.py:#L24-L32` - `load_config()` - Load a YAML config file into a dict
- `src/trend_analysis/config/schema_validation.py:#L35-L38` - `_format_error()` - Format a jsonschema error for display
- `src/trend_analysis/config/schema_validation.py:#L41-L46` - `validate_config_data()` - Validate dict against schema, return list of error strings
- `src/trend_analysis/config/schema_validation.py:#L49-L54` - `validate_config_file()` - End-to-end: file path → list of error strings

---

## `src/trend_analysis/config/schema_generator.py` (Schema generation)

- `src/trend_analysis/config/schema_generator.py:#L231-L279` - `_load_model_overrides()` - Load constraint overrides from config_map.yml
- `src/trend_analysis/config/schema_generator.py:#L282-L287` - `collect_config_sources()` - Find all YAML files in config/ directory
- `src/trend_analysis/config/schema_generator.py:#L290-L298` - `load_yaml()` - Safe YAML load with encoding
- `src/trend_analysis/config/schema_generator.py:#L301-L314` - `merge_defaults()` - Recursive merge of two config dicts
- `src/trend_analysis/config/schema_generator.py:#L317-L344` - `gather_samples()` - Merge multiple config files into combined sample map
- `src/trend_analysis/config/schema_generator.py:#L347-L359` - `_find_inline_comment()` - Extract YAML inline comment for descriptions
- `src/trend_analysis/config/schema_generator.py:#L362-L385` - `extract_inline_comments()` - Parse all inline comments from a YAML file
- `src/trend_analysis/config/schema_generator.py:#L388-L393` - `_sanitize_description()` - Clean description strings
- `src/trend_analysis/config/schema_generator.py:#L396-L404` - `_schema_root_description()` - Build root schema description from config map
- `src/trend_analysis/config/schema_generator.py:#L407-L420` - `_infer_type()` - Infer JSON Schema type from Python value
- `src/trend_analysis/config/schema_generator.py:#L423-L436` - `_infer_array_items()` - Infer array item schema
- `src/trend_analysis/config/schema_generator.py:#L439-L444` - `_infer_schema_type()` - Infer type with override support
- `src/trend_analysis/config/schema_generator.py:#L447-L459` - `_description_for()` - Build description for a schema property
- `src/trend_analysis/config/schema_generator.py:#L462-L471` - `_constraints_for()` - Build min/max/enum constraints for a property
- `src/trend_analysis/config/schema_generator.py:#L474-L478` - `_nl_editable()` - Check if a path is NL-patch editable
- `src/trend_analysis/config/schema_generator.py:#L481-L543` - `build_schema()` - Build full JSON Schema from defaults + samples + overrides
- `src/trend_analysis/config/schema_generator.py:#L546-L562` - `_apply_constraints()` - Apply override constraints to a schema property
- `src/trend_analysis/config/schema_generator.py:#L565-L598` - `generate_schema()` - End-to-end: config dir → JSON Schema dict
- `src/trend_analysis/config/schema_generator.py:#L601-L612` - `_compact_schema()` - Strip descriptions/examples for compact output
- `src/trend_analysis/config/schema_generator.py:#L615-L627` - `write_schema_files()` - Write full + compact schema to disk

---

## `src/trend_analysis/config/coverage.py` (Runtime coverage tracking)

- `src/trend_analysis/config/coverage.py:#L21-L34` - `ConfigCoverageReport` - Dataclass report: validated_keys, read_keys, unvalidated_reads
- `src/trend_analysis/config/coverage.py:#L37-L72` - `ConfigCoverageTracker` - Tracks which config keys are validated vs read at runtime
- `src/trend_analysis/config/coverage.py:#L78-L80` - `activate_config_coverage()` - Set global singleton tracker
- `src/trend_analysis/config/coverage.py:#L83-L85` - `deactivate_config_coverage()` - Clear global singleton
- `src/trend_analysis/config/coverage.py:#L88-L89` - `get_config_coverage_tracker()` - Retrieve active tracker (or None)
- `src/trend_analysis/config/coverage.py:#L92-L97` - `compute_schema_validity()` - Compute fraction of read keys that were validated
- `src/trend_analysis/config/coverage.py:#L100-L173` - `_TrackedMapping` - MutableMapping wrapper that records key access for coverage tracking
- `src/trend_analysis/config/coverage.py:#L176-L204` - `wrap_config_for_coverage()` - Wrap a config object with _TrackedMapping for tracking

---

## `src/trend_analysis/config/__init__.py` (Re-exports)

- `src/trend_analysis/config/__init__.py:#L4` - Re-exports `TrendConfig`, `load_trend_config`, `validate_trend_config` from model.py
- `src/trend_analysis/config/__init__.py:#L5-L16` - Re-exports `DEFAULTS`, `ColumnMapping`, `Config`, `ConfigType`, `ConfigurationState`, `PresetConfig`, `list_available_presets`, `load`, `load_config`, `load_preset` from models.py
- `src/trend_analysis/config/__init__.py:#L17-L26` - Re-exports `ConfigPatch`, `PatchOperation`, `RiskFlag`, `apply_and_diff`, `apply_and_validate`, `apply_config_patch`, `apply_patch`, `diff_configs` from patch.py
- `src/trend_analysis/config/__init__.py:#L27-L32` - Re-exports `ValidationError`, `ValidationResult`, `format_validation_messages`, `validate_config` from validation.py
- Notable omissions from `__all__`: `validate_core_config`, `CoreConfig`, `CoreConfigError` (Tier 1) — only importable directly from `trend.config_schema`

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
