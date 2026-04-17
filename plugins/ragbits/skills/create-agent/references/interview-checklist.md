# Interview Checklist

Domain-specific questions that help produce a more accurate system_prompt and tool selection.
Use these when the user provides minimal context. Combine into one message — do not ask one at a time.

## Core (always relevant)

- **App name**: What directory/module name? (snake_case preferred)
- **Purpose**: Describe the domain and key tasks in 1–3 sentences
- **Model**: Which LLM? (default: `gpt-4.1`; also works with `claude-*`, `gemini/*`, `ollama/*`, etc.)

## Tools

Ask if the agent needs external actions:
- Does it call external APIs? Which ones?
- Does it read or write files? Which formats?
- Does it query a database? Which type (SQL, vector, graph)?
- Should tools run synchronously or asynchronously?
- Are there rate limits or quotas to respect?

## Hooks

Ask if the agent needs lifecycle control:
- Which tool calls need user confirmation before executing? (e.g. delete, send, pay)
- Should sensitive fields (API keys, PII, passwords) be masked in logs?
- Are there topics or phrases that should be blocked as input?
- Is there a maximum input length to enforce?

## MCP Servers

Ask if the agent connects to external tool providers:
- Which MCP server packages are used? (e.g. `@modelcontextprotocol/server-filesystem`)
- Is each server stdio-based (local subprocess) or SSE-based (remote HTTP)?
- Are there auth headers or environment variables required?

## A2A

Ask if the agent serves or coordinates:
- Should this agent expose itself as an A2A server? On which port?
- What skills should appear in the agent card?
- Does this agent coordinate other agents? Which endpoints do they listen on?
- Should the orchestrator wait for all agents or use the first response?
