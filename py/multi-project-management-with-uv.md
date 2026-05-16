# Multi-Project Management with uv

> Last updated: 2026-05-16

## TL;DR

Three ways to organize multiple Python projects inside one Git repository with [uv](https://docs.astral.sh/uv/): **Isolated Projects** (no shared code), **Path Dependencies** (one project imports another as a local library), and **Workspaces** (single shared `.venv` + single `uv.lock` for everything).

**Use this when:**
- structuring a new monorepo that contains more than one Python project
- deciding whether MCP servers / shared libraries should live in their own folders, be path-linked, or share a workspace
- adding a second Python project alongside an existing one

**Don't use this for:**
- single-project setup, tooling baseline (Python version, ruff, pytest) → [`./py-setup-environments.md`](./py-setup-environments.md), [`./py-setup-toolchains.md`](./py-setup-toolchains.md)
- running MCP servers themselves (in-memory / subprocess / streamable HTTP) → [`./how-to-run-mcp-servers.md`](./how-to-run-mcp-servers.md)

## Table of Contents

- [1. Overview](#1-overview)
- [2. Management Methods](#2-management-methods)
  - [2.1. Method 1: Isolated Projects](#21-method-1-isolated-projects)
  - [2.2. Method 2: Path Dependencies](#22-method-2-path-dependencies)
  - [2.3. Method 3: Workspaces](#23-method-3-workspaces)
- [3. Conclusion](#3-conclusion)

## 1. Overview

1. When a single GitHub repository contains multiple Python projects, [uv](https://docs.astral.sh/uv/) — a package and dependency manager — supports several ways to organize them:

   1. **Isolated Projects**: each project has its own virtual environment (`.venv`) and lock file (`uv.lock`).
   2. **Isolated Projects with Path Dependencies**: when one project needs to consume another as a library, it pins that sibling project as a local path dependency.
   3. **Workspaces**: a Cargo-inspired concept that lets a single repository manage multiple Python projects under one unified dependency graph.

## 2. Management Methods

### 2.1. Method 1: Isolated Projects

- **When to use**: the projects live in the same repository for organizational reasons (e.g., they all belong to one customer) but share **no code-level dependencies**.
- **Key trait**: each project has its own `.venv` and its own `uv.lock`. They do not affect each other in any way.
- **Layout**

```text
multi-project-repo/
├── service-a/
│   ├── pyproject.toml
│   ├── uv.lock
│   └── .venv/
└── service-b/
    ├── pyproject.toml
    ├── uv.lock
    └── .venv/
```

```toml
# service-a/pyproject.toml

[project]
name = "service-a"
version = "0.1.0"
dependencies = [
    "fastapi>=0.110.0",
    "uvicorn>=0.29.0"
]
```

```toml
# service-b/pyproject.toml

[project]
name = "service-b"
version = "0.1.0"
dependencies = [
    "flask>=3.0.0",
    "gunicorn>=22.0.0"
]
```

- **Workflow**

```bash
# Create service-a's venv and install dependencies
$ cd service-a
$ uv sync

# Run service-a (assumes source code at src/main.py)
$ uv run uvicorn src.main:app --reload

# Switch to service-b and do the same
$ cd ../service-b
$ uv sync
$ uv run flask run
```

### 2.2. Method 2: Path Dependencies

- **When to use**: an application needs to import another library/package that lives in the same repository. Common when an internal shared library is developed alongside the applications that consume it.
- **Key trait**: the application's `pyproject.toml` points directly at the sibling library's path. On `uv sync`, uv resolves and installs the local library together with the rest of the dependencies.
- **Layout**

```text
path-dependency-repo/
├── internal-lib/         # internal shared library
│   ├── pyproject.toml
│   └── src/
│       └── internal_lib/
│           └── __init__.py
└── main-app/             # application consuming the library
    ├── pyproject.toml
    └── src/
        └── main_app/
            └── main.py
```

```toml
# internal-lib/pyproject.toml

[project]
name = "internal-lib"
version = "0.1.0"
dependencies = [
    "numpy>=1.26.0"
]
```

```toml
# main-app/pyproject.toml

[project]
name = "main-app"
version = "0.1.0"
dependencies = [
    "fastapi>=0.110.0",
    # Pin the sibling library by name and resolve it via [tool.uv.sources] below.
    "internal-lib"
]

[tool.uv.sources]
internal-lib = { path = "../internal-lib" }
```

- **Workflow**

```bash
# Create main-app's venv and install dependencies.
# uv resolves the local path of 'internal-lib' and installs it alongside.
$ cd main-app
$ uv sync

# main-app code can now import internal-lib.
$ uv run fastapi dev src/main_app/main.py
```

### 2.3. Method 3: Workspaces

- **Definitions**
  - **Workspace**: a collection of packages (projects). An entire Git repository can be a single workspace.
  - **Package (workspace member)**: a project unit with its own `pyproject.toml`. Can be an application or a library.
- **When to use**: multiple projects are tightly entangled, and you want to manage their dependencies under a single virtual environment and a single `uv.lock`. This is where the monorepo model really pays off.
- **Key traits**:
  - A `pyproject.toml` at the repository root defines the workspace.
  - All members' dependencies are resolved together into a single `uv.lock` file.
  - The entire repository shares one `.venv`.
  - Inter-member dependencies are expressed concisely via `workspace = true`.
- **Layout**

```text
workspace-repo/
├── .venv/             # single venv shared by the whole repo
├── packages/
│   ├── common-lib/    # shared library
│   │   ├── pyproject.toml
│   │   └── src/common_lib/
│   └── awesome-app/   # application
│       ├── pyproject.toml
│       └── src/awesome_app/
├── pyproject.toml     # ◀ workspace configuration
└── uv.lock            # ◀ single lock file for all dependencies
```

```toml
# workspace-repo/pyproject.toml

[tool.uv.workspace]
members = ["packages/*"]
```

```toml
# packages/common-lib/pyproject.toml

[project]
name = "common-lib"
version = "0.1.0"
dependencies = [
    "numpy>=1.26.0"
]
```

```toml
# packages/awesome-app/pyproject.toml

[project]
name = "awesome-app"
version = "0.1.0"
dependencies = [
    "fastapi>=0.110.0",
    # Reference another workspace member by name.
    "common-lib"
]

# Equivalent explicit form (uv 0.2.14+):
# [project.dependencies]
# common-lib = { workspace = true }
```

- **Workflow**

```bash
# (First time) Resolve all workspace dependencies into a single lock file.
$ uv lock

# (Subsequent) Install all dependencies into the single shared .venv from uv.lock.
$ uv sync

# Run a script in the context of a specific package (awesome-app).
$ uv run --package awesome-app -- uvicorn src.awesome_app.main:app --reload

# Run tests for a specific package (common-lib).
$ uv run --package common-lib -- pytest
```

## 3. Conclusion

### Which method should you use?

- **Isolated Projects**
  - Projects share only business context (e.g., same customer), not code.
  - MCP servers run on separate hosts or instances and never share a process.
  - No code-level dependencies between projects.

- **Path Dependencies**
  - You want to run MCP servers as subprocesses, or import them as libraries from the same host.
  - Internal shared libraries / MCP servers that the team maintains and versions together.
  - You want both independence (per-project lifecycle) and flexibility (selective import) at once.

- **Workspaces**
  - An application depends on many tightly coupled libraries.
  - All projects are closely related and you want to maximize the benefits of a monorepo.