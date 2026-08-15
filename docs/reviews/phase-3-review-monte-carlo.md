# Monte Carlo
**Scope:** `monte_carlo/**` (20 files), `src/trend/mc/{viz,io,charts}.py`

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `runner.py` | 2359 | Orchestration: path loop, mixture/two-layer modes, strategy evaluation, cost RNG |
| `aggregator.py` | 860 | Quantile/path aggregation, breach & shortfall specs |
| `trend/mc/viz.py` | 618 | MC visualisation + CLI |
| `scenario.py` | 530 | Scenario config parsing |
| `registry.py` | 518 | Model/strategy registry, scenario parsing |
| `models/regime.py` | 457 | Regime labeller + regime-conditioned bootstrap |
| `results.py` | 445 | Result containers |
| `costs.py` | 384 | Cost / slippage model |
| `folds.py` | 336 | Fold construction |
| `models/base.py` | 306 | `PricePathModel` ABC, price↔log-return helpers |
| `export.py` | 299 | Export helpers |
| `strategy/sampler.py` | 294 | Distribution parsing + strategy variant sampling |
| `strategy/variant.py` | 278 | `StrategyVariant`, override merging |
| `models/bootstrap.py` | 265 | Stationary bootstrap model |
| `trend/mc/io.py` | 172 | MC artifact IO |
| `cache.py` | 155 | MC caching |
| `export_bundle.py` | 143 | Bundle export |
| `config.py` | 118 | MC config |
| `strategy/validation.py` | 112 | Strategy pack validation |
| `seed.py` | 66 | `SeedManager` |
| `trend/mc/charts.py` | 8 | Chart re-export shim |

---

## Positive / Clean

### `runner.py` is decomposed despite its size
2,359 lines, but the largest function is `run` at 134 lines and the mean is far smaller — the file is *long*, not *monolithic*. Compare the multi-period engine, where a single function is 2,440 lines. Mode-specific paths are separated into `_run_mixture` (`:493`) and `_run_two_layer` (`:394`), and per-strategy work into `_evaluate_strategy` (`:661`). This is a file that could be split by moving whole functions, which is a much cheaper refactor than extracting them first.

### `seed.py` is small, focused, and well-documented
66 lines, one class, a docstring that states the seeding scheme explicitly (`:13-19`), and `_pack_part` (`:63`) length-prefixes each component so distinct inputs can't collide into the same digest. Good single-purpose module.

### The model layer has a real abstraction
`PricePathModel` is a proper ABC with `@abstractmethod simulate` and an abstract `frequency` property (`models/base.py:44-53`), and the concrete models conform. `PricePathResult` (`:34`) gives the return type a name rather than passing tuples around.

---

## Findings

### ① Four helpers duplicated between `models/base.py` and `models/bootstrap.py` — Medium

`_ensure_datetime_index`, `_build_simulation_index`, `_build_multi_columns`, and `_last_valid_prices` are each defined in **both** files. I diffed `_ensure_datetime_index` — byte-identical:

```
base.py:85   def _ensure_datetime_index(index: Iterable[object]) -> pd.DatetimeIndex:
bootstrap.py:27  def _ensure_datetime_index(index: Iterable[object]) -> pd.DatetimeIndex:   # identical
```

These are sibling modules in the same package, one importing the other's concepts. There is no import-cycle reason for the split — `bootstrap.py` could import from `base.py` directly.

**Recommendation:** keep them in `base.py` and import in `bootstrap.py`. Four copies is where drift starts; `_build_simulation_index` in particular encodes index construction that must stay consistent across models.

### ② `normalize_frequency_code` / `_normalize_frequency_code` — alias twins — Low

`models/base.py:91` defines the public function; `:107` defines a private one whose entire body is:

```python
def _normalize_frequency_code(freq: str | None, *, quarterly: str = "M") -> str:
    return normalize_frequency_code(freq, quarterly=quarterly)
```

Two names for one function, differing only by underscore, sitting 16 lines apart. This is the same pattern I flagged in `io/market_data.py` (`_normalise_delta_days` wrapping `_normalize_delta_days`) — it reads as a rename that was half-completed, and it leaves callers a coin-flip about which to use.

**Recommendation:** pick one and delete the other, or make it an explicit module-level alias (`_normalize_frequency_code = normalize_frequency_code`) with a comment saying it's for backwards compatibility.

### ③ Two RNG libraries and two seeding paths — Medium

The subsystem uses both stdlib `random` and NumPy generators, and they are seeded independently:

| Path | Mechanism |
|------|-----------|
| Price paths, costs | `SeedManager` → blake2b → `np.random.default_rng` (`seed.py:39-45`) |
| Strategy variant sampling | `random.Random(seed)` directly (`runner.py:1387`) |

