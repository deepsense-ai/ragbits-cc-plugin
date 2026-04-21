# New Agents Checklist

Quality gates for adding a tool-using `Agent` to the generated chat application.

The `agents` feature turns the chat from a plain LLM stream into an agent that can call tools, surface tool progress in the UI via live updates, and still stream its final reply. Use this when the user wants the bot to **do** things (fetch data, call APIs, execute actions), not only reply.

## Dependency

Agents require `ragbits-agents`. Append it to `pyproject.toml`:

```toml
dependencies = [
    "ragbits-chat",
    "ragbits-agents",
]
```

## Imports

```python
from ragbits.agents import Agent, ToolCall, ToolCallResult
from ragbits.chat.interface.types import LiveUpdateType
```

Add a system prompt via a `Prompt` subclass when the behavior is domain-specific — see `plugins/ragbits/skills/create-agent/references/agent-spec.md` for the full Agent spec.

## Tools Module

Put tool functions in a sibling `tools.py` and re-export a `TOOLS` list:

```python
"""Tools for the {Name} chat."""


def example_tool(query: str) -> str:
    """
    Short description — the LLM reads this to decide when to call the tool.

    Args:
        query: The input or search query.

    Returns:
        A result string.
    """
    # TODO: implement
    return f"Result for: {query}"


TOOLS = [example_tool]
```

Tool quality matters — the LLM calls tools based on their docstrings and parameter names. See `plugins/ragbits/skills/create-agent/references/checklists/new-tool.md` for the full tool-writing checklist (type hints, docstring format, return-type rules).

## Constructor

Instantiate the `Agent` in `__init__` alongside (or replacing) the plain LLM:

```python
from {app_name_snake}.tools import TOOLS


class Chat(ChatInterface):
    conversation_history = True

    def __init__(self) -> None:
        self.llm = LiteLLM(model_name="{model}", use_structured_output=True)
        self.agent = Agent(llm=self.llm, tools=TOOLS)
```

`use_structured_output=True` makes function-calling reliable across OpenAI-compatible models. Drop it only when targeting a model that doesn't support it.

## Streaming Pattern

Use `agent.run_streaming()` and pattern-match the yielded events. Tool calls become live updates so the UI shows the bot's reasoning in real time:

```python
async def chat(
    self,
    message: str,
    history: ChatFormat,
    context: ChatContext,
) -> AsyncGenerator[ChatResponse, None]:
    stream = self.agent.run_streaming(message)
    async for event in stream:
        match event:
            case str() as chunk if chunk.strip():
                yield self.create_text_response(chunk)
            case ToolCall():
                yield self.create_live_update(
                    event.id,
                    LiveUpdateType.START,
                    f"Using {event.name}",
                    "Processing...",
                )
            case ToolCallResult():
                yield self.create_live_update(
                    event.id,
                    LiveUpdateType.FINISH,
                    f"{event.name} completed",
                )
```

Keep the `id` consistent between the `START` and `FINISH` events so the UI pairs them. Other event types (`Usage`, `BasePrompt`, `ConfirmationRequest`) can be ignored unless the feature calls for them — see `references/chat-spec.md` and the agent spec.

## Agent with Structured Input

When the agent needs a Pydantic input model rather than a raw string, build it from `message` and pass to `run_streaming`:

```python
from pydantic import BaseModel


class ChatInput(BaseModel):
    message: str


stream = self.agent.run_streaming(ChatInput(message=message))
```

This pairs with a `Prompt[ChatInput]` subclass — see the agent spec for the full pattern.

## Conversation History

`Agent` doesn't natively replay `history` — you have three options:

1. **Pass `keep_history=True`** to `Agent(...)` and call `.run_streaming()` without manually rebuilding context. The agent remembers across calls **within the same `Chat` instance**, which works because `RagbitsAPI` reuses one instance across a session.
2. **Rebuild messages manually** and drop the agent abstraction — use `self.llm.generate_streaming([*history, ...])` for plain LLM streaming.
3. **Inject history into a `Prompt`** subclass's `conversation_history` parameter when instantiating it per-turn.

Option 1 is the most ergonomic; options 2 and 3 give finer control.

## When Not to Use an Agent

- The chat doesn't need to call any external actions — plain LLM streaming (the base template) is simpler.
- Every turn forces a specific tool regardless of content — code the tool call directly in `chat()` and skip the agent overhead.
- Latency is critical and the tool set has 1–2 entries — a hand-rolled tool call may be faster than the agent loop.

## Validation Checklist

- [ ] `pyproject.toml` lists `ragbits-agents`
- [ ] Tools live in a separate `tools.py` and export a `TOOLS` list — not inline in `app.py`
- [ ] `self.agent` is instantiated in `__init__`, not per-turn
- [ ] `chat()` uses `agent.run_streaming(...)` and pattern-matches the events
- [ ] `ToolCall`/`ToolCallResult` events surface as paired `LiveUpdateType.START`/`FINISH` with the same `id`
- [ ] The LLM uses `use_structured_output=True` when the target model supports it
- [ ] The system prompt is domain-specific, not the default
