# Caching

> Last updated: 2026-05-13

## TL;DR

Caching options for Python services, organized in four layers — pick by scope.
Layer 1 is in-process memoization (stdlib + `cachetools`). Layer 2 is HTTP
response caching (`hishel` for `httpx`, `requests-cache` for `requests`). Layer
3 is a distributed external cache (Redis). Layer 4 is an RDB-backed cache
table when durability beats hit rate.

**Use this when:**
- a hot read path needs to skip recomputation, a slow upstream, or Postgres
- you need to decide *which* cache shape fits the scope (process / HTTP /
  cross-instance / durable)
- you are reaching for "let's just put it in Redis" and want to confirm a
  lighter layer wouldn't do

**Don't use this for:**
- in-process memoization → [Layer 1](#layer-1-function-memoization-single-process-in-memory)
- session/auth state → `./cognito-auth.md`
- vector similarity caches → `./milvus.md`

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Orient | [Quick Reference](#quick-reference), [Decision Tree](#decision-tree) |
| 2. Pick a Layer | [Layer 1: Function Memoization](#layer-1-function-memoization-single-process-in-memory), [Layer 2: HTTP Response Caching](#layer-2-http-response-caching) |
| 3. Pick a Layer | [Layer 3: Distributed / External (Redis)](#layer-3-distributed--external-redis), [Layer 4: RDB-Backed Cache](#layer-4-rdb-backed-cache) |
| 4. Operate | [Pitfalls](#pitfalls) |

## Quick Reference

| Tool | Persistent | TTL | Distributed | Async | Dependency | Maturity |
|------|------------|-----|-------------|-------|------|----------|
| `functools.lru_cache` / `cache` | No | No | No | No ⚠️ | stdlib | Stable |
| `functools.cached_property` | No (per instance) | No | No | No ⚠️ | stdlib | Stable |
| `cachetools` | No | Yes (`TTLCache`) | No | No ⚠️ | `cachetools` | Stable |
| `requests-cache` | Yes | Yes | Optional (Redis backend) | No | `requests-cache` | Stable (1.3.x) |
| `hishel` | Yes | Yes (RFC 9111) | Optional (Redis backend) | Yes | `hishel` | Alpha (single maintainer) |
| Redis | Yes | Yes (`EX` / `EXPIRE`) | Yes | Yes (`redis.asyncio`) | `redis>=5` | Stable |
| RDB (Postgres) | Yes | Manual (`expires_at` column) | Yes | Yes (via async driver) | Existing DB | Stable |

⚠️ Caching an `async def` result with these stores the **coroutine object**,
not its awaited value. Second `await` raises `RuntimeError: cannot reuse
already awaited coroutine`. Don't do it — reach for `asyncache` (Layer 1) or
move the cache out-of-process (Layer 3/4).

## Decision Tree

```text
What are you caching?
│
├── A pure function's result, within one process?
│   │
│   ├── Bounded inputs, want simplest thing?            ── @functools.cache
│   ├── Unbounded inputs, want LRU eviction?            ── @functools.lru_cache(maxsize=N)
│   ├── Expensive per-instance attribute?               ── @functools.cached_property
│   └── Need TTL or LFU?                                ── cachetools.TTLCache / LFUCache
│
├── An HTTP response (same URL, same headers)?
│   │
│   ├── Client is httpx?                                ── hishel
│   └── Client is requests?                             ── requests-cache
│
├── Anything shared across processes / servers?         ── Redis (Layer 3)
│   │   (session caches, rate-limit counters, hot data)
│   │
│   └── Need durability, audit trail, or you already
│       have a Postgres you'd rather not double?        ── RDB cache table (Layer 4)
│
└── async function result?                              ── NOT @lru_cache / @cache
                                                            asyncache, Redis, or RDB
```

## Layer 1: Function Memoization (Single-Process, In-Memory)

Per-process, in-memory caches. Zero infrastructure; lost on restart; useless
across workers. Reach for these first — they're often the right answer.

### `functools.lru_cache` vs `functools.cache`

| | `functools.lru_cache` | `functools.cache` |
|---|---|---|
| Added in | 3.2 | 3.9 |
| Size limit | `maxsize=N` (default 128); `None` = unbounded | Unbounded (no eviction) |
| Eviction | LRU | None |
| Speed | Slightly slower (LRU bookkeeping) | Faster (no bookkeeping) |

When to use:

- `@cache` — bounded, pure functions with a known small input domain
  (config-driven lookups, parsed-once tables). No eviction means unbounded
  inputs leak memory.
- `@lru_cache(maxsize=N)` — unbounded inputs where LRU eviction caps the
  working set.

Common caveats:

- Arguments must be **hashable**. Lists, dicts, sets won't work as-is.
- On methods, `self` is part of the key. The cache lives on the class and
  every instance contributes entries — usually not what you want. Use
  [`cached_property`](#functoolscached_property) for per-instance attributes.
- Introspect with `func.cache_info()` (hits/misses/size); flush with
  `func.cache_clear()`.
- **Never apply to `async def`.** The decorator caches the coroutine object,
  not its awaited value. The first `await` returns the value; every
  subsequent call returns the same already-awaited coroutine and raises
  `RuntimeError: cannot reuse already awaited coroutine`. Use
  [`asyncache`](https://github.com/hephex/asyncache) or move the cache
  out-of-process.

```python
from functools import cache, lru_cache

@cache
def parse_country_table() -> dict[str, str]: ...

@lru_cache(maxsize=1024)
def geocode(address: str) -> tuple[float, float]: ...
```

**Avoid recursion in Python.** The default recursion limit is 1000 and CPython
has no tail-call optimization. Memoized recursion can blow the stack on inputs
that an iterative DP solution handles trivially. Prefer iterative dynamic
programming. Recursion is acceptable only when depth is log-bounded
(balanced-tree walks, divide-and-conquer over halved inputs).

### `functools.cached_property`

Caches a computed attribute on the **instance**. The value lives in the
instance's `__dict__`, so it survives as long as the instance does and is
garbage-collected with it. Python 3.12 removed the internal per-property lock;
concurrent first-access on the same instance from multiple threads may compute
the value more than once.

```python
from functools import cached_property

class Report:
    def __init__(self, rows: list[Row]) -> None:
        self.rows = rows

    @cached_property
    def total(self) -> int:
        return sum(r.amount for r in self.rows)
```

### `cachetools`

Stdlib `functools` lacks TTL and LFU. `cachetools` fills that gap while
staying in-memory and process-local.

```bash
uv add cachetools
```

```python
from cachetools import TTLCache, cached

@cached(cache=TTLCache(maxsize=1024, ttl=600))
def lookup_country(ip: str) -> str: ...
```

Reach for `cachetools` when the stdlib options don't cover what you need
(TTL, LFU, custom eviction) but the cache is still single-process.

## Layer 2: HTTP Response Caching

When the cache key is a URL (plus headers) and the origin sends usable
`Cache-Control` / `ETag`, an HTTP-aware cache is the right shape — it honors
freshness, conditional requests, and `Vary` for free. Useful when re-crawling
or polling stable endpoints.

### `hishel` (for `httpx`)

RFC 9111-compliant HTTP cache transport for `httpx`. File, SQLite, or Redis
storage backends.

```bash
uv add hishel
```

```python
import hishel

async with hishel.AsyncCacheClient(storage=hishel.AsyncFileStorage()) as client:
    response = await client.get("https://api.example.com/v1/list")
```

**Maturity caveat.** `hishel` is marked **Alpha** on PyPI and is essentially a
single-maintainer project. Pin tightly and budget for the possibility that
you'll outgrow it. Alternatives if the risk doesn't fit:

- Switch HTTP client to `requests` and use `requests-cache` (much more mature).
- Roll your own URL→SQLite or URL→Redis layer in front of `httpx` (cheaper than
  it sounds when you only need a simple TTL).
- Solve at the CDN / application layer (e.g. an upstream cache or a manual
  cache table — see [Layer 4](#layer-4-rdb-backed-cache)).

### `requests-cache` (for `requests`)

Drop-in transparent cache for the `requests` library. Backends include SQLite
(default), Redis, MongoDB, DynamoDB, filesystem. Supports `Cache-Control`,
conditional requests (`ETag` / `If-Modified-Since`), and stale-if-error.
Stable on the 1.3.x line, widely used.

```bash
uv add requests-cache
# or with a Redis backend:
uv add "requests-cache[redis]"
```

```python
from requests_cache import CachedSession

session = CachedSession("demo_cache", expire_after=3600, cache_control=True)
response = session.get("https://api.example.com/v1/list")
```

## Layer 3: Distributed / External (Redis)

Reach for Redis when state must be shared across processes or servers:
multi-worker / multi-instance deployments, session caches, rate-limit
counters, and hot read paths where a short TTL is acceptable. Sub-millisecond
latency in-region, native TTL, native pub/sub.

Client: **`redis-py`** (the `redis` package) — unified sync and async surfaces;
the async one lives at `redis.asyncio`. Don't pull in the archived `aioredis`
package.

```bash
uv add redis
```

```python
from redis.asyncio import Redis

redis = Redis.from_url("redis://localhost:6379/0", decode_responses=False)
await redis.setex("orders:order:123:v3", 300, b"<serialized value>")
value = await redis.get("orders:order:123:v3")
```

See also: [redis-py async docs](https://redis.readthedocs.io/en/stable/examples/asyncio_examples.html).

## Layer 4: RDB-Backed Cache

A cache table inside your existing relational database. Slower than Redis but
trades latency for durability, transactionality, and not needing another piece
of infrastructure.

```sql
CREATE TABLE response_cache (
    key         TEXT PRIMARY KEY,
    value       JSONB       NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL
);
CREATE INDEX response_cache_expires_at_idx ON response_cache (expires_at);
```

Trade-offs:

- **Pro:** reuses infrastructure you already operate; transactional with the
  rest of your writes; queryable (you can `SELECT … WHERE value @> '{…}'`).
- **Con:** slower than Redis (single-digit-millisecond minimum); TTL is
  manual — you need a sweep job or a `WHERE expires_at > now()` filter on
  reads; takes connection-pool slots that may be scarce.

**When RDB caching is the right call:**

- You **already operate a Postgres** and adding Redis just for caching isn't
  justified.
- You need an **audit-friendly** record of cached values (when computed, by
  whom, for which inputs).
- **Durability over hit rate** — losing the cache on a node failure is
  unacceptable. Common for expensive LLM responses that cost real money to
  recompute.
- You're really **storing computed results**, not caching them. At that point
  the table is closer to a materialized view than a cache.

**Postgres tip — `UNLOGGED TABLE` caveat.** `CREATE UNLOGGED TABLE` skips the
write-ahead log and is significantly faster to write, at the cost of losing
all rows on a crash or replica promotion. Treat this as an optimization for
**proven write bottlenecks**, not the default choice. Appropriate only when
the table is a genuine cache (every row is recomputable) and you've measured
the WAL overhead actually mattering.

## Pitfalls

- **`@lru_cache` / `@cache` on `async def`.** Caches a coroutine object that
  can only be awaited once. The second hit raises
  `RuntimeError: cannot reuse already awaited coroutine`. Use `asyncache`,
  Redis, or an RDB table.
- **Missing TTL on a cache that should expire.** Layer 1 (`@cache`) and
  Redis keys set without `ex=` live forever; an RDB cache without
  `expires_at` is a junk table. Every cache write that isn't intentionally
  permanent should carry a TTL.
- **Reaching for a heavier layer than the scope needs.** Redis for what a
  per-process `@lru_cache` would have covered; a managed RDB cache table
  when an HTTP-aware cache (Layer 2) does the job for free. Operational
  cost matters — every new layer is another thing to monitor, fail, and
  restart.
- **Caching mutable values in Layer 1.** In-process caches return the same
  object reference on every hit. A consumer that mutates the cached value
  silently mutates the cache. Freeze (tuple, frozen dataclass) or copy on
  read.
- **Choosing Redis when an RDB cache table would have sufficed.** If the
  data is durable-by-nature (you'd be sad to lose it), already lives next
  to a Postgres you operate, or needs to be queryable, an RDB cache table
  is the right tool — adding Redis is operational overhead you don't need.
