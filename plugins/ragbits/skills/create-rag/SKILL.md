---
name: create-rag
description: Scaffold a new ragbits RAG (Retrieval-Augmented Generation) application. Use when the user wants to create a RAG app, document search, knowledge base, or question-answering system powered by ragbits.
disable-model-invocation: true
argument-hint: "[app-name] [--model MODEL] [--embedding-model MODEL] [--vector-store STORE] [--features FEATURES]"
allowed-tools: Bash(mkdir *) Bash(pip *) Bash(uv *) Write Edit Read Glob Grep
---

# Create a Ragbits RAG Application

Generate a complete, ready-to-run ragbits RAG (Retrieval-Augmented Generation) application with document ingestion, vector search, and LLM-powered question answering.

## Arguments

Parse `$ARGUMENTS` to extract:
- **app-name** (first positional arg, default: `my-rag`) — the project directory name
- **--model MODEL** (optional, default: `gpt-4o-mini`) — the LLM model to use via LiteLLM for answer generation
- **--embedding-model MODEL** (optional, default: `text-embedding-3-small`) — the embedding model for document and query embeddings
- **--vector-store STORE** (optional, default: `in-memory`) — the vector store backend. Supported: `in-memory`, `chroma`, `qdrant`
- **--features FEATURES** (optional, comma-separated) — extra features to include. Supported: `chat-ui`, `auth`, `feedback`, `theme`, `reranker`, `rephraser`

If `$ARGUMENTS` is empty, ask the user what they want to build before proceeding. Otherwise infer from the description.

## Step 1: Gather Requirements

Based on the arguments and conversation context, determine:
1. **App name** — used for the directory and module name
2. **LLM model** — which model for answer generation (default: `gpt-4o-mini`)
3. **Embedding model** — which model for embeddings (default: `text-embedding-3-small`)
4. **Vector store** — which backend to use:
   - `in-memory` — InMemoryVectorStore (simple, no persistence, good for demos)
   - `chroma` — ChromaVectorStore (persistent, local)
   - `qdrant` — QdrantVectorStore (persistent, local or server)
5. **Features** — which capabilities to include:
   - `chat-ui` — add ragbits chat web UI for interactive Q&A
   - `auth` — add authentication with test users (requires `chat-ui`)
   - `feedback` — add like/dislike feedback forms (requires `chat-ui`)
   - `theme` — add UI customization with HeroUI theme support (requires `chat-ui`)
   - `reranker` — add LLM-based reranking of search results
   - `rephraser` — add LLM-based query rephrasing for better retrieval

If the user described what they want in natural language, map it to the above. For example, "a knowledge base with web interface and login" means features: `chat-ui`, `auth`.

If any of `auth`, `feedback`, or `theme` is selected, automatically include `chat-ui`.

## Step 2: Create Project Structure

Create the following directory layout:

```
{app-name}/
├── pyproject.toml
├── README.md
├── documents/
│   └── sample.md
└── {app_name_snake}/
    ├── __init__.py
    ├── ingest.py
    └── search.py          (if NOT chat-ui)
    └── app.py             (if chat-ui)
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
    "ragbits-document-search",
    "ragbits-core",
]
```

Add extra dependencies based on configuration:
- If `chat-ui`: add `"ragbits-chat"`
- If `chroma` vector store: add `"ragbits-core[chroma]"`
- If `qdrant` vector store: add `"ragbits-core[qdrant]"`

## Step 4: Generate Sample Documents

Create `documents/sample.md` with example content the user can replace:

```markdown
# Ragbits Framework

Ragbits is a Python framework for building AI-powered applications with a focus on Retrieval-Augmented Generation (RAG).

## Key Features

- **Document Search**: Ingest and search documents using vector embeddings for semantic similarity.
- **Chat Interface**: Build interactive chat applications with streaming responses and web UI.
- **Multiple Vector Stores**: Support for InMemory, Chroma, Qdrant, PgVector, and Weaviate backends.
- **Flexible Embeddings**: Use any embedding model via LiteLLM, including OpenAI, Cohere, and local models.
- **Document Parsing**: Automatically parse PDF, DOCX, Markdown, HTML, and many other formats.

## Getting Started

To build a RAG application with ragbits, you need three components:
1. An **embedder** to convert text into vector representations
2. A **vector store** to store and retrieve document embeddings
3. The **DocumentSearch** class to orchestrate ingestion and retrieval

## Architecture

Ragbits uses an async-first architecture. All core operations (ingestion, search, retrieval) are async,
making it suitable for high-throughput applications and web services.
```

