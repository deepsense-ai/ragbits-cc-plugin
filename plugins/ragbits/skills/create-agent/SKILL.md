---
name: create-agent
description: Scaffold a new ragbits multi-agent application. Use when the user wants to create an agent, multi-agent system, A2A, MCP, flight finder, city explorer, orchestrator, or tool-calling agent powered by ragbits.
disable-model-invocation: true
argument-hint: "[project-type] [--model MODEL] [--name APP_NAME]"
allowed-tools: Bash(mkdir *) Bash(pip *) Bash(uv *) Write Edit Read Glob Grep
---

# Create a Ragbits Agent Application

Generate a complete, ready-to-run ragbits agent application based on one of three project templates.

## Arguments

Parse `$ARGUMENTS` to extract:
- **project-type** (first positional arg, required) — one of: `flight-finder-agent`, `city-explorer-agent`, `orchestrator-agent`
- **--model MODEL** (optional, default: `gpt-4.1`) — the LLM model to use via LiteLLM
- **--name APP_NAME** (optional) — override the project directory name (defaults to the project-type value)

If `$ARGUMENTS` is empty, ask the user which agent they want to create and list the three options:
1. **flight-finder-agent** — A specialized agent that searches for available flights between cities using tool/function calling
2. **city-explorer-agent** — An agent that gathers and synthesizes city information from the internet using MCP (Model Context Protocol)
3. **orchestrator-agent** — A coordinator agent that manages multiple remote agents (flight finder + city explorer) via A2A (Agent-to-Agent) protocol to create comprehensive trip plans

If the user described what they want in natural language, map it to the closest project type. For example, "I want to build something that coordinates multiple agents" maps to `orchestrator-agent`.

## Project Templates

### Template 1: flight-finder-agent

A standalone agent with a custom tool for searching flights.

#### Directory Structure

```
{app-name}/
├── pyproject.toml
├── README.md
├── data/
│   └── flights.json
└── {app_name_snake}/
    ├── __init__.py
    └── agent.py
```

#### pyproject.toml

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "ragbits[a2a]",
]
```

#### data/flights.json

Create this file with mock flight data. Use the data file at `docs/agent-data/flights.json` in the ragbits repo. If that file does not exist, create it at `{app-name}/data/flights.json` with this content:

```json
{
  "routes": [
    {
      "from": "New York",
      "to": "Paris",
      "flights": [
        {"airline": "Air France", "flight_number": "AF001", "departure": "08:00 AM", "arrival": "09:30 PM", "duration": "7h 30m", "price_usd": 650},
        {"airline": "British Airways", "flight_number": "BA178", "departure": "10:00 AM", "arrival": "10:00 PM", "duration": "7h 00m", "price_usd": 720},
        {"airline": "Delta", "flight_number": "DL400", "departure": "01:00 PM", "arrival": "01:00 AM", "duration": "7h 00m", "price_usd": 580}
      ]
    },
    {
      "from": "New York",
      "to": "London",
      "flights": [
        {"airline": "British Airways", "flight_number": "BA117", "departure": "07:00 PM", "arrival": "07:00 AM", "duration": "7h 00m", "price_usd": 550},
        {"airline": "Virgin Atlantic", "flight_number": "VS4", "departure": "09:30 PM", "arrival": "09:30 AM", "duration": "7h 00m", "price_usd": 510}
      ]
    },
    {
      "from": "London",
      "to": "Tokyo",
      "flights": [
        {"airline": "Japan Airlines", "flight_number": "JL44", "departure": "11:00 AM", "arrival": "07:30 AM", "duration": "11h 30m", "price_usd": 890},
        {"airline": "ANA", "flight_number": "NH212", "departure": "02:00 PM", "arrival": "10:00 AM", "duration": "11h 00m", "price_usd": 920}
      ]
    },
    {
      "from": "Paris",
      "to": "Rome",
      "flights": [
        {"airline": "Alitalia", "flight_number": "AZ349", "departure": "06:30 AM", "arrival": "08:45 AM", "duration": "2h 15m", "price_usd": 180},
        {"airline": "Air France", "flight_number": "AF1404", "departure": "12:00 PM", "arrival": "02:10 PM", "duration": "2h 10m", "price_usd": 210}
      ]
    },
    {
      "from": "San Francisco",
      "to": "Sydney",
      "flights": [
        {"airline": "Qantas", "flight_number": "QF74", "departure": "10:00 PM", "arrival": "06:30 AM", "duration": "15h 30m", "price_usd": 1100},
        {"airline": "United", "flight_number": "UA863", "departure": "11:30 PM", "arrival": "08:00 AM", "duration": "15h 30m", "price_usd": 980}
      ]
    }
  ]
}
```

Also create the data file in the ragbits docs directory for reuse: `docs/agent-data/flights.json` (same content).

#### agent.py

```python
import json
from pathlib import Path

