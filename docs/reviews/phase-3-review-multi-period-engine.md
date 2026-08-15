# Multi-Period Engine
**Scope:** `multi_period/{engine,scheduler,loaders,replacer}.py`

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `engine.py` | 4398 | Period scheduling, universe/selection, weighting, turnover & cost, result assembly |
| `loaders.py` | 227 | `load_prices` / `load_membership` / `load_benchmarks` / `detect_index_columns` |
| `replacer.py` | 227 | `Rebalancer` |
| `scheduler.py` | 140 | `generate_periods` |

Size distribution inside `engine.py`:

| Lines | Function | Span |
|------:|----------|------|
| **2440** | `_run_threshold_hold_multi_periods` | L1936–L4375 |
| 343 | `run` | L1591–L1933 |
| 210 | `run_schedule` | L1049–L1258 |
| 182 | `_run_phase1_multi_periods` | L1261–L1442 |
| 126 | `_apply_turnover_and_cost` | L473–L598 |
| 112 | `_build_rebalance_frame` | L1445–L1556 |

---

## Positive / Clean

### The decomposition seams already exist
`engine.py` defines four typed dataclasses and matching module-level helpers that model the per-period lifecycle cleanly: `_PeriodSetup`/`_setup_period` (`:372`, `:379`), `_PeriodWeights`/`_weight_period` (`:419`, `:440`), `_TurnoverCostApplication`/`_apply_turnover_and_cost` (`:426`, `:473`), `_PeriodResultAssembly`/`_assemble_period_result` (`:433`, `:601`). All are keyword-only, fully annotated, and independently testable. This is good structure — the problem in finding ① is that the main loop only partially uses it.

