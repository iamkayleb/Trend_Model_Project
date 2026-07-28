# Phase-3 First-Pass Audit 08: Monte Carlo

**Date:** 2026-07-16
**Reviewer:** Project review (first pass)
**Branch:** `phase-3`
**Scope:** `monte_carlo/*` (20 files, ~8,100 lines) + `src/trend/mc/*`

> **Entirely new subsystem** — never reviewed before. ~8,700 lines total. This pass
> **deep-reviews the correctness-critical core** (seeding/determinism and the sampling
> models) and **scans** the larger orchestration/aggregation/reporting modules. Coverage
> is called out per file at the end.

---

## Subsystem Summary

| File | Lines | Role | Depth |
|------|-------|------|-------|
| `monte_carlo/runner.py` | 2359 | Orchestration, path evaluation, cost RNG | Scan + spot |
| `monte_carlo/aggregator.py` | 860 | Quantile/path aggregation | Scan |
| `monte_carlo/scenario.py` | 530 | Scenario config | Scan |
| `monte_carlo/registry.py` | 518 | Model/strategy registry | Scan |
| `monte_carlo/models/regime.py` | 457 | Regime-conditioned bootstrap | **Deep** |
| `monte_carlo/results.py` | 445 | Result containers | Scan |
| `monte_carlo/costs.py` | 384 | Cost/slippage model | Scan |
| `monte_carlo/folds.py` | 336 | Fold construction | Scan |
| `monte_carlo/models/base.py` | 306 | Price-path base, returns↔prices | Read |
| `monte_carlo/models/bootstrap.py` | 265 | Stationary bootstrap | **Deep** |
| `monte_carlo/strategy/*` | 684 | Sampling distributions, variants | Read (sampler) |
| `monte_carlo/seed.py` | 66 | `SeedManager` (determinism core) | **Deep** |
| `src/trend/mc/viz.py` | 618 | MC visualisation | Not reviewed |

---

## Summary of Findings

| # | Finding | Severity |
|---|---------|----------|
| 1 | Seeds masked to 32 bits → birthday-paradox collisions at high path counts | Medium |
| 2 | Regime path/index sampling uses nested Python loops with per-element RNG (bootstrap is vectorised; regime is not) | Medium (perf) |
| 3 | `_regime_conditioned_indices` uses `np.shares_memory` for control flow | Low |
| — | `SeedManager` deterministic scheme | ✅ Correct |
| — | Stationary bootstrap (Politis–Romano), vectorised | ✅ Correct |
| — | Transition-matrix normalisation | ✅ Correct |
| — | `runner.py` well-decomposed (largest fn 134 lines) | ✅ (contrast G) |

---

## Positive / Correct

### `SeedManager` (`seed.py`) — correct, reproducible
Uses **blake2b** (process-stable, unlike Python's salted `hash()`), **length-prefixed** parts in `_pack_part` (so `("1","23")` and `("12","3")` can't collide), and derives **independent per-path and per-strategy** RNG streams. This is a sound determinism foundation, and `runner.py` uses it consistently (`:410,512,1133`). The unseeded `np.random.default_rng()` fallbacks (`runner.py:2064,2103`) only trigger when **no master seed** is configured — i.e. the user opted out of determinism — which is expected behaviour, not a leak.

### Stationary bootstrap (`bootstrap.py:_stationary_bootstrap_indices`) — correct & vectorised
Textbook Politis–Romano: geometric block lengths via `p = 1/mean_block_len`, `new_block[:,0]=True`, circular continuation `(current+1) % n_obs`, forced restart on `reset`. Fully vectorised across paths. No issues.

### Transition-matrix normalisation (`regime.py:_normalize_transition_matrix`) — correct
Clips negatives, row-normalises, and defensively turns zero/non-finite rows into absorbing states (`identity row`). Correct.

---

## Findings

### ① 32-bit seed space → collisions at scale — Medium

`SeedManager._stable_hash` masks the digest to 32 bits (`seed.py:61`: `& 0xFFFFFFFF`), so every path seed is drawn from a 4.29 B space. By the birthday bound, the probability that **two paths share a seed** is non-trivial as path count grows:

