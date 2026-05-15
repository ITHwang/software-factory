# Python Toolchain Setup

> Last updated: 2026-05-14

## TL;DR

How to wire ruff (lint + format), mypy (typecheck), pre-commit (git hooks), and GitHub Actions (CI) into a uv project. Pyright (default) or Pyrefly (alternative when pyright IDE perf is a bottleneck) are editor-only and intentionally outside this stack.

**Use this when:**
- adding linting/formatting/typecheck/CI to a new Python project
- tightening an existing project's quality bar
- standardizing tool configs across the repo

**Don't use this for:**
- installing the Python runtime / setting up `uv` itself → `./py-setup-environments.md`
- language-level conventions (version, framework choices) → `./python-guidelines.md`
- API test patterns → `./python-tests.md`

## Table of Contents

| Section |
|---------|
| [Setup](#setup) |
| [Stack At A Glance](#stack-at-a-glance) |
| [Project Structure](#project-structure) |
| [Tool Config (pyproject.toml)](#tool-config-pyprojecttoml) |
| [Tools](#tools) |
| [Best Practices](#best-practices) |

## Setup

Run these in order from a freshly-initialized uv project (see `setup-environments.md`).

```bash
# 1. Install tooling into the project
uv add ruff mypy pre-commit

# 2. Write the configs:
#    - pyproject.toml             (Tool Config section)
#    - .pre-commit-config.yaml    (Pre-commit section)
#    - .github/workflows/main.yml (GitHub Actions CI section)

# 3. Install the git hook
pre-commit install

# 4. Run all hooks once over the whole repo
pre-commit run --all-files
```

`pyright` is intentionally **not** in this dependency list — it's editor-only. See [Pyright (Editor Only)](#pyright-editor-only). When pyright IDE responsiveness is a bottleneck, `pyrefly` is the editor-only alternative — see [Pyrefly (Editor Alternative)](#pyrefly-editor-alternative).

## Stack At A Glance

| Tool            | Role                                 | When to run                  |
|-----------------|--------------------------------------|------------------------------|
| `pyright`       | Editor type-check, hover, go-to-def  | In editor                    |
| `pyrefly`       | Editor type-check (alt. when pyright is slow) | In editor             |
| `ruff`          | Lint, format, and import sorting     | In pre-commit                |
| `mypy`          | Static type checking (canonical)     | In pre-commit                |
| `pre-commit`    | Hook orchestrator                    | Local, CI                    |
| GitHub Actions  | CI gate (runs workflow, uploads checks) | PR, main branch           |

## Project Structure

Keep tool settings in `pyproject.toml`, pre-commit hooks in `.pre-commit-config.yaml`, and CI workflows in `.github/workflows/`.

```text
project/
├── pyproject.toml
├── .pre-commit-config.yaml
└── .github/
    └── workflows/
        └── main.yml
```

## Tool Config (pyproject.toml)

`pyproject.toml` is the single source of truth for tool config. Do not split into `mypy.ini`, `.ruff.toml`, `setup.cfg`, or `pyrightconfig.json`.

```toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
extend-select = [
  "E",   # pycodestyle errors
  "F",   # Pyflakes (unused/undefined/import issues)
  "I",   # isort-compatible import sorting
  "UP",  # pyupgrade (modern Python syntax)
  "B",   # flake8-bugbear (bug-prone patterns)
  "SIM", # flake8-simplify (simpler equivalent constructs)
  "C4",  # flake8-comprehensions (better comprehension usage)
  "S",   # security-focused checks (Bandit-compatible family)
]

[tool.mypy]
# Intentionally minimal. mypy infers Python version from `requires-python`.
# Tighten per project as the team commits to fixing the resulting errors:
# add `strict = true`, `disallow_untyped_defs = true`, `warn_return_any = true`, etc.
warn_unused_ignores = true

[tool.pyright]
exclude = [
  "**/.git/**",
  "**/.venv/**",
  "**/node_modules/**",
  "**/.mypy_cache/**",
  "**/.ruff_cache/**",
  "**/.pytest_cache/**",
  "**/build/**",
  "**/dist/**",
  "**/__pycache__/**",
]
```

Pyrefly users can opt-in per-developer by running `pyrefly init`, which reads the existing `[tool.pyright]` + `[tool.mypy]` blocks and writes a `[tool.pyrefly]` block alongside. The pyright block above stays canonical; the pyrefly block is generated, not maintained by hand.

## Tools

### Ruff

```bash
ruff check                  # lint
ruff check --fix            # lint + apply safe fixes
ruff format                 # format
ruff format --check         # CI check (no writes)
```

Run order in pre-commit and CI: `ruff check` → `ruff format --check`.

Ruff infers `target-version` from `[project] requires-python` in `pyproject.toml`. If `requires-python` is unset, ruff falls back to its default and `UP` (pyupgrade) rewrites to the oldest supported syntax. Always set `requires-python` (covered in `setup-environments.md`).

### Mypy

- **Prefer `cast()` for provable narrowing** — it states intent without silencing unrelated checks.
- **Use `# type: ignore[code]` only when necessary** — always include the specific error code; never use bare `# type: ignore`.
- **Delete stale suppressions quickly** — remove casts/ignores when refactors make them unnecessary.

```yaml
additional_dependencies:
  - types-requests==2.32.4.20260107
  - types-PyYAML==6.0.12.20250915
```

### Pyright (Editor Only)

Pyright is **not** a project dependency. It runs only in your editor; the project's lockfile and CI never see it. Install it through your editor:

| Editor | How pyright runs |
|--------|------------------|
| VS Code | Install the [Pylance](https://marketplace.visualstudio.com/items?itemName=ms-python.vscode-pylance) extension. Pylance bundles pyright. |
| Cursor | Pylance is bundled by default. No install needed. |
| Other (Neovim, Helix, Zed) | Install pyright globally (e.g. `npm i -g pyright`) and point your LSP client at it. |

Pyright config still lives inside `[tool.pyright]` in `pyproject.toml` (no separate `pyrightconfig.json`) — your editor reads it from there.

Rules:

- **Never add pyright to CI or pre-commit.** It is an editor-side aid, not a gate.
- If a pyright squiggle disagrees with mypy and mypy is silent, suppress the pyright warning with `# pyright: ignore[reportX]`. Do not change code to please pyright at the cost of mypy clarity.

### Pyrefly (Editor Alternative)

[Pyrefly](https://pyrefly.org/) is Meta's Rust-based type checker (1.0 shipped 2026-05-12; default at Instagram; adopted by PyTorch, JAX, NumPy, Pandas). Keep pyright as the editor default. Reach for Pyrefly only when pyright IDE responsiveness becomes a bottleneck on this codebase — slow rechecks on a large code tree, high RAM, or perceptible save-to-squiggle lag.

Performance signal: pyrefly checks PyTorch in ~2.4 s vs pyright's ~35.2 s; IDE p99 rechecks drop from minutes to seconds on Meta-scale code.

Trade-offs to know before switching:

- **Typing-spec conformance is ~90 %** vs pyright's higher score — a small set of spec edge cases pyright would flag may slip through.
- **Pylance + Pyrefly integration via the new Type Server Protocol is still early-preview at 1.0** (VS Code Insiders only). Adopting Pyrefly in VS Code today means installing the Pyrefly extension and disabling Pylance's typecheck (or removing Pylance), giving up the rest of Pylance's bundled features.

Install — per-developer, dev-only, never in the lockfile that ships to CI:

```bash
uv add --dev pyrefly        # editor-side typechecker
pyrefly init                # reads existing [tool.pyright] + [tool.mypy]
```

Rules:

- **Never add Pyrefly to CI or pre-commit.** Same policy as pyright. Mypy is the canonical CI gate.
- When swapping to Pyrefly in VS Code, install the Pyrefly extension and disable Pylance's typecheck so only one checker emits diagnostics — otherwise you get duplicate squiggles from two checkers disagreeing on conformance edge cases.
- If pyrefly disagrees with mypy and mypy is silent, prefer mypy's verdict — same rule as for pyright. Do not change code to please the editor checker at the cost of mypy clarity.

### Pre-commit

Drop-in `.pre-commit-config.yaml`. Pin tool versions to the current latest at project init; bump quarterly via `pre-commit autoupdate`.

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: check-added-large-files
      - id: check-merge-conflict
      - id: detect-private-key
      - id: end-of-file-fixer
      - id: trailing-whitespace

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.12
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  - repo: https://github.com/sirosen/check-jsonschema
    rev: 0.37.2
    hooks:
      - id: check-github-actions
      - id: check-github-workflows

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.20.2
    hooks:
      - id: mypy
        additional_dependencies:
          - types-requests==2.32.4.20260107
          - types-PyYAML==6.0.12.20250915

  - repo: https://github.com/pryorda/dockerfilelint-precommit-hooks
    rev: v0.1.0
    hooks:
      - id: dockerfilelint
```

Notes:

- The config file lives at the repo root as `.pre-commit-config.yaml`.
- `additional_dependencies` for mypy must list every typed-stub lib it imports — the hook runs in an isolated env, separate from `.venv`.
- `--exit-non-zero-on-fix` on the ruff hook ensures CI fails when the hook had to change files. Without it, `pre-commit run --all-files` mutates the tree silently and exits 0, hiding unfixed code behind a green check.

Install once per clone:

```bash
pre-commit install
pre-commit run --all-files
```

### GitHub Actions CI

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.14"
      - name: Install uv
        uses: astral-sh/setup-uv@v7
        with:
          python-version: "3.14"
          enable-cache: true
          cache-dependency-glob: |
            pyproject.toml
            uv.lock
      - name: Install pre-commit
        run: uv pip install --system pre-commit
      - name: Run all hooks via pre-commit
        run: pre-commit run --all-files --config .pre-commit-config.yaml
```

## Best Practices

### Multi-Package Repos

For repos with multiple installable Python packages:

- Keep each package self-contained: each package owns its `pyproject.toml` with `[tool.ruff]`, `[tool.mypy]`, and `[tool.pyright]`.
- Keep one pre-commit policy per package via `.pre-commit-config.yaml`.
- A sub-package may be **stricter** than the root baseline, but never **looser**.
- Use one Python runtime version across packages (managed at the repo root via `mise.toml`).

### Stub Packages & Untyped Libraries

When mypy reports `error: Library stubs not installed for "X"`:

1. **Install a stub package first** (preferred): `pip install types-X` (PEP 561).
2. **If no stub exists, use a scoped fallback** in `[[tool.mypy.overrides]]`:

```toml
[[tool.mypy.overrides]]
module = ["some_untyped_lib.*", "another.*"]
ignore_missing_imports = true
```

Guidelines:
- Keep `module` patterns as narrow as possible (avoid broad wildcards).
- If upstream typing or `types-*` stubs become available, remove the override and let mypy check that library normally.

**Important**: keep stub-package pins in one source of truth: `additional_dependencies` for the mypy hook in `.pre-commit-config.yaml` (used by both local and CI).

### Suppression Hygiene

| Tool | Syntax | Required? |
|------|--------|-----------|
| ruff | `# noqa: E501,B008` | Always include codes |
| mypy | `# type: ignore[arg-type]` | Always include codes |
| pyright | `# pyright: ignore[reportArgumentType]` | Editor-only |

Use this order of preference:

```
1. Refactor the code so the warning goes away.
2. Add a one-line comment ABOVE the suppression explaining WHY it is needed.
3. Suppress with the specific code.
```