# Pragmatic SQL-First Persistence (SQLAlchemy + Raw SQL)

> Last updated: 2026-05-12

## TL;DR

How to build a SQL-first async persistence layer: SQLAlchemy 2.x as infrastructure (engine, sessions, schema metadata, Alembic), raw SQL via `text()` as the application query interface, a `SqlMapper` for function-name validation, and a request-scoped `RDBClient` that just executes raw SQL.

Layer responsibilities live in [./python-backend-architecture.md](./python-backend-architecture.md); this doc is the persistence-layer specialization.

**Use this when:**
- the team wants explicit SQL with bound parameters, not an ORM query builder
- you need Alembic migrations driven by schema-metadata-only ORM models
- you need async Postgres access with a request-scoped session
- you've already read [./python-backend-architecture.md](./python-backend-architecture.md) for the ports/adapters baseline

**Don't use this for:**
- vector similarity search → `./milvus.md`
- object storage → `./s3-client.md`
- caching layer → `./cache.md`

This guide is async-first. There is no sync variant.


## Table of Contents

| Phase         | Section                                                                                                                    |
| ------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 1. Concepts   | [Quick Reference](#quick-reference), [When To Use](#when-to-use), [Mental Model](#mental-model)                            |
| 2. Rationale  | [Why Explicit SQL Over Query Builder](#why-explicit-sql-over-query-builder)                                                |
| 3. Modeling   | [ORM Models = Schema Metadata Only](#orm-models--schema-metadata-only)                                                     |
| 4. Runtime    | [Engine & Session Lifecycle](#engine--session-lifecycle), [The RDBClient](#the-rdbclient), [Transaction Ownership](#transaction-ownership), [The SqlMapper](#the-sqlmapper) |
| 5. Pattern    | [Repository Example](#repository-example)                                                                                  |
| 6. Operate    | [Migrations With Alembic](#migrations-with-alembic), [Testing Philosophy](#testing-philosophy)                             |
| 7. Discipline | [Anti-Patterns](#anti-patterns), [See Also](#see-also)                                                                     |


## Quick Reference


| Concern                | Choice                                                                       |
| ---------------------- | ---------------------------------------------------------------------------- |
| Python                 | 3.14+ (for `uuid.uuid7`, modern typing)                                      |
| Schema source of truth | Alembic migrations (autogenerate allowed; never trusted blindly)             |
| ORM model role         | Schema metadata only — never the query interface                             |
| Query language         | Raw SQL in `*.sql` files, executed via `text()`                              |
| Query lookup           | `SqlMapper[RepositoryT]` — name-validates SQL file against repository impl   |
| Execution              | Request-scoped `RDBClient` that wraps a live `AsyncSession` and `execute(text(...))` |
| Session DI             | `RDBClient` is request-scoped via `fastapi.Depends(get_rdb_client)`. Services and repositories receive `rdb` as a method argument |
| Transaction boundaries | Default: commit on request success / rollback on exception (owned by `get_rdb_client`). Use-case-driven checkpoints: `await rdb.commit()` mid-method |
| FK constraints         | `ForeignKey()` on `mapped_column` only — no `relationship()`                 |
| Repository contract    | `Protocol` in application layer; `Postgres`* impl in adapters layer; `rdb: RDBClient` is a method argument, not a constructor arg |
| Tests                  | API tests only — mock the `Protocol`, no real DB                             |


## When To Use

Use this pattern when the project values:

- **Explicit query visibility** — the SQL is reviewable as SQL, not reconstructed from method chains.
- **PostgreSQL-native control** — CTEs, window functions, `RETURNING`, `ON CONFLICT`, partial indexes, JSON ops, advisory locks all stay first-class.
- **Operational migration awareness** — schema evolution is treated as a procedure (lock duration, backfills, online indexes), not a diff.
- **Strict layer boundaries** — application layer depends on a `Protocol`; persistence layer is the only place that knows about SQLAlchemy.

## Mental Model

The architecture splits responsibilities across seven layers. Each row owns
exactly one concern:


| Layer                     | Role                                                                     |
| ------------------------- | ------------------------------------------------------------------------ |
| Alembic migrations        | **Schema source of truth** — versioned, reviewed, manually edited        |
| SQLAlchemy ORM models     | Schema metadata for Alembic autogenerate; nothing else                   |
| Raw `*.sql` files         | The application query language                                           |
| `SqlMapper`               | Persistence-layer query lookup with name-match validation at import time |
| `RDBClient`               | Request-scoped session wrapper; executes raw SQL via `text()`; exposes `commit()` for service-level checkpoints |
| Repository (`Postgres`*)  | Persistence adapter implementing the application-layer `Protocol`; receives `rdb` per method call |
| Pydantic / dataclass DTOs | API contracts and domain transport objects                               |


The single rule that holds it together:

```text
SQL is the canonical query contract.
Everything else is plumbing around it.
```

Final mental model — who owns what:

```text
FastAPI Depends:        request-scoped infrastructure lifecycle
dependency_injector:    application-scoped infrastructure + DI wiring
SQLAlchemy:             engine/session infrastructure + schema metadata
Raw SQL:                canonical query language
Repositories:           thin persistence adapters
Services:               business orchestration + transaction decisions
```

## Why Explicit SQL Over Query Builder

Query builders do not remove query complexity — they relocate it. Conditional
joins, optional filters, sort branches, pagination variants, and CTE branching
all become harder to reason about when distributed across method chains.

Compare. Builder style:

```python
stmt = select(UserRow).where(UserRow.email == email)
if include_org:
    stmt = stmt.options(selectinload(UserRow.org))
if active_only:
    stmt = stmt.where(UserRow.is_active.is_(True))
result = await session.exec(stmt)
```

Raw SQL:

```sql
-- name: find_active_with_org
SELECT u.*, o.name AS org_name
FROM users u
JOIN orgs o ON o.id = u.org_id
WHERE u.email = :email AND u.is_active = TRUE;
```

The raw form keeps:

- **Query shape visible** — anyone reviewing the file sees the actual SQL the database will run.
- `**EXPLAIN` analysis trivial** — paste it into `psql` unmodified.
- **Index reasoning explicit** — column order, predicates, and join order are right there.
- **PostgreSQL behavior unhidden** — no abstraction layer to debug through.

The default position:

```text
A query builder must justify itself.
Raw SQL is the default.
```

See also: [SQLAlchemy ORM query guide (the path we're not taking)](https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html).

### Dynamic Queries Stay Plan-Aware

"Raw SQL is the default" does not license building dynamic queries by string
concatenation that obscure the query plan. The classic anti-pattern is the
all-things-to-all-callers query:

```sql
-- ANTI-PATTERN: one query, many shapes, opaque plan
SELECT * FROM users
WHERE (:email IS NULL OR email = :email)
  AND (:org_id IS NULL OR org_id = :org_id)
  AND (:is_active IS NULL OR is_active = :is_active);
```

The planner sees a constant-folded predicate at prepare time but the index
selection becomes brittle, partial indexes go unused, and reviewers can't tell
which call site exercises which branch.

Prefer one of:

- **Explicit query variants** — `find_by_email`, `find_by_org_id`,
  `list_active_in_org` as separate named queries in the same `.sql` file.
  Repository method names mirror the SQL.
- **Separate query shapes** when joins/filters/sorts materially affect the
  plan — a list-with-org join is a different query from a flat list, not the
  same query with a conditional `JOIN`.
- **Selective composition in Python** only for orthogonal modifiers (pagination
  clauses, ordering direction) where the plan stays stable.

The rule: if changing a parameter changes which index the planner picks, it
belongs in a separate named query.

## ORM Models = Schema Metadata Only

ORM models exist to populate `Base.metadata` so Alembic autogenerate can detect
schema drift. They are not the query interface.

The two pieces every "normal" table wants — a UUIDv7 primary key and audit
timestamps — are exposed as **two single-purpose mixins**, not one bundled
`BaseTable`. Splitting keeps `Base` honest as the literal SQLAlchemy declarative
base, lets non-standard tables (legacy bigint PKs, composite keys, reference
tables) compose à la carte from the same vocabulary, and makes field origin
obvious from the class declaration line. `isinstance(model, TimestampMixin)`
also becomes a usable runtime predicate for generic audit/sync helpers.

```python
# persistence/models/base.py
import uuid
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class UUIDv7Mixin:
    """Single-purpose mixin: UUID v7 primary key."""

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True,
        default=uuid.uuid7,
    )


class TimestampMixin:
    """Single-purpose mixin: created_at / updated_at audit columns."""

    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(),
        onupdate=func.now(),
    )
```

A concrete table:

```python
# persistence/models/users.py
import uuid

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base, TimestampMixin, UUIDv7Mixin


class User(Base, UUIDv7Mixin, TimestampMixin):
    __tablename__ = "users"

    email: Mapped[str] = mapped_column(unique=True, index=True)
    name: Mapped[str] = mapped_column()
    org_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("orgs.id"))
```

Two things are conspicuously absent:

1. **No `relationship()`.** It is pure ORM machinery for object traversal. We
  never query through the ORM, so it has zero effect on the database schema
   or the SQL we write. The `ForeignKey()` in `mapped_column` is what creates
   the FK constraint in the database — that is all we need for raw SQL joins
   to enforce referential integrity.
2. **No SQLModel.** SQLModel is `pydantic.BaseModel + sqlalchemy.DeclarativeBase`.
  Since DTOs are separate Pydantic classes anyway and we never use the unified
   model for queries, SQLModel adds no value over pure SQLAlchemy. Use
   SQLAlchemy directly.

See also: [Python `uuid`](https://docs.python.org/3.14/library/uuid.html).

## Engine & Session Lifecycle

The engine is one async resource per process. The sessionmaker is an
application-scoped singleton. Sessions are per-request, and the `RDBClient`
that wraps each session is request-scoped via a FastAPI dependency — *not* a
DI-container factory.

The split is deliberate:

- **DI container** owns long-lived infrastructure (engine, sessionmaker).
- **FastAPI dependency** owns the request-scoped session + `RDBClient` and
  the request-lifecycle commit/rollback.

Container wiring (engine + session_factory + the `get_rdb_client` FastAPI dependency) lives in [./dependency-injector.md#sqlalchemy](./dependency-injector.md#sqlalchemy). This section explains the architectural shape; that section is the canonical code.

`expire_on_commit=False` on the sessionmaker is **load-bearing** — without it, attribute access
after `commit()` triggers an implicit refresh that raises `MissingGreenlet`
in async response serialization.

Each request gets its own `RDBClient` instance wrapping a fresh
`AsyncSession`. `AsyncSession` is not concurrent-safe; every asyncio task that
needs database access needs its own session.

> **Infrastructure-level commit vs. hidden per-call commit.** The
> `await session.commit()` at yield exit is an *infrastructure-level
> lifecycle commit*, not a hidden per-call commit. The principled
> distinction: infrastructure owns the request-lifecycle commit; repositories
> never own per-call commits; services punch intermediate commits explicitly
> via `await rdb.commit()`. Future readers should not collapse the two.

### Why `RDBClient` comes from FastAPI `Depends`

Routes don't manipulate `AsyncSession` directly. That principle stands. But
routes-as-infrastructure means composing request-scoped infrastructure via
`Depends` is the right tool, not the wrong one. The rule sharpens to:

- *Routes* depend on services for business logic and (via `Depends`) on
  request-scoped infrastructure (`get_rdb_client`).
- *Services* never import from `fastapi`. They receive `rdb` and DTOs as
  plain method arguments, threaded by the route.
- *Repositories* are near-stateless adapters; they take `rdb` per call.

A correct route receives `rdb` via `Depends(get_rdb_client)` and a service via `Depends(Provide[Container.user_service])`, then calls `await user_service.get(rdb, user_id)`. The full route-handler shape lives in the canonical sqlalchemy entry linked above.

What's still banned: route code that imports `AsyncSession` and runs queries.
The HTTP layer composes infrastructure via `Depends`; it does not query.

See also: [./python-backend-architecture.md#services-and-fastapi](./python-backend-architecture.md#services-and-fastapi), [SQLAlchemy asyncio extension](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html), [SQLAlchemy PostgreSQL dialect](https://docs.sqlalchemy.org/en/20/dialects/postgresql.html), [asyncpg](https://magicstack.github.io/asyncpg/current/).

## The RDBClient

### Purpose

Request-scoped SQL execution adapter that wraps an `AsyncSession` and exposes `execute` / `commit` / `rollback` to services and repositories.

### Responsibilities

- Execute parameterized raw SQL via `text()` against the wrapped `AsyncSession`.
- Expose `commit` (and, where used, `rollback`) so services can punch durable intermediate checkpoints.
- Surface database errors as exceptions to the caller — no swallowing, no retry policy.
- Participate in the FastAPI dependency-injected request lifecycle; one instance per request.
- **Must not** open or close sessions itself; the session is constructor-injected.
- **Must not** decide business transaction boundaries — those belong to services.
- **Must not** outlive a single request.

### Lifecycle

- **Scope:** request-scoped.
- **Created by:** the `get_rdb_client` FastAPI dependency factory, which opens a fresh `AsyncSession` from the singleton `session_factory` and constructs an `RDBClient` around it.
- **Shared by:** every repository and service participating in the request — threaded explicitly as a method argument, not held as constructor state.
- **Cleaned up by:** `get_rdb_client` at yield exit — commit on success, rollback on uncaught exception, session close in `finally`.

### Relationships

See [Engine & Session Lifecycle](#engine--session-lifecycle) for the full dependency graph (engine → session_factory → AsyncSession → RDBClient → repositories → services).

### Constraints

- Never a singleton; never shared across requests or asyncio tasks (`AsyncSession` is not concurrent-safe).
- Never provided via the `dependency_injector` Container — request-scoped infrastructure belongs to FastAPI `Depends`, not the application-scoped container.
- Never owns commits on behalf of repositories. Repositories never commit; only services do, and only as explicit checkpoints.
- `execute` is never wrapped in a transactional decorator — no hidden commits.

The `RDBClient` is a thin, request-scoped wrapper around a live
`AsyncSession`. Its constructor takes the session; its only persistence
responsibility is to execute raw SQL. It does not open sessions, manage an
`AsyncExitStack`, or wrap `execute` in a transaction decorator. Lifecycle
belongs to `get_rdb_client`; transaction boundaries belong to the service
layer.

```python
# persistence/rdb/rdb_client.py
from typing import Any

from sqlalchemy import Result, text
from sqlalchemy.ext.asyncio import AsyncSession


class RDBClient:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def execute(
        self,
        sql: str,
        params: dict[str, Any] | None = None,
    ) -> Result:
        return await self._session.execute(text(sql), params or {})

    async def commit(self) -> None:
        """Service-level checkpoint commit.

        Use sparingly — only when business logic wants a durable
        intermediate state before continuing. The request-lifecycle
        commit/rollback is owned by `get_rdb_client`.
        """
        await self._session.commit()
```

Notes:

- **One method intentionally.** `execute` covers reads, writes, upserts,
  CTEs, and `RETURNING` clauses uniformly. There is no need for
  domain-specific methods on the client — that is the repository's job.
- **No hidden commits.** `execute` is a pure pass-through. There is no
  transactional decorator wrapping it. Repositories never commit. The
  request-level commit happens at `get_rdb_client` yield exit; intermediate
  commits are explicit calls to `await rdb.commit()` in services.
- **The session is owned by the FastAPI dependency, not by `RDBClient`.**
  `RDBClient.__init__` takes a live `AsyncSession`; it has no `initialize()`
  or `cleanup()` and no `AsyncExitStack`. That responsibility moved up to
  `get_rdb_client`.

See also: [SQLAlchemy `text()` (textual SQL)](https://docs.sqlalchemy.org/en/20/core/connections.html#using-textual-sql).

## Transaction Ownership

Transaction boundaries belong to the application/use-case (service) layer,
not to infrastructure and not to repositories. Two boundary types coexist:

- **Default (request-lifecycle) boundary.** `get_rdb_client` commits on
  successful request completion and rolls back on uncaught exception. This
  is the right default for the overwhelming majority of routes: the request
  is the unit of work.
- **Service-level checkpoint.** A service calls `await rdb.commit()`
  mid-method when business logic wants a *durable intermediate state* — for
  example, persisting that an asynchronous job has started before invoking
  the long-running call it depends on, so that a crash mid-call doesn't lose
  the started-marker.

Worked example:

```python
# application/agents/agent_service.py
class AgentService:
    def __init__(self, run_repo: RunRepository, agent: Agent) -> None:
        self.run_repo = run_repo
        self.agent = agent

    async def run_agent(self, rdb: RDBClient, command: str) -> RunDTO:
        run = await self.run_repo.create_started(rdb, command)
        await rdb.commit()                          # durable checkpoint
        result = await self.agent.run(command)
        await self.run_repo.mark_completed(rdb, run.id, result)
        return run
```

If the agent call crashes after the checkpoint, the started-row survives and
the request-lifecycle rollback only undoes work after the checkpoint. If
nothing crashes, the request-lifecycle commit at yield exit is a no-op
relative to the already-committed checkpoint plus a final commit of the
post-checkpoint statements.

**Repositories never call `commit()` or `rollback()`.** They take `rdb` and
execute against it. Owning transaction boundaries inside a repository hides
business decisions in infrastructure code.

## The SqlMapper

The `SqlMapper` is a persistence-layer utility that loads a `.sql` file into
a dict and validates at import time that its query names exactly match the
repository implementation's public methods. It is the thin compile-time
safety net that replaces the typed-query guarantees you would otherwise get
from a code generator.

```python
# persistence/rdb/sql_mapper.py
import inspect
import re
from pathlib import Path
from typing import Generic, TypeVar

RepositoryT = TypeVar("RepositoryT")

_NAME_RE = re.compile(r"--\s*name:\s*(\w+)")


def _load_sql(path: Path) -> dict[str, str]:
    queries: dict[str, str] = {}
    name: str | None = None
    buf: list[str] = []
    for line in path.read_text().splitlines():
        m = _NAME_RE.match(line)
        if m:
            if name is not None:
                queries[name] = "\n".join(buf).strip()
            name, buf = m.group(1), []
        else:
            buf.append(line)
    if name is not None:
        queries[name] = "\n".join(buf).strip()
    return queries


class SqlMapper(Generic[RepositoryT]):
    def __init__(self, repo_impl: type[RepositoryT], sql_path: Path) -> None:
        self._queries = _load_sql(sql_path)
        impl_methods = {
            n for n, _ in inspect.getmembers(repo_impl, inspect.isfunction)
            if not n.startswith("_")
        }
        sql_names = set(self._queries.keys())
        if impl_methods != sql_names:
            missing = impl_methods - sql_names
            extra = sql_names - impl_methods
            raise ValueError(
                f"sql/repo mismatch in {sql_path.name}: "
                f"missing={sorted(missing)}, extra={sorted(extra)}"
            )

    def __getitem__(self, name: str) -> str:
        return self._queries[name]
```

Key design choices:

- **Validates against the implementation, not the `Protocol`.** The mapper
lives entirely in the persistence layer. Whether the underlying store is
Postgres, MongoDB, or anything else is not its concern. The `Protocol` in
the application layer is store-agnostic and must remain so.
- **One SQL file per repository** — the file lives next to the repository
implementation. This sidesteps cross-file name collisions cleanly.
- **Failure at import time, not first call.** Method-name typos surface
during application startup, not during the first request that hits the
faulty query.
- **Underscore-prefixed methods are ignored.** Helper methods on the
repository (e.g. `_serialize_row`) need no SQL counterpart.

### SqlMapper Limitations

What `SqlMapper` validates:

- **Name correspondence.** Every `-- name: <foo>` block in the SQL file must
  match a public method on the repository implementation, and vice versa.
  Mismatches raise at import time.

What `SqlMapper` does **not** validate:

- **SQL correctness.** Syntax errors and references to nonexistent columns,
  tables, or functions only surface at execution time.
- **Parameter typing.** A method passing `{"id": "not-a-uuid"}` is caught
  by the database, not by the mapper.
- **DTO compatibility.** Whether the `SELECT` column list matches the DTO
  field set is not checked.
- **Selected column shape.** `SELECT *` vs explicit columns, ordering of
  columns, NULL/NOT-NULL semantics — none of it.
- **Schema drift.** A column rename in a migration that the SQL file
  doesn't follow produces a runtime error, not an import error.
- **Runtime query behavior.** Plan changes, lock acquisition, transactional
  effects — out of scope.

Those guarantees require integration tests against a real migrated database.

## Repository Example

The application layer defines the `Protocol`. The adapters layer provides the
`Postgres`* implementation that wires `RDBClient` and `SqlMapper` together.
`RDBClient` is threaded *through* the repository as a method argument, not
held as constructor state.

```python
# application/users/ports/user_repository.py
from typing import Protocol
from uuid import UUID

from persistence.rdb.rdb_client import RDBClient

from ..dto import UserDTO


class UserRepository(Protocol):
    async def get_by_id(self, rdb: RDBClient, user_id: UUID) -> UserDTO | None: ...
    async def find_by_email(self, rdb: RDBClient, email: str) -> UserDTO | None: ...
    async def create(self, rdb: RDBClient, email: str, name: str) -> UserDTO: ...
```

```python
# adapters/users/postgres_user_repository.py
from pathlib import Path
from uuid import UUID

from application.users.dto import UserDTO
from persistence.rdb.rdb_client import RDBClient
from persistence.rdb.sql_mapper import SqlMapper


class PostgresUserRepository:
    def __init__(self) -> None:                                # no rdb at construction
        self._sql = SqlMapper(PostgresUserRepository, Path(__file__).parent / "users.sql")

    async def get_by_id(self, rdb: RDBClient, user_id: UUID) -> UserDTO | None:
        result = await rdb.execute(self._sql["get_by_id"], {"id": user_id})
        row = result.mappings().first()
        return UserDTO.model_validate(row) if row else None

    async def find_by_email(self, rdb: RDBClient, email: str) -> UserDTO | None:
        result = await rdb.execute(self._sql["find_by_email"], {"email": email})
        row = result.mappings().first()
        return UserDTO.model_validate(row) if row else None

    async def create(self, rdb: RDBClient, email: str, name: str) -> UserDTO:
        result = await rdb.execute(
            self._sql["create"],
            {"email": email, "name": name},
        )
        return UserDTO.model_validate(result.mappings().one())
```

The matching `users.sql`:

```sql
-- name: get_by_id
SELECT id, email, name, created_at, updated_at
FROM users
WHERE id = :id;

-- name: find_by_email
SELECT id, email, name, created_at, updated_at
FROM users
WHERE email = :email;

-- name: create
INSERT INTO users (email, name)
VALUES (:email, :name)
RETURNING id, email, name, created_at, updated_at;
```

The Protocol leaks `RDBClient` into the application layer. We accept this:
`RDBClient` is a domain-shaped abstraction ("a thing that executes
parameterized SQL"), not a SQLAlchemy type, and threading it explicitly makes
the session context visible at every call site. The alternative — hiding the
session in repository state — saves keystrokes but obscures *which session*
each call runs against (important the moment a service composes multiple
repositories under one transaction or punches an intermediate `commit`).

The flow, starting at the route:

```text
Route handler
    ↓ Depends(get_rdb_client) → RDBClient(session)
    ↓ Depends(Provide[Container.user_service]) → UserService
service.method(rdb, ...)
    ↓
repository.method(rdb, ...)
    ↓
rdb.execute(sql, params)
    ↓
session.execute(text(sql), params)
```

## Migrations With Alembic

Alembic owns versioning and migration orchestration. Autogenerate is allowed
for convenience but **every generated migration is reviewed manually** before
landing.

Initialize with the async template:

```bash
alembic init -t async migrations
```

`migrations/env.py`:

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config

import persistence.models  # noqa: F401  — populates Base.metadata
from persistence.models.base import Base


config = context.config
fileConfig(config.config_file_name)

target_metadata = Base.metadata


def do_run_migrations(connection) -> None:
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,
        compare_server_default=True,
    )
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


asyncio.run(run_migrations_online())
```

Every model module **must** be imported before `target_metadata` is read,
otherwise autogeneration produces empty migrations. Centralize imports in
`persistence/models/__init__.py`.

### Why autogenerate is never trusted blindly

A naive autogenerated migration:

```sql
ALTER TABLE users
ADD COLUMN email TEXT NOT NULL;
```

is dangerous on a non-empty table — it acquires an `ACCESS EXCLUSIVE` lock
and rejects rows with no value. The reviewed, production-safe form:

```sql
ALTER TABLE users ADD COLUMN email TEXT;

UPDATE users SET email = ...;

ALTER TABLE users ALTER COLUMN email SET NOT NULL;
```

Migrations are operational procedures, not schema diffs. Every `op.*` call in
an autogenerated file gets read with this lens before merge.

Common commands:

```bash
alembic revision --autogenerate -m "add users.email"
alembic upgrade head
alembic downgrade -1
alembic current
alembic history
```

See also: [Alembic docs](https://alembic.sqlalchemy.org/en/latest/), [Alembic autogenerate](https://alembic.sqlalchemy.org/en/latest/autogenerate.html).

## Testing Philosophy

API tests are the default test layer. Unit tests are not part of this
pattern — application logic is exercised through API tests that go through
the full DI container.

Persistence is mocked at the **port** boundary (`UserRepository` Protocol),
not at `RDBClient`. A `FakeUserRepository` is overridden into the container,
and the route handler exercises the full stack (FastAPI → service → fake
repository) without touching a real database. We use `Fake*` rather than
`Mock*` to match the adapter-naming convention from
[./python-backend-architecture.md](./python-backend-architecture.md) — a fake
is just another adapter, useful only in tests.

```python
# tests/api/users/fake_repository.py
from uuid import UUID

from application.users.dto import UserDTO
from application.users.ports.user_repository import UserRepository
from persistence.rdb.rdb_client import RDBClient


class FakeUserRepository:
    """Structurally satisfies the `UserRepository` Protocol — no inheritance."""

    def __init__(self) -> None:
        self._by_id: dict[UUID, UserDTO] = {}
        self._by_email: dict[str, UserDTO] = {}

    async def get_by_id(self, rdb: RDBClient, user_id: UUID) -> UserDTO | None:
        # rdb accepted for Protocol conformance; unused by the in-memory fake.
        return self._by_id.get(user_id)

    async def find_by_email(self, rdb: RDBClient, email: str) -> UserDTO | None:
        return self._by_email.get(email)

    async def create(self, rdb: RDBClient, email: str, name: str) -> UserDTO:
        ...
```

Method signatures on the fake take `rdb: RDBClient` as the first argument
even though the body ignores it — Protocol conformance is structural, so the
signature must match the port exactly.

Wire the fake into the DI container at test setup:

```python
container.user_repository.override(providers.Object(FakeUserRepository()))
```

**Trade-off worth naming:** raw SQL never executes in CI under this policy.
Column-name typos and schema/query drift only surface in deployed
environments. The explicit choice is operational simplicity (no
testcontainers, no test DB lifecycle) over a CI-level safety net.

## Anti-Patterns

The pattern is defined as much by what it forbids as by what it embraces:

1. **No `relationship()`.** Pure ORM-side machinery; useless for raw SQL.
  Use `ForeignKey()` on `mapped_column` for FK constraints — that is what
   creates the database-level constraint your raw joins rely on.
2. **No SQLModel.** It is `pydantic + SQLAlchemy` with nothing extra you use.
  The unified model is wasted when DTOs are separate and queries are raw
   SQL. Use SQLAlchemy directly.
3. **No `AsyncSession` in route signatures.** Routes never touch
   `AsyncSession` directly. Routes depend on services for business logic and
   (via `Depends(get_rdb_client)`) on a request-scoped `RDBClient` for
   transaction lifecycle. The `Depends` mechanism is fine — what's banned is
   route code that imports `AsyncSession` and runs queries. If you see
   `session: AsyncSession = Depends(...)` in a route, that's the smell.
4. **No sync APIs.** `Session`, `create_engine`, sync sessionmakers — none of
  them appear in this stack. Async only.
5. **No ORM query construction in application code.** No `session.query`,
  no `select(User).where(...)`, no `session.get(User, id)`. All queries
   live in `*.sql` files and execute through `RDBClient.execute`.
6. **No SQL strings inline in repositories.** All SQL lives in the
  sibling `*.sql` file and is loaded by `SqlMapper`. Inline strings
   bypass the name-match validation and scatter SQL across Python files.
7. **No "default" composition base class.** Don't fold the mixins back into
  a `BaseTable` (or similarly-named) class as a convenience shortcut.
   "Base" implies universal lineage; the moment a legacy bigint-PK,
   composite-key, or natural-key table can't inherit from it, the name is
   a lie. Renaming it (`StandardTable`, `DefaultTable`) just relocates the
   lie. Keep `UUIDv7Mixin` and `TimestampMixin` as the only building blocks
   and have every model spell out what it has — one extra inheritance term
   per class is the price of honest naming.
8. **No hidden commits inside `RDBClient` or repositories.**
   `RDBClient.execute` never commits. Repository methods never commit. The
   request-lifecycle commit is owned by `get_rdb_client`; intermediate
   durable checkpoints are owned by services calling `await rdb.commit()`
   explicitly. A transactional decorator wrapping `execute` and a repository
   method that calls `session.commit()` are the same anti-pattern in two
   costumes — both put business decisions in infrastructure.
9. **No services importing from `fastapi`.** Services receive `rdb` and
   DTOs as method args. `Depends`, `Request`, `HTTPException` belong in
   routes. A service signature with `Depends(...)` defaults or a service
   body that raises `HTTPException` is the boundary breaking down. See
   [./python-backend-architecture.md](./python-backend-architecture.md).

## See Also

- [./python-backend-architecture.md](./python-backend-architecture.md) — ports/adapters baseline and layer responsibilities.
- [./dependency-injector.md](./dependency-injector.md) — DI container wiring (engine, sessionmaker, services).
- [./milvus.md](./milvus.md) — vector similarity search (different storage; same adapter pattern).
- [./s3-client.md](./s3-client.md) — object storage adapter example.
- [./cache.md](./cache.md) — caching layer.
- [SQLAlchemy asyncio extension](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Alembic docs](https://alembic.sqlalchemy.org/en/latest/)
