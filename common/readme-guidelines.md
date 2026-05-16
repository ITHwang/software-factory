# README Guidelines

> Last updated: 2026-05-16

## TL;DR

How to write README files in projects scaffolded from software-factory. Two shapes: the **project root README** (what / why / high-level design) and the **subproject README** (how to set up and run one component folder).

**Use this when:**
- scaffolding a new project root README
- adding a README to a component folder (`backend/`, `frontend/`, `design/`, …)

**Don't use this for:**
- writing reference docs under `py/`, `rs/`, `ts/` → `../documentation-guidelines.md`
- PRD or feature specs → `../prd.md`

## Table of Contents

| Section |
|---------|
| [Project Root README](#project-root-readme) |
| [Subproject README](#subproject-readme) |

## Project Root README

Lives at the project root: `/README.md`. Audience: someone landing on the repo for the first time — what is this, why does it exist, how is it shaped.

Required H2 sections, in order:

- `## What is [Project Name]` — one-paragraph identity. What the thing is, not how it works.
- `## Why [Project Name]` — the problem it solves and who it's for.
- `## Requirements` — functional and non-functional requirements the project commits to.
- `## Definition` — the domain language and boundaries: the entities, terms, and concepts the rest of the doc uses.
- `## Design` — high-level architecture. Backend / frontend / data flow diagrams (Mermaid welcome). This is where architectural choices live — not in subproject READMEs.
- `## Implementation` — **brief only**: list tech stacks per component (e.g., "Backend: FastAPI + SQLAlchemy + Milvus; Frontend: Next.js"). Detailed setup, commands, and per-component specifics belong in each subproject README, linked from here.

## Subproject README

Lives at the component folder root: `/<component>/README.md` (e.g., `backend/README.md`, `frontend/README.md`). Audience: a developer who needs to run, test, or extend this one component locally.

Required H2 sections, in order:

- `## Prerequisites` — external services, accounts, runtimes, and credentials needed before any setup step works (DBs, API keys, CLI tools, target runtime versions).
- `## Project Setup` — install deps, create venv / install node_modules, configure `.env`. The reader should be able to follow this top-to-bottom once.
- `## Running` — exact commands to start the component locally (dev mode, ports, flags).
- `## Testing` — how tests are organized (unit / integration / e2e), how to run them, what fixtures or mocks exist.
- `## Development` — conventions specific to working in this folder: code layout, where to add a new endpoint / page / module, lint / format commands, anything component-specific not already covered upstream.
