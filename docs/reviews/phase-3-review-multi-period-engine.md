# Phase-3 Review: Multi-Period Engine

**Date:** 2026-07-16
**Reviewer:** Project review
**Branch:** `phase-3`
**Scope:** `src/trend_analysis/multi_period/` — `engine.py`, `scheduler.py`, `loaders.py`, `replacer.py`

*This consolidates and supersedes audit-07 (structure) and audit-07b (correctness). Read this one.*

---

## The short version

The engine is **correct where it counts and hard to maintain where it hurts**.

I went looking for the thing that would actually invalidate results — look-ahead bias — and didn't find it. Window construction, selection, and return realisation are all cleanly separated in time. The turnover and cost logic is genuinely well thought through; a couple of details in there are sharper than I expected.

The problem is structural. One function is 2,440 lines long. That's 55% of the file in a single unit that can't be tested in pieces, can't be reviewed with confidence, and quietly re-implements logic that already exists elsewhere in the codebase. Nothing is broken today. But this is where a future bug will hide, and it will be expensive to find.

---

## What's in here

| File | Lines | What it does |
|------|-------|--------------|
| `engine.py` | 4,398 | Scheduling, universe/selection, weighting, turnover & cost, result assembly |
| `loaders.py` | 227 | `load_prices` / `load_membership` / `load_benchmarks` |
| `replacer.py` | 227 | `Rebalancer` |
| `scheduler.py` | 140 | `generate_periods` |

And the size distribution inside `engine.py`, which is really the story of this review:

| Lines | Function | Span |
|------:|----------|------|
| **2,440** | `_run_threshold_hold_multi_periods` | L1936–L4375 |
| 343 | `run` | L1591–L1933 |
| 210 | `run_schedule` | L1049–L1258 |
| 182 | `_run_phase1_multi_periods` | L1261–L1442 |
| 126 | `_apply_turnover_and_cost` | L473–L598 |
| 112 | `_build_rebalance_frame` | L1445–L1556 |

---

## No look-ahead bias

This was the main thing I wanted to establish, because everything else is cosmetic by comparison. If a backtest engine lets future data leak into past decisions, every number it produces is fiction.

It doesn't. Four checks, all clean:

**The schedule can't overlap.** In `generate_periods`, in-sample ends exactly one period before out-of-sample starts — `in_end_period = out_start_period - 1` (`scheduler.py:96`) and `out_start_period = in_end_period + 1` (`:102`). The windows are adjacent by construction, so there's no arithmetic path that produces an overlap.

**Selection only ever sees in-sample data.** Scoring, z-scores, and ranking all read from `in_df`. I traced the score-frame construction and the selector wiring; nothing reaches into the out-of-sample slice.

**Returns are realised only on out-of-sample data.** Per-period performance comes from `out_df` / `out_scaled` — e.g. `rebalance_returns = (out_scaled * weights_by_date).sum(axis=1)` (`engine.py:4330`).

**The slicer preserves the separation.** `stages/preprocessing._build_sample_windows` resolves both boundaries through the shared `resolve_period_bound` helper with inclusive masks, so the non-overlap the scheduler guarantees survives all the way into the actual data slices.

I couldn't find a route where out-of-sample returns influence same-period selection or weighting.

---

## It reuses the single-period pipeline (good)

Worth calling out, because it's the kind of decision that quietly prevents a whole class of bugs.

The per-period loop doesn't compute returns or statistics itself. It builds the universe, runs selection, produces target weights — then hands off to the single-period pipeline via `_call_pipeline_with_diag → _run_analysis` (`engine.py:4063`), which is the same `_compute_weights_and_stats` reviewed earlier.

Practically, that means per-period return math is *already reviewed code*, and single-period and multi-period runs can't drift apart in how they compute performance. The engine's genuinely unique responsibilities are narrower than the file size suggests: scheduling, the hold-state machine, and carrying turnover/cost state across periods.

---

## The turnover and cost logic is good

`_apply_turnover_and_cost` (L473–598) is the most carefully written part of the engine. Four things it gets right:

**Forced exits don't get diluted.** When a holding trips a z-exit threshold, it's targeted to zero *after* the turnover penalty is applied (`:514-518`), so the penalty can't partially undo a decision the strategy already made.

**Forced exits also survive the turnover cap.** Trades are split into mandatory (forced exits, required min-fund entries) and optional. Mandatory executes first; optional gets scaled into whatever cap remains (`:548-570`). Without this, a below-threshold holding could linger indefinitely just because the book was busy — a subtle and genuinely nasty failure mode that someone clearly thought about.

**Renormalisation is conditional, and deliberately so.** Weights are only rescaled to sum to 1 when they *already* sum to about 1 (`:582-589`). If min-weight floors push the total above 1, or max-weight caps hold it below, that infeasibility is preserved rather than papered over. The comment says as much. This is the sort of thing that looks like a bug until you read why it isn't.

**State carry-forward is consistent.** `_assemble_period_result` (L601–644) increments tenure for surviving holdings and passes non-zero final weights forward as the baseline for next period's turnover calculation.

