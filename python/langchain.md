# LangChain

> Last updated: 2026-05-11

## TL;DR

How to use LangChain agents in an async FastAPI backend: `create_agent`, tool primitives (`@tool`, `BaseTool`, `StructuredTool`), middleware (budgets, retries, fallback, HITL), structured output via `response_format`, streaming.

**Use this when:**
- the workflow needs a tool-using agent loop with bounded multi-step reasoning
- you need built-in middleware for limits, retries, summarization, fallback, or human-in-the-loop
- you need structured final output (`response_format=PydanticModel`) or progress streaming

**Don't use this for:**
- single prompt-to-response calls with no tools → `./litellm.md`
- multi-graph topology, `Send` fan-out, reducers, raw `StateGraph` → `./langgraph.md`
- tracing/observability wiring → `./langfuse.md`

How to use LangChain agents in an async FastAPI backend. Use this guide for
tool-using workflows, bounded multi-step reasoning, structured agent responses,
and streamed progress. For simple prompt-to-response calls with no tools, prefer
the direct LiteLLM client in [`./litellm.md`](./litellm.md).

This guide is async-first. The route calls a service/use-case, the service calls
an agent gateway, and `dependency-injector` assembles the agent from passive
configuration, a `langchain_litellm.ChatLiteLLM` model, tool gateways,
middleware, and tracing callbacks.

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Concepts | [Quick Reference](#quick-reference), [When To Use](#when-to-use) |
| 2. Setup | [Install](#install), [Wire Up](#wire-up) |
| 3. Build | [Core Pattern](#core-pattern), [Tools](#tools), [Middleware](#middleware), [Structured Output](#structured-output), [Streaming](#streaming) |
| 4. Integrate | [FastAPI Example](#fastapi-example) |
| 5. Operate | [Testing](#testing), [Production Checklist](#production-checklist), [Pitfalls](#pitfalls) |

## Quick Reference

| Need | Default |
|------|---------|
| Agent constructor | `langchain.agents.create_agent(...)` |
| Wiring | `dependency-injector` composes settings, clients, tools, middleware, factory, gateway |
| Model | `langchain_litellm.ChatLiteLLM`; direct LLM calls stay in [`./litellm.md`](./litellm.md) |
| Runtime | LangGraph compiled graph behind the agent |
| Async call | `await agent.ainvoke({"messages": [...]}, config=...)` |
| Async stream | `async for chunk in agent.astream(..., stream_mode="messages", version="v2")` |
| Tools | Small async functions decorated with `@tool` |
| Structured response | Pass a Pydantic model as `response_format` and read `state["structured_response"]` |
| Middleware | Hooks into the LangGraph runtime: `before_agent`, `before_model`, `wrap_model_call`, `after_model`, `wrap_tool_call`, `after_agent`. LangChain ships built-ins for budgets, summarization, retries, fallback, HITL, PII |
| API test boundary | Mock the agent gateway, not LangChain internals |

## When To Use

Use a LangChain agent when the workflow needs at least one of these:

| Need | Why an agent helps |
|------|--------------------|
| Tool calls | The model can choose search, retrieval, API, or persistence tools and use their outputs in follow-up steps |
| Multiple iterations | The agent loops until it emits a final response or hits a configured stop condition |
| Progress streaming | LangGraph streams intermediate state, messages, custom events, and token chunks |
| Structured final output | `response_format` validates the final answer before it reaches the service layer |
| Runtime controls | Middleware can limit iterations, cap tool calls, inject state, or filter tools per user |

Use direct LLM calls instead when the request is summarization, classification,
translation, extraction from already-loaded text, or any single prompt-to-answer
flow. Direct calls are cheaper to reason about, faster to test, and easier to
retry. See [`./litellm.md`](./litellm.md).

### Which client where

The agent path and the direct path are independent runtimes. Pick by runtime, not
by feature need.

| Runtime | Client | Owns |
|---------|--------|------|
| LangChain / LangGraph agent | `langchain_litellm.ChatLiteLLM` | Provider params, agent-loop concerns via middleware, tracing via callbacks |
| Direct one-shot call | `LiteLLMClient` (see [`./litellm.md`](./litellm.md)) | Retry, rate-limit, error translation, tracing via LiteLLM SDK callbacks |
| Custom agent without LangChain | `LiteLLMClient` | Same as direct — call the client from your own loop |

Observability is unified through Langfuse, attached to each path separately via
its callback hook. See [`./langfuse.md`](./langfuse.md).

See also: [Langfuse LangChain integration](https://langfuse.com/integrations/frameworks/langchain).

**Per-client `BaseChatModel` rule.** Each LLM client implementation needs a
corresponding `BaseChatModel` for LangChain to plug into. For `LiteLLMClient`,
that is `langchain_litellm.ChatLiteLLM` (shipped — no custom code). For a
future LLM client implementation without an off-the-shelf LangChain
integration, write a custom `BaseChatModel(client=...)` adapter at that time.
Don't wrap a client when a `BaseChatModel` already ships.

## Install

```bash
uv add langchain langchain-litellm langgraph litellm dependency-injector pydantic
```

Optional integrations:

```bash
uv add langfuse          # callback tracing, see ./langfuse.md
```

Use `langchain_litellm.ChatLiteLLM` for the LangChain agent path and
`LiteLLMClient` for direct calls. The two are parallel runtimes, not a fallback
chain.

## Wire Up

Keep LangChain out of route handlers. Build the agent once through the
container, inject the gateway into services, and call the gateway from use-cases.

```text
HTTP request
  -> FastAPI route/controller
  -> service/use-case
  -> AgentGateway
  -> LangChain create_agent / LangGraph runtime
  -> tools / outbound gateways
```

The container shape follows [`./dependency-injector.md`](./dependency-injector.md).
The important ownership rule: settings are data, `AgentFactory` is the actor
that constructs the agent, and the container decides what gets injected.

```python
# containers.py
from dependency_injector import containers, providers

from langchain_litellm import ChatLiteLLM

from .agents import AgentFactory, AgentGateway, AgentSettings, build_middlewares
from .litellm_client import LiteLLMClient, LiteLLMSettings
from .services import AskService
from .search import SearchGateway


class Container(containers.DeclarativeContainer):
    config = providers.Configuration(yaml_files=["configs/app.yaml"])

    llm_settings = providers.Factory(
        LiteLLMSettings,
        model=config.llm.model,
        temperature=config.llm.temperature.as_float(),
        max_tokens=config.llm.max_tokens.as_int(),
        timeout_seconds=config.llm.timeout_seconds.as_int(),
        provider_kwargs=config.llm.provider_kwargs,
    )
    litellm_client = providers.Singleton(LiteLLMClient, settings=llm_settings)
    agent_model = providers.Singleton(
        ChatLiteLLM,
        model=config.llm.model,
        temperature=config.llm.temperature.as_float(),
        max_tokens=config.llm.max_tokens.as_int(),
        request_timeout=config.llm.timeout_seconds.as_int(),
    )

    search_gateway = providers.Singleton(
        SearchGateway,
        base_url=config.search.base_url,
    )
    agent_settings = providers.Factory(
        AgentSettings,
        system_prompt=config.agent.system_prompt,
        max_iterations=config.agent.max_iterations.as_int(),
        max_tool_calls=config.agent.max_tool_calls.as_int(),
    )
    agent_middlewares = providers.Callable(
        build_middlewares,
        settings=agent_settings,
    )
    agent_factory = providers.Factory(
        AgentFactory,
        model=agent_model,
        search_gateway=search_gateway,
        settings=agent_settings,
        middlewares=agent_middlewares,
    )
    agent = providers.Singleton(lambda factory: factory.build(), agent_factory)
    agent_gateway = providers.Factory(AgentGateway, agent=agent)

    ask_service = providers.Factory(AskService, agent_gateway=agent_gateway)
```

## Core Pattern

The factory owns agent construction. The gateway owns invocation. Configuration
does not construct anything; it is injected into the factory.

```python
# agents.py
from collections.abc import Sequence
from typing import Protocol

from langchain.agents import create_agent
from langchain.agents.middleware import (
    AgentMiddleware,
    ModelCallLimitMiddleware,
    ToolCallLimitMiddleware,
)
from langchain_core.language_models.chat_models import BaseChatModel
from langchain_core.tools import tool
from pydantic import BaseModel, Field


class SearchGateway(Protocol):
    async def search(self, query: str) -> list[str]: ...


class AgentAnswer(BaseModel):
    answer: str = Field(description="User-facing answer")
    citations: list[str] = Field(default_factory=list)


class AgentSettings(BaseModel):
    system_prompt: str
    max_iterations: int = 6
    max_tool_calls: int = 4


def build_middlewares(settings: AgentSettings) -> list[AgentMiddleware]:
    return [
        ModelCallLimitMiddleware(run_limit=settings.max_iterations),
        ToolCallLimitMiddleware(run_limit=settings.max_tool_calls),
    ]


def build_search_tool(search_gateway: SearchGateway):
    @tool
    async def search_docs(query: str) -> str:
        """Search approved source documents for relevant snippets."""
        hits = await search_gateway.search(query)
        return "\n".join(hits)

    return search_docs


class AgentFactory:
    def __init__(
        self,
        *,
        model: BaseChatModel,
        search_gateway: SearchGateway,
        settings: AgentSettings,
        middlewares: Sequence[AgentMiddleware],
    ) -> None:
        self._model = model
        self._search_gateway = search_gateway
        self._settings = settings
        self._middlewares = list(middlewares)

    def build(self):
        return create_agent(
            name="ask_agent",
            model=self._model,
            tools=[build_search_tool(self._search_gateway)],
            system_prompt=self._settings.system_prompt,
            response_format=AgentAnswer,
            middleware=self._middlewares,
        )


class AgentGateway:
    def __init__(self, agent) -> None:
        self._agent = agent

    async def answer(
        self,
        *,
        request_id: str,
        user_id: str,
        question: str,
    ) -> AgentAnswer:
        state = await self._agent.ainvoke(
            {"messages": [{"role": "user", "content": question}]},
            config={
                "configurable": {"thread_id": request_id},
                "run_id": request_id,
                "metadata": {"user_id": user_id},
            },
        )
        return state["structured_response"]
```

The agent path uses `langchain_litellm.ChatLiteLLM` directly — no custom adapter.
The direct path uses `LiteLLMClient` and is documented in
[`./litellm.md`](./litellm.md). The two are independent runtimes: behaviors that
belong to one (LangChain middleware, LiteLLM retry policy) do not need to be
shared with the other. Langfuse hooks attach to each separately — see
[`./langfuse.md`](./langfuse.md).

Do not compile the agent inside the request handler. Construction binds the
model, tools, middleware, and response schema. Reusing the compiled runtime
keeps request latency predictable and makes tests patch one gateway boundary.

See also: [LangChain `create_agent` docs](https://docs.langchain.com/oss/python/langchain/agents).

## Tools

Tools should be narrow adapters over existing gateways, repositories, or
services. They are not a second business layer.

| Rule | Why |
|------|-----|
| Prefer async tools | A blocking tool stalls the event loop and delays every streamed update |
| Keep tool descriptions concrete | The model chooses tools from name, schema, and docstring |
| Return compact strings or JSON | Tool output is fed back into model context |
| Enforce authorization before tool execution | Tools can expose data that the route did not fetch directly |
| Limit tool count | Too many tools dilute selection quality and increase prompt cost |

For user-specific tool availability, use a `wrap_model_call` middleware to
filter tools per request — see [Middleware](#middleware) for the canonical
pattern. Don't construct a new agent for every request.

### Library surface

`langchain_core.tools` provides the primitives. Reach for the decorator first; subclass only when you need state or non-trivial init.

| Primitive | Use |
|-----------|-----|
| `@tool` | Decorate an async function. Name, args schema, and description are derived from the signature + docstring. The default and almost-always-right choice. |
| `BaseTool` | Subclass when the tool holds state (a shared HTTP client, a pre-loaded model) or needs custom `_arun`. Define `name`, `description`, `args_schema`, and `async def _arun(...)`. |
| `StructuredTool` | Build a tool from a callable + an explicit `args_schema` without subclassing. Useful when the schema doesn't match the function signature one-for-one. |
| `ToolException` | Raise inside a tool to signal a clean, model-visible failure. The runtime catches it and feeds the message back to the model instead of bubbling the traceback. |
| `InjectedToolArg` / `InjectedState` | Annotate parameters the framework injects from runtime context rather than from LLM-generated args. |

`create_agent(tools=[...])` accepts any of these uniformly. The agent's model sees only the JSON-schema view; the runtime handles dispatch.

See also: [LangChain tools concept](https://python.langchain.com/docs/concepts/tools/), [LangGraph tool-calling how-to](https://langchain-ai.github.io/langgraph/how-tos/tool-calling/), [LangGraph many-tools how-to](https://langchain-ai.github.io/langgraph/how-tos/many-tools/), [LangGraph agentic concepts](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/), [LangGraph types reference](https://langchain-ai.github.io/langgraph/reference/types/).

### Example

A typical `@tool`-decorated coroutine wraps an existing async gateway:

```python
from langchain_core.tools import tool

from .gateways import WebSearchGateway


def build_web_search_tool(gateway: WebSearchGateway):
    @tool
    async def web_search(query: str, limit: int = 5) -> str:
        """Search the web for recent pages matching the query.

        Args:
            query: The user's search phrase.
            limit: Maximum number of results to return.
        """
        results = await gateway.search(query=query, limit=limit)
        return "\n".join(f"- {r.title} ({r.url})" for r in results)

    return web_search
```

The closure-over-gateway pattern keeps the tool a narrow adapter (rule 1) and threads request-bound dependencies in via the factory. Parallel tool calls are dispatched internally by `create_agent` — see [./langgraph.md#reducers](./langgraph.md#reducers) for the underlying `Send` primitive.

See also: [LangChain custom tools how-to](https://python.langchain.com/docs/how_to/custom_tools/), [HTTPX async client](https://www.python-httpx.org/async_client/) (for tools that hit external APIs).

### Async Lifecycle

Tools used inside an async graph must be async. A sync tool blocks the event loop and stalls every concurrent in-flight tool. Three rules:

- **Bound every external call with a hard timeout.** Use `asyncio.timeout(seconds)` (Python 3.11+) inside the tool body, not at the caller. Cooperative cancellation only works if `_arun` awaits regularly — a tight CPU loop ignores `task.cancel()`.
- **Push blocking work to a worker thread.** If the tool must call a sync SDK (a sync vector store, a CPU-heavy chunker), wrap that call in `await asyncio.to_thread(blocking_fn, ...)`. Don't introduce a sync tool.
- **Let `CancelledError` propagate.** Don't `except CancelledError: pass` — it strands the cleanup that LangGraph wires up on client disconnect.

```python
import asyncio

from langchain_core.tools import tool


@tool
async def fetch_page(url: str) -> str:
    """Fetch a single URL and return its text."""
    async with asyncio.timeout(10):
        return await http_client.get_text(url)
```

See also: [`asyncio.timeout`](https://docs.python.org/3/library/asyncio-task.html#asyncio.timeout), [`asyncio.Task`](https://docs.python.org/3/library/asyncio-task.html).

### Tool I/O

LangChain validates tool args against a Pydantic schema derived from the function signature + type hints. For richer schemas — field descriptions the model can read, validators, custom types — pass an explicit `args_schema`:

```python
from langchain_core.tools import tool
from pydantic import BaseModel, Field


class SearchInput(BaseModel):
    query: str = Field(description="What to search for.")
    limit: int = Field(default=5, ge=1, le=20, description="Max results to return.")


@tool(args_schema=SearchInput)
async def web_search(query: str, limit: int = 5) -> str:
    ...
```

Outputs go straight into the model's context as whatever the function returns (str, dict, list — `to_string` is best-effort). Return compact, parseable shapes; the rules table above applies.

See also: [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/).

## Middleware

Middleware is the user-facing extension surface over the LangGraph runtime that
sits behind every `create_agent` call. Recent LangChain versions consolidated
the agent runtime onto LangGraph and exposed middleware as the way to hook
into it — you don't drop into raw graph code for things like history trimming,
retries, budgets, or human-in-the-loop. Pass `middleware=[...]` to
`create_agent` and the runtime composes them per the diagram below.

See also: [LangChain middleware overview](https://docs.langchain.com/oss/python/langchain/middleware/overview).

### Hook surface

| Hook | Fires | Purpose | Async variant |
|------|-------|---------|---------------|
| `before_agent` | Once per invocation | Load memory/resources, validate input | `abefore_agent` |
| `before_model` | Per loop iteration | Trim history, mutate state before the model call | `abefore_model` |
| `wrap_model_call` | Per loop iteration | Wrap the model call (caching, retries, dynamic model/tools) | full async handler |
| `after_model` | Per loop iteration | Inspect model response, gate next step | `aafter_model` |
| `wrap_tool_call` | Per tool execution | Wrap the tool invocation (auth, retries, response shaping) | full async handler |
| `after_agent` | Once per invocation | Persist results, emit events, cleanup | `aafter_agent` |

### Execution flow

```
invocation
   │
   ▼
before_agent          M1 → M2 → M3       (once per invocation, in list order)
   │
   ▼
┌─ agent loop ─────────────────────────────────────────────────┐
│                                                              │
│  before_model       M1 → M2 → M3                             │
│        │                                                     │
│        ▼                                                     │
│  wrap_model_call    M1( M2( M3( <model call> ) ) )           │
│        │            (nested; first middleware is outermost)  │
│        ▼                                                     │
│  after_model        M3 → M2 → M1       (reverse)             │
│        │                                                     │
│        ▼                                                     │
│  tool calls in response?                                     │
│        │                                                     │
│        ├── yes ─▶ wrap_tool_call  M1( M2( M3( <tool> ) ) )   │
│        │                │                                    │
│        │                └──── back to before_model           │
│        │                                                     │
│        └── no  ─▶ exit loop                                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
   │
   ▼
after_agent          M3 → M2 → M1       (reverse)
   │
   ▼
response
```

### Built-in middleware

LangChain ships these in `langchain.agents.middleware`. Prefer them before
subclassing `AgentMiddleware`.

| Class | Group | Purpose |
|-------|-------|---------|
| `ModelCallLimitMiddleware` | Budget | Cap total model calls per invocation (`run_limit`) or thread (`thread_limit`) |
| `ToolCallLimitMiddleware` | Budget | Cap tool calls overall or per `tool_name` |
| `SummarizationMiddleware` | History | Auto-summarize history at token limits |
| `ContextEditingMiddleware` | History | Trim older tool outputs from context |
| `ModelFallbackMiddleware` | Resilience | Switch to a backup model on primary failure |
| `ModelRetryMiddleware` | Resilience | Retry failed model calls with backoff |
| `ToolRetryMiddleware` | Resilience | Retry failed tool calls with backoff |
| `HumanInTheLoopMiddleware` | Safety | Pause for human approval before tool execution |
| `PIIMiddleware` | Safety | Detect and handle PII in model traffic |
| `LLMToolSelectorMiddleware` | Tool selection | Use an LLM to pick relevant tools per request |
| `LLMToolEmulator` | Testing | Emulate tool execution via LLM (no real tool runs) |
| `TodoListMiddleware` | Planning | Task planning and tracking inside the loop |
| `ShellToolMiddleware` | Tooling | Persistent shell sessions for shell tools |
| `FilesystemFileSearchMiddleware` | Tooling | Glob/grep search exposed to the agent |

See also: [LangChain built-in middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in).

### Patterns

**Decorator (single hook).** For a one-shot hook over the model call — for
example, filtering tools by user role per request:

```python
from collections.abc import Callable

from langchain.agents.middleware import ModelRequest, ModelResponse, wrap_model_call


@wrap_model_call
def filter_private_tools(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    context = request.runtime.context or {}
    user_role = (
        context.get("role", "viewer")
        if isinstance(context, dict)
        else getattr(context, "role", "viewer")
    )
    if user_role != "admin":
        request = request.override(
            tools=[tool for tool in request.tools if not tool.name.startswith("admin_")]
        )
    return handler(request)
```

**Class-based (multi-hook, stateful).** Subclass `AgentMiddleware` when you
need more than one hook or per-instance state:

```python
from langchain.agents.middleware import AgentMiddleware


class ProgressEmitter(AgentMiddleware):
    """Emit a progress event after each model turn."""

    def __init__(self, sink) -> None:
        self._sink = sink

    async def aafter_model(self, state, runtime) -> dict | None:
        await self._sink.send({
            "event": "model_turn",
            "iteration": state.get("iteration"),
        })
        return None
```

Both pass into `create_agent(middleware=[...])`; the runtime composes them per
the diagram above.

See also: [LangChain custom middleware](https://docs.langchain.com/oss/python/langchain/middleware/custom), [`AgentMiddleware` reference](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentMiddleware).

### Composition order

| Hook type | Order |
|-----------|-------|
| `before_*` | List order — first registered fires first |
| `wrap_*` | Nested — first registered is outermost; innermost is the actual call |
| `after_*` | Reverse list order — last registered fires first |

Mirrors HTTP middleware semantics. If you've used Express or Starlette, the
mental model is identical.

### Sharp edges

- **Streaming context leakage.** Model calls inside middleware hooks inherit
  the parent agent's streaming config, so tokens emitted from those internal
  calls leak to the outer stream. Isolate with a fresh `RunnableConfig` if you
  need a private model call inside a hook.
- **State schema extensions.** Middleware can extend agent state via
  `state_schema`, but merging multiple extensions is order-sensitive and
  underdocumented — keep extensions narrow.
- **Latency.** Each layer adds orchestration overhead. Don't stack middleware
  speculatively; budget for what you actually need.

## Structured Output

Inside `AgentFactory`, pass a Pydantic model as `response_format` for agent
responses that feed API DTOs, database writes, or downstream workflows.

```python
from pydantic import BaseModel, Field


class TriageDecision(BaseModel):
    category: str = Field(description="One of: bug, billing, account, other")
    confidence: float = Field(ge=0, le=1)
    rationale: str


agent = create_agent(
    model=model,
    tools=[search_docs],
    response_format=TriageDecision,
)

state = await agent.ainvoke({
    "messages": [{"role": "user", "content": "I cannot log in after payment"}]
})
decision: TriageDecision = state["structured_response"]
```

Prefer structured output over parsing prose. Still validate business invariants
in the service layer; schema validation proves shape, not correctness.

### Coverage and strategy

`response_format` is not a universal provider feature. LangChain picks one of two
strategies on `with_structured_output`:

| Strategy | Selected when | Mechanism |
|----------|---------------|-----------|
| `ProviderStrategy` | Model exposes native structured output (e.g. OpenAI, Anthropic, xAI Grok directly) | Provider-side `response_format` |
| `ToolStrategy` | Everything else | Forces a tool call whose schema is the response schema and validates tool args |

Through `ChatLiteLLM`, LangChain typically can't see the underlying provider's
native capability and falls back to `ToolStrategy`. That works for any model
LiteLLM exposes that supports tool calls — broad coverage in practice, but the
output is Pydantic-validated client-side, not provider-guaranteed. Verify a
specific model with `litellm.supports_response_schema(model)` if native support
matters for your use case.

Known sharp edges: combining `tool_choice="none"` with fallbacks can drop JSON
output, and a few `azure_ai`/Mistral SDK versions reject `json_schema` until
LiteLLM's adapter normalizes it.

See also: [LangChain structured-output docs](https://docs.langchain.com/oss/python/langchain/structured-output).

## Streaming

Use `astream()` when the frontend needs progress or tokens. Keep the streaming
translation in the gateway so the route only returns an async iterator.

```python
from collections.abc import AsyncIterator


class AgentGateway:
    ...

    async def stream_answer(
        self,
        *,
        request_id: str,
        question: str,
    ) -> AsyncIterator[str]:
        async for chunk in self._agent.astream(
            {"messages": [{"role": "user", "content": question}]},
            config={"configurable": {"thread_id": request_id}, "run_id": request_id},
            stream_mode="messages",
            version="v2",
        ):
            if chunk["type"] != "messages":
                continue
            message_chunk, metadata = chunk["data"]
            if message_chunk.content:
                yield message_chunk.content
```

`version="v2"` selects LangGraph's unified `StreamPart` chunk shape
(`{type, ns, data}`), so consumer code stays the same regardless of
`stream_mode` or `subgraphs`. The v1 default returns raw chunks or tuples
depending on the call shape, forcing the consumer to branch on configuration.
Requires `langgraph >= 1.1`.

Use `stream_mode="messages"` for token chunks, `stream_mode="updates"` for node
state updates, and `stream_mode="custom"` for explicit progress events.

See also: [LangGraph streaming docs](https://docs.langchain.com/oss/python/langgraph/streaming), and [./langgraph.md#streaming](./langgraph.md#streaming) for the producer-side `stream_writer` primitive.

## FastAPI Example

Route -> service -> gateway. The route owns HTTP shape, the service owns
orchestration, and the gateway owns LangChain.

```python
# schemas.py
from pydantic import BaseModel


class AskRequestDTO(BaseModel):
    question: str


class AskResponseDTO(BaseModel):
    answer: str
    citations: list[str]
```

```python
# services.py
from collections.abc import AsyncIterator

from .agents import AgentGateway
from .schemas import AskResponseDTO


class AskService:
    def __init__(self, agent_gateway: AgentGateway) -> None:
        self._agent_gateway = agent_gateway

    async def ask(
        self,
        *,
        request_id: str,
        user_id: str,
        question: str,
    ) -> AskResponseDTO:
        answer = await self._agent_gateway.answer(
            request_id=request_id,
            user_id=user_id,
            question=question,
        )
        return AskResponseDTO(
            answer=answer.answer,
            citations=answer.citations,
        )

    async def stream(
        self,
        *,
        request_id: str,
        question: str,
    ) -> AsyncIterator[str]:
        async for token in self._agent_gateway.stream_answer(
            request_id=request_id,
            question=question,
        ):
            yield token
```

```python
# routes.py
from typing import Annotated
from uuid import uuid4

from dependency_injector.wiring import Provide, inject
from fastapi import APIRouter, Depends, Request

from .containers import Container
from .schemas import AskRequestDTO, AskResponseDTO
from .services import AskService

router = APIRouter()


@router.post("/ask", response_model=AskResponseDTO)
@inject
async def ask(
    payload: AskRequestDTO,
    request: Request,
    service: Annotated[AskService, Depends(Provide[Container.ask_service])],
) -> AskResponseDTO:
    user_id = request.headers.get("x-user-id", "anonymous")
    return await service.ask(
        request_id=str(uuid4()),
        user_id=user_id,
        question=payload.question,
    )
```

Streaming route:

```python
from fastapi.responses import StreamingResponse


@router.post("/ask/stream")
@inject
async def ask_stream(
    payload: AskRequestDTO,
    service: Annotated[AskService, Depends(Provide[Container.ask_service])],
) -> StreamingResponse:
    async def events():
        async for token in service.stream(
            request_id=str(uuid4()),
            question=payload.question,
        ):
            yield f"data: {token}\n\n"

    return StreamingResponse(events(), media_type="text/event-stream")
```

## Testing

API tests follow [`./python-tests.md`](./python-tests.md): replace the gateway
boundary, then call the real route and service/use-case.

```python
from dependency_injector import providers
from httpx import ASGITransport, AsyncClient

from app.agents import AgentAnswer
from app.containers import Container
from app.main import create_app


class FakeAgentGateway:
    async def answer(self, *, request_id: str, user_id: str, question: str):
        return AgentAnswer(answer=f"Answered: {question}", citations=["doc-1"])


async def test_ask_returns_agent_answer() -> None:
    container = Container()
    container.agent_gateway.override(providers.Object(FakeAgentGateway()))
    app = create_app(container=container)
    transport = ASGITransport(app=app)

    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.post("/ask", json={"question": "What changed?"})

    assert response.status_code == 200
    assert response.json() == {
        "answer": "Answered: What changed?",
        "citations": ["doc-1"],
    }

    container.agent_gateway.reset_override()
```

Service tests can fake `AgentGateway`. Gateway tests should use a fake model or
LangChain test double, not a real provider. Keep real LLM calls in a small,
manually-triggered integration suite with explicit API-key requirements.

## Production Checklist

- Choose direct LiteLLM calls for one-shot tasks before introducing an agent.
- Compile agents at startup for stable tool and middleware sets.
- Make every external tool async and timeout-bounded.
- Set max iterations and tool-call budgets for every agent.
- Prefer built-in middleware from `langchain.agents.middleware` before subclassing `AgentMiddleware`.
- Use structured output for machine-consumed answers.
- Pass `thread_id`, `run_id`, user/session metadata, and tracing callbacks in `config`.
- Stream high-level progress or tokens; never stream hidden reasoning.
- Keep prompt, model, temperature, and tool budgets in passive configuration.
- Use DI factories, not config objects, to build agents.
- Record token usage and provider/model names when the model response exposes them.
- Mock the agent gateway in API tests and disable real tracing callbacks in tests.

## Pitfalls

| Pitfall | Avoid it |
|---------|----------|
| Using an agent for one LLM call | Use [`./litellm.md`](./litellm.md) instead |
| Wrapping `LiteLLMClient` in a custom `BaseChatModel` | `ChatLiteLLM` already ships in `langchain-litellm`. Custom adapters are only for clients without an off-the-shelf LangChain integration |
| Blocking tools | Use async clients or move blocking work to workers |
| Config object builds the agent | Inject passive config into an `AgentFactory` |
| Creating agents in handlers | Build once at startup unless tools are truly dynamic |
| No iteration limit | Add `ModelCallLimitMiddleware(run_limit=...)` so the agent must finalize within budget |
| Reimplementing built-in middleware concerns | Check `langchain.agents.middleware` first (e.g., `SummarizationMiddleware`, `ModelCallLimitMiddleware`, `HumanInTheLoopMiddleware`) before subclassing |
| Tool output too large | Return compact snippets, IDs, and summaries |
| Parsing final prose | Use `response_format` and validate service-level invariants |
| Assuming `response_format` works on every model | Coverage depends on tool-call support (or native structured output). Check `litellm.supports_response_schema(model)` and confirm the model can be tool-called |
| Testing through real providers | Fake the gateway in API tests |
| Leaking project wrappers into examples | Keep public guides on FastAPI, LangChain, and gateway interfaces |
| Blocking calls inside an async tool | Wrap in `await asyncio.to_thread(...)` or use an async SDK — sync calls stall every concurrent tool |
| Tool output too large | Cap result counts and content length at the tool boundary, not in a downstream node — the output feeds back into model context |
| No per-tool timeout | Use `async with asyncio.timeout(seconds):` inside the tool — a hanging external call ties up an agent iteration indefinitely |
| LLM-emitted tool args bypassing schema validation | Pydantic validation runs on the dispatched args — but only if `args_schema` (or an inferable function signature) is set. A loose `**kwargs` tool signature skips validation entirely |
| Swallowing `CancelledError` in a tool | Let it propagate so LangGraph can cancel the in-flight node on client disconnect or upstream timeout |

