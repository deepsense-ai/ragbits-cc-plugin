# New Prompt Checklist

Quality gates for adding a new prompt to a ragbits agent.

## Definition Requirements

- [ ] Prompt subclasses `Prompt[InputModel, OutputType]` with both generic parameters specified
- [ ] Input model is a Pydantic `BaseModel` (or `None` if no input is needed)
- [ ] `user_prompt` class attribute is defined with Jinja2 template placeholders matching input model fields
- [ ] `system_prompt` class attribute is defined (optional but recommended)
- [ ] Template variables in `user_prompt` / `system_prompt` match field names in the input model exactly

## Imports

```python
from pydantic import BaseModel
from ragbits.core.prompt import Prompt
```

## Basic Prompt Pattern

```python
from pydantic import BaseModel
from ragbits.core.prompt import Prompt


class MyInput(BaseModel):
    """Input schema for the prompt."""
    topic: str
    detail_level: str


class MyPrompt(Prompt[MyInput, str]):
    """Prompt for generating content about a topic."""

    system_prompt = """
    You are a helpful assistant that provides information on various topics.
    Adjust your response detail based on the requested level.
    """

    user_prompt = """
    Tell me about {{ topic }} at a {{ detail_level }} level of detail.
    """
```

## Template Syntax

Templates use **Jinja2** syntax:

- `{{ field_name }}` — renders a field from the input model
- Templates are compiled at class definition time via `__init_subclass__`
- Whitespace is automatically formatted via `_format_message()`

## Output Types

### String output (default)

```python
class MyPrompt(Prompt[MyInput, str]):
    user_prompt = "..."
```

### Structured output (Pydantic model)

When the output type is a Pydantic model, the LLM uses JSON mode for structured responses:

```python
class AnalysisResult(BaseModel):
    summary: str
    confidence: float
    tags: list[str]


class AnalysisPrompt(Prompt[MyInput, AnalysisResult]):
    system_prompt = "Analyze the input and return structured results."
    user_prompt = "Analyze: {{ topic }}"
```

### Built-in parsers

The following output types are automatically parsed:
- `str` — no conversion
- `int` — `int(response)`
- `float` — `float(response)`
- `bool` — maps "true", "yes", "1" → `True`
- Any `BaseModel` subclass — `model_validate_json(response)`

### Custom parser

```python
class MyPrompt(Prompt[MyInput, MyCustomType]):
    user_prompt = "..."
    response_parser = my_custom_parse_function  # Callable[[str], MyCustomType]
```

## Usage with Agent

Prompts are passed to the Agent constructor:

```python
from ragbits.agents import Agent
from ragbits.core.llms import LiteLLM

llm = LiteLLM(model_name="gpt-4.1", use_structured_output=True)

agent = Agent(
    llm=llm,
    prompt=MyPrompt,  # pass the class, not an instance
    tools=[...],
)

# Run the agent — input is passed as the prompt's input model
result = await agent.run(MyInput(topic="ragbits", detail_level="detailed"))
print(result.content)
```

## Usage with LLM directly (without Agent)

```python
from ragbits.core.llms import LiteLLM

llm = LiteLLM(model_name="gpt-4.1")

prompt = MyPrompt(input_data=MyInput(topic="ragbits", detail_level="detailed"))
response = await llm.generate(prompt)
parsed = await prompt.parse_response(response)
```

## Prompt without input

For prompts that don't need structured input:

```python
class SimplePrompt(Prompt[None, str]):
    system_prompt = "You are a helpful assistant."
    user_prompt = "What is the meaning of life?"

# Usage
agent = Agent(llm=llm, prompt=SimplePrompt)
result = await agent.run(None)
```

## Few-Shot Examples

Add example input-output pairs for in-context learning:

```python
class MyPrompt(Prompt[MyInput, str]):
    system_prompt = "..."
    user_prompt = "..."

    few_shots: list = [
        ("Example input 1", "Example output 1"),
        ("Example input 2", "Example output 2"),
    ]
```

Or add dynamically at runtime:

```python
prompt = MyPrompt(input_data=my_input)
prompt.add_few_shot(
    user_message="Example question",
    assistant_message="Example answer",
)
```

## Conversation History

Pass prior conversation turns:

```python
prompt = MyPrompt(
    input_data=my_input,
    conversation_history=[
        {"role": "user", "content": "Previous question"},
        {"role": "assistant", "content": "Previous answer"},
    ],
)
```

## Validation Checklist

- [ ] Input model fields match all `{{ }}` template variables in `user_prompt` and `system_prompt`
- [ ] Output type has a matching parser (built-in or custom `response_parser`)
- [ ] System prompt clearly defines the agent's role and behavior
- [ ] User prompt includes all necessary context from input fields
- [ ] If using structured output (`BaseModel`), the LLM is created with `use_structured_output=True`
- [ ] Prompt class is passed to `Agent(prompt=...)` as a class reference, not an instance
