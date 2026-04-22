<div align="center">

<h1>🐰 Ragbits Claude Code Plugin</h1>

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) that scaffolds ready-to-run [ragbits](https://ragbits.deepsense.ai) applications from templates.

[Homepage](https://deepsense.ai/rd-hub/ragbits/) | [Ragbits Documentation](https://ragbits.deepsense.ai) | [Contact](https://deepsense.ai/contact/)

---
</div>

## Available Skills

### `/chat`

Scaffold a ragbits **chat application** with a conversational web UI.

```
/chat my-chatbot --model gpt-4o --features auth,feedback,theme
```

**Options:**
- `--model MODEL` — LLM model via LiteLLM (default: `gpt-4o-mini`)
- `--features` — comma-separated: `auth`, `agents`, `upload`, `feedback`, `theme`, `document-search`

### `/agent`

Scaffold a ragbits **agent application** — a tool-using assistant with the canonical `Agent` + `Prompt` + `LiteLLM` pattern. Also handles A2A servers, MCP tool integration, multi-agent orchestrators, and agent hooks (validation, logging, guardrails, confirmation).

```
/agent customer-support A bot that handles refund questions --model gpt-4.1
```

**Options:**
- `--model MODEL` — any LiteLLM-compatible model (default: `gpt-4.1`)

The skill infers features from the description — mention MCP, A2A, hooks, or orchestration and the relevant components are added automatically.

### `/rag`

Scaffold a ragbits **RAG application** with document ingestion, vector search, and LLM-powered Q&A.

```
/rag my-kb --model gpt-4o --vector-store chroma --features chat-ui,reranker
```

**Options:**
- `--model MODEL` — LLM model for answer generation (default: `gpt-4o-mini`)
- `--embedding-model MODEL` — embedding model (default: `text-embedding-3-small`)
- `--vector-store STORE` — backend: `in-memory`, `chroma`, `qdrant` (default: `in-memory`)
- `--features` — comma-separated: `chat-ui`, `auth`, `feedback`, `theme`, `reranker`, `rephraser`

### `/evaluate`

Scaffold a ragbits **evaluation project** that measures RAG / retrieval / answer quality using the `ragbits-evaluate` framework. Supports document-search precision/recall/F1, LLM-as-judge metrics for question-answering, synthetic dataset generation, and Optuna hyperparameter sweeps.

```
/evaluate retrieval-bench --target document-search --features optimize
```

**Options:**
- `--target TARGET` — `document-search`, `question-answer`, or `custom` (default: `document-search`)
- `--judge-model MODEL` — LLM for judge-style metrics (default: `gpt-4o-mini`)
- `--embedding-model MODEL` — embedding model (default: `text-embedding-3-small`)
- `--features` — comma-separated: `optimize`, `dataset-generator`, `custom-metric`, `custom-pipeline`

### `/mcp`

Scaffold an **MCP (Model Context Protocol) server** that exposes a ragbits agent, RAG pipeline, or prompt to any MCP client (Claude Desktop, Claude Code, Cursor). Built as a bridge: the official `mcp` SDK (FastMCP) hosts the server, ragbits primitives power the tool bodies.

```
/mcp docs-mcp Expose our product documentation RAG to Claude Desktop --backend rag
```

**Options:**
- `--backend BACKEND` — `agent`, `rag`, `prompt`, or `plain` (inferred from the description by default)
- `--transport TRANSPORT` — `stdio` (default), `sse`, or `streamable-http`
- `--model MODEL` — LiteLLM-compatible model for LLM-backed tools (default: `gpt-4.1`)

Note: `/agent` covers the *client* direction (a ragbits agent calling an external MCP server). `/mcp` is the *server* direction — exposing ragbits capabilities to external MCP clients.

## Installation

Add the plugin to your Claude Code project:

```bash
claude plugin marketplace add https://github.com/deepsense-ai/ragbits-cc-plugin.git
claude plugin install ragbits@ragbits-cc-plugin
```

## License

Ragbits is licensed under the [MIT License](https://github.com/deepsense-ai/ragbits-cc-plugin/blob/main/LICENSE).
