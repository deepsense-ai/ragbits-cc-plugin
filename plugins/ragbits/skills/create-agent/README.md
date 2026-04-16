# ragbits:agent

Agent-specific archetype layer for production Ragbits agents.

## Skill

`ragbits:create-agent`

- **Agent interview extensions**: `has_ui`, `has_prompts`, agent experience
- **Agent spec**: Ragbits agent patterns and conventions
- **Implementation guide**: Agent build instructions
- **Component checklists**: quality gates for tools, hook, mcp, a2a, prompt

## What it generates

- Modular structure (src-layout, common/ + service/)
- Full middleware stack (timing, logging, error handling)
- httpx async clients with connection pooling
- Pydantic models with LLM-friendly request validators
- Domain exception hierarchy

## References

| File                                | Purpose                                                   |
|-------------------------------------|-----------------------------------------------------------|
| `references/agent-spec.md`          | Prescriptive Ragbits agent spec                           |
| `references/interview-checklist.md` | Ragbits agen interview extensions                         |
| `references/implementation-guide.md` | Ragbits agen build instructions                           |
| `references/evaluation.md`          | LLM evaluation guide                                      |
| `references/checklists/`            | Component build checklists (tool, hook, mcp, a2a, prompt) |