### `_apply_turnover_and_cost` is well-written
126 lines, keyword-only signature, returns a typed dataclass, and the non-obvious branches carry comments explaining *why* (e.g. `:584-587` documents that renormalisation is skipped when weights don't already sum to ~1, to preserve infeasible bound outcomes). This is the standard the rest of the module should meet.

### `scheduler.py` / `loaders.py` / `replacer.py` are appropriately sized
140–227 lines each, single-purpose, no findings.

---

## Findings

### ① `_run_threshold_hold_multi_periods` — 2,440 lines, 11 parameters, 21 nested closures — High

L1936–L4375. 55% of the file in one function. Beyond raw length, the structural problem is that it defines **21 closures inline**:

```
_parse_month  _valid_universe  _score_frame  _ensure_zscore  _parse_optional_float
_firm  _eligible_sticky_add  _min_tenure_protected  _min_tenure_guard
_reapply_min_tenure_guard  _start_cooldown  _dedupe_one_per_firm
_dedupe_one_per_firm_with_events  _rank_scores_for_bottom_k  _filter_entry_frame
_filter_entry_candidates  _hard_exit_forced  _apply_policy_to_weights
_ensure_holdings_weights  _enforce_min_funds  _compute_weights
```

Each of these closes over the enclosing scope, so none can be imported, called, or unit-tested in isolation — and their dependencies on outer-scope state are implicit rather than declared. Several (`_dedupe_one_per_firm`, `_hard_exit_forced`, `_enforce_min_funds`) are self-contained enough to be module-level functions taking explicit arguments.

The awkward part is that the module already demonstrates the better pattern: the four `_PeriodSetup`-style helpers above are module-level, typed, and keyword-only. The loop calls them, then re-inlines a large amount of logic around them instead of pushing that down too.

**Recommendation:** lift the self-contained closures to module level with explicit parameters, and move the remaining per-stage logic into the existing `_setup_period` / `_weight_period` / `_apply_turnover_and_cost` / `_assemble_period_result` seams. Target shape is the `stages/` layout the single-period pipeline already uses.

### ② Constraint helpers duplicated against `risk.py` / `optimizer.py`, and already divergent — Medium

Three concepts are implemented twice:

| Concept | Multi-period | Single-period |
|---------|--------------|---------------|
| Turnover penalty | `engine.py:966` | `risk.py:145` |
| Max active positions | `engine.py:919` `_enforce_max_active_positions` | `risk.py:193` `_enforce_max_active` |
| Weight bounds | `engine.py:849` `_apply_weight_bounds` | `optimizer.apply_constraints` |

The turnover-penalty formula is identical (`prev + (target − prev) × (1 − λ)`), but the two versions diverge immediately after: `engine.py:981-984` re-applies min/max weight bounds, while `risk.py` normalises and special-cases `λ ≥ 1`. Same config key, two behaviours, depending on which entry point runs.

This is the failure mode duplication actually causes — not a crash, but two copies that were the same once and no longer are, with nothing in the code marking them as needing to stay in sync.

**Recommendation:** extract these three primitives into one module (`risk.py`, or a new `weights/constraints.py`) and have both engines import them. If the bounds-vs-normalise difference is deliberate, express it as a parameter so the divergence is visible at the call site instead of implied by which file you're in.

### ③ `_accepts_keyword` — three byte-identical copies — Low

`multi_period/engine.py:4391`, `multi_period/loaders.py:26`, `pipeline_entrypoints.py:35`. Same eight lines, same `inspect.signature` guard, verbatim in all three:

```python
def _accepts_keyword(func: Any, keyword: str) -> bool:
    try:
        params = inspect.signature(func).parameters
    except (TypeError, ValueError):
        return False
    return keyword in params or any(
        param.kind is inspect.Parameter.VAR_KEYWORD for param in params.values()
    )
```

**Recommendation:** one home in `util/`, import in all three.

### ④ Untyped `*args/**kwargs` shims — Low

Same pattern I flagged in `pipeline.py` (finding ④ there), repeated here at `engine.py:102-106`:

```python
def _run_analysis(*args: Any, **kwargs: Any) -> PipelineResult:
    return _invoke_analysis_with_diag(*args, **kwargs)

def _call_pipeline_with_diag(*args: Any, **kwargs: Any) -> DiagnosticResult[...]:
```

The docstring is explicit that this exists so tests can monkeypatch `_run_analysis` and return raw dicts. That's a legitimate need, but the cost is that the engine's single most important call boundary — the handoff into the analysis pipeline — is untyped in both directions, so nothing checks that the loop is passing what the pipeline expects.

**Recommendation:** give the shim the real signature (or `typing.ParamSpec`) and keep the monkeypatch seam via an explicit injectable rather than signature erasure. Low urgency; worth doing whenever ① is tackled, since both touch the same boundary.

### ⑤ Nine function-level imports — Low

Deferred imports scattered through the module: `:247`, `:1293`, `:1363`, `:1434`, `:1482`, `:1823`, `:2133`, `:2199`, `:2870` — pulling from `..plugins`, `..perf.cache`, `..data`, `..core.rank_selection`, `..selector`.

Some may be deliberate circular-import breaks, but there's no comment saying so on any of them, so a reader can't tell "required" from "grew there." Two of them (`:1363`, `:1434`) import from `..perf.cache` in adjacent code paths, which suggests accretion rather than design.

**Recommendation:** move to module level where no cycle exists; where a cycle does exist, add a one-line comment saying which one, so the next person doesn't "clean it up" and break the build.

### ⑥ `cfg.model_dump()` recomputed five times — Low

`:1277`, `:1952`, `:1961`, `:2254`, `:2357` — three of those inside `_run_threshold_hold_multi_periods`, and `:1961` / `:2254` compute the identical `mp_cfg` value:

```python
mp_cfg = cast(dict[str, Any], cfg.model_dump().get("multi_period", {}) or {})
```

Full Pydantic serialisation of the whole config, repeated, to read one sub-key. Minor cost, but it's also a correctness-adjacent smell: five independent snapshots of config state in one call path.

**Recommendation:** dump once at entry, pass the dict (or the `mp_cfg` slice) down.

### ⑦ Four-deep config key fallback chains — Low

`:2270-2278`:

```python
cooldown_periods_raw = portfolio_cfg.get("cooldown_periods")
if cooldown_periods_raw is None:
    cooldown_periods_raw = portfolio_cfg.get("cooldown_months")
if cooldown_periods_raw is None:
    cooldown_periods_raw = mp_cfg.get("cooldown_periods")
if cooldown_periods_raw is None:
    cooldown_periods_raw = mp_cfg.get("cooldown_months")
```

Four accepted spellings across two config sections, with the precedence order encoded only by statement order and documented nowhere. `sticky_add_x` immediately below follows the same shape. Any of these keys can be set and silently lose to a higher-precedence one.

**Recommendation:** at minimum a comment stating the precedence; better, resolve these aliases in the config layer (`config/` already has `lint_keys.py` for exactly this kind of key hygiene) so the engine reads one canonical name.

### ⑧ Cross-module private import — Low

`:2133` imports `_compute_metric_series` from `..core.rank_selection`. Importing another module's underscore-prefixed symbol means the engine depends on `rank_selection`'s private surface, so a refactor there breaks here with no signal.

**Recommendation:** promote it to a public name in `rank_selection` if it's genuinely shared API.

---

## Verdict

**No correctness defects found in the paths reviewed**, and the module has real structural strengths: the four typed period-lifecycle dataclasses are exactly the right seams, and `_apply_turnover_and_cost` is a well-built, well-commented function.

The findings are almost entirely **structure and hygiene**. The dominant one is ① — a 2,440-line function holding 21 closures, which is untestable in pieces and blocks meaningful review of the code inside it. Everything else is smaller: duplicated constraint helpers that have already drifted (②), a triplicated utility (③), signature erasure at the main call boundary (④), and four low-severity hygiene items (⑤–⑧) that all point the same direction — the module has been extended in place for a while without consolidation.

Order I'd tackle them: ① first (the seams already exist, so this is refactor-not-redesign), then ② while the constraint call sites are fresh, then the rest opportunistically. ⑦ is worth doing sooner than its severity suggests, since config-key precedence bugs are silent.
