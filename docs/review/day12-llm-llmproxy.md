# Day 12 Review: `llm/` and `llm_proxy/` subpackages

**Scope:** 18 files, ~4,146 lines — the language-model integration layer: prompt construction, structured-output chains, injection detection, result validation, metric extraction, NL operation logging, and a thin OpenAI-compatible reverse proxy
**Date:** 2026-05-15
**Reviewer:** Claude

---

## Files Reviewed

- `llm/__init__.py` — Re-exports from all `llm` submodules. Clean barrel file.
- `llm/schema.py` — `load_compact_schema`, `select_schema_sections`: loads the config JSON schema from disk and filters sections by token-matching the user instruction to keep the prompt compact.
- `llm/providers.py` — `LLMProviderConfig` dataclass and `create_llm` factory for openai/anthropic/ollama via LangChain. `_resolve_api_key` checks `TREND_LLM_API_KEY` then a provider-specific env var.
- `llm/variants.py` — `VARIANT_LABELS`, `normalize_variant_label`, `find_duplicate_variant_labels`, `ensure_unique_variant_labels`: small helpers for the conservative/baseline/aggressive label set.
- `llm/nl_logging.py` — `NLOperationLog` Pydantic model and `write_nl_log`: appends structured JSONL records under a threading lock. `rotate_nl_logs` deletes files older than a configurable number of days.
- `llm/tracing.py` — `langsmith_tracing_context`: context manager that activates LangSmith tracing when `LANGSMITH_API_KEY` is set, propagates `LANGCHAIN_PROJECT`, and falls back to a no-op when deps are absent or the env is flagged as a test environment.
- `llm/prompts.py` — All prompt builders: `build_config_patch_prompt`, `build_variant_patch_prompt`, `build_result_summary_prompt`, `build_comparison_prompt`, `build_retry_prompt`, `build_variant_retry_prompt`. Also defines `DEFAULT_*` system prompts, safety rules, and section-header constants.
- `llm/injection.py` — `detect_prompt_injection` / `detect_prompt_injection_payload`: regex-based scanner that also runs across decoded text variants (HTML-unescape, URL-decode, base64, hex, rot13, unicode-escape) to catch obfuscated payloads.
- `llm/validation.py` — `validate_patch_keys`, `flag_unknown_keys`, `normalize_patch_path`: validates ConfigPatch operation paths against the JSON schema. Handles JSON Pointer paths, dotpaths, array indexing, and wildcards.
- `llm/replay.py` — `replay_nl_entry`: re-runs a recorded NL operation log entry against the LLM and diffs the output against the original. Used for auditing model drift.
- `llm/result_feedback.py` — `build_deterministic_feedback`: generates a diagnostics block flagging high turnover, high concentration, large drawdown, and missing result sections. Thresholds are overridable via env vars.
- `llm/result_metrics.py` — `extract_metric_catalog`, `compact_metric_catalog`, `format_metric_catalog`: builds a bounded, prioritized list of `MetricEntry` objects from a result dict. Covers per-fund stats, aggregate stats, weights, benchmark IR, risk diagnostics, and turnover series.
- `llm/result_validation.py` — `validate_result_claims`, `detect_result_hallucinations`, `postprocess_result_text`, `ensure_result_disclaimer`, `apply_metric_citations`: checks that cited numeric values match the metric catalog, flags uncited numbers, detects unknown citation sources, and appends a discrepancy log.
- `llm/chain.py` — The main chain classes: `ConfigPatchChain`, `ConfigPatchVariantsChain`, `ResultSummaryChain`. `_BaseConfigPatchChain` holds shared state and helpers including structured-vs-text output selection, retry loops, injection gating, and schema validation.
- `llm_proxy/__init__.py` — Empty module marker.
- `llm_proxy/__main__.py` — Calls `main()` from `cli.py`.
- `llm_proxy/server.py` — `LLMProxy`: FastAPI + httpx proxy that replaces the caller's Authorization header with the server-side upstream API key. Lazy-loads fastapi/httpx/uvicorn. Strips hop-by-hop headers. Defaults to `0.0.0.0:8799`.
- `llm_proxy/cli.py` — Argparse CLI for the proxy: `--host`, `--port`, `--upstream`, `--log-level`.

---

## Issues Found

### 1. Bare, non-package-qualified import in `llm/schema.py`

Line 9 of `llm/schema.py`:

```python
from utils.paths import proj_path
```

Every other module in the package uses fully qualified imports (`from trend_analysis.utils.paths import ...`). This bare form works only when the interpreter's working directory happens to be the package root. It will silently shadow any unrelated top-level `utils` package that might be installed in the environment, and it will fail outright when the module is imported from a different working directory (e.g., during `pytest` runs from the repo root or when the package is installed as a wheel).

**Recommendation:** Replace with `from trend_analysis.utils.paths import proj_path`, matching the convention used everywhere else in the codebase.

