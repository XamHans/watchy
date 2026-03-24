# Watchy Agent Skills

Skills for AI coding agents to integrate with [Watchy](https://watchy.dev) — the AWS operational graph for serverless teams.

## Install

```bash
npx skills add XamHans/watchy
```

Or install a specific skill:

```bash
npx skills add XamHans/watchy --skill watchy
npx skills add XamHans/watchy --skill watchy-mcp-setup
```

## Available Skills

### watchy

Capture real errors from AWS Lambda functions with criticality levels and business context. The agent analyzes your codebase, asks what matters, and generates a zero-dependency service file directly in your project.

- Interactive workflow: agent scans handlers, suggests criticality, asks what context to capture
- Auto-catches unhandled errors via handler wrapper
- Manual capture for specific errors with business context
- Errors link to your operational graph (Stack -> Lambda -> API Endpoint)
- AI agents query real stack traces via MCP `search_errors` tool

**Location:** `skills/watchy/SKILL.md`

### watchy-mcp-setup

Configure the Watchy MCP server so your AI coding agent can query live AWS operational data — stacks, Lambda errors, metrics, costs, and resource graphs.

- One-time setup: add MCP config to your agent
- Works with Claude Code, Cursor, Windsurf, and any MCP-compatible agent
- Requires a Watchy API key (free tier available)

**Location:** `skills/watchy-mcp-setup/SKILL.md`
