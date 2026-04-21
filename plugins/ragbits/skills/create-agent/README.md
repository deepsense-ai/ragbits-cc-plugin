# ragbits:create-agent

Skill that scaffolds a new ragbits agent application — complete, immediately runnable, with the canonical `Agent` + `Prompt` + `LiteLLM` + tools pattern.

## What it generates

- `pyproject.toml` with the right `ragbits` extra(s) for the components used
- `agent.py` with an `Agent`, a `Prompt` subclass, a `run_agent()` helper that correctly unwraps `AgentResult.content`, and an interactive `main()` loop
- `tools.py` with a placeholder tool and the `TOOLS` list expected by the agent
- `README.md` with setup, run, and customization instructions
- Optional `hooks.py`, MCP server config, or A2A server wrapper — only when the user's description calls for them

## Example: Reddit daily digest agent

This repository includes a full specification you can hand to **ragbits:create-agent** as the agent description (paste the file contents, or attach `@examples/create-reddit-agent.txt` in Cursor).

The prompt in [`examples/create-reddit-agent.txt`](../../../../examples/create-reddit-agent.txt) asks for a **once-per-day** Ragbits agent that:

- Fetches **r/artificial** posts from the last 24 hours via Reddit’s **public `.json` endpoints only** (no OAuth), using `httpx` with browser-like headers and a priming request so requests are not blocked with HTTP 403.
- Defines tools such as **`fetch_reddit_posts`** (structured post fields + JSON envelope) and **`format_digest_as_markdown`** (dated digest layout with overall and per-post summaries).
- Uses **ragbits hooks** for error handling and logging around tool use.
- Keeps the LLM workflow strict: summaries go **into the formatter tool**, then the model replies with exactly **`DONE`**; callers must take Markdown from **tool results**, not from `AgentResult.content` alone (as noted in the prompt).
- Exposes a **cron-friendly** CLI entry point for scheduled runs.

Invoking the skill with that text (or a shortened version that preserves the same constraints) should produce a matching project layout, `pyproject.toml`, `hooks.py`, domain-specific tools, and README run instructions.

## References

| File                                    | Purpose                                                   |
|-----------------------------------------|-----------------------------------------------------------|
| `references/agent-spec.md`              | Advanced Agent patterns — streaming, structured output, orchestrator |
| `references/interview-checklist.md`     | Domain questions for a more accurate system prompt        |
| `references/checklists/new-tool.md`     | Quality gates for adding a tool                           |
| `references/checklists/new-hook.md`     | Quality gates for adding hooks (validation, logging, guardrails)      |
| `references/checklists/new-mcp.md`      | Integrating MCP servers                                   |
| `references/checklists/new-a2a.md`      | Exposing the agent as an A2A server / building orchestrators |
| `references/checklists/new-prompt.md`   | Quality gates for prompts                                 |
