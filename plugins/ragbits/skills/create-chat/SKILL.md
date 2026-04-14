---
name: create-chat
description: Scaffold a new ragbits chat application. Use when the user wants to create a chat app, chatbot, or conversational UI powered by ragbits.
disable-model-invocation: true
argument-hint: "[app-name] [--model MODEL] [--features FEATURES]"
allowed-tools: Bash(mkdir *) Bash(pip *) Bash(uv *) Write Edit Read Glob Grep
---

# Create a Ragbits Chat Application

Generate a complete, ready-to-run ragbits chat application based on user requirements.

## Arguments

Parse `$ARGUMENTS` to extract:
- **app-name** (first positional arg, default: `my-chat`) — the project directory name
- **--model MODEL** (optional, default: `gpt-4o-mini`) — the LLM model to use via LiteLLM
- **--features FEATURES** (optional, comma-separated) — extra features to include. Supported: `auth`, `agents`, `upload`, `feedback`, `theme`, `document-search`

If `$ARGUMENTS` is empty, ask the user what they want to build before proceeding. Otherwise infer from the description.

## Step 1: Gather Requirements

Based on the arguments and conversation context, determine:
1. **App name** — used for the directory and module name
2. **LLM model** — which model to use (default: `gpt-4o-mini`)
3. **Features** — which capabilities to include:
   - `auth` — add authentication with test users
   - `agents` — add agent with tool support
   - `upload` — add file upload handling
   - `feedback` — add like/dislike feedback forms
   - `theme` — add UI customization with HeroUI theme support
   - `document-search` — add RAG with document search

If the user described what they want in natural language, map it to the above features. For example, "a chat with file upload and login" means features: `auth`, `upload`.

## Step 2: Create Project Structure

Create the following directory layout:

```
{app-name}/
├── pyproject.toml
├── README.md
└── {app_name_snake}/
    ├── __init__.py
    └── app.py
```

Where `{app_name_snake}` is the app name converted to snake_case (hyphens to underscores).

**IMPORTANT**: Always create an empty `__init__.py` in the `{app_name_snake}/` directory so it is a valid Python package.

## Step 3: Generate pyproject.toml

Use this template:

```toml
[project]
name = "{app-name}"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "ragbits-chat",
]
```

Add extra dependencies based on features:
- If `agents`: add `"ragbits-agents"`
- If `document-search`: add `"ragbits-document-search"`

## Step 4: Generate the Chat Application (app.py)

Build the `app.py` file by composing these building blocks based on selected features.

### Base (always included)

```python
from collections.abc import AsyncGenerator

from ragbits.chat.interface import ChatInterface
from ragbits.chat.interface.types import ChatContext, ChatResponse
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt.base import ChatFormat
```

### If `feedback` feature is enabled, add:

```python
from typing import Literal
from pydantic import BaseModel, ConfigDict, Field
from ragbits.chat.interface.forms import FeedbackConfig, UserSettings
```

And create Pydantic form models:

```python
class LikeForm(BaseModel):
    model_config = ConfigDict(title="Like Form", json_schema_serialization_defaults_required=True)
    reason: str = Field(description="Why do you like this?", min_length=1)


class DislikeForm(BaseModel):
    model_config = ConfigDict(title="Dislike Form", json_schema_serialization_defaults_required=True)
    issue_type: Literal["Incorrect information", "Not helpful", "Unclear", "Other"] = Field(
        description="What was the issue?"
    )
    feedback: str = Field(description="Please provide more details", min_length=1)
```

### If `auth` feature is enabled, add:

```python
from ragbits.chat.auth import ListAuthenticationBackend
from ragbits.chat.auth.session_store import InMemorySessionStore
```

And create an auth backend factory:

```python
def get_auth_backend() -> ListAuthenticationBackend:
    users = [
        {
            "user_id": "1",
            "username": "admin",
            "password": "admin123",
            "email": "admin@example.com",
            "full_name": "Admin User",
            "roles": ["admin"],
        },
        {
            "user_id": "2",
            "username": "user",
            "password": "user123",
            "email": "user@example.com",
            "full_name": "Regular User",
            "roles": ["user"],
        },
    ]
    return ListAuthenticationBackend(users, session_store=InMemorySessionStore())
```

### If `agents` feature is enabled, add:

```python
from ragbits.agents import Agent, ToolCallResult
from ragbits.core.llms import ToolCall
from ragbits.chat.interface.types import LiveUpdateType
```

### If `upload` feature is enabled, add:

```python
from fastapi import UploadFile
```

### If `theme` feature is enabled, add:

```python
from ragbits.chat.interface.ui_customization import (
    HeaderCustomization,
    PageMetaCustomization,
    UICustomization,
)
```

### ChatInterface class

Build the class with these patterns:

