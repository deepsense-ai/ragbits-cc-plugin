# New Tool Checklist

Quality gates for adding a new tool to a ragbits agent.

## Definition Requirements

- [ ] Tool is a plain Python function with full type hints on all parameters and return type
- [ ] Google-style docstring with `Args:` section describing each parameter
- [ ] Function name is descriptive and uses snake_case (becomes the tool name the LLM sees)
- [ ] Return type is `str` (or JSON-serializable value that will be converted to string for the LLM)

## Imports

```python
# No special imports needed for basic tools — just standard library or project deps.
# The Agent converts plain functions to Tool objects automatically.

# Optional: for tools that need agent context (run_id, deps, downstream agents)
from ragbits.agents import AgentRunContext

# Optional: for streaming tools and tools with hidden metadata
from ragbits.agents.tool import ToolEvent, ToolReturn
```

`ToolReturn` is a dataclass — both `ToolReturn(value)` (positional) and `ToolReturn(value=..., metadata=...)` work.

## Tool Function Pattern

```python
def my_tool(param1: str, param2: int) -> str:
    """
    Short description of what the tool does.

    Args:
        param1: Description of param1.
        param2: Description of param2.

    Returns:
        Description of the return value.
    """
    # Tool implementation
    return json.dumps({"result": "value"})
```

## Registration

Tools are passed to the Agent constructor as a list of callables:

```python
from ragbits.agents import Agent

agent = Agent(
    llm=llm,
    prompt=MyPrompt,
    tools=[my_tool, another_tool],  # plain functions, Tool objects, or nested Agents
)
```

The Agent normalizes all entries:
- `Callable` → converted via `Tool.from_callable()`
- `Tool` instance → used directly
- `Agent` instance → converted via `Tool.from_agent()` (nested agent becomes a callable tool)

## Advanced Patterns

### Tool with AgentRunContext

A tool can optionally accept `AgentRunContext` to access agent state. The context parameter is automatically injected and hidden from the LLM schema:

```python
from ragbits.agents import AgentRunContext

def my_tool(query: str, context: AgentRunContext | None = None) -> str:
    """
    Tool that accesses agent context.

    Args:
        query: The search query.
        context: Agent execution context (injected automatically).
    """
    # context provides access to downstream agent info, dependencies, etc.
    return "result"
```

### Streaming Tool (Generator)

A tool can yield `ToolEvent` objects for streaming partial results:

```python
from ragbits.agents.tool import ToolEvent
from collections.abc import Generator

def streaming_tool(query: str) -> Generator[ToolEvent, None, None]:
    """
    Tool that streams partial results.

    Args:
        query: The search query.
    """
    yield ToolEvent(content="Processing...")
    yield ToolEvent(content="Done: result here")
```

### Tool with Hidden Metadata

Use `ToolReturn` to attach metadata that the LLM cannot see:

```python
from ragbits.agents.tool import ToolReturn

def my_tool(query: str) -> ToolReturn:
    """
    Tool that returns metadata hidden from LLM.

    Args:
        query: The query.
    """
    return ToolReturn(
        value="visible to LLM",
        metadata={"internal_id": "abc123"},  # hidden from LLM
    )
```

### Nested Agent as Tool

A downstream Agent can be used as a tool by another agent:

```python
from ragbits.agents import Agent

specialist_agent = Agent(llm=llm, prompt=SpecialistPrompt, tools=[...])

orchestrator = Agent(
    llm=llm,
    prompt=OrchestratorPrompt,
    tools=[specialist_agent],  # automatically wrapped via Tool.from_agent()
)
```

## Runtime Tool Control

Use `tool_choice` when running the agent:

```python
result = await agent.run(input_data, tool_choice="auto")      # LLM decides (default)
result = await agent.run(input_data, tool_choice="none")       # no tools allowed
result = await agent.run(input_data, tool_choice="required")   # must use a tool
result = await agent.run(input_data, tool_choice=my_tool)      # force specific tool
```

## Validation Checklist

- [ ] All parameters have type annotations
- [ ] Docstring includes `Args:` with descriptions for every parameter
- [ ] Return value is JSON-serializable (prefer `str` with `json.dumps()`)
- [ ] Tool name (function name) is unique across all tools and MCP servers registered on the agent
- [ ] No side effects that could be dangerous without user confirmation (use hooks for that)
- [ ] Tool is registered in the Agent's `tools=[]` list
