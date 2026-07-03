# Agents

Project-scoped agent definitions. One file per agent; filename = agent
name. The file body is the agent's system prompt.

See `../../AGENTS.md` for the project guide; this README documents the
**file format** only.

## Format

```markdown
---
description: What this agent does and when to invoke it. (Required.)
mode: subagent                # primary | subagent | all
model: anthropic/claude-sonnet-4-6   # optional; inherits project default
permission:
  edit: deny
  bash: ask
---

System prompt body. Markdown. Do not duplicate `prompt:` in frontmatter.
```

Frontmatter reference:
- Required: `description`
- Optional: `model`, `variant`, `mode`, `hidden`, `color`, `steps`,
  `options`, `permission`, `disable`, `temperature`, `top_p`

`mode`:
- `primary`  - user-selectable in the TUI
- `subagent` - invoked via the `task` tool
- `all`      - both

## Add a new agent

1. Create `.opencode/agents/<name>.md`.
2. Fill in frontmatter (`description` is required).
3. Write the prompt body in markdown.
4. Mention the agent under "Subagents" in `../../AGENTS.md`.
5. Quit and restart opencode - config is loaded once at startup.

## Disable

Set `disable: true` in frontmatter to keep the file on disk but skip
loading it.
