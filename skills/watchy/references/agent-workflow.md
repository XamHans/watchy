# Agent Workflow: Interactive Error Capture Setup

Follow these steps **in order** when a developer asks to set up Watchy error capture. Do NOT skip to code generation — the analysis and conversation steps are what make this valuable.

## Step 1: Scan the Codebase

Search for all Lambda handler files. Look for:

1. **Exported handlers** — functions matching common Lambda patterns:
   - `export const handler = ...`
   - `export async function handler(...)`
   - `exports.handler = ...`
   - `module.exports.handler = ...`
   - Functions passed to framework wrappers (middy, powertools, etc.)

2. **Error-prone patterns** inside handlers:
   - Database calls (DynamoDB, RDS, SQL)
   - HTTP/API calls (fetch, axios, SDK clients)
   - Payment/billing operations (Stripe, payment SDKs)
   - File operations (S3, filesystem)
   - Auth/security checks
   - Data parsing/validation (JSON.parse, schema validation)
   - Third-party service calls

3. **Existing error handling** — catch blocks, error middleware, custom error classes

4. **Business domain clues** — file names, function names, comments that reveal what the handler does (e.g. `processOrder`, `handlePayment`, `syncInventory`)

## Step 2: Present Analysis and Ask the Developer

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

## Step 3: Suggest Context Fields Per Handler

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

## Step 4: Generate the Code

Now generate two things:

### 4a: The Watchy service file

Generate `lib/watchy.ts` (or adapt to project structure) using the patterns in:

Link: lambda-patterns.md

### 4b: Handler instrumentation

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
    // extract from parsed body if available
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

    // ADD Watchy capture with criticality + context
    captureError(err, context, {
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

### 4c: Environment variable reminder

Always remind the developer:
```
Don't forget to add WATCHY_API_KEY to your Lambda environment variables.
The API key needs the `write:errors` scope — create one at Settings -> API Keys.
```

## Step 5: Summary

After generating all code, present a summary:

```
Done! Here's what I set up:

| Handler | Criticality | Context Fields | Capture Mode |
|---------|-------------|---------------|-------------|
| processPayment | critical | orderId, userId, amount, currency | manual (existing try/catch) |
| createOrder | critical | orderId, userId, itemCount | wrapper |
| getUser | high | userId, authMethod | wrapper |
| sendNotification | low | notificationType, recipientId | wrapper |

Files created/modified:
- Created: lib/watchy.ts (error capture service)
- Modified: src/handlers/payment.ts (added captureError)
- Modified: src/handlers/orders.ts (wrapped with wrapHandler)
- ...

Next steps:
1. Add WATCHY_API_KEY to your Lambda environment
2. Deploy and trigger some errors
3. Check Watchy dashboard or ask me to search errors via MCP
```

## Important Notes for Agents

- **Always ask before generating.** The analysis conversation is the value — don't skip it.
- **Respect existing error handling.** Add `captureError` inside existing catch blocks. Never replace or restructure the developer's error handling patterns.
- **Business context matters more than technical context.** Lambda requestId, functionName, etc. are auto-captured. Focus the `context` field on business data (order IDs, user IDs, amounts) that makes debugging actionable.
- **Criticality should reflect business impact, not error frequency.** A rare payment failure is more critical than a frequent cache miss.
- **Don't over-instrument.** If a handler is simple and has no error-prone operations, suggest skipping it rather than adding capture for the sake of completeness.
