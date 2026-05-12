# AWS S3

> Last updated: 2026-05-12

## TL;DR

An async `S3Client` wrapping `[aioboto3](https://aioboto3.readthedocs.io/en/latest/)` (15+) with a small, opinionated surface — bucket validation, file I/O, presigned URLs, existence checks, and deletion. Business code calls short, intention-revealing methods; anything beyond this surface drops down to the raw `aioboto3` client.

**Use this when:**

- a FastAPI service uploads or downloads objects from S3 or an S3-compatible store (MinIO, R2, LocalStack, Wasabi)
- you need short-lived presigned URLs for browser GET or PUT

**Don't use this for:**

- serializing Python objects → use `orjson`, `msgpack`, or `np.save` / `np.load` and call `upload_file` with the resulting bytes

## Table of Contents


| Phase       | Section                                                                                                                                                                 |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Concepts | [Quick Reference](#quick-reference), [The Wrapper Shape](#the-wrapper-shape)                                                                                            |
| 2. Setup    | [Install](#install), [Wiring Through The Container](#wiring-through-the-container)                                                                                      |
| 3. Auth     | [Credentials And Region](#credentials-and-region), [Bucket Validation And The Default Bucket](#bucket-validation-and-the-default-bucket)                                |
| 4. I/O      | [Uploading And Downloading Files](#uploading-and-downloading-files), [Presigned URLs](#presigned-urls), [Existence Checks And Deletion](#existence-checks-and-deletion) |
| 5. Operate  | [Testing](#testing), [Pitfalls](#pitfalls)                                                                                                                              |


## Quick Reference


| Method                       | Purpose                           | Underlying call                                   | Reference                                                                                                                           |
| ---------------------------- | --------------------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `create_bucket`              | Region-aware bucket creation      | `s3.create_bucket`                                | [boto3 `create_bucket](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/create_bucket.html)`                   |
| `set_bucket`                 | Validate and pin a default bucket | `s3.head_bucket`                                  | [boto3 `head_bucket](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/head_bucket.html)`                       |
| `upload_file`                | Upload a file-like object         | `s3.upload_fileobj`                               | [boto3 `upload_fileobj](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/upload_fileobj.html)`                 |
| `download_file`              | Stream an object to disk          | `s3.download_file`                                | [boto3 `download_file](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/download_file.html)`                   |
| `download_bytes`             | Load an object into memory        | `s3.get_object` + `body.read()`                   | [boto3 `get_object](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/get_object.html)`                         |
| `generate_presigned_get_url` | Signed GET URL                    | `s3.generate_presigned_url("get_object", ...)`    | [boto3 `generate_presigned_url](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/generate_presigned_url.html)` |
| `generate_presigned_put_url` | Signed PUT URL (browser upload)   | `s3.generate_presigned_url("put_object", ...)`    | [presigned URL guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)                          |
| `object_exists`              | Object existence check (bool)     | `s3.head_object`                                  | [boto3 `head_object](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/head_object.html)`                       |
| `delete_object`              | Delete a single object            | `s3.delete_object`                                | [boto3 `delete_object](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/delete_object.html)`                   |
| `delete_all_objects`         | Paginated batch delete            | `list_objects_v2` paginator + `s3.delete_objects` | [boto3 `delete_objects](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/delete_objects.html)`                 |


See also: `[aioboto3` docs]([https://aioboto3.readthedocs.io/en/latest/](https://aioboto3.readthedocs.io/en/latest/)), `[aioboto3` repo]([https://github.com/terricain/aioboto3](https://github.com/terricain/aioboto3)), [boto3 S3 client reference](https://docs.aws.amazon.com/boto3/latest/reference/services/s3.html), [AWS S3 user guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html).

## Install

```bash
uv add "aioboto3>=15"
uv add --dev types-aiobotocore-s3   # optional: type stubs for the underlying client
```

`aioboto3` 15+ pulls in current `aiobotocore` and `botocore` transitively. The stubs are optional and only needed for autocomplete on the low-level `aioboto3.S3Client` type returned from `session.client("s3", ...)`.

See also: `[aioboto3` changelog]([https://aioboto3.readthedocs.io/en/latest/changelog.html](https://aioboto3.readthedocs.io/en/latest/changelog.html)), `[aiobotocore` repo]([https://github.com/aio-libs/aiobotocore](https://github.com/aio-libs/aiobotocore)), `[types-aiobotocore-s3` on PyPI]([https://pypi.org/project/types-aiobotocore-s3/](https://pypi.org/project/types-aiobotocore-s3/)).

## The Wrapper Shape

### Purpose

An async S3 wrapper around `aioboto3` that owns the session-and-client lifecycle plumbing and exposes a narrow surface — upload, download, presign, exists, delete — to repositories and services.

### Responsibilities

- Hold a reference to the already-opened low-level `aioboto3` client provided by the container, plus the active region and (after `set_bucket`) an optional default-bucket name.
- Surface per-call S3 operations through short, intention-revealing methods.
- Validate the default bucket via `head_bucket` and resolve per-call bucket arguments against it.
- Generate presigned GET / PUT URLs with the right `signature_version` and addressing style.
- Map `botocore.ClientError` codes to wrapper-level booleans (`object_exists`) or domain errors; never silently swallow non-404 failures.
- **Must not** own credential discovery — delegate to the boto credential chain and environment.
- **Must not** own bucket-naming conventions — callers decide names; the wrapper only validates and resolves.
- **Must not** be per-request — one `aioboto3.Session` per process; the per-request unit is the client context manager around it.

### Lifecycle

- Scope: app-scoped `Singleton` for the wrapper class itself; the `aioboto3.Session` it depends on is also app-scoped `Singleton`; the low-level client is opened by the container as a `Resource` (async context manager) per process.
- Created by: the DI container at startup, after the `aioboto3.Session` `Singleton` resolves and the client `Resource` opens.
- Shared by: repository adapters (`S3UserRepository`, `S3DocumentRepository`, …) injected into business-layer services.
- Cleaned up by: the container — it closes the client `Resource` at shutdown. The `aioboto3.Session` has no explicit teardown; it dies with the process.

### Relationships

```text
[App Scope]
    ├── [aioboto3.Session]      (singleton)
    └── [aioboto3 S3 client]    (singleton Resource, async context manager)
            │ injected into
            ▼
       [S3Client]               (singleton wrapper)
            │ injected into
            ▼
       [S3*Repository adapters] (singleton, implement business-layer ports)
            │ injected into
            ▼
       [Services]
```

### Constraints

- One `aioboto3.Session` per process; never instantiate per request or per event-loop spin-up (except in tests that own their loop).
- Never instantiated by FastAPI routes; routes depend on repository ports, not on `S3Client`.
- Never accepts raw bucket names from untrusted input without going through bucket validation / default-bucket resolution.
- Stateless beyond the held client, region, and default-bucket name — no per-request mutable state on the wrapper.

---

- `S3Client` is a single class. It holds the opened low-level `aioboto3` client, the active region, and (after `set_bucket`) an optional default-bucket name.
- The wrapper does **not** own the `aioboto3.Session` or the client async lifecycle. The DI container opens the client (an async context manager) once via `providers.Resource` and hands the already-opened object to the wrapper.

See also: `[aioboto3` README — client usage]([https://github.com/terricain/aioboto3#example](https://github.com/terricain/aioboto3#example)), `[./dependency-injector.md](./dependency-injector.md)`.

## Credentials And Region

- `aioboto3` inherits the boto3 credential chain unchanged: explicit constructor args → environment variables → `AssumeRole` / SSO sessions → `~/.aws/credentials` → container or instance metadata. First match wins.
- **Production:** prefer IAM roles (instance profile, ECS / EKS task role, IRSA on Kubernetes) over long-term keys. The chain picks up temporary credentials automatically.
- **Region:** always pass `region_name` explicitly when opening the client. Default region resolution can produce surprising values in CI.
- **S3-compatible stores** (MinIO, R2, LocalStack, Wasabi): pass `endpoint_url=...` at client open time. Some private deployments require `s3={"addressing_style": "path"}` in the `AioConfig`; production AWS buckets always use `virtual`.

See also: [boto3 credentials guide](https://docs.aws.amazon.com/boto3/latest/guide/credentials.html), `[AioConfig](https://github.com/aio-libs/aiobotocore/blob/master/aiobotocore/config.py)`, [AWS S3 — virtual hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html).

## Bucket Validation And The Default Bucket

`create_bucket`, `set_bucket`

- `set_bucket` calls `s3.head_bucket` (200 = exists and visible, 404 = absent, 403 = exists-but-denied). On 404 with `create_if_missing=True`, it falls through to `create_bucket`; otherwise raises. On success, the bucket name is stored so later methods can omit the bucket argument.
- `create_bucket` is region-aware: `us-east-1` takes no `CreateBucketConfiguration`; every other region requires `LocationConstraint`. The wrapper hides the branching.
- A private bucket-resolution helper picks the explicit argument if given, else the default, else raises `ValueError`.

See also: [boto3 `head_bucket](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/head_bucket.html)`, [boto3 `create_bucket](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/create_bucket.html)`, [AWS S3 — bucket restrictions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/BucketRestrictions.html).

## Uploading And Downloading Files

`upload_file`, `download_file`, `download_bytes`

- `upload_file` accepts any file-like object (`BytesIO`, open file, `aiofiles`) and delegates to `s3.upload_fileobj`. Auto-switches to multipart upload above ~8 MiB.
- `download_file` streams to disk via `s3.download_file`; memory bounded by the transfer config. Returns the local path written to.
- `download_bytes` calls `s3.get_object` and reads the body in one shot — only for small artifacts. For anything large, use `download_file` or stream `iter_chunks(chunk_size=...)` directly on the body.
- **Warning:** never call `.read()` on a streaming body in a hot path — it loads the whole object into memory.
- **Warning:** if you drop down to the low-level multipart API, wrap the sequence in `try`/`except` and call `abort_multipart_upload` on failure. Orphan parts accrue cost forever; back it up with a bucket lifecycle rule that expires incomplete uploads after 7 days.
- For per-call metadata (`Content-Type`, `Cache-Control`, custom `x-amz-meta-`*), extend the wrapper signature as concrete needs appear rather than re-exposing the full `ExtraArgs` dict.

See also: [boto3 `upload_fileobj](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/upload_fileobj.html)`, [boto3 `download_file](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/download_file.html)`, [boto3 `get_object](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/get_object.html)`,` [StreamingBody.iter_chunks](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/response.html#botocore.response.StreamingBody.iter_chunks)`, [multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html), [lifecycle: abort incomplete multipart](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html).

## Presigned URLs

`generate_presigned_get_url`, `generate_presigned_put_url`

- Both call `s3.generate_presigned_url("get_object" | "put_object", ...)` under the hood. The signature embeds bucket, key, HTTP method, expiry, and an opaque credential reference; S3 verifies all of it on the request.
- `generate_presigned_get_url`: the wrapper guesses `Content-Type` from the key extension via `mimetypes.guess_type` and pins `ResponseContentDisposition: inline` so browsers render rather than force-download.
- `generate_presigned_put_url`: for browser-side direct-to-S3 uploads. Caller does an HTTP `PUT` with the file as the request body. The optional content-type argument pins what S3 will require on the upload.
- For richer upload constraints (max size, content-type whitelist, key prefix), use `s3.generate_presigned_post` directly — POST is more flexible because of its `Conditions` policy block.
- **Warning:** `expires_in` accepts up to 7 days, but the real maximum is bounded by the credential lifetime of whoever signed the URL. URLs signed under an STS session (assumed role, SSO, IRSA, instance profile, task role) stop working when that session expires — usually within hours. Sign with long-term keys reserved for the purpose, or refresh on demand.

See also: [boto3 presigned URLs guide](https://docs.aws.amazon.com/boto3/latest/guide/s3-presigned-urls.html), [boto3 `generate_presigned_url](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/generate_presigned_url.html)`, [boto3 `generate_presigned_post](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/generate_presigned_post.html)`, [AWS S3 — using a presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html).

## Existence Checks And Deletion

`object_exists`, `delete_object`, `delete_all_objects`

- `object_exists` calls `s3.head_object` and coerces 404 / `ClientError` code `"404"` to `False`; success returns `True`. Other error codes (`403`, etc.) bubble up — they indicate misconfiguration, not absence. Callers that need the response metadata should drop down to `head_object` directly.
- `delete_object` is idempotent in S3 (deleting a missing key returns 204), but the wrapper does an `object_exists` check anyway and logs a warning on miss — cheap way to surface caller bugs.
- `delete_all_objects` paginates `list_objects_v2` and batches deletes via `s3.delete_objects` (1000-key cap per call). Dramatically faster than per-key deletion for large buckets.
- The aioboto3 *resource* API offers `await bucket.objects.all().delete()` as a one-liner alternative, but it requires opening a resource client alongside the client this wrapper holds. The wrapper stays on the client API.
- **Warning:** restrict `delete_all_objects` to admin paths, test teardown, and lifecycle-managed staging buckets. There is no undo.

See also: [boto3 `head_object](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/head_object.html)`, [boto3 `delete_object](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/delete_object.html)`, [boto3 `delete_objects](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/delete_objects.html)`, [boto3` list_objects_v2` paginator]([https://docs.aws.amazon.com/boto3/latest/reference/services/s3/paginator/ListObjectsV2.html](https://docs.aws.amazon.com/boto3/latest/reference/services/s3/paginator/ListObjectsV2.html)).

## Wiring Through The Container

- The wrapper expects an already-opened `aioboto3` client. The DI container holds the session, opens the client, and constructs the wrapper.
- Architectural shape: the container owns the `aioboto3.Session` and the low-level S3 client's async-context lifecycle; the wrapper itself is stateless plumbing on top.

Container wiring (aioboto3 Session `Singleton` + per-request S3 client `Resource`) lives in [./dependency-injector.md#aws-s3](./dependency-injector.md#aws-s3).

See also: `[./dependency-injector.md](./dependency-injector.md)`, [Dependency Injector — Resource provider](https://python-dependency-injector.ets-labs.org/providers/resource.html), `[aioboto3` README]([https://github.com/terricain/aioboto3#example](https://github.com/terricain/aioboto3#example)).

## Testing

`S3Client` is a low-level adapter. The pieces that callers actually depend on are the **repository adapters** built on top of it (e.g. `S3UserRepository`, `S3DocumentRepository`, with a shared `S3BaseRepository`) — each implementing a business-layer port (`UserRepository`, `DocumentRepository`) declared as a `Protocol`. API tests mock at the **port**, not at `S3Client`.

- Define ports in the business layer as `Protocol` types (`UserRepository`, `DocumentRepository`).
- Implement S3-backed adapters in infrastructure (`S3UserRepository`, etc.) that use `S3Client` internally.
- In API tests, override the *repository provider* in the DI container with an in-memory fake — `S3Client` and `aioboto3` never run. This matches the pattern in `[./python-tests.md](./python-tests.md)` and the symmetric SQL example in `[./sqlalchemy.md](./sqlalchemy.md)`.
- Optional: a small contract suite can exercise a single adapter against moto or LocalStack to catch regressions in the S3 wire shape — but keep that suite separate from API tests, and do not stand up moto inside an API-test fixture.
- For manual local runs, point `aioboto3` at MinIO or LocalStack via `endpoint_url=...` in the container's config — no test-code change required.

See also: `[./python-tests.md](./python-tests.md)`, `[./sqlalchemy.md](./sqlalchemy.md)`, `[./dependency-injector.md](./dependency-injector.md)`, [LocalStack — S3](https://docs.localstack.cloud/user-guide/aws/s3/).

## Pitfalls

- **Sharing a session across event loops.** `aioboto3.Session` is bound to the loop that created it. Tests that spin up a new event loop per test must create a fresh session per test (or per fixture scope).
- **Long-lived clients.** The underlying client is an async context manager; treat it as a `Resource` owned by the container, not as a process global.
- **Forgetting `signature_version="s3v4"` and virtual addressing.** Required for presigned URLs in non-`us-east-1` regions. Set both in the `AioConfig` at client creation.
- `**.read()` on `response["Body"]`.** Loads the entire object into memory. Use `download_file` with a local path, or stream `iter_chunks`.
- **Presigned URL outliving its credential.** A URL signed by a service running on a task role dies when the role rotates. Match TTL to credential lifetime, or sign with credentials whose lifetime you control.
- `**delete_all_objects` against production.** No undo. Restrict to admin and test-only paths.
