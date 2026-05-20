# Day 15 Code Review (Take 2): scripts/ — Shell scripts, JS, small Python utilities

**Scope:** All shell scripts under `scripts/`, both JavaScript runner files, and the smaller
Python utilities (roughly <250 lines each). Total: about 50 files, ~5,000 lines. The larger
Python scripts (`run_multi_demo.py`, `evaluate_settings_effectiveness.py`,
`autopilot_metrics_collector.py`, etc.) and the entire `scripts/langchain/` subpackage are
deferred to Days 16 and 17.

---

## Files Reviewed

### Shell scripts

- `scripts/archive_agents.sh` — Reads each `agents/codex-*.md` file, queries `gh issue view` for the issue's state, and either keeps the file (open) or moves it to `archives/agents/<date>-codex-<num>.md` (closed/missing). Dry-run by default.
- `scripts/check_branch.sh` — Full local validation gate (Black, Flake8, mypy, pytest, coverage, install/import sanity). Reinstalls pinned tool versions if they drift from `autofix-versions.env`. Accepts `--verbose`, `--fix`, `--fast`.
- `scripts/check_test_dependencies.sh` — Prints a colorised table of required Python/Node packages, Python ≥3.11 check, and the availability of `uv`, `npm`, and `coverage`. Returns 1 if any required package is missing.
- `scripts/codex_git_bootstrap.sh` — Six-step "stash-or-die, fetch, checkout main, pull" bootstrap. Hardcodes `main` as the branch name.
- `scripts/dev_check.sh` — Ultra-fast 6-step quality check (syntax, import, Black, critical flake8, light mypy, optional keepalive Node tests). Reads `--changed` to restrict the file scope and supports `--fix` to auto-repair formatting.
- `scripts/diagnose_api_usage.sh` — `gh api rate_limit` wrapper that pretty-prints Core / Search / GraphQL rate-limit utilisation, with a 5-level health classification computed via `bc`.
- `scripts/docker_smoke.sh` — Builds the image locally and probes `/health`, looping up to 5 times with a 2-second sleep and dumping container logs on failure. Skips gracefully if `docker info` does not work.
- `scripts/fix_common_issues.sh` — Auto-repair script: Black, mypy stub install, scan for long lines. Called by the installed pre-commit hook on failure.
- `scripts/git_hooks.sh` — Generates `pre-commit`, `pre-push`, and `post-commit` hook scripts in `.git/hooks/`, with `install`/`uninstall`/`status` subcommands.
- `scripts/install_pre_push_style_gate.sh` — Writes a minimal pre-push hook that delegates to `style_gate_local.sh`.
- `scripts/open_pr_from_issue.sh` — Opens a PR for a Codex-bootstrapped branch using the issue title and body as the PR template; inserts a `@codex plan-and-execute` instruction snippet.
- `scripts/pre-commit-check-deps.sh` — Pre-commit hook that runs `sync_test_dependencies.py --verify` and auto-fixes when test dependencies drift.
- `scripts/preflight-pr.sh` — Reads PR state, labels, comments, and recent workflow runs for a given PR number; intended to be run before making claims about PR state.
- `scripts/quality_gate.sh` — Orchestrator combining `style_gate_local.sh`, mypy, optional workflow-lint, optional docker-smoke, `validate_fast.sh`, and (with `--full`) `check_branch.sh --fast`.
- `scripts/quick_check.sh` — Lightweight formatting + flake8 + import check on the last 5 changed `.py` files.
- `scripts/run_streamlit.sh` — Bootstraps a venv if missing, ensures `streamlit` is installed, optionally maps a Codespaces secret into `OPENAI_API_KEY`, writes a chmod-600 secrets file, then `exec`s `streamlit run`.
- `scripts/run_tests.sh` — CI test runner: builds venv, `uv pip sync requirements.lock`, runs pytest with a configurable coverage profile, handles pytest exit code 5 ("no tests collected").
- `scripts/setup_env.sh` — Sources-or-runs detector that creates `.venv`, installs deps, installs pre-commit hooks, and `chmod +x`'s the `trend` / `trend-model` CLI shims. Validates Node.js ≥20.
- `scripts/setup_streamlit_secrets.sh` — Prompts for `OPENAI_API_KEY` if not in env and writes a chmod-600 `.streamlit/secrets.toml`.
- `scripts/style_gate_local.sh` — Mirror of the CI style job: installs pinned Black and Ruff, runs both, then optionally runs mypy with `types-requests` extracted from `pyproject.toml`.
- `scripts/test-release.sh` — Local release dry-run: cleans `dist/`, updates the version in `pyproject.toml`, builds, runs `twine check`, installs both wheel and sdist into a throwaway venv, runs CLI smoke tests.
- `scripts/test_docker.sh` — Pulls the published `ghcr.io/stranske/trend-model:latest` image and runs five smoke tests (pull, startup, `/health`, CLI help, Python imports).
- `scripts/test_health_retry.sh` — Spawns `trend_portfolio_app.health_wrapper` in the background and verifies the retry logic responds within 10 seconds; cleans up the background PID on exit.
- `scripts/validate_fast.sh` — Adaptive validation that picks an `incremental`, `comprehensive`, or `full` strategy based on file-count and file-type changes detected via `git diff`.
- `scripts/verify_ci_stack.sh` — Sequential Black/Ruff/mypy/pytest runner; with `--docker`, also builds the image and probes `/health` 10 times.
- `scripts/workflow_lint.sh` — Downloads pinned `actionlint` v1.6.27 into `.cache/actionlint/` via a remote bash installer, then lints all workflow files.

