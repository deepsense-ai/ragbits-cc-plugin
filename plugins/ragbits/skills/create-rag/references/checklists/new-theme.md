# New Theme Checklist

Quality gates for customizing the chat UI header, branding, welcome message, and starter questions.

Theme implies `chat-ui` — if the user requested theming, also enable the chat UI and follow `new-chat-ui.md`.

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

Attach `ui_customization` as a class attribute on the `ChatInterface` subclass. All fields are optional — write only the ones the user asked about.

```python
class RAGChat(ChatInterface):
    conversation_history = True

    ui_customization = UICustomization(
        header=HeaderCustomization(
            title="{App Name}",
            subtitle="RAG-powered Q&A",
            logo="📚",                      # emoji, URL, or data URI
        ),
        welcome_message="Hello! Ask me anything about the ingested documents.",
        starter_questions=[
            "What are the key features?",
            "How does this work?",
            "Summarize the main topics",
        ],
        meta=PageMetaCustomization(
            favicon="📚",
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
- `welcome_message`: a one-line invitation that names the knowledge base

A "policy-qa RAG over HR policy PDFs" should end up with starter questions like "How many vacation days do I get?", not generic filler.

## Validation Checklist

- [ ] `ui_customization` is declared at the class level, not inside `__init__`
- [ ] Every string field is either filled with domain-specific copy or omitted — leaving `"Welcome!"` in place is a wasted opportunity
- [ ] `starter_questions` contains 2–4 entries aligned with the described corpus
- [ ] When the theme feature is enabled, `chat-ui` is also enabled
- [ ] `pyproject.toml` lists `ragbits-chat`
