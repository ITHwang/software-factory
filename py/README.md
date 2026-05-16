# Python

> Last updated: 2026-05-16

## TL;DR

Python conventions and library patterns. Start at the topic group below; each doc declares its own when-to-use.

## Index

**Setup & Tooling**
- [`./py-setup-environments.md`](./py-setup-environments.md) — uv, Python version, `.env` loading.
- [`./py-setup-toolchains.md`](./py-setup-toolchains.md) — ruff, mypy, CI baseline.

**Architecture & Conventions**
- [`./py-abstractions.md`](./py-abstractions.md) — Protocol vs ABC vs mixin.
- [`./py-backend-architecture.md`](./py-backend-architecture.md) — ports/adapters, layer rules, naming.
- [`./py-data-structures.md`](./py-data-structures.md) — Pydantic, dataclass, DTOs.

**Errors & Logging**
- [`./py-errors.md`](./py-errors.md) — `CustomError`, global exception handler discipline.
- [`./py-logging.md`](./py-logging.md) — Loguru config and levels.

**Testing**
- [`./py-tests.md`](./py-tests.md) — pytest patterns, `LifespanManager`, mocking ports.

**Web & API**
- [`./dependency-injector.md`](./dependency-injector.md) — DI containers, layered composition root.
- [`./fastapi-deployment-stacks.md`](./fastapi-deployment-stacks.md) — deployment stack.
- [`./py-auth.md`](./py-auth.md) — Cognito JWT, `AuthenticationError`.
- [`./py-aws-s3.md`](./py-aws-s3.md) — S3 client + adapter example.
- [`./py-cache.md`](./py-cache.md) — Redis client + cache patterns.
- [`./py-rate-limit.md`](./py-rate-limit.md) — slowapi.
- [`./sqlalchemy.md`](./sqlalchemy.md) — `RDBClient` convention, request-scoped sessions.

**Data & I/O**
- [`./py-file-processing.md`](./py-file-processing.md) — file/PDF extraction.
- [`./py-web-crawling.md`](./py-web-crawling.md) — crawling patterns.

**LLM & AI**
- [`./langchain.md`](./langchain.md)
- [`./langfuse.md`](./langfuse.md) — LLM ports with prompt templates.
- [`./langgraph.md`](./langgraph.md)
- [`./litellm.md`](./litellm.md)
- [`./milvus.md`](./milvus.md) — vector DB client.
