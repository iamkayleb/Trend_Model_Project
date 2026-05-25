# Day 16 Code Review: scripts/ — Larger Python scripts

**Scope:** All Python scripts in `scripts/` that exceed ~135 lines, excluding the
`scripts/langchain/` subpackage (which is Day 17) and the small utilities already
covered in Day 15. Twenty files, roughly 11,000 lines total.

---

## Files Reviewed

- `scripts/run_multi_demo.py` (2,375 lines) — Sprawling end-to-end smoke test that exercises virtually every public surface of `trend_analysis`. Builds the demo dataset, runs single-period and multi-period pipelines, exercises the GUI helpers, registers and tears down test plugins, runs the full pytest suite via `run_tests.sh`, and validates package-level exports. Most of the work runs at module-import time, not inside `main()`.
- `scripts/test_settings_wiring.py` (1,673 lines) — Validates that every Streamlit UI setting actually affects pipeline output. Holds a 80+ entry `SETTINGS_TO_TEST` list, builds baseline + variant runs for each, applies setting-specific mode setup (e.g. switching to `risk_parity` when testing `max_weight`), and reports per-setting PASS/FAIL with direction checks.
- `scripts/evaluate_settings_effectiveness.py` (885 lines) — Companion analysis: pairs baseline and test simulations, extracts categories by AST-parsing `streamlit_app/pages/2_Model.py`, generates structured `SettingResult` records with mode-specific context.
- `scripts/reproduce_ui_run.py` (626 lines) — Debugging harness that re-runs a Streamlit session deterministically from a captured JSON payload. Exports CSVs of weight history, churn statistics, and z-score tail diagnostics.
- `scripts/autopilot_metrics_collector.py` (614 lines) — Appends typed NDJSON records to an auto-pilot metrics log. Three record kinds (`step`, `cycle`, `escalation`), each schema-validated. Supports both ISO and epoch-ms time bounds, derives duration server-side, emits failure / runtime summary records to `AUTOPILOT_METRICS_SUMMARY_PATH` for the workflow.
- `scripts/generate_settings_evidence.py` (584 lines) — Drives `test_settings_wiring` against real Trend Universe data, formats baseline-vs-test metric comparisons, writes per-setting evidence Markdown files under `docs/settings_evidence/`.
- `scripts/ci_cosmetic_repair.py` (566 lines) — Workflow entry point for cosmetic-only test repairs. Runs pytest, classifies failures, parses `COSMETIC_TOLERANCE` and `COSMETIC_SNAPSHOT` payloads from failure messages, applies guard-comment-anchored file edits, then creates a branch and PR via `gh`.
- `scripts/sync_dev_dependencies.py` (452 lines) — Reads `autofix-versions.env` and rewrites pinned dev tools inside `[project.optional-dependencies.dev]` of `pyproject.toml`. Optional `--create-if-missing` plants a minimal core-tools section if absent.
- `scripts/sync_test_dependencies.py` (428 lines) — AST-scans the `tests/` tree for imports, subtracts stdlib/test-framework/local-package modules, compares to declared deps in `pyproject.toml`, and (with `--fix`) appends missing entries to the `dev` extra via `tomlkit`.
- `scripts/streamlit_param_sweep.py` (375 lines) — Runs baseline + variant simulations driven by JSON/YAML test cases, hashes each input DataFrame for caching, optionally calls an LLM advisor (`comparison_llm`) to flag counter-intuitive parameter sensitivities.
- `scripts/diff_ui_runs.py` (309 lines) — Compares two directories produced by `reproduce_ui_run.py`. Per-file SHA-256 hashing, JSON tree diff with leaf-level paths, numeric CSV column diff with configurable tolerance.
- `scripts/benchmark_performance.py` (297 lines) — Benchmarks three hotspots (`CovCache`, multi-period turnover alignment, turnover-cap rebalancing) with `time.perf_counter()`, validates that the Python and vectorised paths produce equal totals, writes JSON summary.
- `scripts/ci_metrics.py` (241 lines) — JUnit XML aggregator. Computes per-outcome counts, captures failure/error details, surfaces slow tests above a configurable threshold; outputs a single JSON consumed by Phase-2 CI dashboards.
- `scripts/run_threshold_churn_demo.py` (210 lines) — Self-contained demo that overlays a threshold-and-soft-strikes churn rule on the multi-period engine; exports period-level churn schedule and z-score summary CSVs.
- `scripts/ledger_migrate_base.py` (210 lines) — Rewrites the `base:` field in every `.agents/issue-*-ledger.yml` to match the repository's default branch. Has a multi-step branch-detection fallback chain (`git remote show origin` → `symbolic-ref` → `rev-parse` → current branch).
- `scripts/cosmetic_repair_workflow.py` (161 lines) — Three CLI subcommands (`capture`, `evaluate`, `render`) that read the JSON summary written by `ci_cosmetic_repair.py` and emit `GITHUB_OUTPUT` lines or Markdown step summaries.
- `scripts/autopilot_step_timer.py` (143 lines) — Tiny CLI that emits an ISO or epoch-ms timestamp keyed by step event (`start`/`end`); destination is stdout, `$GITHUB_ENV`, `$GITHUB_OUTPUT`, or an arbitrary env file.
- `scripts/compare_perf.py` (140 lines) — Compares a current benchmark JSON against a baseline, fails the build if any monitored metric regresses by more than the threshold percentage.
- `scripts/run_real_model.py` (136 lines) — Drives the multi-period engine on `config/long_backtest.yml`, runs `RankSelector` + `ScorePropBayesian` weighting, exports the per-period weight schedule and a stitched out-of-sample portfolio return series.
- `scripts/generate_demo.py` (198 lines) — Produces the 10-year synthetic demo CSV: 21 manager return streams with target Sharpes between ‑0.35 and +0.85, an SPX-like benchmark, a realistic RF series.

