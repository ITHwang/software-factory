# How to Run MCP Servers

> Last updated: 2026-05-16

## TL;DR

Three ways a Python application can talk to MCP (Model Context Protocol) servers: **in-memory** (same process, direct `Client` over an in-memory stream), **subprocess** (each server runs as a child process, stdio transport), and **streamable HTTP** (servers run remotely, HTTP transport).

**Use this when:**

- choosing the transport for a new MCP integration
- wiring MCP session lifecycle into a FastAPI app's `lifespan`
- deciding whether to colocate MCP servers in-process, isolate them as subprocesses, or run them remotely

**Don't use this for:**

- structuring the repo that hosts the MCP server projects → `[./multi-project-management-with-uv.md](./multi-project-management-with-uv.md)`
- LLM capability ports / prompt templates → `[./langfuse.md](./langfuse.md)`
- FastAPI app factory / lifespan basics → `[./py-backend-architecture.md](./py-backend-architecture.md)`

## Table of Contents

- [1. Overview](#1-overview)
- [2. Method 1: In-Memory MCP Server](#2-method-1-in-memory-mcp-server)
- [3. Method 2: Subprocess MCP Server](#3-method-2-subprocess-mcp-server)
- [4. Method 3: Streamable HTTP MCP Server](#4-method-3-streamable-http-mcp-server)
- [5. Recommendations by Use Case](#5-recommendations-by-use-case)

## 1. Overview

**MCP (Model Context Protocol)** is a standard protocol for safely and efficiently exchanging context between applications and AI models. It lets AI agents reach a wide variety of tools and resources and supports extensible architectures.

When MCP server projects are managed via [Path Dependencies](./multi-project-management-with-uv.md) (the recommended layout), all three transports below are available and interchangeable. The **Subprocess** approach is the typical default for application servers that bundle their own tooling.

1. **In-Memory MCP Server**: direct in-process communication between objects.
2. **Subprocess MCP Server**: each server runs as a subprocess and talks over stdio (typical default).
3. **Streamable HTTP MCP Server**: communication over HTTP, possibly to a remote host.

## 2. Method 1: In-Memory MCP Server

1. The client and server live in the same process; FastMCP's `Client` connects directly to a server instance.
2. No subprocess is spawned — only an `anyio` memory object stream is created. The MCP protocol flows over the stream's reader/writer, so memory usage is low and round-trips are fast.

```python
import asyncio

from fastmcp import Client
from web_search import mcp_server


async def use_mcp_server_directly():
    """Example: use a FastMCP server instance directly."""

    # Create the client and connect to the server instance.
    client = Client(mcp_server)

    async with client:
        # Inspect available tools, resources, and prompts.
        tools = await client.list_tools()
        resources = await client.list_resources()
        prompts = await client.list_prompts()

        print(f"Available tools: {[tool.name for tool in tools.tools]}")

        # Invoke a tool.
        result = await client.call_tool("web_search", {"query": "AI trends 2024"})
        print(f"Search result: {result}")
```

## 3. Method 2: Subprocess MCP Server

The typical default for bundled deployments: each MCP server runs as an independent subprocess and the application talks to it over stdio.

### Project layout

```text
<project-root>/
├── app/config/mcp.json                 # MCP server registry
└── mcp-servers/                        # standalone MCP servers
    ├── web-search/
    └── document-retriever/
```

### MCP server registry

```json
// app/config/mcp.json

{
  "mcpServers": {
    "web_search": {
      "command": "python",
      "args": ["-m", "web_search"]
    },
    "document_retriever": {
      "command": "python",
      "args": ["-m", "document_retriever"]
    }
  }
}
```

### Managing MCP sessions inside the FastAPI lifespan

```python
# app/server.py

exit_stack: AsyncExitStack = AsyncExitStack()
cleanup_lock: asyncio.Lock = asyncio.Lock()


def get_mcp_configs() -> dict:
    load_dotenv()
    project_root = os.environ["PROJECT_ROOT"]
    mcp_servers_json_path = Path(project_root) / "app" / "config" / "mcp.json"
    with open(mcp_servers_json_path) as f:
        mcp_configs = json.load(f)
    return mcp_configs


async def get_mcp_session(mcp_configs: dict) -> ClientSession:
    """Initialize subprocess MCP sessions."""
    mcp_sessions: dict[str, dict[str, str | ClientSession]] = {}
    for server_name, server_config in mcp_configs["mcpServers"].items():
        server_params = StdioServerParameters(**server_config)
        stdio_transport = await exit_stack.enter_async_context(
            stdio_client(server_params)
        )
        read, write = stdio_transport
        session = await exit_stack.enter_async_context(ClientSession(read, write))

        mcp_session_id = str(uuid.uuid4())
        mcp_sessions[mcp_session_id] = {
            "name": server_name,
            "session": session,
        }

        logger.info(f"MCP Session for the server {server_name} is initialized")

    return mcp_sessions


async def init_mcp_sessions(app: FastAPI) -> None:
    """Initialize MCP clients."""
    mcp_configs = get_mcp_configs()
    mcp_sessions = await get_mcp_session(mcp_configs)
    app.state.mcp_sessions = mcp_sessions


async def cleanup_mcp_sessions(app: FastAPI) -> None:
    """Tear down MCP clients."""
    async with cleanup_lock:
        await exit_stack.aclose()

        for key, session in app.state.mcp_sessions.items():
            await session.aclose()
            logger.info(f"MCP Session for the server {key} is cleaned up")
        app.state.mcp_sessions.clear()


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage MCP sessions across the application lifecycle."""
    # On startup: initialize MCP sessions.
    await init_mcp_sessions(app=app)

    yield

    # On shutdown: clean up MCP sessions.
    await cleanup_mcp_sessions(app=app)
```

## 4. Method 3: Streamable HTTP MCP Server

Run MCP servers remotely (or on the same host as a separate service) and talk to them over HTTP. Suited to distributed environments and microservice architectures.

### Running a streamable HTTP MCP server

```python
from fastmcp import FastMCP
from web_search.mcp_servers import mcp_server


def run_http_server():
    """Run an MCP server over HTTP."""
    mcp_server.run(transport="http", host="localhost", port=8001)


if __name__ == "__main__":
    run_http_server()
```

## 5. Recommendations by Use Case

### In-Memory MCP Server

- **Integrated inside the FastAPI app**: serve everything from one process.
- **Performance-sensitive paths**: minimal overhead, fastest round-trips.
- **Simple deployments**: no extra infrastructure to manage.

### Subprocess MCP Server

- **Process isolation**: an MCP server crash should not take the main application down.
- **Independent updates**: ship MCP server changes on a schedule independent from the main application.
- **Resource control**: cap each MCP server's resource footprint individually.

### HTTP MCP Server

- **Microservice architectures**: services already live across hosts.
- **Multiple clients**: several applications need to share the same MCP server.
- **Cloud deployments**: ship the MCP server as its own service on AWS ECS, Kubernetes, etc.