```python
class Chat(ChatInterface):
    """Ragbits chat application."""

    conversation_history = True
```

If `theme` feature:
```python
    ui_customization = UICustomization(
        header=HeaderCustomization(
            title="{App Name}",
            subtitle="Powered by ragbits",
            logo="💬",
        ),
        welcome_message="Hello! How can I help you today?",
        starter_questions=[
            "What can you help me with?",
            "Tell me about yourself",
        ],
        meta=PageMetaCustomization(favicon="💬", page_title="{App Name}"),
    )
```

If `feedback` feature:
```python
    feedback_config = FeedbackConfig(
        like_enabled=True,
        like_form=LikeForm,
        dislike_enabled=True,
        dislike_form=DislikeForm,
    )
```

Constructor — if `agents` feature:
```python
    def __init__(self) -> None:
        self.llm = LiteLLM(model_name="{model}")
        self.agent = Agent(llm=self.llm, tools=[...])
```

Otherwise:
```python
    def __init__(self) -> None:
        self.llm = LiteLLM(model_name="{model}")
```

Chat method — if `agents` feature, use agent streaming:
```python
    async def chat(
        self,
        message: str,
        history: ChatFormat,
        context: ChatContext,
    ) -> AsyncGenerator[ChatResponse, None]:
        stream = self.agent.run_streaming(message)
        async for response in stream:
            match response:
                case str():
                    if response.strip():
                        yield self.create_text_response(response)
                case ToolCall():
                    yield self.create_live_update(
                        response.id, LiveUpdateType.START,
                        f"Using {response.name}", "Processing..."
                    )
                case ToolCallResult():
                    yield self.create_live_update(
                        response.id, LiveUpdateType.FINISH,
                        f"{response.name} completed",
                    )
```

Otherwise, use simple LLM streaming:
```python
    async def chat(
        self,
        message: str,
        history: ChatFormat,
        context: ChatContext,
    ) -> AsyncGenerator[ChatResponse, None]:
        streaming_result = self.llm.generate_streaming(
            [*history, {"role": "user", "content": message}]
        )
        async for chunk in streaming_result:
            yield self.create_text_response(chunk)
```

If `upload` feature, add upload handler:
```python
    async def upload_handler(self, file: UploadFile) -> None:
        content = await file.read()
        print(f"Received file: {file.filename}, size: {len(content)} bytes")
```

### Main block

If `auth` feature:
```python
if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    RagbitsAPI(Chat, auth_backend=get_auth_backend()).run()
```

Otherwise:
```python
if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    RagbitsAPI(Chat).run()
```

## Step 5: Generate README.md

Create a README with:
- App name and description
- How to install dependencies: `pip install -e .` or `uv pip install -e .`
- How to run the app — list ALL three methods:
  1. `ragbits api run {app_name_snake}.app:Chat` (recommended, via ragbits CLI)
  2. `python -m {app_name_snake}.app` (note: use dots, NOT slashes)
  3. `python {app_name_snake}/app.py` (direct script execution)
- If `auth`: include the `--auth` flag and test credentials
- List of enabled features

**IMPORTANT**: In all commands and documentation, use **dots** for Python module paths (e.g. `my_chat.app`), never slashes. Slashes are only for filesystem paths when running a script directly with `python path/to/app.py`.

## Step 6: Install Dependencies

**CRITICAL**: After generating all files, you MUST install the project dependencies before the app can run.

### Detect local ragbits development

First, check if a `packages/` directory exists in the ragbits repo root (parent or ancestor of the current working directory). Look for a path like `{some_ancestor}/packages/ragbits-core/`.

**If local ragbits packages are found** (i.e. developing ragbits from source), install from local paths to avoid version conflicts with PyPI:

```bash
pip install -e {path_to_ragbits_repo}/packages/ragbits-core \
            -e {path_to_ragbits_repo}/packages/ragbits-chat
```

Add additional local packages based on features:
- If `agents`: also add `-e {path_to_ragbits_repo}/packages/ragbits-agents`
- If `document-search`: also add `-e {path_to_ragbits_repo}/packages/ragbits-document-search`

**If NOT in a local ragbits repo** (normal user), install from PyPI:

```bash
cd {app-name}
pip install -e .
```

Wait for the installation to complete and verify there are no errors. If installation fails, help the user troubleshoot.

## Step 7: Summary

After creating all files AND installing dependencies, print a summary:

```
Created ragbits chat application: {app-name}/
  - {app_name_snake}/__init__.py (package marker)
  - {app_name_snake}/app.py (main application)
  - pyproject.toml (dependencies)
  - README.md (documentation)

Dependencies installed successfully.

To run:
  cd {app-name}
  ragbits api run {app_name_snake}.app:Chat

Or run directly:
  python {app_name_snake}/app.py

Features: {list of enabled features}
```
