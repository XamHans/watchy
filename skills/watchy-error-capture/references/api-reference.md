# Watchy Error Capture API Reference

## Authentication

All requests require a Watchy API key with `write:errors` scope.

```
Authorization: Bearer wky_xxxxxxxxxxxx
```

API keys are created in Watchy Settings -> API Keys.

## POST /api/v1/errors

Ingest a batch of error events from Lambda functions.

### Request

```
POST https://app.watchy.dev/api/v1/errors
Content-Type: application/json
Authorization: Bearer wky_xxxxxxxxxxxx
```

### Request Body

```json
{
  "events": [
    {
      "errorName": "TypeError",
      "errorMessage": "Cannot read property 'items' of undefined",
      "stackTrace": "TypeError: Cannot read property...\n    at handler (index.ts:47)\n    ...",
      "level": "error",
      "criticality": "critical",
      "functionName": "processOrder",
      "functionVersion": "$LATEST",
      "awsAccountId": "123456789012",
      "awsRegion": "us-east-1",
      "requestId": "abc-def-123-456",
      "logGroup": "/aws/lambda/processOrder",
      "logStream": "2026/03/24/[$LATEST]abc123",
      "memoryMb": 256,
      "coldStart": false,
      "remainingMs": 25000,
      "environment": "production",
      "release": "v2.3.1",
      "tags": { "orderId": "ord-789" },
      "extra": { "userId": "user-456" },
      "context": { "orderId": "ord-789", "userId": "user-456", "amount": 99.99, "currency": "USD" },
      "fingerprint": "optional-custom-fingerprint",
      "sdkVersion": "skill-1.0",
      "timestamp": "2026-03-24T14:30:00.000Z"
    }
  ]
}
```

### Field Reference

| Field | Type | Required | Max Length | Description |
|---|---|---|---|---|
| `errorName` | string | yes | 256 | Error class name (e.g. `TypeError`, `TimeoutError`) |
| `errorMessage` | string | yes | 1024 | Error message text |
| `stackTrace` | string | no | 8192 | Full stack trace |
| `level` | enum | no | - | `error` (default), `warning`, or `fatal` |
| `criticality` | enum | no | - | `low`, `medium` (default), `high`, or `critical` |
| `functionName` | string | yes | 256 | Lambda function name |
| `functionVersion` | string | no | 64 | Lambda function version |
| `awsAccountId` | string | yes | 20 | AWS account ID (12 digits) |
| `awsRegion` | string | yes | 32 | AWS region (e.g. `us-east-1`) |
| `requestId` | string | yes | 256 | Lambda invocation request ID |
| `logGroup` | string | no | 512 | CloudWatch log group |
| `logStream` | string | no | 512 | CloudWatch log stream |
| `memoryMb` | integer | no | - | Lambda memory allocation in MB |
| `coldStart` | boolean | no | - | Whether this was a cold start invocation |
| `remainingMs` | integer | no | - | Milliseconds remaining when error occurred |
| `environment` | string | no | 64 | Deployment environment (`production`, `staging`) |
| `release` | string | no | 128 | Release/version tag |
| `tags` | object | no | - | Key-value string pairs for filtering |
| `extra` | object | no | - | Arbitrary context data (legacy — prefer `context`) |
| `context` | object | no | 16KB | Flexible context bag — business data, user info, anything useful for debugging |
| `fingerprint` | string | no | 64 | Custom dedup key. If omitted, auto-computed from errorName + top 3 stack frames |
| `sdkVersion` | string | no | 32 | Client version identifier |
| `timestamp` | string | yes | - | ISO 8601 datetime when the error occurred |

### Batch Limits

- Minimum: 1 event per request
- Maximum: 25 events per request

### Response

**Success (200):**
```json
{
  "success": true,
  "data": {
    "accepted": 3,
    "dropped": 0
  }
}
```

**Validation Error (400):**
```json
{
  "success": false,
  "code": "VALIDATION_ERROR",
  "error": "Validation failed",
  "details": { "issues": [{ "path": "events.0.timestamp", "message": "Required" }] }
}
```

**Unauthorized (401):**
```json
{
  "success": false,
  "code": "UNAUTHORIZED",
  "error": "Missing or invalid Authorization header"
}
```

**Scope Insufficient (403):**
```json
{
  "success": false,
  "code": "SCOPE_INSUFFICIENT",
  "error": "API key missing required scope: write:errors"
}
```

**Rate Limited (429):**
```json
{
  "success": false,
  "code": "RATE_LIMIT_EXCEEDED",
  "error": "Error ingestion rate limit exceeded for your plan"
}
```

### Rate Limits

| Plan | Events/hour | Events/day |
|---|---|---|
| Hobby | 200 | 1,000 |
| Pro | 5,000 | 50,000 |
| Team | 50,000 | 500,000 |
| Scale | 500,000 | 5,000,000 |

Rate limits are per organization, not per API key. When exceeded, the API returns 429 for the entire batch. Events are not partially accepted when rate limited.