from pydantic import BaseModel

from ragbits.agents import Agent
from ragbits.agents.a2a.server import create_agent_server
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt

DATA_DIR = Path(__file__).parent.parent / "data"


def _load_flights() -> dict:
    """Load flight data from the JSON file."""
    with open(DATA_DIR / "flights.json") as f:
        return json.load(f)


def get_flight_info(departure: str, arrival: str) -> str:
    """
    Returns flight information between two locations.

    Args:
        departure: The departure city.
        arrival: The arrival city.

    Returns:
        A JSON string with flight details.
    """
    data = _load_flights()
    for route in data["routes"]:
        if route["from"].lower() == departure.lower() and route["to"].lower() == arrival.lower():
            return json.dumps(route, indent=2)
    return json.dumps({"from": departure, "to": arrival, "flights": "No flight data available for this route."})


class FlightPromptInput(BaseModel):
    """Defines the structured input schema for the flight search prompt."""

    input: str


class FlightPrompt(Prompt[FlightPromptInput]):
    """Prompt for a flight search assistant."""

    system_prompt = """
    You are a helpful travel assistant that finds available flights between two cities.
    When presenting flights, include airline, flight number, departure/arrival times, duration, and price.
    If no flights are found, suggest the user try different cities.
    """

    user_prompt = """
    {{ input }}
    """


llm = LiteLLM(
    model_name="{model}",
    use_structured_output=True,
)
flight_agent = Agent(llm=llm, prompt=FlightPrompt, tools=[get_flight_info])


async def main() -> None:
    """Runs the flight agent as an A2A server on port 8000."""
    flight_agent_card = await flight_agent.get_agent_card(
        name="Flight Info Agent",
        description="Provides available flight information between two cities including airline, times, and prices.",
        port=8000,
    )
    flight_agent_server = create_agent_server(flight_agent, flight_agent_card, FlightPromptInput)
    await flight_agent_server.serve()


if __name__ == "__main__":
    import asyncio

    asyncio.run(main())
```

Replace `{model}` with the chosen model (default: `gpt-4.1`).

#### README.md

```markdown
# {App Name}

A ragbits-powered flight finder agent that searches for available flights between cities.

## Features

- Tool/function calling to search flight data
- Exposed as an A2A (Agent-to-Agent) server for remote access
- Mock flight data for demo (replace with real API)

## Prerequisites

- Python >= 3.10
- `OPENAI_API_KEY` environment variable set (or configure another LLM provider)

## Setup

Install dependencies:

```bash
pip install -e .
```

## Usage

### Run as A2A server (default)

```bash
python {app_name_snake}/agent.py
```

The agent will start on `http://127.0.0.1:8000`. View the agent card at `http://127.0.0.1:8000/.well-known/agent.json`.

### Test locally (without A2A)

Edit the `main()` function in `agent.py` to run a direct query:

```python
async def main() -> None:
    result = await flight_agent.run(FlightPromptInput(input="Find flights from New York to Paris"))
    print(result.content)
```

## Data

Flight data is stored in `data/flights.json`. Replace with real API integration for production use.

---

### Template 2: city-explorer-agent

An agent that uses MCP (Model Context Protocol) to fetch and synthesize city information from the internet.

#### Directory Structure

```
{app-name}/
├── pyproject.toml
├── README.md
├── data/
│   └── cities.json
└── {app_name_snake}/
    ├── __init__.py
    └── agent.py
```