### JavaScript

- `scripts/keepalive-runner.js` — Orchestrator for the keepalive PR dispatch. Reads PR body/comments, extracts the Scope/Tasks/Acceptance block, builds the instruction payload, and posts via Octokit with multiple library-detection fallbacks.
- `scripts/keepalive_instruction_segment.js` — Helper that extracts the keepalive instruction text from a comment body and measures its byte length.

### Python utilities (under ~250 lines)

- `scripts/__init__.py` — Marker file.
- `scripts/auto_type_hygiene.py` — Injects `# type: ignore[import-untyped, unused-ignore]` on import lines for allowlisted untyped modules; idempotent.
- `scripts/build_autofix_pr_comment.py` — Builds the upsert-able PR status comment from autofix JSON artifacts with a stable HTML marker.
- `scripts/check_config_coverage.py` — Activates the `ConfigCoverageTracker`, runs a full simulation, then enforces both an ignored-keys threshold and a minimum schema-validity ratio.
- `scripts/ci_coverage_delta.py` — Parses `coverage.xml`, compares against an environment-supplied baseline, and writes a structured JSON delta. Exits 1 only when `FAIL_ON_DROP=true` and the drop ≥ `ALERT_DROP`.
- `scripts/ci_feature_assert.py` — Asserts that workflow-side feature artifacts (metrics, history, classification, coverage delta) are present when the corresponding `EXPECT_*` flag is true and absent otherwise.
- `scripts/ci_history.py` — Appends a metrics record to `metrics-history.ndjson` and optionally writes `classification.json` for failure analysis.
- `scripts/classify_ruff.py` — Buckets Ruff diagnostics into "allowed" vs. "new" using `.ruff-residual-allowlist.json`.
- `scripts/classify_test_failures.py` — Parses JUnit XML; uses pytest `user_properties` markers to bucket failures as cosmetic / runtime / unknown.
- `scripts/coverage_history_append.py` — Appends a coverage-trend record to an NDJSON history file with idempotent run_id replacement.
- `scripts/docker_health_response.py` — Reads `HEALTH_RESPONSE` env var, normalises and checks against an allowlist of "success" strings; recursively inspects JSON for nested success markers.
- `scripts/fix_cosmetic_aggregate.py` — Calls a single trend-analysis API to apply cosmetic fixes; thin wrapper.
- `scripts/fix_numpy_asserts.py` — Rewrites `assert <array_name> == [...]` to `assert <array_name>.tolist() == [...]` for four specific test files.
- `scripts/generate_config_schema.py` — Calls `trend_analysis.config.schema_generator.write_schema_files`.
- `scripts/inventory_pipeline_patch_tests.py` — AST scan of test files: enumerates places where pipeline functions are monkeypatched, distinguishing pipeline aliases from package aliases.
- `scripts/ledger_validate.py` — YAML ledger schema check (status enum, hex SHA shapes, ISO-8601 timestamps).
- `scripts/merge_autofix_report.py` — Merges autofix metadata (PR number, timestamp) into the enriched JSON payload.
- `scripts/mypy_autofix.py` — Runs mypy in JSON mode, finds "Name X is not defined" for `typing` symbols, injects the missing `from typing import` line.
- `scripts/prune_allowlist.py` — Removes Ruff allowlist codes that have had zero occurrences across the last N CI history snapshots.
- `scripts/render_mypy_summary.py` — Reads mypy JSON output and writes a plain-text summary; outputs "Success: no mypy issues detected." when there are no diagnostics.
- `scripts/render_security_policy.py` — Emits a hardcoded markdown SECURITY.md document.
- `scripts/sync_tool_versions.py` — Reads `autofix-versions.env`, compares against `pyproject.toml`, optionally rewrites `pyproject.toml` to align.
- `scripts/trigger_gate_workflow.py` — Triggers `pr-00-gate.yml` for a given PR, with `--ensure` and `--status` subcommands.
- `scripts/update_autofix_expectations.py` — Re-runs each test module's `compute_expected_*` function and rewrites the matching constant via regex.
- `scripts/validate_config.py` — Thin CLI wrapper over `trend_analysis.config.schema_validation.validate_config_file`.
- `scripts/validate_llm_deps.py` — Verifies Python ≥3.10, Pydantic ≥2.0, and LangChain at exact major.minor versions.
- `scripts/validate_strategy_pack.py` — Thin CLI wrapper over `trend_analysis.monte_carlo.strategy.validation.validate_strategy_pack`.
- `scripts/walk_forward.py` — CLI entry point for `trend_analysis.walk_forward.run_from_config`; minimal argparse wrapper.
- `scripts/walkforward_cli.py` — Older parallel CLI for `trend_analysis.engine.walkforward.walk_forward`; supports regime CSVs.
- `scripts/workflow_smoke_tests.py` — Smoke-test that exercises `tools.validate_quarantine_ttl` end-to-end with a synthetic fixture.

