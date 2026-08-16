# LLM Integration
**Scope:** `llm/*` (16 files), `llm_proxy/*`, `proxy/*`

---

## Subsystem Summary

| File | Lines | Role |
|------|-------|------|
| `llm/chain.py` | 904 | LangChain wrappers: `ConfigPatchChain`, `ConfigPatchVariantsChain`, `ResultSummaryChain` |
| `llm/result_metrics.py` | 605 | Metric catalog extraction/compaction for prompts |
| `llm/prompts.py` | 408 | Prompt templates |
| `llm/result_validation.py` | 392 | Validation of result-explanation outputs |
| `llm/injection.py` | 277 | Prompt-injection defences |
| `llm/validation.py` | 268 | Validation of `ConfigPatch` ops against the config schema |
| `llm/replay.py` | 247 | NL operation replay |
| `llm/tracing.py` | 245 | LangSmith tracing setup |
| `llm/result_feedback.py` | 180 | Deterministic feedback construction |
| `llm/analysis_fleet.py` | 130 | Fleet event emission |
| `llm/__init__.py` | 121 | Package exports |
| `llm/nl_logging.py` | 114 | NL operation logging |
| `llm/providers.py` | 94 | Provider factory (`openai` / `anthropic` / `ollama`) |
| `llm/schema.py` | 85 | Response schemas |
| `llm/variants.py` | 52 | Variant helpers |
| `llm/_env.py` | 15 | Env parsing helper |
| `proxy/*` | 369 | Streamlit **WebSocket** proxy |
| `llm_proxy/*` | 313 | **LLM** proxy for shared deployments |

Largest functions:

| Lines | Function | Location |
|------:|----------|----------|
| **187** | `_run_chain` | `chain.py:417` |
| 98 | `compact_metric_catalog` | `result_metrics.py:151` |
| 81 | `validate_result_claims` | `result_validation.py:179` |
| 60 | `build_deterministic_feedback` | `result_feedback.py:33` |

---

## Positive / Clean

### `providers.py` is a textbook factory
94 lines doing exactly one job well. Providers are imported lazily via `importlib` (`:48`) so an uninstalled backend doesn't break import of the package, and the failure is converted into an actionable message rather than a raw `ImportError`:

```python
raise RuntimeError(
    f"Provider dependency '{module_name}' is not installed. "
    "Install the Trend Model LLM extras to use this provider."
) from exc
```

Config is a `dataclass(slots=True)`, kwargs are assembled explicitly with `None` checks rather than dict-splatting, and the API-key fallback chain (`explicit → TREND_LLM_API_KEY → provider-specific env`, `:79-90`) is readable in one pass. This module is the quality bar for the rest of the subsystem.

### `chain.py` has a real inheritance structure
Despite being the largest file, it isn't a flat pile of functions: `_LLMRuntimeMixin` (`:147`) holds shared runtime behaviour, `_BaseConfigPatchChain` (`:236`) holds patch-chain behaviour, and `ConfigPatchChain` (`:606`) / `ConfigPatchVariantsChain` (`:679`) are thin specialisations. `ResultSummaryChain` (`:756`) reuses the mixin independently. The abstraction is doing real work.

### Error handling is mostly deliberate
13 broad `except Exception` across the package, but only **one** swallows silently with `pass`; four log. For glue code against a flaky external API, that ratio is better than typical.

### `_env.py` resists the obvious temptation
15 lines, one function, and it *raises* on a malformed value rather than silently falling back to the default:

```python
raise ValueError(f"{name} must be a float, got {value!r}.")
```

Small file, but it's the right call — a typo'd env var surfaces instead of silently changing behaviour.

---

## Findings

### ① `_normalize_text` — same name, two different behaviours — Medium

Defined in two modules in the same package, and they do **not** compute the same thing:

```python
# injection.py — Unicode NFKC normalisation
normalized = unicodedata.normalize("NFKC", text)
return " ".join(normalized.lower().split())

# result_metrics.py — regex tokenisation, space-padded
normalized = _TOKEN_RE.sub(" ", text.lower()).strip()
if not normalized:
    return ""
return f" {normalized} "
```

One does Unicode canonicalisation and collapses whitespace; the other strips non-token characters and pads with sentinel spaces for substring matching. Same name, same package, different contracts, different outputs for the same input.

This is worse than a plain duplicate. A duplicate is a maintenance cost; this is a **trap** — someone reading `injection.py` and then `result_metrics.py` will reasonably assume the shared name means shared behaviour, and refactoring one to call the other would silently break whichever caller depended on the other semantics.

**Recommendation:** rename to say what each does — `_nfkc_fold` and `_tokenize_padded`, or similar. This is cheap and eliminates the ambiguity entirely.

