---
name: add-agent-hooks
description: Add ragbits agent hooks to an existing project. Use when the user wants to add lifecycle hooks, tool validation, confirmation dialogs, input guardrails, output logging, data masking, or streaming event transforms to their ragbits agent.
disable-model-invocation: true
argument-hint: "[hook-type] [--tools TOOLS]"
allowed-tools: Bash(mkdir *) Bash(pip *) Bash(uv *) Write Edit Read Glob Grep
---

# Add Ragbits Agent Hooks

Add lifecycle hooks to an existing ragbits agent project. Hooks let you intercept and modify agent behavior at key points: before/after tool calls, before/after agent runs, and during streaming events.

## Arguments

Parse `$ARGUMENTS` to extract:
- **hook-type** (first positional arg, required) — one of: `confirmation`, `validation`, `logging`, `guardrails`, `streaming-transform`, `custom`
- **--tools TOOLS** (optional, comma-separated) — tool names to target (default: all tools)

If `$ARGUMENTS` is empty, ask the user which hook type they want and list the options:
1. **confirmation** — Require user confirmation before specific tools execute (e.g., delete, send, write operations)
2. **validation** — Validate and sanitize tool inputs before execution (e.g., email format, allowed domains, path safety)
3. **logging** — Log tool inputs/outputs and agent results for observability and debugging
4. **guardrails** — Block or rewrite agent input based on content policies (blocked topics, PII detection)
5. **streaming-transform** — Transform streaming output in real-time (e.g., redact words, format text, inject metadata)
6. **custom** — Generate a blank hook scaffold for all five event types so the user can fill in their logic

## Hook System Overview

The ragbits hooks system provides 5 event types that fire at different points in the agent lifecycle:

| Event Type | When it fires | Callback signature | What it can do |
|---|---|---|---|
| `PRE_TOOL` | Before a tool is invoked | `async (ToolCall) -> ToolCall` | Validate/modify arguments, deny or request confirmation |
| `POST_TOOL` | After a tool completes | `async (ToolCall, ToolReturn) -> ToolReturn` | Modify/mask output, add logging metadata |
| `PRE_RUN` | Before the agent run starts | `async (input, AgentOptions, AgentRunContext) -> input` | Validate/transform input, apply guardrails |
| `POST_RUN` | After the agent run completes | `async (AgentResult, AgentOptions, AgentRunContext) -> AgentResult` | Modify final result, add metadata, log |
| `ON_EVENT` | For each streaming event | `async (StreamingEvent) -> StreamingEvent \| None` | Transform/filter/suppress streaming chunks |

### Key concepts

- **Priority**: Lower numbers execute first (default: 100). Use this to order hooks (e.g., validation at 10, sanitization at 20).
- **Tool filtering**: Hooks can target specific tools via `tool_names` or apply to all tools (when `tool_names=None`).
- **Chaining**: Hooks are chained — each hook receives the output of the previous one.
- **Decision control** (PRE_TOOL only): A hook can set `decision` to:
  - `"pass"` — allow execution (default)
  - `"deny"` — block execution (requires `reason`)
  - `"ask"` — pause and request user confirmation (requires `reason`)

## Step 1: Detect Project Structure

Read the project directory to find:
1. The main agent file (look for `from ragbits.agents import Agent` or `Agent(`)
2. Existing tools defined or imported in the agent file
3. Whether the project already has hooks registered

If no ragbits agent is found, inform the user and ask them to create one first (e.g., with `/create-agent`).

## Step 2: Generate Hook Code

Based on the selected hook type, generate the appropriate hook code. Create a new file `hooks.py` in the same directory as the agent file (or update it if it already exists).

### Template: confirmation

Creates a confirmation hook that requires user approval for specified tools.

```python
"""
Agent hooks: tool confirmation.

Requires user confirmation before executing destructive or sensitive tools.
"""

from ragbits.agents.hooks import Hook, EventType, create_confirmation_hook


def get_confirmation_hooks(tool_names: list[str] | None = None) -> list[Hook]:
    """
    Create hooks that require user confirmation for specified tools.

    Args:
        tool_names: List of tool names requiring confirmation. None = all tools.

    Returns:
        List of hooks to pass to the Agent constructor.
    """
    return [
        create_confirmation_hook(tool_names=tool_names, priority=1),
    ]
```

