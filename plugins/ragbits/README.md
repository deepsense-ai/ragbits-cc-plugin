# Ragbits Claude Code Plugin

Scaffold ready-to-run [ragbits](https://ragbits.deepsense.ai) applications directly from Claude Code. Three skills cover the common shapes of a ragbits project: conversational chat apps, tool-using agents, and RAG pipelines.

## Install

From a Claude Code session:

```
/plugin marketplace add https://github.com/deepsense-ai/ragbits-cc-plugin.git
/plugin install ragbits@ragbits-cc-plugin
```

Or non-interactively:

```bash
claude plugin marketplace add https://github.com/deepsense-ai/ragbits-cc-plugin.git
claude plugin install ragbits@ragbits-cc-plugin
```

## Skills

### `/create-chat`

Scaffold a ragbits chat application with a conversational web UI.

```
/create-chat my-chatbot --model gpt-4o --features auth,feedback,theme
```

Features: `auth`, `agents`, `upload`, `feedback`, `theme`, `document-search`.

### `/create-agent`

Scaffold a ragbits agent with the canonical `Agent` + `Prompt` + `LiteLLM` + tools pattern. Also handles A2A servers, MCP tool integration, multi-agent orchestrators, and agent hooks (validation, logging, guardrails, confirmation, streaming transforms).

```
/create-agent customer-support A bot that handles refund questions --model gpt-4.1
```

Mention MCP, A2A, hooks, or orchestration in the description and the relevant components are added without follow-up questions.

### `/create-rag`

Scaffold a ragbits RAG application with document ingestion, vector search, and LLM-powered Q&A.

```
/create-rag my-kb --model gpt-4o --vector-store chroma --features chat-ui,reranker
```

Vector stores: `in-memory`, `chroma`, `qdrant`.
Features: `chat-ui`, `auth`, `feedback`, `theme`, `reranker`, `rephraser`.

## Prerequisites

- Python 3.10+
- An API key for the model you choose — typically `OPENAI_API_KEY`; any LiteLLM-supported provider works (`ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, local models via `ollama/*`, etc.)

## Links

- [Ragbits documentation](https://ragbits.deepsense.ai/)
- [Ragbits homepage](https://deepsense.ai/rd-hub/ragbits/)
- [Marketplace repository](https://github.com/deepsense-ai/ragbits-cc-plugin/)
- [Contact](https://deepsense.ai/contact/)

## License

Licensed under the [MIT License](./LICENSE).
