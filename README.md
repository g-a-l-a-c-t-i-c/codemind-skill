# Codemind Skill

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches your coding agent to hand routine JavaScript and TypeScript tasks to [Codemind](https://codemindhq.dev/?utm_source=github&utm_medium=skill-repo&utm_campaign=codemind-skill-readme), a remote MCP server. Your agent sends a task and acceptance criteria; Codemind returns tested files.

- **Pricing:** $0.10 per verified build. Failed builds are free. A free allowance is included.
- **Works with:** Claude Code, Cursor, Codex and OpenCode.
- **Stacks:** JavaScript and TypeScript.

## Use it

The Skill lets an agent create its own free account and connect itself, with no key pasted into chat. Copy `SKILL.md` into your agent's skills folder, or open [codemindhq.dev/agent-setup](https://codemindhq.dev/agent-setup?utm_source=github&utm_medium=skill-repo&utm_campaign=codemind-skill-readme) and paste the one setup prompt it shows.

The MCP server is `https://api.codemindhq.dev/mcp` (also listed in the official MCP registry as `dev.codemindhq/codemind`).

## What is in here

| File | Purpose |
|---|---|
| `SKILL.md` | The Skill: setup, how to write a task, how to read results and failures |
| `reference/spec-writing.md` | How to write acceptance criteria that build first time |
| `reference/tools.md` | Every tool the server exposes |
| `reference/patch-mode.md` | Changing existing files |
| `reference/stacks-and-errors.md` | Supported stacks and error codes |
| `EVALS.md` | Scenarios the Skill was checked against |

Questions: support@codemindhq.dev
