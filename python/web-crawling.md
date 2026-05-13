# Web Crawling

> Last updated: 2026-05-13

## TL;DR

Two tracks for fetching content from the public web — `httpx.AsyncClient` for direct HTTP and Playwright for JavaScript-rendered pages — plus rate-limit, retry, header, and stealth tactics that keep both off the 4xx blocklist.

**Use this when:**
- scraping or crawling public web pages from an async backend
- you need per-host rate limiting, retries with backoff, and `Retry-After` honoring
- pages require JavaScript execution and you need a managed headless browser pool

**Don't use this for:**
- outbound HTTP to your own / internal APIs — `httpx.AsyncClient` direct, no crawl-etiquette tactics needed
- large-scale crawling — use a dedicated scraping service
- agent-driven web search invoked from inside a tool → wrap a crawler from here as a `@tool`, see `./langchain.md#tools`

## Table of Contents

| # | Phase | What you produce |
|---|-------|------------------|
| 1 | Pick the track | Decision: direct HTTP via `httpx.AsyncClient`, or headless browser via Playwright |
| 2 | Configure the client | Realistic headers, tiered timeouts, connection pool, follow_redirects, http2 |
| 3 | Tune retry & rate limits | `aiolimiter` per-host, `tenacity` over the inner request, `Retry-After` honored |
| 4 | Add anti-block tactics | Stealth patches, sandbox flags, viewport/locale, cookie handoff |
| 5 | Operate in production | Lifespan-scoped pool, browsers baked into the image, robots.txt cached |

## Quick Reference