One more: when weights change *within* an out-of-sample window, the engine recomputes `out_user_stats` on the actual date-varying path rather than reporting the static-weight approximation (`:4326-4352`), including cash returns. That's the honest number, and it would have been easy to skip.

---

## Findings

### 1. `_run_threshold_hold_multi_periods` is 2,440 lines — High (maintainability)

One function, 11 parameters, L1936–L4375. Over half the file. It contains the entire threshold-hold path inline: period generation, universe construction, scoring, Bayesian weighting, turnover and cost, regime overrides, result assembly — plus dozens of nested closures that make up much of the bulk.

Why this matters beyond aesthetics:

- **It can't be unit tested.** There are no seams. You can test the whole thing or nothing.
- **It can't be reviewed with confidence.** I'll say plainly below which parts of it I did *not* verify, and the reason is length.
- **Bugs will be expensive.** A defect in the threshold-hold path lives somewhere in 2,440 lines with no smaller unit to isolate it in.

The frustrating part is that the seams already exist. `_setup_period`, `_weight_period`, `_apply_turnover_and_cost`, and `_assemble_period_result` are all module-level helpers with clean signatures — the loop calls them, but then re-inlines a great deal of logic around them instead of pushing that logic down too.

**Recommendation:** decompose along the same lines the single-period pipeline already uses (`stages/`): period setup → selection → weighting → turnover/cost → assembly. The existing helpers are the natural starting points. This is the highest-value change available in this subsystem.

### 2. Constraint logic is re-implemented, and it has already drifted — Medium

The engine carries its own copies of logic that exists in the single-period risk stack:

| Concept | Multi-period | Single-period |
|---------|--------------|---------------|
| Turnover penalty | `engine.py:966` | `risk.py:145` |
| Max active positions | `engine.py:919` | `risk.py:193` |
| Weight bounds | `engine.py:849` | `optimizer.apply_constraints` |

The turnover-penalty *formula* is the same in both — `prev + (target − prev) × (1 − λ)`. What differs is what happens next: the engine re-applies min/max weight bounds (`:981-984`), while `risk.py` renormalises and special-cases `λ ≥ 1`.

So the same `lambda_tc` in the same config produces slightly different behaviour depending on which engine you run. That's not a crash, it's worse — it's a silent inconsistency between two paths a user reasonably expects to agree. And two copies drift further apart over time, not closer.

**Recommendation:** pull the turnover / bounds / max-active primitives into one place (`risk.py`, or a new `weights/constraints.py`) and have both engines call it. If the bounds-vs-renormalise difference is intentional, make it an explicit parameter so the choice is visible.

### 3. `_accepts_keyword` exists three times — Low

`multi_period/engine.py:4391`, `multi_period/loaders.py:26`, `pipeline_entrypoints.py:35`. Trivial helper, trivial fix, but it's three copies of the same six lines.

---

## What I did not verify

I want to be straight about coverage, because "reviewed" can mean very different things for a function this size.

The **hold-state machine** — implemented as nested closures inside the main loop — I mapped and spot-checked, but did **not** verify branch by branch:

- soft and hard z-entry / z-exit thresholds (`_hard_exit_forced`, `_filter_entry_candidates`)
- sticky add/drop counters, `min_tenure` protection, cooldown reseed logic
- one-per-firm dedup (`_dedupe_one_per_firm`)
- buy-and-hold and random selection modes
- turnover-budget max-changes gating (`engine.py:3734+`)

This is where most of the strategy's actual semantics live, and these rules interact — a fund can be simultaneously cooldown-blocked, tenure-protected, and z-exit-forced, and the resolution order matters.

Reading 1,600 lines of interleaved conditionals is not a reliable way to verify that. The right tool is a fixture test: a small synthetic universe, a handful of periods, and explicit assertions about who enters, who exits, and what tenure counters look like at each step. I'd rather flag this honestly than imply coverage I don't have.

---

## Verdict

On the invariants that decide whether a backtest engine is trustworthy — no look-ahead, correct out-of-sample return realisation, correct turnover and cost carry-forward — this engine is **sound**. It also earns real credit for delegating return math to the single-period pipeline instead of duplicating it.

Its debt is size and duplication, not correctness. Two things worth doing, in order:

1. **Decompose the 2,440-line function**, using the existing module-level helpers as the seams.
2. **Write fixture tests for the hold-state machine** — both because it's the least-verified part of the system, and because decomposition is much safer with those tests already in place.

Consolidating the duplicated constraint helpers (finding 2) is a good third, and gets easier once the first is done.

---

## Files reviewed

- [x] `scheduler.py::generate_periods` — full; window construction and look-ahead check
- [x] `engine.py::_apply_turnover_and_cost` (L473–598) — full
- [x] `engine.py::_assemble_period_result` (L601–644) — full
- [x] `engine.py::_apply_turnover_penalty` (L966) — full, compared against `risk.py`
- [x] `engine.py` — loop structure, pipeline delegation (`:4063`), return realisation (`:4326-4352`)
- [x] `loaders.py`, `replacer.py` — scan level, no concerns
- [ ] `_run_threshold_hold_multi_periods` hold-state machine — **mapped, not branch-verified** (see above)
