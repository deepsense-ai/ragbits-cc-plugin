# New MCP Server Checklist

Quality gates for integrating MCP (Model Context Protocol) servers with a ragbits agent.

## Definition Requirements

- [ ] MCP server type is chosen: `MCPServerStdio` (subprocess), `MCPServerSse` (HTTP/SSE), or `MCPServerStreamableHttp` (bidirectional HTTP)
- [ ] Server is used as an async context manager (`async with ... as server:`)
- [ ] Server is passed to the Agent constructor via `mcp_servers=[server]`
- [ ] Tool names from MCP servers do not conflict with native tool names on the agent

## Imports

```python
# Pick the server type you need:
from ragbits.agents.mcp import MCPServerStdio
from ragbits.agents.mcp import MCPServerSse
from ragbits.agents.mcp import MCPServerStreamableHttp

from ragbits.agents import Agent
```

## Server Types

### MCPServerStdio — Subprocess-Based

Spawns a subprocess and communicates via stdin/stdout. Best for local MCP servers.

```python
from ragbits.agents.mcp import MCPServerStdio

server = MCPServerStdio(
    params={
        "command": "python",                # Required: executable to run
        "args": ["-m", "mcp_server_fetch"], # Optional: command-line arguments
        "env": {"API_KEY": "..."},          # Optional: environment variables
        "cwd": "/path/to/server",           # Optional: working directory
    },
    cache_tools_list=True,                  # Cache tools list (set True if tools don't change)
    name="fetch-server",                    # Optional: readable name
    client_session_timeout_seconds=60,      # Optional: read timeout (default: 5)
)
```

**Params type (`MCPServerStdioParams`):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | `str` | Yes | Executable to run (e.g., `"python"`, `"node"`, `"npx"`) |
| `args` | `list[str]` | No | Command-line arguments |
| `env` | `dict[str, str]` | No | Environment variables |
| `cwd` | `str \| Path` | No | Working directory |
| `encoding` | `str` | No | Text encoding (default: `"utf-8"`) |
| `encoding_error_handler` | `"strict" \| "ignore" \| "replace"` | No | Error handling (default: `"strict"`) |

### MCPServerSse — HTTP with Server-Sent Events

Connects to a remote MCP server via HTTP with SSE streaming. Best for remote servers.

```python
from ragbits.agents.mcp import MCPServerSse

server = MCPServerSse(
    params={
        "url": "http://localhost:8000/mcp",       # Required: server URL
        "headers": {"Authorization": "Bearer ..."}, # Optional: HTTP headers
        "timeout": 10,                              # Optional: HTTP timeout (default: 5s)
        "sse_read_timeout": 300,                    # Optional: SSE timeout (default: 300s)
    },
    cache_tools_list=True,
    name="remote-mcp-server",
    client_session_timeout_seconds=10,
)
```

### MCPServerStreamableHttp — Bidirectional HTTP Streaming

Uses bidirectional HTTP streaming for persistent connections. Best for long-running interactions.

```python
from ragbits.agents.mcp import MCPServerStreamableHttp
from datetime import timedelta

server = MCPServerStreamableHttp(
    params={
        "url": "http://localhost:8000/mcp",           # Required: server URL
        "headers": {"X-API-Key": "..."},              # Optional: HTTP headers
        "timeout": timedelta(seconds=15),             # Optional: accepts timedelta or float
        "sse_read_timeout": timedelta(minutes=10),    # Optional: stream timeout
        "terminate_on_close": True,                   # Optional: terminate on close (default: True)
    },
    cache_tools_list=True,
    name="streamable-server",
)
```

## Common Constructor Parameters

All three server types share these constructor parameters:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `params` | dict | — | Server-specific connection parameters (see above) |
| `cache_tools_list` | `bool` | `False` | Cache the tools list. Set `True` if server tools don't change at runtime |
| `name` | `str \| None` | `None` | Readable name for the server (auto-generated if not provided) |
| `client_session_timeout_seconds` | `float \| None` | `5` | Read timeout for MCP ClientSession communication |

## Usage Pattern with Agent

MCP servers must be used as async context managers. The agent is created inside the context:

```python
from ragbits.agents import Agent
from ragbits.agents.mcp import MCPServerStdio
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt
from pydantic import BaseModel


class MyInput(BaseModel):
    input: str


class MyPrompt(Prompt[MyInput]):
    system_prompt = "You are a helpful assistant with access to external tools."
    user_prompt = "{{ input }}"


async def main() -> None:
    async with MCPServerStdio(
        params={
            "command": "python",
            "args": ["-m", "mcp_server_fetch"],
        },
        client_session_timeout_seconds=60,
    ) as server:
        llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)

        agent = Agent(
            llm=llm,
            prompt=MyPrompt,
            tools=[my_native_tool],       # native tools (optional)
            mcp_servers=[server],          # MCP servers providing additional tools
        )

        result = await agent.run(MyInput(input="Fetch data from example.com"))
        print(result.content)


if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

## Multiple MCP Servers

An agent can use multiple MCP servers simultaneously:

```python
async with MCPServerStdio(params={"command": "python", "args": ["-m", "mcp_server_fetch"]}) as fetch_server, \
     MCPServerSse(params={"url": "http://localhost:9000/mcp"}) as api_server:

    agent = Agent(
        llm=llm,
        prompt=MyPrompt,
        mcp_servers=[fetch_server, api_server],
    )
```

## Combining with A2A Server

When exposing an MCP-enabled agent as an A2A server, the MCP context must wrap the server lifecycle:

```python
from ragbits.agents.a2a.server import create_agent_server

async def main() -> None:
    async with MCPServerStdio(
        params={"command": "python", "args": ["-m", "mcp_server_fetch"]},
        client_session_timeout_seconds=60,
    ) as mcp_server:
        llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)
        agent = Agent(llm=llm, prompt=MyPrompt, mcp_servers=[mcp_server])

        agent_card = await agent.get_agent_card(
            name="MCP-Enabled Agent",
            description="Agent with external tool access via MCP.",
            port=8001,
        )
        server = create_agent_server(agent, agent_card, MyInput)
        await server.serve()
```

## How MCP Tools are Resolved

- Tools from MCP servers are dynamically fetched during agent execution via `_get_all_tools()`
- MCP tools are merged with native tools
- Tool names must be **unique** across all sources — if a native tool and an MCP tool share the same name, `AgentToolDuplicateError` is raised
- If `cache_tools_list=True`, tools are fetched once and cached; otherwise they are fetched on every agent run

## Validation Checklist

- [ ] Server type matches the MCP server's transport (stdio for local, SSE/HTTP for remote)
- [ ] Server is used inside `async with` context manager
- [ ] `client_session_timeout_seconds` is large enough for slow tools (default 5s may be too low)
- [ ] `cache_tools_list=True` is set if MCP server tools don't change at runtime
- [ ] No tool name conflicts between native tools and MCP server tools
- [ ] MCP server package is installed (e.g., `pip install mcp-server-fetch`)
- [ ] Agent is created inside the MCP server context (not before)
- [ ] If combining with A2A, the A2A server lifecycle is inside the MCP context