| Concern | Tool | Notes |
|---|---|---|
| Direct HTTP, JSON / static HTML | `httpx.AsyncClient` | Default. Pool, headers, timeouts, follow_redirects, http2. |
| JavaScript-rendered pages | `playwright.async_api` + `playwright_stealth` | 50–100× slower; only when JS execution is required. |
| Per-host rate limit | `aiolimiter.AsyncLimiter(N, time_period=1.0)` | Async context manager. Per-event-loop. |
| Retry on 429/5xx + transient errors | `tenacity.retry` (async-aware) | Wrap the inner request, not the whole pipeline. |
| Bounded concurrency across hosts | `asyncio.Semaphore` | Distinct from per-host rate limiting. |
| Robots.txt etiquette | `urllib.robotparser` | Read once per host, cache. |
| HTML parsing | `bs4 + lxml` (default), `selectolax` (CSS-only + scale) | See [HTML Parsing](#html-parsing). |
| Anti-fingerprint (cookies, user-agent) | `playwright_stealth` | Patches `navigator.webdriver`, plugins, WebGL, etc. |

## When To Use

- **Use `httpx.AsyncClient`** for REST APIs, static HTML, file downloads, anything that returns content without running JavaScript. This is the default — try it first.
- **Use Playwright** for SPA pages, dashboards rendered via XHR, click-to-export buttons, and sites with anti-bot challenges that fail on plain HTTP.
- **Decision rule.** Start with `httpx` and a realistic header set. Move to Playwright only when the page measurably needs JS execution (empty `<body>` on raw HTML, or content injected by client-side fetch).

Cross-links:

- [`./rate-limit.md`](./rate-limit.md) — *server-side* rate limiting (your API throttling its callers). The opposite direction from this doc.
- [`./langchain.md#tools`](./langchain.md#tools) — when the crawler is invoked from inside an agent tool.
- [`./file-processing.md`](./file-processing.md) — what to do with the bytes once fetched.
- [`./dependency-injector.md`](./dependency-injector.md) — wiring the fetcher and browser pool into the app.
- [`./python-tests.md`](./python-tests.md) — fixtures and mocks for crawler tests.

## Install

```bash
uv add httpx tenacity aiolimiter beautifulsoup4 selectolax
uv add playwright playwright-stealth
uv run playwright install chromium  # one-time download (~170 MB)
```

In Docker, run `playwright install --with-deps chromium` during the image build, never at runtime — the binary download is too slow for cold start and pulls system libraries that may not be present in slim base images.

```dockerfile
RUN pip install playwright && playwright install --with-deps chromium
```

## Wire Up

Two long-lived singletons — an HTTP fetcher and a browser pool — owned by the FastAPI lifespan. See [`./dependency-injector.md`](./dependency-injector.md) for the container pattern.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class CrawlerSettings:
    user_agent: str
    per_host_rate: float  # requests per second
    request_timeout_s: float  # default read timeout
    connect_timeout_s: float  # connect timeout
    max_concurrent_hosts: int  # global cap on in-flight requests
    max_connections: int  # httpx connection pool ceiling
    max_keepalive: int  # idle connections to keep open
```

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = CrawlerSettings(
        user_agent="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
        "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.6 Safari/605.1.15",
        per_host_rate=2.0,
        request_timeout_s=30.0,
        connect_timeout_s=5.0,
        max_concurrent_hosts=16,
        max_connections=100,
        max_keepalive=20,
    )
    fetcher = CrawlerHttpClient(settings)
    browser_pool = BrowserPool(settings)
    await browser_pool.start()
    app.state.fetcher = fetcher
    app.state.browser_pool = browser_pool
    try:
        yield
    finally:
        await fetcher.aclose()
        await browser_pool.stop()


app = FastAPI(lifespan=lifespan)
```

The `CrawlerHttpClient` and `BrowserPool` classes appear in the sections below.

## Realistic Request Headers

Sites filter on header *plausibility*, not just `User-Agent`. The minimum set that looks like a real browser:

```python
DEFAULT_HEADERS: dict[str, str] = {
    "User-Agent": (
        "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
        "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.6 Safari/605.1.15"
    ),
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.5",
    "Connection": "keep-alive",
    "Upgrade-Insecure-Requests": "1",
}
```

Per-host overrides for sites that require an `Origin`, `Authority`, or `Referer`:

```python
HOST_HEADER_OVERRIDES: dict[str, dict[str, str]] = {
    "data.example.com": {
        "Origin": "https://www.example.com",
        "Referer": "https://www.example.com/",
        "Accept": "application/json",
    },
}


def headers_for(host: str) -> dict[str, str]:
    return DEFAULT_HEADERS | HOST_HEADER_OVERRIDES.get(host, {})
```

`httpx` automatically sends `Accept-Encoding: gzip, deflate, br` and decodes the response — do not override unless you have a reason. User-Agent rotation across requests *to the same host* is rarely worth it: the site correlates on TLS fingerprint and cookies, not just UA. For real fingerprint evasion, the active library is [`curl_cffi`](https://github.com/lexiforest/curl_cffi) (out of scope here — pointer only).

## CrawlerHttpClient (Direct HTTP Track)

### Purpose

Async HTTP client wrapping `httpx.AsyncClient` with per-host rate limiting, retry-after honoring, tiered timeouts, and bounded global concurrency for crawling external sites.

### Responsibilities

- Own one `httpx.AsyncClient` and its connection pool — no caller constructs or holds the client directly.
- Own the per-host token-bucket limiter map (`dict[str, AsyncLimiter]`); create entries lazily inside the running event loop.
- Own retry decoration around `get` (status-allowlist, exponential backoff, `Retry-After` parsing).
- Honor `Retry-After` headers on 429 / 503 before falling back to exponential backoff.
- Enforce the global concurrency `Semaphore` so total in-flight requests stay bounded across hosts.
- Clean up the `AsyncClient` (`aclose()`) on FastAPI lifespan shutdown — no per-request close.

### Lifecycle

- **Scope**: app-scoped Singleton — one instance per process.
- **Created by**: FastAPI lifespan startup (`CrawlerHttpClient(settings)`); see [Wire Up](#wire-up).
- **Shared by**: every crawler, worker, and tool that needs outbound HTTP — concurrent callers share the same client and pool.
- **Cleaned up by**: FastAPI lifespan shutdown calls `await fetcher.aclose()` exactly once.

### Relationships

```text
[App Scope]
    └── [CrawlerHttpClient]              (singleton)
            │ holds
            ├── [httpx.AsyncClient]    (singleton, connection pool inside)
            ├── [dict[host, AsyncLimiter]]  (per-host, lazy)
            └── [asyncio.Semaphore]    (global concurrency cap)

[Request Scope]
    └── [crawl worker] ── calls ──▶ [CrawlerHttpClient.get()]
```

### Constraints

- One `CrawlerHttpClient` per process. Never per-request, never per-task.
- Never share the `httpx.AsyncClient` reference outside the fetcher — callers go through `get()` so rate-limit + semaphore + retry stay enforced.
- The per-host limiter map must be bounded. If host cardinality grows unbounded (open-web crawl), evict via LRU; do not let the dict grow without ceiling.
- `AsyncLimiter` instances must be created lazily inside the running event loop, never at import time — they are bound to the loop that creates them.
- `aclose()` runs exactly once, during lifespan shutdown. Calling it mid-request leaks in-flight responses.

```python
import asyncio
import httpx
from aiolimiter import AsyncLimiter


class CrawlerHttpClient:
    def __init__(self, settings: CrawlerSettings) -> None:
        self._settings = settings
        self._client = httpx.AsyncClient(
            headers=DEFAULT_HEADERS,
            timeout=httpx.Timeout(
                settings.request_timeout_s,
                connect=settings.connect_timeout_s,
            ),
            limits=httpx.Limits(
                max_connections=settings.max_connections,
                max_keepalive_connections=settings.max_keepalive,
            ),
            follow_redirects=True,
            http2=True,
        )
        self._limiters: dict[str, AsyncLimiter] = {}
        self._semaphore = asyncio.Semaphore(settings.max_concurrent_hosts)

    def _limiter_for(self, host: str) -> AsyncLimiter:
        if host not in self._limiters:
            self._limiters[host] = AsyncLimiter(
                max_rate=self._settings.per_host_rate,
                time_period=1.0,
            )
        return self._limiters[host]

    async def get(self, url: str, **kwargs) -> httpx.Response:
        host = httpx.URL(url).host
        async with self._semaphore, self._limiter_for(host):
            response = await self._client.get(
                url,
                headers=headers_for(host),
                **kwargs,
            )
            response.raise_for_status()
            return response

    async def aclose(self) -> None:
        await self._client.aclose()
```

One `AsyncClient` per process. Per-call clients defeat the connection pool and add a TLS handshake to every request.

See also: [`httpx` async client](https://www.python-httpx.org/async/).

## Per-Host Rate Limiting

Two distinct concerns, both required:

- `asyncio.Semaphore` — global concurrency cap (open sockets, file descriptors, total in-flight).
- `aiolimiter.AsyncLimiter` — per-host request budget (requests per second per host).

```python
limiter = AsyncLimiter(max_rate=2.0, time_period=1.0)  # 2 req/sec per host

async with limiter:
    response = await client.get(url)
```

`AsyncLimiter.acquire()` suspends the coroutine until capacity is available; it never raises. Default `time_period` is 60 seconds — pass `time_period=1.0` explicitly for "N requests per second" semantics. Each `AsyncLimiter` is bound to one event loop, so create them lazily inside the async context, not at import time.

Calibration heuristic: start at **1 RPS per host**. Bump to 2 only after observing zero 429 / 503 over 1 000 requests against that host. Sites publish wildly different effective limits; one number does not generalize.

See also: [`aiolimiter` docs](https://aiolimiter.readthedocs.io/).

## Retry With Tenacity

`tenacity.retry` is awaitable-aware — the same decorator works on async functions.

```python
import asyncio
import httpx
from tenacity import (
    retry,
    retry_if_exception_type,
    stop_after_attempt,
    wait_exponential,
)


@retry(
    retry=retry_if_exception_type((httpx.RequestError, httpx.HTTPStatusError)),
    wait=wait_exponential(multiplier=2, min=1, max=60),
    stop=stop_after_attempt(4),
    reraise=True,
)
async def request_with_retry(client: httpx.AsyncClient, url: str) -> httpx.Response:
    response = await client.get(url)
    if response.status_code in {429, 503}:
        retry_after = response.headers.get("Retry-After")
        if retry_after is not None:
            await asyncio.sleep(min(_parse_retry_after(retry_after), 60))
        response.raise_for_status()
    elif response.status_code >= 500:
        response.raise_for_status()
    return response


def _parse_retry_after(value: str) -> float:
    try:
        return float(value)
    except ValueError:
        # Retry-After can also be an HTTP-date; clamp to 30 if so.
        return 30.0
```

Three retry presets, tuned to the workload:

| Workload | `wait` | `stop` | Why |
|---|---|---|---|
| Tight (rate-limited public site) | `wait_exponential(multiplier=10, max=60)` | `stop_after_attempt(3)` | Site has narrow tolerance; long backoff lets the bucket refill. |
| Heavy download (large file, slow recover) | `wait_exponential(multiplier=60, max=180)` | `stop_after_attempt(3)` | Recovery takes minutes; short backoff burns retries before any chance of success. |
| Lenient (generous public API) | `wait_incrementing(start=0.1, increment=0.3)` | `stop_after_attempt(3)` | Cheap to retry; tight backoff keeps p95 latency low. |

Retry rules to live by:

- **Do not retry 4xx other than 408 / 425 / 429.** Retrying 401 / 403 / 404 wastes time and can trigger account-level lockout on the target site.
- **Wrap the inner request only**, never the whole pipeline. Tenacity over `fetch + parse + persist` will replay persistence on a parser bug.
- **Honor `Retry-After`.** When present, it is the site telling you the answer; default backoff is irrelevant.

See also: [`tenacity` docs](https://tenacity.readthedocs.io/), [RFC 9110 §10.2.3 (`Retry-After` semantics)](https://datatracker.ietf.org/doc/html/rfc9110#section-10.2.3).

## Tiered Timeouts

```python
TIMEOUTS: dict[str, httpx.Timeout] = {
    "lookup": httpx.Timeout(10.0, connect=5.0),  # cheap GET, JSON
    "default": httpx.Timeout(30.0, connect=5.0),  # standard HTML page
    "download": httpx.Timeout(120.0, connect=5.0),  # large file, archive
}
```

Connect timeout always tight (5–10 s). Read timeout depends on payload — 10 / 30 / 120 covers the common cases. Per-call timeouts override the `AsyncClient` default.

See also: [`httpx` timeouts (connect/read/write/pool tiering)](https://www.python-httpx.org/advanced/timeouts/).

## Status-Code Handling

| Status | Default action | When to override |
|---|---|---|
| 2xx | Return | — |
| 301 / 302 / 307 / 308 | Auto-follow (`follow_redirects=True`) | Disable when verifying that a CDN URL stays canonical |
| 401 / 403 | Fail fast — auth, not rate | Cache the URL as-is for paywalled corpora to avoid re-fetching |
| 404 | Fail fast | Cache 404 to avoid re-fetching dead URLs |
| 408 | Retry | — |
| 425 | Retry | Rare; safe to retry per RFC 9110 |
| 429 | Honor `Retry-After`, back off, retry | Lower per-host RPS budget for the rest of the session |
| 5xx | Retry with exponential backoff | Treat persistent 503 as soft block — pause crawl |

For corpora where some URLs are intentionally restricted but you still want to record the URL, keep a `NON_FATAL_AUTH = {402, 403}` allowlist and short-circuit those status codes before `raise_for_status()` — record-and-move-on rather than treating them as fetch failures.

## BrowserPool (Headless Browser Track)

### Purpose

Async browser-context pool wrapping a single long-lived Playwright/Chromium process for stealth-enabled page rendering.

### Responsibilities

- Own the Chromium process (`Playwright` + `Browser` handles) — no caller launches Chromium directly.
- Hand out a fresh `BrowserContext` per request via `async with browser_pool.context()`; close on `__aexit__`. Never reuse a context across unrelated callers — contexts carry cookie / storage / permission state.
- Apply stealth patches in the documented order — context created, viewport/UA/locale set, `apply_stealth_async(ctx)` before any page work.
- Cap concurrent contexts by available memory — each loaded context is roughly 200 MB RSS.
- Clean up the browser process (`browser.close()` then `playwright.stop()`) on FastAPI lifespan shutdown.

### Lifecycle

- **Scope**: app-scoped Singleton — one Chromium process per app process.
- **Created by**: FastAPI lifespan startup calls `BrowserPool(settings)` then `await browser_pool.start()`. Startup is heavy (seconds — Chromium launch + stealth setup), so it must not happen on the request path.
- **Shared by**: every crawl worker / tool that needs JS rendering; each worker borrows a context via the async context manager.
- **Cleaned up by**: FastAPI lifespan shutdown calls `await browser_pool.stop()`, which closes the browser and stops Playwright in that order.

### Relationships

```text
[App Scope]
    └── [BrowserPool]              (singleton)
            │ holds
            ├── [Playwright]            (singleton)
            ├── [Browser / Chromium]    (singleton process)
            └── [BrowserContextFactory]   (per-request, concurrency-capped)

[Request Scope]
    └── [crawl worker]
            └── async with [BrowserPool.context()] ── borrows ──▶ [BrowserContext]
                                                       returns on __aexit__
```

### Constraints

- One `BrowserPool` per app process. Chromium is memory-bound (typically 200–500 MB resident before any page loads); a second pool doubles the budget for no gain.
- Never per-request. Browser startup is seconds; paying that on the hot path serializes every crawl.
- Contexts must be returned to the pool via the `async with` exit — never leak. A leaked context is ~200 MB held until process exit.
- Stealth patches must run in the documented order: context constructed with viewport / UA / locale / timezone, *then* `apply_stealth_async(ctx)`, *then* `new_page()`. Patching after a page exists misses early script hooks.
- Inconsistent stealth across contexts is worse than none — apply on every context the pool hands out, not selectively.

One Chromium process, many contexts. Each context is a fresh cookie jar — open one per request (or per session bucket) to keep state isolated.

```python
from contextlib import asynccontextmanager
from playwright.async_api import (
    async_playwright,
    Browser,
    BrowserContext,
    Playwright,
)
from playwright_stealth import Stealth


CHROMIUM_ARGS: list[str] = [
    "--no-sandbox",  # required in containers
    "--disable-gpu",  # no display in headless
    "--disable-dev-shm-usage",  # /dev/shm too small in Docker
    "--disable-setuid-sandbox",  # required in containers without setuid
    "--disable-blink-features=AutomationControlled",  # masks a key fingerprint flag
]


class BrowserPool:
    def __init__(self, settings: CrawlerSettings) -> None:
        self._settings = settings
        self._stealth = Stealth()
        self._playwright: Playwright | None = None
        self._browser: Browser | None = None

    async def start(self) -> None:
        self._playwright = await async_playwright().start()
        self._browser = await self._playwright.chromium.launch(
            headless=True,
            args=CHROMIUM_ARGS,
        )

    async def stop(self) -> None:
        if self._browser is not None:
            await self._browser.close()
        if self._playwright is not None:
            await self._playwright.stop()

    @asynccontextmanager
    async def context(self) -> BrowserContext:
        if self._browser is None:
            raise RuntimeError("BrowserPool.start() not called")
        ctx = await self._browser.new_context(
            viewport={"width": 1920, "height": 1080},
            user_agent=self._settings.user_agent,
            locale="en-US",
            timezone_id="America/New_York",
        )
        await self._stealth.apply_stealth_async(ctx)
        try:
            yield ctx
        finally:
            await ctx.close()
```

Why each Chromium flag matters:

| Flag | Reason |
|---|---|
| `--no-sandbox` | Containers cannot create user namespaces by default; without this the browser fails to start. |
| `--disable-gpu` | No display device in headless mode; GPU init wastes time and memory. |
| `--disable-dev-shm-usage` | Default Docker `/dev/shm` is 64 MB; large pages crash without this flag. |
| `--disable-setuid-sandbox` | Setuid not available in unprivileged containers. |
| `--disable-blink-features=AutomationControlled` | Removes the `navigator.webdriver` indicator that bot-detection scripts read. |

The pool size — how many concurrent contexts to allow — should be capped by available memory: each Chromium context is roughly 200 MB RSS once it has loaded a page.

See also: [Playwright async entry point](https://playwright.dev/python/docs/api/class-playwright), [`BrowserContext`](https://playwright.dev/python/docs/api/class-browsercontext).

## Stealth — What `playwright-stealth` Patches

Wrap `async_playwright()` in `Stealth().use_async(...)` for whole-process stealth (recommended), or apply per-context with `apply_stealth_async`:

```python
from playwright.async_api import async_playwright
from playwright_stealth import Stealth

async with Stealth().use_async(async_playwright()) as p:
    browser = await p.chromium.launch(headless=True, args=CHROMIUM_ARGS)
    page = await browser.new_page()
    await page.goto("https://example.com")
```

What gets patched (non-exhaustive, per the upstream README and source):

- `navigator.webdriver` (the most-checked flag)
- `navigator.plugins`, `navigator.languages`, `navigator.permissions`
- `window.chrome.runtime` and iframe contentWindow shims
- WebGL vendor / renderer strings
- Console method shims that reveal headless mode

Per-evasion control:

```python
from playwright_stealth import Stealth, ALL_EVASIONS_DISABLED_KWARGS

# Start from "all off", enable just one patch.
stealth = Stealth(**{**ALL_EVASIONS_DISABLED_KWARGS, "navigator_webdriver": True})
```

Stealth is **not bulletproof.** CloudFlare Turnstile, hCaptcha, PerimeterX, Akamai BotManager, and DataDome all detect headless via TLS fingerprint, mouse-event timing, canvas hashes, and behavioral signals that no script-injection layer can hide. If you hit a wall there, the answer is rarely "more stealth" — it is to find an API or a licensed feed.

See also: [`playwright_stealth` repo](https://github.com/Mattwmaster58/playwright_stealth).

## Page Wait Strategy

Pick the right `wait_until` mode. The four modes (`commit`, `domcontentloaded`, `load`, `networkidle`) are not interchangeable.

```python
async def fetch_rendered(browser_pool: BrowserPool, url: str) -> str:
    async with browser_pool.context() as ctx:
        page = await ctx.new_page()
        await page.goto(url, wait_until="domcontentloaded", timeout=30_000)
        # Explicit signal that the content you want has rendered.
        await page.locator("table.results").wait_for(timeout=15_000)
        return await page.content()
```

| Mode | Fires when | Use for |
|---|---|---|
| `commit` | First response received, before parsing | API-style endpoints; rare |
| `domcontentloaded` | DOMContentLoaded fires | Most SPAs — pair with explicit selector wait |
| `load` | `load` event fires (images, etc.) | Pages that need full asset load |
| `networkidle` | 500 ms with no network | **Avoid** — analytics beacons keep this hanging |

Default is `load` — override to `domcontentloaded` for speed and pair with `locator(...).wait_for(...)` for correctness.

For click-to-export buttons (CSV / PDF dump triggered by a button that initiates a download), use `expect_download` as an async context manager:

```python
async def export_csv(
    browser_pool: BrowserPool, url: str, button_selector: str
) -> bytes: ...


# Inside the page: register the download-waiter, *then* click.
async with page.expect_download(timeout=60_000) as download_info:
    await page.locator(button_selector).click()
download = await download_info.value
```

See also: [Playwright `Page` (`goto`, `expect_download`, `wait_until` modes)](https://playwright.dev/python/docs/api/class-page).

## Cookie & Session Handoff

When a site gates everything behind a JS challenge but is static after, the cheap path is: use Playwright once to clear the interstitial, export cookies, replay them on `httpx.AsyncClient`.

```python
async with browser_pool.context() as ctx:
    page = await ctx.new_page()
    await page.goto(login_url, wait_until="domcontentloaded", timeout=60_000)
    await page.locator("body[data-ready]").wait_for(timeout=30_000)
    cookies = {c["name"]: c["value"] for c in await ctx.cookies()}

response = await fetcher.get(url, cookies=cookies)
```

Refresh cookies on a 401 / 403 / cookie-expiry signal — never on a fixed schedule.

## Robots.txt And Crawl Etiquette

If you do not own the data and cannot obtain an API key, respect `robots.txt`. RFC 9309 is the spec ([rfc-editor.org/rfc/rfc9309](https://www.rfc-editor.org/rfc/rfc9309.html)).

```python
import asyncio
from urllib.robotparser import RobotFileParser
from urllib.parse import urlparse


class RobotsCache:
    def __init__(self, fetcher: CrawlerHttpClient) -> None: ...

    async def can_fetch(self, url: str, user_agent: str) -> bool: ...

    async def _parser_for(self, host: str) -> RobotFileParser:
        async with self._lock:
            if host in self._cache:
                return self._cache[host]
            parser = RobotFileParser()
            try:
                response = await self._fetcher.get(f"https://{host}/robots.txt")
                parser.parse(response.text.splitlines())
            except Exception:
                # On fetch failure, treat as permissive (matches RobotFileParser default).
                parser.parse([])
            self._cache[host] = parser
            return parser
```

See also: [RFC 9309 — Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309.html).

## Bounded Concurrency

Per-host limiters bound RPS per site; a global semaphore bounds total in-flight requests across all sites.

```python
semaphore = asyncio.Semaphore(max_concurrent)


async def one(u: str) -> bytes:
    async with semaphore:
        return (await fetcher.get(u)).content


results = await asyncio.gather(*(one(u) for u in urls), return_exceptions=True)
```

`return_exceptions=True` keeps partial successes — typical for batch crawls. With `return_exceptions=False`, the first failure cancels every sibling task; use that only when an error in any URL invalidates the whole batch.

## HTML Parsing

`bs4 + lxml` is the default. `selectolax` wins on speed (5–10×) but only on the narrow path where the extraction is CSS-selector-only — it lacks `find_parent`, `next_sibling`, lambda-based `find_all`, and the rest of bs4's tree-traversal API.

### Decide

| Situation | Use |
|---|---|
| Default — tree traversal, parent/sibling navigation, text matching, mixed extraction | `bs4 + lxml` |
| CSS-only extraction **and** scale or parse-bound | `selectolax` |
| CSS-only extraction at small scale | `bs4 + lxml` (no real win) |

CSS-only is a prerequisite, not a trigger. If extraction needs traversal, selectolax is off the table at any scale. If extraction is CSS-only but you're scraping a few hundred pages, the speedup does not pay back the migration cost.

### Migration caveat

Porting bs4 → selectolax is not 1:1. `find_parent`, `next_sibling`, `find_all(lambda ...)`, and text-content predicates have no equivalent — they require restructuring around CSS selectors. Decide upfront when scale is known.

See also: [`selectolax` repo](https://github.com/rushter/selectolax), [`beautifulsoup4` docs](https://www.crummy.com/software/BeautifulSoup/bs4/doc/).

## Testing

Three layers:

### Mock `httpx` with `respx`

```python
import httpx
import pytest
import respx


@pytest.mark.asyncio
async def test_fetcher_get_ok() -> None:
    settings = CrawlerSettings(
        user_agent="ua",
        per_host_rate=10.0,
        request_timeout_s=5.0,
        connect_timeout_s=1.0,
        max_concurrent_hosts=1,
        max_connections=1,
        max_keepalive=1,
    )
    fetcher = CrawlerHttpClient(settings)
    try:
        with respx.mock(assert_all_called=True) as router:
            router.get("https://example.com/").mock(
                return_value=httpx.Response(200, text="hello"),
            )
            response = await fetcher.get("https://example.com/")
        assert response.text == "hello"
    finally:
        await fetcher.aclose()
```

### Disable backoff in retry tests

`tenacity.wait_none()` skips sleep so retry tests run instantly.

```python
from tenacity import wait_none

# In a fixture:
monkeypatch.setattr("tenacity.nap.sleep", lambda _: None)
# Or override decorators in the function under test to use wait_none() in tests.
```

### Fixture server for Playwright

Stand up a tiny ASGI app that serves canned HTML; point Playwright at it. Never hit live sites in CI — flake rate is unbounded and your IP enters the target's block lists.

```python
import pytest
from starlette.applications import Starlette
from starlette.responses import HTMLResponse
from starlette.routing import Route
import uvicorn


async def page(_):
    return HTMLResponse("<html><body><table class='results'></table></body></html>")


app = Starlette(routes=[Route("/", page)])

# Run on a fixed port in a fixture; tear down after the test session.
```

See [`./python-tests.md`](./python-tests.md) for fixture organization and [`./langgraph.md#testing`](./langgraph.md#testing) for testing crawlers invoked from agents.

See also: [`respx` (httpx mock library)](https://github.com/lundberg/respx).

## Production Checklist

| Item | Why |
|---|---|
| Per-host `AsyncLimiter` configured | Prevents bursts that trigger 429 |
| Global `Semaphore` cap | Caps total concurrent sockets, prevents fd exhaustion |
| Realistic `User-Agent` + `Accept-Language` + `Referer` | Sites filter on plausibility, not just UA |
| `Retry-After` honored on 429 / 503 | Site-told backoff overrides default |
| Tiered timeouts (cheap / heavy / page) | Prevents slow downloads from starving the loop |
| `follow_redirects=True` only when expected | Avoid silent landing-page swaps |
| `http2=True` on `httpx.AsyncClient` | Multiplexes — fewer TCP handshakes per host |
| Browser pool size capped (memory budget) | Each Chromium context is ~200 MB RSS |
| Browsers installed at image-build time | Avoids 170 MB cold start on container boot |
| Robots.txt read + cached per host | Etiquette and audit defense |
| 4xx (other than 408 / 425 / 429) **not** retried | Hammering 401 / 403 leads to bans |
| Crawler-only logging context (host, status, elapsed, attempt) | Forensics when a site goes 503 |
| `aclose()` and `BrowserPool.stop()` in lifespan teardown | Avoids leaked connections and orphan Chromium PIDs |
| One `AsyncClient` per process | Per-call clients defeat the connection pool |
| Stealth flags applied on **all** Chromium contexts | Inconsistent stealth is worse than none |
| Per-host rate limiter created lazily inside the event loop | `AsyncLimiter` must not be created at import time |

## Pitfalls

- **Hammering 4xx.** Retrying 401 / 403 / 404 wastes time and can flag the IP for review. Whitelist retry status codes explicitly to `{408, 425, 429, 500, 502, 503, 504}`.
- **One `BrowserContext` shared across requests.** Cookies leak between callers. Open one context per request (or per session bucket). Pay the ~200 MB cost.
- **`wait_until="networkidle"`.** Analytics pixels keep the network busy effectively forever. Use `domcontentloaded` plus an explicit selector wait.
- **Synchronous HTTP from async routes.** Blocks the event loop and serializes every request. Always `httpx.AsyncClient`; if you must call a sync library, wrap with `asyncio.to_thread`.
- **Sync-only rate-limit decorators.** Decorator-based throttles built around `time.sleep` do not throttle correctly under `asyncio` — they sleep the thread, not the loop. Use `aiolimiter`.
- **Forgetting `playwright install` in the Dockerfile.** Runtime crash on first crawl, 30 s into prod. Bake browsers into the image.
- **Per-call `httpx.AsyncClient`.** Defeats connection pooling; adds a TLS handshake per request. One client per process, scoped to the lifespan.
- **Tenacity over the whole pipeline.** `fetch + parse + persist` wrapped in retry replays the persistence step on a parser bug. Wrap the inner request only.
- **Stealth as a license to scrape anything.** It is not. CloudFlare Turnstile, hCaptcha, PerimeterX, DataDome, and Akamai still flag headless via TLS fingerprint, canvas hashes, and timing — none of which `playwright_stealth` can patch.
- **Storing cookies in version-controlled config.** Leak path. Use a vault or env-injected secrets and rotate on a schedule.
- **No `Retry-After` parsing.** Honoring exponential backoff while ignoring the explicit hint the site gave you costs latency and burns the rate budget.
- **`AsyncLimiter` created at import time.** Bound to a different event loop. Always lazy-create per-host inside the running loop.
- **Per-call cookies plus cookie jar on the client.** `httpx.AsyncClient` merges them; surprising precedence rules. Pick one source of truth for cookies per call.
- **Trusting `response.raise_for_status()` to surface 429.** It does — but if you do not parse `Retry-After` first, you immediately enter exponential backoff and miss the hint.
