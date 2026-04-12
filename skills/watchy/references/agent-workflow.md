# Agent Workflow: Full Setup + Error Capture

Follow these steps **in order** when a developer asks to set up Watchy error capture. Do NOT skip to code generation — the analysis and conversation steps are what make this valuable.

## Step 0: Prerequisites Check

Before anything, verify the developer has:

1. **A Watchy account** with at least one connected AWS account
2. **A Watchy API key** with `write:errors` scope

If they don't have an API key yet, tell them:
```
You need a Watchy API key with write:errors scope.
Go to watchy.dev -> Settings -> API Keys -> Create Key.
Copy the key — you'll need it in the next step.
```

**Wait for the developer to confirm they have the API key before proceeding.**

## Step 1: Store the API Key in AWS Secrets Manager

**Do NOT store the API key as a plain Lambda environment variable.** Use AWS Secrets Manager.

Ask the developer:
```
I'll store your Watchy API key in AWS Secrets Manager so your Lambdas can access it securely.

Which AWS region should I store the secret in? (same region as your Lambdas)
```

Then run via CLI (or instruct the developer to run):

```bash
aws secretsmanager create-secret \
  --name watchy/api-key \
  --description "Watchy API key for error capture" \
  --secret-string "wky_YOUR_API_KEY_HERE" \
  --region us-east-1
```

If the secret already exists, update it:
```bash
aws secretsmanager put-secret-value \
  --secret-id watchy/api-key \
  --secret-string "wky_YOUR_API_KEY_HERE" \
  --region us-east-1
```

**Grant Lambda access** — the Lambda execution role needs `secretsmanager:GetSecretValue` permission. Check the current role policy. If it doesn't have it, tell the developer:

```
Your Lambda execution role needs permission to read the secret.
Add this to your IAM policy (or CloudFormation/SAM template):

{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:watchy/api-key*"
}
```

**Important:** The developer must confirm the secret is created and the IAM permission is in place before proceeding.

## Step 2: Scan the Codebase

Search for all Lambda handler files. Look for:

1. **Exported handlers** — functions matching common Lambda patterns:
   - `export const handler = ...`
   - `export async function handler(...)`
   - `exports.handler = ...`
   - `module.exports.handler = ...`
   - Functions passed to framework wrappers (middy, powertools, etc.)

2. **Shared abstractions and middleware** — this is the most important scan:
   - Base handler wrappers (`createHandler()`, `withErrorHandling()`, `baseHandler()`)
   - Middleware chains (middy, custom middleware, onion patterns)
   - Centralized error handling (`handleError()`, error middleware, global catch)
   - Custom error class hierarchies (`AppError`, `BusinessError`, `PaymentError`)
   - DI containers, event processors, queue consumers that wrap handler logic
   - **If you find any of these, Watchy should be integrated THERE — not in every handler.**

3. **Error-prone patterns** inside handlers:
   - Database calls (DynamoDB, RDS, SQL)
   - HTTP/API calls (fetch, axios, SDK clients)
   - Payment/billing operations (Stripe, payment SDKs)
   - File operations (S3, filesystem)
   - Auth/security checks
   - Data parsing/validation (JSON.parse, schema validation)
   - Third-party service calls

4. **Existing error handling** — catch blocks, error middleware, custom error classes

5. **Business domain clues** — file names, function names, comments that reveal what the handler does (e.g. `processOrder`, `handlePayment`, `syncInventory`)

6. **Code style** — TypeScript vs JavaScript, ESM vs CommonJS, named vs default exports, strict mode

## Step 3: Present Analysis and Ask the Developer

Present a table of discovered handlers with your suggested criticality and reasoning:

```
I found N Lambda handlers in your project. Here's my analysis:

| # | Handler | File | Suggested Criticality | Reason |
|---|---------|------|----------------------|--------|
| 1 | processPayment | src/handlers/payment.ts | critical | Payment processing — revenue impact, data mutation |
| 2 | createOrder | src/handlers/orders.ts | critical | Order creation — core business flow |
| 3 | getUser | src/handlers/users.ts | high | User-facing API — auth flow |
| 4 | sendNotification | src/handlers/notify.ts | low | Notifications are async and retryable |
| 5 | syncAnalytics | src/handlers/analytics.ts | low | Background analytics — no user impact |

Questions for you:
1. Does this criticality mapping look right? Anything you'd change?
2. Are there handlers I missed?
3. Any handlers you want to EXCLUDE from error capture?
```