## Step 5: Generate the Ingestion Script (ingest.py)

This script handles document ingestion into the vector store.

### Imports — always included:

```python
import asyncio
from pathlib import Path

from ragbits.core.embeddings.dense import LiteLLMEmbedder
from ragbits.document_search import DocumentSearch
from ragbits.document_search.documents.document import DocumentMeta
```

### Vector store imports — based on selection:

If `in-memory`:
```python
from ragbits.core.vector_stores.in_memory import InMemoryVectorStore
```

If `chroma`:
```python
import chromadb
from ragbits.core.vector_stores.chroma import ChromaVectorStore
```

If `qdrant`:
```python
from qdrant_client import AsyncQdrantClient
from ragbits.core.vector_stores.qdrant import QdrantVectorStore
```

### Build the ingest script:

```python
DOCUMENTS_DIR = Path(__file__).parent.parent / "documents"


async def get_document_search() -> DocumentSearch:
    """Create and return a configured DocumentSearch instance."""
    embedder = LiteLLMEmbedder(model_name="{embedding_model}")
```

Vector store initialization based on selection:

If `in-memory`:
```python
    vector_store = InMemoryVectorStore(embedder=embedder)
```

If `chroma`:
```python
    chroma_client = chromadb.PersistentClient(path=".chroma_data")
    vector_store = ChromaVectorStore(
        client=chroma_client,
        index_name="{app_name_snake}",
        embedder=embedder,
    )
```

If `qdrant`:
```python
    qdrant_client = AsyncQdrantClient(path=".qdrant_data")
    vector_store = QdrantVectorStore(
        client=qdrant_client,
        index_name="{app_name_snake}",
        embedder=embedder,
    )
```

Continue building DocumentSearch with optional components:

If `reranker`:
```python
    from ragbits.core.llms import LiteLLM
    from ragbits.document_search.retrieval.rerankers.litellm import LiteLLMReranker

    llm = LiteLLM(model_name="{model}")
    reranker = LiteLLMReranker(model="{model}")
```

If `rephraser`:
```python
    from ragbits.core.llms import LiteLLM
    from ragbits.document_search.retrieval.rephrasers.llm import LLMQueryRephraser

    llm = LiteLLM(model_name="{model}")
    rephraser = LLMQueryRephraser(llm=llm)
```

Assemble DocumentSearch:
```python
    document_search = DocumentSearch(
        vector_store=vector_store,
```

Add optional kwargs:
- If `reranker`: add `reranker=reranker,`
- If `rephraser`: add `query_rephraser=rephraser,`

```python
    )
    return document_search


async def ingest_documents() -> None:
    """Ingest all documents from the documents directory."""
    document_search = await get_document_search()

    documents = []
    for file_path in DOCUMENTS_DIR.iterdir():
        if file_path.is_file():
            documents.append(DocumentMeta.from_local_path(file_path))

    if not documents:
        print("No documents found in the documents/ directory.")
        return

    print(f"Ingesting {len(documents)} document(s)...")
    result = await document_search.ingest(documents)
    print(f"Successfully ingested {len(result.successful)} document(s).")
    if result.failed:
        print(f"Failed to ingest {len(result.failed)} document(s).")


if __name__ == "__main__":
    asyncio.run(ingest_documents())
```

## Step 6A: Generate Search Script (search.py) — if NOT chat-ui

This is a CLI-based search interface for querying ingested documents.