---

## Issues Found

### 1. `check_branch.sh` line 222: duplicate `then` makes the script fail bash syntax validation

```bash
if ! run_validation "Test coverage" "rm -f .coverage .coverage.* && pytest --cov=src --cov-report=term-missing --cov-fail-under=80 --cov-branch $XDIST_FLAG" ""; then then
```

`bash -n scripts/check_branch.sh` reports:

```
syntax error near unexpected token `then'
```

The script is called by `quality_gate.sh --full` and by the pre-push hook generated by `git_hooks.sh`. Both invocations now exit immediately with a parse error before doing anything. Fix: delete the extra `then`.

---

### 2. `test-release.sh` line 36: `sed -i .bak` is GNU/BSD-incompatible

```bash
sed -i .bak "s/version = \".*\"/version = \"${VERSION}\"/" pyproject.toml
```

On macOS (BSD sed) the syntax requires `-i` followed by a separate backup extension argument, so this works. On Linux (GNU sed) the same line is interpreted as `-i` (in-place, no backup) plus a separate input file path `.bak`, which does not exist, so the script aborts before it edits `pyproject.toml`. The conventional GNU form is `sed -i.bak ...` (no space). For cross-platform portability, either drop the backup entirely and rely on git, or use a Python one-liner.

---

### 3. `quick_check.sh` lines 22–26: `$?` after a command-substitution assignment is always 0

```bash
CHANGED_FILES=$(git diff --name-only HEAD~1 2>/dev/null | grep -E '\.(py)$' 2>/dev/null | ... | head -5)
if [[ $? -ne 0 ]]; then
    echo "::warning::git diff command failed..."
    CHANGED_FILES=""
fi
```

In bash, the exit status of an assignment statement is the status of the most recent command substitution executed, but with a pipeline the relevant status is the *last* command in the pipe (`head -5` here) — which essentially always succeeds. Without `set -o pipefail`, the first git command failing in the chain is silently masked. The check inside the `if` therefore never fires, and the warning message is unreachable code.

A correct pattern is to test the substitution result directly:

```bash
if ! CHANGED_FILES=$(git diff --name-only HEAD~1 2>/dev/null | ... | head -5); then
    ...
