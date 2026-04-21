# New Feedback Checklist

Quality gates for adding like/dislike feedback forms to the generated chat UI.

Feedback implies `chat-ui` — if the user requested feedback, also enable the chat UI and follow `new-chat-ui.md`.

## Dependency

Feedback ships with `ragbits-chat`. No extra install beyond the base chat package.

## Imports

```python
from typing import Literal

from pydantic import BaseModel, ConfigDict, Field

from ragbits.chat.interface.forms import FeedbackConfig
```

## Form Models

Ragbits renders Pydantic models as web forms. Keep the models small and purposeful — fewer fields lift response rates.

```python
class LikeForm(BaseModel):
    model_config = ConfigDict(
        title="Like Form",
        json_schema_serialization_defaults_required=True,
    )
    reason: str = Field(description="Why do you like this?", min_length=1)


class DislikeForm(BaseModel):
    model_config = ConfigDict(
        title="Dislike Form",
        json_schema_serialization_defaults_required=True,
    )
    issue_type: Literal[
        "Incorrect information", "Not helpful", "Unclear", "Other"
    ] = Field(description="What was the issue?")
    feedback: str = Field(description="Please provide more details", min_length=1)
```

The `json_schema_serialization_defaults_required=True` flag is important — without it, optional fields are silently ignored by the UI renderer.

## Wiring into `ChatInterface`

Attach a `feedback_config` class attribute on your `RAGChat`:

```python
class RAGChat(ChatInterface):
    conversation_history = True

    feedback_config = FeedbackConfig(
        like_enabled=True,
        like_form=LikeForm,
        dislike_enabled=True,
        dislike_form=DislikeForm,
    )
```

Leave `like_enabled=False` or `dislike_enabled=False` when one direction isn't wanted — the opposite button simply disappears from the UI.

## Persisting Feedback

By default, feedback is surfaced as a webhook event via `ChatInterface.handle_feedback` (which no-ops in the template). Override it when the user wants to log or persist feedback:

```python
async def handle_feedback(
    self,
    message_id: str,
    feedback_type: Literal["like", "dislike"],
    payload: dict,
) -> None:
    # payload matches the Pydantic model the user submitted
    logger.info("Feedback %s: %s", feedback_type, payload)
```

## Validation Checklist

- [ ] `LikeForm` and `DislikeForm` have at least one required field each — empty forms confuse users
- [ ] Both form models set `json_schema_serialization_defaults_required=True`
- [ ] `feedback_config` is assigned at the class level, not inside `__init__` — `ChatInterface` reads it before instantiation
- [ ] `like_enabled`/`dislike_enabled` match whether the corresponding form is defined
- [ ] When feedback is enabled, `chat-ui` is also enabled
- [ ] `pyproject.toml` lists `ragbits-chat`