If the user specified `--tools`, pre-fill the tool names list. Otherwise use `None` (all tools) and add a comment suggesting which tools to target.

### Template: validation

Creates PRE_TOOL hooks for input validation and sanitization.

```python
"""
Agent hooks: input validation and sanitization.

Validates tool arguments before execution and sanitizes inputs to enforce business rules.
"""

import re

from ragbits.agents.hooks import EventType, Hook
from ragbits.core.llms.base import ToolCall


async def validate_arguments(tool_call: ToolCall) -> ToolCall:
    """
    Validate tool arguments before execution.

    Checks for empty required fields and argument format.
    Customize the validation rules below for your specific tools.

    Args:
        tool_call: The tool call to validate.

    Returns:
        The tool call, potentially with decision set to "deny" if validation fails.
    """
    args = tool_call.arguments

    # Example: Block empty string arguments
    for key, value in args.items():
        if isinstance(value, str) and not value.strip():
            return tool_call.model_copy(
                update={
                    "decision": "deny",
                    "reason": f"Argument '{key}' cannot be empty.",
                }
            )

    # Example: Validate email format (customize for your tools)
    if "email" in args:
        email_pattern = r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
        if not re.match(email_pattern, args["email"]):
            return tool_call.model_copy(
                update={
                    "decision": "deny",
                    "reason": f"Invalid email format: {args['email']}",
                }
            )

    # Example: Validate URL format
    if "url" in args:
        url_pattern = r"^https?://.+"
        if not re.match(url_pattern, args["url"]):
            return tool_call.model_copy(
                update={
                    "decision": "deny",
                    "reason": f"Invalid URL format: {args['url']}. Must start with http:// or https://",
                }
            )

    return tool_call


async def sanitize_arguments(tool_call: ToolCall) -> ToolCall:
    """
    Sanitize tool arguments before execution.

    Cleans and normalizes argument values to enforce business rules.
    Customize the sanitization rules below for your specific tools.

    Args:
        tool_call: The tool call to sanitize.

    Returns:
        The tool call with sanitized arguments.
    """
    modified_args = tool_call.arguments.copy()

    # Example: Strip whitespace from all string arguments
    for key, value in modified_args.items():
        if isinstance(value, str):
            modified_args[key] = value.strip()

    # Example: Enforce allowed domains for email arguments
    if "email" in modified_args:
        allowed_domains = ["example.com", "company.com"]  # Customize this list
        email = modified_args["email"]
        domain = email.split("@")[-1] if "@" in email else ""
        if domain not in allowed_domains:
            modified_args["email"] = email.split("@")[0] + "@example.com"

    if modified_args != tool_call.arguments:
        return tool_call.model_copy(update={"arguments": modified_args})

    return tool_call


def get_validation_hooks(tool_names: list[str] | None = None) -> list[Hook]:
    """
    Create hooks for input validation and sanitization.

    Args:
        tool_names: List of tool names to validate. None = all tools.

    Returns:
        List of hooks to pass to the Agent constructor.
    """
    return [
        Hook(
            event_type=EventType.PRE_TOOL,
            callback=validate_arguments,
            tool_names=tool_names,
            priority=10,  # Validate first
        ),
        Hook(
            event_type=EventType.PRE_TOOL,
            callback=sanitize_arguments,
            tool_names=tool_names,
            priority=20,  # Sanitize after validation passes
        ),
    ]
```

### Template: logging

Creates POST_TOOL and POST_RUN hooks for observability.