The `Distribution.sample` protocol in `strategy/sampler.py` is typed against stdlib `random.Random` (`:79`, `:90`, `:102`), so the whole sampler branch is structurally unable to use `SeedManager`, which only vends `np.random.Generator`.

The result is that `seed.py`'s documented seeding scheme — the module that exists specifically to centralise determinism — doesn't actually cover one of the two things that consumes randomness. Anyone reading `seed.py` would reasonably assume it does.

**Recommendation:** standardise on `np.random.Generator` and have the sampler distributions take one, so `SeedManager` becomes the single seeding entry point. If stdlib `random` is required for some reason, add a `SeedManager.get_stdlib_rng()` so at least both streams derive from the same master seed and the contract is visible in one place.

### ④ `np.shares_memory` used as control flow — Low

`models/regime.py:213`:

```python
if not np.shares_memory(bucket, current_bucket):
```

This decides whether to reseed a block position by testing whether two arrays share a buffer — i.e. it relies on the regime→bucket dict handing back the *same array object* each time. It works given how `regime_buckets` is currently built, but it's a load-bearing dependency on object identity that nothing declares or tests. If that dict ever returns a copy, view, or freshly-built array, the branch silently inverts.

The surrounding code already has the explicit signal it needs — the regime id is right there in the loop and is compared directly on the line above.

**Recommendation:** compare regime ids rather than buffer identity.

### ⑤ Duplicated coercion helpers across the subsystem — Low

Beyond the model-layer copies in ①, the same small utilities recur:

| Helper | Files |
|--------|-------|
| `_coerce_float` | `costs.py`, `scenario.py`, `strategy/sampler.py` |
| `_is_number` | `runner.py`, `strategy/variant.py`, `strategy/sampler.py` |
| `_require_mapping`, `_require_non_empty_str` | `strategy/sampler.py`, + siblings |
| `_coerce_formats`, `_coerce_tags`, `_export_frame`, `_metric_columns` | `export.py` / `export_bundle.py` |

15 helper names are defined more than once across the subsystem. Individually trivial; collectively it means config-coercion semantics are defined in several places and can diverge without anything flagging it.

**Recommendation:** one `monte_carlo/_utils.py` (or reuse the existing `trend_analysis/util/`) for the coercion primitives.

### ⑥ Function-level imports, and one absolute import among relatives — Low

Four deferred imports, none commented: `runner.py:1235` (`..multi_period.scheduler`), `:2184` (`concurrent.futures`), `:2240` (`yaml`), and `scenario.py:446`.

The last one is also stylistically inconsistent — it uses a fully-qualified absolute path while the rest of the package uses relative imports:

```python
from trend_analysis.monte_carlo.models.base import normalize_frequency_code
```

`runner.py:1235` is plausibly a genuine cycle break (`multi_period` ↔ `monte_carlo`), but nothing says so.

**Recommendation:** hoist the ones that aren't cycle breaks; comment the ones that are; make `scenario.py:446` relative for consistency.

### ⑦ Nested per-element loops in the regime model — Low (performance)

`models/regime.py:152-155` and `:201-218` iterate `n_paths × n_periods` in Python, calling `rng.choice` / `rng.integers` once per element. The sibling model in `bootstrap.py:110-121` does the equivalent work fully vectorised across paths.

Two implementations of the same conceptual step in the same package, one vectorised and one not, is an inconsistency a reader will trip over — and `rng.choice(..., p=...)` per element is the expensive form.

**Recommendation:** vectorise via cumulative-probability rows plus `searchsorted`, matching the style already used in `bootstrap.py`.

### ⑧ `trend/mc/charts.py` is an 8-line shim — Low

Worth confirming it's still load-bearing rather than a leftover from a move. If it only re-exports, either document why the indirection exists or fold it into callers.

---

## Verdict

**No structural defects that block anything**, and the subsystem is in better shape than its size suggests: `runner.py` is long but genuinely decomposed, `seed.py` is a clean focused module, and the model layer has a real ABC rather than duck-typed conventions.

The findings are consistency and hygiene, and they cluster around one theme — **the same code exists in more than one place**. Four byte-identical helpers across the two model modules (①), an alias twin 16 lines from its original (②), 15 duplicated helper names subsystem-wide (⑤), and two parallel RNG stacks (③). Individually each is small; together they say the package grew module-by-module without a shared utility layer, and the pieces were copied rather than imported.

Order I'd tackle them: ③ first, because it's the one with a real behavioural consequence — `seed.py` advertises a determinism contract it doesn't actually cover. Then ① and ⑤ together as a single "pull the shared helpers into one place" pass, which also removes ②. ④ and ⑦ are localised to `regime.py` and can be done whenever that file is next touched.
