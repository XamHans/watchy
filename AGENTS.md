# Watchy Agent Skills

Skills for AI coding agents to integrate with [Watchy](https://watchy.dev) — the AWS operational graph for serverless teams.

## Install

```bash
npx skills add watchy-dev/watchy
```

Or install a specific skill:

```bash
npx skills add watchy-dev/watchy --skill watchy-error-capture
npx skills add watchy-dev/watchy --skill watchy-mcp-setup
```

## Available Skills

### watchy-error-capture

Capture real errors from AWS Lambda functions and send them to Watchy. The agent generates a zero-dependency service file (~100 lines of TypeScript) directly in your project — no npm packages needed.

- Auto-catches unhandled errors via handler wrapper
- Manual capture for specific errors with context
- Errors link to your operational graph (Stack -> Lambda -> API Endpoint)
- AI agents query real stack traces via MCP `search_errors` tool

**Location:** `skills/watchy-error-capture/SKILL.md`

### watchy-mcp-setup

Configure the Watchy MCP server so your AI coding agent can query live AWS operational data — stacks, Lambda errors, metrics, costs, and resource graphs.

- One-time setup: add MCP config to your agent
- Works with Claude Code, Cursor, Windsurf, and any MCP-compatible agent
- Requires a Watchy API key (free tier available)

**Location:** `skills/watchy-mcp-setup/SKILL.md`

## Skill Format

Skills follow the [Agent Skills specification](https://www.agentskills.in/docs/getting-started). Each skill is a directory containing:

- `SKILL.md` — Entry point with YAML frontmatter (name, description) and instructions
- `references/` — Optional detailed docs the agent can read progressively

## For Skill Authors

See [AGENTS.md naming rules](https://github.com/neondatabase/agent-skills/blob/main/AGENTS.md):

- Directory names: kebab-case, 1-64 chars, alphanumeric + hyphens
- `SKILL.md`: YAML frontmatter with `name` and `description` (max 1024 chars)
- Keep SKILL.md under 500 lines for context efficiency
- Use `references/` for detailed docs — agents read them on demand