```python
"""
Agent hooks: output logging.

Logs tool inputs/outputs and agent run results for observability and debugging.
"""

import json
import logging
from typing import Any

from ragbits.agents._main import AgentOptions, AgentResult, AgentRunContext
from ragbits.agents.hooks import EventType, Hook
from ragbits.agents.tool import ToolReturn
from ragbits.core.llms.base import ToolCall

logger = logging.getLogger(__name__)


async def log_tool_call(tool_call: ToolCall, tool_return: ToolReturn) -> ToolReturn:
    """
    Log tool inputs and outputs after execution.

    Args:
        tool_call: The tool call that was executed.
        tool_return: The result from the tool.

    Returns:
        The unmodified tool return (logging is side-effect only).
    """
    logger.info(
        "Tool executed: %s | Args: %s | Result: %s",
        tool_call.name,
        json.dumps(tool_call.arguments, default=str),
        str(tool_return.value)[:200],
    )
    return tool_return


async def log_agent_result(
    result: AgentResult,
    options: AgentOptions,
    context: AgentRunContext,
) -> AgentResult:
    """
    Log the final agent result after a run completes.

    Args:
        result: The agent result.
        options: The agent options used for this run.
        context: The agent run context.

    Returns:
        The unmodified agent result (logging is side-effect only).
    """
    tool_names = [tc.name for tc in result.tool_calls] if result.tool_calls else []
    logger.info(
        "Agent run completed | Tools used: %s | Usage: prompt=%d, completion=%d | Content preview: %s",
        tool_names or "none",
        result.usage.input_tokens or 0,
        result.usage.output_tokens or 0,
        str(result.content)[:200],
    )
    return result


async def mask_sensitive_fields(tool_call: ToolCall, tool_return: ToolReturn) -> ToolReturn:
    """
    Mask sensitive fields in tool output before they reach the LLM.

    Customize the SENSITIVE_FIELDS set below for your data model.

    Args:
        tool_call: The tool call that was executed.
        tool_return: The result from the tool.

    Returns:
        The tool return with sensitive fields masked.
    """
    SENSITIVE_FIELDS = {"ssn", "password", "secret", "token", "api_key", "credit_card"}  # Customize this

    def mask_dict(data: dict) -> dict:
        masked = {}
        for key, value in data.items():
            if key.lower() in SENSITIVE_FIELDS:
                masked[key] = "***REDACTED***"
            elif isinstance(value, dict):
                masked[key] = mask_dict(value)
            else:
                masked[key] = value
        return masked

    if isinstance(tool_return.value, dict):
        return ToolReturn(mask_dict(tool_return.value), metadata=tool_return.metadata)

    return tool_return


def get_logging_hooks(
    tool_names: list[str] | None = None,
    mask_sensitive: bool = True,
) -> list[Hook]:
    """
    Create hooks for logging and optional sensitive data masking.

    Args:
        tool_names: List of tool names to log. None = all tools.
        mask_sensitive: Whether to mask sensitive fields in tool output.

    Returns:
        List of hooks to pass to the Agent constructor.
    """
    hooks = [
        Hook(
            event_type=EventType.POST_TOOL,
            callback=log_tool_call,
            tool_names=tool_names,
            priority=90,  # Log after other post-tool hooks
        ),
        Hook(
            event_type=EventType.POST_RUN,
            callback=log_agent_result,
            priority=90,
        ),
    ]

    if mask_sensitive:
        hooks.append(
            Hook(
                event_type=EventType.POST_TOOL,
                callback=mask_sensitive_fields,
                tool_names=tool_names,
                priority=10,  # Mask before logging
            )
        )

    return hooks
```

### Template: guardrails

Creates PRE_RUN hooks for content-based input filtering.

