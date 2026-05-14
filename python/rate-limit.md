# Rate Limiting

> Last updated: 2026-05-14

## TL;DR

Rate-limit a FastAPI service: per-IP / per-user / per-API-key quotas, shared counters across routes, and a Redis backend that survives multi-worker / multi-instance deployments. The reference binding is [slowapi](https://slowapi.readthedocs.io/) — a FastAPI/Starlette wrapper around the [`limits`](https://limits.readthedocs.io/) library that owns the actual algorithms and storage backends. This doc is policy-named (rate limiting), not library-named; swap the binding if the runtime changes, the policy guidance carries over.

**Use this when:**
- throttling per-IP, per-user, or per-API-key on FastAPI routes
- protecting a hot or expensive endpoint from abuse
- you need shared limits across multiple routes or a storage backend (Redis)

**Don't use this for:**
- backpressure on outbound calls → use `asyncio.Semaphore` at the call site
- auth / quota enforcement → `./cognito-auth.md`
- cost budgeting inside an LLM agent loop → `ModelCallLimitMiddleware` in `./langchain.md`

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Quick Reference](#quick-reference), [When To Use](#when-to-use) |
| 2. Setup | [Install](#install), [Wire Up The Limiter](#wire-up-the-limiter) ([Limiter object](#limiter-object), [Middleware vs Decorator](#middleware-vs-decorator), [Wiring sequence](#wiring-sequence)) |
| 3. Apply Limits | [Apply Limits To Routes](#apply-limits-to-routes), [Default Route Limits](#default-route-limits), [Application-Wide Limits](#application-wide-limits), [Shared Limits](#shared-limits) |
| 4. Configure | [Strategies](#strategies), [Storage Backends](#storage-backends), [Custom Key Functions](#custom-key-functions), [Conditional Exemption](#conditional-exemption), [Headers](#headers) |
| 5. Operate | [Error Handling](#error-handling), [Async And WebSockets](#async-and-websockets), [Testing](#testing), [Production Checklist](#production-checklist), [Pitfalls](#pitfalls) |

## Quick Reference

| Feature | Support | Notes |
|---------|---------|-------|
| Per-route decorator | `@limiter.limit("100/minute")` | Stack multiple decorators on one route |
| Per-method limits | `per_method=True` on the decorator | One counter per HTTP verb |
| Custom key function | `key_func=...` (constructor or per-route) | Receives the Starlette `Request` |
| Conditional exemption | `exempt_when=callable` on the decorator | Returns `True` to skip the limit |
| Function exemption | `limiter.exempt(fn)` | Skips all limits, including defaults |
| Default route limits | `default_limits=["20/minute"]` | Safety net applied to every slowapi-handled route unless overridden |
| Application-wide limit | `application_limits=[...]` | One counter shared across every request |
| Strategies | `fixed-window`, `moving-window`, `sliding-window-counter` | Library default is `fixed-window`; recommended production default is `moving-window` |
| Storage backends | memory, Redis (incl. Sentinel / Cluster URI variants) | Set via `storage_uri` |
| Standard headers | `headers_enabled=True` | Emits `X-RateLimit-*` and `Retry-After` |
| Soft fail on storage error | `swallow_errors=True` | Logs the error, allows the request (availability > strict enforcement) |

## When To Use

Use slowapi when the service is FastAPI or Starlette and you need decorator-style limits with a `limits`-compatible storage backend. It is a practical default for HTTP-only services, but adopt it deliberately: SlowAPI's own documentation describes the project as alpha-quality and warns that the API may change.

Reach for [`fastapi-limiter`](https://pypi.org/project/fastapi-limiter/) only when you must rate-limit **WebSocket** connections — slowapi targets HTTP routes and does not support WebSocket limits.

## Install

```bash
uv add slowapi
```

Optional storage drivers:

```bash
uv add redis              # for storage_uri="redis://..."
```

slowapi pulls in `limits` automatically — do not add it explicitly.

## Wire Up The Limiter

### Limiter object

#### Purpose
Per-FastAPI-app rate-limit enforcer that intercepts inbound requests and short-circuits with HTTP 429 when keyed quotas are exceeded.

#### Responsibilities
- Owns key resolution — IP, user id, API key, or a caller-supplied custom `key_func`.
- Owns the storage-backend connection (in-memory or Redis via `storage_uri`) and the chosen window strategy.
- Applies per-route limits (decorator), `default_limits` (every route), and `application_limits` (cross-route counter).
- Emits `X-RateLimit-*` and `Retry-After` headers when `headers_enabled=True`.
- Raises `RateLimitExceeded` so the registered exception handler converts it into HTTP 429.
- Must not own business identity — callers supply the `key_func` that maps a `Request` to a bucket key.
- Must not own retry/backoff — clients decide whether to honor `Retry-After`.
- Must not store request bodies — only counters keyed by `key_func` output.

#### Lifecycle
- Scope: App Scope (`Singleton`) — one `Limiter` per FastAPI app / worker process.
- Created by: the DI container or the app factory at startup.
- Shared by: every route on that FastAPI app, via `app.state.limiter`.
- Cleaned up by: process shutdown (storage backends — especially Redis — close on their own lifecycle, not the limiter's).

#### Relationships
```text
[App Scope]
Container ──(app-scoped Singleton)──▶ Limiter
                              │
                              ▼
                       FastAPI.state.limiter
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
 SlowAPIMiddleware    @limiter.limit routes    RateLimitExceeded handler
```

#### Constraints
- One `Limiter` per FastAPI app — do not share across apps with overlapping route names.
- `SlowAPIMiddleware` must be registered **before** dependent middleware that reads rate-limit headers, and the `RateLimitExceeded` exception handler must be registered so the raised exception converts to 429 (load-bearing order — see [Wiring sequence](#wiring-sequence)).
- The storage backend must outlive the app: a Redis instance shared across replicas must be reachable for the limiter's full lifetime, and `in_memory_fallback_enabled=True` is the only safe degradation path.
- Route handlers covered by the limiter must declare `request: Request` as the first parameter — slowapi resolves the limiter through `request.app.state`, not closure capture.

### Middleware vs Decorator

`SlowAPIMiddleware` and `@limiter.limit(...)` are **complementary, not alternatives** — together they form the complete rate-limit surface.

| Piece | Role |
|-------|------|
| `SlowAPIMiddleware` | Wires the limiter into the FastAPI request/response lifecycle. Enforces `default_limits` on every covered route, emits `X-RateLimit-*` and `Retry-After` headers, and surfaces `RateLimitExceeded` so the registered handler returns HTTP 429. |
| `@limiter.limit(...)` | Declares **per-route** quotas. Overrides `default_limits` (when `override_defaults=True`) or stacks on top of them (default `override_defaults=False`). |

The middleware is the chassis (lifecycle integration, headers, defaults). The decorator is the policy declaration (per-route quotas). Use both — pick one and you lose either the headers/defaults (no middleware) or per-route precision (no decorators).

### Wiring sequence

A FastAPI app needs four wiring steps:

1. Construct a `Limiter`.
2. Attach it to `app.state.limiter` (slowapi reads it from there).
3. Register `SlowAPIMiddleware` so the limiter sees every request.
4. Register `_rate_limit_exceeded_handler` for the `RateLimitExceeded` exception.

```python
from fastapi import FastAPI
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["20/minute"],
    headers_enabled=True,
    strategy="moving-window",
    retry_after="http-date",
)

app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
app.add_middleware(SlowAPIMiddleware)
```

`app.state.limiter` is mandatory. Without it, route decorators raise at request time because they look up the limiter through the request's app state.

## Apply Limits To Routes

The slowapi limit decorator requires the route handler's first argument to be the Starlette `Request`. FastAPI binds it automatically when you declare the type.

```python
from fastapi import Request

@app.get("/items/{item_id}")
@limiter.limit("100/minute")
async def read_item(request: Request, item_id: int) -> dict:
    return {"item_id": item_id}
```

Stack decorators to compose limits:

```python
@app.post("/search")
@limiter.limit("10/second")
@limiter.limit("100/minute")
async def search(request: Request) -> list[dict]:
    ...
```

Decorator parameters:

| Param | Purpose |
|-------|---------|
| `limit_value` | The rate string (`"100/minute"`, `"10/second"`, `"1000/hour"`) |
| `key_func` | Per-route override of the limiter-level `key_func` |
| `per_method` | `True` keeps a separate counter per HTTP verb |
| `methods` | Restrict the limit to specific verbs |
| `error_message` | Custom message body when the limit trips |
| `exempt_when` | Callable returning `True` to bypass the limit |
| `override_defaults` | `False` keeps `default_limits` in addition to this one |

## Default Route Limits

`default_limits` is the **per-route safety net** applied to every slowapi-handled route unless `override_defaults=True` is set on a per-route decorator. It is not a single global counter — each route maintains its own bucket under the same limit string. Use it to catch forgotten routes.

```python
limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["1000/hour", "20/minute"],
)
```

A request to `/items/1` and a request to `/items/2` count against separate buckets under the same `"20/minute"` ceiling.

## Application-Wide Limits

`application_limits` is the **truly global counter** — every request, regardless of route, increments the same bucket. Use only when that's the desired behavior (e.g., a hard cap on total inbound traffic to a service).

```python
limiter = Limiter(
    key_func=get_remote_address,
    application_limits=["10000/minute"],
)
```

**Caveat:** one hot endpoint can starve unrelated traffic. If `/search` saturates the application-wide budget, `/healthz` and `/version` get 429s too. Prefer `default_limits` plus per-route overrides for general protection; reserve `application_limits` for explicit total-traffic ceilings.

## Shared Limits

`shared_limit` lets several routes share one counter. Identify the group with a **semantically meaningful** `scope` name (the bucket key surfaces in storage and logs — make it greppable).

```python
search_limit = limiter.shared_limit("60/minute", scope="search")

@app.get("/search/items")
@search_limit
async def search_items(request: Request) -> list[dict]:
    ...

@app.get("/search/users")
@search_limit
async def search_users(request: Request) -> list[dict]:
    ...
```

A request to either route increments the same `search` counter — useful when several endpoints hit the same expensive downstream (a search index, an LLM, a third-party API) and you want one shared budget across them.

## Strategies

`limits` exposes three strategies; slowapi picks them through the `strategy` constructor argument.

| Strategy | Behavior | When to choose |
|----------|----------|----------------|
| `fixed-window` | Counter resets at the boundary (e.g. on the minute) | **Library default.** Cheapest; bursty users get full quota right at the boundary |
| `moving-window` | Counts requests in the trailing window | **Recommended production default.** Smoothest enforcement; no boundary-burst |
| `sliding-window-counter` | Approximate moving window using two fixed counters | Cheaper than `moving-window` on Redis at scale |

```python
limiter = Limiter(key_func=get_remote_address, strategy="moving-window")
```

The distinction is load-bearing: the **library default** is `fixed-window` (chosen for cheapness), but the **recommended production default** is `moving-window` (chosen for smooth enforcement). Pick `moving-window` unless storage cost forces otherwise.

## Storage Backends

`storage_uri` selects the backend. The URI format is owned by `limits`.

| Backend | URI | Notes |
|---------|-----|-------|
| In-memory | `memory://` (default) | Local dev, tests, single-worker deployments. Per-process; useless behind multiple workers |
| Redis | `redis://host:6379/0` | Default for multi-worker / multi-instance deployments. Sentinel uses `redis+sentinel://host:26379/mymaster`; Cluster uses `redis+cluster://host1:6379,host2:6379` as URI variants |

```python
limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://redis:6379/0",
    storage_options={"socket_connect_timeout": 1},
    in_memory_fallback_enabled=True,
    in_memory_fallback=["50/minute"],
)
```

`in_memory_fallback_enabled=True` keeps the service serving when Redis is unreachable, applying `in_memory_fallback` limits per-process until the backend recovers.

## Custom Key Functions

The key function decides "what bucket does this request count against." The default `get_remote_address` keys on client IP.

**Guidance:**
- `get_remote_address` (default) is acceptable for **unauthenticated routes** where the IP is the best identity you have.
- **Authenticated APIs should key on `user_id` or API key**, not IP — one user behind a corporate NAT must not block another, and a shared API key must not be defeated by IP rotation.
- Behind a load balancer or reverse proxy, the resolved client IP must be **verified-trustworthy** before IP-based limits are reliable: `X-Forwarded-For` parsing must be correct and trusted-hops must be configured (Starlette's `ProxyHeadersMiddleware` or equivalent). An untrusted forwarded-for is spoofable, and the limiter will key on whatever the attacker sends.

```python
def user_or_ip_key(request: Request) -> str:
    user_id = getattr(request.state, "user_id", None)
    if user_id:
        return f"user:{user_id}"
    return f"ip:{get_remote_address(request)}"

limiter = Limiter(key_func=user_or_ip_key, default_limits=["20/minute"])
```

`request.state.user_id` is expected to be populated by an upstream auth middleware; slowapi has no opinion on how it gets there.

Override per-route when one endpoint needs a different identity:

```python
@app.post("/admin/reindex")
@limiter.limit("1/minute", key_func=lambda r: f"tenant:{r.state.tenant_id}")
async def reindex(request: Request) -> None:
    ...
```

## Conditional Exemption

`exempt_when` runs once per request. Return `True` to skip the limit.

The two-decorator pattern below applies different limits to authenticated vs. anonymous callers on a public endpoint:

```python
def is_authenticated(request: Request) -> bool:
    return getattr(request.state, "user_id", None) is not None

@app.get("/public/ping")
@limiter.limit("100/minute", exempt_when=lambda r: not is_authenticated(r))
@limiter.limit("10/minute", exempt_when=is_authenticated)
async def ping(request: Request) -> dict:
    return {"ok": True}
```

Outer decorator (`100/minute`) applies only when the user is authenticated; inner decorator (`10/minute`) applies only when they are anonymous. Each request hits exactly one bucket.

To exempt a function from **all** limits — including `default_limits`:

```python
@app.get("/healthz")
@limiter.exempt
async def healthz(request: Request) -> dict:
    return {"ok": True}
```

## Headers

Set `headers_enabled=True` so clients can self-throttle.

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | The active limit in this window |
| `X-RateLimit-Remaining` | Requests left before the limit trips |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | How long to wait before retrying (only on 429) |

`retry_after="http-date"` emits an HTTP-date instead of seconds; pick the format your clients expect.

## Error Handling

When a limit trips, slowapi raises `slowapi.errors.RateLimitExceeded`. The bundled `_rate_limit_exceeded_handler` returns HTTP 429 with the configured headers. Replace it with a custom handler when you need a different response shape.

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from slowapi.errors import RateLimitExceeded

@app.exception_handler(RateLimitExceeded)
async def rate_limit_handler(request: Request, exc: RateLimitExceeded) -> JSONResponse:
    return JSONResponse(
        status_code=429,
        content={
            "error_code": "RATE_LIMIT_EXCEEDED",
            "error_msg": "Too many requests, slow down.",
            "limit": str(exc.limit.limit),
        },
    )
```

Keep the response shape consistent with `python-guidelines.md` (Appendix → Exception Handling).

## Async And WebSockets

slowapi supports both `def` and `async def` route handlers identically — no special wiring.

slowapi does **not** rate-limit WebSocket connections. If you need that, scope a separate dependency (`fastapi-limiter`) for the WebSocket routes only and keep slowapi for HTTP.

## Testing

Disable the limiter inside tests so quota math does not pollute behavior assertions. Two options:

1. Toggle the live limiter via `enabled = False`:

```python
import pytest
from fastapi.testclient import TestClient

@pytest.fixture
def client(app) -> TestClient:
    app.state.limiter.enabled = False
    return TestClient(app)
```

2. Construct the Limiter with `enabled=False` in the test app factory.

When the suite specifically tests rate-limit behavior, leave it on and use a unique `key_prefix` per test (or flush the storage between tests) to avoid cross-test contamination.

## Production Checklist

| Item | Setting |
|------|---------|
| Storage | `storage_uri="redis://..."` (never `memory://` behind multiple workers) |
| Strategy | `strategy="moving-window"` |
| Headers | `headers_enabled=True`, `retry_after="http-date"` |
| Resilience | `swallow_errors=True`, `in_memory_fallback_enabled=True` |
| Multi-tenancy | `key_prefix="<service-name>"` so two services do not collide on shared Redis |
| Identity | `key_func` returns user id when authenticated, IP otherwise |
| Defaults | `default_limits` set as a safety net for forgotten routes |

`swallow_errors=True` favors **availability over strict enforcement** — a storage blip silently lets traffic through (the request is allowed, the error is logged). Pair it with `in_memory_fallback_enabled=True` so the limiter keeps counting locally while the backend is unreachable; otherwise a Redis outage uncaps every route for the duration.

## Pitfalls

- The route handler **must** declare `request: Request` as the first parameter. slowapi looks it up by name; without it the limiter cannot read the app state and the decorator raises.
- `app.state.limiter` is required even if every route uses an explicit decorator — slowapi resolves the limiter through the request, not through closure capture.
- `SlowAPIMiddleware` must be registered. Without it, headers are not emitted and `default_limits` do not apply.
- `key_func` receives a Starlette `Request` argument. Do not pull request data from anywhere else (no thread-locals, no contextvars) — the `Request` is the only authoritative source.
- `memory://` storage is per-process. With `--workers 4` (uvicorn/gunicorn) each worker tracks its own counter — quotas effectively multiply by worker count. Use Redis as soon as the deployment is multi-process.
- `application_limits` counts every route together. If one endpoint dominates traffic it will starve the rest. Prefer `default_limits` plus per-route overrides.
- `swallow_errors=True` favors availability over strict enforcement — a storage blip silently lets traffic through. Always pair with `in_memory_fallback_enabled=True` so the limiter keeps counting locally while the backend is unreachable; without the fallback, a Redis outage uncaps every route.
- Behind a load balancer, IP-based keys are only as trustworthy as your `X-Forwarded-For` parsing. Configure trusted hops before relying on `get_remote_address` in production.
- Stacking two `@limiter.limit(...)` decorators with overlapping `exempt_when` callbacks is the cleanest way to express "different limits for different audiences." Avoid branching inside a custom `key_func` for this — exemption logic lives at the decorator layer.
