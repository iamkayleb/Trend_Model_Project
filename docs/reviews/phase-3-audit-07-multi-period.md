# Phase-3 First-Pass Audit 07: Multi-Period Engine

**Date:** 2026-07-16
**Reviewer:** Project review (first pass)
**Branch:** `phase-3`
**Scope:** `multi_period/{engine,scheduler,loaders,replacer}.py`

> **Coverage honesty:** `engine.py` is 4,398 lines and is dominated by a single
> 2,440-line function. This is a **structural review**, not a line-by-line correctness
> audit of the threshold-hold loop. Correctness of that loop is **not verified** here.

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `multi_period/engine.py` | 4398 | Multi-period scheduling, weighting, turnover/cost, threshold-hold path |
| `multi_period/loaders.py` | 227 | `load_prices` / `load_membership` / `load_benchmarks` |
| `multi_period/replacer.py` | 227 | `Rebalancer` |
| `multi_period/scheduler.py` | 140 | `generate_periods` |

Largest functions in `engine.py`:

| Lines | Function | Span |
|------:|----------|------|
| **2440** | `_run_threshold_hold_multi_periods` | L1936–L4375 |
| 343 | `run` | L1591–L1933 |
| 210 | `run_schedule` | L1049–L1258 |
| 182 | `_run_phase1_multi_periods` | L1261–L1442 |
| 126 | `_apply_turnover_and_cost` | L473–L598 |

---

## Summary of Findings

| # | Finding | Severity |
|---|---------|----------|
| 1 | `_run_threshold_hold_multi_periods` is a single 2,440-line function | High (maintainability) |
| 2 | Constraint/turnover helpers reimplemented here, parallel to `risk.py`/`optimizer.py` | Medium |
| 3 | `_accepts_keyword` duplicated in 3 modules | Low |
| — | Turnover-based cost model (confirms 05b-③ is a deliberate single-period simplification) | ✅ Cross-ref |
| — | `periods_per_year = 12` hardcoded **with rationale** (month-end cadence) | ✅ Context for 05b-② |

---

## Findings

### ① `_run_threshold_hold_multi_periods` — 2,440-line function — High (maintainability)

`engine.py:1936-4375` is one function spanning more than half the file (55%). It takes 11 parameters and implements the entire threshold-hold multi-period path inline (period generation, universe build, scoring, Bayesian weighting, turnover/cost, regime overrides, result assembly).

**Impact:** this is effectively un-unit-testable as a unit, very hard to review for correctness, and a high-risk locus for regressions. Any bug in the threshold-hold path lives somewhere in these 2,440 lines with no smaller seams to isolate it.

**Recommendation:** decompose into named stages mirroring the single-period pipeline (`stages/`): period setup, per-period selection, per-period weighting, turnover/cost application, result assembly. Several module-level helpers already exist (`_setup_period`, `_weight_period`, `_apply_turnover_and_cost`, `_assemble_period_result`) — the monster function should be delegating to those rather than re-inlining logic. This is the single highest-value refactor in the subsystem.

### ② Constraint/turnover logic reimplemented — Medium

The engine carries its own constraint/turnover helpers that parallel the single-period risk stack:

| Concept | Multi-period (engine.py) | Single-period |
|---------|--------------------------|---------------|
| Turnover penalty | `_apply_turnover_penalty` (`:966`) | `risk.py::_apply_turnover_penalty` (`:145`) |
| Max active positions | `_enforce_max_active_positions` (`:919`) | `risk.py::_enforce_max_active` (`:193`) |
| Weight bounds | `_apply_weight_bounds` (`:849`) | `optimizer.apply_constraints` |

The turnover-penalty **formula** matches (`prev + (target − prev)·(1 − λ)`), but the surrounding behaviour diverges: the engine re-applies min/max weight bounds (`:981-984`), while `risk.py` renormalises and special-cases `λ ≥ 1`. So single-period and multi-period damp turnover **slightly differently** for the same `lambda_tc`.

**Impact:** the same config can produce subtly different turnover behaviour depending on which engine runs. That is a latent inconsistency, and two copies drift independently.

**Recommendation:** extract the shared turnover/bounds/max-active primitives into one module (e.g. `risk.py` or a `weights/constraints.py`) and have both engines call it, parameterising the bounds-vs-renormalise difference explicitly if it is intended.

### ③ `_accepts_keyword` ×3 — Low

Identical-purpose helper in `multi_period/engine.py:4391`, `multi_period/loaders.py:26`, and `pipeline_entrypoints.py:35`. Trivial, but a clean shared-util candidate.

---

## Cross-References (context for earlier findings)

### Turnover-based cost model — confirms 05b-③
The multi-period engine applies **turnover-based** transaction costs (`_period_turnover_cost` `:358`, `_apply_turnover_and_cost` `:473`, `_TurnoverCostApplication` `:426`). This confirms the audit-05b/06 note: the single-period pipeline's flat `monthly_cost` per-asset drag is a deliberate simplification, with the realistic turnover-driven cost accounting living here.

### `periods_per_year = 12` — context for 05b-②
`engine.py:1956` hardcodes `periods_per_year = 12` with an explicit comment: *"Multi-period engine normalizes data to month-end cadence. Keep metric annualisation consistent with monthly returns."* This documents a **system-wide convention that all analysis runs on monthly data**. It lowers the practical risk of 05b finding ② (hardcoded 12 in `_compute_stats`) — the whole system assumes monthly cadence — though the *internal inconsistency* in `_compute_weights_and_stats` (threading `window.periods_per_year` into risk calcs while hardcoding 12 in stats) still stands as a latent trap if the monthly assumption is ever relaxed.

---

## Verdict for Subsystem G

**No correctness verdict is offered on the threshold-hold path** — it is a 2,440-line function that was not line-audited, and I will not claim coverage I do not have. Structurally, the dominant issue is that function's size (①); it should be decomposed into the same kind of named stages the single-period pipeline already uses. The constraint/turnover reimplementation (②) is a real single-vs-multi divergence risk worth consolidating.

`scheduler.py`, `loaders.py`, `replacer.py` are appropriately small and single-purpose; reviewed at scan level with no red flags.

---

## Files Reviewed

- [x] `src/trend_analysis/multi_period/engine.py` — structural map; `_apply_turnover_penalty`, cost helpers, `_run_threshold_hold_multi_periods` head (NOT full line-audit of the 2,440-line body)
- [x] `src/trend_analysis/multi_period/scheduler.py` (inventory)
- [x] `src/trend_analysis/multi_period/loaders.py` (inventory)
- [x] `src/trend_analysis/multi_period/replacer.py` (inventory)
- [ ] Full correctness audit of `_run_threshold_hold_multi_periods` — **deferred** (requires decomposition first, or a dedicated fixture-driven pass)
