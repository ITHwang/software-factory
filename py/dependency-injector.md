# Dependency Injection (dependency-injector)

> Last updated: 2026-05-16

## TL;DR

How to wire async services, repositories, clients, and configuration in a FastAPI service with `dependency-injector`: declarative container, typed provider matrix (Singleton/Factory/Resource/Selector/Aggregate), async-aware resource lifecycles, and FastAPI `Depends` wiring.

**Use this when:**
- composing services + repositories + outbound clients into a FastAPI app
- loading multi-environment YAML configuration via a `Configuration` provider
- overriding providers in tests without rewriting wiring

**Don't use this for:**
- runtime-discoverable plug-ins — use a plugin loader, not DI

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Quick Reference](#quick-reference), [Lifecycle Scope Mapping](#lifecycle-scope-mapping), [When To Use](#when-to-use) |
| 2. Setup | [Install](#install), [Container Layout](#container-layout) |
| 3. Configure | [Configuration Provider](#configuration-provider), [YAML To Python Object Flow](#yaml-to-python-object-flow) |
| 4. Build | [Provider Types Deep Dive](#provider-types-deep-dive), [Async Resource Lifecycle](#async-resource-lifecycle) |
| 5. Integrate | [FastAPI Integration](#fastapi-integration), [Wiring Modules](#wiring-modules), [Typical Container Patterns](#typical-container-patterns) |
| 6. Operate | [Testing And Overrides](#testing-and-overrides), [Sync Variant](#sync-variant), [Production Checklist](#production-checklist), [Pitfalls](#pitfalls) |

## Quick Reference

| Provider | Role | When to use |
|----------|------|-------------|
| `Singleton` | Caches the first call result, reuses for the container's lifetime. Variants: `ThreadSafeSingleton` (locked, for threaded workers), `ThreadLocalSingleton` (per-thread), `ContextLocalSingleton` (per `contextvars` context). | App-scoped pure/shared objects without explicit teardown |
| `Factory` | New instance on every provider call | Request-owned or owned/transient objects; not request scope by itself |
| `Resource` | Init plus teardown lifecycle, sync or async | App-scoped clients/resources with `open/close`, `connect/disconnect` |
| `Configuration` | YAML / env / dict / pydantic-settings, dotted access | YAML + env vars + pydantic-settings, all as dotted-attribute access |
| `Callable` | Wraps a callable with bound dependencies | Plain function pipelines |
| `Coroutine` | Like `Callable` but awaits a coroutine | Async helper bindings |
| `List` | Aggregates providers into a `list[...]` | Plugin lists, tool registries |
| `Dict` | Aggregates providers into a `dict[str, ...]` | Named pipelines |
| `Aggregate` | Collection of named providers selected at call time | Strategy pattern — pick one of many at runtime |
| `Selector` | Picks one of several providers by config flag | Switching backends per env |
| `Object` | Wraps a literal value | Constants, sentinels |
| `Dependency` | Typed placeholder, must be overridden | Test doubles, abstract deps |
| `DependenciesContainer` | Cross-container injection placeholder | Layered containers |

See also: [dependency-injector docs](https://python-dependency-injector.ets-labs.org/), [repo](https://github.com/ets-labs/python-dependency-injector), [PyPI](https://pypi.org/project/dependency-injector/).

## Lifecycle Scope Mapping

Use the same scope names as `documentation-guidelines.md` in container examples and lifecycle diagrams.

| Scope | DI / FastAPI shape | Rule |
|---|---|---|
| **App Scope** | `providers.Singleton` or `providers.Resource`; resources are initialized and shut down from FastAPI lifespan. | One object per Python worker/process app lifespan. Use for shared clients, connection pools, compiled agents/graphs, model weights, and safe shared adapters. |
| **Request Scope** | Strict: FastAPI `Depends` / `yield` dependencies. Loose: one factory-built object created/received by the request path and passed downward. | Use strict `Depends` for request-derived state and cleanup-bound objects such as auth context, request IDs, DB sessions, tracing handlers, commit/rollback, and close/rollback semantics. |
| **Owned-Transient Scope** | `providers.Factory`, `providers.Callable`, direct constructors, or context managers. | New object per call or per owner. Cleanup-sensitive objects must use explicit cleanup, not implicit finalization. |

`Factory` is the common source of confusion: it guarantees **new per provider call**, not **one per request**. A factory-built service is request-owned only when the route/dependency resolution creates one service for the request and that service is passed through the call chain. Do not store request-owned objects on app-scoped singletons.

## When To Use

Use `dependency-injector` for any FastAPI service that owns more than two stateful clients (database engine, vector store, object store, cache) or that needs to swap implementations per environment. It is the single point that touches YAML and environment variables; everything below it receives typed values through providers.

Skip it for single-file scripts, throwaway CLI utilities, and pure compute libraries where there is nothing to inject. Skip it when you have exactly one client and one route — FastAPI's built-in `Depends` is enough.

This is the canonical reference for the Provider/Container pattern. Sibling docs (`./sqlalchemy.md`, `./py-aws-s3.md`, `./py-cache.md`, `./milvus.md`, `./py-auth.md`, `./langfuse.md`, `./langchain.md`, `./langgraph.md`, `./litellm.md`, `./py-file-processing.md`) carry the actual wiring snippets for their specific clients; the [Typical Container Patterns](#typical-container-patterns) section here covers the archetypes those wirings instantiate.

The library is Cython-compiled, ships mypy stubs in-tree, and is BSD-3-Clause licensed. Pin a compatible version in `pyproject.toml` and align with the project Python baseline in [`./py-setup-toolchains.md`](./py-setup-toolchains.md).

### Relationship to FastAPI Depends

FastAPI's `Depends` and `dependency-injector` cover overlapping but distinct ground.

- **FastAPI `Depends` owns HTTP request-state**: path/header/query/body parsing, request-scope objects bound to per-request cleanup (DB sessions via commit/rollback, auth context, request IDs). This is the **strict request scope** in `py-backend-architecture.md` and `py-tests.md`.
- **`dependency-injector` owns app-scope and owned-transient wiring**: shared clients, compiled artifacts, request-owned services built via `Factory`. A `Factory`-built service is request-owned only when the route resolves one and passes it through the call chain — it is not strict request scope.

A route typically uses both: `Depends(get_rdb_client)` for the request-scoped session, `Depends(Provide[Container.user_service])` for the request-owned service that consumes it. See [`./py-tests.md`](./py-tests.md) for how each is mocked.

## Install

```bash
uv add dependency-injector pyyaml
```

Optional integrations:

```bash
uv add pydantic-settings   # for config.from_pydantic(Settings())
```

`pyyaml` is only required when calling `config.from_yaml(...)`. `pydantic-settings` v2 is only required when wiring config from a `BaseSettings` object.

## Container Layout

Layered composition is the default. The `ApplicationContainer` is the **composition root** — it loads environment configuration, declares infrastructure sub-containers, and composes per-domain containers that consume them.

```text
ApplicationContainer
    ├── DatabaseContainer        (infra siblings)
    ├── CacheContainer
    ├── ClientsContainer
    └── UserContainer            (per-domain siblings)
    └── OrderContainer
```

```python
# containers/database.py
from dependency_injector import containers, providers

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from .clients import init_engine


class DatabaseContainer(containers.DeclarativeContainer):
    config = providers.Configuration()

    engine = providers.Resource(init_engine, url=config.url)
    session_factory = providers.Singleton(
        async_sessionmaker,
        bind=engine,
        class_=AsyncSession,
        expire_on_commit=False,
    )


# containers/cache.py
from dependency_injector import containers, providers

from .clients import init_redis


class CacheContainer(containers.DeclarativeContainer):
    config = providers.Configuration()
    redis = providers.Resource(init_redis, url=config.url)


# containers/clients.py
from dependency_injector import containers, providers

from .adapters import EmailSender


class ClientsContainer(containers.DeclarativeContainer):
    config = providers.Configuration()

    email_sender = providers.Singleton(
        EmailSender,
        api_key=config.email.api_key,
    )


# containers/users.py — per-domain container
from dependency_injector import containers, providers

from .repositories import PostgresUserRepository
from .services import UserService


class UserContainer(containers.DeclarativeContainer):
    database = providers.DependenciesContainer()
    cache = providers.DependenciesContainer()
    clients = providers.DependenciesContainer()

    user_repository = providers.Factory(PostgresUserRepository)
    user_service = providers.Factory(
        UserService,
        repo=user_repository,
        cache=cache.redis,
        email_sender=clients.email_sender,
    )


# containers/application.py — composition root
from dependency_injector import containers, providers

from .cache import CacheContainer
from .clients import ClientsContainer
from .database import DatabaseContainer
from .users import UserContainer


class ApplicationContainer(containers.DeclarativeContainer):
    config = providers.Configuration(yaml_files=["configs/dev.yaml"])

    database = providers.Container(DatabaseContainer, config=config.db)
    cache = providers.Container(CacheContainer, config=config.cache)
    clients = providers.Container(ClientsContainer, config=config.clients)

    users = providers.Container(
        UserContainer,
        database=database,
        cache=cache,
        clients=clients,
    )
```

Each domain container declares per-concern `providers.DependenciesContainer()` placeholders for the infra it needs; `ApplicationContainer` wires them by passing the actual sub-containers as keyword arguments to `providers.Container(...)`. The placeholders act as typed contracts — a domain container cannot reach into infra it did not declare.

`PostgresUserRepository` is stateless; the per-request `RDBClient` is passed by the route as a method argument via `Depends(get_rdb_client)`, not injected into the service or repository. See [Request-scoped infrastructure outside the container](#request-scoped-infrastructure-outside-the-container) and [`./sqlalchemy.md`](./sqlalchemy.md) for the `RDBClient` contract.

`DatabaseContainer` exposes `session_factory` so the request-scoped `get_rdb_client` FastAPI dep factory can build a fresh session per request. Domain containers do not declare `database = providers.DependenciesContainer()` unless they have a non-RDBClient reason to reach into it (e.g., domain-level migrations).

Each container subclass goes in its own module under `containers/`. Declaration order within a container does not matter — `dependency-injector` resolves dependencies lazily at provider call time.

### Routes consume specific providers, not the container itself

`ApplicationContainer` is a composition root — it's where sub-containers and providers are wired together. It should **not** be injected into routes or use-cases as a single dependency. Doing so turns the route into a Service Locator: its signature hides what it actually depends on, and the wiring becomes ad-hoc.

Anti-pattern (route receives the whole container):

```python
@router.get("/dashboard")
@inject
async def dashboard(
    container: ApplicationContainer = Depends(Provide[ApplicationContainer]),
):
    user = await container.users.user_service().get_current()
    orders = await container.orders.order_service().recent()
    return {"user": user, "orders": orders}
```

Recommended (route receives specific providers via dotted-path `Provide`):

```python
@router.get("/dashboard")
@inject
async def dashboard(
    user_service: UserService = Depends(Provide[ApplicationContainer.users.user_service]),
    order_service: OrderService = Depends(Provide[ApplicationContainer.orders.order_service]),
):
    user = await user_service.get_current()
    orders = await order_service.recent()
    return {"user": user, "orders": orders}
```

The dotted path through sub-containers is fully supported by `dependency-injector` wiring. The signature now declares exactly what the route depends on, which makes tests easier (you override the specific providers, not the whole container) and refactors safer (removing a service breaks the route signature, not silently at runtime).

### Minimal single-container alternative

For single-file scripts, throwaway CLI utilities, or one-route demos where there's nothing to layer:

```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration(yaml_files=["configs/dev.yaml"])

    redis = providers.Resource(init_redis, url=config.redis.url)
    email_sender = providers.Singleton(
        EmailSender,
        api_key=config.email.api_key,
    )

    user_service = providers.Factory(
        UserService,
        cache=redis,
        email_sender=email_sender,
    )
```

See also: [declarative container](https://python-dependency-injector.ets-labs.org/containers/declarative.html).

## Configuration Provider

`Configuration` is the bridge between YAML / env / pydantic-settings and the rest of the container. It exposes nested dotted attributes (`config.db.url`) that other providers reference at declaration time.

### Sources

| Source | Call | Notes |
|--------|------|-------|
| YAML at construction | `providers.Configuration(yaml_files=["configs/dev.yaml"])` | Loaded eagerly at build |
| YAML at runtime | `container.config.from_yaml("configs/dev.yaml")` | Merge recursively, last wins |
| Environment variable | `container.config.db.url.from_env("DATABASE_URL", required=True)` | Per-leaf override |
| Dictionary | `container.config.from_dict({...})` | Test fixtures |
| INI | `container.config.from_ini("configs/dev.ini")` | Legacy |
| pydantic-settings | `container.config.from_pydantic(Settings())` | v2 `BaseSettings` |

```python
container = Container()
container.config.from_yaml("configs/dev.yaml")
container.config.db.url.from_env("DATABASE_URL", required=True)
```

### Environment Configuration

We prefer one YAML file per environment, a mostly static provider graph, and startup-time wiring:

```text
configs/
    local.yaml
    test.yaml
    dev.yaml
    prod.yaml
```

Wire the active env's YAML in the app factory:

```python
import os

def create_app() -> FastAPI:
    container = ApplicationContainer()
    env = os.environ.get("APP_ENV", "local")
    container.config.from_yaml(f"configs/{env}.yaml")
    # ... rest of factory
```

The selected YAML determines database backend, cache backend, storage backend, external clients, and feature integrations — the provider graph is deterministic per environment.

Rationale:

- **Deterministic**: the live provider graph is known from one file, not a combination of runtime branches.
- **Debug-friendly**: misconfiguration failures point at a single config source.
- **Diagram-friendly**: architecture diagrams can be drawn from the YAML without runtime introspection.
- **Selector usually unnecessary**: when each env has its own YAML, the provider choice for that env is already fixed — no need for `Selector` for per-env switching. See [Strategy switch by config](#strategy-switch-by-config) for the cases where `Selector` is still appropriate.

Avoid excessive runtime provider branching.

### Type Coercion

`Configuration` stores values as strings or nested dicts. Coerce at the call site (chained on the consumer) or at the source (via `from_env`).

| Coercion | Example |
|----------|---------|
| `as_int()` | `config.api.port.as_int()` |
| `as_float()` | `config.http.backoff.as_float()` |
| `as_(t)` | `config.fee.as_(decimal.Decimal)` |
| `required()` | `config.db.url.required()` raises if not set; chain before `as_int()` |

```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    # Chained at the consumer side
    http_client = providers.Singleton(
        HttpClient,
        timeout=config.http.timeout.as_int(),
        backoff=config.http.backoff.as_float(),
    )

# Or coerced at the source via from_env:
container.config.api.port.from_env("API_PORT", as_=int, default=8080)
```

### Required And Strict

`required()` raises an error at injection time if the key is missing. It is the right way to fail fast on a misconfigured deployment.

```python
url = config.db.url.required()
port = config.api.port.required().as_int()
```

For container-level strictness, construct the configuration with `strict=True` so missing files or env vars raise immediately.

```python
config = providers.Configuration(strict=True)
```

### Env Interpolation In YAML

YAML loaded through `from_yaml` supports `${VAR}` and `${VAR:default}` interpolation.

```yaml
# configs/dev.yaml
db:
  url: ${DATABASE_URL:postgresql+asyncpg://localhost/dev}
redis:
  url: ${REDIS_URL:redis://localhost:6379/0}
api:
  timeout: ${HTTP_TIMEOUT:30}
```

To require every env var (no defaults silently swallowed) pass `envs_required=True` to `from_yaml`. To disable env interpolation entirely pass `envs_required=None`.

### pydantic-settings v2

Seed a `BaseSettings` instance at container construction; its fields fold into the configuration tree.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_nested_delimiter="__")
    db_url: str
    redis_url: str


class Container(containers.DeclarativeContainer):
    config = providers.Configuration(
        yaml_files=["configs/dev.yaml"],
        pydantic_settings=[Settings()],
    )
```

See also: [`Configuration` provider docs](https://python-dependency-injector.ets-labs.org/providers/configuration.html).

## YAML To Python Object Flow

YAML files are the deployment-time source of truth. The `Configuration` provider parses them, the env layer overrides individual leaves, and the rest of the container reads typed values out of the tree at provider call time.

```text
configs/dev.yaml
    db:
      url: ${DATABASE_URL:postgresql+asyncpg://localhost/dev}
    redis:
      url: ${REDIS_URL:redis://localhost:6379/0}
    api:
      timeout: ${HTTP_TIMEOUT:30}

           |
           v   container.config.from_yaml("configs/dev.yaml")
           |   container.config.db.url.from_env("DATABASE_URL", required=True)
           v

providers.Configuration
    |-- config.db.url        -> str
    |-- config.redis.url     -> str
    |-- config.api.timeout   -> .as_int() -> int

           |
           v   referenced at provider declaration
           v

providers.Resource(init_engine, url=config.db.url)
providers.Resource(init_redis,  url=config.redis.url)

           |
           v   await container.init_resources()
           v

Live AsyncEngine, Redis client - one app-scoped instance per worker process.
```

The container is the single point that touches YAML and env; everything below it receives typed objects through providers. Env vars may be hydrated from a secrets manager (AWS Secrets Manager, Doppler, Vault) at deploy time — the container reads decoded values, it doesn't fetch.

## Provider Types Deep Dive

### Singleton

Caches the first call and returns it for the lifetime of the container. Use it for clients that are safe to share across requests and have no per-request state.

```python
import aioboto3

class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    aioboto_session = providers.Singleton(aioboto3.Session)
```

`Singleton` is app-scoped and lazy: the object is built on the first provider call unless you explicitly resolve it during startup. `Singleton` itself is **not** thread-safe. Under threaded workers use `ThreadSafeSingleton` or `Resource`.

### Factory

Builds a new instance on every provider call. Use it for request-owned services/repositories and owned/transient helper objects when the caller controls the object lifetime. It is not automatically one-per-request; avoid calling the same factory repeatedly inside one request unless multiple independent instances are intended.

```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    user_repository = providers.Factory(PostgresUserRepository)
    user_service = providers.Factory(UserService, repo=user_repository)
```

### Resource

Owns an init-plus-teardown lifecycle. Sync resources accept a regular generator; async resources accept an async generator. When `init_resources()` / `shutdown_resources()` are driven by FastAPI lifespan, the yielded object is app-scoped for that Python worker/process.

```python
from collections.abc import AsyncIterator
from redis.asyncio import Redis, from_url


async def init_redis(url: str) -> AsyncIterator[Redis]:
    client = from_url(url, encoding="utf-8", decode_responses=True)
    try:
        yield client
    finally:
        await client.aclose()


class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    redis = providers.Resource(init_redis, url=config.redis.url)
```

### Selector

Picks one provider out of several based on a config switch. Useful for swapping a real client with a fake without rewriting downstream wiring.

```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    real_email = providers.Singleton(SesEmailClient, region=config.aws.region)
    mock_email = providers.Singleton(InMemoryEmailClient)

    email = providers.Selector(
        config.email.backend,
        ses=real_email,
        memory=mock_email,
    )
```

`config.email.backend` resolves to `"ses"` or `"memory"` and the selector dispatches accordingly.

### Dependency

Typed placeholder that must be overridden before the provider is called. Ideal for test doubles and abstract dependencies.

```python
class Container(containers.DeclarativeContainer):
    clock: providers.Provider[Clock] = providers.Dependency(instance_of=Clock)
    user_service = providers.Factory(UserService, clock=clock)
```

In tests:

```python
container.clock.override(providers.Object(MockClock(now=datetime(2026, 1, 1))))
```

See also: [provider types index](https://python-dependency-injector.ets-labs.org/providers/index.html), [Singleton](https://python-dependency-injector.ets-labs.org/providers/singleton.html), [Factory](https://python-dependency-injector.ets-labs.org/providers/factory.html), [Selector](https://python-dependency-injector.ets-labs.org/providers/selector.html), [Aggregate](https://python-dependency-injector.ets-labs.org/providers/aggregate.html), [Dependency](https://python-dependency-injector.ets-labs.org/providers/dependency.html).

## Async Resource Lifecycle

Every async client that owns external resources is a `Resource` backed by an async generator. The generator yields exactly once. Code before `yield` runs at `init_resources()`. Code after `yield` (typically inside a `try/finally`) runs at `shutdown_resources()`.

**Purpose**: define a reusable app-scoped init-plus-teardown lifecycle owner so each stateful client is created once per worker process, shared across consumers, and torn down deterministically when the app shuts down.

**Responsibilities**:
- Manage the init/teardown lifecycle of resources with async generators (`yield`).
- Ensure single-resource sharing across consumers within the app-scoped container.
- Tie cleanup to FastAPI lifespan / container shutdown.

```python
from collections.abc import AsyncIterator
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine


async def init_engine(url: str) -> AsyncIterator[AsyncEngine]:
    engine = create_async_engine(url, echo=False, pool_pre_ping=True)
    try:
        yield engine
    finally:
        await engine.dispose()
```

Drive the lifecycle from FastAPI's `lifespan` context:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

from .containers import Container


@asynccontextmanager
async def lifespan(app: FastAPI):
    container: Container = app.state.container
    await container.init_resources()
    try:
        yield
    finally:
        await container.shutdown_resources()
```

If any provider in the container is async, you **must** `await` both `init_resources()` and `shutdown_resources()`. The container's lifecycle methods return coroutines when any resource is async; calling them synchronously raises at runtime.

`dependency-injector` also ships a Starlette adapter at `dependency_injector.ext.starlette.Lifespan` that uses a `Self()` provider to wire the container into FastAPI's lifespan automatically. Prefer the explicit `@asynccontextmanager` pattern above for clarity; reach for the adapter only when you already use `Self()` elsewhere.

See also: [Resource provider](https://python-dependency-injector.ets-labs.org/providers/resource.html).

## FastAPI Integration

Wiring connects the container to FastAPI route handlers. The pattern is `@inject` plus an `Annotated[T, Depends(Provide[Container.x])]` parameter.

```python
# endpoints.py
from typing import Annotated

from dependency_injector.wiring import Provide, inject
from fastapi import APIRouter, Depends

from .containers import Container
from .services import UserService

router = APIRouter()


@router.get("/users/{user_id}")
@inject
async def get_user(
    user_id: int,
    service: Annotated[UserService, Depends(Provide[Container.user_service])],
) -> dict[str, object]:
    return await service.get(user_id)
```

Decorator order: `@inject` must sit directly above `async def` (closest to the function); the FastAPI route decorator (`@router.get`) sits above `@inject`. In source order, top to bottom: `@router.get(...)` → `@inject` → `async def`. Reversing them silently breaks injection — `@inject` patches the function it wraps, so the route decorator must wrap the already-injected function, not the other way around.

App factory tying it all together:

```python
# main.py
from contextlib import asynccontextmanager

from fastapi import FastAPI

from .containers import Container
from .endpoints import router


@asynccontextmanager
async def lifespan(app: FastAPI):
    container: Container = app.state.container
    await container.init_resources()
    try:
        yield
    finally:
        await container.shutdown_resources()


def create_app() -> FastAPI:
    container = Container()
    container.config.from_yaml("configs/dev.yaml")
    container.config.db.url.from_env("DATABASE_URL", required=True)
    container.wire(modules=["myapp.endpoints"])

    app = FastAPI(lifespan=lifespan)
    app.state.container = container
    app.include_router(router)
    return app
```

`app.state.container` is the conventional handle for tests and extensions to reach the container.

See also: [FastAPI example](https://python-dependency-injector.ets-labs.org/examples/fastapi.html).

## Wiring Modules

`container.wire(modules=[...])` patches every `@inject`-decorated function in the listed modules so `Provide[...]` markers resolve to live providers. Wire once at boot, before requests reach any route.

```python
container.wire(modules=[
    "myapp.endpoints",
    "myapp.dependencies",
    "myapp.background",
])
```

Alternative: declare a `WiringConfiguration` inside the container so it auto-wires when instantiated.

```python
class Container(containers.DeclarativeContainer):
    wiring_config = containers.WiringConfiguration(
        modules=["myapp.endpoints"],
        auto_wire=True,
    )
```

`auto_wire=True` (the default) wires at container instantiation. Prefer the explicit `container.wire(...)` call in your app factory — it is easier to reason about than wiring as a side effect of construction.

`wire(...)` also accepts `packages=[...]` for recursive package-wide discovery and already-imported module objects (not just dotted strings).

Discipline: keep wiring in `main.py` / `application.py`, never inside business modules. That avoids cyclic imports between containers and the modules they wire.

See also: [wiring docs](https://python-dependency-injector.ets-labs.org/wiring.html).

## Typical Container Patterns

When you add a new tool to the container, identify which archetype matches and copy the shape. Deviate only when the tool's lifecycle or sharing semantics force it.

### App-scoped client with init/teardown

`Resource` + async generator for any client that owns external state and needs deterministic teardown (DB engine, gRPC client, HTTP session pool).

```python
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine


async def init_engine(url: str) -> AsyncIterator[AsyncEngine]:
    engine = create_async_engine(url, echo=False, pool_pre_ping=True)
    try:
        yield engine
    finally:
        await engine.dispose()


class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    engine = providers.Resource(init_engine, url=config.db.url)
```

### App-scoped stateless singleton

`Singleton` for a stateful-but-safe-to-share adapter that owns construction config but no init/teardown lifecycle (JWT verifier, hashing helper, prompt-cached summarizer).

```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    jwt_verifier = providers.Singleton(
        JWTVerifier,
        issuer=config.auth.issuer,
        audience=config.auth.audience,
        jwks_url=config.auth.jwks_url,
    )
```

### Request-owned service composition

`Factory` for a service that consumes app-scoped clients but is built fresh per resolution. The service is request-owned only because the route resolves one and passes it down — there is no per-request scope hook.

```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    redis = providers.Resource(init_redis, url=config.redis.url)

    user_repository = providers.Factory(PostgresUserRepository)
    user_service = providers.Factory(
        UserService,
        repo=user_repository,
        cache=redis,
    )
```

`PostgresUserRepository` is stateless; the per-request `RDBClient` is passed by the route as a method argument, not injected. See [Request-scoped infrastructure outside the container](#request-scoped-infrastructure-outside-the-container) below and [`./sqlalchemy.md`](./sqlalchemy.md) for the `RDBClient` contract.

### Strategy switch by config

`Selector` picks one of several pre-declared providers based on a config key. Use it for **within-env runtime variability** when multiple interchangeable adapters satisfy the same port in the same deployment. If per-env YAML already determines the provider graph (the recommended approach — see [Environment Configuration](#environment-configuration)), `Selector` is usually unnecessary for the per-env case.

```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    backend_a = providers.Singleton(BackendA, region=config.backend.region)
    backend_b = providers.Singleton(BackendB, mode=config.backend.mode)

    backend = providers.Selector(
        config.backend.kind,        # "a" | "b"
        a=backend_a,
        b=backend_b,
    )
```

Don't use `Selector` for:

- **business-rule dispatch** (e.g., choosing a pricing strategy by customer tier) — build a strategy/router service in the domain layer instead
- **fallback chains** (e.g., try primary, fall back to secondary) — wrap the fallback logic in a small class and inject it as a `Factory`
- **retry logic** — same as fallback chains
- **request-level dynamic routing** — that's a strategy/router service in the domain layer, not provider config

### Compiled artifact built once

`Singleton` of a build call for expensive artifacts (compiled graph topologies, agent build with bound tools, loaded model weights). Build once at startup; flow per-request context through the call signature, not through a new instance.

```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    agent_factory = providers.Factory(AgentFactory, settings=config.agent)
    agent = providers.Singleton(lambda factory: factory.build(), agent_factory)
```

Compiled topologies, agent graphs, or model-loading work goes in `Singleton`, not `Factory` — building is expensive, per-request context flows separately.

### Request-scoped infrastructure outside the container

Per-request transactional infrastructure (`RDBClient` wrapping a fresh `AsyncSession`) is **not** a container provider. It's yielded via a FastAPI dependency factory that owns the commit-on-success / rollback-on-exception contract. The container provides the `session_factory`; the FastAPI dep owns the per-request session.

```python
# api/dependencies/rdb.py — FastAPI dependency factory
from collections.abc import AsyncIterator

from dependency_injector.wiring import Provide, inject
from fastapi import Depends

from .containers import Container
from .rdb import RDBClient


@inject
async def get_rdb_client(
    session_factory=Depends(Provide[Container.session_factory]),
) -> AsyncIterator[RDBClient]:
    async with session_factory() as session:
        rdb = RDBClient(session)
        try:
            yield rdb
            await session.commit()
        except Exception:
            await session.rollback()
            raise


# api/routes/users.py — route with both FastAPI Depends and DI Provide
@router.get("/users/{user_id}")
@inject
async def get_user(
    user_id: UUID,
    rdb: RDBClient = Depends(get_rdb_client),
    service: UserService = Depends(Provide[Container.user_service]),
):
    return await service.get(rdb, user_id)
```

The `await session.commit()` at yield exit is an **infrastructure-level lifecycle commit**, not a per-call hidden commit.

### Archetype decision table

| Archetype | Provider shape | Use when |
|---|---|---|
| App-scoped client with init/teardown | `Resource` + async generator | Client owns external state (connections, gRPC, HTTP session) and needs deterministic cleanup |
| App-scoped stateless singleton | `Singleton(Cls, **config)` | Stateful-but-safe-to-share adapter with no init/teardown |
| Request-owned service composition | `Factory(Service, deps=...)` | Service consumes app-scoped clients and is built per resolution |
| Strategy switch by config | `Selector(config.key, a=..., b=...)` | Two or more adapters satisfy the same port; choice is per-environment |
| Compiled artifact built once | `Singleton(lambda f: f.build(), factory)` | Build is expensive; per-request context flows through call signature |
| Request-scoped infrastructure outside the container | FastAPI dependency factory (not a provider) | Per-request session/txn with commit/rollback semantics |

See also: [FastAPI + SQLAlchemy example](https://python-dependency-injector.ets-labs.org/examples/fastapi-sqlalchemy.html), [FastAPI + Redis example](https://python-dependency-injector.ets-labs.org/examples/fastapi-redis.html), [`miniapps` example repo](https://github.com/ets-labs/python-dependency-injector/tree/master/examples/miniapps).

## Testing And Overrides

Every provider can be replaced with a stand-in. `override` is the primary way to swap a port's real adapter for a test double that satisfies the same `Protocol` — per `py-backend-architecture.md`, ports live in the business layer and adapters in `adapters/` or `clients/`. Mocks injected via `override` implement the port `Protocol`s, not the adapter classes.

For the FastAPI route-test fixture (canonical shape with `LifespanManager(app)` and provider overrides) see [`./py-tests.md`](./py-tests.md). This section covers the `dependency-injector`-specific mechanics: the `override` API, the `Dependency` placeholder, and `reset_override` discipline.

`override` also works as a context manager, scoped to a single test:

```python
# Clock is a port — Protocol in the business layer.
# MockClock satisfies the same Protocol.
def test_user_service_uses_mock_clock(container):
    with container.clock.override(providers.Object(MockClock())):
        service = container.user_service()
        assert isinstance(service.clock, MockClock)
```

`providers.Dependency(instance_of=...)` declares a placeholder that **must** be overridden before use, so the test author cannot accidentally exercise the real client.

```python
class Container(containers.DeclarativeContainer):
    clock: providers.Provider[Clock] = providers.Dependency(instance_of=Clock)


container = Container()
container.clock.override(providers.Object(MockClock()))   # required before container.clock()
```

Always `reset_override()` (or use the context-manager form). A leaked override pollutes every later test.

See also: [provider overriding](https://python-dependency-injector.ets-labs.org/providers/overriding.html).

## Sync Variant

The sync story is identical except `Resource` accepts a regular generator and the container exposes synchronous lifecycle methods.

```python
from collections.abc import Iterator
from sqlalchemy import Engine, create_engine


def init_sync_engine(url: str) -> Iterator[Engine]:
    engine = create_engine(url, pool_pre_ping=True)
    try:
        yield engine
    finally:
        engine.dispose()


class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    engine = providers.Resource(init_sync_engine, url=config.db.url)


container = Container()
container.init_resources()
try:
    ...
finally:
    container.shutdown_resources()
```

If any provider in a container is async, the **entire** container must be driven asynchronously. Mixing sync `init_resources()` with async resources raises at runtime.

## Production Checklist

| Item | Setting |
|------|---------|
| Lifecycle | `await container.init_resources()` and `await container.shutdown_resources()` driven from FastAPI `lifespan` |
| Config layering | `from_yaml(...)` for defaults, `from_env(..., required=True)` for secrets and per-environment overrides |
| Secrets | Pull from env, never check into YAML |
| Wiring | One `container.wire(modules=[...])` call in the app factory; never inside business modules |
| Container handle | `app.state.container = container` so tests and extensions can reach it |
| Threading | Use `ThreadSafeSingleton` (or `Resource`) under threaded workers; plain `Singleton` is not thread-safe |
| Failure mode | `config.x.required()` on every must-have key so misconfig fails at boot, not on first request |
| Tests | In API tests, override port providers with mocks that satisfy the port Protocol; use real external resources only in a separate integration suite |

## Pitfalls

- **Forgetting to await teardown.** `container.shutdown_resources()` returns a coroutine when any resource is async. Awaiting it inside `lifespan`'s `finally` block prevents connection leaks at shutdown.
- **Using `Singleton` under threads.** Plain `Singleton` is not thread-safe. Switch to `ThreadSafeSingleton` or model the client as a `Resource` so the container owns its lifecycle.
- **Reading config outside an injection site.** `container.config.db.url` returns a `Provider`, not a string. Outside an injection (for example inside a pure-Python helper) call it: `container.config.db.url()`. Better: declare a provider that consumes `config.db.url` and inject that.
- **Wiring after route registration.** `@inject` patches functions when `wire(modules=[...])` runs. If routes are imported and registered before wiring, the markers stay unresolved. Wire in the app factory before `app.include_router(...)`.
- **Cyclic imports.** Importing the container from inside a business module that the container also wires creates a cycle. Keep wiring in `main.py` / `application.py`; modules that get wired should depend only on `Provide` and protocol types.
- **Forgetting `reset_override()` in tests.** A leaked override pollutes every later test. Use the context-manager form (`with container.x.override(mock):`) or call `container.unwire()` plus `container.reset_override()` in a fixture's teardown.
- **Putting secrets in YAML.** YAML is for shape; env is for values. `${VAR:default}` interpolation lets you keep the shape in YAML and the secret in the environment.
- **Mixing async and sync resources in one container.** If any provider is async, the whole container must be driven asynchronously. Calling sync `init_resources()` on an async container raises.
- **`@inject` decorator order.** It must be the innermost decorator on a route handler — between the route decorator and the function itself. Putting it outside the route decorator silently disables wiring.
- **Calling `container.wire(...)` more than once per process.** It is idempotent for repeat module lists but expensive. Wire once at boot.
- **Confusing `Selector` with fallback chains.** `Selector` switches the entire provider implementation by config key. It does not chain fallbacks. For fallback chains, build a small wrapper class and inject it as a `Factory`.
