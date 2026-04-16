---
name: create-agent
description: Spec-driven ragbits agent. Invoke when asked to create, scaffold, or generate a new ragbits agent.
---

# Ragbits Agent Archetype

Build a Python ragbits agent end-to-end by delegating the workflow with ragbits agent context.

## What This Skill Does

This skill creates Ragbits agent. It supplies interview extensions, and implementation instructions to the generic `ragbits:create-agent` workflow. All lifecycle orchestration (prerequisites, interview, research, approval, implementation, validation) is handled by the base workflow.

## Invocation

Invoke `ragbits:create-agent` using the Skill tool with the following arguments:

```
--archetype-name "Ragbits agent" \
--domain-interview "../../references/interview-checklist.md" \
--spec-paths ../../references/agent-spec.md \
--default-libraries ragbits \
--implementation-guide "../../references/implementation-guide.md" \
--component-checklists "../../references/checklists/" \
```

Paths above are relative to this SKILL.md file (i.e., relative to the plugin's `skills/create-agent/` directory).

## Ragbits Agent Context

The arguments above wire in the following agent-specific resources:

| Argument | Agent Resource                | Purpose                                                   |
|----------|-------------------------------|-----------------------------------------------------------|
| `--domain-interview` | `references/interview-checklist.md` | Agent specific questions: `has_ui`, `has_prompts`         |
| `--spec-paths` | `references/agent-spec.md`    | Ragbits agent specification                               |
| `--implementation-guide` | `references/implementation-guide.md` | Agent build instructions                                  |
| `--component-checklists` | `references/checklists/`      | Per-component quality gates (tool, hook, mcp, prompt,a2a) |
| `--default-libraries` | `ragbits`                     | Core library; integration libs added during interview     |

## Important Notes

- **Direct implementation**: No intermediate skill generation -- the workflow builds the agent.
- **Spec-driven**: Code patterns come from `agent.md`, not from this file.