#### pyproject.toml

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "ragbits[a2a,mcp]",
    "mcp-server-fetch",
]
```

#### data/cities.json

Create this file with city reference data. Also create it at `docs/agent-data/cities.json` in the ragbits repo for reuse. Content:

```json
{
  "cities": [
    {
      "name": "Paris",
      "country": "France",
      "wikipedia_url": "https://en.wikipedia.org/wiki/Paris",
      "highlights": ["Eiffel Tower", "Louvre Museum", "Notre-Dame Cathedral", "Champs-Élysées"],
      "best_season": "Spring (April-June)",
      "language": "French",
      "currency": "EUR"
    },
    {
      "name": "London",
      "country": "United Kingdom",
      "wikipedia_url": "https://en.wikipedia.org/wiki/London",
      "highlights": ["Big Ben", "Tower of London", "British Museum", "Buckingham Palace"],
      "best_season": "Summer (June-August)",
      "language": "English",
      "currency": "GBP"
    },
    {
      "name": "Tokyo",
      "country": "Japan",
      "wikipedia_url": "https://en.wikipedia.org/wiki/Tokyo",
      "highlights": ["Shibuya Crossing", "Senso-ji Temple", "Tokyo Tower", "Meiji Shrine"],
      "best_season": "Spring (March-May)",
      "language": "Japanese",
      "currency": "JPY"
    },
    {
      "name": "Rome",
      "country": "Italy",
      "wikipedia_url": "https://en.wikipedia.org/wiki/Rome",
      "highlights": ["Colosseum", "Vatican City", "Trevi Fountain", "Pantheon"],
      "best_season": "Spring (April-June)",
      "language": "Italian",
      "currency": "EUR"
    },
    {
      "name": "Sydney",
      "country": "Australia",
      "wikipedia_url": "https://en.wikipedia.org/wiki/Sydney",
      "highlights": ["Sydney Opera House", "Harbour Bridge", "Bondi Beach", "Royal Botanic Garden"],
      "best_season": "Spring (September-November)",
      "language": "English",
      "currency": "AUD"
    },
    {
      "name": "New York",
      "country": "United States",
      "wikipedia_url": "https://en.wikipedia.org/wiki/New_York_City",
      "highlights": ["Statue of Liberty", "Central Park", "Times Square", "Empire State Building"],
      "best_season": "Fall (September-November)",
      "language": "English",
      "currency": "USD"
    }
  ]
}
```

#### agent.py

```python
import json
from pathlib import Path

from pydantic import BaseModel

from ragbits.agents import Agent
from ragbits.agents.a2a.server import create_agent_server
from ragbits.agents.mcp import MCPServerStdio
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt

DATA_DIR = Path(__file__).parent.parent / "data"


def _load_cities() -> dict:
    """Load city reference data from the JSON file."""
    with open(DATA_DIR / "cities.json") as f:
        return json.load(f)


def get_city_basics(city_name: str) -> str:
    """
    Returns basic reference information about a city (highlights, best season, currency).

    Args:
        city_name: The name of the city to look up.

    Returns:
        A JSON string with city reference data, or a message if the city is not in the database.
    """
    data = _load_cities()
    for city in data["cities"]:
        if city["name"].lower() == city_name.lower():
            return json.dumps(city, indent=2)
    return json.dumps({"city": city_name, "info": "City not in local database. Use the MCP fetch tool to look it up on Wikipedia."})


class CityExplorerPromptInput(BaseModel):
    """Defines the structured input schema for the city explorer prompt."""

    input: str


class CityExplorerPrompt(Prompt[CityExplorerPromptInput]):
    """Prompt for a city explorer assistant."""

    system_prompt = """
    You are a helpful travel assistant that gathers and synthesizes city information.

    You have two sources of information:
    1. A local database accessible via the `get_city_basics` tool — use this first for quick reference data.
    2. The MCP fetch server — use this to get detailed, up-to-date information from Wikipedia.
       Construct the Wikipedia URL like: https://en.wikipedia.org/wiki/{City_Name}

    Always start with the local database, then enrich with Wikipedia data for a comprehensive summary.
    Include: key highlights, best time to visit, language, currency, and interesting facts.
    """

    user_prompt = """
    {{ input }}
    """


async def main() -> None:
    """Runs the city explorer agent as an A2A server on port 8001."""
    async with MCPServerStdio(
        params={
            "command": "python",
            "args": ["-m", "mcp_server_fetch"],
        },
        client_session_timeout_seconds=60,
    ) as server:
        llm = LiteLLM(
            model_name="{model}",
            use_structured_output=True,
        )
        city_explorer_agent = Agent(
            llm=llm,
            prompt=CityExplorerPrompt,
            tools=[get_city_basics],
            mcp_servers=[server],
        )

        city_explorer_agent_card = await city_explorer_agent.get_agent_card(
            name="City Explorer Agent",
            description="Provides detailed information about cities including highlights, best time to visit, and cultural details.",
            port=8001,
        )
        city_explorer_server = create_agent_server(
            city_explorer_agent, city_explorer_agent_card, CityExplorerPromptInput
        )
        await city_explorer_server.serve()


