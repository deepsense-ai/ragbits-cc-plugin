# Ragbits Agent Specification

Advanced patterns for the ragbits `Agent` class. Read this when the base template in SKILL.md isn't enough.

## Core Imports

Prefer the public re-exports from `ragbits.agents` — the `_main` submodule is internal and subject to change. Everything needed for agents, options, results, context, and tool-call types is already surfaced on the package root.

```python
from ragbits.agents import (
    Agent,
    AgentOptions,
    AgentResult,
    AgentRunContext,
    EventType,
    Hook,
    ToolCall,
    ToolCallResult,
)
from ragbits.agents.tool import ToolEvent, ToolReturn
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt
```

## Agent Constructor

```python
agent = Agent(
    llm=LiteLLM(model_name="gpt-4.1", use_structured_output=True),
    prompt=MyPrompt,
    tools=[my_tool, another_tool],   # optional — plain callables, Tool instances, or nested Agents
    hooks=[my_hook],                  # optional — see references/checklists/new-hook.md
    mcp_servers=[mcp_server],         # optional — see references/checklists/new-mcp.md
    keep_history=True,                # multi-turn memory across run() calls
)
```

## Running the Agent

### Basic — returns `AgentResult`

```python
result = await agent.run(MyInput(message="Hello"))
print(result.content)              # text reply (str, or your output model)
print(result.tool_calls)           # list[ToolCallResult] | None
print(result.usage.total_tokens)   # token usage
print(result.history)              # ChatFormat with the full transcript
```

`agent.run()` returns `AgentResult[PromptOutputT]`, not a bare string. Forgetting `.content` is the top scaffolding mistake — callers end up serializing the whole dataclass.

### Streaming — token-by-token with tool-call visibility

```python
from ragbits.agents import ToolCall, ToolCallResult

async for chunk in agent.run_streaming(MyInput(message="Hello")):
    match chunk:
        case ToolCall():
            print(f"[Calling {chunk.name}: {chunk.arguments}]")
        case ToolCallResult():
            print(f"[Result from {chunk.name}: {chunk.result}]")
        case str():
            print(chunk, end="", flush=True)
```

Only match on the cases you care about; the generator also yields `Usage`, `BasePrompt`, `DownstreamAgentResult`, and `ConfirmationRequest` — safe to ignore if you're just printing text.

## Prompt Patterns

See `references/checklists/new-prompt.md` for the full reference.

### Basic (string output)

```python
class MyInput(BaseModel):
    message: str

class MyPrompt(Prompt[MyInput]):
    system_prompt = "You are a helpful assistant."
    user_prompt = "{{ message }}"
```

Single generic parameter → output type defaults to `str`.

### Structured output (Pydantic model)

```python
class MyOutput(BaseModel):
    answer: str
    confidence: float

class MyPrompt(Prompt[MyInput, MyOutput]):
    system_prompt = "Extract structured data."
    user_prompt = "{{ message }}"

# Structured output needs the LLM configured for it:
llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)
agent = Agent(llm=llm, prompt=MyPrompt)
result = await agent.run(MyInput(message="..."))
assert isinstance(result.content, MyOutput)
```

### Without input

```python
class StaticPrompt(Prompt[None]):
    system_prompt = "You are a code reviewer."
    user_prompt = "Review the provided diff."

agent = Agent(llm=llm, prompt=StaticPrompt)
result = await agent.run(None)
```

### Jinja2 in templates

```python
class MyPrompt(Prompt[MyInput]):
    system_prompt = """
    You manage {{ items | length }} items.
    {% if urgent %}Respond urgently.{% endif %}
    """
    user_prompt = "{{ message }}"
```

## Tool Patterns

See `references/checklists/new-tool.md` for the full reference.

### Basic tool

```python
def search_docs(query: str) -> str:
    """Search documentation for the given query.

    Args:
        query: The search term.

    Returns:
        Matching documentation excerpts.
    """
    return results
```

### Tool with agent run context

```python
from ragbits.agents import AgentRunContext

def audited_action(input_data: str, context: AgentRunContext) -> str:
    """Perform an action and log it.

    Args:
        input_data: The data to process.
        context: Injected automatically — do not pass manually.

    Returns:
        Result of the action.
    """
    # context.deps — dependencies container
    # context.usage — running token usage
    # context.downstream_agents — registry of agents that participated in this run
    return process(input_data)
```

