# LangGraph

> Last updated: 2026-05-11

## TL;DR

How to build multi-step agent graphs on top of LangGraph: `create_agent` as the default building block, raw `StateGraph` for exotic topology, reducers for parallel-branch aggregation, streaming, per-request context, testing.

**Use this when:**
- building a tool-loop agent or supervisor-subagent architecture
- you need `Send`-based parallel fan-out and reducer aggregation
- you need to stream execution events (token chunks, custom events, state updates) from a running graph

**Don't use this for:**
- one-shot prompt-to-response LLM calls → `./litellm.md`
- per-tool primitives (`@tool`, `BaseTool`, `StructuredTool`, args validation) → `./langchain.md#tools`
- middleware hooks, `response_format`, agent gateways → `./langchain.md`

How to build LangGraph graphs as implementations of agent-gateway ports. The runtime here is `langgraph.graph.StateGraph` and its `compile()` output (`CompiledStateGraph`); a graph is one implementation of whichever business-layer port it serves (`CompactionAgent`, `ResearchAgent`, …). For most graphs in this codebase, `langchain.agents.create_agent` is the building block — it returns a `CompiledStateGraph` directly, with the agent loop, tool dispatch, and parallel tool calls already wired up.

This guide is async-first. The production stack is `FastAPI → async LangGraph → async tools and LLM (await tool.ainvoke / await llm.ainvoke) → Send fan-out → reducer aggregation → astream streaming`. Async must propagate end-to-end; mixing sync calls into an async stack stalls the event loop. Mental model:


| Concern         | Primitive                             |
| --------------- | ------------------------------------- |
| Async execution | `await graph.ainvoke(...)`            |
| Parallelism     | `Send("worker", ...)`                 |
| Aggregation     | reducer (`Annotated[T, fn]`)          |
| Streaming       | `async for ... in graph.astream(...)` |


