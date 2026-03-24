---
name: watchy-error-capture
description: >-
  Capture Lambda errors and send them to Watchy for operational graph correlation
  and AI agent debugging. Use when a user wants to track errors from AWS Lambda
  functions, set up error reporting for serverless apps, integrate Watchy error
  capture, or connect Lambda errors to their Watchy dashboard. Triggers include:
  "capture Lambda errors", "send errors to Watchy", "error tracking for Lambda",
  "watchy error setup", "instrument Lambda with Watchy".
---

# Watchy Error Capture

Send real errors from AWS Lambda functions to Watchy. Errors are linked to your operational graph (Stack -> Lambda -> API Endpoint) and queryable by AI agents via MCP.

**No npm packages required.** You generate a small service file (~120 lines) that uses plain `fetch()` to send errors to Watchy's ingestion API.

## How to Use This Skill (Agent Workflow)

**Do NOT just generate code immediately.** Follow this interactive workflow to help the developer capture the right errors with the right context and criticality.

Link: references/agent-workflow.md

## What This Does

1. **Analyzes the codebase** — finds Lambda handlers, error-prone patterns, business-critical code
2. **Asks the developer** — suggests criticality levels and what context to capture per handler
3. **Generates tailored code** — service file + per-handler capture with criticality and business context
4. Errors appear in Watchy linked to the operational graph, queryable by AI agents via MCP

## Prerequisites

- A Watchy account with at least one connected AWS account
- A Watchy API key with `write:errors` scope (Settings -> API Keys)
- `WATCHY_API_KEY` environment variable set on your Lambda functions

## Criticality Levels

Every captured error has a criticality level that determines how it's prioritized in Watchy:

| Level | When to use | Examples |
|---|---|---|
| `critical` | Revenue impact, data loss, security breach | Payment processing, data mutations, auth failures |
| `high` | User-facing failures, SLA violations | API responses, form submissions, file uploads |
| `medium` | Degraded experience, retryable failures | Cache misses, slow queries, partial feature failure |
| `low` | Background tasks, non-blocking operations | Analytics, notifications, cleanup jobs |

## Context — The Flexible Bag

Every error can carry a `context` object — a free-form JSON bag where you pack whatever makes debugging useful. The agent should suggest context fields based on what the handler does.

Examples:
```typescript
// Payment handler
captureError(err, ctx, {
  criticality: 'critical',
  context: { orderId, userId, amount, currency, paymentProvider: 'stripe' }
});

// User auth handler
captureError(err, ctx, {
  criticality: 'high',
  context: { userId, authMethod: 'oauth', provider: 'google', ip: event.requestContext.identity.sourceIp }
});

// Background sync job
captureError(err, ctx, {
  criticality: 'low',
  context: { batchId, processedCount: 47, totalCount: 100, retryable: true }
});
```

## API Reference

Full endpoint specification (request/response schemas, auth, rate limits).

Link: references/api-reference.md

## Code Patterns

Complete generated service file with all patterns (wrapper, manual capture, criticality, context).

Link: references/lambda-patterns.md