**Wait for the developer's response before proceeding.**

## Step 4: Suggest Context Fields Per Handler

For each handler the developer approved, suggest what business context to capture in the flexible `context` field:

```
Here's what I recommend capturing in the `context` field for each handler:

**processPayment** (critical):
- orderId, userId, amount, currency
- paymentMethod, paymentProvider
- Why: when this fails, you need to know which order/user was affected and the payment details

**createOrder** (critical):
- orderId, userId, items (count), totalAmount
- Why: order creation failures need full order context for debugging and customer support

**getUser** (high):
- userId, authMethod
- Why: auth failures need user identity and method to diagnose

**sendNotification** (low):
- notificationType, recipientId, channel
- Why: lightweight context for retry debugging

Should I adjust any of these? Want to add/remove fields?
```

**Wait for the developer's response before proceeding.**

## Step 5: Generate the Code

Now generate two things:

### 5a: The Watchy service file

Generate `lib/watchy.ts` (or adapt to project structure) using the patterns in:

Link: lambda-patterns.md

**Important:** The generated service file fetches the API key from Secrets Manager at cold start — NOT from a plain environment variable. See lambda-patterns.md for the Secrets Manager caching pattern.

### 5b: Handler instrumentation

For each approved handler, add error capture. Choose the right pattern:

**For handlers WITHOUT existing try/catch:**
Wrap with `wrapHandler` and set default criticality + context extraction:

```typescript
import { wrapHandler } from '../lib/watchy';

// Before
export const handler = async (event, context) => {
  const order = await createOrder(event.body);
  return { statusCode: 200, body: JSON.stringify(order) };
};

// After
export const handler = wrapHandler(async (event, context) => {
  const order = await createOrder(event.body);
  return { statusCode: 200, body: JSON.stringify(order) };
}, {
  criticality: 'critical',
  contextExtractor: (event) => ({
    userId: event.requestContext?.authorizer?.userId,
  }),
});
```

**For handlers WITH existing try/catch:**
Add `captureError` inside existing catch blocks — do NOT restructure their code:

```typescript
import { captureError } from '../lib/watchy';

export const handler = async (event, context) => {
  try {
    const payment = await processPayment(event.body);
    return { statusCode: 200, body: JSON.stringify(payment) };
  } catch (err) {
    // EXISTING error handling — don't remove
    logger.error('Payment failed', err);

    // ADD Watchy capture — MUST await, Lambda freezes after handler returns
    await captureError(err, context, {
      criticality: 'critical',
      context: {
        orderId: event.body?.orderId,
        userId: event.requestContext?.authorizer?.userId,
        amount: event.body?.amount,
      },
    });

    return { statusCode: 500, body: 'Payment failed' };
  }
};
```

**For codebases WITH shared middleware/wrapper:**
Integrate into the centralized error handler — do NOT add to individual handlers:

```typescript
// If the user has something like this:
import { captureError } from '../lib/watchy';

// Their existing middleware/wrapper
export function createHandler(config, handlerFn) {
  return async (event, context) => {
    try {
      return await handlerFn(event, context);
    } catch (err) {
      logger.error(err);

      // Add Watchy here — one place covers ALL handlers
      await captureError(err, context, {
        criticality: config.criticality || 'medium',
        context: config.contextExtractor?.(event) || {},
      });

      return { statusCode: 500, body: 'Internal error' };
    }
  };
}
```
```

## Step 6: Verify with a Test Error

After code generation, send a test error via curl to verify the full pipeline works.

Tell the developer:
```
Let's verify everything works end-to-end. Run this curl to send a test error:
```

```bash
curl -X POST https://watchy.dev/api/v1/errors \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer wky_YOUR_API_KEY" \
  -d '{
    "events": [{
      "errorName": "WatchySetupTest",
      "errorMessage": "This is a test error from Watchy skill setup — safe to ignore",
      "stackTrace": "WatchySetupTest: test error\n    at setup-verification (test:1:1)",
      "level": "warning",
      "criticality": "low",
      "functionName": "watchy-setup-test",
      "awsAccountId": "YOUR_AWS_ACCOUNT_ID",
      "awsRegion": "us-east-1",
      "requestId": "test-setup-verification",
      "environment": "setup-test",
      "context": { "source": "watchy-skill-setup", "step": "verification" },
      "sdkVersion": "skill-1.0",
      "timestamp": "'"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)"'"
    }]
  }'