---

## Issues Found

### 1. `ci_cosmetic_repair.py` line 307: deprecated `datetime.utcnow()`

```python
branch_suffix = branch_suffix or datetime.utcnow().strftime("%Y%m%d%H%M%S")
```

`datetime.utcnow()` is deprecated in Python 3.12 and emits a DeprecationWarning. The same file already imports `UTC` and uses the modern form at line 370:

```python
timestamp = datetime.now(UTC).replace(microsecond=0).isoformat()
```

So the inconsistency is internal: one timestamp generator was updated and another was missed. The fix is `datetime.now(UTC).strftime(...)`. This is the same bug class flagged in Day 14 for `coverage_guard.py`.

---

### 2. `run_multi_demo.py`: module-level side effects make this script un-importable

The 2,375-line "smoke test" runs most of its work at module scope rather than inside `main()`:

```python
# run_multi_demo.py:81
setup_script_logging(app_name="multi-demo", module_file=__file__)
...
# line 2209
_check_module_exports()
...
# line 2237
_check_cli_help()
...
# line 2240+
pkg_cfg = ta.config.load("config/demo.yml")
pkg_df = ta.load_csv(pkg_cfg.data["csv_path"])
...
# line 2372-2375 — the last lines of the file
run_tests = Path(__file__).resolve().with_name("run_tests.sh")
result = subprocess.run([str(run_tests)], shell=False)
if result.returncode != 0:
    raise SystemExit(f"{run_tests} failed with exit code {result.returncode}")
```

`python -c "import scripts.run_multi_demo"` triggers the full pytest suite. Any tooling that pre-imports the `scripts` package (linters, dependency scanners, IDE indexers) does the same. The work should live inside a `main()` guarded by `if __name__ == "__main__":`.

---

### 3. `compare_perf.py` `_default_threshold` reads `.env` as a single float