if __name__ == "__main__":
    import asyncio

    asyncio.run(main())
```

Replace `{model}` with the chosen model (default: `gpt-4.1`).

#### README.md

```markdown
# {App Name}

A ragbits-powered city explorer agent that gathers and synthesizes information about cities using MCP (Model Context Protocol).

## Features

- Local city database with quick reference data
- MCP integration with web fetcher for live Wikipedia data
- Exposed as an A2A (Agent-to-Agent) server for remote access

## Prerequisites

- Python >= 3.10
- `OPENAI_API_KEY` environment variable set (or configure another LLM provider)

## Setup

Install dependencies:

```bash
pip install -e .
```

## Usage

### Run as A2A server (default)

```bash
python {app_name_snake}/agent.py
```

The agent will start on `http://127.0.0.1:8001`. View the agent card at `http://127.0.0.1:8001/.well-known/agent.json`.

### Test locally (without A2A)

Edit the `main()` function in `agent.py` to run a direct query:

```python
async def main() -> None:
    async with MCPServerStdio(...) as server:
        # ... agent setup ...
        result = await city_explorer_agent.run(CityExplorerPromptInput(input="Tell me about Paris"))
        print(result.content)
```

## Data

City reference data is stored in `data/cities.json`. Add more cities as needed.
The agent also fetches live data from Wikipedia via MCP for comprehensive answers.

---

### Template 3: orchestrator-agent

A coordinator agent that manages the flight finder and city explorer agents via A2A protocol.

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

