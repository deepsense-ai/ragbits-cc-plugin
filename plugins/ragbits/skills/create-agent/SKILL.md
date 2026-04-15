---
name: create-agent
description: Scaffold a new ragbits orchestrator agent application. Use when the user wants to create a multi-agent orchestrator or A2A coordinator powered by ragbits.
disable-model-invocation: true
argument-hint: "[app-name] [--model MODEL] [--agent1-endpoint HOST:PORT] [--agent2-endpoint HOST:PORT]"
allowed-tools: Bash(mkdir *) Bash(pip *) Bash(uv *) Write Edit Read Glob Grep
---

# Create a Ragbits Agent Application

Generate a complete, ready-to-run ragbits orchestrator agent that coordinates multiple remote agents via A2A (Agent-to-Agent) protocol.

## Arguments

Parse `$ARGUMENTS` to extract:
- **app-name** (first positional arg, default: `orchestrator-agent`) — the project directory name
- **--model MODEL** (optional, default: `gpt-4.1`) — the LLM model to use via LiteLLM
- **--agent1-endpoint HOST:PORT** (optional, default: `127.0.0.1:8000`) — the endpoint for the first remote agent
- **--agent2-endpoint HOST:PORT** (optional, default: `127.0.0.1:8001`) — the endpoint for the second remote agent

If `$ARGUMENTS` is empty, use the default app name `orchestrator-agent`.

## Project Template

### orchestrator-agent

A coordinator agent that manages multiple remote agents (agent1 and agent2) via A2A protocol.

#### Directory Structure

```
{app-name}/
├── pyproject.toml
├── README.md
└── {app_name_snake}/
    ├── __init__.py
    └── orchestrator.py
```

#### pyproject.toml

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "ragbits[a2a]",
    "requests",
]
```

#### orchestrator.py

```python
import asyncio
import json

import requests
from pydantic import BaseModel

from ragbits.agents import Agent, ToolCallResult
from ragbits.core.llms import LiteLLM, ToolCall
from ragbits.core.prompt import Prompt

# Agent endpoints — configured via --agent1-endpoint and --agent2-endpoint
AGENT_ENDPOINTS = [
    ("{agent1_host}", {agent1_port}),  # Agent 1
    ("{agent2_host}", {agent2_port}),  # Agent 2
]

AGENTS_CARDS: dict[str, dict] = {}


def fetch_agent_card(host: str, port: int, protocol: str = "http") -> dict:
    """Fetches the agent card from the given host and port."""
    url = f"{protocol}://{host}:{port}"
    return requests.get(f"{url}/.well-known/agent.json", timeout=10).json()


def discover_agents() -> str:
    """Discover and register all available remote agents."""
    for host, port in AGENT_ENDPOINTS:
        try:
            agent_card = fetch_agent_card(host, port)
            AGENTS_CARDS[agent_card["name"]] = agent_card
            print(f"  Discovered: {agent_card['name']} at {agent_card['url']}")
        except Exception as e:
            print(f"  Warning: Could not reach agent at {host}:{port} — {e}")

    return "\n".join(
        f"name: {name}, description: {card['description']}, skills: {card['skills']}"
        for name, card in AGENTS_CARDS.items()
    )


class OrchestratorPromptInput(BaseModel):
    """Represents the orchestrator prompt input."""

    message: str
    agents: str


class OrchestratorPrompt(Prompt[OrchestratorPromptInput]):
    """Prompt template for the orchestrator agent."""

    system_prompt = """
    You are an Orchestrator Agent that coordinates multiple specialized agents to help users.

    To help the user, you have access to the following remote agents:
    {{ agents }}

    Use the `execute_agent` tool to interact with these agents.
    You can call multiple agents to gather comprehensive information.

    Coordinate the available agents to fulfill the user's request.
    Combine the information from multiple agents into a coherent response.

    Be conversational and helpful. Ask clarifying questions if needed.
    """

    user_prompt = "{{ message }}"


def execute_agent(agent_name: str, query: str) -> str:
    """
    Executes a specified remote agent with the given query.

    Args:
        agent_name: Name of the agent to execute (must match a discovered agent name).
        query: The query to pass to the agent.

    Returns:
        JSON string of the execution result.
    """
    if agent_name not in AGENTS_CARDS:
        available = ", ".join(AGENTS_CARDS.keys()) or "none"
        return json.dumps({"status": "error", "message": f"Agent '{agent_name}' not found. Available agents: {available}"})

    payload = {"params": {"input": query}}
    raw_response = requests.post(AGENTS_CARDS[agent_name]["url"], json=payload, timeout=60)
    raw_response.raise_for_status()

    response = raw_response.json()
    result_data = response["result"]

    tool_calls = [
        {"name": call["name"], "arguments": call["arguments"], "output": call["result"]}
        for call in result_data.get("tool_calls", [])
    ] or None

    return json.dumps(
        {
            "status": "success",
            "agent_name": agent_name,
            "result": {
                "content": result_data["content"],
                "metadata": result_data.get("metadata", {}),
                "tool_calls": tool_calls,
            },
        }
    )


