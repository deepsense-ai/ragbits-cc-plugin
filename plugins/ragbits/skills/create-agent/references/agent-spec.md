# Ragbits Agent Specification

Advanced patterns for the ragbits `Agent` class. Read this when the base template in SKILL.md isn't enough.

## Core Imports

```python
from ragbits.agents import Agent, ToolCallResult
from ragbits.core.llms import LiteLLM, ToolCall
from ragbits.core.prompt import Prompt
from ragbits.agents.types import AgentRunContext
```

## Agent Constructor

```python
agent = Agent(
    llm=LiteLLM(model_name="gpt-4.1"),
    prompt=MyPrompt,
    tools=[my_tool, another_tool],   # optional
    hooks=[my_hook],                  # optional — see new-hook.md
    mcp_servers=[mcp_server],         # optional — see new-mcp.md
    keep_history=True,                # multi-turn memory across run() calls
)
```

## Running the Agent

### Basic (returns full result as string)

```python
result: str = await agent.run(MyInput(message="Hello"))
```

### Streaming (token-by-token with tool call visibility)

```python
async for chunk in agent.run_streaming(MyInput(message="Hello")):
    match chunk:
        case ToolCall():
            print(f"[Calling {chunk.name}: {chunk.arguments}]")
        case ToolCallResult():
            print(f"[Result from {chunk.name}: {chunk.result}]")
        case _:
            print(chunk, end="", flush=True)
print()
```

## Prompt Patterns

### Basic (string output)

```python
class MyInput(BaseModel):
    message: str

class MyPrompt(Prompt[MyInput]):
    system_prompt = "You are a helpful assistant."
    user_prompt = "{{ message }}"
```

### Structured output (Pydantic model)

```python
class MyOutput(BaseModel):
    answer: str
    confidence: float

class MyPrompt(Prompt[MyInput, MyOutput]):
    system_prompt = "Extract structured data."
    user_prompt = "{{ message }}"
```

### Without input

```python
class StaticPrompt(Prompt):
    system_prompt = "You are a code reviewer."
    user_prompt = "Review the provided diff."
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

### Basic tool

```python
def search_docs(query: str) -> str:
    """Search documentation for the given query.

    Args:
        query: The search term.

    Returns:
        Matching documentation excerpts.
    """
    # implementation
    return results
```

### Tool with agent run context

```python
from ragbits.agents.types import AgentRunContext

def audited_action(input_data: str, context: AgentRunContext) -> str:
    """Perform an action and log it.

    Args:
        input_data: The data to process.
        context: Injected automatically — do not pass manually.

    Returns:
        Result of the action.
    """
    print(f"[run_id={context.run_id}] Processing: {input_data}")
    return process(input_data)
```

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

## Multi-turn Conversation History

```python
agent = Agent(llm=llm, prompt=MyPrompt, keep_history=True)

# Each run() call continues the same conversation
result1 = await agent.run(MyInput(message="My name is Alice."))
result2 = await agent.run(MyInput(message="What is my name?"))  # remembers "Alice"
```

## A2A Quick Reference

For the full A2A pattern, read `references/checklists/new-a2a.md`. Quick reference:

```python
from ragbits.agents.a2a import create_agent_server, get_agent_card

# Expose an agent as an A2A server
server = create_agent_server(
    agent=agent,
    agent_card=get_agent_card(
        name="MyAgent",
        description="What this agent does.",
        url="http://localhost:8000",
        skills=[{"id": "main", "name": "Main skill", "description": "..."}],
    ),
)

import uvicorn
uvicorn.run(server, host="0.0.0.0", port=8000)
```

### Remote agent as tool (orchestrator pattern)

```python
import requests, json

AGENTS: dict[str, dict] = {}  # name → agent card

def discover_agents(endpoints: list[tuple[str, int]]) -> str:
    for host, port in endpoints:
        card = requests.get(f"http://{host}:{port}/.well-known/agent.json").json()
        AGENTS[card["name"]] = card
    return "\n".join(f"{n}: {c['description']}" for n, c in AGENTS.items())

def execute_agent(agent_name: str, query: str) -> str:
    """Call a remote A2A agent by name.

    Args:
        agent_name: Name of the agent (from discovery).
        query: The task to delegate.

    Returns:
        JSON string with the agent's response.
    """
    if agent_name not in AGENTS:
        return json.dumps({"error": f"Unknown agent '{agent_name}'"})
    resp = requests.post(AGENTS[agent_name]["url"], json={"params": {"input": query}}, timeout=60)
    return json.dumps(resp.json()["result"])
```