```

Replace `wky_YOUR_API_KEY` with the actual API key and `YOUR_AWS_ACCOUNT_ID` with the 12-digit AWS account ID connected to Watchy.

**Expected response:**
```json
{ "success": true, "data": { "accepted": 1, "dropped": 0 } }
```

If it returns `accepted: 1`, tell the developer:

```
The test error was accepted. Now open your Watchy dashboard:

  watchy.dev -> Your Account -> Errors tab

You should see the "WatchySetupTest" error appear within a few seconds.
Can you confirm you see it?
```

**Wait for the developer to confirm the error shows in the dashboard before proceeding.**

If the curl fails:
- **401 Unauthorized** → API key is wrong or missing `write:errors` scope
- **429 Rate Limited** → org has exceeded error ingestion limits
- **Connection error** → check network/firewall

## Step 7: Summary

After everything is verified, present a summary:

```
Setup complete! Here's what we configured:

**API Key Storage:**
- Secret: watchy/api-key in AWS Secrets Manager (us-east-1)
- Lambda role has secretsmanager:GetSecretValue permission

**Error Capture:**
| Handler | Criticality | Context Fields | Capture Mode |
|---------|-------------|---------------|-------------|
| processPayment | critical | orderId, userId, amount, currency | manual (existing try/catch) |
| createOrder | critical | orderId, userId, itemCount | wrapper |
| getUser | high | userId, authMethod | wrapper |
| sendNotification | low | notificationType, recipientId | wrapper |

**Files created/modified:**
- Created: lib/watchy.ts (error capture service with Secrets Manager caching)
- Modified: src/handlers/payment.ts (added captureError)
- Modified: src/handlers/orders.ts (wrapped with wrapHandler)
- ...

**Verified:**
- Test error sent via curl → accepted by API
- Test error visible in Watchy dashboard

**Next steps:**
1. Deploy your changes
2. Trigger a real error (or wait for one to occur naturally)
3. Check your Watchy dashboard or ask me to search errors via the Watchy REST API
```

## Important Notes for Agents

- **Always ask before generating.** The analysis conversation is the value — don't skip it.
- **Respect existing error handling.** Add `captureError` inside existing catch blocks. Never replace or restructure the developer's error handling patterns.
- **Business context matters more than technical context.** Lambda requestId, functionName, etc. are auto-captured. Focus the `context` field on business data (order IDs, user IDs, amounts) that makes debugging actionable.
- **Criticality should reflect business impact, not error frequency.** A rare payment failure is more critical than a frequent cache miss.
- **Don't over-instrument.** If a handler is simple and has no error-prone operations, suggest skipping it rather than adding capture for the sake of completeness.
- **Use Secrets Manager, not plain env vars.** The API key is a credential — treat it like one.
- **Always verify with a test error.** Don't declare setup complete until the developer confirms they see the test error in the dashboard.

### CRITICAL: Always `await captureError()` in Lambda

- **`captureError` is async — it sends an HTTP request to Watchy.** Lambda freezes the execution environment the moment the handler function returns. If `captureError` is not awaited, the outbound HTTP request to Watchy may never complete. The error is silently lost.
- **Always write `await captureError(...)`, never bare `captureError(...)`.** This is the single most important integration rule.
- **`wrapHandler` handles this automatically** — it awaits `flush()` in its `finally` block. But any manual `captureError()` in the user's own code MUST be awaited.
- **If the user has middleware that catches errors** — the `await captureError(...)` must happen before the middleware returns. Verify the middleware is async and the call chain preserves the await.

### CRITICAL: Adapt to the user's codebase — don't fight the architecture

- **Before writing any handler code, look for shared patterns.** If the user has a `createHandler()`, `withMiddleware()`, base class, error boundary, or any centralized error-catching layer — integrate Watchy THERE. One integration point beats 20 scattered ones.
- **If the user has a custom error class hierarchy** (e.g. `AppError`, `HttpError`, `PaymentError`) — map their error types to Watchy criticality levels automatically instead of hardcoding per-handler.
- **If the user uses middy or a similar middleware framework** — add Watchy as a middleware, not as inline code in every handler.
- **Match the user's code style exactly.** TypeScript vs JS, ESM vs CJS, named vs default exports, semicolons, quotes — follow what they already use. Don't introduce a different style.
- **Ask where errors bubble up to.** The best place to add Watchy is wherever errors are already being caught or logged. Don't add new catch blocks if the user already has comprehensive error handling.