- 1,000 paths → ~0.01%
- 10,000 paths → ~1.1%
- ~77,000 paths → ~50%

Two paths with the same seed produce **identical draws** — i.e. duplicate "independent" scenarios — which biases the Monte Carlo distribution by **understating dispersion** (tails and variance). For small path counts it's negligible; for large studies it silently corrupts the result.

**Recommendation:** use NumPy's designed-for-this mechanism — `np.random.SeedSequence(master_seed).spawn(n_paths)` — or at minimum stop masking to 32 bits (blake2b already yields 64; `default_rng` accepts it). This removes the collision risk entirely and is a small change localised to `seed.py`.

### ② Regime sampling not vectorised — Medium (performance/scalability)

Both regime routines use nested Python loops with **per-element** RNG calls:

- `_simulate_regime_path` (`regime.py:152-155`): `for t: for path: rng.choice(n_regimes, p=transition[prev])` → `n_periods × n_paths` individual `rng.choice` calls.
- `_regime_conditioned_indices` (`regime.py:201-218`): per-(path, period) `rng.integers` calls.

For a realistic study (e.g. 10k paths × 120 periods = 1.2 M calls), this is very slow, and it stands in contrast to the plain bootstrap, which is fully vectorised. `rng.choice(..., p=...)` is especially expensive per call.

**Recommendation:** vectorise via inverse-CDF sampling — precompute per-regime cumulative transition rows and draw all paths at timestep `t` with one `rng.random(n_paths)` + `searchsorted`. Same for the index sampler. Correctness is unaffected; this is purely throughput.

### ③ `np.shares_memory` used for control flow — Low

`_regime_conditioned_indices:213` decides whether to reseed the block position via `if not np.shares_memory(bucket, current_bucket)`. This relies on numpy array **object identity** (the regime→bucket dict returning the same array instance) as a control signal. It works today, but it is fragile: if `regime_buckets` ever returned copies/views, the block-continuation logic would silently change. Prefer comparing the regime id (an explicit integer) that already drives the surrounding branch.

---

## Not Reviewed / Scanned Only (flagged honestly)

- `runner.py` (2359) — structure and seeding spot-checked; the mixture/two-layer evaluation paths (`_run_mixture`, `_run_two_layer`) were **not** correctness-audited.
- `aggregator.py` (860) — quantile-frame schema noted; percentile math **not** verified.
- `costs.py`, `scenario.py`, `registry.py`, `results.py`, `folds.py`, `strategy/variant.py` — scan level only.
- `src/trend/mc/viz.py` (618), `io.py`, `charts.py` — **not reviewed** (visualisation/IO).

---

## Verdict for Subsystem H

The **determinism core and sampling mathematics are sound** — `SeedManager`, the stationary bootstrap, and the Markov regime machinery are all correct, and `runner.py` is far better decomposed than the multi-period engine (G). The two substantive findings are a **seed-space limitation** (①, a real statistical bias at high path counts, cheap to fix) and **un-vectorised regime sampling** (②, a scalability issue). Neither is a logic error in the sampled distribution at small scale.

Because the subsystem is ~8,700 lines, this is explicitly a **core-focused** review; the orchestration, aggregation, cost, and visualisation layers warrant their own dedicated passes.

---

## Files Reviewed

- [x] `monte_carlo/seed.py` (full — deep)
- [x] `monte_carlo/models/bootstrap.py` (`_stationary_bootstrap_indices` + model — deep)
- [x] `monte_carlo/models/regime.py` (`_simulate_regime_path`, `_normalize_transition_matrix`, `_regime_conditioned_indices` — deep)
- [x] `monte_carlo/models/base.py` (returns↔prices helpers — read)
- [x] `monte_carlo/strategy/sampler.py` (distributions — read)
- [x] `monte_carlo/runner.py` (structure, seeding, `_cost_rng`/`_sample_strategy` — spot)
- [~] `aggregator.py`, `costs.py`, `scenario.py`, `registry.py`, `results.py`, `folds.py` (scan)
- [ ] `src/trend/mc/*` (visualisation/IO — not reviewed)
