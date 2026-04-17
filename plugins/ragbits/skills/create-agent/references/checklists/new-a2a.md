# New A2A Server Checklist

Quality gates for exposing a ragbits agent as an A2A (Agent-to-Agent) server.

## Definition Requirements

- [ ] Agent has a Pydantic `BaseModel` input schema (used to validate incoming A2A requests)
- [ ] Agent has a prompt, LLM, and optionally tools configured
- [ ] `get_agent_card()` is called with a descriptive `name` and `description`
- [ ] Port is unique and not conflicting with other agents in the system

## Imports

```python
from pydantic import BaseModel
from ragbits.agents import Agent
from ragbits.agents.a2a.server import create_agent_server
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt
```

## A2A Server Pattern

### 1. Define the input schema

The input model is used to validate incoming A2A requests:

```python
class MyAgentInput(BaseModel):
    """Input schema for the A2A agent."""
    input: str
```

### 2. Define the prompt

```python
class MyAgentPrompt(Prompt[MyAgentInput]):
    system_prompt = """
    You are a specialized assistant.
    """

    user_prompt = """
    {{ input }}
    """
```

### 3. Create the agent

```python
llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)
agent = Agent(llm=llm, prompt=MyAgentPrompt, tools=[my_tool])
```

### 4. Create AgentCard and start server

```python
async def main() -> None:
    agent_card = await agent.get_agent_card(
        name="My Agent",
        description="Description of what the agent does.",
        port=8000,
    )
    server = create_agent_server(agent, agent_card, MyAgentInput)
    await server.serve()


if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

## `get_agent_card()` API

```python
async def get_agent_card(
    self,
    name: str,                                        # Human-readable agent name
    description: str,                                  # What the agent does
    version: str = "0.0.0",                           # Agent version
    host: str = "127.0.0.1",                          # Hostname/IP
    port: int = 8000,                                  # Port number
    protocol: str = "http",                            # URL scheme
    default_input_modes: list[str] | None = None,     # Input content types (default: ["text"])
    default_output_modes: list[str] | None = None,    # Output content types (default: ["text"])
    capabilities: "AgentCapabilities | None" = None,  # Agent capabilities
    skills: list["AgentSkill"] | None = None,         # Skills list (auto-extracted from tools if None)
) -> "AgentCard"
```

Returns an `AgentCard` with URL constructed as `f"{protocol}://{host}:{port}"`.

When `skills` is `None`, skills are automatically extracted from the agent's registered tools — each tool becomes an `AgentSkill` with its name, description, and parameters.

## `create_agent_server()` API

```python
def create_agent_server(
    agent: Agent,
    agent_card: "AgentCard",
    input_model: type[BaseModel],
) -> "uvicorn.Server"
```

Creates a Uvicorn HTTP server that exposes two endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/.well-known/agent.json` | GET | Returns the AgentCard (agent discovery) |
| `/` | POST | Handles agent requests |

### Request format

```json
{
  "params": {
    "input": "user query here"
  }
}
```

The `params` dict is validated against the `input_model` Pydantic class.

### Response format

```json
{
  "result": {
    "content": "agent response text",
    "metadata": {},
    "history": [...],
    "tool_calls": [
      {"name": "tool_name", "arguments": {...}, "result": "..."}
    ]
  },
  "error": null
}
```

On error:
```json
{
  "result": null,
  "error": "error description"
}
```

## Remote Agent Discovery & Communication

From an orchestrator or client, discover and call A2A agents:

### Discovery

```python
import requests

url = f"http://{host}:{port}"
agent_card = requests.get(f"{url}/.well-known/agent.json", timeout=10).json()
# agent_card contains: name, description, url, skills, capabilities, etc.
```

### Invocation

```python
payload = {"params": {"input": "user query"}}
response = requests.post(agent_card["url"], json=payload, timeout=60)
result = response.json()

content = result["result"]["content"]
tool_calls = result["result"].get("tool_calls", [])
```

## Validation Checklist

- [ ] Input model is a Pydantic `BaseModel` with fields matching what remote callers will send
- [ ] `name` in `get_agent_card()` is descriptive and unique across the system
- [ ] `description` clearly states what the agent does (used for agent discovery)
- [ ] `port` does not conflict with other agents (common pattern: 8000, 8001, 8002, ...)
- [ ] Server is started with `await server.serve()` inside an async main
- [ ] Agent has all tools and prompt configured before creating the server
- [ ] The `/.well-known/agent.json` endpoint is accessible for agent discovery