### ② `_strip_code_fence` duplicated across packages, byte-identical — Medium

`config/patch.py:252` and `llm/chain.py:105`. I diffed them: identical.

The surrounding code is a parallel structure too — both packages implement the same "parse LLM output into a validated object, with retries" pattern:

| Concern | `config/patch.py` | `llm/chain.py` |
|---------|-------------------|----------------|
| Strip markdown fence | `_strip_code_fence:252` | `_strip_code_fence:105` (identical) |
| Parse with retries | `parse_config_patch_with_retries:205` | `parse_config_patch_variants_with_retries:118` |
| Format retry error | `format_retry_error:515` | *(imports from patch.py)* |

So the LLM package already imports `format_retry_error` from `config/patch.py` — meaning the dependency edge exists and is accepted — but `_strip_code_fence` was copied instead of imported alongside it.

**Recommendation:** import `_strip_code_fence` from the same place `format_retry_error` comes from (promoting it to a public name), and consider whether the two retry-parsers should share one generic implementation parameterised by target model.

### ③ `_fund_from_weight_path` duplicated — Low

`result_metrics.py` and `result_feedback.py`. Two modules in the same package deriving a fund identifier from a weight path independently.

**Recommendation:** one definition; `result_feedback.py` already sits downstream of `result_metrics.py` conceptually, so importing is natural.

### ④ `_run_chain` is 187 lines — Low

`chain.py:417`, inside `_BaseConfigPatchChain`. Roughly double the next-largest function in the subsystem and the one place where the file's otherwise-good structure breaks down. Given that the class hierarchy around it is well cut, this reads like the method that accumulated the retry/validate/trace/emit concerns rather than delegating them.

**Recommendation:** extract the retry loop, the validation step, and the fleet-event emission into methods on the mixin.

### ⑤ `proxy/` and `llm_proxy/` are unrelated despite the names — Low

Two sibling packages, similar names, entirely different jobs:

- `proxy/` — *"Streamlit WebSocket proxy server… forwards HTTP requests and WebSocket connections to a Streamlit application"*
- `llm_proxy/` — *"LLM proxy server for shared Streamlit deployments"*

I diffed them expecting duplication and found 365 differing lines in `server.py` — they're genuinely separate implementations. But both expose `cli.py`, `server.py`, `__main__.py`, and one is named as if it were the general case of the other.

`proxy/` also isn't really part of this subsystem — it proxies Streamlit traffic and has nothing to do with LLMs.

**Recommendation:** rename `proxy/` to `streamlit_proxy/` so the two are distinguishable at a glance and the LLM one isn't implied to be a variant of it.

### ⑥ `tracing.py` mutates the process environment — Low

`tracing.py:37-38`:

```python
if not os.environ.get("LANGCHAIN_API_KEY"):
    os.environ["LANGCHAIN_API_KEY"] = api_key
```

Copying `LANGSMITH_API_KEY` into `LANGCHAIN_API_KEY` is a pragmatic bridge between two SDK generations, but writing a credential into `os.environ` is a global, process-wide side effect from a library module. It persists for anything else in the process, including subprocesses spawned later.

It is guarded (only sets when unset), so it won't clobber an explicit value.

**Recommendation:** pass the key to the tracing client explicitly if the SDK allows it; if the env var is genuinely the only channel, scope it to a context manager that restores the prior state — `tracing.py` already has `langsmith_tracing_context` (`:180`) which would be the natural home.

### ⑦ Default model hardcoded in two places — Low

`"gpt-4o-mini"` appears as a default in `providers.py:16` and again in `replay.py:113`. Two independent defaults for the same decision will drift when the default changes.

**Recommendation:** single module-level constant; `replay.py` should fall back to `LLMProviderConfig`'s default rather than restating it.

---

## Verdict

Solid subsystem with one genuinely excellent module. `providers.py` is the best-written file I've read in this codebase — lazy imports, actionable errors, explicit kwargs, readable fallback chain — and `chain.py`'s mixin/base-class structure shows the same care at a larger scale.

The findings are mostly small, but ① is worth prioritising despite its size. Two functions sharing a name while computing different things is the kind of thing that reads as fine in review and bites during a refactor, and it costs nothing to fix. ② is next: the two packages have already accepted a dependency edge between them (`format_retry_error` is imported across it), so copying `_strip_code_fence` instead of importing it is an inconsistency rather than a deliberate decoupling.

Order: ① and ③ together (both renames/imports within `llm/`), then ②, then ⑤ if package naming is being touched anyway. ⑥ is worth doing before anything else starts depending on the env var being set as a side effect.
