# Day 2 Review: Configuration System (`src/trend_analysis/config/` + `src/trend/config_schema.py`)

**Date:** 2026-03-23
**Reviewer:** Project review
**Scope:** 10 files, 4834 lines — the entire configuration stack from YAML loading through startup validation, UI-to-config translation, NL-driven patching, JSON Schema enforcement, and runtime coverage tracking.

---

## Subsystem Summary

| File | Lines | Purpose | Status |
|------|-------|---------|--------|
| `trend/config_schema.py` | 331 | Tier-1 lightweight stdlib-only validator (CoreConfig, DataSettings, CostSettings) | Active, clean |
| `config/model.py` | 610 | Tier-2 Pydantic startup validator (TrendConfig, DataSettings, PortfolioSettings, RiskSettings) | Active, clean |
| `config/legacy.py` | 111 | Tier-3 raw Config loader — thin wrapper around YAML + `output→export` migration | Active |
| `config/models.py` | 869 | Full Config Pydantic model with Pydantic-or-fallback machinery | Active, complex |
| `config/bridge.py` | 94 | Streamlit↔CLI payload sync — builds and validates lightweight payloads | Active, thin |
| `config/ui_mapping.py` | 570 | Streamlit UI state → Config translation (`build_config_from_ui_state`) | Active, large |
| `config/patch.py` | 533 | NL-driven config mutation: PatchOperation/ConfigPatch + risk-flag detection | Active, clean |
| `config/validation.py` | 996 | Full structural validation (JSON Schema + business-logic checks) | Active, large |
| `config/schema_validation.py` | 57 | Load `config.schema.json` and run Draft202012Validator | Active, thin |
| `config/schema_generator.py` | 634 | Generate `config.schema.json` / compact schema from `defaults.yml` | Active |
| `config/coverage.py` | 204 | Runtime key-access tracker (read vs validated alignment) | Active, instrumentation |
| `config/__init__.py` | 62 | Re-exports all public symbols from above modules | Active, clean |

---

## Architecture: Three-Tier Validation

The config system has a deliberate three-tier design documented in the tier-1 docstring:

```
Tier 1: trend/config_schema.py       (stdlib-only, fast startup, CoreConfig)
           ↓ used by CLI --check and Streamlit bridge
Tier 2: config/model.py              (Pydantic, TrendConfig with DataSettings/PortfolioSettings/RiskSettings)
           ↓ used by CLI run and load_trend_config()
Tier 3: config/legacy.py + models.py (full Config, used by the pipeline engine)
           ↓ used by pipeline.py and the Streamlit app
```

This is intentional: Tier 1 starts in milliseconds with no Pydantic import. Tier 2 adds full field-level validation before the pipeline starts. Tier 3 is the complete config object consumed downstream.

---

## Duplicate Analysis

### `DataSettings` — name collision between Tier 1 and Tier 2

Two unrelated classes share the same name in different modules:

| Class | Module | Fields | Backend |
|-------|--------|--------|---------|
| `DataSettings` | `trend.config_schema` | `csv_path`, `managers_glob`, `date_column`, `frequency`, `universe_membership_path` (5) | stdlib dataclass |
| `DataSettings` | `trend_analysis.config.model` | All 5 above + `missing_policy`, `missing_limit`, `risk_free_column`, `allow_risk_free_fallback` (9) | Pydantic BaseModel |

**Risk:** Any import statement `from trend_analysis.config import DataSettings` gets the Tier-2 class; `from trend.config_schema import DataSettings` gets the Tier-1 class. There is no overlap in the inheritance chain. If a caller passes a Tier-1 `DataSettings` where a Tier-2 is expected (or vice versa), runtime duck-typing may succeed silently with missing fields.

**Recommendation:** Rename the Tier-1 class to `CoreDataSettings` (matching the `CoreConfig` pattern already used in that module) to prevent silent confusion.

### `ValidationError` — name collision with Pydantic

`trend_analysis.config.validation` defines its own `ValidationError(BaseModel)` — a Pydantic model used as a data container. This shadows the standard `pydantic.ValidationError` exception class in any scope that imports both. The `validation.py` module itself imports `pydantic.BaseModel` and uses it for the `ValidationResult` model but never directly raises `pydantic.ValidationError` — however any test or caller that does `from trend_analysis.config.validation import ValidationError` and then tries to catch a pydantic validation error by name will silently catch the wrong type.

**Recommendation:** Rename the internal DTO to `ConfigIssue` or `ValidationIssue` to free up `ValidationError` for the standard exception class.

### `validate_config` — two functions, same name

| Function | Module | Does |
|----------|--------|------|
| `validate_config` | `trend_analysis.config.validation` | Full structural check (JSON Schema + business logic), returns `ValidationResult` |
| `validate_trend_config` | `trend_analysis.config.model` | Pydantic model validation, raises `ValueError` |
| `validate_core_config` | `trend.config_schema` | Lightweight stdlib validation, returns `CoreConfig` |

These have different signatures and return types, so are not true duplicates, but callers must import from the right module. The `__init__.py` re-exports only `validate_config` (the full structural one), which could mislead callers expecting startup-safe validation.

---

## Key Findings

### `config/models.py` — Pydantic-or-fallback machinery is fragile

`models.py` (869 lines) has the most complex code in the subsystem. It:
- Guards all Pydantic imports with `try/except ImportError`
- Caches the class identity in `builtins._TREND_CONFIG_CLASS` to prevent re-definition on module reimport
- Builds `REQUIRED_DICT_FIELDS` dynamically by introspecting type annotations via `get_origin`/`get_args`
- Maintains a `SimpleBaseModel` fallback path that bypasses all validation

