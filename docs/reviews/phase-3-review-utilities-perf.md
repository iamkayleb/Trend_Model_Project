# Utilities & Perf
**Scope:** `trend_analysis/util/*`, `trend_analysis/perf/*`, `utils/paths.py`, `backtest/utils.py`

*Final subsystem in the phase-3 series. Because `util/` is where shared helpers belong, this review also closes the loop on duplication findings raised across the other reviews.*

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `trend_analysis/util/missing.py` | 232 | `apply_missing_policy`, policy/limit coercion |
| `trend_analysis/util/frequency.py` | 186 | `detect_frequency`, `infer_periods_per_year` |
| `trend_analysis/util/hash.py` | 129 | `content_run_id`, `working_run_id`, JSON normalisation |
| `trend_analysis/util/rolling.py` | 72 | `rolling_shifted` |
| `trend_analysis/util/weights.py` | 40 | `normalize_weights` |
| `trend_analysis/util/risk_free.py` | 36 | `resolve_risk_free_settings` |
| `trend_analysis/util/joblib_shim.py` | 27 | `dump` / `load` fallback |
| `trend_analysis/util/git.py` | 20 | `git_hash` |
| `trend_analysis/perf/cache.py` | 267 | Covariance cache |
| `trend_analysis/perf/rolling_cache.py` | 195 | Rolling metric cache |
| `trend_analysis/perf/timing.py` | 81 | Timing helpers |
| `utils/paths.py` | 40 | `proj_path` |
| `backtest/utils.py` | 46 | Backtest helpers |

Every file is under 270 lines and each has a single clear purpose. No oversized functions anywhere in the subsystem — a first in this review series.

---

## Positive / Clean

### `util/` is well-built and correctly scoped
Eight focused modules, each one concern, largest 232 lines. The package docstring states the design rule explicitly:

> *"New modules should keep dependencies minimal so they remain light-weight and importable in isolation."*

That's the right constraint for a utility package, it's written down, and the modules honour it.

### `utils/paths.py` handles the environment properly
`proj_path` resolves relative to the repo root regardless of cwd, with a `TREND_REPO_ROOT` override documented for containerised deployments. 40 lines, does one thing, anticipates the deployment case.

### `joblib_shim.py` is an honest dependency fallback
27 lines providing `dump`/`load` when joblib isn't installed, rather than making joblib a hard requirement for the whole package.

---

## Findings

### ① `apply_missing_policy` exists twice with **incompatible signatures** — High

| Location | Signature |
|----------|-----------|
| `util/missing.py:134` | `(df, policy, limit=None, *, columns=None, enforce_completeness=True)` |
| `io/market_data.py:354` | `(frame, policy, *, limit=None)` |

`limit` is **positional-or-keyword** in one and **keyword-only** in the other. So:

```python
apply_missing_policy(df, "ffill", 3)   # works with util/, TypeError with market_data's
```

The `util/` version also takes `columns` and `enforce_completeness`, which the other lacks entirely.

Both are live. The pipeline imports the `util/` one:

```
stages/preprocessing.py:13   from ..util.missing import MissingPolicyResult, apply_missing_policy
multi_period/engine.py:62    from ..util.missing import apply_missing_policy
pipeline.py:97               from .util.missing import MissingPolicyResult, apply_missing_policy
```

while `io/market_data.py` uses its own during ingestion.

This is the most dangerous duplication I've found in the codebase. It's not two copies of the same function — it's two functions with the same name, overlapping-but-different parameters, and different capabilities, both handling missing-data policy on the ingest and analysis paths. A reader who greps the name will find whichever comes first and reasonably assume it's *the* implementation.

**Recommendation:** make `io/market_data.py` import the `util/` version. If ingestion genuinely needs different behaviour, it needs a different name (`apply_ingest_missing_policy`) so the two are distinguishable at the call site.

### ② `infer_periods_per_year` — two implementations, same signature, different arithmetic — Medium

`util/frequency.py:159` and `walk_forward.py:143`, both `(index: pd.DatetimeIndex) -> int`. Same thresholds and return values (252/52/12/4), but they compute the median interval differently:

```python
# util/frequency.py — via int64 nanoseconds
diffs = np.diff(index.values.astype("datetime64[ns]").astype(np.int64))
median_days = np.median(diffs) / (24 * 60 * 60 * 1e9)

# walk_forward.py — via timedelta64 days
diffs_days = np.diff(index.to_numpy()) / np.timedelta64(1, "D")
median_days = float(np.median(diffs_days))
```

`walk_forward.py` doesn't import from `util/` at all — the canonical helper exists one package away and the module is unaware of it.

Both will agree on ordinary inputs. They can diverge on edge cases (integer truncation vs float division, non-nanosecond dtypes), and there's no test pinning them together.

**Recommendation:** `walk_forward.py` should import from `util.frequency` and its local copy should go.

