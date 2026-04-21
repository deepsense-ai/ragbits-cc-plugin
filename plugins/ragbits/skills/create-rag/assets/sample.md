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