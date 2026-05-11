# Documentation Guidelines

> Last updated: 2026-05-11

## TL;DR

How every reference doc in this repo is structured so a coding agent can:

- decide whether a doc is relevant from the first ~512 tokens (TL;DR + navigation table fit in that window),
- jump straight to the section it needs from a table at the top,
- read external citations inline at the section that needs them — no round-trip to a References footer.

**Use this when:** writing or revising any `*.md` reference doc under a language directory (`python/`, `rust/`, `typescript/`, …).

**Don't use this for:** project-memory files (`CLAUDE.md`), PR descriptions, or one-off RFCs — those have their own shape.

## Required structure

Every doc has this skeleton, in this order:

1. **H1 title** — the doc's topic. One per file.
2. **`> Last updated: YYYY-MM-DD`** — single-line blockquote, immediately under the H1. Bump on any non-trivial content change.
3. **`## TL;DR` section** — purpose + triggers. See [TL;DR section](#tldr-section).
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

Keep it concrete. "When you need X" beats "general guide to X". The "Don't use this for" bullets point at the sibling doc that *does* own that scope — they redirect the agent away.

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
