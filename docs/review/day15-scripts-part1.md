# Day 15 Code Review: scripts/ (Part 1 — Shell, JavaScript, and Utility Python)

**Scope:** Shell scripts, JavaScript runner files, and smaller Python utility scripts inside `scripts/`.
Approximately 30 files, ~3,500 lines. The larger Python scripts (`run_multi_demo.py`,
`evaluate_settings_effectiveness.py`, etc.) and the `scripts/langchain/` subpackage are
deferred to Days 16 and 17.

---

## Files Reviewed

- `scripts/archive_agents.sh` — Scans `agents/codex-*.md`, checks each issue's state via `gh issue view`, and moves closed-issue files to `archives/agents/`. Dry-run by default; pass `--apply` to commit changes.
- `scripts/check_branch.sh` — Comprehensive local pre-merge validation gate: Black, Flake8, mypy, package install, imports, pytest, and coverage. Supports `--verbose`, `--fix`, and `--fast` flags.
- `scripts/codex_git_bootstrap.sh` — Minimal Git bootstrap for Codex: checks for uncommitted changes, fetches from origin, checks out a hardcoded `main` branch, and pulls.
- `scripts/docker_smoke.sh` — Builds the Docker image locally and probes the `/health` endpoint with exponential-backoff retries, mirroring the Gate workflow's Docker smoke job.
- `scripts/fix_common_issues.sh` — Auto-repair script that runs Black and installs mypy stubs; used by the pre-commit hook's auto-fix fallback path.
- `scripts/git_hooks.sh` — Installs, uninstalls, or reports the status of three local Git hooks (`pre-commit`, `pre-push`, `post-commit`).
- `scripts/install_pre_push_style_gate.sh` — Writes a minimal pre-push hook that delegates to `style_gate_local.sh`.
- `scripts/pre-commit-check-deps.sh` — Pre-commit hook that calls `sync_test_dependencies.py --verify` and auto-fixes on failure.
- `scripts/preflight-pr.sh` — Reads live PR state (labels, comments, recent workflow runs) from the GitHub API via `gh`; intended to be run before making any claims about PR state.
- `scripts/quality_gate.sh` — Orchestrates the full local quality gate by calling `style_gate_local.sh`, mypy, optional workflow lint, optional Docker smoke, `validate_fast.sh`, and optionally `check_branch.sh --fast`.
- `scripts/run_tests.sh` — CI-style test runner: creates a venv if absent, installs deps via `uv pip sync`, runs pytest with coverage, and handles the "no tests collected" exit code 5.
- `scripts/style_gate_local.sh` — Mirrors the CI style job exactly: installs pinned Black and Ruff from `autofix-versions.env`, runs both checks, then optionally runs mypy.
- `scripts/validate_fast.sh` — Adaptive validation script that inspects which files changed and selects `incremental`, `comprehensive`, or `full` strategies accordingly; re-validates after auto-fixes.
- `scripts/workflow_lint.sh` — Downloads a pinned `actionlint` binary into `.cache/actionlint/` if absent, then lints all workflow files.
- `scripts/keepalive-runner.js` — Node.js script that drives keepalive dispatch: reads PR state, finds the current Codex instruction, constructs the comment payload, and posts via the Octokit API with multiple runtime fallbacks.
- `scripts/keepalive_instruction_segment.js` — Extracts and measures the keepalive instruction segment from a PR comment body; used by `keepalive-runner.js`.
- `scripts/auto_type_hygiene.py` — Injects `# type: ignore[import-untyped, unused-ignore]` comments on allowlisted import lines; checks for stub packages before touching a file; idempotent and dry-run capable.
- `scripts/build_autofix_pr_comment.py` — Assembles the consolidated autofix PR status comment from `autofix_report_enriched.json`, `ci/autofix/history.json`, and `ci/autofix/trend.json`; outputs markdown with a stable HTML marker for upsert.
- `scripts/classify_ruff.py` — Classifies Ruff JSON diagnostics into "allowed" vs. "new" buckets against a `.ruff-residual-allowlist.json` file; writes a summary JSON and never raises.
- `scripts/classify_test_failures.py` — Parses JUnit XML reports and buckets failing tests as `cosmetic`, `runtime`, or `unknown` by reading `pytest` marker annotations from `user_properties`.
- `scripts/coverage_history_append.py` — Appends a single coverage-trend JSON record to an NDJSON history file; deduplicates by `run_id` and sorts by `run_number` before writing via an atomic rename.
- `scripts/ledger_validate.py` — Validates `.agents/issue-*-ledger.yml` files against a schema (status enum, hex commit SHAs, ISO-8601 timestamps); fetches missing commits from origin when needed.
- `scripts/mypy_autofix.py` — Runs mypy in JSON mode, finds `Name "X" is not defined` diagnostics for well-known `typing` symbols, and injects the missing `from typing import ...` line; idempotent and dry-run capable.
- `scripts/prune_allowlist.py` — Removes Ruff allowlist codes that have had zero occurrences across the last N CI history snapshots; exits 0 always (non-fatal).
- `scripts/sync_tool_versions.py` — Compares pinned tool versions in `autofix-versions.env` against `pyproject.toml` and optionally rewrites the file to bring them into alignment.
- `scripts/trigger_gate_workflow.py` — Triggers `pr-00-gate.yml` for a given PR number via `gh workflow run`; supports `--ensure` (skip if already running for that SHA) and `--status` subcommands.
- `scripts/update_autofix_expectations.py` — Re-evaluates hardcoded test-expectation constants by calling the module's own `compute_expected_*` function and rewriting the constant in-place with `repr()`.
- `scripts/validate_config.py` — Thin CLI wrapper over `trend_analysis.config.schema_validation.validate_config_file`; exits 1 on validation errors.
- `scripts/validate_llm_deps.py` — Checks Python version (≥3.10), Pydantic major version (≥2), and exact LangChain major.minor versions against a hardcoded table at import time.
- `scripts/verify_codex_bootstrap.py` — End-to-end verification harness for Codex bootstrap scenarios; creates real GitHub issues, triggers workflows, sleeps for a fixed interval, downloads artifacts, and reports pass/fail.
- `scripts/walkforward_cli.py` — CLI wrapper over `trend_analysis.engine.walkforward.walk_forward`; validates input CSV, optionally loads a regime CSV, and prints full-period, OOS, and per-window aggregates.
- `scripts/workflow_smoke_tests.py` — Smoke-tests the quarantine TTL validator with a synthetic fixture; minimal integration check that `tools.validate_quarantine_ttl` runs end-to-end.

