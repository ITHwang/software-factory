# Langfuse

> Last updated: 2026-05-12

## TL;DR

Langfuse covers two concerns in this stack: **prompt management** (versioned prompts edited outside the codebase) and **tracing** (LLM/agent observability). Treat them as independent.

- **Prompt management is an adapter-internal concern.** Domain ports are capability-oriented (`TextSummarizer.summarize(text) -> str`); the adapter (`OpenAITextSummarizer`, etc.) loads its prompt template from Langfuse or a local YAML file and constructs its own LLM client. See [./python-backend-architecture.md](./python-backend-architecture.md) for the port/adapter pattern.
- **Tracing uses the Langfuse SDK directly** — no gateway. Three modes:
  - `CallbackHandler` for LangChain / LangGraph runs (the easy default for agents).
  - `@observe` decorator for function-scoped spans (the default high-level pattern).
  - Manual spans (`start_as_current_observation`, `propagate_attributes`, `create_trace_id`) for deterministic trace IDs and custom propagation.
- **Skip the SDK** when you need to: another language, an edge runtime, a lightweight script, or an endpoint not yet wrapped — call the public REST API at `/api/public/`* directly.

**Use this when:**

- prompts must be versioned and edited outside the codebase
- traces of LangChain/LangGraph runs are needed, correlated by user/session/request
- spans around outbound LLM calls need lifecycle management

**Don't use this for:**

- direct one-shot LLM calls without tracing → `[./litellm.md](./litellm.md)`
- agent construction itself → `[./langchain.md](./langchain.md)` (Langfuse attaches as a callback)
- generic application logs → loguru