This guide layers ABOVE [./langchain.md](./langchain.md): the single-agent `create_agent` / `@tool` / `astream` primitives belong there. For tools, see [./langchain.md#tools](./langchain.md#tools); for streaming and `Runtime` / `stream_writer`, see [Streaming](#streaming); for tests, see [Testing](#testing).


## Table of Contents

| Phase         | Section                                                                                                                                                 |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Concepts   | [Quick Reference](#quick-reference), [When To Use](#when-to-use)                                                                                        |
| 2. Setup      | [Install](#install), [Wire Up](#wire-up)                                                                                                                |
| 3. Build      | [Building A Graph](#building-a-graph), [Architecture Examples](#architecture-examples), [Reducers](#reducers), [Streaming](#streaming)                  |
| 4. Cross-cuts | [Per-request context](#per-request-context), [Configuration](#configuration), [Adding A Graph](#adding-a-graph), [Mixins](#mixins), [Testing](#testing) |
| 5. Operate    | [Production Checklist](#production-checklist), [Pitfalls](#pitfalls)                                                                                    |


## Quick Reference

> If your graph would be one LLM call, **don't build a graph** — call the LLM client directly. See [./litellm.md](./litellm.md). LangGraph's value is multi-step orchestration; a one-prompt graph just rebrands an LLM call.

Two agent architectures cover most multi-step work, both built on `langchain.agents.create_agent`:


| Architecture        | When to use                                                                    | Composition                                                                                           |
| ------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| One-agent           | One agent, all context shared, tool loop with optional parallel tool calls     | One `create_agent` + tools + middleware                                                               |
| Supervisor-subagent | Multi-agent, isolated contexts per subagent, supervisor delegates and collects | Outer `create_agent` (supervisor) + each subagent exposed as a tool that runs an inner `create_agent` |


Anything more exotic is just nodes + edges on a raw `StateGraph` — drop down to that only when `create_agent`'s loop genuinely doesn't fit.

See also: [LangGraph docs](https://langchain-ai.github.io/langgraph/).

## When To Use


| Need                                                                                        | Pattern                                                                                          |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| One LLM call, no tool loop                                                                  | LLM client directly — see [./litellm.md](./litellm.md)                                           |
| One agent loop with tools                                                                   | One-agent architecture (`create_agent`) — see [./langchain.md](./langchain.md) for the primitive |
| Multi-agent with isolated contexts                                                          | Supervisor-subagent architecture (see below)                                                     |
| Custom topology that doesn't fit `create_agent` (exotic routing, fanout outside tool calls) | Raw `StateGraph`                                                                                 |


## Install

```bash
uv add langgraph langchain langchain-core pydantic dependency-injector
```

Optional integrations:

```bash
uv add langfuse        # tracing callbacks, see ./langfuse.md
uv add httpx           # async HTTP client for tool gateways
```

This guide assumes Python 3.14, [LangGraph](https://github.com/langchain-ai/langgraph), and an event-loop runtime (FastAPI / Starlette / standalone `asyncio`).

## Wire Up

Routes call services. Services depend on a **business-layer port**, declared as a `typing.Protocol` (DDD ports & adapters). The port's implementation is a graph — and a graph satisfies the Protocol structurally, no inheritance needed.

```python
# business/ports.py
from typing import Protocol


class ResearchAgent(Protocol):
    async def run(self, input_state) -> object: ...
```

The container builds the implementation once at startup and hands it to the route. The container shape follows [./dependency-injector.md](./dependency-injector.md).

```python
# containers.py
from dependency_injector import containers, providers
from langchain_litellm import ChatLiteLLM

from .graphs import CompactionGraph, ResearchGraph


class Container(containers.DeclarativeContainer):
    config = providers.Configuration(yaml_files=["configs/app.yaml"])

    chat_model = providers.Singleton(
        ChatLiteLLM,
        model=config.llm.model,
        temperature=config.llm.temperature,
    )

    # One Singleton per concrete graph. Each graph compiles its StateGraph
    # in __init__ and reuses it across requests; per-request context flows
    # via Runtime[ContextT] and RunnableConfig — see Per-request context.
    compaction_agent = providers.Singleton(
        CompactionGraph,
        chat_model=chat_model,
        config=config.graphs.compaction,
    )
    research_agent = providers.Singleton(
        ResearchGraph,
        chat_model=chat_model,
        config=config.graphs.research,
    )
```

`providers.Singleton` is correct for compiled graphs: building the topology and binding nodes is expensive and must happen at startup. If a route accepts a request-driven choice between graphs, do that selection in the route handler with a small `match` block — there is no global registry.

## Building A Graph

To build a graph: pick the building block, use the library's state and middleware types, drop to raw `StateGraph` only when needed, and reach for a base class only when 3+ graphs share infra.

### Use `create_agent` As The Building Block

`create_agent(model, tools, ...)` returns a `CompiledStateGraph` with the agent loop, tool dispatch, and parallel tool calls already wired (parallel tool fanout is dispatched internally via `Send("tools", [tool_call])` — see `langchain/agents/factory.py` in the LangChain source). For most graphs in this codebase, this is the entire build step.

```python
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
from langchain_litellm import ChatLiteLLM

agent = create_agent(
    model=ChatLiteLLM(model="gpt-5.2"),
    tools=[search_tool, fetch_tool],
    system_prompt="You are a research assistant.",
)
result = await agent.ainvoke({"messages": [HumanMessage("Find recent papers on …")]})
```

See the [LangChain `create_agent` docs](https://docs.langchain.com/oss/python/langchain/agents) for the full parameter list.

### Use Library State And Middleware Types

Reach for the library types from `langchain.agents.middleware`:


| Type                                                                             | What it is                                                                                                                       | When to use                                                                            |
| -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `AgentState`                                                                     | TypedDict — `messages: Annotated[list[AnyMessage], add_messages]`, `jump_to`, `structured_response`                              | Default state schema for `create_agent`. Subclass (TypedDict-style) for custom fields. |
| `AgentMiddleware`                                                                | Generic base class with hooks: `before_agent`, `before_model`, `wrap_model_call`, `after_model`, `wrap_tool_call`, `after_agent` | Stateful middleware or middleware that registers additional tools                      |
| `before_model` / `after_model` / `wrap_model_call` / `wrap_tool_call` decorators | Function-level alternatives to subclassing                                                                                       | Stateless hooks                                                                        |
| `OmitFromInput`, `OmitFromOutput`, `PrivateStateAttr`                            | Annotations marking state fields out of the input or output schema                                                               | When extended state has internal fields the API shouldn't see                          |


```python
from typing import Annotated

from langchain.agents import create_agent
from langchain.agents.middleware import AgentState, OmitFromInput, PrivateStateAttr


class ResearchAgentState(AgentState):
    """Custom state extending the library default."""

    query: str
    citations: Annotated[list[str], OmitFromInput]
    _scratch: Annotated[str, PrivateStateAttr]


agent = create_agent(
    model=chat_model,
    tools=[search_tool, fetch_tool],
    state_schema=ResearchAgentState,
)
```

Cross-link [LangChain middleware overview](https://docs.langchain.com/oss/python/langchain/middleware/overview) for the hook surface and built-in middleware inventory; see [./langchain.md#middleware](./langchain.md#middleware) for our composition guidance.

See also: [`AgentState` reference](https://reference.langchain.com/python/langchain/agents/middleware/AgentState), [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/).

### When To Drop To Raw `StateGraph`

Drop to `StateGraph` only when topology genuinely doesn't fit `create_agent`'s loop — exotic routing, parallel fanout outside tool calls, or a multi-step pipeline that isn't an agent. State for raw `StateGraph` can be `TypedDict`, `@dataclass`, or `pydantic.BaseModel` (`langgraph._internal._typing.StateLike` documents the union — private, so reference, don't import). Every node is `async def`.

```python
from langgraph.graph import END, START, StateGraph
from typing_extensions import TypedDict


class IngestionState(TypedDict):
    raw_pages: list[str]
    cleaned_pages: list[str]


async def _clean(state: IngestionState) -> dict:
    return {"cleaned_pages": [await sanitize(p) for p in state["raw_pages"]]}


async def _dedupe(state: IngestionState) -> dict:
    return {"cleaned_pages": list({p: None for p in state["cleaned_pages"]})}


def _build() -> StateGraph:
    graph = StateGraph(IngestionState)
    graph.add_node("clean", _clean)
    graph.add_node("dedupe", _dedupe)
    graph.add_edge(START, "clean")
    graph.add_edge("clean", "dedupe")
    graph.add_edge("dedupe", END)
    return graph
```

This is no longer the spine of the doc — most LangGraph code in this codebase lives behind `create_agent`.

See also: [LangGraph low-level concepts](https://langchain-ai.github.io/langgraph/concepts/low_level/).

## Architecture Examples

Two architectures, both built on `create_agent`.

### One-agent architecture

One `create_agent` with a few tools. All context lives on a single shared `AgentState`. The agent's model can emit multiple parallel tool calls in one `AIMessage`, and `create_agent` dispatches them in parallel via `Send("tools", [tool_call])` (see `langchain/agents/factory.py:1743`) — no work in user code.

```python
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
from langchain_litellm import ChatLiteLLM


class OneAgentResearch:
    def __init__(self, *, chat_model: ChatLiteLLM, config: dict) -> None:
        self.compiled = create_agent(
            model=chat_model,
            tools=[search_tool, fetch_tool],
            system_prompt=config["system_prompt"],
        )

    async def run(self, *, query: str) -> str:
        result = await self.compiled.ainvoke({"messages": [HumanMessage(query)]})
        return result["messages"][-1].content
```

Use this when one agent can hold the entire context: the user's question, the tool calls, and the final answer fit on one message thread without overflowing the model's context window.

### Supervisor-subagent architecture

A supervisor `create_agent` that binds *subagent-tools*. Each subagent-tool, when invoked, runs an isolated inner `create_agent` with its own `AgentState`. The supervisor's only job is to **route and delegate**: which subagent, what task, what partial context. Each subagent sees only what the supervisor passes via the tool's arguments — never the supervisor's full message history.

```python
from langchain.agents import create_agent
from langchain_core.messages import HumanMessage
from langchain_core.tools import tool
from langchain_litellm import ChatLiteLLM


@tool
async def research_subagent(task: str, focus_keywords: list[str]) -> str:
    """Spawn a research subagent with isolated context.

    Args:
        task: The delegated subtask to research.
        focus_keywords: Hints to narrow the subagent's search.
    """
    inner = create_agent(
        model=ChatLiteLLM(model="gpt-5.2"),
        tools=[search_tool, fetch_tool],
        system_prompt=f"Research focus: {focus_keywords}. Task: {task}",
    )
    output = await inner.ainvoke({"messages": [HumanMessage(task)]})
    return output["messages"][-1].content


supervisor = create_agent(
    model=ChatLiteLLM(model="gpt-5.2"),
    tools=[research_subagent, summarize_subagent],
    system_prompt=(
        "Decompose the user's question into subtasks. "
        "Spawn research_subagent or summarize_subagent in parallel "
        "for each subtask. Combine the results into a final answer."
    ),
)
```

Design rules:

- **Tools = subagents.** Spawning a subagent is just a tool call. The supervisor selects which subagent to run by selecting which tool to call.
- **Tools — own or delegated.** A subagent may bring its own fixed tool list, *or* receive a curated tool selection from the supervisor as part of the tool arguments. Pick whichever fits the delegation model — a focused subagent with fixed tools, or a flexible subagent that runs whatever the supervisor hands it.
- **Context isolation.** Each subagent gets only what the supervisor passes via tool arguments — task, partial context, optimized query, optionally a list of allowed tools. The supervisor's message history is not propagated.
- **Parallel spawn.** When the supervisor's model emits multiple subagent-tool calls in one response, `create_agent` dispatches them in parallel via `Send`. No special wiring needed.
- **Composition.** A subagent may itself be another `create_agent`, a custom raw-`StateGraph`, or a small one-shot LLM call. From the supervisor's view it's just a tool.

Reach for this when: the task decomposes naturally into independent subtasks; a single agent's context window can't hold all the work; you want context-isolated retries that don't pollute the supervisor's history.

## Reducers

When parallel branches write the same state key, declare a reducer; without one, branches overwrite each other. The canonical pattern is **dynamic fan-out**: planner → `Send("worker", partial_state)` → workers writing to a reducer key → aggregate.

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict

from langgraph.graph import END, START, StateGraph
from langgraph.types import Send


class ToolResult(TypedDict):
    tool: str
    input: str
    output: str
    error: str | None


class State(TypedDict):
    queries: list[str]
    results: Annotated[list[ToolResult], operator.add]
    final_answer: str


async def plan(state: State) -> dict:
    return {"queries": ["query A", "query B", "query C"]}


def fanout(state: State) -> list[Send]:
    return [Send("worker", {"query": q}) for q in state["queries"]]


async def worker(state: dict) -> dict:
    query = state["query"]
    output = await search_tool.ainvoke(query)
    return {
        "results": [
            {
                "tool": "search",
                "input": query,
                "output": output,
                "error": None,
            }
        ]
    }


async def aggregate(state: State) -> dict:
    return {"final_answer": synthesize(state["results"])}


builder = StateGraph(State)
builder.add_node("plan", plan)
builder.add_node("worker", worker)
builder.add_node("aggregate", aggregate)
builder.add_edge(START, "plan")
builder.add_conditional_edges("plan", fanout, ["worker"])
builder.add_edge("worker", "aggregate")
builder.add_edge("aggregate", END)
graph = builder.compile()
```

Key elements:

- **Reducer key.** `Annotated[list[ToolResult], operator.add]` concatenates branch outputs instead of overwriting. For custom merge logic (deep-merge dicts, dedup by id), define `(left: T, right: T) -> T` and pass it in place of `operator.add`. `add_messages` from `langgraph.graph.message` is the same idea, specialized for message dedup — use it on `messages` fields.
- **Structured aggregation.** Aggregate into typed dicts (e.g., `ToolResult`), not raw strings. Structure makes retries, debugging, observability, and downstream synthesis easier.
- `**Send` is for I/O-bound work.** Tool calls, retrieval, API requests, multiple LLM calls, DB lookups — these benefit from parallel fanout. CPU-heavy work (NumPy passes, large regex sweeps) blocks the event loop and doesn't gain from `Send` parallelism — run those in a worker pool, not as graph nodes.

`create_agent` already does this internally for parallel tool calls — you only build a custom planner / fanout / aggregate when the parallelism is *outside* the model's tool-call loop.

See also: [LangGraph state-reducers how-to](https://langchain-ai.github.io/langgraph/how-tos/state-reducers/).

## Streaming

How to surface execution events from a running graph. The runtime is `graph.astream(stream_mode=...)` — no custom event bus, no `asyncio.Queue`. Producers emit; consumers iterate the async iterator. Async stack required end-to-end (FastAPI → graph → tools → LLM). How those events reach a client (SSE framing, WebSocket, heartbeats, cancellation propagation) is transport, not LangGraph — handle at the route layer.

### `stream_mode`

What kind of execution data gets streamed back while the graph runs:


| Mode       | What it yields                                | Typical use                          |
| ---------- | --------------------------------------------- | ------------------------------------ |
| `updates`  | State **delta** returned by each node         | Progress updates, normal backend use |
| `messages` | LLM tokens/messages as they arrive            | Chat UI token streaming              |
| `custom`   | Whatever you wrote via `stream_writer({...})` | Tool progress, status, logs          |
| `values`   | Full graph state after each step              | Debugging / full inspection          |


Pick one, or pass a list to multiplex. With a list, the iterator yields `(mode, chunk)` tuples so the consumer can dispatch on `mode`. `updates` is the usual default; for chat UIs, `["updates", "messages"]` is the common combination.

```python
async for chunk in graph.astream(input_state, stream_mode="updates"):
    print(chunk)
```

```python
async for mode, chunk in graph.astream(
    input_state,
    stream_mode=["updates", "messages"],
):
    if mode == "updates":
        print("state update:", chunk)
    elif mode == "messages":
        token, metadata = chunk
        print("llm token:", token.content)
```

See also: [LangGraph streaming concepts](https://langchain-ai.github.io/langgraph/concepts/streaming/), [`astream` reference](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.CompiledStateGraph.astream).

### `stream_writer` — emitting custom events

LangGraph exposes a writer via the run's contextvar. Producers call it with arbitrary dicts; the consumer sees those dicts under `mode == "custom"`. Give every event a `type` discriminator so the consumer can dispatch and ignore unknown shapes.

From a middleware hook — `Runtime` exposes the writer:

```python
from langchain.agents.middleware import before_agent
from langgraph.runtime import Runtime


@before_agent
async def announce_start(state, runtime: Runtime) -> None:
    runtime.stream_writer({"type": "progress", "stage": "thinking"})
```

From a raw `StateGraph` node — resolve via `langgraph.config`:

```python
from langgraph.config import get_stream_writer


async def fetch_node(state):
    write = get_stream_writer()
    write({"type": "tool_event", "tool_type": "fetch", "status": "started"})
    results = await fetch_tool.ainvoke(state["query"])
    write({"type": "tool_event", "tool_type": "fetch", "status": "completed"})
    return {"fetched": results}
```

Sub-graph events (e.g., an inner `create_agent` invoked as a tool) inherit the parent's stream context and appear in the outer `astream` output automatically — no extra wiring.

See also: [`get_stream_writer` reference](https://langchain-ai.github.io/langgraph/reference/config/#langgraph.config.get_stream_writer).

### Streaming Pitfalls

- **Calling `graph.invoke(...)` from a streaming caller.** Sync `invoke` blocks the event loop and never yields chunks. Use `astream`.
- **Default `stream_mode` is `values`.** Useful for debugging, rarely useful for UI. Pick a mode deliberately.
- **No `type` on custom events.** Frontend has nothing to dispatch on. Always include a discriminator.
- **Calling `get_stream_writer()` from a background thread or `asyncio.to_thread` helper.** The writer is a contextvar — without explicit propagation it isn't there. Emit from inside the async graph.

## Per-request context

Per-request data — `request_id`, `user_id`, `tenant_id`, tracing fields — flows through library-provided types. Don't invent a custom carrier.


| Carries                           | Use                                                                                                                                                                                                 |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Typed per-request context         | `Runtime[ContextT]` — parameterize `ContextT` with a project-defined `TypedDict`. `ModelRequest.runtime` carries it through middleware; nodes receive it as a parameter.                            |
| `run_id`, `metadata`, `callbacks` | `RunnableConfig` — passed via `agent.ainvoke(input, config={...})`                                                                                                                                  |
| Custom stream events              | `runtime.stream_writer({...})` (inside middleware hooks) or `langgraph.config.get_stream_writer()({...})` (inside raw `StateGraph` nodes); see [Streaming](#streaming)                              |


One illustrative `RequestContext` shape (yours will differ — fields are per-project, not canon):

```python
from typing_extensions import TypedDict

from langchain.agents.middleware import before_model
from langgraph.runtime import Runtime


class RequestContext(TypedDict):
    request_id: str
    user_id: str
    tenant_id: str | None


@before_model
async def log_request(state, runtime: Runtime[RequestContext]) -> None:
    print(runtime.context["request_id"])


# Pass at invocation:
await agent.ainvoke(
    {"messages": [HumanMessage("...")]},
    context={"request_id": "...", "user_id": "...", "tenant_id": None},
)
```

Use the library types directly. A custom `MetaInput` / `RequestMeta` class drifts away from LangChain's middleware contract and reinvents what `Runtime[ContextT]` already gives you.

## Configuration

Graph configuration flows through the DI container's `providers.Configuration(yaml_files=[...])` — the same mechanism every other gateway uses. There is **no** `configs/{ClassName}.yaml` convention: name your YAML files however the project's `app.yaml` / `graphs.yaml` layout makes sense, and let the container pass each graph the slice it needs.

```yaml
# configs/app.yaml
graphs:
  compaction:
    timeout_seconds: 60
    max_iterations: 4
  research:
    timeout_seconds: 180
    max_iterations: 12
```

```python
# containers.py
compaction_agent = providers.Singleton(
    CompactionGraph,
    chat_model=chat_model,
    config=config.graphs.compaction,
)
```

See [./dependency-injector.md](./dependency-injector.md) for the full `Configuration` provider walkthrough.

See also: [dependency-injector `Configuration` provider](https://python-dependency-injector.ets-labs.org/providers/configuration.html).

## Adding A Graph

Tight checklist when introducing a new graph. Each line is the minimum bar — flesh out per project.

- **Implement** with `create_agent` from `langchain.agents` (or raw `StateGraph` only when topology genuinely doesn't fit).
- **State**: extend `langchain.agents.middleware.AgentState` for `create_agent`-based graphs; `TypedDict` / `@dataclass` / `BaseModel` for raw `StateGraph`.
- **Per-request context**: pass via `Runtime[ContextT]` and `RunnableConfig` — don't invent a carrier.
- **Configure** via the DI container (`providers.Configuration(yaml_files=[...])`) — no per-class YAML mandate. See [./dependency-injector.md](./dependency-injector.md).
- **Wire** as a Protocol-typed port in the business layer (`ResearchAgent`, `CompactionAgent`, …); the graph satisfies it structurally — no inheritance.
- **Expose** via `providers.Singleton` in the container so the graph compiles once at startup.
- **Service-exposed graphs** need a route + auth + rate-limit policy. See [./slowapi.md](./slowapi.md).
- **Tests** in two paths: API tests mock the gateway Protocol; graph functional tests use `LLMToolEmulator` with a real LLM. See [Testing](#testing).

## Mixins

For the general Protocol / ABC / Mixin / Composition trade-offs, see [./abstractions.md](./abstractions.md). Graph-specific guidance: reach for a mixin only when 2+ graphs need the same utility behavior — emitting token usage, structured logging hooks, a small cache wrapper.

Example — emit token usage at end-of-run as a custom stream event, shared by every graph that produces an `AIMessage` chain:

```python
# mixins/token_usage.py
from langchain_core.messages import AnyMessage
from langgraph.config import get_stream_writer


class TokenUsageEmitMixin:
    """Emit accumulated token usage to the LangGraph stream as a custom event."""

    def emit_token_usage(self, *, messages: list[AnyMessage]) -> None:
        usage = sum_usage_metadata(messages)
        get_stream_writer()({"type": "usage", **usage})
```

```python
class ResearchGraph(TokenUsageEmitMixin):
    def __init__(self, *, chat_model, tools, config) -> None:
        self.compiled = create_agent(
            model=chat_model, tools=tools, system_prompt=config["system_prompt"]
        )

    async def run(self, *, query: str) -> str:
        result = await self.compiled.ainvoke({"messages": [HumanMessage(query)]})
        self.emit_token_usage(messages=result["messages"])
        return result["messages"][-1].content
```

The hard rule: **mixins do not call `graph.add_node`, `graph.add_edge`, or `graph.add_conditional_edges`.** A mixin that adds graph topology turns into a hidden base class — topology lives in the graph class, not in a sibling.

## Testing

Two test paths, different purposes:

- **API tests** verify the *backend* works functionally. Mock the gateway `Protocol` (a one-line fake satisfies it structurally); don't mock LiteLLM, individual tools, or anything below the port. Cheap, fast, deterministic.
- **Graph functional tests** verify the *graph* works functionally. Run against the real LLM (via `ChatLiteLLM`) and emulate tools with `langchain.agents.middleware.LLMToolEmulator` — the emulator itself calls a real LLM to generate plausible tool responses, so tools are stubbed without leaving the LLM-driven loop. Slower, real-LLM cost, non-deterministic — gate behind a marker for scheduled / pre-release runs (specific marker convention TBU).

Test layout:

```
tests/
  conftest.py
  api/                    # API tests — gateway Protocol mocks
    test_research_route.py
  graphs/
    {domain}/             # graph functional tests with LLMToolEmulator
      test_research_graph.py
```

For app-lifespan setup, `pytest_asyncio.fixture` wiring, and `LifespanManager`, see [./python-tests.md](./python-tests.md).

See also: [pytest](https://docs.pytest.org/en/stable/), [pytest-asyncio](https://pytest-asyncio.readthedocs.io/en/latest/), [pytest markers](https://docs.pytest.org/en/stable/how-to/mark.html).

### API Tests

The agent gateway is a `Protocol`. Implement it structurally with a one-line dataclass fake — no inheritance, no scripted-LLM ceremony.

```python
# tests/api/conftest.py
from dataclasses import dataclass, field
from typing import Any

from app.business.ports import ResearchAgent  # Protocol


@dataclass
class MockResearchAgent:
    """Implements ResearchAgent structurally — no inheritance needed."""

    output: Any
    calls: list[Any] = field(default_factory=list)

    async def run(self, input_state) -> Any:
        self.calls.append(input_state)
        return self.output
```

Wire via a DI override fixture — see [./dependency-injector.md](./dependency-injector.md):

```python
# tests/api/conftest.py (continued)
import pytest
from dependency_injector import providers

from app.containers import Container


@pytest.fixture
def mock_agent() -> MockResearchAgent:
    return MockResearchAgent(output=None)


@pytest.fixture
def container(mock_agent: MockResearchAgent) -> Container:
    container = Container()
    container.research_agent.override(providers.Object(mock_agent))
    yield container
    container.research_agent.reset_override()
```

Then the route test exercises route + DI; everything below the port is the trivial fake:

```python
# tests/api/test_research_route.py
from app.schemas import ResearchOutputState


async def test_research_route_returns_agent_output(client, mock_agent):
    mock_agent.output = ResearchOutputState(answer="found it")

    response = await client.post("/research/run", json={"query": "..."})

    assert response.status_code == 200
    assert response.json()["answer"] == "found it"
    assert mock_agent.calls[0].query == "..."
```

For repository / vector DB / external service mocking, override the corresponding container providers the same way.

See also: [`typing.Protocol`](https://docs.python.org/3/library/typing.html#typing.Protocol), [Dependency Injector overriding](https://python-dependency-injector.ets-labs.org/providers/overriding.html), [FastAPI testing dependencies](https://fastapi.tiangolo.com/advanced/testing-dependencies/), [HTTPX async client](https://www.python-httpx.org/async_client/).

### Graph Functional Tests

Check that the graph itself produces sensible behavior end-to-end with the real LLM. To keep tools off the network, use `langchain.agents.middleware.LLMToolEmulator` as middleware: the emulator short-circuits real tool execution and asks an LLM to generate a plausible tool response based on the tool's name, description, and arguments.

```python
# tests/graphs/research/test_research_graph.py
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolEmulator
from langchain_core.messages import HumanMessage
from langchain_litellm import ChatLiteLLM

from app.tools import fetch_tool, search_tool


async def test_research_agent_finds_relevant_sources():
    agent = create_agent(
        model=ChatLiteLLM(model="gpt-5.2"),
        tools=[search_tool, fetch_tool],
        middleware=[
            LLMToolEmulator(),  # emulate ALL tools by default
        ],
        system_prompt="You are a research assistant.",
    )

    result = await agent.ainvoke({
        "messages": [HumanMessage("Find recent papers on transformer scaling laws")]
    })

    final = result["messages"][-1].content
    assert isinstance(final, str)
    assert len(final) > 50
    assert "transformer" in final.lower() or "scaling" in final.lower()
```

Notes:

- **`LLMToolEmulator()` with no args emulates all tools.** Pass tool names or `BaseTool` instances to emulate selectively: `LLMToolEmulator(tools=["search_tool"])` keeps `fetch_tool` real (rare — usually you emulate everything in tests).
- **Default emulation model** is `anthropic:claude-sonnet-4-5-20250929`. Override with `LLMToolEmulator(model="openai:gpt-4o")` or pass a `BaseChatModel` instance.
- **Real LLM calls cost money and aren't deterministic.** Don't run them on every PR — run them on a schedule or before release. Marker convention is TBU; until settled, declare any marker you use in `pyproject.toml` under `markers = [...]` and run with `--strict-markers`.
- **Behavioral assertions, not exact strings.** Assert that the final answer contains the topic, isn't empty, and has the expected shape. Don't assert exact strings — the LLM produces variation.
- **Graph topology is the LLM's job to navigate.** Don't assert on node order or visited paths inside `create_agent`'s loop — that's an implementation detail of LangChain. Assert on inputs and outputs.

See also: [`LLMToolEmulator` reference](https://reference.langchain.com/python/langchain/agents/middleware/tool_emulator/LLMToolEmulator).

### Testing Pitfalls

- **Inheriting from the gateway `Protocol` in your fake.** Protocol is structural — implementing the methods is enough. Inheritance is the wrong tool here (see [./abstractions.md](./abstractions.md) for the ABC vs. Protocol vs. Mixin split).
- **Mocking LiteLLM or individual tools in API tests.** The gateway encapsulates them — overriding lower layers means you're testing the wrong seam.
- **Asserting exact strings on LLM output.** Real LLMs vary across runs. Assert on substrings, length bounds, structured-output fields, or message-count bounds.
- **Forgetting that `LLMToolEmulator` still calls a real LLM.** It emulates *tools* via an LLM, not "removes the LLM." If you want zero-cost tests, that's the API path (Protocol mock). The two paths exist for different reasons.
- **Mocking the LangGraph runtime.** The runtime is the thing under test in graph functional tests; mocking it means the test only proves the test wrote itself.

## Checkpointer (Optional)

Use a LangGraph checkpointer only when graph runtime state must survive across invocations:

- multi-turn graph threads
- interrupt/resume (human-in-the-loop)
- retry-from-checkpoint
- time-travel debugging / replay
- long-running resumable execution

Conversation history, agent execution records, audit logs, and final outputs belong in the application's **primary persistence layer** — not in the checkpointer. The checkpointer is a runtime continuation mechanism, not a system of record. For one-shot agents (run the graph, persist the outcome via the app persistence layer, discard transient runtime state), a checkpointer is unnecessary.

If you do need one, pick a backend:

- `InMemorySaver` (`langgraph.checkpoint.memory`) — tests / single-process scripts.
- `AsyncSqliteSaver` (`langgraph.checkpoint.sqlite.aio`) — local dev, single-instance services.
- `AsyncPostgresSaver` (`langgraph.checkpoint.postgres.aio`) — production; call `await saver.setup()` on app startup to run DDL migrations.

Wire it through `create_agent(..., checkpointer=saver)` (or `StateGraph.compile(checkpointer=saver)` for raw graphs). At invoke time, pass the thread under `config={"configurable": {"thread_id": "..."}}` — without `thread_id`, checkpointing silently no-ops.

For the deeper API (`aget_state`, `aget_state_history`, `interrupt(...)`, `Command(resume=...)`, cross-thread `Store`), see the [LangGraph persistence docs](https://langchain-ai.github.io/langgraph/concepts/persistence/).

## Production Checklist


| Item                     | Setting                                                                                                                                                                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Graph instances          | Built once at startup as `providers.Singleton`; never construct in a route handler                                                                                                                                                                            |
| State shape              | `langchain.agents.middleware.AgentState` (or a TypedDict extending it) for `create_agent`-based graphs; `TypedDict` / `@dataclass` / `BaseModel` for raw `StateGraph`                                                                                         |
| Per-request context      | Pass via `Runtime[ContextT]` (typed) or `RunnableConfig` (`run_id` / `metadata` / `callbacks`). For custom streaming events, use `runtime.stream_writer` or `langgraph.config.get_stream_writer()` — see [Streaming](#streaming)                                                                                |
| Async nodes              | Every node is `async def`. Use `await llm.ainvoke(...)` and `await tool.ainvoke(...)` — never the sync versions                                                                                                                                               |
| Graph execution          | `await graph.ainvoke(...)` or `async for ... in graph.astream(...)` — never `graph.invoke(...)` from an async stack                                                                                                                                           |
| Reducers                 | Declare `Annotated[list[...], add_messages]` or `operator.add` (or a custom reducer) for any field two parallel branches write to                                                                                                                             |
| Checkpointer (optional)  | Only when runtime state must survive across invocations (multi-turn threads, interrupt/resume, replay) — see [Checkpointer (Optional)](#checkpointer-optional). One-shot agents don't need one.                                                               |
| Iteration / tool budgets | Set per-graph in config; enforce via middleware — see [./langchain.md](./langchain.md)                                                                                                                                                                        |
| Configuration            | Sourced through `providers.Configuration(yaml_files=[...])` in the DI container — see [./dependency-injector.md](./dependency-injector.md)                                                                                                                    |


## Pitfalls

- **Wrapping a one-LLM-call task in a graph.** A graph with one node and two edges is just a roundabout LLM call. Use [./litellm.md](./litellm.md) directly.
- **Sharing context between supervisor and subagent.** The point of the multi-agent pattern is context isolation. If the subagent needs the supervisor's full message history, it's not really a separate agent — it's a one-agent architecture in disguise.
- **Hand-rolling a per-request context carrier.** `Runtime[ContextT]` already exists — parameterize it and pass it through. A custom `MetaInput` / `RequestMeta` class is reinventing the wheel and drifts away from LangChain's middleware contract.
- **Mixing sync and async randomly.** Once any layer is async, every layer above it must `await`. A `def` node that calls a sync tool from inside an `async` graph runs but blocks the event loop on every step. Fix the inner sync call; don't paper over it with `asyncio.to_thread`.
- **Aggregating giant raw strings across parallel branches.** Use a structured shape (e.g., `ToolResult` TypedDict) — string concatenation loses structure and breaks debuggability.
- **CPU-heavy work inside async nodes.** `Send` parallelism is for I/O-bound work. Heavy NumPy / pandas / regex passes inside a node block the event loop just like sync I/O — push them out to a worker pool.
- **Building the graph in a request handler.** Compiling a `StateGraph` (or calling `create_agent`) binds nodes, edges, and middleware — doing that on every request burns latency and breaks any caching the graph maintains. Build once in the DI container, inject the singleton.
- **Putting nodes or edges in mixins.** A mixin that calls `graph.add_node` or `graph.add_edge` couples the mixin to a specific topology and turns it into a hidden base. Topology lives in the graph class.
- **Sharing mutable state across requests on the graph instance.** The compiled graph is a singleton; per-request data lives in the state passed to `ainvoke`. Never store request-scoped data as `self.something` on the graph.
- **Forgetting reducers on parallel writes.** Two `Send`-fanned-out workers both updating `state.results` will silently overwrite unless `Annotated[list[...], operator.add]` is declared.
- **Skipping schema control.** For `create_agent`-based graphs, use `OmitFromInput` / `OmitFromOutput` / `PrivateStateAttr` annotations to keep internal state out of the API. For raw `StateGraph`, pass `input_schema=` and `output_schema=` to `StateGraph(...)`.