---

## Issues Found

### 1. `check_branch.sh` line 222: duplicate `then` — bash syntax error

Line 222 has two consecutive `then` keywords, which makes the script fail the bash syntax check and exit before any validation runs.

```bash
# check_branch.sh:222 (broken)
if ! run_validation "Test coverage" "rm -f .coverage .coverage.* && pytest --cov=src --cov-report=term-missing --cov-fail-under=80 --cov-branch $XDIST_FLAG" ""; then then
```

```bash
# fix: remove the extra `then`
if ! run_validation "Test coverage" "rm -f .coverage .coverage.* && pytest --cov=src --cov-report=term-missing --cov-fail-under=80 --cov-branch $XDIST_FLAG" ""; then
```

`bash -n scripts/check_branch.sh` reproduces the failure immediately. The script is called by `quality_gate.sh --full` and by the pre-push hook, so this silently breaks comprehensive local validation for every developer.

---

### 2. `workflow_lint.sh`: actionlint binary installed via curl-pipe-bash

The script fetches actionlint by piping a remote shell script directly into bash, which executes whatever that URL returns at that moment.

```bash
# workflow_lint.sh:16
curl -sSfL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash | bash -s "$VERSION"
```

If the remote URL is compromised or returns different content, arbitrary code runs in the developer's shell. A safer approach is to download the binary directly to a known path and verify it against a hardcoded checksum before executing.

---

### 3. `validate_fast.sh` line 282: unquoted `$PYTHON_FILES` in `for` loop

Files with spaces in their names will be split into multiple tokens by word expansion.

```bash
# validate_fast.sh:282
for file in $PYTHON_FILES; do
    if [[ -f "$file" ]]; then
        if ! python -m py_compile "$file" 2>/dev/null; then
```

```bash
# fix
while IFS= read -r file; do
    [[ -f "$file" ]] || continue
    python -m py_compile "$file" 2>/dev/null || { ... }
done <<< "$PYTHON_FILES"
```