```python
"""
Agent hooks: input guardrails.

Blocks or rewrites agent input based on content policies before the agent processes it.
"""

from ragbits.agents._main import AgentOptions, AgentRunContext
from ragbits.agents.hooks import EventType, Hook


BLOCKED_TOPICS: list[str] = [
    # Add topics that should be blocked. Example:
    # "politics",
    # "religion",
    # "violence",
]

MAX_INPUT_LENGTH: int = 5000  # Maximum allowed input length in characters


async def block_topics(
    input: str | None,
    options: AgentOptions,
    context: AgentRunContext,
) -> str | None:
    """
    Block requests containing forbidden topics.

    Customize the BLOCKED_TOPICS list above for your use case.

    Args:
        input: The user input.
        options: The agent options.
        context: The agent run context.

    Returns:
        The original input, or a rejection message if a blocked topic is detected.
    """
    if input is None:
        return input

    input_lower = str(input).lower()
    for topic in BLOCKED_TOPICS:
        if topic in input_lower:
            return f"I cannot help with that request. The topic '{topic}' is not allowed by our content policy."

    return input


async def enforce_length_limit(
    input: str | None,
    options: AgentOptions,
    context: AgentRunContext,
) -> str | None:
    """
    Enforce maximum input length to prevent abuse.

    Args:
        input: The user input.
        options: The agent options.
        context: The agent run context.

    Returns:
        The original input, or a rejection message if too long.
    """
    if input is None:
        return input

    if len(str(input)) > MAX_INPUT_LENGTH:
        return (
            f"Your input is too long ({len(str(input))} characters). "
            f"Please keep it under {MAX_INPUT_LENGTH} characters."
        )

    return input


async def detect_prompt_injection(
    input: str | None,
    options: AgentOptions,
    context: AgentRunContext,
) -> str | None:
    """
    Basic prompt injection detection.

    Checks for common prompt injection patterns. For production use,
    consider integrating a dedicated guardrails library (e.g., ragbits-guardrails).

    Args:
        input: The user input.
        options: The agent options.
        context: The agent run context.

    Returns:
        The original input, or a rejection message if injection is detected.
    """
    if input is None:
        return input

    injection_patterns = [
        "ignore previous instructions",
        "ignore all previous",
        "disregard your instructions",
        "forget your instructions",
        "you are now",
        "new instructions:",
        "system prompt:",
    ]

    input_lower = str(input).lower()
    for pattern in injection_patterns:
        if pattern in input_lower:
            return "I cannot process this request. It appears to contain instructions that conflict with my guidelines."

    return input


def get_guardrail_hooks() -> list[Hook]:
    """
    Create hooks for input guardrails.

    Returns:
        List of hooks to pass to the Agent constructor.
    """
    return [
        Hook(
            event_type=EventType.PRE_RUN,
            callback=detect_prompt_injection,
            priority=1,  # Check injection first
        ),
        Hook(
            event_type=EventType.PRE_RUN,
            callback=block_topics,
            priority=10,
        ),
        Hook(
            event_type=EventType.PRE_RUN,
            callback=enforce_length_limit,
            priority=20,
        ),
    ]
```

### Template: streaming-transform

Creates ON_EVENT hooks for real-time streaming output transformation.

```python
"""
Agent hooks: streaming event transforms.

Transforms streaming output in real-time as the agent generates responses.
"""

from ragbits.agents.hooks import EventType, Hook
from ragbits.agents.hooks.types import OnEventCallback, StreamingEvent


def create_word_replace_hook(replacements: dict[str, str]) -> OnEventCallback:
    """
    Create an ON_EVENT hook that replaces words in streaming text chunks.

    Args:
        replacements: Dictionary mapping original words to replacements.

    Returns:
        An async callback for the ON_EVENT hook.

    Example:
        replacements = {"password": "***", "secret": "***"}
    """

    async def replace_words(event: StreamingEvent) -> StreamingEvent | None:
        if isinstance(event, str):
            result = event
            for original, replacement in replacements.items():
                result = result.replace(original, replacement)
            return result
        return event

    return replace_words


def create_upper_case_hook(words: list[str]) -> OnEventCallback:
    """
    Create an ON_EVENT hook that upper-cases specified words in streaming text.

    Args:
        words: List of words to upper-case.

    Returns:
        An async callback for the ON_EVENT hook.
    """

    async def upper_words(event: StreamingEvent) -> StreamingEvent | None:
        if isinstance(event, str):
            result = event
            for word in words:
                result = result.replace(word, word.upper())
            return result
        return event

    return upper_words


def create_text_filter_hook(blocked_phrases: list[str], replacement: str = "[REDACTED]") -> OnEventCallback:
    """
    Create an ON_EVENT hook that redacts blocked phrases from streaming text.

    Args:
        blocked_phrases: List of phrases to redact.
        replacement: Text to replace blocked phrases with.

    Returns:
        An async callback for the ON_EVENT hook.
    """

    async def filter_text(event: StreamingEvent) -> StreamingEvent | None:
        if isinstance(event, str):
            result = event
            for phrase in blocked_phrases:
                if phrase.lower() in result.lower():
                    # Case-insensitive replacement
                    import re
                    result = re.sub(re.escape(phrase), replacement, result, flags=re.IGNORECASE)
            return result
        return event

    return filter_text


def get_streaming_hooks(
    replacements: dict[str, str] | None = None,
    upper_words: list[str] | None = None,
    blocked_phrases: list[str] | None = None,
) -> list[Hook]:
    """
    Create hooks for streaming event transformation.

    Args:
        replacements: Word replacements dict (e.g., {"password": "***"}).
        upper_words: Words to upper-case in output.
        blocked_phrases: Phrases to redact from output.

    Returns:
        List of hooks to pass to the Agent constructor.
    """
    hooks: list[Hook] = []

    if replacements:
        hooks.append(
            Hook(
                event_type=EventType.ON_EVENT,
                callback=create_word_replace_hook(replacements),
                priority=10,
            )
        )

    if upper_words:
        hooks.append(
            Hook(
                event_type=EventType.ON_EVENT,
                callback=create_upper_case_hook(upper_words),
                priority=20,
            )
        )

    if blocked_phrases:
        hooks.append(
            Hook(
                event_type=EventType.ON_EVENT,
                callback=create_text_filter_hook(blocked_phrases),
                priority=5,  # Redact before other transforms
            )
        )

    return hooks
```