```python
# compare_perf.py:41-52
def _default_threshold() -> float:
    env_path = proj_path(".env")
    if not env_path.exists():
        return 15.0
    try:
        return float(env_path.read_text().strip())
    except ValueError:
        print(
            f"WARNING: Invalid threshold value in {env_path} (expected a numeric value), defaulting to 15.0",
            file=sys.stderr,
        )
        return 15.0
```

A `.env` file in any realistic project layout contains `KEY=VALUE` lines, not a single numeric value — calling `float()` on the whole file content will always `ValueError` and silently fall back to 15.0. Either:

- The intent is a dedicated `.perf-threshold` file (the name `.env` is the bug), or
- The function should parse `KEY=VALUE` and read a specific key.

As written, the override path is unreachable in any normal environment.

---

### 4. `evaluate_settings_effectiveness.py` lines 141–146: dead `ast.Str` branch

```python
def _extract_literal_str(node: ast.AST) -> str | None:
    if isinstance(node, ast.Constant) and isinstance(node.value, str):
        return node.value
    if isinstance(node, ast.Str) and isinstance(node.s, str):
        return node.s
    return None
```

`ast.Str` and `ast.Str.s` have been deprecated since Python 3.8 (replaced by `ast.Constant`/`ast.Constant.value`) and emit a DeprecationWarning when accessed in Python 3.12+. Because the first branch already handles every string literal that `ast.Str` would have matched on supported Pythons, the `ast.Str` branch is dead code.

---

### 5. `run_real_model.py` line 121: `rf_series.loc[portfolio.index]` raises `KeyError` on date mismatch

```python
rf_series = df_all.get("Risk-Free Rate")
rf_aligned = rf_series.loc[portfolio.index] if rf_series is not None else 0.0
```

`Series.loc[idx]` raises `KeyError` if any label in `idx` is not in the Series — which will fire if the stitched OOS portfolio includes dates outside the loaded RF range. The correct primitive for this kind of "best-effort alignment" is `reindex`, which fills missing labels with NaN:

```python
rf_aligned = rf_series.reindex(portfolio.index) if rf_series is not None else 0.0
```

This is a real crash waiting to happen on any backtest configuration where the OOS window extends beyond the RF series.

---

## Notes

- **`generate_demo.py` line 22: filesystem side effect at import time.** `os.makedirs(OUT_DIR, exist_ok=True)` runs the moment anything imports the module, creating a `demo/` directory under the caller's current working directory. The call should live inside `main()`.

- **`run_multi_demo.py` sys.modules mutation.** Lines 290–349 install fake `ipydatagrid` and `demo_plugin` modules into `sys.modules` and remove them in `finally`. This works in isolation but is fragile if pytest or another import-system consumer caches the missing-module state.

- **`autopilot_metrics_collector.py` lines 108–115: `@dataclass(frozen=True)` on Exception subclass.**
  ```python
  @dataclass(frozen=True)
  class ValidationError(Exception):
      message: str
      def __str__(self) -> str:
          return self.message
  ```
  The dataclass-generated `__init__` shadows `Exception.__init__`, so `self.args` is never populated. The `__str__` override compensates, but anything that introspects `exc.args` (some logging filters, sentry frames, structured loggers) will see an empty tuple. A plain `class ValidationError(Exception): pass` would be safer.

- **`ci_cosmetic_repair.py` line 325: `git push --force`.** Hard force-pushes a freshly created branch. The branch is unique per timestamp so collision is unlikely, but `git push --force-with-lease` would catch the race condition where the same branch already exists upstream with different content.

- **`sync_test_dependencies.py`: `--fix` and `--verify` are not mutually exclusive.** When both flags are passed, `--fix` runs first, returns 0, and the `--verify` branch is unreachable. Argparse `add_mutually_exclusive_group` would make the constraint explicit; the help text doesn't mention the precedence.