---

### 2. `detect_result_hallucinations` silences the caller's logger

`result_validation.py`, inside `detect_result_hallucinations` (around line 270):

```python
issues = validate_result_claims(text, entries, logger=None)
```

The outer function accepts a `logger` parameter and passes it to its own logging calls, but it hard-codes `logger=None` when it delegates to `validate_result_claims`. Any validation-level findings — unknown citation sources, mismatched numeric values — are discarded rather than routed through the caller's logger. A caller who passes `logger=my_logger` expecting complete audit coverage will silently miss those inner findings.

**Recommendation:** Forward the logger: `validate_result_claims(text, entries, logger=logger)`.

---

### 3. `ConfigPatchChain.run()` and `ConfigPatchVariantsChain.run()` are ~200 lines of copy-pasted logic

Both methods follow the same structure, step for step: build the prompt, run the injection check, choose between structured and text output, enter the retry loop, validate the schema, and execute the `finally` logging block. The only meaningful differences are the prompt-builder called and the return type.

This duplication means any bug fix or improvement to the retry logic, injection gating, or logging must be applied in two places. Since the two methods already share `_BaseConfigPatchChain` for state, the shared execution logic belongs there too.

**Recommendation:** Extract the shared scaffold into a `_BaseConfigPatchChain._run_chain(prompt_builder, parser)` method that accepts the differing pieces as callables. Both `ConfigPatchChain.run()` and `ConfigPatchVariantsChain.run()` become thin wrappers.

---

### 4. `ResultSummaryChain` reimplements `_bind_llm` rather than inheriting it

`ResultSummaryChain` is a `@dataclass(slots=True)` that does not inherit from `_BaseConfigPatchChain`. As a result it contains its own `_bind_llm` (lines 928–939 of `chain.py`) and `_invoke_llm` (lines 890–926) implementations whose logic is identical to the base-class versions.

The practical impact is the same as issue 3: improvements to LLM binding — adding retry-with-backoff, switching structured-output modes, changing how the trace URL is extracted — need to be made in two separate places that will drift apart.

**Recommendation:** Either make `ResultSummaryChain` inherit from `_BaseConfigPatchChain`, or promote `_bind_llm_with` and the shared invocation logic to a standalone module-level function that both classes call.

---

### 5. `_read_env_float` defined identically in two files

`chain.py:942` and `result_feedback.py:95` both define:

```python
def _read_env_float(name: str, default: float) -> float:
    try:
        return float(os.environ[name])
    except (KeyError, ValueError):
        return default
```

**Recommendation:** Move to `llm/_env.py` (or `utils/env.py` if one exists) and import from there.

---

## Notes

**`_iter_decoded_variants` repeats its decode blocks.** `injection.py` applies the four decode operations (`_maybe_decode_base64`, `_maybe_decode_hex`, `_maybe_decode_rot13`, `_maybe_decode_unicode_escape`) once for a first-pass set of decoded candidates and then applies the same four operations again identically for a second-pass set. The second-pass block is a copy-paste of the first. Extracting a `_apply_decoders(text)` helper and calling it twice would make the intent (two rounds of decoding) clearer and easier to maintain.

**`rotate_nl_logs` runs outside the write lock.** `nl_logging.py` acquires `_NL_LOG_LOCK` only for the actual JSONL append. `rotate_nl_logs`, which deletes files older than the configured retention window, is not covered by the lock. Under concurrent access, two threads could both enumerate old log files and attempt to delete the same file simultaneously, producing a `FileNotFoundError` on the second deletion attempt. This is low-probability in normal operation but worth guarding: either acquire the lock at the top of `rotate_nl_logs`, or use `pathlib.Path.unlink(missing_ok=True)`.

**`build_retry_prompt` and `build_variant_retry_prompt` are near-identical.** Both call their respective base builder then append the `SECTION_RETRY_ERROR` block. The only difference is which base builder they delegate to. A single `build_retry_prompt(base_builder, ...)` that accepts the base builder as an argument would eliminate the duplication.

**`_LLMResponse` uses `__iter__` to carry two values.** `chain.py:79` defines `_LLMResponse` as a `str` subclass with a `trace_url` attribute. Its `__iter__` yields `str(self)` then `self.trace_url or ""`, so callers can unpack it as `text, url = response`. This is an unusual convention that makes the type appear iterable in contexts where only the string is expected (e.g., passing to `str.join`). A simple `NamedTuple("LLMResponse", [("text", str), ("trace_url", str | None)])` would be clearer and harder to misuse.

**`llm_proxy/server.py` binds `0.0.0.0` by default.** The default host exposes the proxy on all interfaces. In production the proxy carries the upstream API key, so binding on all interfaces without authentication is a meaningful attack surface. The default should be `127.0.0.1` unless the deployment context explicitly requires external reachability.
