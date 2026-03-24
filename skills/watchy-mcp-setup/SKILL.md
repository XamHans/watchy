---
name: watchy-mcp-setup
description: >-
  Configure the Watchy MCP server so AI coding agents can query live AWS
  operational data — stacks, Lambda errors, metrics, costs, and resource graphs.
  Use when a user wants to connect their AI agent to Watchy, set up MCP for
  AWS debugging, configure Claude Code or Cursor with Watchy, or enable
  production-aware coding. Triggers include: "connect watchy to claude",
  "set up watchy mcp", "configure mcp server", "watchy for cursor",
  "query aws from my agent", "production-aware coding".
---

# Watchy MCP Server Setup

Connect your AI coding agent to Watchy so it can query live AWS operational data while you code.

## What Your Agent Gets

Once configured, your agent can:
- **List stacks** — see all CloudFormation stacks across connected AWS accounts
- **Get stack overview** — resources, endpoints, Lambdas, and their relationships
- **Get Lambda errors** — 4xx/5xx error rates from API Gateway metrics
- **Search errors** — real error messages and stack traces (requires error capture skill)

## Prerequisites

- A Watchy account at [watchy.dev](https://watchy.dev)
- At least one connected AWS account
- A Watchy API key (Settings -> API Keys)

## Setup by Agent

### Claude Code

Add to `.claude/settings.json` (project-level) or `~/.claude/settings.json` (global):

```json
{
  "mcpServers": {
    "watchy": {
      "type": "http",
      "url": "https://mcp.watchy.dev",
      "headers": {
        "Authorization": "Bearer YOUR_WATCHY_API_KEY"
      }
    }
  }
}
```

Replace `YOUR_WATCHY_API_KEY` with the API key from Watchy Settings -> API Keys.

### Cursor

Add to `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "watchy": {
      "type": "http",
      "url": "https://mcp.watchy.dev",
      "headers": {
        "Authorization": "Bearer YOUR_WATCHY_API_KEY"
      }
    }
  }
}
```

### Windsurf

Add to `.windsurf/mcp.json`:

```json
{
  "mcpServers": {
    "watchy": {
      "type": "http",
      "url": "https://mcp.watchy.dev",
      "headers": {
        "Authorization": "Bearer YOUR_WATCHY_API_KEY"
      }
    }
  }
}
```

### VS Code + Copilot

Add to `.vscode/mcp.json`:

```json
{
  "servers": {
    "watchy": {
      "type": "http",
      "url": "https://mcp.watchy.dev",
      "headers": {
        "Authorization": "Bearer YOUR_WATCHY_API_KEY"
      }
    }
  }
}
```

## Available MCP Tools

| Tool | Description |
|---|---|
| `list_stacks` | List all CloudFormation stacks across connected AWS accounts |
| `get_stack_overview` | Get full details on a stack — resources, endpoints, Lambdas, edges |
| `get_lambda_errors` | Get 4xx/5xx error rates from API Gateway metrics for Lambda-backed endpoints |
| `search_errors` | Search real error events with stack traces (requires error capture integration) |

## API Key Scopes

The API key needs these scopes for full MCP access:

| Scope | Tools it enables |
|---|---|
| `read:stacks` | `list_stacks`, `get_stack_overview` |
| `read:metrics` | `get_lambda_errors` |
| `read:logs` | `search_errors` |

The default key scopes include all of these.

## Verify It Works

After configuration, ask your agent:

> "List my AWS stacks"

or

> "What Lambda errors happened in the last 24 hours?"

The agent should call the Watchy MCP tools and return real data from your connected AWS accounts.