The `context` parameter is auto-injected by the agent and hidden from the LLM's tool schema.

### Streaming tool (partial results)

```python
from collections.abc import Generator

def stream_data(query: str) -> Generator[str, None, None]:
    """Stream results progressively.

    Args:
        query: What to fetch.

    Yields:
        Data chunks as they become available.
    """
    for chunk in fetch_chunks(query):
        yield chunk
```

### Tool with hidden metadata

```python
from ragbits.agents.tool import ToolReturn

def my_tool(query: str) -> ToolReturn:
    """Tool that returns metadata hidden from the LLM.

    Args:
        query: The query.
    """
    return ToolReturn(
        value="visible to LLM",
        metadata={"internal_id": "abc123"},  # hidden from LLM, available via result.tool_calls[*].metadata
    )
```

## Multi-turn Conversation History

```python
agent = Agent(llm=llm, prompt=MyPrompt, keep_history=True)

# Each run() call continues the same conversation
result1 = await agent.run(MyInput(message="My name is Alice."))
result2 = await agent.run(MyInput(message="What is my name?"))  # remembers "Alice"
```

## A2A Quick Reference

For the full A2A pattern, read `references/checklists/new-a2a.md`. Three facts people miss:

1. `get_agent_card` is a **method on `Agent`**, not a free function. It's async.
2. `create_agent_server` already wires host/port from the card — you call `.serve()` on the returned `uvicorn.Server` directly. No extra `uvicorn.run(...)` call.
3. The `create_agent_server` signature is `(agent, agent_card, input_model)` — the input model class is required so the server can validate incoming request params.

```python
from ragbits.agents.a2a.server import create_agent_server


async def main() -> None:
    agent_card = await agent.get_agent_card(
        name="MyAgent",
        description="What this agent does.",
        port=8000,
    )
    server = create_agent_server(agent, agent_card, MyInput)
    await server.serve()
```

### Orchestrator pattern (calling remote A2A agents)

The canonical heavy pattern (see `examples/agents/a2a/agent_orchestrator.py` in the ragbits repo) subclasses `Agent` with routing and summarization prompts. For lightweight coordinators that just register a handful of endpoints and dispatch by name, the registry pattern below works as a pair of tools on a regular `Agent`:

```python
import json
from typing import Any

import requests


class RemoteAgentRegistry:
    """Discover and call A2A agents at known endpoints."""

    def __init__(self) -> None:
        self._agents: dict[str, dict[str, Any]] = {}

    def discover(self, endpoints: list[tuple[str, int]], timeout: float = 10.0) -> str:
        """Fetch agent cards for the given (host, port) pairs."""
        for host, port in endpoints:
            url = f"http://{host}:{port}"
            card = requests.get(f"{url}/.well-known/agent.json", timeout=timeout).json()
            self._agents[card["name"]] = card
        return "\n".join(f"{name}: {c['description']}" for name, c in self._agents.items())

    def call(self, agent_name: str, params: dict[str, Any], timeout: float = 60.0) -> dict[str, Any]:
        """Send a request to a discovered agent. `params` must match its input model."""
        if agent_name not in self._agents:
            return {"error": f"Unknown agent '{agent_name}'"}
        resp = requests.post(self._agents[agent_name]["url"], json={"params": params}, timeout=timeout)
        resp.raise_for_status()
        return resp.json()["result"]


registry = RemoteAgentRegistry()


def discover_agents(host_port_pairs: str) -> str:
    """Register remote A2A agents at the given endpoints.

    Args:
        host_port_pairs: Comma-separated "host:port" list, e.g. "localhost:8000,localhost:8001".
    """
    pairs = [(p.split(":")[0], int(p.split(":")[1])) for p in host_port_pairs.split(",")]
    return registry.discover(pairs)


def execute_agent(agent_name: str, query: str) -> str:
    """Call a registered remote agent by name and return its response.

    Args:
        agent_name: Name as returned by discover_agents.
        query: Task to delegate. Sent as `{"input": query}` by convention — adjust the payload
            below if the target agent's input model uses a different field name.
    """
    result = registry.call(agent_name, params={"input": query})
    return json.dumps(result)
```

The `params={"input": query}` convention matches the A2A examples in the ragbits repo (hotel/flight/city-explorer all define an input model with an `input` field). If the remote agent exposes a different field, change the payload accordingly.