The builtins-caching pattern (`setattr(_bi, "_TREND_CONFIG_CLASS", ...)`) is unusual and fragile — it leaks state into the global interpreter, which can cause test-order-dependent failures if the class is rebuilt differently in different test sessions.

**Recommendation:** If Pydantic is a hard runtime dependency (it is listed in `pyproject.toml`), remove the fallback path entirely. If the fallback is needed for lightweight environments, isolate it into a separate `config_fallback.py` module rather than embedding it in the same file.

### `config/bridge.py` — validates only Tier 1 fields

`bridge.py::validate_payload` calls `validate_core_config` (Tier 1), which only validates `data` and `portfolio.cost_model`. The `vol_adjust`, `portfolio.rebalance_calendar`, `portfolio.max_turnover`, and `portfolio.selection_mode` fields are never validated at this layer. The Streamlit app accepts and passes through these unchecked fields silently.

**Recommendation:** Consider calling `validate_trend_config` (Tier 2) from `validate_payload` instead, since Tier 2 covers `vol_adjust` and `portfolio` minimums. The bridge currently returns a partial `validated` dict that is a mix of Tier-1-checked and unchecked fields.

### `config/ui_mapping.py::build_config_from_ui_state` — 270-line function

The main builder function is large but straightforward: it reads from `model_state`, coerces types, and assembles the full `Config` struct. The function is easy to trace but has no intermediate validation — if `model_state` contains inconsistent values (e.g. `vol_target = -1.0`), `coerce_positive_float` silently clamps it to `0.0` rather than raising. The negative-value silencing differs from the Pydantic validators in `model.py` which raise `ValueError`.

**Recommendation:** Document the silent-coercion behavior explicitly in the function docstring so callers know the contract.

### `config/patch.py` — risk-flag thresholds are hardcoded

`_VOL_TARGET_RISK_THRESHOLD = 0.15` on line 31 of `patch.py` is the only hardcoded numeric threshold in the NL patch system. `_increases_leverage()` raises `INCREASES_LEVERAGE` if `vol_adjust.target_vol > 0.15`. This threshold is not configurable and not documented in the schema.

**Recommendation:** Either document this as a deliberate product decision (and add a comment explaining why 15% is the threshold) or make it configurable via the config schema.

### `config/coverage.py` — instrumentation is opt-in and rarely activated

`ConfigCoverageTracker` is only activated when callers call `activate_config_coverage()` (a global singleton pattern). The only activation seen in Tier-1 is `config_schema.py::validate_core_config` which calls `get_config_coverage_tracker()` and tracks 5 keys — but only if a tracker has been activated. In practice the tracker is never activated in production paths; its activation appears limited to tests.

This is not a bug, but the pattern adds 50 lines of coverage tracking scattered across production code for a purely test/diagnostic feature.

**Recommendation:** Move coverage instrumentation into a test-only wrapper or use a context manager to make activation/deactivation explicit.

---

## Structural Observation: `validation.py` runs three independent passes

`validate_config()` runs in sequence:
1. `_run_schema_validation` — JSON Schema (Draft202012) against `config.schema.json`
2. `_run_required_validation` — hand-coded required-field checks (partially redundant with schema)
3. `_run_sample_split_validation` — business-logic date ordering
4. `_run_portfolio_validation` — portfolio selection mode cross-field checks

The hand-coded required-field checks in pass 2 duplicate what the JSON Schema in pass 1 already enforces. This means some fields produce duplicate error messages — both a schema `"required"` error and a `"Required field is missing"` error from `_check_required_fields`.

The `skip_required_fields=True` flag exists to suppress pass 2 but not pass 1, so callers using it still get JSON Schema required errors.

**Recommendation:** Audit for fields that are both in the JSON Schema `required` arrays and in `_check_required_fields`/`_check_required_sections`. Remove the hand-coded duplicates and rely on the schema exclusively for required-field checking.

---

## Action Items (prioritized)

1. **HIGH** — Rename `trend.config_schema.DataSettings` → `CoreDataSettings` to prevent class-name collision with `trend_analysis.config.model.DataSettings`
2. **HIGH** — Rename `trend_analysis.config.validation.ValidationError` → `ConfigIssue` to prevent collision with `pydantic.ValidationError`
3. **MEDIUM** — Audit `validation.py` for required-field checks duplicated by the JSON Schema; remove hand-coded duplicates
4. **MEDIUM** — Remove the `builtins._TREND_CONFIG_CLASS` caching pattern in `models.py`; simplify the Pydantic-or-fallback approach
5. **MEDIUM** — Expand `bridge.py::validate_payload` to include Tier-2 (TrendConfig) validation
6. **LOW** — Document the `_VOL_TARGET_RISK_THRESHOLD = 0.15` threshold in `patch.py`
7. **LOW** — Document silent-coercion behavior in `ui_mapping.py::build_config_from_ui_state`
8. **LOW** — Move `ConfigCoverageTracker` activation out of production code paths

---

## Files Reviewed

- [x] `src/trend/config_schema.py` (331 lines)
- [x] `src/trend_analysis/config/__init__.py` (62 lines)
- [x] `src/trend_analysis/config/model.py` (610 lines)
- [x] `src/trend_analysis/config/models.py` (869 lines)
- [x] `src/trend_analysis/config/legacy.py` (111 lines)
- [x] `src/trend_analysis/config/bridge.py` (94 lines)
- [x] `src/trend_analysis/config/ui_mapping.py` (570 lines)
- [x] `src/trend_analysis/config/patch.py` (533 lines)
- [x] `src/trend_analysis/config/validation.py` (996 lines)
- [x] `src/trend_analysis/config/schema_validation.py` (57 lines)
- [x] `src/trend_analysis/config/schema_generator.py` (634 lines, top reviewed)
- [x] `src/trend_analysis/config/coverage.py` (204 lines)
