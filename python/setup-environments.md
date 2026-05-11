# Python Environment Setup

> Last updated: 2026-05-11

## TL;DR

How to set up a Python development environment on macOS or Ubuntu using a layered tool stack: `brew`/`apt` for system packages, `mise` for the Python runtime, and `uv` for project dependencies.

**Use this when:**
- setting up a fresh development machine
- onboarding a new repo and the Python runtime / `uv` aren't installed
- standardizing tooling across macOS and Ubuntu contributors

**Don't use this for:**
- linter / formatter / typechecker / CI setup → `./setup-toolchains.md`
- repo-wide Python conventions (version, frameworks) → `./python-guidelines.md`

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Tooling Layers](#tooling-layers), [Layer Boundaries](#layer-boundaries), [Responsibilities](#responsibilities) |
| 2. Machine setup | [Install Tools](#install-tools) |
| 3. Project setup | [Set Up A Project](#set-up-a-project) |
| 4. Others | [Multi-Package Repos](#multi-package-repos) |

## Tooling Layers

```text
┌─────────────────────────────────────────────────────┐
│ Project / dependencies   →   uv                     │   OS-independent
├─────────────────────────────────────────────────────┤
│ Runtime version          →   mise                   │   OS-independent
├──────────────────────────┬──────────────────────────┤
│ System packages          │   brew      |   apt      │   OS-specific
├──────────────────────────┼──────────────────────────┤
│ OS                       │   macOS     |   Ubuntu   │
└──────────────────────────┴──────────────────────────┘
```

Read top-down for what each layer installs. Everything above the split is identical on every OS; only the bottom two layers change with the operating system.

| Layer | Owner | Owns |
|-------|-------|------|
| OS | macOS or Ubuntu | Kernel, shell |
| System packages | `brew` (macOS) or `apt` (Ubuntu) | Compilers and system libraries (e.g. `curl`, `git`, `build-essential`) |
| Runtime version | `mise` | Per-project Python (and Node, Go, Rust) versions |
| Project / dependencies | `uv` | `pyproject.toml`, virtualenv, lockfile, package install, run |

### Layer Boundaries

Keep each responsibility on the layer that owns it:

- Do not use `brew` or `apt` to install project Python packages.
- Do not use `uv` to manage the Python interpreter version — that is `mise`'s job.
- Do not use `mise` to manage Python project dependencies — that is `uv`'s job.
- Do not use `mise` to load environment variables. `mise` manages tool versions only. Load env vars from whichever source fits the variable: project values from a gitignored `.env` (read by your app or a loader), per-shell values from your shell rc (`~/.bashrc` / `~/.zshrc`), and secrets from a secret manager.

## Responsibilities

| Tool | Owns |
|------|------|
| `mise` | Python and other tool versions |
| `uv` | Virtual environments, dependency installation, lockfiles, packaging workflow |
| `pyproject.toml` | Python package metadata and tool configuration |

Keep all three aligned:

- `mise.toml` pins Python to `3.14`.
- `pyproject.toml` sets `requires-python = ">=3.14,<3.15"`.
- `.venv` is local, ignored by git, and recreated when needed.

## Install Tools

Run once per machine. Both installers are official shell scripts and work the same on macOS and Ubuntu.

Do not use `brew` or `apt` for `mise` or `uv` — use each tool's official shell installer so you get the native installation path and behavior the tool authors support.

### mise

On Ubuntu, make sure the system prerequisites are present first:

```bash
sudo apt install -y curl git build-essential
```

Install `mise` (macOS and Ubuntu):

```bash
curl https://mise.run | sh
```

Activate the shim in your shell. Pick the line that matches your shell:

```bash
# bash
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc

# zsh
echo 'eval "$(~/.local/bin/mise activate zsh)"' >> ~/.zshrc
```

Open a new shell, then verify:

```bash
~/.local/bin/mise --version
mise doctor
```

If `mise doctor` reports shell activation problems, follow its shell-specific instructions before creating a project. Python shims should be active automatically after `cd` into the project.

### uv

Install `uv` (macOS and Ubuntu):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

## Set Up A Project

Run once per project (or after cloning). `mise` pins the interpreter; `uv` owns the virtualenv and dependencies.

### Pin Python

From the project root:

```bash
mise use python@3.14   # writes the pin to mise.toml AND installs Python 3.14
mise install           # re-install everything pinned in mise.toml; idempotent.
                       # Use after `git clone` or after editing mise.toml by hand.
python --version
```

Commit `mise.toml` so every developer and automation runner resolves the same Python minor version.

### Initialize The uv Project

Workflow assumption: the folder is created by copying a bootstrap markdown set out of the software-factory repo, so it already has docs (including this one) but no Python project metadata yet.

From the project root:

```bash
uv init
```

`uv init` writes `pyproject.toml`, `main.py`, `.gitignore`, `.python-version`, and (only if missing) `README.md`. It does not remove or overwrite existing files. Re-running it after a project exists fails by design.

Then drop the Python version file — `mise.toml` owns the interpreter pin:

```bash
rm -rf .python-version
```

Finally, edit the generated `pyproject.toml` so its required Python version matches the `mise.toml` pin:

```toml
[project]
requires-python = ">=3.14,<3.15"
```

### Create The Virtual Environment

`uv` is the project manager — it creates and owns `.venv` automatically based on `pyproject.toml` and `uv.lock`. After `uv init`, `uv sync` has a `pyproject.toml` to read. Do not pre-create the venv with `uv venv`; let `uv sync` handle it.

```bash
uv sync                          # creates .venv (if missing) and installs all deps from uv.lock
source .venv/bin/activate
```

After activation, `python`, `ruff`, `mypy`, `pytest`, and other dev dependencies come from `.venv`. Re-run `uv sync` whenever `pyproject.toml` or `uv.lock` changes.

## Multi-Package Repos

For a repo with multiple installable Python packages:

- Put one root `mise.toml` at the repo root.
- Pin one shared Python version in that root `mise.toml`.
- Keep each package's `pyproject.toml` `requires-python` constraint compatible
  with the root Python pin.
- Sub-packages may use stricter lint or type settings, but they should not
  require a different Python minor version unless the repo is intentionally
  split into separate runtime targets.