- **`run_real_model.py` line 30: no argparse for the config path.**
  ```python
  def main(cfg_path: str = "config/long_backtest.yml") -> int:
  ```
  The CLI invocation `python scripts/run_real_model.py` cannot override the path without editing the source. Compare to `scripts/walkforward_cli.py`, which exposes the same surface via argparse.

- **`streamlit_param_sweep.py` line 71: `path.read_text()` without `encoding`.** Inherits the platform default, which on Windows is cp1252 and will mangle non-ASCII content. Every other read in the file uses `encoding="utf-8"`.

- **`reproduce_ui_run.py` `_norm_cdf`, `_safe_float`, `_parse_period_end`: bare `except Exception`.** Three different helpers swallow all errors and return NaN. This pattern works for genuinely scalar conversions but masks bugs further upstream (e.g. a renamed column that arrives as a Series instead of a scalar would silently become NaN rather than raising).

- **`compare_perf.py` lines 33–38: `sys.path.insert` to load `utils.paths.proj_path`.** The script depends on `src/utils/paths.py`, which the `sys.path.insert` block adds — but only if `src/` exists, with no fallback. Running this script outside the repo or from a wheel install fails with `ModuleNotFoundError`.

- **`generate_demo.py` lines 14–17: numpy import fallback into `_FallbackRng`.** The script supports running without numpy by falling back to the stdlib `random` module, but every numpy-touching code path then needs `if np is None` branches (e.g. lines 112–115, 164–182). A cleaner approach would be to require numpy (the rest of the codebase does) or push the fallback into a dedicated module.

- **`benchmark_performance.py`: deliberately imports a private helper.** Line 32 imports `from trend_analysis.multi_period.engine import _compute_turnover_state` — a leading-underscore private symbol. A regression in the public API would be invisible to this benchmark; renaming the private function would silently break the benchmark.

- **`test_settings_wiring.py`: 80+ entries in `SETTINGS_TO_TEST` with no test-list ownership.** Adding a new Streamlit setting requires updating this list manually; there's no AST-driven discovery that would prevent drift. The companion `evaluate_settings_effectiveness.py` does AST-parse the model page, but the wiring file maintains its own parallel list.

- **`autopilot_metrics_collector.py` line 587–589: env override only applied to default path.**
  ```python
  log_path = Path(args.path)
  env_log_path = os.environ.get("AUTOPILOT_METRICS_LOG_PATH")
  if env_log_path and args.path == "autopilot-metrics.ndjson":
      log_path = Path(env_log_path)
  ```
  The env override only fires when `--path` retains its default. If the user passes `--path=metrics.ndjson` (a non-default value that happens to differ from the env override), the env var is ignored. A small footgun in workflows that set both.

- **`ledger_migrate_base.py` line 88: stripping `origin/` from `git rev-parse --abbrev-ref` output.**
  ```python
  if rev and rev != "origin/HEAD":
      return rev.split("/", 1)[-1]
  ```
  `rev.split("/", 1)[-1]` is brittle for branch names containing a slash (e.g. `release/2025-q4`). After the split, that branch name becomes `2025-q4` and the wrong base is written into every ledger. `rev.removeprefix("origin/")` would be safer.

- **`run_threshold_churn_demo.py` line 67: hardcoded `start = "2015-01"`.** The script then "extends the multi-period start to 2017-01" in the docstring but the actual code sets it to 2015-01. The docstring and code disagree; the docstring is the older version.

- **`diff_ui_runs.py` lines 83–87: list diffs are coarse.** If any element of a list differs, the entire list is reported as the difference. For long fund-weight lists this produces unhelpful output. Element-by-element diffing or a position-indexed report would surface the actual changed cell.

- **`ci_metrics.py` line 209: timezone-aware timestamp via `_dt.UTC`.** Uses `from collections.abc import Iterable, Sequence` and `import datetime as _dt`, then references `_dt.UTC` at lines 208–210. `_dt.UTC` was added in Python 3.11, so this file silently requires 3.11+ even though `pyproject.toml` may declare a lower floor.
