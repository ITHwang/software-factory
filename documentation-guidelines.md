# Documentation Guidelines

> Last updated: 2026-05-14

## TL;DR

**Why** every reference doc is structured this way: to pre-decide architecture / tooling / when-to-use questions so an agent applies them without escalating to the human (see [Purpose](#purpose)). **How** the structure helps the agent:

- decide whether a doc is relevant from the first ~512 tokens (TL;DR + navigation table fit in that window),
- jump straight to the section it needs from a table at the top,
- read external citations inline at the section that needs them — no round-trip to a References footer,
- write content at section level that names architectural intent without over-specifying derivable implementation.

**Use this when:** writing or revising any `*.md` reference doc under a language directory (`python/`, `rust/`, `typescript/`, …).

**Don't use this for:** project-memory files (`CLAUDE.md`), PR descriptions, or one-off RFCs — those have their own shape.

## Purpose

This corpus exists to **pre-decide** the architecture, tooling, and when-to-use questions that a coding agent would otherwise interrupt to ask. The agent's job is to *apply* these decisions, not to re-derive them on every task.

- **Binding intent over derivable implementation.** When a doc says "use X for Y" or "name ports for the capability," that *is* the answer — not one option among alternatives the agent should weigh against training-data defaults. The decisions in these docs are load-bearing; the agent follows them.
- **Decisions live here; implementations live in code.** A doc names the architectural rule, the tool choice, the layer boundary, the trigger condition. The exact body the agent writes is derivable from those rules plus the surrounding codebase.
- **Silence is the only escalation signal.** If a doc covers the question, follow it. If no doc covers it, *then* ask the human. The `Use when` and `Don't use for` lines in every TL;DR exist so the agent knows which doc owns which decision — and which sibling doc to read instead when this one doesn't.

Every other rule in this file — structure, TL;DR shape, TOC placement, section discipline — exists to serve that purpose. Structure makes a decision findable in the first ~512 tokens; section discipline keeps the decision from being buried under derivable detail.

## Required structure

Every doc has this skeleton, in this order:

1. **H1 title** — the doc's topic. One per file.
2. **`> Last updated: YYYY-MM-DD`** — single-line blockquote, immediately under the H1. Bump on any non-trivial content change.
3. **`## TL;DR` section** — one-line purpose, decision triggers (`Use this when`), and redirects to the sibling docs that own adjacent decisions (`Don't use this for`). See [TL;DR section](#tldr-section).
4. **`## Table of Contents` section** — phase-grouped table inside it (recommended) or flat TOC of section anchors. See [Table of Contents](#table-of-contents).
5. **Body sections** — H2 sections in reading order. External links live inline at the section's end as `See also:` lines.
6. **No trailing `## References`** section. References belong at the section that uses them, not in a footer.

## TL;DR section

The TL;DR is the "should I read this doc?" gate. Three blocks, in this order:

```markdown
## TL;DR

One-to-two sentences naming what the doc covers.

**Use this when:**
- concrete trigger condition 1
- concrete trigger condition 2

**Don't use this for:**
- anti-pattern → sibling doc that handles it (`./other-doc.md`)
```

Keep it concrete. "When you need X" beats "general guide to X". The two bullet groups are decision triage, not navigation hints:

- **Use this when** lines are *decision triggers* — the concrete situations on which this doc's pre-committed decisions activate. List tasks an agent will recognize ("starting a new domain package," "picking a production stack"), not abstract topics ("about architecture").
- **Don't use this for** lines are *redirects* — they name the sibling doc that owns the adjacent decision, so the agent stops reading here and goes there. Always link the sibling.

**Size cap: 512 tokens (~400 words, ~50 lines).** This is the ceiling — most TL;DRs land in the 100–200-token range. The cap exists because the TL;DR + the navigation table that follows must both fit in a coding-agent's first peek window (also ~512 tokens). If the TL;DR is hitting the cap, the doc probably covers too much — split it.

## Table of Contents

The Table of Contents is its own `## Table of Contents` H2 section, placed immediately after `## TL;DR` and before any body section. Together they form the navigation header — the first ~512 tokens an agent peeks. Everything that follows the TOC (Quick Reference, When To Use, Install, …) is a regular body section listed *inside* the TOC.

Inside the section, a phase-grouped table is the recommended shape: it groups H2s by reading order (Concepts → Setup → Build → Operate) and tells the agent *what to read first*, not just *what's available*.

```markdown
## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Quick Reference](#quick-reference), [When To Use](#when-to-use) |
| 2. Setup | [Install](#install), [Wire Up](#wire-up) |
| 3. Build | [Core Pattern](#core-pattern), [Tools](#tools) |
| 4. Operate | [Testing](#testing), [Production Checklist](#production-checklist) |
```

A flat TOC is acceptable for short docs where reading order is obvious. Whichever shape, every entry must link to an in-page anchor.

## Section structure

Each H2/H3 section is self-contained. External links go at the **end of the section** as a single line:

```markdown
See also: [LangChain agents docs](https://docs.langchain.com/oss/python/langchain/agents), [`AgentMiddleware` reference](https://reference.langchain.com/...).
```

Place at most one `See also:` line per section. If you have many links, the section may need splitting into subsections, each with its own `See also:`.

## Last-updated

The `> Last updated: YYYY-MM-DD` line is the freshness signal. Bump it when:

- you add, remove, or restructure a section,
- you update a code example to track a library API change,
- you correct factual content.

Don't bump for typo fixes or formatting-only changes. The date is a hint to the agent ("if it's >6 months old, treat conventions as possibly stale"), not a changelog.

## Example skeleton

```markdown
# Caching

> Last updated: 2026-05-11

## TL;DR

How to add a Redis-backed cache to a FastAPI service: client, key conventions, TTL strategy, invalidation.

**Use this when:**
- a hot read path is hitting Postgres on every request
- you need to share state across worker processes

**Don't use this for:**
- in-process memoization → `functools.lru_cache` (stdlib)
- session storage → `./cognito-auth.md`

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Quick Reference](#quick-reference) |
| 2. Setup | [Install](#install), [Wire Up](#wire-up) |
| 3. Build | [Keys](#keys), [TTL](#ttl), [Invalidation](#invalidation) |
| 4. Operate | [Pitfalls](#pitfalls) |

## Quick Reference

…

See also: [redis-py async docs](https://redis.readthedocs.io/...).

## Install
…
```

## Content Discipline

`Required structure` and `Section structure` cover *how a doc is laid out*. This section covers *what goes inside a section*. The two dimensions are independent — the same doc can pair a structurally compliant TOC with content that over-specifies implementation, and an agent reading it will copy the over-specification.

### Core principle

Humans define intent; agents synthesize implementation. The job of a reference doc is to **pre-decide** what the agent must not violate (see [Purpose](#purpose)), not to spell out every line the agent can derive. A doc's value is the decisions it carries — architectural rules, tool choices, naming conventions, layer boundaries. Derivable filler dilutes the binding signal: the agent has to weigh which lines are normative and which are illustrative, and that ambiguity is exactly what re-introduces the escalation the corpus is meant to eliminate.

The decision is **section-level, not file-level**. A single `cache.md` doc legitimately contains an install section (verbatim commands), TTL/key recipe sections (recipe code), and a `RedisClient` lifecycle section (architectural scaffold). Each section picks its content style from what *that section* defines, not from what file it lives in.

### Decide what the section defines

Before writing a section, answer: *what is this section defining?* The answer picks the content style.

| If the section defines… | Then content should be… |
|---|---|
| an object, boundary, lifecycle, dependency relationship, or architectural role | intent-driven prose with the [object scaffold](#object-scaffold) |
| a tool recipe, library usage pattern, or API surface | task-focused prose + the calls/signatures needed to use it |
| project setup, CI workflow, dependency pin, lockfile-adjacent config | verbatim code/config — see [When full code is allowed](#when-full-code-is-allowed) |
| install steps or one-liner commands | exact commands |

Most over-harnessed docs come from picking row 1's *subject* and row 2's *style* — writing a `RedisClient` section as if it were a Redis recipe. The decision rule above prevents that.

### Snippet style

For sections that are *not* verbatim-reproducible, prefer shape over body.

**Prefer**:
- function and method signatures (`async def execute(...) -> Result: ...`)
- class signatures and `Protocol` definitions
- dependency-graph sketches
- lifecycle-sensitive minimal examples (a few lines that show ordering, not the full body)

**Avoid**:
- full method bodies for mechanical operations
- repetitive CRUD implementation
- framework glue the agent can safely infer (basic FastAPI route registration, basic Pydantic model construction, etc.)
- long examples whose body adds no constraint the prose hasn't already named

Default style:

```python
class RDBClient:
    def __init__(self, session: AsyncSession) -> None: ...
    async def execute(self, sql: str, params: dict[str, Any] | None = None) -> Result: ...
    async def commit(self) -> None: ...
    async def rollback(self) -> None: ...
```

This tells the agent the object's contract — its dependencies, its method shape, its return types — without over-constraining the implementation body. If a future caller passes a different parameter dict shape, the doc still describes the right object.

### When full code is allowed

Two cases, both legitimate:

1. **Load-bearing control flow** — the body carries architectural risk that prose can't fully capture, and getting the order wrong is expensive to debug. Examples:
   - FastAPI yield-dependency cleanup ordering (commit-on-yield-exit, rollback-on-exception)
   - SQLAlchemy async session lifecycle (engine dispose, session close, commit/rollback ordering)
   - transaction boundary behavior
   - context manager ordering
   - resource cleanup on exception
   - concurrency-sensitive primitives (locks, queues, signal handlers)

2. **Verbatim-reproducible content** — the section's job is to be copied as-is into a new project. Examples:
   - project bootstrap commands (`uv add ruff mypy pre-commit`)
   - CI workflows (`.github/workflows/*.yml`)
   - `pyproject.toml` tool configs (`[tool.ruff.lint]`, `[tool.mypy]`)
   - `.pre-commit-config.yaml` pin lists
   - Dockerfile snippets

Rule: *include full code only when omission creates high debugging risk, or when the section must be copy-pasted to be useful.* Outside those two cases, prefer signatures plus prose.

### Object scaffold

Use this scaffold when a section defines an object, boundary, lifecycle, dependency relationship, or architectural role. Do **not** force it onto recipe / install / API-reference / config sections — the scaffold becomes noise when applied to material it wasn't designed for.

Template:

```markdown
## ObjectName

### Purpose
One sentence: why this object exists.

### Responsibilities
- What the object owns.
- What it must not own (and who owns it instead).
- Transaction / error / concurrency ownership.

### Lifecycle
- Scope: App Scope / Request Scope / Owned-Transient Scope.
- Who creates it.
- Who shares it.
- Who cleans it up.

### Relationships
ASCII graph showing dependency direction, ownership, and lifecycle scope.

### Constraints
- Must not depend on X.
- Must not be shared across Y.
- Must not own Z.
```

### Lifecycle scope tags

Use these three scope names in lifecycle sections and relationship graphs:

| Scope | Meaning | Typical owner |
|---|---|---|
| **App Scope** | One object per Python worker/process app lifespan. It starts when that worker app starts and ends when that worker app shuts down; it is not deployment-wide. | `dependency-injector` `Singleton` or `Resource`, usually initialized/closed by FastAPI lifespan. |
| **Request Scope** | One object or context for one HTTP request, cleaned up after the response path finishes. | FastAPI `Depends` / `yield` dependencies for strict lifecycle ownership; a request-owned call chain for lightweight factory-built objects. |
| **Owned-Transient Scope** | Object created by an owner object or operation and used only while that owner/call needs it. | Normal Python ownership, function scope, composition, or context managers. |

Provider names are not scopes by themselves. `providers.Factory` means "new object per provider call"; it becomes request-owned only when a request-scoped caller creates/receives it once and passes it down. Cleanup-sensitive owned/transient objects must use explicit cleanup (`with`, `async with`, `try/finally`, or `close()`), not implicit finalization.

Lifecycle-aware relationship graph (worked example for an async SQLAlchemy stack):

```text
[App Scope]
    ├── [AsyncEngine]        (app-scoped Resource)
    └── [sessionmaker]       (app-scoped Singleton)

[Request Scope]
    └── [AsyncSession]       (request-scoped Depends/yield)
            │ shared by
            ▼
       [RDBClient]           (request-scoped Depends/yield)
            │ injected into
            ▼
       [Repositories]        (request-owned Factory)
            │ used by
            ▼
       [Services]            (request-owned Factory)
```

Lifecycle belongs to the relationship graph as much as to the object. "AsyncSession is request-scoped" really means *"one AsyncSession is created per request, shared by request-local repositories/services, cleaned up at the end of that request."* The scope tag on a node is shorthand; the graph carries the actual contract.

### Patterns as relationship constraints

Document design patterns and architectural styles (SOLID, DDD, hexagonal, ports-and-adapters) primarily as **relationship rules**, not code templates.

Examples of the right shape:

- Services depend on ports, not concrete adapters.
- Adapters implement ports; routes don't reach past services into adapters.
- Repositories never own transaction boundaries.
- Request-scoped mutable objects must not leak into singletons.
- Singleton objects must be stateless or concurrency-safe.
- Factory-built objects must not be described as request-scoped unless a request-scoped owner controls one instance for the request.
- Owned/transient objects with external resources must have explicit cleanup; implicit finalization is not an architectural lifecycle owner.

Rules generalize; code templates rot. A constraint like *"repositories never own transaction boundaries"* lets the agent apply the pattern to new repositories the doc never anticipated. A code template showing one transaction-aware repository teaches the agent to copy that exact shape and breaks the moment the next repository has a different signature.