This is the same class of bug noted in Day 14's `tools/pre-commit`. The same unquoted expansion also appears in `run_fast_check` when it constructs `sanitised_files` at lines 203–204, though those paths go through a `tr` pipeline that strips newlines rather than a shell `for` loop.

---

### 4. `fix_common_issues.sh` line 45: `while read` missing `-r` and `IFS=`

```bash
# fix_common_issues.sh:45 (broken)
find src/ tests/ scripts/ -name "*.py" -exec grep -l ".\{120,\}" {} \; | while read file; do
```

Without `IFS=` the leading and trailing whitespace is stripped from each filename. Without `-r` a backslash in a path is consumed as a line-continuation character. The portable form is:

```bash
find src/ tests/ scripts/ -name "*.py" -exec grep -l ".\{120,\}" {} \; | while IFS= read -r file; do
```

---

### 5. `validate_llm_deps.py`: hardcoded major.minor version table will silently fail on upgrades

The `expected_major_minor` dict pins exact minor versions that will become stale the moment any LangChain package ships a minor-version bump.

```python
# validate_llm_deps.py:40-44
expected_major_minor = {
    "langchain": (1, 2),
    "langchain-core": (1, 2),
    "langchain-community": (0, 4),
}
```

When the installed version moves to, say, `(1, 3)`, the check prints an error and exits 1, blocking CI — not because anything is actually broken, but because the table was never updated. A minimum-version floor (`>=`) or a range check would be more resilient, or the expected values should be sourced from `pyproject.toml` rather than duplicated here.

---

## Notes

- **`prune_allowlist.py` lines 91–93: dead code.** The expression `_ = any(...)` computes a boolean about historical allowlist presence but assigns the result to a throwaway variable and never uses it. The comment says "presence in tail already zero; prune regardless," meaning the historical guard was removed but the computation was left behind.

- **`mypy_autofix.py` lines 83–84: `"Required"` appears twice in `TYPING_SYMBOLS`.** A set silently deduplicates it, so behaviour is unaffected, but it indicates a copy-paste that was never caught.

- **`quality_gate.sh` line 31: `HEAD~1` assumed to exist.** The workflow-change detector uses `git diff --name-only HEAD~1 2>/dev/null`. On a branch with a single commit (for example, a fresh Codex bootstrap branch), `HEAD~1` does not exist and git exits non-zero; the `2>/dev/null` masks the error but the diff is then empty, causing workflow-lint to be silently skipped even when the initial commit touches workflow files.

- **`verify_codex_bootstrap.py`: hardcoded sleep durations.** Scenarios use `sleep(8)`, `sleep(10)`, etc. to wait for GitHub Actions to process label events. These values are inherently flaky: too short in a congested runner pool, unnecessarily slow otherwise. Using a polling loop with a timeout (checking for the artifact or PR state) would make the harness both faster and more reliable.

- **`check_branch.sh`: `eval "$command"` for all validation checks.** The `run_validation` function passes command strings through `eval`, which works because all call sites pass hardcoded literals. But the pattern is fragile: if a caller ever passes a string derived from external data (e.g., a file path with shell metacharacters), `eval` will execute it as code. Using an array and `"${cmd[@]}"` for each check would eliminate the ambiguity.

- **`coverage_history_append.py`: silently exits 0 on missing record file.** When `RECORD_PATH` does not exist the script prints a warning to stderr and returns 0, which means the workflow step succeeds and the history file receives no new data. Returning 1 would make missing data visible as a CI failure rather than a silent no-op.

- **`validate_fast.sh`: `ensure_package_version` duplicated from `check_branch.sh`.** Both scripts contain an identical function body for upgrading pinned tools. Extracting it into a shared `scripts/_common.sh` sourced by both would prevent the two copies drifting out of sync.

- **`workflow_lint.sh`: version hardcoded in the script body.** The actionlint version (`1.6.27`) lives inside the script rather than in `autofix-versions.env` alongside the other tool pins. It should be managed in the same place to keep all pinned versions visible and updateable in one file.

- **`keepalive-runner.js`: multi-layer Octokit fallback in `buildOctokitInstance`.** The function tries `github.getOctokit`, then `github.constructor`, then `@actions/github`, then `@octokit/rest`, silently swallowing errors between each attempt. While the defensive intent is clear, this layering makes failures very hard to diagnose — a partially-broken `@actions/github` would fall through to `@octokit/rest` and potentially use a different API client than intended.
