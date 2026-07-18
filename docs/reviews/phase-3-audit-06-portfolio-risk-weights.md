# Phase-3 First-Pass Audit 06: Portfolio / Risk / Weights

**Date:** 2026-07-16
**Reviewer:** Project review (first pass)
**Branch:** `phase-3`
**Scope:** `risk.py`, `weighting.py`, `engine/optimizer.py`, `engine/walkforward.py`, `rebalancing/strategies.py`, `portfolio/weight_policy.py`, `weights/*`

**Headline:** this subsystem owns `compute_constrained_weights`, so it is where audit-05b finding ① ("vol-targeting scale double-applied") is resolved. **It is now numerically tested — and the finding is substantially corrected (see below).**

---

## Resolution of audit-05b Finding ① — CORRECTED

I ran a numerical fixture against the real `compute_constrained_weights` (2 assets, `vol_A ≈ 2·vol_B`, equal base, `target_vol=0.10`, 240 monthly obs):

```
returned weights:      {A: 0.3372, B: 0.6628}
diag.scale_factors:    {A: 0.3576, B: 0.7030}
weight ratio B/A = scale ratio B/A = 1.9659
normalize(base*scale)  = {A: 0.3372, B: 0.6628}   # exact match to returned weights
realised vol, portfolio.py path (scale in weights AND returns) = 0.0771
realised vol, "single scale" (my proposed 05b fix)             = 0.1394
target_vol = 0.10   (theoretical diversified per-asset target ≈ 0.073)
```

**What this confirms:** the returned weights **do** embed the scale tilt (`weights == normalize(base·scale)`, exactly), and the returns are scaled by `scale_factors` again in `portfolio.py`. The arithmetic in 05b was right.

**What this refutes:** the *characterisation* as a bug, and my proposed fix. The `portfolio.py` "double-scale" path produces a **coherent** vol-targeted result (0.077, sensibly below the 0.10 target due to diversification across two assets each levered to target). My proposed remedy — feeding unscaled returns to the user portfolio — produces **0.139**, which *overshoots* the target and is *further* from correct. So my suggested fix was wrong.

**Corrected conclusion:** `compute_constrained_weights` + `portfolio.py` implement a defensible per-asset volatility-targeting scheme with a low-volatility capital tilt. **Not a correctness bug.** Finding ① is downgraded from "High (if confirmed)" to a **Low documentation observation** (see residual below). The 05b document has been updated accordingly.

### Residual (Low) — "equal" weighting is not equal under vol targeting

With `weighting_scheme="equal"` (or `custom_weights=None`) **and** `target_vol` set, the effective capital weights become `normalize(base·scale) = normalize(scale)` — i.e. tilted toward low-volatility assets — rather than equal. This is standard vol-targeting behaviour, but a user who selects "equal weights" and sees a low-vol-tilted book may be surprised, and the equal-weight *benchmark* (which is genuinely equal over the scaled assets) is constructed on a different basis than the user portfolio.

**Recommendation:** document that vol targeting re-tilts capital; no code change required.

**05b Finding ② still stands** (independent of the above): `_compute_stats` hardcodes annualisation to 12 by not passing `window.periods_per_year`. Latent bug for non-monthly data. Medium.

---

## Subsystem Summary

| File | Lines | Role | Notes |
|------|-------|------|-------|
| `risk.py` | 340 | `compute_constrained_weights`, `realised_volatility` | Reviewed; logic sound |
| `engine/optimizer.py` | 314 | `apply_constraints` (clip + redistribute) | Reviewed; sound |
| `engine/walkforward.py` | 271 | `walk_forward`, `Split` | Scan-level |
| `weighting.py` | 172 | Equal/ScoreProp/AdaptiveBayes weighting | Reviewed Day 3 |
| `rebalancing/strategies.py` | 453 | Turnover-cap / drift-band / vol-target rebalancers | Scan-level |
| `portfolio/weight_policy.py` | 125 | `apply_weight_policy` (drop/keep on signal) | Scan-level |
| `weights/robust_weighting.py` | 390 | Shrinkage (Ledoit-Wolf, OAS), safe-mode | Reviewed key paths |
| `weights/{risk_parity,equal_risk_contribution,hierarchical_risk_parity}.py` | 50/102/140 | Covariance-based engines | Scan-level |
| `weights/constrained_optimization.py` | 93 | Convex constrained engine | New; scan-level |
| `weights/robust_config.py` | 81 | Robustness→engine-params mapping | New; scan-level |

---

## Findings

### ① `compute_constrained_weights` — sound

The pipeline is coherent: `_normalise(base)` → `apply_constraints` → `_normalise` → multiply by `scale_factors` → `_normalise` → `apply_constraints` → turnover penalty → turnover cap → max-active → final `_normalise` (`risk.py:256-301`). The `RiskDiagnostics.scale_factors` returned are the raw per-asset scale, correctly consumed downstream. Numerically verified to produce sensible vol-targeted weights (above).

### ② `engine/optimizer.apply_constraints` — sound

`_clip_series` + `_redistribute` enforce `max_weight` by clipping excess and redistributing proportionally to assets with capacity, raising `ConstraintViolation` when infeasible (`optimizer.py:35-60`). Long-only and group-cap handling layered cleanly. No defects found on review.

### ③ Three risk-parity-family engines — Low (observation)

`RiskParity`, `EqualRiskContribution`, and `RobustRiskParity` (`weights/`) are related but genuinely distinct algorithms (naive inverse-vol vs. ERC iteration vs. shrinkage-robust). Not duplication, but worth a short docstring cross-reference so future readers understand why all three exist.

### ④ `robust_weighting` shrinkage + safe-mode — sound (scan)

`ledoit_wolf_shrinkage`, `oas_shrinkage`, `diagonal_loading`, and a condition-number `safe_mode` fallback are present with diagnostics surfaced up to `_compute_weights_and_stats` (the `used_safe_mode` path reviewed in audit-05b). Full numerical validation of the shrinkage estimators was not performed here.

---

## Verdict for Subsystem F

The core risk/weight machinery is **sound**. Most importantly, the deep-correctness concern from 05b (finding ①) **does not hold** once tested numerically — the vol-targeting is coherent and my proposed fix was wrong; I've corrected the 05b record. The one genuine residual is a documentation gap ("equal" ≠ equal under vol targeting) plus the still-valid 05b finding ② (hardcoded annualisation in `_compute_stats`).

This is a good reminder of the value of the numerical check: a static read produced a plausible-but-wrong "High severity" alarm.

---

## Files Reviewed

- [x] `src/trend_analysis/risk.py` (`compute_constrained_weights` — numerically tested)
- [x] `src/trend_analysis/engine/optimizer.py` (`apply_constraints`)
- [x] `src/trend_analysis/weights/robust_weighting.py` (shrinkage/safe-mode paths)
- [x] `src/trend_analysis/weights/constrained_optimization.py` (new; scan)
- [x] `src/trend_analysis/weights/{risk_parity,equal_risk_contribution,hierarchical_risk_parity}.py` (scan)
- [x] `src/trend_analysis/weights/robust_config.py` (new; scan)
- [~] `engine/walkforward.py`, `rebalancing/strategies.py`, `portfolio/weight_policy.py` (scan-level only)
- Numerical fixture: `scratchpad/check_vol_scale.py`
