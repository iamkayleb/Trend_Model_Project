# Phase-3 Verification 02: Configuration System

**Date:** 2026-07-16
**Reviewer:** Project review (second pass)
**Branch:** `phase-3`
**Purpose:** Verify whether the Day 2 (configuration system) action items were correctly applied on the `phase-3` branch.
**Scope:** `src/trend/config_schema.py` + `src/trend_analysis/config/`

---

## Subsystem Summary

| File | Lines | Purpose | Change vs Day 2 |
|------|-------|---------|-----------------|
| `trend/config_schema.py` | 340 | Tier-1 stdlib validator | +9 lines |
| `config/model.py` | 684 | Tier-2 Pydantic startup validator | +74 lines |
| `config/models.py` | 901 | Full Config (Pydantic-or-fallback) | +32 lines |
| `config/legacy.py` | 111 | Tier-3 raw Config loader | Unchanged |
| `config/bridge.py` | 141 | Streamlit ↔ CLI payload sync | **+47 lines (rewritten)** |
| `config/ui_mapping.py` | 570 | Streamlit UI → Config | Unchanged |
| `config/patch.py` | 533 | NL-driven config mutation | Unchanged |
| `config/validation.py` | 1218 | Full structural validation | +222 lines |
| `config/schema_validation.py` | 57 | JSON Schema runner | Unchanged |
| `config/schema_generator.py` | 639 | Schema generation | +5 lines |
| `config/coverage.py` | 205 | Runtime coverage tracking | +1 line |
| `config/lint_keys.py` | 246 | **New** — config key linting | New file |

---

## Verification Results

| # | Original Suggestion (Day 2) | Priority | Verdict |
|---|------------------------------|----------|---------|
| 1 | Rename `config_schema.DataSettings` → `CoreDataSettings` | HIGH | ❌ Not applied |
| 2 | Rename `validation.ValidationError` → `ConfigIssue` | HIGH | ❌ Not applied |
| 3 | Remove required-field checks duplicated by JSON Schema | MEDIUM | ❌ Not applied |
| 4 | Remove `builtins._TREND_CONFIG_CLASS` caching in `models.py` | MEDIUM | ❌ Not applied |
| 5 | Expand `bridge.validate_payload` to Tier-2 validation | MEDIUM | ✅ Applied |
| 6 | Document `_VOL_TARGET_RISK_THRESHOLD = 0.15` in `patch.py` | LOW | ❌ Not applied |
| 7 | Document silent-coercion in `build_config_from_ui_state` | LOW | ❌ Not applied |
| 8 | Move `ConfigCoverageTracker` activation out of production | LOW | ⚠️ Partial |
| — | Migrate `ui_mapping`/`analysis_runner` off `legacy.Config` | (extra) | ❌ Not applied |

---

## Detail

### ① Rename `DataSettings` → `CoreDataSettings` — ❌ Not applied

`src/trend/config_schema.py:59` still declares `class DataSettings:`. No `CoreDataSettings` symbol exists anywhere in the tree. The name still collides with `trend_analysis.config.model.DataSettings` (the 9-field Pydantic model). The HIGH-priority collision is unresolved.

### ② Rename `ValidationError` → `ConfigIssue` — ❌ Not applied

`src/trend_analysis/config/validation.py:24` still declares `class ValidationError(BaseModel):`. It continues to shadow `pydantic.ValidationError` in any scope importing both. No `ConfigIssue` symbol exists.

### ③ Remove duplicated required-field checks — ❌ Not applied

`validate_config` still runs both passes:
- `_run_schema_validation` (`validation.py:118`) — JSON Schema
- `_run_required_validation` (`validation.py:126`) → `_check_required_sections` (`:308`), `_check_required_fields` (`:331`)

The hand-coded required-field checks still coexist with the JSON Schema `required` enforcement, so the same missing field can still produce duplicate messages. The file grew by ~222 lines, but the redundant validation structure is unchanged.

### ④ Remove `builtins._TREND_CONFIG_CLASS` caching — ❌ Not applied

