# Milvus Vector Database

> Last updated: 2026-05-14

## TL;DR

How to add Milvus as the vector store for a FastAPI service via the async pymilvus SDK. PostgreSQL stays the relational source of truth; Milvus owns embeddings and similarity search.

**Use this when:**
- similarity search over embeddings (dense, sparse, hybrid)
- you need partition-key tenant isolation or per-collection lifecycle
- hybrid search combining dense vectors with BM25-style scoring

**Don't use this for:**
- relational data → `./sqlalchemy.md`
- generic full-text search (use Postgres `tsvector` or a search engine)
- simple vector search where pgvector inside Postgres or an in-memory FAISS index is easier to operate; benchmark before introducing a dedicated vector database

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Quick Reference](#quick-reference), [When To Use](#when-to-use) |
| 2. Setup | [Install](#install), [Wire Up The Client](#wire-up-the-client), [Configuration](#configuration) |
| 3. Schema | [Collections And Schema](#collections-and-schema), [Partition Keys vs Partitions](#partition-keys-vs-partitions) |
| 4. Search | [Ingest](#ingest), [Single-Field Search](#single-field-search), [Hybrid Search](#hybrid-search), [Consistency Levels](#consistency-levels) |
| 5. Operate | [Drop And Delete](#drop-and-delete), [DocumentRetriever And DocumentIndexer Ports](#documentretriever-and-documentindexer-ports), [Sync Variant](#sync-variant), [Testing](#testing), [Production Checklist](#production-checklist), [Pitfalls](#pitfalls) |

## Quick Reference

| Feature | Support | Notes |
|---------|---------|-------|
| Async client | `AsyncMilvusClient(uri=...)` | Default SDK client inside Milvus adapters for an async FastAPI service (pymilvus's SDK class — used as imported; our ports/adapters don't carry an `Async*` prefix) |
| Sync client | `MilvusClient(uri=...)` | Migration scripts and CLIs only |
| Dense vectors | `DataType.FLOAT_VECTOR` with `dim=...` | Required for ANN search |
| Sparse vectors | `DataType.SPARSE_FLOAT_VECTOR` (no `dim`) | Pair with `SPARSE_INVERTED_INDEX` |
| Index types | `IVF_FLAT`, `HNSW`, `DISKANN` | See [decision table](#collections-and-schema) |
| Hybrid search | `client.hybrid_search([req1, req2], ranker, limit)` | Multi-vector field reranking |
| Rerankers | `RRFRanker(k=60)`, `WeightedRanker(*weights)` | Rank-based vs score-based |
| Partition keys | `is_partition_key=True` on a scalar VARCHAR | Multi-tenant SaaS default |
| Manual partitions | `client.create_partition(coll, name)` | Hard cap of 1024 per collection |
| Batched ingest | `await client.insert(coll, rows[:5000])` | Tune batch size 1000–5000 |
| Consistency | `Strong`, `Bounded` (default), `Session`, `Eventually` | Set per collection |
| Local testing | `AsyncMilvusClient("./test.db")` (Milvus Lite) | SQLite-backed, no Docker |

See also: [Milvus repo](https://github.com/milvus-io/milvus), [`pymilvus` repo](https://github.com/milvus-io/pymilvus), [Milvus docs](https://milvus.io/docs), [`MilvusClient` API reference](https://milvus.io/api-reference/pymilvus/v2.6.x/MilvusClient/Client/MilvusClient.md), [`AsyncMilvusClient` API reference](https://milvus.io/api-reference/pymilvus/v2.6.x/MilvusClient/Client/AsyncMilvusClient.md), [Milvus SDK v2 native async support (blog)](https://milvus.io/blog/introducing-milvus-sdk-v2-native-async-support-unified-apis-and-superior-performance.md).

## When To Use

Pick **Milvus** when the system needs a dedicated vector database: semantic or hybrid retrieval is a primary capability, vector search should scale independently from PostgreSQL, you need multiple vector fields per row (dense + sparse), tenant partitioning, per-collection lifecycle, replicas, or operational vector-search observability.

Skip it when vector search is simple enough to keep inside the existing storage layer or process. For small or early workloads, benchmark pgvector, in-memory FAISS, and Milvus Lite before adding a standalone Milvus cluster.

## Install

```bash
uv add pymilvus
```

Add the optional extras only when the matching feature is used:

```bash
uv add "pymilvus[model]"        # local embedding helpers
uv add "pymilvus[bulk_writer]"  # offline bulk import to Milvus 2.x
```

The async client ships in the base `pymilvus` package — no separate install.

See also: [Milvus Lite repo](https://github.com/milvus-io/milvus-lite), [Introducing Milvus Lite (blog)](https://milvus.io/blog/introducing-milvus-lite.md).

## Wire Up The Client

Construct one `AsyncMilvusClient` per process, inject it into Milvus adapters through the DI container, and close it on shutdown. Use a Dependency Injector `Resource` provider so the client lifecycle ties to the FastAPI app lifespan.

### AsyncMilvusClient

#### Purpose
Pymilvus async client wrapping a single long-lived connection to a Milvus cluster, surfaced as a DI `Resource` so consumers share one connection per app instance.

#### Responsibilities
- Hold the gRPC connection to the Milvus cluster for the process lifetime.
- Expose async `query`, `insert`, `upsert`, `search`, and `hybrid_search` calls.
- Raise on RPC failures so callers can map them to domain errors.
- Flush in-flight writes and disconnect on container shutdown.
- Must not own search-strategy decisions (ranker choice, index params) — callers compose those.
- Must not own schema migrations — those live in collection bootstrap scripts.
- Must not be created per-request.

#### Lifecycle
- Scope: App Scope (`Resource`) — one `AsyncMilvusClient` per worker process.
- Created by: `providers.Resource(init_async_milvus_client, ...)` at container startup.
- Shared by: every Milvus adapter inside the same process.
- Cleaned up by: the async generator teardown in `init_async_milvus_client`, run at container shutdown.

#### Relationships
```text
[App Scope]
    └── [Container]
            └── [AsyncMilvusClient]             (app-scoped Resource)
                    │ injected into
                    ▼
               [MilvusDocumentRetriever, MilvusDocumentIndexer adapters]
                                               (app-scoped Singleton providers)
                    │ wired behind
                    ▼
               [DocumentRetriever, DocumentIndexer ports]
                    │ used by
                    ▼
               [Services] ──called by──▶ [Routes]
```

#### Constraints
- Must not be reached by routes or services directly. The `MilvusDocumentRetriever` and `MilvusDocumentIndexer` adapters hold it; routes depend on the `DocumentRetriever` or `DocumentIndexer` port via a service. See [`./python-backend-architecture.md#services-and-fastapi`](./python-backend-architecture.md#services-and-fastapi) for the route-handler DI rule.
- Must not be shared across processes (each process owns its own connection).
- Must not be torn down inside request scope.

Container wiring (the `AsyncMilvusClient` `Resource` + two adapters) lives in [./dependency-injector.md#milvus](./dependency-injector.md#milvus).

Routes depend on a service that holds the `DocumentRetriever` or `DocumentIndexer` port — never on `AsyncMilvusClient` directly. The container wires the SDK client as a singleton `Resource` and injects it into both `MilvusDocumentRetriever` and `MilvusDocumentIndexer`; services depend on whichever port they need. The Resource provider guarantees `client.close()` runs even when the app crashes during startup.

See also: [Use AsyncMilvusClient with asyncio](https://milvus.io/docs/use-async-milvus-client-with-asyncio.md), [`simple_async.py` example](https://github.com/milvus-io/pymilvus/blob/master/examples/simple_async.py).

## Configuration

| Key | Env | Default | Notes |
|-----|-----|---------|-------|
| `uri` | `MILVUS_URI` | `http://localhost:19530` | Use `https://...` in production |
| `user` | `MILVUS_USER` | `root` | Required when auth is enabled |
| `password` | `MILVUS_PASSWORD` | — | Pull from a secret store |
| `token` | `MILVUS_TOKEN` | — | Use `token` instead of `user`/`password` for Zilliz Cloud |
| `db_name` | `MILVUS_DB_NAME` | `default` | Logical isolation, cheap to create |
| `consistency_level` | `MILVUS_CONSISTENCY_LEVEL` | `Bounded` | Set per collection at create time |

`AsyncMilvusClient` accepts a `timeout=` kwarg; leave it `None` (no timeout) for ingest jobs and set explicit per-call timeouts on user-facing search.

## Collections And Schema

A collection is the unit Milvus indexes and queries. Build the schema, add fields, attach an index for every vector field, then create the collection.

```python
from pymilvus import AsyncMilvusClient, DataType


async def ensure_collection(client: AsyncMilvusClient, name: str, dim: int) -> None:
    schema = client.create_schema(auto_id=False, enable_dynamic_field=False)
    schema.add_field("strid", DataType.VARCHAR, is_primary=True, max_length=72)
    schema.add_field("tenant_strid", DataType.VARCHAR, max_length=72, is_partition_key=True)
    schema.add_field("doc_strid", DataType.VARCHAR, max_length=72)
    schema.add_field("dense", DataType.FLOAT_VECTOR, dim=dim)
    schema.add_field("sparse", DataType.SPARSE_FLOAT_VECTOR)

    index_params = client.prepare_index_params()
    index_params.add_index(
        field_name="dense",
        index_type="HNSW",
        metric_type="IP",
        params={"M": 16, "efConstruction": 200},
    )
    index_params.add_index(
        field_name="sparse",
        index_type="SPARSE_INVERTED_INDEX",
        metric_type="IP",
    )

    await client.create_collection(
        collection_name=name,
        schema=schema,
        index_params=index_params,
        partition_key_field="tenant_strid",
        num_partitions=64,
        consistency_level="Bounded",
    )
```

Pick the dense index from this table:

| Index | Best for | Build cost | Query latency | Memory |
|-------|----------|-----------|---------------|--------|
| `IVF_FLAT` | Up to ~10M vectors, no compression desired | Low | Medium | Medium |
| `HNSW` | High-recall ANN, RAM-bound | Medium | Low | High |
| `DISKANN` | Datasets larger than RAM | High | Medium | Low (disk-backed) |

`HNSW` is the sensible default. Switch to `DISKANN` when the working set exceeds available RAM; switch to `IVF_FLAT` only for tiny collections where build cost dominates.

See also: [Manage collections](https://milvus.io/docs/manage_collections.md), [Schema](https://milvus.io/docs/schema.md), [String fields](https://milvus.io/docs/string.md), [Sparse vectors](https://milvus.io/docs/sparse_vector.md).

## Partition Keys vs Partitions

Two mechanisms partition data. They solve different problems and should not be mixed in the same collection.

| Mechanism | Cardinality | Best for |
|-----------|-------------|----------|
| Manual partition (`create_partition`) | Up to 1024 per collection | A few stable, named buckets (e.g. `production`, `staging`) |
| Partition key (`is_partition_key=True`) | Unbounded tenants → hashed into `num_partitions` (default 64, max 1024) | Multi-tenant SaaS where tenants outgrow 1024 |

For multi-tenant services, mark `tenant_strid` as the partition key and filter at query time:

```python
results = await client.search(
    collection_name="documents",
    data=[query_embedding],
    anns_field="dense",
    search_params={"metric_type": "IP", "params": {"ef": 64}},
    limit=10,
    filter='tenant_strid == "tnt_01J..."',
    output_fields=["strid", "doc_strid"],
)
```

Milvus routes the query to the partition bucket that owns the tenant — IO scales with bucket size, not with total collection size.

See also: [Manage partitions](https://milvus.io/docs/manage-partitions.md), [Use partition key](https://milvus.io/docs/use-partition-key.md), [Multi-tenancy](https://milvus.io/docs/multi_tenancy.md).

## Ingest

Use `insert` for new rows and `upsert` for idempotent writes. Batch in chunks of 1000–5000 rows and parallelise batches with `asyncio.gather` when throughput matters.

```python
import asyncio
from pymilvus import AsyncMilvusClient


type SparseVector = dict[int, float]
type MilvusDocumentRow = dict[str, str | list[float] | SparseVector]


async def ingest(
    client: AsyncMilvusClient,
    coll: str,
    rows: list[MilvusDocumentRow],
    batch_size: int = 2000,
) -> None:
    batches = [rows[i : i + batch_size] for i in range(0, len(rows), batch_size)]
    await asyncio.gather(*(client.upsert(coll, b) for b in batches))
```

Each row is a plain dict matching the schema:

```python
row = {
    "strid": "doc_01J...",
    "tenant_strid": "tnt_01J...",
    "doc_strid": "doc_01J...",
    "dense": [0.12, -0.04, ...],
    "sparse": {12: 0.7, 198: 0.3, 4221: 0.1},
}
```

Call `await client.flush(coll)` only when a reader needs to see the writes immediately under `Bounded` consistency — flushing seals segments and is expensive on hot paths.

See also: [Insert / update / delete](https://milvus.io/docs/insert-update-delete.md).

## Single-Field Search

`AsyncMilvusClient.search` is the primary ANN entry point. Pass one query vector (or a batch), the field to search, and search params tuned for the index type.

```python
results = await client.search(
    collection_name="documents",
    data=[query_embedding],
    anns_field="dense",
    search_params={"metric_type": "IP", "params": {"ef": 64}},
    limit=10,
    filter='tenant_strid == "tnt_01J..." and doc_strid != "doc_01J..."',
    output_fields=["strid", "doc_strid"],
)

hits = [(hit.get("entity", {}).get("strid"), hit["distance"]) for hit in results[0]]
```

Search-param keys depend on the index:

| Index | Param | Typical |
|-------|-------|---------|
| `HNSW` | `ef` | 32–128 (latency / recall trade-off) |
| `IVF_FLAT` | `nprobe` | 8–32 |
| `DISKANN` | `search_list` | 32–100 |
| `SPARSE_INVERTED_INDEX` | `drop_ratio_search` | 0.0–0.3 (drop low-weight terms) |

See also: [Index](https://milvus.io/docs/index.md), [Index explained](https://milvus.io/docs/index-explained.md).

## Hybrid Search

Hybrid search runs one ANN request per vector field and reranks the merged result. Build one `AnnSearchRequest` per field, pick a ranker, and call `hybrid_search`.

```python
from pymilvus import AnnSearchRequest, RRFRanker, WeightedRanker


dense_req = AnnSearchRequest(
    data=[dense_q],
    anns_field="dense",
    param={"metric_type": "IP", "params": {"ef": 64}},
    limit=20,
)
sparse_req = AnnSearchRequest(
    data=[sparse_q],
    anns_field="sparse",
    param={"metric_type": "IP", "params": {"drop_ratio_search": 0.2}},
    limit=20,
)

rrf_results = await client.hybrid_search(
    collection_name="documents",
    reqs=[dense_req, sparse_req],
    ranker=RRFRanker(k=60),
    limit=10,
    output_fields=["strid", "doc_strid"],
)

weighted_results = await client.hybrid_search(
    collection_name="documents",
    reqs=[dense_req, sparse_req],
    ranker=WeightedRanker(0.7, 0.3),
    limit=10,
    output_fields=["strid", "doc_strid"],
)
```

Pick the ranker:

| Ranker | Strategy | Pick when |
|--------|----------|-----------|
| `RRFRanker(k=60)` | Rank-based; ignores absolute scores | Fields use different metrics or scores are not comparable |
| `WeightedRanker(*weights)` | Score-based; weights sum to 1.0 | Relative importance of dense vs sparse is calibrated |

Per-request `limit` should be 2–4× the final `limit` so the reranker has enough candidates to work with.

See also: [Hybrid search](https://milvus.io/docs/hybrid_search.md), [Multi-vector search](https://milvus.io/docs/multi-vector-search.md), [RRF ranker](https://milvus.io/docs/rrf-ranker.md), [Reranking](https://milvus.io/docs/reranking.md), [`hybrid_search.py` example](https://github.com/milvus-io/pymilvus/blob/master/examples/hybrid_search.py), [`sparse.py` example](https://github.com/milvus-io/pymilvus/blob/master/examples/sparse.py).

## Consistency Levels

| Level | Latency | Visibility | When |
|-------|---------|-----------|------|
| `Strong` | Highest | Read-after-write across all nodes | Financial, audit, tests |
| `Bounded` (default) | Medium | Stale by a bounded window (~hundreds of ms) | Production default |
| `Session` | Low | Read-after-write within the same client | Single-session UX |
| `Eventually` | Lowest | Best effort | Batch analytics, offline jobs |

Set the level at collection creation. Override per request via the `consistency_level=` argument on `search` and `query` when one path needs different semantics:

```python
await client.search(
    collection_name="documents",
    data=[q],
    anns_field="dense",
    limit=10,
    consistency_level="Strong",  # one-off override
)
```

Use `Bounded` everywhere except tests; use `Strong` in tests so assertions do not race the replication window.

See also: [Consistency](https://milvus.io/docs/consistency.md).

## Drop And Delete

| Operation | Call | Scope |
|-----------|------|-------|
| Drop a collection | `await client.drop_collection(name)` | Destructive, full collection |
| Drop manual partition | `await client.drop_partition(coll, name)` | Manual partitions only — partition keys are managed by Milvus |
| Delete by expression | `await client.delete(coll, filter='strid in ["a","b"]')` | Row-level, supports filter expressions |
| Soft delete | `is_deleted: BOOL` field + filter at query time | Cheaper for high-churn workloads; vacuum periodically |

```python
await client.delete(
    collection_name="documents",
    filter='tenant_strid == "tnt_01J..." and doc_strid == "doc_01J..."',
)
```

Soft-delete pattern beats hard delete when the same rows churn many times per day — Milvus rewrites segments on compaction and reclaims space without per-row index work.

## DocumentRetriever And DocumentIndexer Ports

Define two ports so application code speaks in document-indexing and document-retrieval use cases, not Milvus collections and filter expressions. The retriever is consumed by search-side services; the indexer is consumed by ingestion-side workers. Most services need only one of the two, so splitting prevents the write surface from leaking into search paths. See [`./python-backend-architecture.md#naming-ports`](./python-backend-architecture.md#naming-ports) for the CQRS-style port-split convention.

```python
from dataclasses import dataclass
from typing import Protocol


type ScalarMetadata = str | int | float | bool


@dataclass(frozen=True)
class IndexedDocumentChunk:
    tenant_strid: str
    document_strid: str
    chunk_strid: str
    text: str
    dense_embedding: list[float]
    sparse_embedding: dict[int, float] | None = None
    metadata: dict[str, ScalarMetadata] | None = None


@dataclass(frozen=True)
class DocumentSearchQuery:
    tenant_strid: str
    dense_embedding: list[float]
    sparse_embedding: dict[int, float] | None = None
    limit: int = 10
    exclude_document_strids: list[str] | None = None


@dataclass(frozen=True)
class DocumentSearchHit:
    chunk_strid: str
    document_strid: str
    score: float
    text: str | None = None
    metadata: dict[str, ScalarMetadata] | None = None


class DocumentRetriever(Protocol):
    async def search_similar_chunks(self, query: DocumentSearchQuery) -> list[DocumentSearchHit]: ...


class DocumentIndexer(Protocol):
    async def index_chunks(self, chunks: list[IndexedDocumentChunk]) -> None: ...

    async def delete_document_chunks(
        self,
        *,
        tenant_strid: str,
        document_strid: str,
    ) -> None: ...
```

Adapters: `MilvusDocumentRetriever` (implements `DocumentRetriever`) and `MilvusDocumentIndexer` (implements `DocumentIndexer`). Both hold the same singleton `AsyncMilvusClient` `Resource` from the container. Collection names, schema DDL, Milvus filter strings, batching details, and raw SDK result shapes stay inside the adapters or collection-bootstrap scripts. Upper layers depend on `DocumentRetriever` or `DocumentIndexer`, never on `AsyncMilvusClient`.

The same shape applies to any future vector store: `[Implementation]DocumentRetriever` + `[Implementation]DocumentIndexer`. The `Document` domain noun keeps the names generic across project domains that share Milvus.

## Sync Variant

`MilvusClient` (sync) is fine for migration scripts, Alembic-style data backfills, and Typer CLIs that run outside the FastAPI process. Never call it from inside an async route — every method blocks the event loop and starves concurrent requests.

```python
# scripts/backfill.py
from pymilvus import MilvusClient


client = MilvusClient(uri="http://localhost:19530")
try:
    client.upsert("documents", rows)
finally:
    client.close()
```

If a script must run inside the same process as the API (e.g. a startup migration), spawn it on a thread with `asyncio.to_thread` so it does not block the loop.

Async-vs-sync is a pymilvus SDK choice — two SDK classes ship for the two execution models. The `DocumentRetriever` / `DocumentIndexer` port shape doesn't change between them; the adapter's method signatures (`async def` vs plain `def`) carry the distinction.

## Testing

Three layers, three approaches:

| Layer | Approach |
|-------|----------|
| Unit tests of business logic | Mock the `DocumentRetriever` / `DocumentIndexer` ports — no real Milvus |
| Integration tests of the adapter | Milvus Lite via `AsyncMilvusClient("./test.db")` — no Docker |
| End-to-end | Standalone Milvus via Docker Compose, `consistency_level="Strong"` |

```python
# tests/conftest.py
from collections.abc import AsyncIterator
from pathlib import Path

import pytest_asyncio
from pymilvus import AsyncMilvusClient


@pytest_asyncio.fixture
async def milvus(tmp_path: Path) -> AsyncIterator[AsyncMilvusClient]:
    client = AsyncMilvusClient(uri=str(tmp_path / "milvus.db"))
    try:
        yield client
    finally:
        await client.close()
```

Milvus Lite supports the full `AsyncMilvusClient` surface for collections, ingest, search, and hybrid search. It does not support distributed features (replicas, partition rebalancing) — those need standalone Milvus.

Use `consistency_level="Strong"` in tests so assertions do not race writes.

## Production Checklist

| Item | Setting |
|------|---------|
| Transport | `uri="https://..."` with one-way TLS at minimum |
| Auth | `user` / `password` or `token` from a secret store, never hard-coded |
| Index (dense) | `HNSW` with `M=16..32`, `efConstruction=200..400` (or `DISKANN` when working set > RAM) |
| Index (sparse) | `SPARSE_INVERTED_INDEX` with `metric_type="IP"` |
| Replicas | 2+ replicas via Milvus distributed mode for HA reads |
| Segment size | Tune `dataCoord.segment.maxSize` (default 1024 MB) for ingest patterns |
| Monitoring | Prometheus `milvus_*` metrics into the standard Grafana dashboard |
| Connection pool | One `AsyncMilvusClient` per process via DI Resource provider |
| Consistency | `Bounded` everywhere except tests |
| Backup | Periodic snapshot with `milvus-backup` or object-store replication on the MinIO/S3 layer |

See also: [Limit collection counts](https://milvus.io/docs/limit_collection_counts.md), [Limitations](https://milvus.io/docs/limitations.md), [TLS](https://milvus.io/docs/tls.md).

## Pitfalls

- **VARCHAR primary key `max_length` is in bytes, not characters.** UTF-8 multi-byte chars consume more — set `max_length=72` for ULID safely, but verify encoding before going below that.
- **`SPARSE_FLOAT_VECTOR` does not take `dim`.** Sparse fields are dimensionless; passing `dim=...` raises a schema error.
- **Stale reads under `Bounded` consistency.** Fresh writes can be invisible to a search for hundreds of milliseconds. Either call `await client.flush(coll)` before the search or accept the staleness window.
- **Manual partitions cap at 1024 per collection.** Past ~64 tenants, switch to a partition key — manual partitions are a foot-gun for SaaS workloads.
- **Sync `MilvusClient` inside an async route.** Blocks the event loop. Milvus adapters in async services should hold `AsyncMilvusClient`; if a sync call is unavoidable, wrap it in `asyncio.to_thread`.
- **Forgetting to `await client.close()` on shutdown.** Leaks the gRPC channel. Use the DI `Resource` lifecycle, not a module-level global.
- **Tests with `consistency_level="Bounded"`.** Flaky. Use `Strong` for test collections; the latency cost is irrelevant in tests.
- **Mixing rankers in one call.** `RRFRanker(k=60)` and `WeightedRanker(0.7, 0.3)` cannot both be passed to `hybrid_search` — pick one ranker per request.
- **Per-request `limit` smaller than the final `limit`.** Starves the reranker. Set per-request `limit` to 2–4× the final `limit`.
- **Reusing a partition-keyed collection with manual partitions.** Mixing the two in one collection produces undefined routing. Pick one mechanism per collection.
