# Python API Tests

> Last updated: 2026-05-16

## TL;DR

How to write FastAPI API tests: exercise the HTTP boundary, routing, validation, services, and DI wiring real; mock only persistence repositories and outbound adapters.

**Use this when:**
- writing API tests for a FastAPI service
- setting up `httpx.AsyncClient` + `LifespanManager` + DI overrides
- deciding what to mock vs. what to keep real

**Don't use this for:**
- graph-functional tests with `LLMToolEmulator` → `./langgraph.md#testing`
- tooling / CI setup → `./py-setup-toolchains.md`
- testing repository / SQL row internals → write repository unit tests separately

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Testing Boundary](#testing-boundary), [Quick Reference](#quick-reference) |
| 2. Setup | [FastAPI Lifespan And `asgi-lifespan`](#fastapi-lifespan-and-asgi-lifespan), [Target Shape](#target-shape) |
| 3. Build | [Dependency Injector Override](#dependency-injector-override), [FastAPI Dependency Override (Strict Request Scope)](#fastapi-dependency-override-strict-request-scope), [Outbound HTTP Adapters](#outbound-http-adapters) |
| 4. Operate | [What To Assert](#what-to-assert), [Anti-Patterns](#anti-patterns) |

## Testing Boundary

FastAPI API tests verify business logic and domain-model behavior while replacing the **outbound-infrastructure** layer — the non-deterministic edges (DB, external APIs, JWT verification) — with deterministic mocks. The mock classes implement business-layer Protocols (defined in `ports/`); they are stand-ins for the adapters, not for SDK calls.

| Boundary | What It Owns | Test Double |
|----------|--------------|-------------|
| **Outbound infrastructure** | Adapters and clients implementing business-layer Protocols from `ports/`. Sub-categories: <br>• Persistence (repositories) — `UserRepository`, `OrderRepository`, … <br>• External APIs / SDK wrappers — `EmailSender`, `TextSummarizer`, `EmbeddingGenerator`, … <br>• Auth verifier — `JWTVerifier` (behind `Depends(get_current_user)`) | `Mock*` class implementing the same Protocol: <br>• `MockUserRepository`, `MockOrderRepository`, … <br>• `MockEmailSender`, `MockTextSummarizer`, … <br>• `MockJWTVerifier`; see [`./py-auth.md#testing`](./py-auth.md#testing) |

All three sub-categories share the same architectural seam. The mechanics differ — persistence and most external adapters override the DI container provider (see [Dependency Injector Override](#dependency-injector-override)); the auth-verifier path has two valid override points because it sits behind a FastAPI `Depends` exception (see [FastAPI Dependency Override (Strict Request Scope)](#fastapi-dependency-override-strict-request-scope)).

### Where mocks live

Tests mirror the source folder structure:

```text
app/users/services/user_service.py   →  tests/users/services/test_user_service.py
app/users/apis/router.py             →  tests/users/apis/test_router.py
```

Mock classes live **inline in the test file** that uses them — defined alongside the fixture or test function. Don't pre-emptively centralize into `tests/mocks/`; lift to a shared module only when the same mock is genuinely reused across multiple test files.

Everything else in the request path should be real: routing, validation,
dependency wiring, service/use-case logic, exception handlers, and response
serialization.

## Quick Reference

| Need | Default |
|------|---------|
| Test runner | `pytest` |
| Async test support | `pytest-asyncio` with `asyncio_mode = auto` |
| HTTP client | `httpx.AsyncClient` |
| App transport | `httpx.ASGITransport(app=app)` |
| FastAPI dependency overrides | `app.dependency_overrides[...] = ...` |
| Dependency Injector overrides | `container.provider.override(providers.Object(fake))` |
| Mock scope | Outbound-infrastructure dependencies only (adapters/clients implementing business-layer Protocols) |

Install:

```bash
uv add pytest pytest-asyncio httpx
```

Optional when the app relies on FastAPI lifespan startup/shutdown (see
[FastAPI Lifespan And `asgi-lifespan`](#fastapi-lifespan-and-asgi-lifespan)):

```bash
uv add asgi-lifespan
```

## FastAPI Lifespan And `asgi-lifespan`

This section is about the **App Scope** boundary — startup runs once per worker process, shutdown runs at process exit. `LifespanManager` exercises that boundary inside tests.

`httpx.AsyncClient` (with `ASGITransport`) does **not** drive ASGI lifespan events — HTTPX's team has declared lifespan orchestration out of scope for the HTTP client. Without help, requests work but `startup` / `shutdown` never run, so any resources initialized in lifespan (db pools, redis, model loading, DI containers) are absent. `asgi-lifespan`'s `LifespanManager` fills that gap by sending the ASGI startup event before the test and shutdown after.

```python
from asgi_lifespan import LifespanManager
from httpx import ASGITransport, AsyncClient


async def test_health_with_lifespan(app) -> None:
    async with LifespanManager(app):
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            response = await client.get("/health")
    assert response.status_code == 200
```

| Scenario | Approach |
|----------|----------|
| Sync test | `with TestClient(app)` (runs lifespan automatically as a context manager) |
| Async test | `AsyncClient` + `ASGITransport` + `LifespanManager` |
| Production app lifecycle | `FastAPI(lifespan=lifespan)` |
| Manual ASGI lifecycle control | `asgi-lifespan` |

`asgi-lifespan` is in maintenance-only mode (last release v2.1.0 in March 2023) but remains the canonical pattern in FastAPI's 2026 official docs. `httpx`'s team has declared lifespan orchestration out of scope for the HTTP client, so no native replacement is expected. Monitor the library; if a community fork or native httpx support emerges, the recipe will change.

See also: [FastAPI async tests](https://fastapi.tiangolo.com/advanced/async-tests/), [HTTPX ASGI transport](https://www.python-httpx.org/advanced/transports/), [asgi-lifespan](https://github.com/florimondmanca/asgi-lifespan).

## Target Shape

The route should depend on a service/use case. The service should depend on a
repository and any outbound adapters. Tests replace the repository and adapter,
then call the real API.

```text
HTTP request
  -> FastAPI route/controller
  -> service/use case
  -> mocked outbound infrastructure   # adapters / clients implementing
                                       # business-layer Protocols
                                       # (Mock* class per Protocol)
```

This keeps the test focused on what the backend can prove: request validation,
status codes, response bodies, orchestration, and error mapping. It does not try
to prove SQL syntax, S3 behavior, or third-party API contracts in every API
test.

The fixture sets up per-test state (test-scoped); within each test, `await client.get(...)` exercises one **Request Scope** — the FastAPI `Depends` / `yield` chain runs, services run, mocks record calls, and the response unwinds the chain.

## Dependency Injector Override

**This is the canonical pattern for mocking the outbound-infrastructure boundary.** Routes obtain services via `Depends(Provide[Container.user_service])`; tests override the provider with `container.x.override(providers.Object(mock))`. The mock is a class implementing the same business-layer Protocol the real adapter implements — the SDK / library / external resource the adapter would otherwise call is irrelevant. The fixture replaces those providers, exercises the real route + DI chain, and asserts the recorded mock calls.

```python
# app/users/ports.py — domain ports for the users package.
# (Per py-backend-architecture.md, ports live under app/<domain>/ports/;
#  this example combines two ports in one comment block for brevity.)
from typing import Protocol


class UserRepository(Protocol):
    async def create(self, email: str) -> dict[str, object]: ...


class EmailSender(Protocol):
    async def send_welcome(self, email: str) -> None: ...
```

```python
# tests/test_users_api.py
from collections.abc import AsyncIterator

import pytest
import pytest_asyncio
from asgi_lifespan import LifespanManager
from dependency_injector import providers
from httpx import ASGITransport, AsyncClient

from app.containers import Container
from app.main import create_app_with_container


class MockUserRepository:
    def __init__(self) -> None:
        self.created_emails: list[str] = []
        self.raise_duplicate = False

    async def create(self, email: str) -> dict[str, object]:
        if self.raise_duplicate:
            raise ValueError("duplicate_email")
        self.created_emails.append(email)
        return {"id": "usr_01J", "email": email}


class MockEmailSender:
    def __init__(self) -> None:
        self.welcome_emails: list[str] = []

    async def send_welcome(self, email: str) -> None:
        self.welcome_emails.append(email)


@pytest.fixture
def repository() -> MockUserRepository:
    return MockUserRepository()


@pytest.fixture
def email_sender() -> MockEmailSender:
    return MockEmailSender()
```

`LifespanManager` is included by default — skip it only when your fixture overrides every container provider AND the app does no other lifespan work (logging, signal handlers, config validation). The default-correct choice is to leave it in.

```python
@pytest_asyncio.fixture
async def client(
    repository: MockUserRepository,
    email_sender: MockEmailSender,
) -> AsyncIterator[AsyncClient]:
    container = Container()
    container.user_repository.override(providers.Object(repository))
    container.email_sender.override(providers.Object(email_sender))

    app = create_app_with_container(container)
    async with LifespanManager(app):
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            yield client

    container.user_repository.reset_override()
    container.email_sender.reset_override()
    container.unwire()
```

Success path:

```python
async def test_create_user_returns_201_and_sends_welcome_email(
    client: AsyncClient,
    repository: MockUserRepository,
    email_sender: MockEmailSender,
) -> None:
    response = await client.post("/users", json={"email": "a@example.com"})

    assert response.status_code == 201
    assert response.json() == {"id": "usr_01J", "email": "a@example.com"}
    assert repository.created_emails == ["a@example.com"]
    assert email_sender.welcome_emails == ["a@example.com"]
```

Error path:

```python
async def test_duplicate_email_returns_409_and_skips_sender(
    client: AsyncClient,
    repository: MockUserRepository,
    email_sender: MockEmailSender,
) -> None:
    repository.raise_duplicate = True

    response = await client.post("/users", json={"email": "a@example.com"})

    assert response.status_code == 409
    assert response.json()["error_code"] == "DUPLICATE_EMAIL"
    assert email_sender.welcome_emails == []
```

The mock repository records the persistence call. The mock sender records the
external side effect. The test does not import SQLAlchemy rows, open a database
session, or call the real email service.

Prefer explicit mock classes over `AsyncMock` for business-level API tests. Mocks
make state and behavior obvious. Use `AsyncMock` when the test only needs to
assert one call and a hand-written mock would be more code than signal.

```python
from unittest.mock import AsyncMock


async def test_email_sender_called_once(client: AsyncClient) -> None:
    email_sender = AsyncMock()
    email_sender.send_welcome.return_value = None

    # Override the sender provider with `email_sender`, then call the API.
    ...

    email_sender.send_welcome.assert_awaited_once_with("a@example.com")
```

See also: [Dependency Injector provider overriding](https://python-dependency-injector.ets-labs.org/providers/overriding.html).

## FastAPI Dependency Override (Strict Request Scope)

Use `app.dependency_overrides` only for the two strict-request-scope dependencies FastAPI owns directly:

- `Depends(get_rdb_client)` — per-request transactional RDB session.
- `Depends(get_current_user)` — auth context derived from the bearer token.

Service / repository / adapter overrides belong in [Dependency Injector Override](#dependency-injector-override). Per [`./py-backend-architecture.md#services-and-fastapi`](./py-backend-architecture.md#services-and-fastapi), routes hold the DI container by default; these two are the documented exceptions.

These two share an override **mechanism** (FastAPI `Depends` for strict request scope) but represent different architectural **categories** per [`./py-backend-architecture.md`](./py-backend-architecture.md):

- `JWTVerifier` is a **port** — a `Protocol` in the business layer's `ports/`. Mocking it crosses the same outbound-infrastructure boundary as the previous section.
- `RDBClient` is a **low-level client** — domain-agnostic infrastructure in `clients/`, providing session/transaction lifecycle rather than a business capability.

Both use FastAPI `Depends(...)` overrides because their lifecycle is tied to the request itself, not the app-scoped container. For the auth verifier specifically, you have a choice: override the FastAPI `Depends` function directly (shown here) **or** override `container.jwt_verifier` if the test wants to exercise the verifier's own logic — see [`./py-auth.md#testing`](./py-auth.md#testing) for the `MockJWTVerifier` variant.

`get_rdb_client` override:

```python
from app.clients.rdb import get_rdb_client


class MockRDBClient:
    async def execute(
        self, sql: str, params: dict[str, object] | None = None
    ) -> object: ...
    async def commit(self) -> None: ...
    async def rollback(self) -> None: ...


app.dependency_overrides[get_rdb_client] = lambda: MockRDBClient()
```

Strict-scope overrides install on `app.dependency_overrides[fn]` and must be removed in teardown so they don't leak across tests. The pattern below shows the canonical fixture shape — assign in setup, `pop` in `finally`. The same pattern applies to `get_current_user`.

```python
@pytest_asyncio.fixture
async def app_with_rdb_override() -> AsyncIterator[FastAPI]:
    container = Container()
    app = create_app_with_container(container)
    app.dependency_overrides[get_rdb_client] = lambda: MockRDBClient()
    try:
        async with LifespanManager(app):
            yield app
    finally:
        app.dependency_overrides.pop(get_rdb_client, None)
```

See [`./sqlalchemy.md`](./sqlalchemy.md) for the canonical session-lifecycle test pattern.

`get_current_user` override:

```python
from app.auth import CurrentUser
from app.dependencies import get_current_user


def _fake_user() -> CurrentUser:
    return CurrentUser(
        subject="user-123",
        username="user-123",
        scopes=frozenset({"documents:read"}),
        groups=frozenset(),
        claims={},
    )


app.dependency_overrides[get_current_user] = _fake_user
```

The alternative is to override the verifier behind the dependency — see [`./py-auth.md#testing`](./py-auth.md#testing) for the `MockJWTVerifier` pattern, used when the test wants to exercise the verifier's own logic.

Always clear `app.dependency_overrides` in fixture teardown so overrides do not leak across tests.

See also: [FastAPI dependency overrides](https://fastapi.tiangolo.com/advanced/testing-dependencies/).

## Outbound HTTP Adapters

**Note**: HTTPX `MockTransport` is for testing the adapter's wire-protocol behavior — its **internal** contract (the adapter's internal contract with the remote service). In API tests you mock the adapter itself by implementing its Protocol; never let HTTPX `MockTransport` leak into API tests as a substitute for Protocol-level mocking.

If the adapter itself wraps HTTPX, test the adapter separately with
`httpx.MockTransport`. API tests should usually mock the adapter object, not the
HTTP transport below it.

```python
import httpx


def handler(request: httpx.Request) -> httpx.Response:
    assert request.url.path == "/v1/messages"
    return httpx.Response(202, json={"accepted": True})


transport = httpx.MockTransport(handler)


async def test_email_sender_sends_request() -> None:
    async with httpx.AsyncClient(
        transport=transport,
        base_url="https://api.example",
    ) as http:
        email_sender = EmailSender(http)
        await email_sender.send_welcome("a@example.com")
```

Use this pattern for adapter tests, contract tests, or local simulations of an
external API. In API tests, replace `EmailSender` with a mock and keep the HTTP
transport out of the test.

See also: [HTTPX mock transport](https://www.python-httpx.org/advanced/transports/#mock-transports).

## What To Assert

Assert the behavior visible at the HTTP boundary plus the expected calls across
the two mocked boundaries.

| Scenario | Assertions |
|----------|------------|
| Success | Status code, response JSON, repository write/read, expected sender call |
| Validation error | `422`, response error shape, no repository or adapter call |
| Persistence error | Mapped status code, response error code, no adapter call when the write failed |
| Adapter error | Chosen policy: propagate `502/503`, return accepted status, or record retry intent |
| Authorization | Status code, response body, no persistence/adapter call when denied |

Keep assertions tight. Do not snapshot whole responses unless the exact full
shape is the public contract.

## Anti-Patterns

- **Opening a real database in every API test.** That turns a controller/service
  test into an integration test and makes failures harder to locate.
- **Mocking internal helper functions.** Mock only at the outbound-infrastructure
  boundary — i.e., a class implementing a business-layer Protocol. If a helper
  is hard to run in an API test, it probably needs to be lifted behind a
  Protocol-defined port.
- **Calling external services from API tests.** Network calls make tests slow,
  flaky, and expensive. Put real external calls in a small contract or smoke
  suite.
- **Leaving overrides installed.** Always clear `app.dependency_overrides` or
  reset provider overrides in fixture teardown.
- **Testing SQLAlchemy / ORM rows through the API.** API tests assert DTO/JSON behavior.
  Repository tests can cover rows and SQL separately when that becomes useful.