fi
```

or enable `pipefail` before the assignment and inspect `$?` from a stored copy.

---

### 4. `open_pr_from_issue.sh` lines 53 and 59: escaped backticks emit literal backslashes in the PR body

```bash
echo '\`\`\`markdown'
echo '@codex plan-and-execute'
echo
echo 'Codex, reuse the scope, acceptance criteria, and task list from the source issue.'
...
echo '\`\`\`'
```

Inside single quotes, `\` is a literal backslash and `` ` `` is a literal backtick. So the strings produced are `` \`\`\`markdown `` and `` \`\`\` `` — backslashes included. GitHub renders those as escaped characters, not a code fence, so the `@codex plan-and-execute` block in the auto-opened PR body will not be a fenced code block. The fix is just plain single-quoted backticks: `echo '```markdown'` and `echo '```'`.

---

### 5. `validate_llm_deps.py` lines 40–44: exact-pin major.minor check breaks on every LangChain release

```python
expected_major_minor = {
    "langchain": (1, 2),
    "langchain-core": (1, 2),
    "langchain-community": (0, 4),
}
```

Any minor version drift (e.g. installed `langchain==1.3.0`) trips the "incompatible" branch and exits 1, even though the installed version is presumably the new floor. The check is essentially "pin twice — once in `pyproject.toml`, once here" and goes stale unilaterally. A floor check (`>=1.2`) plus a pyproject-sourced upper bound would be more resilient.

---

## Notes

- **`check_branch.sh` and `validate_fast.sh` duplicate `ensure_package_version`.** The same function body is copy-pasted in both scripts (and `dev_check.sh` has a third copy). Extracting a shared `scripts/_lib.sh` would eliminate the drift risk.

- **`validate_fast.sh` line 282: unquoted `$PYTHON_FILES` in a `for` loop.** Same word-splitting pattern flagged in Day 14's `tools/pre-commit` — filenames with spaces will be split into multiple tokens. The fix is `while IFS= read -r file; do ... done <<< "$PYTHON_FILES"`.

- **`fix_common_issues.sh` line 45: `while read file` without `-r` or `IFS=`.** Strips leading whitespace and treats backslashes as line continuations. Less likely to hurt anything in this codebase, but a portability footgun.

- **`workflow_lint.sh` line 16: curl-pipe-bash for actionlint.** `curl … | bash -s "$VERSION"` executes whatever that URL returns at the moment of execution. Downloading the binary directly to `.cache/actionlint/` and verifying a sha256 would close the loop.

- **`workflow_lint.sh` line 11: actionlint version is hardcoded in the script body**, separate from the other tool pins in `.github/workflows/autofix-versions.env`. Move it there for a single source of truth.

- **`test_docker.sh` line 7: hardcoded `ghcr.io/stranske/trend-model:latest`.** This repo is `iamkayleb/Trend_Model_Project`. If the image is also republished under the iamkayleb namespace, this script needs updating; otherwise it tests a stale upstream image.

- **`prune_allowlist.py` lines 91–94: dead `_ = any(...)` expression.** The result of `any(...)` is assigned to a throwaway variable and never used. The accompanying comment ("presence in tail already zero; prune regardless") indicates the historical guard was removed but the computation was left in place.

- **`mypy_autofix.py` lines 83–84: duplicate `"Required"` entry in `TYPING_SYMBOLS`.** Behaviour is unaffected (sets deduplicate), but it is a copy-paste error.

- **`merge_autofix_report.py` line 36: `datetime.now(timezone.utc)` evaluated at argument-parse time.** This is correct (the default is computed each time `parse_args()` runs), but it is worth noting because the same pattern would silently freeze the value if the script were ever imported as a long-running library.

- **`coverage_history_append.py` line 43: silent exit 0 on missing record.** The script prints a warning to stderr and returns 0 when `RECORD_PATH` does not exist; this makes the workflow step succeed even if no coverage data was produced.

- **`verify_codex_bootstrap.py` scenario sleeps.** Hardcoded `sleep(8)`, `sleep(10)` to wait for GitHub Actions are inherently flaky under load — too short in a busy runner pool, unnecessarily slow otherwise. A polling loop with a timeout would be more reliable.

- **`run_streamlit.sh` lines 24–38: ceremonial path-traversal "validation".** The check compares `realpath("$ROOT_DIR/streamlit_app/app.py")` against itself. There is no user-supplied input being validated — the path is hardcoded from the script's own location. The check provides no real security benefit but adds 15 lines of complexity.

- **`quality_gate.sh` line 31: assumes `HEAD~1` exists.** On a single-commit branch (e.g. a fresh Codex bootstrap), `git diff --name-only HEAD~1` errors out and the workflow-lint step is silently skipped even when the initial commit touches workflow files.

- **`setup_env.sh` lines 44–47, 49–60: tabs and spaces are mixed in the indentation.** Cosmetic, but a `cat -A` reveals visible drift between sibling commands.

- **`keepalive-runner.js`: layered Octokit fallback in `buildOctokitInstance`.** Four attempts (`github.getOctokit`, `github.constructor`, `@actions/github`, `@octokit/rest`), each swallowing exceptions to fall through to the next. The defensive intent is clear but masks the root cause when failures occur.