```python
import asyncio

from {app_name_snake}.ingest import get_document_search
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt.base import ChatFormat


SYSTEM_PROMPT = (
    "You are a helpful assistant that answers questions based on the provided context. "
    "Use the context below to answer the user's question. "
    "If the context doesn't contain relevant information, say so.\n\n"
    "Context:\n{context}"
)


async def search_and_answer(query: str) -> str:
    """Search documents and generate an answer using RAG."""
    document_search = await get_document_search()
    llm = LiteLLM(model_name="{model}")

    # Retrieve relevant documents
    results = await document_search.search(query)

    if not results:
        return "No relevant documents found for your query."

    # Build context from search results
    context = "\n\n---\n\n".join(
        element.text_representation
        for element in results
        if element.text_representation
    )

    # Generate answer with LLM
    messages: ChatFormat = [
        {"role": "system", "content": SYSTEM_PROMPT.format(context=context)},
        {"role": "user", "content": query},
    ]

    response = await llm.generate(messages)
    return response


async def main() -> None:
    """Interactive search loop."""
    print("RAG Search Application")
    print("Type your questions below. Type 'quit' to exit.\n")

    while True:
        query = input("Question: ").strip()
        if query.lower() in ("quit", "exit", "q"):
            break
        if not query:
            continue

        print("\nSearching...")
        answer = await search_and_answer(query)
        print(f"\nAnswer: {answer}\n")


if __name__ == "__main__":
    asyncio.run(main())
```

## Step 6B: Generate Chat Application (app.py) — if chat-ui

Build a web-based RAG chat interface using ragbits ChatInterface.

### Imports — always included for chat-ui:

```python
from collections.abc import AsyncGenerator

from ragbits.chat.interface import ChatInterface
from ragbits.chat.interface.types import ChatContext, ChatResponse
from ragbits.core.llms import LiteLLM
from ragbits.core.prompt.base import ChatFormat

from {app_name_snake}.ingest import get_document_search
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

### If `theme` feature is enabled, add:

```python
from ragbits.chat.interface.ui_customization import (
    HeaderCustomization,
    PageMetaCustomization,
    UICustomization,
)
```

### ChatInterface class

```python
SYSTEM_PROMPT = (
    "You are a helpful assistant that answers questions based on the provided context. "
    "Use the context below to answer the user's question. "
    "If the context doesn't contain relevant information, say so.\n\n"
    "Context:\n{context}"
)


class RAGChat(ChatInterface):
    """Ragbits RAG chat application with document search."""

    conversation_history = True
