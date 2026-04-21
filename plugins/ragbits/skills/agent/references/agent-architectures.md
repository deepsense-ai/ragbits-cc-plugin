# Agent Architectures

Five architectural patterns for building agents with ragbits. Pick one during scaffolding based on the user's description, then shape `agent.py` around it. All five compose with tools, hooks, MCP, and A2A — those are orthogonal concerns.

The patterns below are ordered from simplest to most complex. Prefer the simplest pattern that fits; add structure only when the task actually needs it.

| Pattern | Use when | Components |
|---------|----------|------------|
| [Tool-use agent](#1-tool-use-agent) | Single-purpose assistant that calls tools to accomplish one task | 1 prompt, N tools |
| [Prompt chaining](#2-prompt-chaining) | Work decomposes into a fixed linear pipeline | 2+ prompts run in sequence |
| [Routing](#3-routing) | Inputs fall into disjoint categories handled differently | 1 classifier + N specialists |
| [Orchestrator-workers](#4-orchestrator-workers) | Subtasks are discovered dynamically at runtime | 1 planner + dynamic worker calls |
| [Evaluator-optimizer](#5-evaluator-optimizer) | Output quality improves with critique and revision | Generator + critic loop |

---

## 1. Tool-use agent

A single prompt paired with a set of tools. The LLM decides when to call a tool, reads the result, and either calls another tool or produces a final answer. This is the default — the base template in `SKILL.md` already implements it.

### When to use
- The task is single-purpose: "answer Python questions", "summarize CSV", "search docs and cite"
- Success is a single reply to a single user message
- No fixed multi-step procedure is required

### When not to use
- The work has a known sequence of distinct steps → use prompt chaining
- Inputs split into categories with very different handling → use routing
- You need deliberate quality loops over the output → use evaluator-optimizer

### Sketch

```python
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class AssistantPrompt(Prompt[AssistantInput]):
    system_prompt = "You are a {domain} assistant. Use tools when needed."
    user_prompt = "{{ message }}"


async def run_agent(message: str) -> str:
    llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)
    agent = Agent(llm=llm, prompt=AssistantPrompt, tools=TOOLS)
    result = await agent.run(AssistantInput(message=message))
    return result.content
```

### Cross-references
- Tools: `references/checklists/new-tool.md`
- Hooks (confirmation, logging, guardrails): `references/checklists/new-hook.md`

---

## 2. Prompt chaining

Decompose the work into a linear sequence where each step's output feeds the next step's input. Each step uses a focused prompt (and optionally different tools or models). A gate between steps can short-circuit on failure.

### When to use
- The steps are known in advance and always run in the same order (e.g. *extract → validate → format*)
- Each step benefits from a narrower prompt than a single monolithic one
- You want intermediate state to inspect, cache, or gate on

### When not to use
- The next step depends on which *category* the input falls into → use routing
- You don't know the subtasks until runtime → use orchestrator-workers
- Only one step is actually needed → use tool-use

### Sketch

Two agents run in order. The second consumes the first's structured output. Use `Prompt[In, Out]` with a Pydantic model to make the handoff typesafe.

```python
from pydantic import BaseModel
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class RawInput(BaseModel):
    text: str


class Extracted(BaseModel):
    title: str
    bullet_points: list[str]


class ExtractPrompt(Prompt[RawInput, Extracted]):
    system_prompt = "Extract a title and bullet points from the input."
    user_prompt = "{{ text }}"


class SummarizePrompt(Prompt[Extracted]):
    system_prompt = "Write a 2-sentence summary from the title and bullets."
    user_prompt = "Title: {{ title }}\nBullets:\n{% for b in bullet_points %}- {{ b }}\n{% endfor %}"


async def run_chain(text: str) -> str:
    llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)
    extractor = Agent(llm=llm, prompt=ExtractPrompt)
    summarizer = Agent(llm=llm, prompt=SummarizePrompt)

    extracted = (await extractor.run(RawInput(text=text))).content
    summary = (await summarizer.run(extracted)).content
    return summary
```

### Notes
- Put a plain-Python gate between steps when a check is deterministic (`if not extracted.bullet_points: return "no content"`) — don't burn an LLM call on it.
- Keep the chain short. Three to four stages is usually the limit before orchestrator-workers is a better fit.

---

## 3. Routing

A lightweight classifier agent inspects the input and picks which specialist agent handles it. Each specialist has its own prompt and (optionally) its own tools. Inputs go down exactly one branch.

### When to use
- Inputs are heterogeneous and different categories benefit from different system prompts, tools, or models
- "Cheap classifier + expensive specialist" keeps cost and latency down on easy inputs
- Examples: support tickets (refund / tech / billing), query types (factual / creative / code)

### When not to use
- All inputs need the same handling → use tool-use
- Categories are not actually disjoint or need fused output → use orchestrator-workers
- The split is a fixed pipeline regardless of input → use prompt chaining

### Sketch

A classifier returns an enum; a dispatch function picks the specialist. Using structured output on the classifier is what makes this reliable — don't parse free text.

```python
from enum import Enum
from pydantic import BaseModel
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class Category(str, Enum):
    REFUND = "refund"
    TECH = "tech"
    BILLING = "billing"


class TicketInput(BaseModel):
    message: str


class Classification(BaseModel):
    category: Category
    reason: str


class ClassifierPrompt(Prompt[TicketInput, Classification]):
    system_prompt = "Classify the support ticket into one of: refund, tech, billing."
    user_prompt = "{{ message }}"


class RefundPrompt(Prompt[TicketInput]):
    system_prompt = "You handle refund requests. Be empathetic and follow the refund policy."
    user_prompt = "{{ message }}"


class TechPrompt(Prompt[TicketInput]):
    system_prompt = "You are a senior engineer. Diagnose technical issues."
    user_prompt = "{{ message }}"


class BillingPrompt(Prompt[TicketInput]):
    system_prompt = "You handle billing questions. Cite invoice line items when relevant."
    user_prompt = "{{ message }}"


SPECIALISTS = {
    Category.REFUND: RefundPrompt,
    Category.TECH: TechPrompt,
    Category.BILLING: BillingPrompt,
}


async def run_routing(message: str) -> str:
    llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)
    classifier = Agent(llm=llm, prompt=ClassifierPrompt)
    decision = (await classifier.run(TicketInput(message=message))).content

    specialist = Agent(llm=llm, prompt=SPECIALISTS[decision.category])
    result = await specialist.run(TicketInput(message=message))
    return result.content
```

### Notes
- The classifier prompt should list categories *and* rough examples for each. Vague category names are the top source of misrouting.
- A cheaper/faster model for the classifier is a legitimate optimization; the specialists can stay on the default model.

---

## 4. Orchestrator-workers

A planner agent decomposes the task into subtasks *at runtime*, then delegates each subtask to a worker. Workers can be local agents, remote A2A agents, or tool calls. The orchestrator gathers results and composes the final answer.

### When to use
- Subtasks aren't known until the input is seen (e.g. "research X and summarize": the sources aren't fixed)
- Multiple specialized agents already exist and should be composed dynamically
- Parallel execution of independent subtasks meaningfully reduces latency

### When not to use
- The subtasks are a fixed sequence → use prompt chaining
- Inputs split into disjoint categories with no fusion → use routing
- You're reaching for this to add "flexibility" without a concrete need — the planning step is expensive and fragile. Stick to simpler patterns until you need this.

### Sketch

For multi-agent orchestration in ragbits, the canonical path is the A2A protocol: workers are remote ragbits agents exposing an HTTP endpoint, and the orchestrator discovers them and calls them via a tool.

See `references/checklists/new-a2a.md` for the full setup (agent cards, server wrapper, discovery). The essential shape:

```python
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class OrchestratorPrompt(Prompt[OrchestratorInput]):
    system_prompt = """
    You coordinate specialist agents. Break the user's request into subtasks,
    call the relevant agents via execute_agent, and synthesize a final answer.
    Available agents: {{ agent_descriptions }}
    """
    user_prompt = "{{ message }}"


async def execute_agent(endpoint: str, message: str) -> str:
    """Send a message to a remote A2A agent and return its reply."""
    # Implementation: HTTP call to the agent's A2A endpoint — see new-a2a.md.
    ...


async def run_orchestrator(message: str) -> str:
    llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)
    agent = Agent(llm=llm, prompt=OrchestratorPrompt, tools=[execute_agent])
    result = await agent.run(OrchestratorInput(message=message))
    return result.content
```

### Notes
- Purely local orchestration (no A2A) also works: use nested `Agent` instances as tools. Reach for A2A when workers must be separately deployable or reusable across systems.
- Keep the orchestrator's prompt short. Long planning prompts encourage the model to over-decompose trivial requests.

### Cross-references
- Full A2A setup, agent cards, discovery: `references/checklists/new-a2a.md`

---

## 5. Evaluator-optimizer

A generator produces a draft; a critic evaluates it against explicit criteria; the generator revises based on the critique. Loop until the critic accepts or a max-iteration cap is hit.

### When to use
- Quality benefits visibly from revision: code with tests, long-form writing, translations, structured extraction with strict rules
- The acceptance criteria can be expressed clearly enough for an LLM to grade against
- A single-shot answer is noticeably worse than a revised one

### When not to use
- The task is simple enough that one pass is fine — the loop doubles/triples cost and latency
- Quality can be checked with a deterministic validator (regex, schema, test run) — use that in a chain instead of a critic LLM
- There are no clear criteria; the critic will wander

### Sketch

```python
from pydantic import BaseModel
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt import Prompt


class DraftInput(BaseModel):
    task: str
    previous_draft: str | None = None
    critique: str | None = None


class GeneratorPrompt(Prompt[DraftInput]):
    system_prompt = "Produce a draft for the task. If a previous draft and critique are given, revise accordingly."
    user_prompt = """
    Task: {{ task }}
    {% if previous_draft %}Previous draft: {{ previous_draft }}{% endif %}
    {% if critique %}Critique to address: {{ critique }}{% endif %}
    """


class CritiqueInput(BaseModel):
    task: str
    draft: str


class Critique(BaseModel):
    accepted: bool
    feedback: str


class CriticPrompt(Prompt[CritiqueInput, Critique]):
    system_prompt = """
    Judge the draft against these criteria: {list domain-specific criteria here}.
    Set accepted=true only when all criteria are met. Otherwise return specific, actionable feedback.
    """
    user_prompt = "Task: {{ task }}\nDraft: {{ draft }}"


async def run_loop(task: str, max_iterations: int = 3) -> str:
    llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)
    generator = Agent(llm=llm, prompt=GeneratorPrompt)
    critic = Agent(llm=llm, prompt=CriticPrompt)

    draft = (await generator.run(DraftInput(task=task))).content
    for _ in range(max_iterations):
        verdict = (await critic.run(CritiqueInput(task=task, draft=draft))).content
        if verdict.accepted:
            return draft
        draft = (await generator.run(
            DraftInput(task=task, previous_draft=draft, critique=verdict.feedback)
        )).content
    return draft
```

### Notes
- Always bound the loop with `max_iterations`. A critic that keeps finding nits is more common than one that accepts too easily.
- Spell out the acceptance criteria explicitly in the critic's system prompt. "Is this good?" produces noise; "Does it include X, Y, Z and stay under N words?" produces signal.
- If the check can be deterministic (does the code compile? does the JSON validate?) run it as a plain Python gate and skip the LLM critic.

---

## Combining patterns

These are building blocks, not exclusive choices. Common combinations:

- **Routing + tool-use**: classifier picks a specialist; each specialist is itself a tool-use agent. Covered above — routing is usually layered on tool-use.
- **Chain with a quality gate**: prompt chain whose final step is an evaluator-optimizer loop.
- **Orchestrator with routing workers**: the orchestrator dispatches to a router, which dispatches to specialists. Useful in large multi-agent systems.

When in doubt, start with tool-use and add structure only when you can name the concrete problem the extra structure solves.
