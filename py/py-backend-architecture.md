# Python Backend Architecture (Ports & Adapters)

> Last updated: 2026-05-16

## TL;DR

Lightweight ports-and-adapters. Three layers: **entities**, **services + ports**, **adapters + clients**, fronted by **apis**. Ports are named after the capability the application needs (`TextSummarizer`, not `LLMGateway`). Adapters are named `[Implementation][Capability]` (`OpenAITextSummarizer`). Routes are infrastructure; service code never imports from `fastapi`.

**Use this when:**
- laying out a new project's `app/` skeleton: top-level folders, global wiring files, entry point
- starting a new domain package and laying out `apis/services/ports/entities/adapters`
- deciding whether a new outbound dependency deserves a port or should call a client directly
- naming a new port or adapter
- reviewing a PR for layer/dependency violations

**Don't use this for:**
- the Protocol vs ABC vs mixin decision -> `./abstractions.md`
- persistence-layer mechanics (SQLAlchemy, repositories, transactions) -> `./sqlalchemy.md`
- LLM capability ports with Langfuse-backed prompt templates -> `./langfuse.md`
- DI container wiring details -> `./dependency-injector.md`
- tooling baseline (Python version, ruff, FastAPI, pytest) -> `./README.md`

## Table of Contents

| Section |
|---------|
| [Quick Reference](#quick-reference) |
| [Project Skeleton](#project-skeleton) |
| [Layer Layout](#layer-layout) |
| [Layer Responsibilities](#layer-responsibilities) |
| [Naming: Ports](#naming-ports) |
| [Naming: Adapters](#naming-adapters) |
| [When To Use Ports](#when-to-use-ports) |
| [Services And FastAPI](#services-and-fastapi) |
| [See Also](#see-also) |
| [Anti-Patterns](#anti-patterns) |

## Quick Reference

| Layer | Role |
|-------|------|
| `apis/` | HTTP transport: routers, request/response schemas, dependency wiring. |
| `services/` | Business orchestration, transaction boundaries, calls into ports. |
| `ports/` | `Protocol`-based contracts owned by the business layer. |
| `entities/` | Domain objects and DTOs, independent of HTTP, DB, or SDK shapes. |
| `adapters/` | Infrastructure implementations of ports (`PostgresUserRepository`, `OpenAITextSummarizer`). |
| `common/` | Shared, domain-agnostic infrastructure: low-level SDK/DB clients, shared error types, FastAPI middleware/dependencies. |

## Project Skeleton

What sits above a domain: the entry point, the package, and the global wiring files that compose domains into a running FastAPI app.

```text
<project-root>/
├── run.py                   # entry point: env validation + uvicorn launch
├── pyproject.toml
├── tests/
└── app/
    ├── __init__.py
    ├── common/              # shared, domain-agnostic infra
    │   ├── clients/         # RDBClient, LiteLLMClient, S3Client, ...
    │   ├── errors/          # CustomError hierarchy + global exception handlers
    │   └── fastapi/         # middleware, request-scoped dependencies
    ├── config/              # env + yaml settings loading
    ├── domains/
    │   ├── users/
    │   ├── documents/
    │   └── chats/
    ├── utils/               # pure-function helpers
    ├── containers.py        # global DI container (composes domain containers, wires APIs)
    ├── routers.py           # aggregates domain routers under prefixes + tags
    └── server.py            # FastAPI app factory, lifespan, exception-handler registration
```

- **`run.py`** — process entry. Loads `.env`, validates `API_ENV`, calls `uvicorn.run`. Typically a `click` CLI exposing `--port` / `--ssl-keyfile` options.
- **`app/common/`** — shared infra. Holds low-level `clients/`, the `CustomError` hierarchy + global handlers, and FastAPI middleware/dependencies. No business logic. Never imports from `app/domains/`.
- **`app/config/`** — settings loading (env vars + yaml). Other layers receive config via the DI container; they do not import `config` directly.
- **`app/domains/`** — one folder per business domain. Each follows the [layer layout](#layer-layout) below.
- **`app/utils/`** — pure-function helpers. No I/O, no DI, no domain types.
- **`app/containers.py`** — global DI container. Loads config, composes per-domain sub-containers, declares wiring for the API packages.
- **`app/routers.py`** — aggregates each domain's router under its prefix (e.g., `/api/v1`) and tag. Single import point for `server.py`.
- **`app/server.py`** — FastAPI app factory. Registers middleware, mounts `app_router`, wires global exception handlers, owns the `lifespan` (startup/shutdown of long-lived resources like DB pools or MCP sessions).

## Layer Layout

Domain-scoped folders under `app/domains/`. Each business domain (`users`, `documents`, `chats`, `payments`, ...) gets its own slice of the five inner layers.

```text
app/domains/
├── users/
│   ├── apis/         # routers, request/response schemas
│   ├── services/     # orchestration, txn boundaries
│   ├── ports/        # Protocol contracts (UserRepository, ...)
│   ├── entities/     # User, UserDTO, value objects
│   └── adapters/     # PostgresUserRepository, SESEmailSender
├── documents/
│   ├── apis/
│   ├── services/
│   ├── ports/        # DocumentRepository, DocumentExtractor, ...
│   ├── entities/
│   └── adapters/     # S3DocumentRepository, PdfDocumentExtractor
└── chats/
    ├── apis/
    ├── services/
    ├── ports/        # ChatCompleter, ChatHistoryRepository, ...
    ├── entities/
    └── adapters/     # OpenAIChatCompleter, PostgresChatHistoryRepository
```

## Layer Responsibilities

### apis

- HTTP transport only: routers, Pydantic request/response schemas, `Depends(...)` wiring.
- Translate domain results into response schemas; translate domain errors into HTTP responses via global exception handlers.
- May pass request schemas directly into services when most fields are needed; do not unpack into kwargs just to "hide" the schema.
- Own the `fastapi` import surface for the package. Anything that touches `Depends`, `Request`, `HTTPException`, or `status` lives here.

### services

- Business orchestration: call ports, sequence work, decide on commits and rollbacks.
- Receive request DTOs/schemas as plain arguments; return domain objects (entities or DTOs), never response schemas.
- Own transaction boundaries (commit points are a business decision, not an infrastructure one). See `./sqlalchemy.md` for the `RDBClient`-threaded pattern.
- **Must not import from `fastapi`** (no `Depends`, `Request`, `HTTPException`, `status`). Raise domain errors (`CustomError` subclasses) instead.

### ports

- `Protocol`-based contracts the business layer needs from the outside world.
- Owned by the business layer: signatures express domain intent (`summarize(text: str) -> str`), not vendor primitives.
- Take and return domain types only. No SDK objects, no HTTP response classes, no SQLAlchemy rows leak into a port signature.
- Tend to be small (3-8 methods). If a port grows large, the capability is probably two capabilities.

### entities

- Domain objects: dataclasses, Pydantic models used as DTOs, value objects, domain enums.
- Independent of transport, persistence, and SDKs. An entity does not know whether it came from Postgres or an HTTP body.
- Shared across `services`, `ports`, and `adapters` within the same domain.

### adapters

**Purpose**: Translate between domain types defined by ports and vendor-specific SDK types.

**Responsibilities**:
- Own SDK / infrastructure construction and lifecycle (or receive a low-level client from `clients/`).
- Map domain types (on the port surface) to and from vendor types (internally).
- Catch vendor-specific errors and re-raise them as domain errors (`CustomError` subclasses).
- Hold infrastructure state (configured model name, prompt label, regional endpoint). Stateful adapters are fine; they're infrastructure, not entities.

**Constraints**:
- Must not leak vendor types (SDK request/response classes, ORM rows, HTTP response objects) across the port surface into services or ports.
- Must not contain business logic — orchestration, sequencing, transaction decisions belong in services.
- Must not be called directly by routes. Routes obtain a service from DI; the service holds the port.

- Concrete implementations of ports. One adapter per `(implementation, capability)` pair.
- Translate between domain types (on the port surface) and vendor types (internally).
- Construct the infrastructure they need (LLM SDK clients, HTTP clients) or receive a low-level client from `clients/`.
- Hold infrastructure state (configured model name, prompt label, regional endpoint). Stateful adapters are fine; they're infrastructure, not entities.

### clients

**Purpose**: Provide shared, domain-agnostic wrappers around infrastructure SDKs and connections that adapters and (where no port is warranted) services consume.

**Responsibilities**:
- Wrap a single piece of infrastructure (DB session executor, SDK, HTTP transport) with a minimal, capability-neutral surface.
- Own infrastructure-level concerns: connection construction, session lifecycle, retry/timeout primitives.
- Translate transport-level errors into a stable shape adapters can map onto domain errors.

**Constraints**:
- Must not encode business contracts — a client is `LiteLLMClient`, not `TextSummarizer`. Domain intent belongs in ports.
- Must not live inside a domain folder. Clients are shared infrastructure under `app/common/clients/`.
- Must not import from any domain package (`users/`, `documents/`, ...). The dependency direction is domain -> client, never the reverse.

- Lightweight wrappers around infrastructure: DB session executors (`RDBClient`), SDK wrappers (`LiteLLMClient`), HTTP clients.
- Not tied to a business contract. Services may use a client directly when no port abstraction is warranted (see [When To Use Ports](#when-to-use-ports)).
- Shared across domains. Live under `app/common/clients/`, not inside a domain folder.

## Naming: Ports

Capability-oriented. The name describes what the application asks for, not what infrastructure provides.

| Port | Capability |
|------|-----------|
| `UserRepository` | Persistence and CRUD for users. |
| `EmailSender` | Send an email. (Not `EmailGateway`.) |
| `TextSummarizer` | Summarize text. (Not `LLMGateway`.) |
| `EmbeddingGenerator` | Produce embeddings for text. |
| `PaymentProcessor` | Charge a payment method. |

Rules:

- **`Repository` is reserved for persistence/CRUD.** Don't use it for non-persistence concerns ("a `PromptRepository` that fetches templates" is the warning sign).
- **All other outbound ports use the capability name.** Verb-shaped (`EmailSender`, `TextSummarizer`) or role-shaped (`PaymentProcessor`).
- **Generic `Gateway` is deprecated.** It encodes "the thing on the other side of the wall" instead of the capability the business needs. Use the capability name (verb-shaped like `EmailSender`, or role-shaped like `PaymentProcessor`). When a clean capability noun is hard to find, the `*Port` postfix on the port (and `*Adapter` postfix on the adapter) is acceptable — both pure (`EmailSender`) and postfixed (`EmailSenderPort`) are valid; pick whichever reads naturally for the capability.
- **Port shape is use-case-driven.** Small ports (1–5 methods) are normal in ports-and-adapters; each port reflects a use case. When read consumers and write consumers diverge strongly, split the port into a pair — `*Retriever` + `*Indexer`, `*Reader` + `*Writer`, or any natural read/write division. Default to unified when callers always touch both sides.
- **`Repository` is the conservative default for persistence; it CAN be split** when read and write use cases diverge (e.g., a search-only service and an ingest-only worker should not share a CRUD port). The underlying principle is the same: ports are use-case-shaped, and small ports are not a violation.
- **Don't prefix our ports/adapters with `Async*` / `Sync*`.** Async-vs-sync lives at the method signature (`async def search(...)`). SDK-imposed names (e.g., pymilvus's `AsyncMilvusClient`) are used as imported — we don't rename what we don't own — but our own ports and adapters stay capability-named.

## Naming: Adapters

Convention: **`[Implementation][Capability]`**. The implementation prefix is the vendor, technology, or variant; the capability is the port name.

| Adapter | Port |
|---------|------|
| `PostgresUserRepository` | `UserRepository` |
| `SESEmailSender`, `ResendEmailSender` | `EmailSender` |
| `OpenAITextSummarizer`, `ClaudeTextSummarizer` | `TextSummarizer` |
| `VoyageEmbeddingGenerator`, `OpenAIEmbeddingGenerator` | `EmbeddingGenerator` |
| `StripePaymentProcessor` | `PaymentProcessor` |

- **`*Adapter` suffix is opt-in signaling.** The default adapter shape is `[Implementation][Capability]` without a suffix (`OpenAITextSummarizer`, not `OpenAITextSummarizerAdapter`). Use the `*Adapter` suffix only when the class is explicitly the **GoF Adapter Pattern** — wrapping a third-party SDK / library / external resource to match a port shape — and you want the name to signal that intent. Pure capability implementations (in-memory, mocks, project-internal) don't need the suffix; the architecture's port-adapter role is implicit.

Variants beyond vendor swap:

- **Test doubles:** `MockTextSummarizer`, `MockUserRepository`. Prefer `Mock*` over `Fake*` — it matches the stdlib `unittest.mock` vocabulary the project's tests already use, and reads naturally as the test-context indicator.
- **Experimentation:** `V1Summarizer`, `V2Summarizer` when the business cares which generation answered.
- **Composition variants:** `PrimaryTextSummarizer` and `FallbackTextSummarizer` composed by a service when one provider acts as the failover for another.

## When To Use Ports

Two criteria. **Both must hold** — if neither does, skip the port and have the service call a client (or a vendor SDK) directly.

1. **The business layer should own the contract.** Business requirements evolve first; infrastructure follows. If the shape of the call would change because the business asked for a new capability (not because we swapped vendors), it belongs behind a port.
2. **The external dependency should be replaceable.** PostgreSQL -> DocumentDB, SES -> SendGrid, OpenAI -> Claude. If the dependency is effectively fixed for the life of the project (the standard library, a config file format, a logger), a port is overhead.

Lightweight infra clients (DB session wrappers, SDK wrappers, HTTP clients) live in `app/common/clients/`. Unlike adapters, clients aren't tied to business contracts and may be used directly by services when no port abstraction is warranted. The split: ports express what the business needs; clients express what infrastructure offers.

## Services And FastAPI

Services are framework-agnostic. The boundary is sharp.

- Services receive request data as plain DTOs / Pydantic schemas / primitive args, not as `Request` objects.
- Services do not depend on `fastapi.Depends`, `fastapi.HTTPException`, or `fastapi.status` — they raise domain errors (`CustomError` subclasses, see [`./py-errors.md`](./py-errors.md)) and let global exception handlers translate to HTTP.
- Routes translate domain results into response schemas and domain errors into HTTP responses. The route is the only layer that legally imports from `fastapi`.
- A grep for `from fastapi` in `services/` should return nothing. Treat hits as review blockers.

**Route handler dependency rules.** Routes hold one thing by default: the DI container. Services are obtained via `Depends(Provide[Container.user_service])` (or equivalent). Adapters, clients, SDK objects, and entity types do not appear in route signatures — services own those.

**Lifecycle scope rules.** App-scoped objects (`Singleton` / `Resource`) are shared for one worker-process app lifespan and must not retain request-scoped mutable state. Request scope has two implementation shapes:

- **Strict request scope:** FastAPI `Depends` / `yield` dependencies own request-derived state and cleanup-bound state.
- **Request-owned Factory:** `providers.Factory` may build a service/repository for the request path, but `Factory` means new-per-call, not one-per-request. The request path must resolve it once and pass it downward when one instance is required.

**Strict request-scope exceptions.** Two cases use FastAPI's `Depends(...)` instead of plain `Provide`, because their lifecycle is tied to the request itself, not to the app-scoped container:

- **`RDBClient`** (per-request transactional session). Routes receive it via `Depends(get_rdb_client)` and thread it into service method calls. See [`./sqlalchemy.md`](./sqlalchemy.md) for the worked pattern.
- **Auth context** (`CurrentUser`, derived from the request's bearer token). Routes receive it via `Depends(get_current_user)`. See [`./py-auth.md`](./py-auth.md) for the worked pattern.

Both exceptions exist because the underlying state is **strict request-scoped**. Everything else (app-scoped singletons/resources, request-owned factory services, services holding ports) flows through `dependency-injector`'s `@inject` + `Provide`.

## See Also

- [./abstractions.md](./abstractions.md) — Protocol vs ABC vs mixin decision.
- [./aws-s3.md](./aws-s3.md) — worked Repository + adapter example.
- [./dependency-injector.md](./dependency-injector.md) — DI wiring patterns.
- [./sqlalchemy.md](./sqlalchemy.md) — persistence layer in this style (`RDBClient`, `UserRepository`).
- [./langfuse.md](./langfuse.md) — LLM capability ports with Langfuse-backed prompt templates.
- [./multi-project-management-with-uv.md](./multi-project-management-with-uv.md) — multi-project repo layout with uv path dependencies / workspaces.
- [./how-to-run-mcp-servers.md](./how-to-run-mcp-servers.md) — three modes for running MCP servers from a Python app.
- [./README.md](./README.md) — Python folder index.
- [./py-errors.md](./py-errors.md) — `CustomError` discipline and exception-handling convention.

## Anti-Patterns

1. **Generic `*Gateway` names for new code.** Pick the capability name (`EmailSender`, `TextSummarizer`). Reserve domain framing for the port; reserve vendor framing for the adapter.
2. **`Repository` for non-persistence concerns.** A "`PromptRepository`" that loads templates is a capability port wearing a persistence costume — name it for the capability (`TextSummarizer`) and let the adapter own the template-loading detail.
3. **Service imports from `fastapi`.** Anything from `fastapi` — `Depends`, `Request`, `HTTPException`, `status` — belongs in routes. Services raise domain errors.
4. **A port for everything.** If neither business-ownership nor replaceability applies, the port is dead weight. Call a client (or the SDK) directly from the service.
5. **Adapters bleeding vendor types into ports.** Port signatures take and return domain types; the adapter does the translation. If the port has `OpenAIResponse` in a return type, the port is wrong.
6. **Constructing vendor SDKs in services.** Services receive ports and clients via DI; they don't `import openai` or `import boto3`. That import surface lives in adapters and clients.