### Template: custom

Creates a scaffold file with example hooks for all 5 event types.

```python
"""
Agent hooks: custom hooks scaffold.

This file provides a starting point for implementing custom hooks.
Each event type has an example callback you can customize.
"""

from typing import Any

from ragbits.agents._main import AgentOptions, AgentResult, AgentRunContext
from ragbits.agents.hooks import EventType, Hook
from ragbits.agents.hooks.types import StreamingEvent
from ragbits.agents.tool import ToolReturn
from ragbits.core.llms.base import ToolCall


# --- PRE_TOOL: Runs before a tool is invoked ---


async def pre_tool_hook(tool_call: ToolCall) -> ToolCall:
    """
    Intercept tool calls before execution.

    You can:
    - Validate arguments
    - Modify arguments (return updated ToolCall)
    - Deny execution: tool_call.model_copy(update={"decision": "deny", "reason": "..."})
    - Request confirmation: tool_call.model_copy(update={"decision": "ask", "reason": "..."})

    Args:
        tool_call: The tool call about to be executed.

    Returns:
        The (potentially modified) tool call.
    """
    # Add your pre-tool logic here
    return tool_call


# --- POST_TOOL: Runs after a tool completes ---


async def post_tool_hook(tool_call: ToolCall, tool_return: ToolReturn) -> ToolReturn:
    """
    Process tool results after execution.

    You can:
    - Log the result
    - Mask sensitive data
    - Modify the return value (what the LLM sees)
    - Add metadata (not seen by LLM, available in application)

    Args:
        tool_call: The tool call that was executed.
        tool_return: The result from the tool.

    Returns:
        The (potentially modified) tool return.
    """
    # Add your post-tool logic here
    return tool_return


# --- PRE_RUN: Runs before the agent starts processing ---


async def pre_run_hook(
    input: Any,
    options: AgentOptions,
    context: AgentRunContext,
) -> Any:
    """
    Process input before the agent run starts.

    You can:
    - Validate the input
    - Transform or enrich the input
    - Replace with rejection message to block the run
    - Access/modify run context

    Args:
        input: The user input (string, BaseModel, or None).
        options: Agent run options.
        context: Agent run context.

    Returns:
        The (potentially modified) input.
    """
    # Add your pre-run logic here
    return input


# --- POST_RUN: Runs after the agent completes ---


async def post_run_hook(
    result: AgentResult,
    options: AgentOptions,
    context: AgentRunContext,
) -> AgentResult:
    """
    Process the agent result after the run completes.

    You can:
    - Log the result
    - Modify the content
    - Access tool call history via result.tool_calls
    - Check token usage via result.usage

    Args:
        result: The agent result.
        options: Agent run options.
        context: Agent run context.

    Returns:
        The (potentially modified) agent result.
    """
    # Add your post-run logic here
    return result


# --- ON_EVENT: Runs for each streaming event ---


async def on_event_hook(event: StreamingEvent) -> StreamingEvent | None:
    """
    Process individual streaming events during run_streaming().

    You can:
    - Transform text chunks (when event is str)
    - Filter/suppress events (return None)
    - Pass through non-text events unchanged

    StreamingEvent can be: str, ToolCall, ToolCallResult, ToolEvent,
    DownstreamAgentResult, BasePrompt, Usage, ConfirmationRequest.

    Args:
        event: The streaming event.

    Returns:
        The (potentially modified) event, or None to suppress it.
    """
    # Add your on-event logic here
    return event


def get_custom_hooks(tool_names: list[str] | None = None) -> list[Hook]:
    """
    Create all custom hooks.

    Args:
        tool_names: List of tool names for PRE_TOOL/POST_TOOL hooks. None = all tools.

    Returns:
        List of hooks to pass to the Agent constructor.
    """
    return [
        Hook(event_type=EventType.PRE_TOOL, callback=pre_tool_hook, tool_names=tool_names, priority=100),
        Hook(event_type=EventType.POST_TOOL, callback=post_tool_hook, tool_names=tool_names, priority=100),
        Hook(event_type=EventType.PRE_RUN, callback=pre_run_hook, priority=100),
        Hook(event_type=EventType.POST_RUN, callback=post_run_hook, priority=100),
        Hook(event_type=EventType.ON_EVENT, callback=on_event_hook, priority=100),
    ]
```

