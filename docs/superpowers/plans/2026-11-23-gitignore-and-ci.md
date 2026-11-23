# Gitignore and CI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ensure generated Python and test artifacts are ignored and add GitHub Actions validation for every push and pull request targeting `main`.

**Architecture:** Extend the existing repository-level `.gitignore` with conventional Python coverage cache entries, then add one focused GitHub Actions workflow that synchronizes the locked environment and runs the project's existing quality commands.

**Tech Stack:** Git, GitHub Actions, uv, Ruff, BasedPyright, pytest

**Spec:** User-approved short design in the 2026-11-23 conversation: add/complete `.gitignore`, add CI, and run Ruff formatting/linting, BasedPyright, and pytest on `main`.

## Global Constraints

- Python floor: `>=3.12`
- CI must use the committed `uv.lock`
- CI must remain on pull requests and pushes to `main`
- Do not ignore source, configuration, or dependency lock files
- Preserve the existing `.env.example` allowlist

## Review Focus

- Generated cache directories must be ignored: verify representative `.venv`, `.pytest_cache`, `__pycache__`, and `.ruff_cache` paths with `git check-ignore`.
- Environment files must not leak: verify `.env` is ignored and `.env.example` remains trackable.
- Coverage files must not leak: verify `.coverage`, `.coverage.*`, and `coverage.xml` are ignored.
- CI lockfile drift must fail: run `uv sync --locked --all-groups` so CI uses only the committed resolution.
- CI quality commands must match project configuration: run `ruff format --check .`, `ruff check .`, `basedpyright`, and `pytest`.

---

### Task 1: Complete Repository Ignore Rules

**Files:**
- Modify: `.gitignore`
- Test: none

**Interfaces:**
- Consumes: existing repository ignore patterns and generated files in the workspace
- Produces: ignore rules that cover Python bytecode, virtual environments, tooling caches, environment files, coverage output, and common IDE/OS files

- [ ] **Step 1: Extend coverage and local-tooling ignore entries**

Add the following entries under the existing appropriate headings in `.gitignore`:

- `.coverage`
- `.coverage.*`
- `coverage.xml`
- `.mypy_cache/` (if present in the file or project)
- `.hypothesis/`
- `*.pyc`
- `*.pyo`

Keep existing `.venv/`, `.pytest_cache/`, `.ruff_cache/`, `.env`, `.env.*`, IDE, and OS entries unchanged except for the existing `.env.example` exception.

- [ ] **Step 2: Verify ignore behavior**

Run:

```bash
git check-ignore -v .venv/placeholder .pytest_cache/placeholder app/__pycache__/main.pyc .ruff_cache/placeholder .coverage .coverage.foo coverage.xml .env .idea/placeholder .DS_Store
git check-ignore -v --no-index .env.example
```

Expected: the first command reports matching ignore patterns. The final `.env.example` command must produce no output and exit successfully.

### Task 2: Add GitHub Actions CI

**Files:**
- Create: `.github/workflows/ci.yml`
- Test: none

**Interfaces:**
- Consumes: `uv.lock`, `pyproject.toml`, Python 3.12
- Produces: a `ci` workflow that runs formatting, linting, type checking, and tests

- [ ] **Step 1: Create the workflow**

Create `.github/workflows/ci.yml` with:

- workflow name `CI`
- push trigger for `main`
- pull request trigger for `main`
- one `ci` job on `ubuntu-latest`
- Python setup for `3.12`
- `astral-sh/setup-uv` with caching enabled
- `uv sync --locked --all-groups`
- `uv run ruff format --check .`
- `uv run ruff check .`
- `uv run basedpyright`
- `uv run pytest`

Use `astral-sh/setup-uv@v6` and `actions/setup-python@v6`.

- [ ] **Step 2: Validate workflow YAML**

Run:

```bash
python - <<'PY'
import yaml

with open(".github/workflows/ci.yml", "r", encoding="utf-8") as workflow:
    data = yaml.safe_load(workflow)

assert data["name"] == "CI"
assert data["on"]["push"]["branches"] == ["main"]
assert data["on"]["pull_request"]["branches"] == ["main"]
PY
```

Expected: the workflow parses and both triggers are pinned to `main`.

- [ ] **Step 3: Run the complete local CI sequence**

Run:

```bash
uv sync --locked --all-groups
uv run ruff format --check .
uv run ruff check .
uv run basedpyright
uv run pytest
```

Expected: dependency sync succeeds, all format/lint/type checks pass, and pytest reports zero failures.

- [ ] **Step 4: Review the final diff**

Run:

```bash
git --no-pager diff --stat
git --no-pager diff -- .gitignore .github/workflows/ci.yml
git --no-pager status --short
```

Expected: only `.gitignore` and `.github/workflows/ci.yml` are added or modified for this task, apart from any pre-existing dependency changes.
