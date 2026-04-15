<div align="center">

<h1>🐰 Ragbits Claude Code Plugin</h1>

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) that scaffolds ready-to-run [ragbits](https://ragbits.deepsense.ai) applications from templates.

[Homepage](https://deepsense.ai/rd-hub/ragbits/) | [Ragbits Documentation](https://ragbits.deepsense.ai) | [Contact](https://deepsense.ai/contact/)

---
</div>

## Available Skills

### `/create-chat`

Scaffold a ragbits **chat application** with a conversational web UI.

```
/create-chat my-chatbot --model gpt-4o --features auth,feedback,theme
```

**Options:**
- `--model MODEL` — LLM model via LiteLLM (default: `gpt-4o-mini`)
- `--features` — comma-separated: `auth`, `agents`, `upload`, `feedback`, `theme`, `document-search`

### `/create-rag`

Scaffold a ragbits **RAG application** with document ingestion, vector search, and LLM-powered Q&A.

```
/create-rag my-kb --model gpt-4o --vector-store chroma --features chat-ui,reranker
```

**Options:**
- `--model MODEL` — LLM model for answer generation (default: `gpt-4o-mini`)
- `--embedding-model MODEL` — embedding model (default: `text-embedding-3-small`)
- `--vector-store STORE` — backend: `in-memory`, `chroma`, `qdrant` (default: `in-memory`)
- `--features` — comma-separated: `chat-ui`, `auth`, `feedback`, `theme`, `reranker`, `rephraser`

## Installation

Add the plugin to your Claude Code project:

```bash
claude plugin marketplace add https://github.com/deepsense-ai/ragbits-cc-plugin.git
claude plugin install ragbits@ragbits-cc-plugin
```

## License

Ragbits is licensed under the [MIT License](https://github.com/deepsense-ai/ragbits-cc-plugin/blob/main/LICENSE).