# Default agent endpoints — adjust if running on different hosts/ports
AGENT_ENDPOINTS = [
    ("127.0.0.1", 8000),  # Flight Info Agent
    ("127.0.0.1", 8001),  # City Explorer Agent
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
    """Prompt template for the trip planning orchestrator."""

    system_prompt = """
    You are a Trip Planning Agent that helps users plan their travels.

    To help the user, you have access to the following remote agents:
    {{ agents }}

    Use the `execute_agent` tool to interact with these agents.
    You can call multiple agents to gather comprehensive information.

    When planning a trip:
    1. Use the City Explorer Agent to get information about the destination
    2. Use the Flight Info Agent to find available flights
    3. Combine the information into a helpful trip plan

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
    """Runs the orchestrator as an interactive trip planning chat."""
    print("Trip Planning Orchestrator")
    print("=" * 40)
    print("Discovering remote agents...")

    agents_info = discover_agents()

    if not AGENTS_CARDS:
        print("\nNo agents found! Make sure the flight-finder-agent and city-explorer-agent are running:")
        print("  Terminal 1: cd ../flight-finder-agent && python flight_finder_agent/agent.py")
        print("  Terminal 2: cd ../city-explorer-agent && python city_explorer_agent/agent.py")
        return

    print(f"\nReady! {len(AGENTS_CARDS)} agent(s) available.")
    print("Type your travel questions below. Type 'quit' to exit.\n")

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

Replace `{model}` with the chosen model (default: `gpt-4.1`).

#### README.md

```markdown
# {App Name}

A ragbits-powered orchestrator agent that coordinates multiple specialized agents via A2A (Agent-to-Agent) protocol to create comprehensive trip plans.

## Features

- Automatic discovery of remote agents via A2A protocol
- Coordinates Flight Finder and City Explorer agents
- Interactive conversational interface with streaming responses
- Maintains conversation history for multi-turn planning

## Architecture

```
User <-> Orchestrator Agent <-> Flight Info Agent (port 8000)
                            <-> City Explorer Agent (port 8001)


## Prerequisites

- Python >= 3.10
- `OPENAI_API_KEY` environment variable set (or configure another LLM provider)
- The **flight-finder-agent** and **city-explorer-agent** must be running (see below)

## Setup

Install dependencies:

```bash
pip install -e .
```

## Usage

### Step 1: Start the specialized agents

In separate terminals, start the flight finder and city explorer agents:

```bash
# Terminal 1 — Flight Finder Agent
cd ../flight-finder-agent
python flight_finder_agent/agent.py
# Starts on http://127.0.0.1:8000

# Terminal 2 — City Explorer Agent
cd ../city-explorer-agent
python city_explorer_agent/agent.py
# Starts on http://127.0.0.1:8001
```

### Step 2: Run the orchestrator

```bash
python {app_name_snake}/orchestrator.py
```

### Step 3: Interact

```
YOU: I want to visit Paris from New York. Find me flights and tell me about the city.

ASSISTANT:
  [Calling execute_agent: ...]
  [Calling execute_agent: ...]
  Here's your trip plan for New York to Paris...
```

## Customization

- Add more agent endpoints in the `AGENT_ENDPOINTS` list in `orchestrator.py`
- Create new specialized agents using `create-agent flight-finder-agent` or `create-agent city-explorer-agent` as templates
```

---

## Execution Steps

### Step 1: Determine Project Type

Parse `$ARGUMENTS` and identify which template to use. If unclear, ask the user.

### Step 2: Create Project Structure

Create the directory layout as specified in the selected template above.

**IMPORTANT**: Always create an empty `__init__.py` in the `{app_name_snake}/` directory so it is a valid Python package.

Where `{app_name_snake}` is the app name converted to snake_case (hyphens to underscores).

### Step 3: Create Data Files

If the project type is `flight-finder-agent` or `city-explorer-agent`, create the corresponding data files:
- In the project directory: `{app-name}/data/flights.json` or `{app-name}/data/cities.json`
- In the ragbits docs directory: `docs/agent-data/flights.json` or `docs/agent-data/cities.json` (for reuse across projects)

If creating the orchestrator, skip this step (it uses the other agents' data remotely).

### Step 4: Generate Source Files

Generate `agent.py` (or `orchestrator.py`) using the template for the selected project type. Replace all `{model}` placeholders with the chosen model.

### Step 5: Generate pyproject.toml and README.md

Use the templates from the selected project type.

### Step 6: Install Dependencies

**CRITICAL**: After generating all files, you MUST install the project dependencies before the app can run.

#### Detect local ragbits development

First, check if a `packages/` directory exists in the ragbits repo root (parent or ancestor of the current working directory). Look for a path like `{some_ancestor}/packages/ragbits-core/`.

**If local ragbits packages are found** (i.e. developing ragbits from source), install from local paths to avoid version conflicts with PyPI:

```bash
pip install -e {path_to_ragbits_repo}/packages/ragbits-core \
            -e {path_to_ragbits_repo}/packages/ragbits-agents
```

Additional packages based on project type:
- If `city-explorer-agent`: also add `mcp-server-fetch`
- If `orchestrator-agent`: also add `requests`

**If NOT in a local ragbits repo** (normal user), install from PyPI:

```bash
cd {app-name}
pip install -e .
```

Wait for the installation to complete and verify there are no errors. If installation fails, help the user troubleshoot.

### Step 7: Summary

After creating all files AND installing dependencies, print a summary.

For `flight-finder-agent`:
```
Created ragbits agent application: {app-name}/
  - {app_name_snake}/__init__.py (package marker)
  - {app_name_snake}/agent.py (flight finder agent)
  - data/flights.json (mock flight data)
  - pyproject.toml (dependencies)
  - README.md (documentation)

Dependencies installed successfully.

To run:
  cd {app-name}
  python {app_name_snake}/agent.py

The agent will start as an A2A server on http://127.0.0.1:8000
Agent card: http://127.0.0.1:8000/.well-known/agent.json
```

For `city-explorer-agent`:
```
Created ragbits agent application: {app-name}/
  - {app_name_snake}/__init__.py (package marker)
  - {app_name_snake}/agent.py (city explorer agent)
  - data/cities.json (city reference data)
  - pyproject.toml (dependencies)
  - README.md (documentation)

Dependencies installed successfully.

To run:
  cd {app-name}
  python {app_name_snake}/agent.py

The agent will start as an A2A server on http://127.0.0.1:8001
Agent card: http://127.0.0.1:8001/.well-known/agent.json
```

For `orchestrator-agent`:
```
Created ragbits agent application: {app-name}/
  - {app_name_snake}/__init__.py (package marker)
  - {app_name_snake}/orchestrator.py (trip planning orchestrator)
  - pyproject.toml (dependencies)
  - README.md (documentation)

Dependencies installed successfully.

Before running, start the specialized agents:
  Terminal 1: cd flight-finder-agent && python flight_finder_agent/agent.py
  Terminal 2: cd city-explorer-agent && python city_explorer_agent/agent.py

Then run the orchestrator:
  cd {app-name}
  python {app_name_snake}/orchestrator.py

The orchestrator will discover agents and start an interactive chat session.
```