Targets **Langfuse Python SDK v4** ([overview](https://langfuse.com/docs/observability/sdk/python/overview)).

## Table of Contents


| Phase        | Section                                                                                                                                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Concepts  | [Quick Reference](#quick-reference), [When To Use](#when-to-use), [SDK Usage Modes](#sdk-usage-modes)                                                                                                                      |
| 2. Setup     | [Install](#install)                                                                                                                                                                                                        |
| 3. Build     | [Prompt Management](#prompt-management), [Tracing With LangChain](#tracing-with-langchain), [Tracing With @observe](#tracing-with-observe), [Tracing With Manual Spans](#tracing-with-manual-spans), [REST API](#rest-api) |
| 4. Integrate | [FastAPI Example](#fastapi-example)                                                                                                                                                                                        |
| 5. Operate   | [Production Checklist](#production-checklist), [Pitfalls](#pitfalls)                                                                                                                                                       |


## Quick Reference


| Need                   | Default                                                                    | Docs                                                                                                         |
| ---------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Python package         | `langfuse` (v4)                                                            | [SDK overview](https://langfuse.com/docs/observability/sdk/python/overview)                                  |
| Client                 | `from langfuse import get_client`                                          | [Python API reference](https://python.reference.langfuse.com)                                                |
| Prompt fetch (inside adapter) | `langfuse.get_prompt(name, type="chat", label=..., cache_ttl_seconds=...)` + `prompt.compile(**variables)` | See [Prompt Management](#prompt-management) |
| Cache TTL              | SDK default 60s; override per fetch                                        | See [Prompt Management](#prompt-management)                                                                  |
| LangChain trace        | `from langfuse.langchain import CallbackHandler`                           | [LangChain integration](https://langfuse.com/docs/integrations/langchain/tracing)                            |
| Function trace         | `from langfuse import observe` → `@observe()`                              | [Observe wrapper](https://langfuse.com/docs/observability/sdk/instrumentation#observe-wrapper)               |
| Manual span            | `langfuse.start_as_current_observation(as_type="span", ...)`               | [Manual observations](https://langfuse.com/docs/observability/sdk/instrumentation#manual-observations)       |
| Deterministic trace ID | `langfuse.create_trace_id(seed=request_id)`                                | [Custom instrumentation](https://langfuse.com/docs/observability/sdk/instrumentation#custom-instrumentation) |
| Shutdown               | `get_client().shutdown()`                                                  | [SDK overview](https://langfuse.com/docs/observability/sdk/python/overview)                                  |
| REST                   | `POST /api/public/`*                                                       | [REST API reference](https://api.reference.langfuse.com)                                                     |


## When To Use

Reach for Langfuse when prompt iteration or observability needs to be independent of application code deploys. Skip it in prototypes where prompts are stable and tracing isn't yet needed; add it later behind a capability-oriented port (`TextSummarizer`, `Translator`, ...) whose adapter reads its prompt from Langfuse — and direct SDK calls for tracing — when prompt changes or trace inspection become part of the operating model.

## SDK Usage Modes

The Python SDK exposes three tracing entry points. The right choice depends on what you control.


| Mode                 | When                                                                 | Entry point                                      | Docs                                                                                                   |
| -------------------- | -------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| `CallbackHandler`    | LangChain / LangGraph agent or graph runs                            | `from langfuse.langchain import CallbackHandler` | [LangChain integration](https://langfuse.com/docs/integrations/langchain/tracing)                      |
| `@observe` decorator | Function-scoped traces with minimal ceremony                         | `from langfuse import observe`                   | [Observe wrapper](https://langfuse.com/docs/observability/sdk/instrumentation#observe-wrapper)         |
| Manual spans         | Deterministic trace IDs or custom propagation                        | `langfuse.start_as_current_observation(...)`     | [Manual observations](https://langfuse.com/docs/observability/sdk/instrumentation#manual-observations) |
| REST API             | No SDK dependency (other languages, edge, scripts, custom transport) | `POST /api/public/*`                             | [REST API reference](https://api.reference.langfuse.com)                                               |


The first three compose freely — a `CallbackHandler` attached to an agent invocation inherits the active span opened by `@observe` or `start_as_current_observation`. Pick the highest-level mode that still expresses what you need.

## Install

```bash
uv add langfuse
```

Optional when tracing LangChain or LangGraph:

```bash
uv add langchain langgraph
```

Environment variables:

```bash
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_BASE_URL=https://cloud.langfuse.com
```

Use the correct regional or self-hosted `LANGFUSE_BASE_URL`. With no credentials present, the client initializes in no-op mode — useful for tests and dev environments that should never talk to the real Langfuse backend.

## Prompt Management

Don't abstract prompt fetching itself. Abstract the business capability the prompt powers. A `TextSummarizer` port hides the prompt entirely — the business layer calls `summarize(text)` and gets a result. The adapter (`OpenAITextSummarizer`) decides where its prompt template lives (Langfuse for hot-iterable copy, local YAML for source-controlled fallback) and what params and provider to use.

```python
# app/summaries/ports/text_summarizer.py
from typing import Protocol


class TextSummarizer(Protocol):
    async def summarize(self, text: str) -> str: ...
```

**Purpose**: one sentence — concrete `TextSummarizer` adapter that owns its own OpenAI client, reads its prompt template (Langfuse or local YAML), and returns a summarized string.

**Responsibilities**:
- Construct and hold the OpenAI client.
- Resolve the prompt template at startup (Langfuse-first, YAML fallback).
- Apply model/max_tokens/temperature config.
- Cache the resolved prompt for `cache_ttl_seconds`.

**Must not**:
- Be injected with an LLM client externally — the adapter owns this.
- Expose Langfuse-specific types across the `TextSummarizer` interface.

```python
# app/summaries/adapters/openai_text_summarizer.py
class OpenAITextSummarizer:
    def __init__(
        self,
        *,
        prompt_name: str,
        prompt_label: str,
        model: str,
        max_tokens: int,
        temperature: float,
        cache_ttl_seconds: int,
    ) -> None: ...

    async def summarize(self, text: str) -> str: ...
```

A `LocalYamlTextSummarizer` that reads its prompt template from a YAML file in source control is structurally the same — just swap `langfuse.get_prompt` for a file read at construction time. Use it as a deployment fallback for critical capabilities, or as the test fake (no separate `FakeTextSummarizer` needed if YAML covers the test surface).

The same port shape supports many variants: provider swap (`OpenAITextSummarizer` ↔ `ClaudeTextSummarizer`), model or prompt experimentation (`V1Summarizer` ↔ `V2Summarizer`), deterministic tests (`FakeTextSummarizer`), and failure/cost fallback (`PrimaryTextSummarizer` + `FallbackTextSummarizer` composed in a service). All of this happens behind the port — business code never picks the adapter.

The port carries domain arguments (`text`), not prompt variables (`{"text": ...}`). Business code never sees the prompt schema. If you find yourself passing a `variables: dict[str, object]` around, you've reinvented a generic prompt gateway.

See also: [./python-backend-architecture.md](./python-backend-architecture.md), [./litellm.md](./litellm.md), [prompt management — get started](https://langfuse.com/docs/prompt-management/get-started), [version control](https://langfuse.com/docs/prompt-management/features/prompt-version-control), [caching](https://langfuse.com/docs/prompt-management/features/caching), [observability — get started](https://langfuse.com/docs/observability/get-started).

## Tracing With LangChain

The lowest-ceremony way to trace agent or graph runs. Pass a per-invocation `CallbackHandler()` through the LangChain/LangGraph invocation `config`; model calls, tool calls, and graph nodes nest automatically under the active span.

```python
from langfuse.langchain import CallbackHandler

state = await agent.ainvoke(
    {"messages": [{"role": "user", "content": question}]},
    config={
        "callbacks": [CallbackHandler()],
        "configurable": {"thread_id": request_id},
        "run_id": request_id,
    },
)
```

Create one handler per request/invocation rather than reusing a singleton — concurrent requests race on per-handler state (e.g., `last_trace_id`).

See also: [LangChain integration](https://langfuse.com/docs/integrations/langchain/tracing).

## Tracing With @observe

Decorate a function; a span appears automatically with the function name and input/output captured. The default high-level pattern when you own the function.

```python
from langfuse import observe, propagate_attributes


@observe()
async def summarize(text: str, *, user_id: str, request_id: str) -> str:
    with propagate_attributes(user_id=user_id, session_id=request_id, tags=["summary"]):
        return await litellm_client.complete_text(messages)
```

`propagate_attributes` flows user/session/tag attributes down to every child observation created within its context — including LangChain `CallbackHandler` spans started inside the decorated function.

See also: `[@observe` wrapper](https://langfuse.com/docs/observability/sdk/instrumentation#observe-wrapper), [Python SDK API reference](https://python.reference.langfuse.com).

## Tracing With Manual Spans

Use manual spans when the decorator can't express what you need — most commonly, you want a deterministic trace ID seeded from a request ID so the trace can be looked up from a log line, or you want to manage span lifecycle yourself.

```python
from langfuse import get_client, propagate_attributes

langfuse = get_client()
trace_id = langfuse.create_trace_id(seed=request_id)

with langfuse.start_as_current_observation(
    as_type="span",
    name="summaries.create",
    input={"text_length": len(text)},
    trace_context={"trace_id": trace_id},
) as span:
    with propagate_attributes(user_id=user_id, session_id=request_id, tags=["summary"]):
        summary = await litellm_client.complete_text(messages)
    span.update(output={"summary": summary})
```

`create_trace_id` is an instance method on the client in SDK v4 — call it on the `langfuse` instance returned by `get_client()`, not as a class-level helper.

See also: [manual observations](https://langfuse.com/docs/observability/sdk/instrumentation#manual-observations), [custom instrumentation](https://langfuse.com/docs/observability/sdk/instrumentation#custom-instrumentation).

## REST API

When the Python SDK isn't the right fit — another language, an edge runtime, a lightweight script, a custom transport, or an endpoint not yet wrapped — Langfuse's public API at `/api/public/*` accepts ingestion and read calls directly.

```bash
curl -X POST https://cloud.langfuse.com/api/public/ingestion \
  -u "$LANGFUSE_PUBLIC_KEY:$LANGFUSE_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"batch": [...]}'
```

See also: [REST API reference](https://api.reference.langfuse.com), [OpenAPI spec](https://cloud.langfuse.com/generated/api/openapi.yml), [public API overview](https://langfuse.com/docs/api-and-data-platform/features/public-api).

## FastAPI Example

Wire capability-oriented ports through `dependency-injector` so route handlers depend on services, and services depend on Protocols — never on Langfuse, prompt names, or variables. Tracing is service-side and uses the Langfuse client directly — no gateway.

Container wiring for the `TextSummarizer` port lives in [./dependency-injector.md#langfuse](./dependency-injector.md#langfuse). Tests override at the port level: `container.text_summarizer.override(providers.Object(FakeTextSummarizer()))`.

Routes depend on services; services depend on the `TextSummarizer` Protocol. No Langfuse, no prompt name, no variables in business code. See [./python-tests.md](./python-tests.md) for port-level overrides in API tests.

## Production Checklist

- Fetch prompts by deployment label, not by unqualified latest version.
- Use a positive `cache_ttl_seconds` in production; `0` only in local development.
- Prefetch critical prompts on startup when cold-start failure is unacceptable.
- Keep a source-controlled YAML-backed adapter (e.g., `LocalYamlTextSummarizer`) wired up for capabilities whose downtime is unacceptable.
- Validate prompt variables before calling the LLM.
- Open a root span per request, seeded with `langfuse.create_trace_id(seed=request_id)`, so traces are reachable from a log line.
- Attach user/session/tags at the root span via `propagate_attributes`; let nested spans inherit.
- Pass one `CallbackHandler()` per LangChain/LangGraph invocation — never a shared singleton.
- Call `get_client().shutdown()` on FastAPI shutdown and on script/worker process exit; use `flush()` only when a short-lived process must block for delivery.
- Override capability ports in tests (`container.text_summarizer.override(...)`). Langfuse client no-ops without credentials, so leaving the Langfuse-backed adapter wired in API tests still works — but a `FakeTextSummarizer` is faster and deterministic.

## Pitfalls


| Pitfall                                                      | Avoid it                                                                                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| Generic `PromptGateway` returning `list[dict[str, str]]`     | Port the domain capability (`TextSummarizer.summarize(text)`), not prompt-fetch primitives. Business code shouldn't see the message-list shape. |
| Injecting an LLM client into the adapter                     | Adapter constructs its own LLM client. If the LLM client itself needs to be swappable, that's a separate (lower-level) port — but in practice model/provider swap happens at the adapter level. |
| Building a `TraceGateway` wrapper                            | Use the Langfuse client directly — tracing isn't substitutable and isn't a domain-stability concern, so a gateway adds no value |
| Unqualified prompt fetches                                   | Fetch by `label` or pinned `version`                                                                                            |
| Assuming prompt updates are instant                          | Account for SDK cache TTL and background revalidation                                                                           |
| Missing prompt variables at compile time                     | Validate before the LLM call                                                                                                    |
| Reusing one `CallbackHandler` across requests                | Create one per invocation — handler state (e.g., `last_trace_id`) is not concurrency-safe                                       |
| Updating trace metadata in nested spans                      | Let only the request root own trace-level metadata                                                                              |
| Real Langfuse credentials in CI                              | Use `FakeTextSummarizer` (or a YAML-backed adapter) in API tests; the Langfuse client no-ops without credentials, but a fake is faster and deterministic. |
| Calling `create_trace_id` as a static class method (v3 form) | v4 made it an instance method — call `langfuse.create_trace_id(seed=...)` on the client returned by `get_client()`              |
| Forgetting `shutdown()` in workers/scripts                   | Call `get_client().shutdown()` at process end                                                                                   |

## See Also

- [./python-backend-architecture.md](./python-backend-architecture.md) — ports/adapters layer model and naming.
- [./litellm.md](./litellm.md) — direct LLM client used inside summarizer-style adapters.
- [./langchain.md](./langchain.md) — agent construction; Langfuse attaches via `CallbackHandler`.
