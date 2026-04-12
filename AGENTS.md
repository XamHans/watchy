# Watchy Agent Skills

Skills for AI coding agents to integrate with [Watchy](https://watchy.dev) — the AWS operational graph for serverless teams.

## Install

```bash
npx skills add XamHans/watchy
```

Or install a specific skill:

```bash
npx skills add XamHans/watchy --skill watchy
```

## Available Skills

### watchy

Capture real errors from AWS Lambda functions with criticality levels and business context. The agent analyzes your codebase, asks what matters, and generates a zero-dependency service file directly in your project.

- Interactive workflow: agent scans handlers, suggests criticality, asks what context to capture
- Auto-catches unhandled errors via handler wrapper
- Manual capture for specific errors with business context
- Errors link to your operational graph (Stack -> Lambda -> API Endpoint)
- AI agents query real stack traces via Watchy REST API

**Location:** `skills/watchy/SKILL.md`

### watchy-debug

Investigate AWS Lambda production errors using Watchy's operational graph, CloudWatch logs, git history, and CloudTrail events. The agent follows a senior debugger's playbook to find root cause and suggest fixes.

- Handles multi-account AWS setups: detects existing profiles, matches error's account to the right profile automatically
- Guides AWS CLI install + SSO/IAM setup when missing (macOS, Linux, Windows)
- Pulls CloudWatch logs + REPORT line for memory/timeout diagnosis
- Checks git history and CloudTrail deployments to correlate with recent changes
- Records resolution feedback via Watchy API after fix is applied

**Location:** `skills/watchy-debug/SKILL.md`