```

If `theme` feature:
```python
    ui_customization = UICustomization(
        header=HeaderCustomization(
            title="{App Name}",
            subtitle="RAG-powered Q&A",
            logo="📚",
        ),
        welcome_message="Hello! Ask me anything about the ingested documents.",
        starter_questions=[
            "What are the key features?",
            "How does this work?",
            "Summarize the main topics",
        ],
        meta=PageMetaCustomization(favicon="📚", page_title="{App Name}"),
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

Constructor:
```python
    def __init__(self) -> None:
        self.llm = LiteLLM(model_name="{model}")
```

Chat method with RAG:
```python
    async def chat(
        self,
        message: str,
        history: ChatFormat,
        context: ChatContext,
    ) -> AsyncGenerator[ChatResponse, None]:
        # Retrieve relevant documents
        document_search = await get_document_search()
        results = await document_search.search(message)

        # Build context from search results
        rag_context = "\n\n---\n\n".join(
            element.text_representation
            for element in results
            if element.text_representation
        )

        # Yield references for each retrieved document
        for element in results:
            if element.text_representation:
                source = "Document"
                if element.document_meta and element.document_meta.source:
                    source = str(element.document_meta.source)
                yield self.create_reference(
                    title=source,
                    content=element.text_representation[:200] + "..." if len(element.text_representation) > 200 else element.text_representation,
                )

        # Generate answer with context
        messages: ChatFormat = [
            {"role": "system", "content": SYSTEM_PROMPT.format(context=rag_context)},
            *history,
            {"role": "user", "content": message},
        ]

        streaming_result = self.llm.generate_streaming(messages)
        async for chunk in streaming_result:
            yield self.create_text_response(chunk)
```

### Main block

If `auth` feature:
```python
if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    RagbitsAPI(RAGChat, auth_backend=get_auth_backend()).run()
```

Otherwise:
```python
if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    RagbitsAPI(RAGChat).run()
```

## Step 7: Generate README.md

Create a README with:
- App name and description
- Prerequisites (API keys needed, e.g., `OPENAI_API_KEY`)
- How to install dependencies: `pip install -e .` or `uv pip install -e .`
- How to ingest documents:
  1. Place documents (PDF, DOCX, MD, TXT, HTML, etc.) in the `documents/` directory
  2. Run `python {app_name_snake}/ingest.py`
- How to search/run the app:
  - If NOT `chat-ui`: `python {app_name_snake}/search.py`
  - If `chat-ui`: list ALL three methods:
    1. `ragbits api run {app_name_snake}.app:RAGChat` (recommended, via ragbits CLI)
    2. `python -m {app_name_snake}.app` (note: use dots, NOT slashes)
    3. `python {app_name_snake}/app.py` (direct script execution)
  - If `auth`: include the `--auth` flag and test credentials
- Vector store info (selected backend and any setup needed)
- List of enabled features
- How to add more documents (place in `documents/` and re-run ingest)

**IMPORTANT**: In all commands and documentation, use **dots** for Python module paths (e.g. `my_rag.app`), never slashes. Slashes are only for filesystem paths when running a script directly with `python path/to/app.py`.

## Step 8: Install Dependencies

**CRITICAL**: After generating all files, you MUST install the project dependencies before the app can run.

### Detect local ragbits development

First, check if a `packages/` directory exists in the ragbits repo root (parent or ancestor of the current working directory). Look for a path like `{some_ancestor}/packages/ragbits-core/`.

**If local ragbits packages are found** (i.e. developing ragbits from source), install from local paths to avoid version conflicts with PyPI:

```bash
pip install -e {path_to_ragbits_repo}/packages/ragbits-core \
            -e {path_to_ragbits_repo}/packages/ragbits-document-search
```

Add additional local packages based on features:
- If `chat-ui`: also add `-e {path_to_ragbits_repo}/packages/ragbits-chat`
- If `chroma` vector store: also add `-e "{path_to_ragbits_repo}/packages/ragbits-core[chroma]"`
- If `qdrant` vector store: also add `-e "{path_to_ragbits_repo}/packages/ragbits-core[qdrant]"`

**If NOT in a local ragbits repo** (normal user), install from PyPI:

```bash
cd {app-name}
pip install -e .
```

Wait for the installation to complete and verify there are no errors. If installation fails, help the user troubleshoot.

## Step 9: Summary

After creating all files AND installing dependencies, print a summary:

If NOT `chat-ui`:
```
Created ragbits RAG application: {app-name}/
  - {app_name_snake}/__init__.py (package marker)
  - {app_name_snake}/ingest.py (document ingestion)
  - {app_name_snake}/search.py (search & Q&A)
  - documents/sample.md (sample document)
  - pyproject.toml (dependencies)
  - README.md (documentation)

Dependencies installed successfully.

To get started:
  1. Set your API key: export OPENAI_API_KEY=your-key
  2. Add documents to the documents/ directory
  3. Ingest documents: python {app_name_snake}/ingest.py
  4. Search: python {app_name_snake}/search.py

Vector store: {selected store}
Features: {list of enabled features}
```

If `chat-ui`:
```
Created ragbits RAG application: {app-name}/
  - {app_name_snake}/__init__.py (package marker)
  - {app_name_snake}/ingest.py (document ingestion)
  - {app_name_snake}/app.py (chat UI application)
  - documents/sample.md (sample document)
  - pyproject.toml (dependencies)
  - README.md (documentation)

Dependencies installed successfully.

To get started:
  1. Set your API key: export OPENAI_API_KEY=your-key
  2. Add documents to the documents/ directory
  3. Ingest documents: python {app_name_snake}/ingest.py
  4. Run the app: ragbits api run {app_name_snake}.app:RAGChat

Or run directly:
  python {app_name_snake}/app.py

Vector store: {selected store}
Features: {list of enabled features}
```
