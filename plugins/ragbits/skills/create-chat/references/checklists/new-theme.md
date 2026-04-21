# New Theme Checklist

Quality gates for customizing the chat UI header, branding, welcome message, and starter questions.

## Dependency

Theme support ships with `ragbits-chat`. No extra install.

## Imports

```python
from ragbits.chat.interface.ui_customization import (
    HeaderCustomization,
    PageMetaCustomization,
    UICustomization,
)
```

## Canonical Block

Attach `ui_customization` as a class attribute on the `ChatInterface` subclass. All fields are optional — write only the ones the user asked about, but fill them with domain-specific copy rather than leaving boilerplate.

```python
class Chat(ChatInterface):
    conversation_history = True

    ui_customization = UICustomization(
        header=HeaderCustomization(
            title="{App Name}",
            subtitle="{one-line tagline}",
            logo="💬",                     # emoji, URL, or data URI
        ),
        welcome_message="Hello! How can I help you today?",
        starter_questions=[
            "What can you help me with?",
            "Show me an example",
            "How does this work?",
        ],
        meta=PageMetaCustomization(
            favicon="💬",
            page_title="{App Name}",
        ),
    )
```

### Field-by-field

| Field                           | Purpose                                                             |
|---------------------------------|---------------------------------------------------------------------|
| `header.title`                  | Main header text — usually the app name                             |
| `header.subtitle`               | Tagline below the title                                             |
| `header.logo`                   | Emoji, URL, or data URI rendered before the title                   |
| `welcome_message`               | Shown above the chat before the first turn                          |
| `starter_questions`             | Clickable prompts that seed the first turn                          |
| `meta.favicon`                  | Browser tab icon (emoji or URL)                                     |
| `meta.page_title`               | Browser tab title                                                   |

## Derive Values from the App

When generating, pull copy from the app description rather than leaving boilerplate:

- `title`: a Title Case version of the app name
- `subtitle`: the first sentence of the description, trimmed
- `starter_questions`: 2–4 realistic questions drawn from the described domain
- `welcome_message`: a one-line invitation that names the purpose

A "support-bot chat assistant that answers product questions" should end up with starter questions like "How do I reset my password?", not generic filler.

## Validation Checklist

- [ ] `ui_customization` is declared at the class level, not inside `__init__`
- [ ] Every string field is either filled with domain-specific copy or omitted — leaving `"Welcome!"` in place is a wasted opportunity
- [ ] `starter_questions` contains 2–4 entries aligned with the described purpose
- [ ] `pyproject.toml` lists `ragbits-chat`
- [ ] Logo and favicon are either both emojis or both URLs — mixing them produces inconsistent rendering