async def main() -> None:
    """Runs the orchestrator as an interactive chat."""
    print("Orchestrator Agent")
    print("=" * 40)
    print("Discovering remote agents...")

    agents_info = discover_agents()

    if not AGENTS_CARDS:
        print("\nNo agents found! Make sure the remote agents are running at the configured endpoints.")
        print("  Expected endpoints:")
        for host, port in AGENT_ENDPOINTS:
            print(f"    http://{host}:{port}")
        return

    print(f"\nReady! {len(AGENTS_CARDS)} agent(s) available.")
    print("Type your questions below. Type 'quit' to exit.\n")

    llm = LiteLLM(
        model_name="{model}",
        use_structured_output=True,
    )

    agent = Agent(llm=llm, prompt=OrchestratorPrompt, tools=[execute_agent], keep_history=True)

    while True:
        user_input = input("\nYOU: ").strip()
        if user_input.lower() in ("quit", "exit", "q"):
            print("Goodbye!")
            break
        if not user_input:
            continue

        print("\nASSISTANT:")
        async for chunk in agent.run_streaming(
            OrchestratorPromptInput(message=user_input, agents=agents_info)
        ):
            match chunk:
                case ToolCall():
                    print(f"  [Calling {chunk.name}: {chunk.arguments}]")
                case ToolCallResult():
                    result_preview = chunk.result[:80] + "..." if len(chunk.result) > 80 else chunk.result
                    print(f"  [Result from {chunk.name}: {result_preview}]")
                case _:
                    print(chunk, end="", flush=True)
        print()


if __name__ == "__main__":
    asyncio.run(main())
```

Replace placeholders:
- `{model}` with the chosen model (default: `gpt-4.1`)
- `{agent1_host}` and `{agent1_port}` with the agent1 endpoint (default: `127.0.0.1` and `8000`)
- `{agent2_host}` and `{agent2_port}` with the agent2 endpoint (default: `127.0.0.1` and `8001`)

#### README.md

```markdown
# {App Name}

A ragbits-powered orchestrator agent that coordinates multiple specialized agents via A2A (Agent-to-Agent) protocol.

## Features

- Automatic discovery of remote agents via A2A protocol
- Coordinates agent1 and agent2 (or any A2A-compatible agents)
- Interactive conversational interface with streaming responses
- Maintains conversation history for multi-turn interactions

## Architecture

```
User <-> Orchestrator Agent <-> Agent 1 ({agent1_host}:{agent1_port})
                            <-> Agent 2 ({agent2_host}:{agent2_port})
```

## Prerequisites

- Python >= 3.10
- `OPENAI_API_KEY` environment variable set (or configure another LLM provider)
- Remote agents must be running at the configured endpoints (see below)

## Setup

Install dependencies:

```bash
pip install -e .
```

## Usage

### Step 1: Start the specialized agents

In separate terminals, start the remote agents:

```bash
# Terminal 1 — Agent 1
cd ../agent1
python agent1/agent.py
# Starts on http://{agent1_host}:{agent1_port}

# Terminal 2 — Agent 2
cd ../agent2
python agent2/agent.py
# Starts on http://{agent2_host}:{agent2_port}
```

### Step 2: Run the orchestrator

```bash
python {app_name_snake}/orchestrator.py
```

### Step 3: Interact

```
YOU: <your question here>

ASSISTANT:
  [Calling execute_agent: ...]
  [Calling execute_agent: ...]
  <combined response from agents>
```

## Customization

- Add more agent endpoints in the `AGENT_ENDPOINTS` list in `orchestrator.py`
- Any A2A-compatible agent can be added as a remote agent
```

---

## Execution Steps

### Step 1: Create Project Structure

Create the directory layout as specified in the template above.

**IMPORTANT**: Always create an empty `__init__.py` in the `{app_name_snake}/` directory so it is a valid Python package.

Where `{app_name_snake}` is the app name converted to snake_case (hyphens to underscores).

### Step 2: Generate Source Files

Generate `orchestrator.py` using the template above. Replace all placeholders: `{model}` with the chosen model, `{agent1_host}`/`{agent1_port}` and `{agent2_host}`/`{agent2_port}` with the endpoint values.

### Step 3: Generate pyproject.toml and README.md

Use the templates from the project template above.

### Step 4: Install Dependencies

**CRITICAL**: After generating all files, you MUST install the project dependencies before the app can run.

#### Detect local ragbits development

First, check if a `packages/` directory exists in the ragbits repo root (parent or ancestor of the current working directory). Look for a path like `{some_ancestor}/packages/ragbits-core/`.

**If local ragbits packages are found** (i.e. developing ragbits from source), install from local paths to avoid version conflicts with PyPI:

```bash
pip install -e {path_to_ragbits_repo}/packages/ragbits-core \
            -e {path_to_ragbits_repo}/packages/ragbits-agents \
            requests
```

**If NOT in a local ragbits repo** (normal user), install from PyPI:

```bash
cd {app-name}
pip install -e .
```

Wait for the installation to complete and verify there are no errors. If installation fails, help the user troubleshoot.

### Step 5: Summary

After creating all files AND installing dependencies, print a summary:

```
Created ragbits agent application: {app-name}/
  - {app_name_snake}/__init__.py (package marker)
  - {app_name_snake}/orchestrator.py (orchestrator agent)
  - pyproject.toml (dependencies)
  - README.md (documentation)

Dependencies installed successfully.

Before running, start the remote agents at:
  Agent 1: {agent1_host}:{agent1_port}
  Agent 2: {agent2_host}:{agent2_port}

Then run the orchestrator:
  cd {app-name}
  python {app_name_snake}/orchestrator.py

The orchestrator will discover agents and start an interactive chat session.
```