### ③ `detect_frequency` — same name, different contract — Medium

| Location | Signature |
|----------|-----------|
| `util/frequency.py:125` | `(index: Iterable[object]) -> FrequencySummary` |
| `io/validators.py:132` | `(df: pd.DataFrame) -> str` |

One takes an index and returns a structured summary object; the other takes a DataFrame and returns a bare string. Genuinely different functions sharing a name across packages.

**Recommendation:** rename the `io/validators.py` one to reflect what it is (`detect_frequency_label`), or have it delegate to the `util/` version and format the result.

### ④ Three separate utility packages — Medium

- `trend_analysis/util/` (742 lines) — singular
- `utils/` (45 lines) — **plural**, top-level, holds only `proj_path`
- `backtest/utils.py` (46 lines) — a third

`util` vs `utils` differing by one character at different levels of the tree is a genuine readability hazard — `from utils.paths import proj_path` and `from ..util.missing import ...` appear in the same files.

`utils/paths.py` is imported very widely (config, data, tooling), so it's load-bearing and can't simply be folded in without touching many call sites — but the naming should not stay as-is.

**Recommendation:** rename the top-level package to something unambiguous (`project_paths/`, or move `proj_path` into `trend_analysis/util/paths.py`) so there is one utility namespace, not two differing by a plural.

### ⑤ `util/` is under-used — the duplicates flagged across this review series belong here — Medium

This is the cross-cutting observation the rest of the series builds to. `util/` is well-designed but thin, while the same handful of helpers are re-implemented across the codebase:

| Helper | Copies | Where |
|--------|-------:|-------|
| `_json_default` | 4 | `trend/cli.py`, `walk_forward.py`, `backtesting/harness.py`, `core/rank_selection.py` |
| `_accepts_keyword` | 3 | `multi_period/engine.py`, `multi_period/loaders.py`, `pipeline_entrypoints.py` |
| `_coerce_series` | 3 | `unified.py`, `quick_summary.py`, `reporting/portfolio_series.py` |
| `_coerce_float` | 3 | `monte_carlo/{costs,scenario,strategy/sampler}.py` |
| `_is_number` | 3 | `monte_carlo/{runner,strategy/variant,strategy/sampler}.py` |
| `_strip_code_fence` | 2 | `config/patch.py`, `llm/chain.py` |
| `_fix_invalid_day` | 2 | `io/market_data.py`, `trend/input_validation.py` |
| `_ensure_datetime_index` | 2 | `monte_carlo/models/{base,bootstrap}.py` |

Several are byte-identical (`_accepts_keyword`, `_strip_code_fence`, `_fix_invalid_day`, `_ensure_datetime_index`). Others have quietly diverged (`_coerce_series` — one raises, one returns empty; `_normalize_text` in `llm/` — two different algorithms).

The package that should hold these exists, is documented, and has the right dependency rule. It just isn't being reached for.

**Recommendation:** a single consolidation pass adding `util/json_compat.py` (`_json_default`), `util/introspect.py` (`_accepts_keyword`), `util/coerce.py` (`_coerce_float`, `_is_number`, `_coerce_series`), and `util/text.py` (`_strip_code_fence`). That's four small modules retiring ~20 scattered definitions. Worth doing as one deliberate piece of work rather than opportunistically — the value is in the consistency, not any individual removal.

### ⑥ `util/hash.py` defines `sha` three times — Low

`grep '^def '` reports `sha` at three separate points in the 129-line module (plus `normalise_for_json`, `content_run_id`, `_config_payload`, `working_run_id`). These are almost certainly `@overload` declarations for a typed signature — in which case this is fine and I'm noting it only because it's indistinguishable from redefinition at a glance.

**Recommendation:** none if they're overloads. Worth a confirming look, since an accidental redefinition would silently shadow.

---

## Verdict

**The best-constructed subsystem in the repo, and simultaneously the most under-used.** Every module is small, single-purpose, and honours a documented dependency rule; there isn't an oversized function anywhere in 1,387 lines. `utils/paths.py` even anticipates containerised deployment.

The findings aren't really about the code that's here — they're about what isn't. Three canonical helpers (`apply_missing_policy`, `infer_periods_per_year`, `detect_frequency`) have competing implementations elsewhere in the tree, and roughly twenty scattered private helpers across the other subsystems are duplicating work that belongs in this package.

① is genuinely urgent: two functions named `apply_missing_policy` with incompatible parameter binding, both live, both on data-handling paths. That will produce a `TypeError` for anyone who calls it positionally against the wrong one, and worse, it means missing-data policy is implemented twice with different capabilities.

Then ② and ③ (reconnect `walk_forward` and `io/validators` to the canonical versions), then ⑤ as a deliberate consolidation pass. ④ is a rename that makes the whole thing easier to reason about and should probably come first, since it decides where the consolidated modules land.