## Step 3: Integrate Hooks into the Agent

After generating the hooks file, update the existing agent file to import and register the hooks.

Find the `Agent(` constructor call in the user's agent file and add the `hooks` parameter.

### If the agent already has hooks

Merge the new hooks with existing ones. For example, if the agent already has:
```python
agent = Agent(llm=llm, tools=[...], hooks=[existing_hook])
```

Update to:
```python
from {module}.hooks import get_{hook_type}_hooks

agent = Agent(llm=llm, tools=[...], hooks=[existing_hook, *get_{hook_type}_hooks()])
```

### If the agent does not have hooks

Add the hooks parameter. For example, if the agent has:
```python
agent = Agent(llm=llm, tools=[...])
```

Update to:
```python
from {module}.hooks import get_{hook_type}_hooks

agent = Agent(llm=llm, tools=[...], hooks=get_{hook_type}_hooks())
```

### Hook function name mapping

| Hook type | Import function |
|---|---|
| `confirmation` | `get_confirmation_hooks(tool_names=[...])` |
| `validation` | `get_validation_hooks(tool_names=[...])` |
| `logging` | `get_logging_hooks(tool_names=[...])` |
| `guardrails` | `get_guardrail_hooks()` |
| `streaming-transform` | `get_streaming_hooks(replacements={...})` |
| `custom` | `get_custom_hooks(tool_names=[...])` |

If `--tools` was specified, pass those tool names to the function. Otherwise:
- For `confirmation`, `validation`, `logging`: pass `tool_names=None` (all tools) with a comment to customize
- For `guardrails`: no tool_names parameter (PRE_RUN hooks apply to all runs)
- For `streaming-transform`: provide example parameters with a comment to customize
- For `custom`: pass `tool_names=None` with a comment to customize

## Step 4: Install Dependencies (if needed)

Check if `ragbits-agents` is already installed. If not:

### Detect local ragbits development

First, check if a `packages/` directory exists in the ragbits repo root. Look for `{some_ancestor}/packages/ragbits-agents/`.

**If local ragbits packages are found**:

```bash
pip install -e {path_to_ragbits_repo}/packages/ragbits-core \
            -e {path_to_ragbits_repo}/packages/ragbits-agents
```

**If NOT in a local ragbits repo**:

```bash
pip install ragbits-agents
```

If the `guardrails` hook type was selected, also install:

```bash
pip install ragbits-guardrails
```

## Step 5: Summary

After creating the hooks file and updating the agent, print a summary:

```
Added {hook_type} hooks to your agent project:
  - {hooks_file_path} (hook definitions)
  - {agent_file_path} (updated agent with hooks)

Hooks registered:
  {list each hook with event type, priority, and description}

To customize:
  Edit {hooks_file_path} and modify the callback functions.

Hook priority order (lower = executes first):
  {list hooks in priority order}

Quick reference:
  - PRE_TOOL callbacks: async (ToolCall) -> ToolCall
  - POST_TOOL callbacks: async (ToolCall, ToolReturn) -> ToolReturn
  - PRE_RUN callbacks: async (input, AgentOptions, AgentRunContext) -> input
  - POST_RUN callbacks: async (AgentResult, AgentOptions, AgentRunContext) -> AgentResult
  - ON_EVENT callbacks: async (StreamingEvent) -> StreamingEvent | None
```

### Multiple hook types

If the user asks for multiple hook types (e.g., "add validation and logging"), generate separate functions in the same `hooks.py` file and register all of them with the agent. Combine the `get_*_hooks()` calls:

```python
hooks = [
    *get_validation_hooks(tool_names=[...]),
    *get_logging_hooks(tool_names=[...]),
]
agent = Agent(llm=llm, tools=[...], hooks=hooks)
```
