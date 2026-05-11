# Pragmatic SQL-First Persistence (SQLAlchemy + Raw SQL)

> Last updated: 2026-05-11

## TL;DR

How to build a SQL-first async persistence layer: SQLAlchemy 2.x as infrastructure (engine, sessions, schema metadata, Alembic), raw SQL via `text()` as the application query interface, a `SqlMapper` for function-name validation, and a request-scoped `RDBClient` that just executes raw SQL.

**Use this when:**
- the team wants explicit SQL with bound parameters, not an ORM query builder
- you need Alembic migrations driven by schema-metadata-only ORM models
- you need async Postgres access with a request-scoped session

**Don't use this for:**
- vector similarity search → `./milvus.md`
- object storage → `./s3-client.md`
- caching layer → `./cache.md`

How to build a SQL-first persistence layer in async Python: SQLAlchemy 2.x as
infrastructure (engine, sessions, schema metadata, Alembic), raw SQL via
`text()` as the application query interface, a `SqlMapper` for function-name
validation, and a request-scoped `RDBClient` that just executes raw SQL.

This guide is async-first. There is no sync variant.


## Table of Contents

| Phase         | Section                                                                                                                    |
| ------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 1. Concepts   | [Quick Reference](#quick-reference), [When To Use](#when-to-use), [Mental Model](#mental-model)                            |
| 2. Rationale  | [Why Explicit SQL Over Query Builder](#why-explicit-sql-over-query-builder)                                                |
| 3. Modeling   | [ORM Models = Schema Metadata Only](#orm-models--schema-metadata-only)                                                     |
| 4. Runtime    | [Engine & Session Lifecycle](#engine--session-lifecycle), [The RDBClient](#the-rdbclient), [The SqlMapper](#the-sqlmapper) |
| 5. Pattern    | [Repository Example](#repository-example)                                                                                  |
| 6. Operate    | [Migrations With Alembic](#migrations-with-alembic), [Testing Philosophy](#testing-philosophy)                             |
| 7. Discipline | [Anti-Patterns](#anti-patterns)                                                                                            |


## Quick Reference


| Concern                | Choice                                                                       |
| ---------------------- | ---------------------------------------------------------------------------- |
| Schema source of truth | Alembic migrations (autogenerate allowed; never trusted blindly)             |
| ORM model role         | Schema metadata only — never the query interface                             |
| Query language         | Raw SQL in `*.sql` files, executed via `text()`                              |
| Query lookup           | `SqlMapper[RepositoryT]` — name-validates SQL file against repository impl   |
| Execution              | Request-scoped `RDBClient` that owns the session and `execute(text(...))`    |
| Session DI             | Wired through the DI container; **never** through `fastapi.Depends` directly |
| FK constraints         | `ForeignKey()` on `mapped_column` only — no `relationship()`                 |
| Repository contract    | `Protocol` in application layer; `Sql`* impl in persistence layer            |
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
| `RDBClient`               | Request-scoped session owner; executes raw SQL via `text()`              |
| Repository (`Sql`*)       | Persistence adapter implementing the application-layer `Protocol`        |
| Pydantic / dataclass DTOs | API contracts and domain transport objects                               |


The single rule that holds it together:

```text
SQL is the canonical query contract.
Everything else is plumbing around it.
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

## ORM Models = Schema Metadata Only

ORM models exist to populate `Base.metadata` so Alembic autogenerate can detect
schema drift. They are not the query interface.

```python
# persistence/models/base.py
import uuid
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class BaseTable:
    """Mixin: UUID v7 primary key + audit timestamps."""

    id: Mapped[uuid.UUID] = mapped_column(
        primary_key=True,
        default=uuid.uuid7,
    )
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

from .base import Base, BaseTable


class User(Base, BaseTable):
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

The engine is one async resource per process. Sessions are per-request. The
DI container owns both.

```python
# persistence/database.py
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine


async def init_engine(url: str) -> AsyncIterator[AsyncEngine]:
    engine = create_async_engine(
        url,
        echo=False,
        pool_pre_ping=True,
        pool_size=10,
        max_overflow=20,
        pool_recycle=1800,
    )
    try:
        yield engine
    finally:
        await engine.dispose()
```

```python
# persistence/containers.py
from dependency_injector import containers, providers
from sqlalchemy.ext.asyncio import async_sessionmaker, AsyncSession

from .database import init_engine
from .rdb_client import RDBClient


class PersistenceContainer(containers.DeclarativeContainer):
    config = providers.Configuration()

    engine = providers.Resource(init_engine, url=config.db.url)

    session_factory = providers.Singleton(
        async_sessionmaker,
        bind=engine,
        class_=AsyncSession,
        expire_on_commit=False,
    )

    rdb_client = providers.Factory(RDBClient, session_factory=session_factory)
```

`expire_on_commit=False` is **load-bearing** — without it, attribute access
after `commit()` triggers an implicit refresh that raises `MissingGreenlet`
in async response serialization.

`rdb_client` is a `Factory` provider: every request gets its own
`RDBClient` instance, which holds a fresh `AsyncSession`. `AsyncSession` is
not concurrent-safe; every asyncio task that needs database access needs its
own session.

### Why sessions never come from `fastapi.Depends`

Sessions belong to the persistence layer. Routing is the API layer. The two
must not couple:

```python
# api/routes/users.py — CORRECT
@router.get("/users/{user_id}")
@inject
async def get_user(
    user_id: UUID,
    user_service: UserService = Depends(Provide[Container.user_service]),
) -> UserPublicDTO:
    user = await user_service.get(user_id)
    ...
```

The route depends on a **service**, not on a session or repository. The
service depends on the repository `Protocol`. The repository implementation
depends on `RDBClient`. The session is created and disposed inside that
chain — the API layer never sees `AsyncSession`.

Putting `Depends(get_session)` in route signatures inverts this: the HTTP
layer becomes responsible for persistence-layer lifecycle, and the layer
boundary breaks down.

See also: [SQLAlchemy asyncio extension](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html), [SQLAlchemy PostgreSQL dialect](https://docs.sqlalchemy.org/en/20/dialects/postgresql.html), [asyncpg](https://magicstack.github.io/asyncpg/current/).

## The RDBClient

The `RDBClient` is request-scoped, instance-based, and owns the session
lifecycle. Its only job is to execute raw SQL.

```python
# persistence/rdb/rdb_client.py
from contextlib import AsyncExitStack
from typing import Any

from sqlalchemy import Result, text
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from .transactional import transactional


class RDBClient:
    def __init__(self, session_factory: async_sessionmaker[AsyncSession]) -> None:
        self._session_factory = session_factory
        self._session: AsyncSession | None = None
        self._stack: AsyncExitStack | None = None

    async def initialize(self) -> None:
        if self._session is not None:
            return
        self._stack = AsyncExitStack()
        self._session = await self._stack.enter_async_context(self._session_factory())

    async def cleanup(self) -> None:
        if self._stack is not None:
            await self._stack.aclose()
        self._session = None
        self._stack = None

    @transactional
    async def execute(
        self,
        sql: str,
        params: dict[str, Any] | None = None,
    ) -> Result:
        assert self._session is not None, "RDBClient not initialized"
        return await self._session.execute(text(sql), params or {})
```

Notes:

- **Same lifecycle as the existing boilerplate client** (init/cleanup,
`@transactional` decorator). The difference is the public surface: the
current ORM-builder client exposes `insert(table, model)`, `find_by_id`,
`update`, etc.; this one exposes only `execute(sql, params)`.
- **One method intentionally.** `execute` covers reads, writes, upserts,
CTEs, and `RETURNING` clauses uniformly. There is no need for
domain-specific methods on the client itself — that is the repository's
job.
- `**@transactional` decorator** wraps commit/rollback at the call boundary.
Multi-statement transactions across repositories are achieved by entering
a transaction on the shared session before calling repository methods.

See also: [SQLAlchemy `text()` (textual SQL)](https://docs.sqlalchemy.org/en/20/core/connections.html#using-textual-sql).

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

## Repository Example

The application layer defines the `Protocol`. The persistence layer provides
the `Sql`* implementation that wires `RDBClient` and `SqlMapper` together.

```python
# application/users/repository.py
from typing import Protocol
from uuid import UUID

from .dto import UserDTO


class UserRepository(Protocol):
    async def get_by_id(self, user_id: UUID) -> UserDTO | None: ...
    async def find_by_email(self, email: str) -> UserDTO | None: ...
    async def create(self, email: str, name: str) -> UserDTO: ...
```

```python
# persistence/users/sql_user_repository.py
from pathlib import Path
from uuid import UUID

from application.users.dto import UserDTO
from persistence.rdb.rdb_client import RDBClient
from persistence.rdb.sql_mapper import SqlMapper


class SqlUserRepository:
    def __init__(self, rdb: RDBClient) -> None:
        self._rdb = rdb
        self._sql = SqlMapper(SqlUserRepository, Path(__file__).parent / "users.sql")

    async def get_by_id(self, user_id: UUID) -> UserDTO | None:
        result = await self._rdb.execute(self._sql["get_by_id"], {"id": user_id})
        row = result.mappings().first()
        return UserDTO.model_validate(row) if row else None

    async def find_by_email(self, email: str) -> UserDTO | None:
        result = await self._rdb.execute(self._sql["find_by_email"], {"email": email})
        row = result.mappings().first()
        return UserDTO.model_validate(row) if row else None

    async def create(self, email: str, name: str) -> UserDTO:
        result = await self._rdb.execute(
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

The flow:

```text
Application service
    ↓ depends on
UserRepository (Protocol, application layer)
    ↓ implemented by
SqlUserRepository (persistence layer)
    ↓ uses
RDBClient.execute(sql, params)  +  SqlMapper["..."]
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

Persistence is mocked at the `Protocol` boundary: a fake `MockUserRepository`
is overridden into the container, and the route handler exercises the full
stack (FastAPI → service → mock repository) without touching a real database.

```python
# tests/api/users/mock_repository.py
from uuid import UUID

from application.users.dto import UserDTO
from application.users.repository import UserRepository


class MockUserRepository:
    """Structurally satisfies the `UserRepository` Protocol — no inheritance."""

    def __init__(self) -> None:
        self._by_id: dict[UUID, UserDTO] = {}
        self._by_email: dict[str, UserDTO] = {}

    async def get_by_id(self, user_id: UUID) -> UserDTO | None:
        return self._by_id.get(user_id)

    async def find_by_email(self, email: str) -> UserDTO | None:
        return self._by_email.get(email)

    async def create(self, email: str, name: str) -> UserDTO:
        ...
```

Wire the mock into the DI container at test setup:

```python
container.user_repository.override(providers.Object(MockUserRepository()))
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
3. **No `fastapi.Depends` for sessions.** Sessions are persistence-layer
  resources injected via the DI container. Routes depend on services, not
   on `AsyncSession` or repositories.
4. **No sync APIs.** `Session`, `create_engine`, sync sessionmakers — none of
  them appear in this stack. Async only.
5. **No ORM query construction in application code.** No `session.query`,
  no `select(User).where(...)`, no `session.get(User, id)`. All queries
   live in `*.sql` files and execute through `RDBClient.execute`.
6. **No SQL strings inline in repositories.** All SQL lives in the
  sibling `*.sql` file and is loaded by `SqlMapper`. Inline strings
   bypass the name-match validation and scatter SQL across Python files.


