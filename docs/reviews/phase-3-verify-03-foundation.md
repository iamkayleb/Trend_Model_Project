# Phase-3 Verification 03: Foundation / Root Standalone Files

**Date:** 2026-07-16
**Reviewer:** Project review (second pass)
**Branch:** `phase-3`
**Scope:** `src/trend_analysis/` root standalone modules (Day 3 subsystem)

---

## Important caveat: Day 3 produced no action items

Day 3 was a **function-reference list** (`FilePath:#L – name – summary`), not a findings-and-recommendations review. It was also cut off before completion. Consequently **there are no Day 3 recommendations to verify as "applied / not applied."**

This document therefore does three things instead:

1. Cross-checks the one *actionable* item that lives in this subsystem but came from **Day 1** (`_json_default` in `walk_forward.py`).
2. Records which foundation files **changed** since the Day 3 snapshot.
3. Gives a short **first-pass review** of the two **new** files (`identity.py`, `regime_utils.py`).

A full first-pass audit of all 24 foundation modules has **not** been performed here — flagged as available follow-up.

---

## Subsystem Summary (change vs Day 3 snapshot)

| File | Lines (phase-3) | Lines (Day 3) | Δ |
|------|-----------------|---------------|---|
| `time_utils.py` | 259 | 226 | +33 |
| `schedules.py` | 230 | 203 | +27 |
| `regimes.py` | 610 | 551 | +59 |
| `data.py` | 622 | 609 | +13 |
| `walk_forward.py` | 415 | 412 | +3 |
| `universe.py` | 307 | 306 | +1 |
| `diagnostics.py` | 177 | 186 | −9 |
| `identity.py` | 146 | — | **new** |
| `regime_utils.py` | 25 | — | **new** |
| constants, _typing, typing, cash_policy, timefreq, universe_catalog, signals, signal_presets, presets, selector, weighting, risk, logging, logging_setup, script_logging, tool_layer | unchanged | = | 0 |

16 of the 24 previously reviewed foundation modules are byte-for-byte unchanged.

---

## Cross-Reference: Day 1 item living in this subsystem

### `_json_default` — 4 copies (Day 1 MEDIUM) — ❌ Not applied

Day 1 flagged `_json_default` duplicated across 4 files, one of which is `walk_forward.py` (a foundation module). On phase-3 it remains duplicated:

- `src/trend_analysis/walk_forward.py`
- `src/trend/cli.py`
- `src/trend_analysis/backtesting/harness.py`
- `src/trend_analysis/core/rank_selection.py`

No shared JSON helper was introduced. Consistent with the finding in verification-01 (CLI).

---

## First-Pass Review: New Files

### `identity.py` (146) — deterministic entity identity resolution

Clean, single-purpose. `EntityId` dataclass + `IdentityMap` with exact and normalized lookups, plus config/universe loaders.

Observations (minor, not blocking):
- `IdentityMap.from_config` reads and parses universe YAML files eagerly (`_entities_from_universe_paths`, `identity.py:119`). File I/O during identity construction is a side effect worth noting; failures are silently skipped (`if not path.exists() ... continue`).
- Unknown labels resolve to a synthetic `EntityId(canonical_id=f"unknown:{raw}", resolved=False)` (`identity.py:77`) — good: no exceptions, and `resolved=False` is surfaceable in manifests.
- No obvious duplication with existing `universe_catalog.py`, though both parse universe YAML `members`; a future consolidation candidate if the two diverge.

### `regime_utils.py` (25) — regime key normalisation/aliasing

Small and clean. One thing to flag for correctness review:

`alias_regime_key` (`regime_utils.py:18`) is a **bidirectional** map:
```python
{"riskon": "calm", "riskoff": "stress", "calm": "riskon", "stress": "riskoff"}
```
This translates between two vocabularies (risk-on/off ↔ calm/stress). That is presumably intentional, but the two-way nature means `alias(alias(x))` round-trips (`riskon → calm → riskon`). Any caller that iteratively resolves aliases could loop or oscillate. Worth confirming callers apply it exactly once.

---

## Verdict for Subsystem C

**No Day 3 action items existed, so nothing to verify as applied.** The one applicable item (Day 1 `_json_default`, present here via `walk_forward.py`) remains **not applied**.

The two new foundation files are clean and well-scoped. Most foundation modules are unchanged; a handful grew (`regimes.py` +59, `time_utils.py` +33, `schedules.py` +27) but were not re-audited in depth.

**Recommendation:** if a genuine review of this subsystem is wanted, schedule a **first-pass audit** (not a verification) of the changed modules — `regimes.py`, `time_utils.py`, `schedules.py`, `data.py` — since those carry the most new logic.

---

## Files Reviewed

- [x] `src/trend_analysis/identity.py` (new)
- [x] `src/trend_analysis/regime_utils.py` (new)
- [x] `src/trend_analysis/walk_forward.py` (`_json_default` cross-check)
- [x] Line-count drift check across all 24 foundation modules
- [ ] Deep first-pass audit of changed modules (`regimes.py`, `time_utils.py`, `schedules.py`, `data.py`) — deferred
