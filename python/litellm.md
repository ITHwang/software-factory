# LiteLLM: Provider-Neutral LLM Calls

> Last updated: 2026-05-14

## TL;DR

How to use LiteLLM from an async FastAPI backend for direct, provider-neutral prompt-to-response LLM calls: client wrapping, retries, streaming, structured output, error classification, usage extraction.

**Use this when:**
- one-shot LLM tasks: summarization, classification, translation, extraction, rewriting
- you need provider-neutral routing across OpenAI/Anthropic/Bedrock/Azure/etc.
- you need fine-grained retry, error classification, or token-usage accounting on direct calls

**Don't use this for:**
- tool-using agent loops with multi-step reasoning → `./langchain.md`
- multi-graph topologies, `Send` fan-out, reducers → `./langgraph.md`
- tracing/observability → `./langfuse.md` (attaches as a callback)

How to use LiteLLM from an async FastAPI backend. `LiteLLMClient` is the
implementation of the LLM client layer for direct, prompt-to-response tasks:
summarization, classification, translation, extraction, rewriting. For agent
loops (tools, multi-step reasoning, streamed progress), see
[`./langchain.md`](./langchain.md).

The backend shape is route → service/use-case → `LiteLLMClient`. The client
owns provider-neutral LiteLLM calls, retries, streaming, structured output,
error classification, and usage extraction. Tools live on the agent layer and
are not managed here.

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Quick Reference](#quick-reference), [When To Use](#when-to-use) |
| 2. Setup | [Install](#install), [Wire Up](#wire-up) |
| 3. Build | [Core Pattern](#core-pattern), [Structured Output](#structured-output), [Inspecting Provider Requests](#inspecting-provider-requests), [Streaming](#streaming), [Retries And Errors](#retries-and-errors), [Token Usage](#token-usage), [LangChain Agent Integration](#langchain-agent-integration) |
| 4. Integrate | [FastAPI Example](#fastapi-example) |
| 5. Operate | [Testing](#testing), [Production Checklist](#production-checklist), [Pitfalls](#pitfalls) |

## Quick Reference

| Need | Default |
|------|---------|
| Python package | `litellm` |
| Async completion | `await litellm.acompletion(...)` |
| Sync completion | `litellm.completion(...)` for CLI/scripts only |
| Provider selection | Provider-prefixed model string such as `openai/gpt-4o-mini` |
| Messages | OpenAI chat completion format: `{"role": "...", "content": "..."}` |
| Streaming | `await litellm.acompletion(..., stream=True)` then `async for chunk in stream` |
| Structured output | Pass `response_format=<PydanticModel>`; verify the per-provider request/response once with verbose logging |
| Usage | Read `response.usage` or chunk usage when included by the provider |
| LangChain agent integration | Use `langchain_litellm.ChatLiteLLM` directly; see [`./langchain.md`](./langchain.md) |
| API test boundary | Mock `LiteLLMClient`, not LiteLLM itself |

See also: [LiteLLM docs home](https://docs.litellm.ai/), [LiteLLM repo](https://github.com/BerriAI/litellm).

## When To Use

Use LiteLLM directly when the service needs one LLM call and the application
controls the workflow.

| Scenario | Fit |
|----------|-----|
| Summarize text | Direct call |
| Classify or route a ticket | Direct call with structured output |
| Translate or rewrite content | Direct call |
| Extract fields from a document | Direct call with `response_format` |
| Run search/fetch tools selected by the model | Use [`./langchain.md`](./langchain.md) |
| Multi-step agent loop with progress events | Use [`./langchain.md`](./langchain.md) |

LiteLLM is the provider-neutral layer. It translates OpenAI-compatible chat
completion parameters across providers and normalizes responses and exceptions.
Use it because it keeps provider churn out of services: OpenAI, Azure OpenAI,
Anthropic, Gemini/Vertex, Bedrock, OpenRouter, local OpenAI-compatible servers,
and other providers can sit behind the same client surface.

### Where this fits

| Layer | Responsibility | Where it lives |
|-------|----------------|----------------|
| LLM client | Multi-provider I/O — retries, error classification, structured-output translation, streaming, token usage | `LiteLLMClient` (this doc) |
| Agent runner | Tools, middleware, graph runtime, multi-step orchestration | `langchain_litellm.ChatLiteLLM` + LangChain (see [`./langchain.md`](./langchain.md)) |

`LiteLLMClient` is the LLM client implementation we use today. Future
provider-specific clients (e.g., transports outside LiteLLM's coverage) would
sit at the same layer with their own `BaseChatModel` adapter for LangChain.

## Install

```bash
uv add litellm pydantic tenacity dependency-injector
```

For tests:

```bash
uv add --dev pytest pytest-asyncio httpx
```

Provider API keys are read from environment variables such as
`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `AZURE_API_KEY`, or provider-specific
settings documented by LiteLLM. Do not pass secrets through route payloads.

## Wire Up

The client is a singleton dependency. It is cheap to construct, but centralizing
it keeps model configuration, provider options, retries, usage tracking, and
error mapping consistent.

```text
HTTP request
  -> FastAPI route/controller
  -> service/use-case
  -> LiteLLMClient
  -> litellm.acompletion()
```

```python
# litellm_client.py
from typing import Any

from pydantic import BaseModel, Field


class LiteLLMSettings(BaseModel):
    model: str = "openai/gpt-4o-mini"
    temperature: float | None = 0
    max_tokens: int | None = 800
    timeout_seconds: float = 30
    api_base: str | None = None
    custom_llm_provider: str | None = None
    provider_kwargs: dict[str, Any] = Field(default_factory=dict)
```

Container wiring (settings `Factory` + app-scoped client `Singleton` + request-owned service `Factory`) lives in [./dependency-injector.md#litellm](./dependency-injector.md#litellm).

## Core Pattern

Keep the LiteLLM call, provider option merging, error mapping, and response
extraction in one client.

**Purpose:** unified async client over the LiteLLM SDK that hides provider differences from services and emits typed completions.

**Responsibilities:** own provider configuration (model, `api_base`, headers); call LiteLLM's completion API; map LiteLLM errors into a small set of typed exceptions; expose async text completions and structured (Pydantic-validated) completions.

**Must not:** own retry policy (decorated externally via `@retry`); own token accounting (callers extract from response); own per-call provider switching (one client per configured provider).

```python
# litellm_client.py
from collections.abc import AsyncIterator, Sequence
from typing import Any, TypeVar

import litellm
from pydantic import BaseModel

StructuredT = TypeVar("StructuredT", bound=BaseModel)

Message = dict[str, str]


class LiteLLMClientError(Exception):
    def __init__(self, message: str, *, transient: bool) -> None:
        super().__init__(message)
        self.transient = transient


class LiteLLMClient:
    def __init__(self, settings: LiteLLMSettings) -> None: ...

    async def complete_text(self, messages: Sequence[Message]) -> str: ...

    async def complete_structured(
        self,
        messages: Sequence[Message],
        response_model: type[StructuredT],
    ) -> StructuredT: ...

    async def acomplete(self, messages: Sequence[Message], **kwargs: Any) -> Any: ...
```

`acomplete` merges configured defaults with per-call `kwargs`, drops `None`
entries, and delegates to `litellm.acompletion`; exceptions pass through
`classify_litellm_error` (see [Retries And Errors](#retries-and-errors)).
`complete_text` returns `response.choices[0].message.content`;
`complete_structured` passes `response_format=response_model` and validates the
returned JSON with `response_model.model_validate_json`.

Never call `litellm.completion()` from an async FastAPI request handler. It is a
blocking call and will stall the event loop. Reserve sync completion for CLI
scripts, migrations, and offline jobs.

See also: [LiteLLM completion input](https://docs.litellm.ai/docs/completion/input), [LiteLLM request/response patterns](https://docs.litellm.ai/docs/guides/core_request_response_patterns).

## Structured Output

Use structured output when the service needs fields, flags, scores, or other
machine-consumed data.

```python
from pydantic import BaseModel, Field


class ModerationDecision(BaseModel):
    allowed: bool
    category: str = Field(description="policy category or 'none'")
    explanation: str


decision = await litellm_client.complete_structured(
    [
        {"role": "system", "content": "Return only valid JSON for the schema."},
        {"role": "user", "content": "Can this comment be published? ..."},
    ],
    ModerationDecision,
)
```

Before standardizing on structured output for a model, verify support with
LiteLLM's model support helpers such as `get_supported_openai_params(...)` and
`supports_response_schema(...)`. Keep schema fields small and explicit. If a
provider returns malformed content, let `model_validate_json` fail and map that
failure to a service-level error or retry.

LiteLLM accepts OpenAI-compatible params and translates `response_format` into
each provider's native shape — OpenAI structured outputs, Anthropic tool-use,
Gemini `response_schema`, and so on. Pass OpenAI-compatible config and trust
this translation as the default.

> When adopting a new provider/model, verify the outgoing request body and the
> raw provider response once with verbose logging (see
> [Inspecting Provider Requests](#inspecting-provider-requests)). The
> translation is mostly transparent but can drift on nested schemas, optional
> fields, and tool calling.

See also: [LiteLLM JSON mode](https://docs.litellm.ai/docs/completion/json_mode), [LangChain structured output](https://docs.langchain.com/oss/python/langchain/structured-output).

## Inspecting Provider Requests

LiteLLM presents an OpenAI-compatible surface and translates each call into the
provider's native request shape. The translation is mostly transparent for plain
chat completions, but can drift for `response_format`, `tools`/`tool_choice`,
`stream_options`, and provider-specific JSON-mode flags. Verify the actual wire
traffic once per new provider/model combination, then trust the client.

Enable verbose logging in-process for tests or local validation:

```python
import litellm

litellm.set_verbose = True  # or: litellm._turn_on_debug()
```

Or via environment variable for ops, with no code change:

```bash
export LITELLM_LOG=DEBUG
```

What to confirm in the logs:

- Outgoing request body matches the provider's documented schema.
- `response_format`, `tools`, and `tool_choice` translate as expected for that provider.
- Raw provider response shape, before LiteLLM normalizes it into the OpenAI envelope.
- `usage` fields are populated when you depend on them for billing or limits.

For deeper provider-specific patterns, defer to LiteLLM's own guides rather
than re-documenting them here:

- Core request/response patterns: https://docs.litellm.ai/docs/guides/core_request_response_patterns
- Tools integrations: https://docs.litellm.ai/docs/guides/tools_integrations

Do not leave verbose logging enabled in production. Use Langfuse or another
observability layer through callbacks for steady-state visibility (see
[Production Checklist](#production-checklist)).

## Streaming

Expose streaming from the client as an async iterator. The route can translate
that iterator into Server-Sent Events or newline-delimited JSON.

```python
class LiteLLMClient:
    ...

    async def stream_text(self, messages: Sequence[Message]) -> AsyncIterator[str]:
        try:
            stream = await self.acomplete(
                messages=messages,
                stream=True,
                stream_options={"include_usage": True},
            )
            async for chunk in stream:
                delta = chunk.choices[0].delta.content or ""
                if delta:
                    yield delta
        except Exception as exc:
            raise classify_litellm_error(exc) from exc
```

Treat streamed usage as opportunistic. Some providers include usage in the final
chunk with `stream_options`, and some do not. If exact usage is required for
billing, use the provider/model combination that reliably emits usage or record
usage through a callback/observability layer.

## Retries And Errors

Classify errors before they reach the service layer. The service should know
whether the failure is retryable or user-facing; it should not know provider
exception details.

```python
import litellm


TRANSIENT_ERRORS = (
    litellm.APIConnectionError,
    litellm.RateLimitError,
    litellm.InternalServerError,
    litellm.ServiceUnavailableError,
    litellm.APIError,
)

PERMANENT_ERRORS = (
    litellm.AuthenticationError,
    litellm.BadRequestError,
    litellm.ContextWindowExceededError,
)


def classify_litellm_error(exc: Exception) -> LiteLLMClientError:
    if isinstance(exc, TRANSIENT_ERRORS):
        return LiteLLMClientError(str(exc), transient=True)
    if isinstance(exc, PERMANENT_ERRORS):
        return LiteLLMClientError(str(exc), transient=False)
    return LiteLLMClientError("LLM request failed", transient=False)
```

Retry only transient errors, and cap total latency. Tenacity is a simple
service-local option.

```python
from tenacity import retry, retry_if_exception, stop_after_attempt, wait_exponential


def is_transient_client_error(exc: BaseException) -> bool:
    return isinstance(exc, LiteLLMClientError) and exc.transient


class LiteLLMClient:
    ...

    @retry(
        reraise=True,
        retry=retry_if_exception(is_transient_client_error),
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=0.5, min=0.5, max=4),
    )
    async def complete_text(self, messages: Sequence[Message]) -> str:
        response = await self.acomplete(messages=messages)
        return response.choices[0].message.content or ""
```

Do not retry authentication, invalid-request, context-window, or schema
validation failures without changing the request.

## Token Usage

Normalize usage at the client boundary.

```python
from pydantic import BaseModel


class LLMUsage(BaseModel):
    model: str
    input_tokens: int
    output_tokens: int
    total_tokens: int


def extract_usage(response) -> LLMUsage | None:
    usage = getattr(response, "usage", None)
    if usage is None:
        return None
    input_tokens = getattr(usage, "prompt_tokens", 0) or 0
    output_tokens = getattr(usage, "completion_tokens", 0) or 0
    return LLMUsage(
        model=getattr(response, "model", "unknown"),
        input_tokens=input_tokens,
        output_tokens=output_tokens,
        total_tokens=getattr(usage, "total_tokens", input_tokens + output_tokens),
    )
```

Return usage to the service only when it has a clear consumer: billing,
rate-limit accounting, observability, or debugging. Avoid leaking provider
response objects above the client boundary.

## LangChain Agent Integration

When a workflow needs tools or a multi-step loop, use
`langchain_litellm.ChatLiteLLM` — the `BaseChatModel` adapter that ships in
the `langchain-litellm` package. It calls LiteLLM directly, without going
through `LiteLLMClient`; the two are parallel runtimes by design. See
[`./langchain.md`](./langchain.md) for the full agent setup.

`LiteLLMClient` stays in service/use-case code for direct, prompt-to-response
calls. Do not wrap `LiteLLMClient` in a custom `BaseChatModel` when
`ChatLiteLLM` already ships — custom adapters are reserved for future LLM
client implementations that don't have an off-the-shelf LangChain
integration.

Do not build direct prompt-to-answer flows with LangChain just to call the LLM.
That adds an agent framework where a provider-neutral client is enough.

See also: [LiteLLM tool integrations](https://docs.litellm.ai/docs/guides/tools_integrations), [`langchain-litellm` repo](https://github.com/langchain-ai/langchain-litellm).

## FastAPI Example

Route -> service -> client.

```python
# schemas.py
from pydantic import BaseModel


class SummaryRequestDTO(BaseModel):
    text: str


class SummaryResponseDTO(BaseModel):
    summary: str
```

```python
# services.py
from .litellm_client import LiteLLMClient
from .schemas import SummaryResponseDTO


class SummaryService:
    def __init__(self, litellm_client: LiteLLMClient) -> None:
        self._litellm_client = litellm_client

    async def summarize(self, text: str) -> SummaryResponseDTO:
        summary = await self._litellm_client.complete_text([
            {"role": "system", "content": "Summarize in three concise bullets."},
            {"role": "user", "content": text},
        ])
        return SummaryResponseDTO(summary=summary)
```

DI wiring for these routes (`@inject` + `Depends(Provide[Container.summary_service])`) lives in [./dependency-injector.md#litellm](./dependency-injector.md#litellm). Route bodies focus on tool capability below.

```python
# routes.py
from fastapi import APIRouter

from .schemas import SummaryRequestDTO, SummaryResponseDTO
from .services import SummaryService

router = APIRouter()


@router.post("/summaries", response_model=SummaryResponseDTO)
async def create_summary(
    payload: SummaryRequestDTO,
    service: SummaryService,  # injected via container
) -> SummaryResponseDTO:
    return await service.summarize(payload.text)
```

Streaming route:

```python
from fastapi.responses import StreamingResponse

from .litellm_client import LiteLLMClient


@router.post("/summaries/stream")
async def stream_summary(
    payload: SummaryRequestDTO,
    litellm_client: LiteLLMClient,  # injected via container
) -> StreamingResponse:
    messages = [
        {"role": "system", "content": "Summarize in three concise bullets."},
        {"role": "user", "content": payload.text},
    ]

    async def events():
        async for token in litellm_client.stream_text(messages):
            yield f"data: {token}\n\n"

    return StreamingResponse(events(), media_type="text/event-stream")
```

## Testing

API tests follow [`./python-tests.md`](./python-tests.md): override the client
dependency and call the real route and service.

```python
from dependency_injector import providers
from httpx import ASGITransport, AsyncClient

from app.containers import Container
from app.main import create_app


class MockLiteLLMClient:
    async def complete_text(self, messages):
        assert messages[-1]["content"] == "Long text"
        return "Short summary"


async def test_create_summary() -> None:
    container = Container()
    container.litellm_client.override(providers.Object(MockLiteLLMClient()))
    app = create_app(container=container)
    transport = ASGITransport(app=app)

    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.post("/summaries", json={"text": "Long text"})

    assert response.status_code == 200
    assert response.json() == {"summary": "Short summary"}

    container.litellm_client.reset_override()
```

Client unit tests can monkeypatch `litellm.acompletion` with an async mock and
assert retries, error classification, usage extraction, and response parsing.
API tests should never call real LiteLLM providers.

## Production Checklist

- Use `litellm.acompletion()` in FastAPI request paths.
- Centralize model, timeout, temperature, max token, and provider options in `LiteLLMClient`.
- Classify provider errors into transient and permanent categories.
- Retry only transient errors with a small attempt and latency budget.
- Validate structured output with Pydantic before returning it.
- Track model name, token usage, request ID, user/session ID, and latency.
- Bound prompt size before calling the model; handle context-window errors explicitly.
- Keep provider credentials in environment or secret stores.
- Mock `LiteLLMClient` in API tests.
- Use Langfuse or another observability layer through injected clients or callbacks, not directly in routes.

## Pitfalls

| Pitfall | Avoid it |
|---------|----------|
| Calling `litellm.completion()` in FastAPI | Use `await litellm.acompletion()` |
| Letting provider exceptions leak | Map them to client/service errors |
| Retrying permanent failures | Retry only transient classes |
| Returning raw LiteLLM responses from services | Convert to DTOs or domain models |
| Parsing prose for structured data | Use `response_format` and Pydantic validation |
| Assuming all providers support the same params | Check provider/model support |
| Depending on streamed usage everywhere | Treat stream usage as provider-dependent |
| Real LLM calls in API tests | Override the client boundary |
| Adopting a new provider without checking the translated request | Enable LiteLLM verbose logging once to confirm `response_format` and tool params translate correctly, then disable |
