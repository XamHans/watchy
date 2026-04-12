---
name: watchy
description: >-
  Integrate AWS Lambda projects with Watchy — capture errors with criticality
  and business context, then investigate production issues using Watchy's
  operational graph, CloudWatch logs, git history, and CloudTrail events.
  Use when a user wants to: track Lambda errors, set up error reporting,
  investigate production errors, debug a 500, find root cause of Lambda failures,
  or instrument Lambda handlers. Triggers include: "set up watchy",
  "capture Lambda errors", "investigate this error", "debug this 500",
  "what's wrong with my Lambda", "production-aware debugging".
---

# Watchy

Integrate your AWS Lambda project with [Watchy](https://watchy.dev) — the operational graph for serverless teams.

**What this skill does:**
1. Stores your Watchy API key securely in **AWS Secrets Manager**
2. Helps you **capture errors** from Lambda handlers with criticality levels and business context
3. **Verifies the setup** with a test error you can see in the Watchy dashboard
4. Errors flow into Watchy's operational graph (Stack -> Lambda -> API Endpoint)
5. AI agents **query real errors** via Watchy's REST API — stack traces, not just metric counts

**No npm packages required.** The generated service file uses only `@aws-sdk/client-secrets-manager` (bundled in Lambda runtimes) and `fetch()`.

## How to Use This Skill (Agent Workflow)

**Do NOT just generate code immediately.** Follow this interactive workflow to help the developer set up securely, capture the right errors, and verify everything works end-to-end.

Link: references/agent-workflow.md

## Full Setup Flow

### Step 1: API Key in Secrets Manager

The agent stores your Watchy API key in AWS Secrets Manager (not a plain env var) and ensures your Lambda role has read access.

### Step 2: Analyze & Capture

The agent scans your codebase, asks what matters, and generates error capture code tailored to your handlers — with per-handler criticality and business context.

```typescript
import { wrapHandler, captureError } from './lib/watchy';

// Auto-catch with criticality + context extraction
export const handler = wrapHandler(async (event, context) => {
  const order = await createOrder(event.body);
  return { statusCode: 200, body: JSON.stringify(order) };
}, {
  criticality: 'critical',
  contextExtractor: (event) => ({ orderId: event.body?.orderId, userId: event.body?.userId }),
});

// Manual capture in existing catch blocks — MUST await in Lambda
catch (err) {
  await captureError(err, context, {
    criticality: 'critical',
    context: { orderId, amount, currency },
  });
  throw err;
}
```

### Step 3: Verify with Test Error

The agent sends a curl test error to the Watchy API and asks you to confirm it appears in your dashboard — setup isn't complete until you verify.

### Step 4: Query via REST API

Once errors flow into Watchy, AI agents query them via the REST API:

```
Agent: "What errors happened in the last 24 hours?"
→ GET /api/v1/errors?time_range=24h returns: real stack traces, criticality, business context, linked to your operational graph

Agent: "Show me critical errors in the payments stack"
→ GET /api/v1/errors?time_range=24h&include_traces=true returns: TimeoutError in processPayment, 47 occurrences, context: { orderId, amount }
```

Create an API key with `read:errors` + `read:stacks` scopes to query. See the `watchy-debug` skill for the full agent workflow.

## Prerequisites

- A Watchy account with at least one connected AWS account
- A Watchy API key with `write:errors` scope (Settings -> API Keys)
- AWS CLI configured with permissions to create Secrets Manager secrets

## Criticality Levels

Every captured error has a criticality level that determines how it's prioritized:

| Level | When to use | Examples |
|---|---|---|
| `critical` | Revenue impact, data loss, security breach | Payment processing, data mutations, auth failures |
| `high` | User-facing failures, SLA violations | API responses, form submissions, file uploads |
| `medium` | Degraded experience, retryable failures | Cache misses, slow queries, partial feature failure |
| `low` | Background tasks, non-blocking operations | Analytics, notifications, cleanup jobs |

## Context — The Flexible Bag

Every error carries a `context` object — free-form JSON where you pack whatever makes debugging useful. The agent suggests context fields based on what each handler does.

```typescript
// Payment handler → pack business data
{ orderId, userId, amount, currency, paymentProvider: 'stripe' }

// Auth handler → pack identity data
{ userId, authMethod: 'oauth', provider: 'google' }

// Background job → pack progress data
{ batchId, processedCount: 47, totalCount: 100, retryable: true }
```

## Step 5: Investigate Errors

Once errors are flowing into Watchy, you can investigate them using the full investigation playbook. When a developer says "investigate this error" or "debug this 500", follow the playbook:

Link: references/investigation-playbook.md

The investigation playbook gives you a step-by-step workflow to:
1. Find and retrieve error context from Watchy's API (stack traces, business context, resource graph)
2. Pull CloudWatch logs for the specific Lambda invocation
3. Check execution metadata (memory/timeout issues)
4. Check git history for recent changes
5. Correlate with CloudTrail deployments
6. Produce a structured root cause analysis with suggested fix

The key endpoint is `GET /api/v1/errors/{id}/investigate` — it returns everything needed in one call: error details, Lambda config, resource graph, recent deployments, and error trend.

## References

- **Interactive workflow** (step-by-step for agents): references/agent-workflow.md
- **Investigation playbook** (error debugging for agents): references/investigation-playbook.md
- **API specification** (endpoint, schemas, rate limits): references/api-reference.md
- **Code patterns** (complete service file + integration patterns): references/lambda-patterns.md
