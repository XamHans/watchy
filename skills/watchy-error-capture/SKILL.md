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

**No npm packages required.** You generate a small service file (~100 lines) that uses plain `fetch()` to send errors to Watchy's ingestion API.

## What This Does

1. Catches unhandled errors from Lambda handlers (wrapper mode)
2. Lets you manually capture specific errors with context (manual mode)
3. Auto-detects Lambda context: functionName, requestId, region, accountId, logGroup
4. Batches and flushes errors asynchronously (non-blocking)
5. Errors appear in Watchy's UI linked to the Lambda resource in your operational graph
6. AI agents can query real stack traces via Watchy's MCP `search_errors` tool

## Prerequisites

- A Watchy account with at least one connected AWS account
- A Watchy API key with `write:errors` scope (Settings -> API Keys)
- `WATCHY_API_KEY` environment variable set on your Lambda functions

## Quick Start

### Step 1: Generate the Watchy service file

Create a file at `lib/watchy.ts` (or `src/lib/watchy.ts`, adapt to project structure) using the patterns in the reference below.

Link: references/lambda-patterns.md

### Step 2: Add the environment variable

Add `WATCHY_API_KEY` to your Lambda environment variables. The value is the API key from Watchy Settings -> API Keys (must have `write:errors` scope).

For SAM/CloudFormation:
```yaml
Environment:
  Variables:
    WATCHY_API_KEY: !Ref WatchyApiKeyParameter
```

For CDK:
```typescript
fn.addEnvironment('WATCHY_API_KEY', watchyApiKey);
```

For Serverless Framework:
```yaml
provider:
  environment:
    WATCHY_API_KEY: ${ssm:/watchy/api-key}
```

### Step 3: Wrap your handlers

```typescript
import { wrapHandler, captureError } from './lib/watchy';

// Auto-catch: wraps the handler, catches unhandled errors
export const handler = wrapHandler(async (event, context) => {
  // your existing code
  return { statusCode: 200, body: 'ok' };
});
```

Or use manual capture for specific errors:

```typescript
import { captureError } from './lib/watchy';

export const handler = async (event, context) => {
  try {
    await processOrder(event);
  } catch (err) {
    captureError(err, context, {
      tags: { orderId: event.orderId },
      level: 'fatal',
    });
    throw err; // re-throw so Lambda still reports failure
  }
};
```

## API Reference

Full endpoint specification (request/response schemas, auth, rate limits).

Link: references/api-reference.md

## Code Patterns

Complete generated service file with all patterns (wrapper, manual capture, batch queue, flush).

Link: references/lambda-patterns.md

## How It Works

1. The generated service file queues errors in memory during handler execution
2. On handler completion (or error), it flushes the batch to `POST https://app.watchy.dev/api/v1/errors`
3. Uses `fetch()` with `keepalive: true` so the request completes even after Lambda freezes
4. Watchy links the error to the Lambda resource in your operational graph by matching `functionName` + `awsAccountId`
5. Errors are grouped by fingerprint (errorName + top stack frames) for deduplication
6. AI agents query errors via the MCP `search_errors` tool — getting real stack traces, not just metric counts