The builtins-caching pattern is intact in `config/models.py`:
- `import builtins as _bi` (`:182`)
- `_cached = getattr(_bi, "_TREND_CONFIG_CLASS", None)` (`:184`)
- `setattr(_bi, "_TREND_CONFIG_CLASS", _runtime_base)` (`:187`)
- `setattr(_bi, "_TREND_CONFIG_CLASS", _PydanticConfigImpl)` (`:405`)

The Pydantic-or-fallback machinery and interpreter-global class caching remain.

### ⑤ Expand `bridge.validate_payload` to Tier-2 — ✅ Applied

This is the one action item solidly implemented, and correctly. `bridge.py` was rewritten:

- Now imports `validate_trend_config` (`bridge.py:9`) and `validate_config` + `format_validation_messages` (`:10`).
- `validate_payload` now calls **both** `validate_core_config` (Tier 1) **and** `validate_trend_config` (Tier 2) (`:78-79`).
- It additionally runs a semantic `validate_config` pass (`:107`) and surfaces its errors (`:112-113`).
- The returned `validated` dict now carries Tier-2-derived values: `portfolio.max_turnover` (`:130`) and `vol_adjust.target_vol` (`:139`) — exactly the fields flagged as previously unchecked.

This directly addresses the Day 2 gap ("vol_adjust, portfolio.rebalance_calendar, portfolio.max_turnover never validated at this layer").

### ⑥ Document `_VOL_TARGET_RISK_THRESHOLD` — ❌ Not applied

`patch.py:31` is still a bare constant with no comment:

```python
_VOL_TARGET_RISK_THRESHOLD = 0.15
```

No rationale for the 15% threshold; still not configurable. Used at `:479` and `:482`.

### ⑦ Document silent-coercion in `build_config_from_ui_state` — ❌ Not applied

`ui_mapping.py:303` still has **no docstring** — the function body starts immediately at line 311. The silent-clamping behavior of `coerce_positive_float`/`coerce_positive_int` remains undocumented.

### ⑧ Move `ConfigCoverageTracker` activation out of production — ⚠️ Partial

Activation is **still in production code** (`src/trend/cli.py:45-49`, `src/trend_analysis/cli.py`), so the literal recommendation ("move out of production code paths") was not followed.

However, the underlying concern — implicit/always-on activation — is mitigated: activation is now gated behind an explicit, opt-in CLI flag `--config-coverage` (`trend/cli.py:332, 374, 392`). It only runs when the user asks for it. Spirit addressed, letter not.

### ⑨ (Extra) Migrate off `legacy.Config` — ❌ Not applied

`legacy.py` is unchanged (111 lines) and both production consumers still import it:
- `src/trend_analysis/config/ui_mapping.py:9` → `from trend_analysis.config.legacy import Config`
- `streamlit_app/components/analysis_runner.py:15` → `from trend_analysis.config.legacy import Config`

Plus scripts/tests/docs. No migration to `models.Config` occurred.

---

## Verdict for Subsystem B

**1 applied, 1 partial, 7 not applied.**

The bridge Tier-2 validation (#5) was implemented well — it does more than suggested (adds a full semantic `validate_config` pass) and closes the exact gap identified. Coverage activation (#8) is reasonably handled via an opt-in flag even though it stayed in production code.

Everything else is untouched, including both HIGH-priority naming collisions (`DataSettings`, `ValidationError`) that were the top items from Day 2. `validation.py` grew substantially without addressing the duplicate required-field checks, and the fragile `builtins` caching in `models.py` remains.

Note: a new `config/lint_keys.py` (246 lines) appeared — unrelated to the Day 2 items; not evaluated here.

---

## Files Reviewed

- [x] `src/trend/config_schema.py`
- [x] `src/trend_analysis/config/model.py`
- [x] `src/trend_analysis/config/models.py`
- [x] `src/trend_analysis/config/legacy.py`
- [x] `src/trend_analysis/config/bridge.py`
- [x] `src/trend_analysis/config/ui_mapping.py`
- [x] `src/trend_analysis/config/patch.py`
- [x] `src/trend_analysis/config/validation.py`
- [x] `src/trend_analysis/config/coverage.py`
- [x] `src/trend_analysis/config/schema_validation.py` (unchanged)
- [x] `src/trend_analysis/config/schema_generator.py` (unchanged in substance)
- [ ] `src/trend_analysis/config/lint_keys.py` (new; out of scope for Day 2 verification)
