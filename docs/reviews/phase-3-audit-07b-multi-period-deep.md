# Phase-3 Deep Correctness Pass 07b: Multi-Period Engine

**Date:** 2026-07-16
**Reviewer:** Project review (deep correctness pass)
**Branch:** `phase-3`
**Scope:** `multi_period/engine.py` — the `_run_threshold_hold_multi_periods` loop (L1936–L4375) and its top-level helpers (`_setup_period`, `_apply_turnover_and_cost`, `_assemble_period_result`), plus `scheduler.generate_periods`.

> Follow-up to audit-07, which deferred the correctness audit of the 2,440-line loop. This
> pass verifies the **backtest-correctness invariants** (look-ahead, return realisation,
> cost/turnover carry-forward). It does **not** exhaustively verify every branch of the
> hold-state machine (see "Not fully verified").

---

## Headline: no look-ahead bias found

The central risk in any backtest engine is **look-ahead** — using out-of-sample (future) data to make in-sample decisions. Traced end-to-end, the engine is sound on this:

1. **Window construction is strictly non-overlapping.** `scheduler.generate_periods` sets `in_end_period = out_start_period − 1` (`scheduler.py:96`) and `out_start_period = in_end_period + 1` (`:102`) — in-sample ends exactly one month before out-of-sample begins. Windows are adjacent, never overlapping.
2. **Selection/scoring uses only in-sample data.** `_score_frame` and the metric computations operate on `in_df` (the in-sample slice); selection, z-scores, and ranking are all derived from it.
3. **Returns are realised only on out-of-sample data.** Per-period returns come from `out_df`/`out_scaled` (the out-of-sample slice) — e.g. `rebalance_returns = (out_scaled * weights_by_date).sum(axis=1)` (`engine.py:4330`).
4. **The window slicer itself** (`stages/preprocessing._build_sample_windows`, reviewed in audit-05) resolves both bounds through the shared `resolve_period_bound` with inclusive masks, so the schedule's non-overlap is preserved through to slicing.

I found no path where out-of-sample returns feed back into selection or weighting for the same period.

---

## Architecture: the engine reuses the single-period pipeline

The per-period loop does **not** reimplement return/stat computation. It builds the universe, selection, and target weights, then calls the single-period pipeline via `_call_pipeline_with_diag → _run_analysis` (`engine.py:4063`), which runs `stages/portfolio._compute_weights_and_stats` (reviewed in audit-05b/06). Out-of-sample stats (`out_ew_stats`, `out_user_stats`) come from there.

**Implication:** the correctness of per-period return math reduces to code already reviewed. The multi-period-specific responsibilities are (a) period scheduling [verified], (b) the hold-state machine, and (c) turnover/cost carry-forward across periods.

---

## Verified correct

### Turnover/cost carry-forward (`_apply_turnover_and_cost`, L473-598)
Careful and correct:
- Turnover penalty applied as a convex combination toward previous weights (`:508`).
- **Forced exits are protected** from turnover-penalty dilution and from the turnover cap (`:514-518`, `:548-556`) — below-threshold holdings can't linger just because turnover is capped.
- Turnover-cap allocation splits trades into **mandatory** (forced exits + mandatory min-funds entries) and **optional**, executing mandatory first and scaling optional by the remaining cap (`:562-570`). Correct prioritisation.
- **Conditional renormalisation** (`:582-589`): weights are only rescaled to sum to 1 when they already sum to ≈1, deliberately preserving infeasible-bound outcomes (min-weight floors too large → sum > 1; max-weight caps too tight → sum < 1). This is a thoughtful correctness detail, not an oversight.

### Intra-period rebalance stat recomputation (`engine.py:4326-4352`)
When weights vary within the out-of-sample window (`weights_by_date`), the engine recomputes `out_user_stats` on the **actual realised** date-varying path rather than the pipeline's static-weight figure, correctly adding cash returns (`rf_out * cash_weight_series`) and using out-of-sample cost-adjusted returns. Sound.

### State carry-forward (`_assemble_period_result`, L601-644)
Tenure is incremented per surviving holding, `prev_final_weights` is filtered to non-zero and carried to the next period for turnover computation. Consistent.

---

## Cross-references to earlier findings

- **05b-② confirmed to reach this path.** The engine uses the same `_compute_stats` (`engine.py:4337,4350`) that hardcodes `periods_per_year` via metric defaults. Consistent with the engine's documented month-end cadence convention (`engine.py:1956`), so low practical risk — but it is the same latent trap if the monthly assumption is relaxed.
- **audit-07 findings stand:** the 2,440-line function size (High, maintainability) and the turnover/max-active/bounds helpers duplicated between `engine.py` and `risk.py`/`optimizer.py` (Medium). This deep pass did not change those verdicts.

---

## Not fully verified (honest scope)

The **hold-state machine** — implemented as dozens of nested closures inside the loop — was mapped and spot-checked but **not exhaustively verified branch-by-branch**:

- soft/hard z-entry and z-exit thresholds (`_hard_exit_forced`, `_filter_entry_candidates`)
- sticky add/drop counters, `min_tenure` protection, cooldown reseed skips
- one-per-firm dedup (`_dedupe_one_per_firm`) and buy-and-hold / random selection modes
- turnover-budget max-changes gating (`engine.py:3734+`)

These encode the bulk of the strategy semantics and interact in ways that are hard to reason about statically in a 1,600-line loop body. A fixture-driven test suite (small synthetic universe, asserting entries/exits/tenure across a few periods) is the right tool — not static reading.

---

## Verdict

On the invariants that matter most for a backtest engine — **no look-ahead, correct out-of-sample return realisation, and correct turnover/cost carry-forward** — the multi-period engine is **sound**. It also earns credit for **reusing the single-period pipeline** instead of duplicating return math.

The unchanged concerns are the ones from audit-07: the function is far too large to be confidently maintained or fully verified by reading, and it re-implements constraint helpers that live elsewhere. The right follow-up is **decomposition + fixture tests for the hold-state machine**, not further static reading.

---

## Files Reviewed

- [x] `multi_period/scheduler.py::generate_periods` (window construction — look-ahead check)
- [x] `multi_period/engine.py::_apply_turnover_and_cost` (L473-598, full)
- [x] `multi_period/engine.py::_assemble_period_result` (L601-644, full)
- [x] `multi_period/engine.py` loop structure + return realisation (`:4063`, `:4326-4352`)
- [~] `_run_threshold_hold_multi_periods` hold-state machine (mapped; **not** branch-verified)
